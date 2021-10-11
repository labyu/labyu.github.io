---
title: "MSA 환경에서 gRPC 통신 구축하기"
categories:
  - tech
tags:
  - grpc
  - msa
last_modified_at: 2021-10-11T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/devetc/03/HomeServer.png"
---

### 개요
이번 포스트에서는 MSA환경에서 gRPC를 도입한 사례를 소개해드리려고 합니다. 먼저 gRPC는 프로토콜 버퍼를 사용하는 통신 플랫폼으로 다양한 언어를 지원합니다.

# 직렬화와 프로토콜 버퍼
기존 HTTP/1.1은 Body가 Text였습니다. 그리고 대부분의 Rest API 시스템에서는 JSON을 직렬화 프로토콜로 사용했습니다. 이 때 읽기스키마(Read-Scheme)와 쓰기쓰키마(Write-Scheme)의 일치를 위해 OpenAPI와 같은 모듈도 등장했습니다.

JSON의 읽기/쓰기 스키마의 일치는 다음과 같이 이루어집니다. 만약 서버의 응답이 다음과 같다면 읽는 쪽에서 "foo"라는 키를 가진 DataType에 매핑시킵니다.
```json
{
  "foo": "bar"
}
```

즉 이는 통신 데이터에 Key가 String으로 모두 담긴다는 것인데. 이 Key에 의해 데이터는 커지고 통신에는 오버헤드가 발생합니다.

하지만 프로토콜버퍼는 Field Tag라는 값을 이용하여 읽기스키마와 쓰기스키마를 일치시킵니다.

![](/assets/images/grpc-on-msa/protobuf.png)

proto는 다음과 같이 선언되며 모든 값에 Field Number가 붙고 쓰기스키마에서 값에 이 Field Number(Field Tag)를 붙여 전송합니다. 읽는 쪽에서는 해당 Field Number를 통해 읽기 스키마에 매핑시켜 읽습니다.

```proto
message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}
```

이는 통신 데이터에서 Key라는 String을 뻄으로서 더 작은 데이터를 통해 통신할 수 있게 만들어주었습니다. 또한 OpenAPI처럼 proto에서는 codegen을 다양한 언어에서 지원합니다

- python
- kotlin
- java
- go
- etc..

### 프로토콜 버퍼와 상위/하위 호환성
OpenAPI와 Json의 경우 Key가 없는 경우에 매핑에서 무시되기 때문에 호환성을 지킬 수 있었습니다. 프로토콜 버퍼에서도 default 설정이 optional이기 때문에 Field Number가 바뀌지 않는 한 하위/상위 호환성을 유지할 수 있습니다.

또한 repeated 라는 개념이 있어 프로토콜 버퍼는 Array, List와도 상위/하위호환성을 지킬 수 있습니다. 아래의 경우 쓰기스키마에서 foo를 배열로 던져주어도 읽는 쪽에서는 [0] 인덱스만 가져와 매핑합니다.
```proto
읽기 스키마
message SomeMessage {
  string foo = 1;
}

쓰기 스키마
message SomeMessage {
  repeated string foo = 1;
}
```

---

# gRPC 도입

### gRPC 도입에 앞선 고민들
gRPC는 OpenAPI처럼 proto라는 IDL을 공유해야하기 때문에 MSA로 이루어진 환경, 이종 언어와 프레임워크에서 같은 IDL을 공유해야 했습니다. 이를 위해 다음과 같은 의문이 생겼습니다.

#### IDL(proto)과 Generated된 Code는 어떻게 관리되어야 하는가?
- MSA로 의해 분리된 Source Repository마다 proto 파일을 유지해야하는가?
- proto 파일을 Build (Code Generate)하는 역할은 각 레포지토리에서 담당해야하는가?

#### 다양한 솔루션 고민들
(1) proto파일을 사용하는 repo마다 복사하여 사용하자
- 각 서비스 Repo에 proto파일이 모두 별도로 존재하게됨
- 필요한 서비스에서 직접 proto파일을 빌드하고 사용, Sync는 개발자가 직접 맞추어주어야함
- 셋업 비용 없음

(2) 사설 pypi repository와 maven repository를 구축하고 이용하자
- proto 파일 repo 분리
- proto 파일이 업데이트 될때마다 protoc으로 각 언어에 대한 패키지를 만들고 사설 pypi repository, maven repository에 라이브러리 형태로 배포함
- rpc를 사용하는 Repo에서는 dependency를 추가하여 사용함
- PyPi Repository, maven Repository 각각 구축하거나.. Nexus 같은 솔루션 사용해야함 (비용)
  이게 PyPI Repository의 경우에는 Docker Image로도 제공되는데 maven repository는 따로 구축하면 어떻게해야할지..

(3) git submodule을 이용하자
- proto 파일 repo 분리
- git submodule 명령을 이용하여 사용하는 Repo에서 해당 코드를 끌어다 가져오기

(4) proto 파일을 받아서 각각 소스에서 빌드해서 사용하기 vs 빌드된 소스만 받아서 그대로 사용하기
- proto 파일만 받아 각각 소스에서 빌드해서 사용할 경우 protoc 버전에 따라 차이가 발생할 수 있어서 generated된 코드를 받아 사용하는 것이 안전할 것이라고 판단함
- 각 언어에 맞는 protoc plugin을 받아서 idl 레포에서 한번에 generate 시켜 사용하자

