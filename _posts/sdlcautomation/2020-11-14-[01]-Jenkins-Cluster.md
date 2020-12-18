---
title: "[SDLC Automation.01] Jenkins Cluster"
categories: 
  - SDLC Automation
tags:
  - kubernetes
  - jenkins
last_modified_at: 2020-11-01T00:00:00+09:00
author_profile: false
sitemap :
  changefreq : daily
  priority : 1.0
---
#### 환경
- Kubernetes Cluster 1.19.2
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- Jenkins Inbount Agent Image :  [DockerHub](https://hub.docker.com/r/jenkins/inbound-agent)

#### 개요
이번 포스트에서는 Kubernetes에서 Jenkins Master - Agent로 구성된 Cluster를 구축해볼 예정입니다. 