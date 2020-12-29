---
title: "[SDLC Automation.02] GitHub Webhook"
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
  og_image: "/assets/images/sdlcautomation/02/github-jenkins-integration.png"
---

#### 환경
- Kubernetes Cluster 1.19.2
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- GitHub

### 개요
이번 포스트에서는 GitHub의 Hook 이벤트를 알아보고 Jenkins와 연동해보겠습니다.

![](/assets/images/sdlcautomation/02/github-jenkins-integration.png)

#### GitHub와 연동되는 방법
먼저 Hook과 Polling 방식의 차이를 알아야 합니다. GitHub의 WebHook은 GIthub -> Jenkins로 요청을 보내는 것이고 Polling 방식은 Jenkins가 GitHub의 Repository가 업데이트 되었는지 안되었는지 주기적으로 확인하는 방식입니다.

GitHub의 PR은 opened, closed, ready_for_review 등 다양한 이벤트가 있습니다. 중요한 것은 Opened와 Synchronize 이벤트 입니다. 이 두 이벤트는 다음과 같은 상황입니다.

- (1) PR Opened : PR이 새로 생김
- (2) PR Synchronize : PR이 이미 만들어져 있는 상태에서 해당 브랜치로 Push

예를 들어 PR에 대한 브랜치 테스트를 진행하려한다면 위 두 이벤트 모두에 걸어야 정상적으로 동작합니다. Synchronize 빼먹을 경우 PR이 생성되었을 때 커밋을 제외하고는 테스트되지 않습니다. [여러 PR 이벤트](https://docs.github.com/en/free-pro-team@latest/developers/webhooks-and-events/webhook-events-and-payloads#pull_request)

아래는 Jenkins에서 GitHub Webhook과 연동할 수 있는 여러 방법입니다. 저는 Polling방식을 제외하고 나머지 두 플러그인을 필요한 곳에 사용했습니다.

- Git SCM Polling : GitHub Plugin(기본)에 포함되어있는 트리거
- GitHub Pull Request Builder : GitHub PR과 연동되어 Commit Status까지 설정할 수 있는 아주 좋은 플러그인
- Generic WebHook Trigger : WebHook의 파라미터 기반으로 다양한 이벤트를 걸 수 있는 플러그인

### 실습
#### Pull Request Builder 연동
먼저 PR WebHook부터 연동하면 위의 `GitHub Pull Request Builder`플러그인을 사용했습니다. 깃허브에서 설정은아래와같이 해주었습니다. 여기서 중요한 것이 Enable SSL Verification인데, 저는 Jenkins에 letsencrypt를 이용하여 TLS를 적용했음에도. letsencrypt 인증서는 Github에서 verify를 해주지 않습니다.

이벤트 설정은 `Let me select individual events`하신 후에 `Pull requests`설정하시면 됩니다.

![](/assets/images/sdlcautomation/02/pr-webhook.png)

Jenkins에서는 `Pull Request Builder`로 빌드 트리거를 설정해주시고 whitelist만 설정해주시면 잘 동작합니다. 마지막에 Commit Status 정의도 해주세요.

![](/assets/images/sdlcautomation/02/pr-status.png)

#### Generic WebHook Trigger 연동
`jenkinsurl.me/generic-webhook-trigger/invoke`로 URL을 설정해주시고 `Just the push event`설정해주시면 잘 동작합니다. 다만 Token과 Basic Auth 등을 설정해주시면 더 좋습니다.

Generic WebHook Trigger는 Plugin으로 선언할 수 있습니다.
```groovy
pipeline {
    triggers {
        GenericTrigger (
            genericVariables: [
                [key: 'PUSH_REF', value: '$.ref'],
            ],

            causeString: 'Triggered',
            token: 'tokenvalue',

            printContributedVariables: false,
            printPostContent: false,

            silentResponse: false,

            regexpFilterText: '$PUSH_REF',
            regexpFilterExpression: 'refs/heads/develop'
        )
    }
}
```