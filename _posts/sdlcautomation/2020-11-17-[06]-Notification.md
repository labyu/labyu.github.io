---
title: "[SDLC Automation.06] Notification"
categories: 
  - SDLC Automation
tags:
  - argocd
  - jenkins
  - slack
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

### 실습
#### Slack App 만들기
Slack Workspace에서 App을 만들어주세요. 그러면 아래 이미지처럼 Token을 얻을 수 있습니다. 그리고 Slack에서 알림을 받을 채널에 Slack App을 초대해주세요.

![](/assets/images/sdlcautomation/06/slack-token.png)

#### Jenkins Slack 연동
Slack Token으로 Credentials를 만들어 주신 후 `[Jenkins > Jenkins 관리 > 시스템 설정]`에서 아래와 같이 설정해주세요. 이 때 만약 프론트가 깨져있어서 저장이 안되는 상황이라면 Jenkins의 모든 플러그인들을 한번 업데이트한 후 진행해주세요.

![](/assets/images/sdlcautomation/06/jenkins-config.png)

#### Jenkins Pipeline
아래와 같이 시작했을 때와 Post 부분에서 Notification을 주도록 설정했습니다. Jenkinsfiles의 원본 파일은 [여기](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/jenkins/jenkinsfiles/apjung-backend-dev-build) 에서 확인하실 수 있습니다.

```groovy
pipeline {
    stages {
        stage('Start') {
            steps {
                slackSend (channel: '#apjung-log', color: '#00FF00', message: """개발서버 CI 시작
파이프라인 : ${env.JOB_NAME} [${env.BUILD_NUMBER}]
확인 : (${env.BUILD_URL})""")
            }
        }
    }
    post {
        success {
            slackSend (channel: '#apjung-log', color: '#00FF00', message: """:white_check_mark: 성공 : 개발서버 CI
파이프라인 : ${env.JOB_NAME} [${env.BUILD_NUMBER}]
확인 : (${env.BUILD_URL})""")
        }
        failure {
            slackSend (channel: '#apjung-log', color: '#00FF00', message: """:octagonal_sign: 실패 : 개발서버 CI
파이프라인 : ${env.JOB_NAME} [${env.BUILD_NUMBER}]
확인 : (${env.BUILD_URL})""")
        }
    }
}
```

#### ArgoCD Notification 설정하기
ArgoCD에서는 위의 매뉴얼을 읽어보시면 `argocd-notifications-secret` 시크릿과 `argocd-notifications-cm` 컨피그맵을 설정해주시면 됩니다. 아래는 제 컨피그맵입니다. 시크릿에서는 위에서 발급받은 토큰을 넣어주시면 됩니다.

```yaml
apiVersion: v1
data:
  config.yaml: |
    triggers:
      - name: on-sync-succeeded
        enabled: true
        template: custom-on-sync-succeeded
      - name: on-sync-failed
        enabled: true
        template: custom-on-sync-failed
      - name: on-health-degraded
        enabled: true
        template: custom-on-health-degraded
    templates:
      - name: custom-on-sync-succeeded
        body: |
          {{if eq .context.notificationType "slack"}}:heavy_check_mark:{{end}} 애플리케이션 {{.app.metadata.name}} 서버 배포를 완료했습니다.
          시간 : {{.app.status.operationState.finishedAt}}
          확인 : {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true
      - name: custom-on-sync-failed
        body: |
          {{if eq .context.notificationType "slack"}}:octagonal_sign:{{end}} 애플리케이션 {{.app.metadata.name}} 서버 배포를 실패했습니다.
          에러 : {{.app.status.operationState.message}}
          시간 : {{.app.status.operationState.finishedAt}}
          확인 : {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true
      - name: custom-on-health-degraded
        body: |
          {{if eq .context.notificationType "slack"}}:heavy_check_mark:{{end}} 애플리케이션 {{.app.metadata.name}} 서버가 비정상적입니다.
          시간 : {{.app.status.operationState.finishedAt}}
          확인 : {{.context.argocdUrl}}/applications/{{.app.metadata.name}}?operation=true
```