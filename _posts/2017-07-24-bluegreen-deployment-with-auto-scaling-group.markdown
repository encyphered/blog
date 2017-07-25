---
layout: post
title: "배포시스템 삽질기 - 2"
description: "Blue-green deployment with Auto Scaling Group"
categories: dev
---

지난 [배포시스템 삽질기](/blog/dev/2016/12/17/deployment-in-aws.html)에 이어서, 그 뒤로 시간이 지났고 변경도 있었기에 기록.

개선해야 할 점으로 꼽았던 것은 두 가지가 있었다.

1. 한두대 정도에 먼저 배포해서 테스트를 할 때는 여전히 SSH로 서버에 직접 들어가서 작업을 하고 있다.
2. 배포를 하려면 docker-compose.yml 에 이미지 버전을 올려서 커밋&푸시하고 ASG 설정을 직접 고쳐야한다.

### 현황

지난번 글에서 끄트머리에 적어두었던대로, bootstrap 을 위한 initialization 스크립트 및 설정파일들은 저장소에 푸시하면
CI에서 S3로 업로드하도록 변경했다. EC2 인스턴스에서 S3 로의 접근은 VPC 에서 VPC Endpoint 를 설정하고
S3 의 버킷에 권한을 설정해 주면, 별다른 설정 없이도 curl/wget 만으로도 읽어올 수 있으니 간단하다.

그리고, 배포를 할 때마다 번거롭게 파일 수정하고 배포하고 S3에 업로드할때까지 기다려야 하는 문제 때문에,
live 될 Docker 이미지의 버전을 직접 지정하지 않고 bootstrap 의 docker-compose.yml 에는 버전을 고정시키도록 변경했다.
그리고 실제 사용할 버전은 ASG의 Tag 로 지정해두고, 인스턴스가 뜨면서 그 값을 읽어서 Docker 이미지를 땡겨서 docker-compose.yml 에
지정된 버전으로 태깅하도록 변경.[^1]

그럼 이제, 한대만 배포해서 모니터링할때도 좀 더 편하다. ASG의 tag 수정하고 인스턴스 하나 늘려서 확인하면 되니까.

#### Blue-green 배포

그러나 롤백도 문제다. 새 버전으로 인스턴스 싹 교체해놨는데 배포직후 급히 롤백하려면 다시 구 버전으로 새로 띄워야한다.
구버전 인스턴스를 끄지 않고 놔뒀다가 다시 투입하면 되긴 하지만, 이 타이밍에 scaleout 이 돌거나 할때 귀찮은게 문제다.
그리고 그걸 직접 해줘야해서 귀찮기도 하고.

그래서 blue-green 배포로 변경. 원래 Beanstalk 처럼 CNAME swap 같은 방식으로 하려다가, DNS 캐시때문에 영 찝찝했던지라
그냥 같은 ELB를 사용하는 ASG를 두개를 만들었다.

1. blue ASG에서 서비스가 되고 있다면 green ASG의 docker image version 태그를 변경한다.
2. blue ASG의 capacity 를 green ASG로 복사해오면 새 버전으로 인스턴스가 뜬다.
3. 인스턴스가 뜬 후, blue ASG의 인스턴스의 lifecycle 을 전부 standby 로 돌리면, 구 버전 인스턴스들은 실행은 되어있고 ASG에는 등록되어있으나 ELB에서는 빠진다.
4. 롤백을 해야 한다면, blue ASG의 standby 중인 인스턴스들을 in service 로 돌려서 다시 ELB에 투입하고 green ASG의 capacity 를 다시 0으로 돌린다.
5. 롤백이 필요 없다면, blue ASG의 standby 중인 인스턴스들을 그대로 detach[^2] 혹은 terminate[^3] 시키고, scaling policy / scheduled aciton 을 blue 에서 green 으로 옮겨온다.

4번의 과정에서 CNAME swap등에 비해 좀 더 번거롭고 시간이 걸리긴 하겠지만, DNS전파에도 속도가 걸리는건 마찬가지고,
ELB의 healthcheck 주기를 좀 더 짧고 민감하게 가져가서 해결할 수 있다. 큰 문제는 되지 않음.

결론적으로, 배포도 롤백도 이제 2분 정도에 완료.  
아, 그리고 당연히 이걸 손으로 하긴 번거로우니, 스크립트로 만들어서 사내 pip repository 에 올려두고 사용중이다.

ps.  
현재 시점에서는 이 상태이지만, 요즘엔 다시 container orchestration을 (또-\_-) 궁리중이라 언제까지 유지될지는 글쎄...

[^1]:
    참고로 ASG의 tag에서 ​Tag on instance 를 체크해두더라도, 인스턴스에 ASG의 tag가 쓰여지는 시점이 명확히 보장되지 않는다.
    그렇기 때문에 cloud-init 가 실행되는 시점에 ASG의 tag가 인스턴스에는 없을 수도 있다(!). 현재 인스턴스의 tag 대신,
    현재 인스턴스가 속한 ASG의 tag를 읽어와야 한다.

[^2]:
    인스턴스를 standby로 전환할 때 ShouldDecrementDesiredCapacity 옵션을 줘서 capacity 를 줄여놨다면, ​detach 할때는
    이미 capacity 가 줄어들어 있는 상태기 때문에 ShouldDecrementDesiredCapacity 를 False 로 지정해야 한다.

[^3]:
    terminate 전에 detach 를 먼저 해줘야한다. 안그러면 terminated 상태인 인스턴스가 ASG에 standby 상태로 물려있는 괴상한 꼬라지가...
