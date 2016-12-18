---
layout: post
title: "배포시스템 삽질기"
description: "AWS 배포시스템 삽질 : Beanstalk-Capistrano-Fabric-Docker"
categories: dev
---

2016년 봄 언저리쯤에, PaaS 기반의 시스템에서 IaaS(정확히는 GAE에서 AWS로)로 일부 전환을 하면서 배포시스템에 대한 고민이 필요해졌다.

대상 컴포넌트는 Python(Flask) 하나, Java 두개. 사실 Java 컴포넌트는 war 빌드해서 Tomcat 이 설치된 서버에
집어넣기만 하면 되는데다 JVM만 깔려있으면 특별히 뭐 해줄게 없으니 신경쓸일이 별로 없지만, Python 은 일단 같이
설치 및 빌드해줘야하는 라이브러리라거나, Python 인터프리터 버전의 문제나 이거저거 신경쓸게 좀 있는지라 고민의 초점은
주로 전자에 맞춰져 있었다.

### 솔루션 고민

#### Elastic Beanstalk

AWS 를 쓴다면 제일 먼저 생각할 수 있는 솔루션은 Beanstalk 일테지만, 다음과 같은 이유로 잠깐 테스트해보고 접었다.

1. 당시에는 Jenkins 를 쓰고 있었는데, 당시 Beanstalk 플러그인이 AWS 서울 리전이 지원하지 않았다[^1].
2. 단순히 AWS SDK for Java 버전의 문제라 패치보내고 금방 적용되긴 했지만 사실 플러그인 자체가 좀 부실했다.
3. 게다가 이런 류의 솔루션은 나중에 뭔가 좀 많이 커스터마이징하려면 결국 다시 더 로우레벨로 돌아오게 되더라.
4. 뭔가 기억은 잘 안나는데 ImageMagick 관련해서 이슈가 있었음.

4번은 어이없지만(...) 사실 저게 이 글을 쓰기 시작한 이유다. 나머지도 까먹기 전에 기록해두려고 orz

#### Capistrano / Fabric

제일 먼저 생각난건 써본적 있는 Capistrano 였지만, 일단 이걸 쓰던게 하도 오래전이라(버전1 시절이었나... 대충 2009년쯤?)
어차피 새로 봐야할거같았고, 언어셋을 하나 더 추가하고 싶지가 않다는 지극히 개인적인 이유로 접었다.

그다음에 생각난건 Fabric. 어차피 Capistrano 의 deploy/rollback task 자체는 대충 버전별로 디렉토리에 소스를 받아서 체크아웃하고
심볼릭 링크를 걸고 롤백시에는 다시 이전 버전으로 링크하는 형태라 그다지 어렵지 않게 흉내낼 수 있으리라 생각했고,
이왕 파이썬을 썼으니 이것도 맞춰서 Fabric 쪽으로 굳혀갔다.

그러나 코드의 배포 이외에도 uWSGI/nginx 도 필요하고 pyenv 등도 필요하고 하여튼 프로비저닝때문에 다시 고민.
Chef/Ansiable 등을 이용한 구성부터, AMI를 아예 미리 구워버리는 것도 생각하다가 동료의 "Docker 는 어때요?" 라는 말에 급 선회.

생각해보니 이미지를 통으로 빌드하고 그걸 배포하면 모든게 해결되네? 게다가 Java 컴포넌트는 AspectJ 로 (self invocation 등의 문제 때문에)
LTW를 사용하게 구성할 생각이었는데, weaving agent 붙이는 문제도 같이 해결되니 일타쌍피.

#### Docker

처음에는 ECR 을 떠올렸으나 ​서울 리전은 지원하지 않길래 좌절.(어차피 나중엔 서울리전 올텐데 일본리전에 올리기는 죽어도 싫었던지라)
할수없이 그냥 EC2 에 올리기로 하고, 일단 Docker private registry 부터 만들었다. 이건 뭐 사실 private registry 자체도
컨테이너로 제공이 되서 S3 백엔드로 설정해주고 인증서 및 웹서버만 붙이면 끝이었고...

각각 nginx / uwsgi / fluentd(사실 이건 나중에 추가) 이미지들을 만들어서 docker-compose 로 띄우게 설정하고,
어플리케이션 이미지는 CI에서 release tag 가 붙으면 빌드해서 푸시하도록 만들었다.

EC2 인스턴스는 특성상 수도없이 꺼지고 켜지고 하니, 이걸 매번 서버 띄울때마다 손으로 할 수는 없는지라 S3 에 entrypoint 스크립트를 만들어서
ASG의 user data 에 넣어뒀다. 이 스크립트는 github repository 에 있는 인스턴스 initialization 스크립트를 실행하고​,
이 initialization 스크립트가 다시 같은 곳에 있는 각종 설정파일(nginx 설정, docker-compose.yml 등)들을 체크아웃받고 최종적으로 컨테이너들을 띄우게 했다.
굳이 이걸 S3 에 넣지 않은 이유는 시스템 설정 및 어플리케이션 설정도 VCS를 통해 관리하고 싶어서였다.

그러니까,

1. cloud-init 는 (user data로) S3 의 entrypoint 스크립트를 실행[^2]  
2. S3 의 entrypoint 스크립트는 github 의 initialization 스크립트를 실행
3. initialization 스크립트는 github 의 서버 설정 repository 를 체크아웃하고 docker-compose 를 실행

### 배포

