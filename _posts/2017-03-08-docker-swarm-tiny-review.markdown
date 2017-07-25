---
layout: post
title: "Docker swarm mode 간략 소감"
description: "겉핥기식 대충 swarm mode 간단 후기"
categories: dev
---

AWS 인프라를 셋업하면서 Docker 로 배포시스템 구성을 해두었지만, AWS 한국리전에는 ECS가 아직 없다보니
그냥 배포만 Docker 로 하고있는지라 (뭐 지금으로 그럭저럭 잘 운영하고 있지만) Container Orchestration이
필요하지 않나 싶었다.

근데 Kubernetes나 Marathon은 뭔가 해줘야하는게 많은지라 귀찮아서 생각만 하고 있었는데,
Swarm mode 가 Docker 로 아예 들어왔길래(그리고 간편하길래) 몇달째 미루다가 개발서버에 한번 적용해봤다.

겉핥기식으로 대충 돌려보고 난 간단한 소감(다른 녀석들도 제대로 써본건 아니라서 비교는 아님)

#### 장점

* 역시 따로 뭔가 더 하지 않아도 Docker 만 설치해두면 준비 끝
* 끝(...)

#### 단점

* Service discovery 를 위해 아무것도 없어도 되지만, 반대로 뭔가 좀더 복잡하게 하려면 역시 힘들다.
  ASG랑 엮어서 클러스터 구성을 자동으로 하려니, 어떻게든 되긴 하지만 이쁘게 되진 않는다.
* Constraints 를 상세하게 지정하기 어렵다. max replica per node 나 placement rule 등을
  설정해주고 싶었으나...
* 아직 개발중이라 그런지 문서화가 매우 부족하다. 사실 이건 Swarm mode 만의 문제는 아니고,
  Docker 자체가 전반적으로 좀 그런 면이 있다.[^1] 그와중에 인라인 헬프도 엉망.
* Swarm 초기버전에 비하면 Service discovery 의 안정성은 많이 좋아졌지만 아직도 성능은 미묘하다.
  실제로 속도를 측정해보면 여기서 까먹는게 제법 있다. 초창기의 Docker 같은 느낌적인 느낌...?
  standalone 으로 쓰면 host 모드로 좀 커버가 되겠지만 swarm mode 에서는 불가능.
* 그 외로, 미묘하게 기능이나 성능과 관련없이 툴 자체의 완성도가 조금 떨어진다. 예를들면 stack 으로
  좍 묶어서 배포할때 이미지를 새로 받아오는 경우 버전 태깅이 누락된다거나, 메시지가 짤려서 inspect로
  JSON 쳐다보고 있어야 한다거나 등.
* 그리고 투정 : 이왕 내장시킬꺼 ​data volume orchestration 도 같이 만들어주지...

#### 소감

개발/테스트 환경 정도에서는 그럭저럭 쓸만한듯. 프로덕션 레벨에서 제대로 쓰려면 그냥 Kubernetes 가
나을거같긴 한데, 처음에 테스트해보던 작년 가을쯤인가 시점에 비해서는 훨씬 좋아진거 보니[^2],
좀만 더 기다리자(...)

[^1]:
    예를들어 작년 여름에 변경된 ​리소스 관련 설정들은, 그 값이 뭘 의미하는지 아직도 설명이 없다(...)
    [소스](https://github.com/docker/docker/blob/v1.12.0-rc4/daemon/cluster/executor/container/container.go#L312-L335)
    를 열어보고나서야 알았다.

[^2]:
    근데 이것도 예전의 기억이 다 사라져서 "전보다는 나아졌네?" 정도의 느낌적인 느낌만 아련히;;;
    남아있다. 추억팔이도 아니고...
