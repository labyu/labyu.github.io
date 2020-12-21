---
title: "[SDLC Automation.06] Notification"
categories: 
  - SDLC Automation
tags:
  - kubernetes
  - jenkins
  - GitHub
last_modified_at: 2020-11-01T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/sdlcautomation/06/notification.png"
---
#### 환경
- Kubernetes Cluster 1.19.2
- ArgoCD : [GitHub](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/argocd/install.yaml)
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- Slack

### 개요
이번 포스트에서는 Jenkins와 ArgoCD에서 Slack으로 알림을 보내는 작업을 구축해볼 예정입니다. Jenkins는 빌드 시작, 성공/실패여부와 ArgoCD에서는 Health 상태, Sync(Deploy) 성공/실패 여부에 대해서 알림을 줄 예정입니다.

![](/assets/images/sdlcautomation/06/notification.png)

Jenkins에서는 Slack Notifcation Plugin을 사용했습니다.
- Jenkins Slack Notifiaction Plugin : [Link](https://plugins.jenkins.io/slack/)

ArgoCD에서는 ArgoCD Notifcations를 사용했습니다. Argo Kube Ntoifier, Kube Watch 등 다양한 방법으로도 Notifcation 설정이 가능합니다.
- ArgoCD Notifications : [Link](https://argoproj-labs.github.io/argocd-notifications/)