처음에는, Fabric 으로 서버에 접속해서 위의 initialization 스크립트가 체크아웃해둔 서버 설정 repository(docker-compose.yml 이 포함되어 있는)를
업데이트(pull)해서 docker-compose.yml 에 지정된 컨테이너 버전을 업데이트하고, 이미지들을 재시작하게 했다. 물론 그 전에 한대만 먼저 올려서 모니터링하고.

그러다가 이거 매번 fabric 으로 서버마다 ssh붙어서 git pull 하고 docker-compose up -d 하는것도 뻘짓같아서, Docker REST API를 열고 docker-py 로
그냥 우르르 업데이트하게 했었는데, 이건 보안 문제도 있는데다 그렇다고 ACL걸자니 외부에서는 또 귀찮고 인증걸려면 더 귀찮아서 다시 집어치움.

근데 기존에 물리서버를 운용하던 상식으로는 이런 식으로 기존의 운영중인 서버들에 대고 롤링 업데이트를 하는게 당연한 거였는데,
가만히 생각해보니 AutoScalingGroup 설정해두고 쓰는데 EC2 인스턴스를 이렇게 운용할 이유가 없었다.

그래서 변경 :

1. release tag 를 붙여서 푸시하면 CI는 Docker 이미지를 빌드하고 private repository 에 푸시한다.
2. 서버 설정 repository 의 docker-compose.yml 에 이미지 버전을 올려서 푸시하고, ASG 인스턴스 수를 현재의 두배로 늘린다.
3. 이제 새로운 버전의 인스턴스들이 우르르 뜬다.
4. 좀 기다리다 보면 ASG 트리거가 다시 돌면서 현재 상태를 정상으로 파악할 테고 인스턴스 숫자는 원래대로 돌아온다.
Termination policy 를 OldestInstance 로 사용하고 있으므로, 자연스럽게 구버전이 돌던 인스턴스는 전부 종료된다.

그러면 배포 끝.  
이 형태로 몇달째 운영하고 있다.

### 개선해야 할 점

나름대로 깔끔하고 쉽게 배포를 하고 있지만, 그럼에도 불구하고 개선하고 싶은 점들이 있다.

1. 한두대 정도에 먼저 배포해서 테스트를 할 때는 여전히 SSH로 서버에 직접 들어가서 작업을 하고 있다.
2. 배포를 하려면 docker-compose.yml 에 이미지 버전을 올려서 커밋&푸시하고 ASG 설정을 직접 고쳐야한다.

그래서 다시 변경을 해볼까 궁리중이다.  

일단 1번을 해결하기 위한 생각:

1. release tag 가 붙어서 Docker 이미지를 빌드하는 경우, 빌드 및 푸시가 완료된 시점에 테스트 배포를 위해 EC2 인스턴스를 별도로 하나 더 띄운다.
2. 이때 ​user data 에는 기존의 entrypoint 스크립트 이외에 몇가지 환경변수를 추가로 지정한다.
3. 인스턴스 initialization 스크립트는 환경변수를 체크해서, 초기화가 완료된 이후 로드밸런서에 스스로를 추가한다.
4. 이 인스턴스는 ASG에 포함되지 않으며 production 환경 배포 전 사전 모니터링 용도로만 사용된다.

그리고 2번을 해결하기 위한 생각:

1. 서버 설정 repository 에 post-receive hook 을 추가해서 설정파일들이 수정된 경우 ASG 설정을 업데이트한다.
2. 끝

혹은 git hook 에서 바로 ASG 설정을 바꾸는거보다, 슬랙으로 메시지를 보내서 interactive button 으로 최종 승인을 하는 방안은 어떨까 싶기도 하다.
다만 이 경우 이미 컨테이너 버전의 수정사항이 푸시된 상황이기 때문에, 이 시점에서 ASG가 트리거링되어 스케일아웃이 돌면 의도치 않게 배포가 되는 문제가 있으므로,
initialization 스크립트에서 설정파일들을 바로 체크아웃받는 형태는 불가능하다. 이 경우 서버 설정 repository 는 S3로 싱크시켜두고,
initialization 스크립트는 S3에 싱크된 설정들을 받아오며, 슬랙에서 ​배포승인을 할 경우 repository 의 최신 버전을 S3로 반영한 뒤에
ASG설정을 건드리는 형태로 변경해야 할 듯.

(써두고 보니 이건 바꿀게 많네. 여기다 안써놨으면 나중에 까먹었을듯...)

혹은 더 괜찮은 아예 다른 방법은 없을까도 늘 고민이다.  

ps.  
사실 "했다" 라고 짧게 쓰긴 했지만 AWS / Docker 모두 production 레벨에서는 완전히 처음 써보는지라 숨은 삽질은 어마어마하게 많았다. 컴알못이라 눈물이...

[^1]:
    이외에도 서울리전 나오고 한동안 이런저런 라이브러리에서 서울리전을 지원하지 않던 문제가 제법 있었다.
    CloudStorage / S3 의 추상화를 위해 사용하던 libcloud 도 같은 문제가 있어서 APNE2 지원을 위해 따로 패치를 보냈으나
    업스트림 반영 및 릴리즈가 한참 늦어져서 따로 포크해서 썼었다.

[^2]:
    Launch configuration 의 user data 는 수정이 불가능한지라 인증 등이 변경될 수 있는 github 에서 뭔가를 직접 받아오기보다는
    IAM Profile 만으로 접근이 가능한 S3에 entrypoint 스크립트를 따로 올려두고 얘가 github 에 있는 실제 스크립트를 실행하도록 했다.