#### 결정된 솔루션
이종 언어, 프레임워크의 MSA 환경과 지속가능성을 고려했을 때 결정된 솔루션은 다음과 같습니다.
- IDL Repo를 별도로 분리하여 proto를 정의한다
- IDL Codegen 역할은 IDL Repo만 가진다 각 레포에서는 Build 금지한다
- git submodule을 이용하여 각 서비스별 레포에 Generated Code를 가져와 사용한다

만약 서비스 MSA를 이루고 있는 언어와 프레임워크가 단일화 되어있다면 사설 Depenency Repository를 이용하여 Generated Code를 배포했을 것 같습니다.


## IDL Repo
IDL Repo는 다음과 같이 구성했습니다. 각 codegen을 위해 사용한 플러그인은 다음과 같습니다.
- [java](https://github.com/grpc/grpc-java)
- [kotlin](https://github.com/grpc/grpc-kotlin)
- [python](https://github.com/dropbox/mypy-protobuf)

```
- gen : Generated된 Code가 저장되어 있음
  - python
  - java
  - kotlin
- src : proto 파일들이 저장되어 있음
- scripts : Dockerfile과 docker-compose 파일들이 저장되어 있음
- build.sh : Code Generate Script // sh build.sh
```

- 우선 모든 개발자가 IDL을 작성하고 빌드할 수 있도록 모든 codegen 과정을 Docker안에서 이루어지도록 했습니다. 사용자는 clone repo한 후 `sh build.sh`만 실행하면 proto를 build할 수 있습니다. 모든 Docker와 관련된 파일은 scripts안에 정의했습니다
- 우선적으로 사용되는 서비스들인 python, kotlin, java를 위해 각각 protobuf과 grpc 플러그인을 통해 codegen 했습니다
- Github Action을 이용해 PR에서 proto 변경사항이 있다면 codegen 되었는지 검증하는 CI를 추가했습니다. 개발자의 실수로 인해 proto만 변경하고 codegen을 하지 않는 불상사를 막기 위함입니다.

## git submodule
- 각 레포에서는 git submodule을 이용하여 IDL Repo를 서비스의 자식Repo로 추가한 후 사용하도록 했습니다. git submodule은 submodule repo의 commit hash로 버전이 유지되기 때문에 개발자들이 일일이 버전업을 해주어야 하는 작은 오버헤드가 있습니다.
- CI/CD 과정에서 git submodule update를 위해 Repository SSH Key나 Personal Access Token 등을 통한 추가 작업이 필요합니다. 이부분이 꽤 큰 고생일 수 있습니다.

### Generated Code 사용하기
Generated Code는 git submodule로 받은 IDL Repo의 gen 폴더 안에 있습니다. 각각 프로젝트마다 다음과 같이 gen밑에 있는 프로젝트를 사용할 수 있습니다.

Java Project의 경우 proto파일의 package에 의존하고 python의 경우 gen 디렉토리의 디렉토리를 package에 맞추어주는 과정이 필요합니다. -> 이는 src 디렉토리를 package와 똑같이 설정해주면 됩니다.

#### gradle project(java or kotlin)의 경우
서버의 경우 [spring-boot-grpc-starter](https://github.com/LogNet/grpc-spring-boot-starter) 를 사용했습니다
- build.grade의 SourceSet을 IDL Repo의 java or kotlin 디렉토리를 지정한다
- gRPCServiceImpleBase를 Implementation하고 `@GrpcService` 를 붙여준다

#### python의 경우
- pyproject.toml로 idl 설정해주고 setup.py 만든다
- from ~ import로 가져와서 사용한다

## gRPC 서버 구축
gRPC 서버의 경우 기존 HTTP 서버와 거의 동일하며 만약 HTTP와 gRPC를 같이 서빙하는 서버의 경우 포트를 두개 띄워주면 됩니다. HTTP <> gRPC간 변환을 해주는 Gateway도 존재하는데, MSA 환경에서 서비스간 통신이라면 이것은 의미가 없다고 판단하여 도입하지 않았습니다.

### gRPC의 Healthcheck
gRPC는 기본적으로 HealthCheck용 서비스를 제공합니다. 다만 별도로 사용하고 싶은 경우에는 Heatlcheck Service를 직접 만들어주면 됩니다. 이 때 parameter와 return값이 없다면 proto의 Well-Known 타입인 Empty를 사용하면 됩니다.

```proto
service HealthCheckService {
  rpc check(google.protobuf.Empty) returns (google.protobuf.Empty);
}
```

gRPC의 Status Code는 HTTP와 매우 다르기 때문에 AWS나 k8s에서 gRPC용 Healthcheck Response Stauts Code를 입력할때는 확인이 필요합니다.

[gRPC Status Code](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)

url은 `packages.HealthCheckService/check` 과 같은 형식이 됩니다

# 결론
gRPC는 IDL 기반 통신이어서 OpenAPI 처럼 codegen의 이점을 가지고 있으면서 빠릅니다. MSA에서 서비스간 통신이 필요하다면 도입을 고려해보면 좋을 것 같습니다.


