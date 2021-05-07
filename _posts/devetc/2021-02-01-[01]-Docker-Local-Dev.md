---
title: "[Dev Etc] 도커 로컬 개발환경 설정"
categories: 
  - Dev Etc
tags:
  - docker
  - docker-compose
  - docker-sync
last_modified_at: 2020-11-01T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/docker.png"
---
### 개요
이번 포스트에서는 Docker를 이용해 로컬 개발환경 설정을 설정해볼 예정입니다. 개발환경을 최대한 배포환경과 비슷하게 하기 위해서 일반적으로 Virtual Box나 VMWare 등을 이용해 Linux 환경을 설정하는데 이는 다음과 같은 문제점이 있습니다.

- VMWare나 VirtualBox 이미지는 개발자간 공유하기가 어려움
- VMWare나 VirtualBox의 공유 디렉토리 (마운트 디렉토리)를 사용할 경우 성능이 매우 느림 (호스트 OS와 가상머신간의 File Systsem의 차이에 의해서 지연 발생)

첫번째 공유의 어려움은 도커를 이용하게 되면 사내에 Docker Registry를 배포하여 간단하게 개발자들끼리 공유할 수 있습니다. 개발자의 PC에 도커만 설치한 후 docker-compose나 docker-sync파일을 소스코드 저장소 (Git 등)에서 애플리케이션 소스코드와 함께 버전관리한다면 개발자는 소스코드를 내려받은 후 `docker-compose up` 또는 `docker-sync-stack start` 명령을 통해 손쉽게 로컬개발환경을 구축할 수 있습니다.

두번째 이유의 경우에는 Docker에서도 Volume Mount를 사용할 경우 지연이 발생합니다. Docker for Mac의 경우에는 Cached Volume이 있다곤 하지만 파일이 많아질 경우 이 또한 좋은 성능을 보장하지 않는 다는 것을 확인했습니다.

그래서 이번에는 `docker-sync`라고 하는 오픈소스를 이용해 도커 로컬개발환경을 구성해보았습니다.


#### Docker Sync

- [docker sync](https://docker-sync.readthedocs.io/en/latest/index.html#)

docker sync는 File Synchronizer가 설치된 사이드카 컨테이너를 띄워 호스트의 디렉토리와 File을 Sync하고 애플리케이션 컨테이너에 볼륨마운트를 하여 호스트 PC의 코드가 애플리케이션 컨테이너 안으로 동기화될 수 있도록 도와주는 도커 서드파티 소프트웨어입니다.

#### Installation
ruby의 gem을 이용하여 설치합니다 Optional로 unison file synchronizer를 사용할 수 있습니다. 윈도우는 WSL2에서 진행해주시면 됩니다.
```bash
gem install docker-sync
```

### 실습
애플리케이션 컨테이너 이미지를 빌드한 후 소스디렉토리에 아래 파일들을 생성합니다. 제가 사용한 컨테이너는 PHP 애플리케이션 컨테이너입니다.

- `docker-compose.yml`
- `docker-compose-dev.yml`
- `docker-sync.yml`

`docker-compose.yml`과 `docker-compose-dev.yml`은 편의상 똑같은 내용으로 만들어주시면 됩니다. 만약 DB나 Redis등 더 추가하실 것이 있으면 여기서 추가해주시면 됩니다.

#### docker-sync.yml
```yaml
version: "2"
options:
  verbose: true
  max_attempt: 5
syncs:
  code-container:
    src: '{소스코드가 있는 디렉토리}'

    cli_mode: 'auto'
    sync_prefer: 'default'
    sync_excludes: ['.git', '.idea']
    watch_in_container: true
```

#### docker-compose.yml
```yaml
version: "3.1"
services:
  app:
    image: {애플리케이션 컨테이너 이미지}
    privileged: true
    container_name: app-container
    volumes:
      - code-container:/home/app/source
    ports:
      - "80:80"
      - "443:443"

volumes:
  code-container:
    external: true
```

명령어
```bash
docker-sync-stack start
```