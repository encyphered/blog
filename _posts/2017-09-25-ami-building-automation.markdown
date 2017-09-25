---
layout: post
title: "AMI 빌드 자동화"
description: "Automated AMI building with CloudFormation"
categories: dev
---

인스턴스가 뜰 때마다 새로 프로비저닝을 처음부터 하는 건 여러모로 비효율적이다. 스케일업에 들어가는 시간도 만만치 않고.
그러다보니 AMI로 만들어서 관리를 하는 편인데, AMI를 만드는거는 쉽지만 프로비저닝 이력의 관리+자동화는 생각보다 귀찮다.
그간 이력 관리 정도만 하다가 이참에 CloudFormation 을 이용해 템플릿을 만들었다.

이미 [이런 게](https://github.com/awslabs/ami-builder-packer) 있긴 하지만, 달랑 이미지 하나 빌드하자고
Packer 에다 SNS 동원하는게 과하다 싶어서 CloudFormation 만을 이용해서 만들었다.

일단 전제조건.

1. 인스턴스 프로비저닝은 user data 로 대충 해결이 가능하지만, 이게 완료된 다음 AMI 생성 요청을 해야 한다.
2. CloudFormation 으로는 AMI 생성이 불가능하며, AMI 생성을 위해서는 별도로 API호출을 통해야 한다.

이걸 CloudFormation 만을 이용해서 풀어낸다면 대충 다음과 같은 식이다.

1. AMI 생성을 위한 EC2 인스턴스를 만들고, 프로비저닝을 한다.
2. Lambda function 을 하나 만든다. 이 함수는 EC2 인스턴스를 이용해 AMI를 만들고 CloudFormation 에 완료 사인을 준다.
3. `AWS::CloudFormation::WaitCondition` 리소스를 하나 만들고 적당히 타임아웃을 준다.
4. Custom Resource 를 정의하고 위의 WaitCondition 리소스에 의존성을 걸어준다.
이 Custom Resource 는 `Fn::GetAtt` 을 통해 2번의 Lambda function 을 호출한다.
5. 1번에서의 인스턴스 프로비저닝이 끝나면 `cfn-signal` 을 이용해 WaitCondition 을 완료시킨다.

코드로는 대략 다음과 같다.

    Description: This CloudFormation creates a AMI image.
    Parameters:
      ImageId:
        Type: AWS::EC2::Image::Id
        Description: Image ID for base EC2 instance.
        Default: ami-d28a53bc
    Resources:
      InstanceReady:
        Type: AWS::CloudFormation::WaitCondition
        CreationPolicy:
          ResourceSignal:
            Timeout: PT10M
      BaseInstance:
        Type: AWS::EC2::Instance
        Properties:
          ImageId: !Ref ImageId
          InstanceType: t2.micro
          KeyName: masterkey
            - sg-31415926
          UserData:
            "Fn::Base64": !Sub |
              #!/bin/bash -x
              # INSTANCE PROVISIONING HERE

              pip install https://s3.amazonaws.com/cloudformation-examples/aws-cfn-bootstrap-latest.tar.gz
              cfn-signal --region ${AWS::Region} --stack ${AWS::StackName} --resource InstanceReady --exit-code 0

              shutdown -h now
      AMIBuildingFunctionInvoker:
        Type: Custom::FunctionInvoker
        DependsOn: InstanceReady
        Properties:
          LambdaArn: !GetAtt CreateImage.Arn
          BaseImageId: !Ref ImageId
          InstanceId: !Ref BaseInstance
      CreateImage:
        Type: AWS::Lambda::Function
        Properties:
          Handler: index.handle
          Role: !GetAtt LambdaExecutionRole.Arn
          Code:
            ZipFile: !Sub |
              import boto3
              import cfnresponse
              import time

              ec2 = boto3.client('ec2')

              def handle(event, context):
                instance_id = event['ResourceProperties']['InstanceId']
                image_name = '%s-%09x' % (event['ResourceProperties']['BaseImageId'], int(time.time()))
                response = ec2.create_image(InstanceId=instance_id, Name=image_name)
                cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, response['ImageId'])
          Runtime: python3.6
          Timeout: 300
      LambdaExecutionRole:
        Type: AWS::IAM::Role
        Properties:
          AssumeRolePolicyDocument:
            Version: '2012-10-17'
            Statement:
            - Effect: Allow
              Principal: {Service: [lambda.amazonaws.com]}
              Action: ['sts:AssumeRole']
          Path: /
          ManagedPolicyArns:
          - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
          - arn:aws:iam::aws:policy/service-role/AWSLambdaRole
          Policies:
          - PolicyName: EC2Policy
            PolicyDocument:
              Version: '2012-10-17'
              Statement:
                - Effect: Allow
                  Action:
                  - 'ec2:CreateImage'
                  Resource: ['*']

이걸 실행시키면...

1. BaseInstance 리소스가 생성되며 EC2 인스턴트가 뜨고 UserData 의 스크립트에 따라 프로비저닝이 된다.
2. CreateImage 리소스의 Lambda function 은 동시에 생성이 되고, BaseInstance 보다 이쪽이 먼저 완료된다.
3. 인스턴스 프로비저닝이 끝나면 `cfn-signal` 에 의해 InstanceReady 리소스가 완료된다.
4. InstanceReady 리소스가 완료되면 의존성이 걸려있는 AMIBuildingFunctionInvoker 가 동작하고, `Fn::GetAtt` 에 의해 CreateImage 의 Lambda function 이 실행된다.
5. Lambda function 은 AMI 생성 명령을 실행한다
6. PROFIT!

이제 생성된 CloudFormation 스택을 삭제하면 EC2 인스턴스와 Lambda function, 그리고 role 까지 깔끔하게 삭제되고,
갓 구워진 AMI만 남게 된다.

좀 더 좋은 방법이 있을지도 모르겠지만 일단 여기서 만족.
