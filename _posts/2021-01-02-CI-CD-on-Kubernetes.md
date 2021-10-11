---
title: "Kubernetes에서 Jenkins와 ArgoCD로 CI/CD 구축하기"
categories: 
  - tech
tags:
  - kubernetes
  - jenkins
  - argocd
last_modified_at: 2021-10-11T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/sdlcautomation/01/jenkins-agent.png"
---

# [1] Jenkins Cluster 구축
#### 환경
- Kubernetes Cluster 1.19.2
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- Jenkins Inbount Agent Image :  [DockerHub](https://hub.docker.com/r/jenkins/inbound-agent)

### 개요
Kubernetes에서 Jenkins Master - Agent로 구성된 Cluster를 구축해볼 예정입니다. Jenkins Agent는 일반적으로 JNLP(JAVA Web Start)방식으로 노드들간 통신을 합니다. 또한 파이프라인에서 도커 컨테이너 혹은 쿠버네티스 파드를 프로비저닝하면서 에이전트로서 사용(pod Template 등)할 수 있지만 그 에이전트 스펙을 확실하게 잡기 위해서 일단 미리 프로비저닝된 쿠버네티스 파드를 통해 구성할 예정입니다.

![](/assets/images/sdlcautomation/01/jenkins-agent.png)

#### Master - Agent
Jenkins는 Application의 CI/CD 과정에서 사용함으로 다양한 빌드, 테스트 환경이 필요합니다. 여기서 환경이라는 것은 다양한 OS, 환경 변수, 설치된 모듈들 등을 의미합니다. Jenkins Master 즉 Jenkins Software를 여러개로 두는 것도 방법이지만 Jenkins는 Agent를 통해 클러스터를 구성함으로써 이를 해결할 수 있습니다.

즉 다음과 같은 특징이 있습니다.
- 다양한 환경 프로비저닝이 가능
- Jenkins Master의 부하가 줄어듬

#### Jenkins Master HA
Jenkins Cluster를 구성한 후 보면 Jenkins Master가 죽으면 결국 Cluster 전체가 죽는 것을 알 수 있습니다. SPOF를 제거하기 위해 보통은 Active - Active HA를 구성하지만 조금 조사해본 결과 Jenkins는 Stateful 한 Application이라서 Active - Active HA가 불가능합니다. Kubernetes를 사용하시는 분이라면 PV로 데이터 유지만 시켜 놓으시면 Kubelet에 의해 자동으로 재실행. 즉 Active - Standby 효과를 얻으실 수 있기 때문에 그냥 두시면 될 것 같습니다.

![](/assets/images/sdlcautomation/01/jenkins-ha.png)

---

### 실습

#### Jenkins Master 구축
Jenkins Master의 Deployment Spec은 아래와 같습니다. replicas는 1로 해주시고 저는 jdk11버전을 사용하기 때문에 저 이미지를 사용했습니다. 8버전이신 분들은 latest사용하셔도 무방합니다. `/var/jenkins_home`을 꼭 PV로 마운트 시켜놓으셔야 합니다.
```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: jenkins-master-pod
  template:
    metadata:
      labels:
        app.kubernetes.io/name: jenkins-master
        app.kubernetes.io/instance: jenkins-master-pod
    spec:
      containers:
        - name: jenkins-master-pod
          image: jenkins/jenkins:jdk11
          ports:
            - containerPort: 8080
            - containerPort: 50000
          resources:
            requests:
              memory: "1000Mi"
              cpu: "300m"
            limits:
              memory: "1000Mi"
              cpu: "300m"
          env:
            - name: JENKINS_JAVA_OPTIONS
              value: "-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Seoul"
          volumeMounts:
            - mountPath: /var/jenkins_home
              name: jenkins-master-pvc
      volumes:
        - name: jenkins-master-pvc
          persistentVolumeClaim:
            claimName: jenkins-master-pvc
```

컨테이너가 생성되면 `kubectl exec`명령을 통해 토큰을 넣고 설치해주세요. 설치가 완료된 이후에는 플러그인을 포함한 모든 업데이트를 완료해주시면 됩니다. 이후에 `Jenkins 관리 > 노드관리 > 노드 추가`에서 아래와 같이 입력해주시면 됩니다..

![](/assets/images/sdlcautomation/01/jenkins-node-manage.png)
- Name : Agent의 이름입니다. 
- Description : Agent의 설명입니다.
- of executors : Agent가 동시에 실행하는 잡의 개수입니다
- Remote root direcory : Agent의 Root 디렉토리입니다
- Labels : Agent의 레이블입니다.
- Launch method : 실행 방법, 즉 Agent에서 잡을 실행하는 방법인데, JNLP와 SSH 방식중 선택하실 수 있습니다. 지금은 JNLP방식인 `Launch agent by connecting it to the master`를 선택합니다.

성공적으로 노드를 생성하셨다면 `Jenkins 관리 > 스크립트 콘솔`에 접속하셔서 아래 스크립트를 실행합니다. 노드의 이름과 토큰을 확인할 수 있습니다.
```groovy
for (aSlave in hudson.model.Hudson.instance.slaves) 
{ println aSlave.name + "," + aSlave.getComputer().getJnlpMac() }
```

#### Jenkins Agent 구축
Agent를 구축하기 위해서는 아래 세가지가 필요합니다. Docker Hub의 Agent 이미지 정보를 보시면 환경변수로 입력하는 것을 알 수 있는데, Kubenretes Secrets을 이용하여 주입할 예정입니다.

- Jenkins Master의 Host(Jenkins Master를 k8s Service로 노출시켜 놓아주세요)
- Agent의 이름(위에서 사용한 Name 옵션)
- Agent의 토큰(위에서 확인한 토큰)

Deployment의 스펙은 아래와 같습니다. 아래 `labyu/testjenkins:test`이미지는 제가 사용할 Agent가 Docker Daemon이 필요하기 때문에 `jenkins/inbound-agent`이미지를 FROM하여 만든 이미지입니다. 보통의 상황이시라면 상단에 참조한 이미지를 사용해주세요. 또한 Secret으로 부터 환경변수를 주입받고 나머지는 직접 작성해놓은 것을 확인하실 수 있습니다.
```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/instance: jenkins-apjung-admin-cicd-agent-pod
  template:
    metadata:
      labels:
        app.kubernetes.io/name: jenkins-apjung-admin-cicd-agent-agent
        app.kubernetes.io/instance: jenkins-apjung-admin-cicd-agent-pod
    spec:
      containers:
        - name: jenkins-apjung-admin-cicd-agent-pod
          image: labyu/testjenkins:test
          ports:
            - containerPort: 8080
            - containerPort: 50000
          resources:
            requests:
              memory: "500Mi"
              cpu: "500m"
            limits:
              memory: "500Mi"
              cpu: "500m"
          env:
            - name: JENKINS_JAVA_OPTIONS
              value: "-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Seoul"
            - name: JENKINS_URL
              value: "http://jenkins-master-svc"
            - name: JENKINS_AGENT_NAME
              value: "apjung-admin-cicd-agent"
          envFrom:
            - secretRef:
                name: jenkins-apjung-admin-cicd-agent.secret
          volumeMounts:
            - mountPath: /var/run/docker.sock
              name: docker-socket
      volumes:
        - name: docker-socket
          hostPath:
            path: /var/run/docker.sock
```

#### 동작 확인
성공적으로 Agent 컨테이너를 실행시키셨다면 Jenkins 좌측 하단에 노드가 보이고 파란색으로 변한 것을 확인하실 수 있습니다.

![](/assets/images/sdlcautomation/01/jenkins-node-list.png)

파이프라인을 작성해서 실행할 차례입니다. label을 통해 원하는 노드(Master or Agent)에서 실행할 수 있습니다.
```groovy
pipeline {
    agent {
        label 'apjung-admin-cicd-agent'
    }
}
```

위 파이프라인을 실행하시면 아래처럼 동작합니다. 너무 빨리 끝나서 안보이실 수도 있으니 스테이지를 추가하셔서 확인해보시는 것이 좋을 것 같습니다

![](/assets/images/sdlcautomation/01/jenkins-node-execute.png)

# [2] GitHub Webhook 연동
GitHub의 Hook 이벤트를 알아보고 Jenkins와 연동해보겠습니다.

![](/assets/images/sdlcautomation/02/github-jenkins-integration.png)

#### 환경
- Kubernetes Cluster 1.19.2
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- GitHub

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

---

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

# [3] Sonarqube, Jacoco를 사용하여 CodeCoverage 측정

#### 환경
- Kubernetes Cluster 1.19.2
- Sonarqube Image : [DockerHub](https://hub.docker.com/_/sonarqube)

### 개요
이번 포스트에서는 SonarQube를 통해 코드 스멜, 버그 등을 정적으로 분석하고 Jacoco를 통해 코드커버리지를 분석, SonarQube와 연동해 확인할 수 있도록 해볼 예정입니다. 코드 정적 분석은 예기치 못한 버그를 유발할 수 있는 코드를 고치고 클린 코드를 유지하는데 중요한 정보를 줍니다.

![](/assets/images/sdlcautomation/03/sonarqube.png)

#### 코드 커버리지란
코드 커버리지란 테스트의 수행 결과를 수치로 나타내는 방법으로 아래와 같은 커버리지 종류가 있습니다. 일반적으로 많이 사용하는 것은 구문(Statement) 커버리지로 테스트코드들이 전체코드중에서 몇라인이 실행되었는지에 대한 수치입니다. 예를 들엉 if ~ else 구문에서 else구문이 실행되지 않는 다면 해당 라인의 코드는 실행되지 않음음으로 코드 커버리지를 떨어뜨리는 요인이 됩니다. 이런 것들을 고려하면서 테스트 코드를 작성하시면 코드커버리지를 올리실 수 있습니다.

![](/assets/images/sdlcautomation/03/codecoverage.jpg)

#### 소나큐브(SonarQube)란
코드 정적분석 툴로 다음과 같은 지표 등을 확인할 수 있습니다. 소나큐브에 해당 언어 혹은 프레임워크의 플러그인을 설치하여 사용하는 방식으로 무료와 유료로 나누어져 있습니다. CI에서 사용되며 지표에 따라 CI를 실패시킬 수 있습니다. gradle, maven 등과 연동할 수 있습니다.

- 버그 : 잘못된 코드 혹은 버그를 유발할 수 있는 코드 (NPE 등)
- 코드스멜 : 정상적으로 동작하지만 해당 언어의 코드 컨벤션에 맞지 않거나 중복되고 복잡한 코드 등
- 보안취약점 : 보안에 취약한 코드

---

### 실습

#### sonarqube 서버 설치
상단에 공유한 도커 이미지에는 소나큐브가 설치되는데 필요한 것들이 모두 들어있습니다.(DB 등) 따로 구축하여 연동할 수도 있습니다. 아래 주석된 부분은 PV 마운트로 필요한 것만 마운트해서 사용하시면 됩니다.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sonarqube-deployment
  namespace: sonarqube
  labels:
    app: sonarqube-deployment

spec:
  replicas: 1
  selector:
    matchLabels:
      app: sonarqube-pod

  template:
    metadata:
      labels:
        app: sonarqube-pod
    spec:
      containers:
        - name: sonarqube-pod
          image: sonarqube:latest
          ports:
            - containerPort: 9000
          volumeMounts:
            - mountPath: /opt/sonarqube/data
              name: sonarqube-volume-data
            #- mountPath: /opt/sonarqube/extensions
            #  name: sonarqube-volume-extensions
            #- mountPath: /opt/sonarqube/conf
            #  name: sonarqube-volume-conf
      volumes:
        - name: sonarqube-volume-data
          persistentVolumeClaim:
            claimName: pvc-sonarqube-data
        #- name: sonarqube-volume-extensions
        #  persistentVolumeClaim:
        #    claimName: pvc-sonarqube-extensions
        #- name: sonarqube-volume-conf
        #  persistentVolumeClaim:
        #    claimName: pvc-sonarqube-conf
```

#### SonarQube 프로젝트 생성 및 플러그인 설치
처음 SonarQube를 실행하신 후 New Project를 통해 프로젝트를 생성해줍니다. 이 때 프로젝트의 키를 설정하는데 이것이 프로젝트의 이름이라고 생각하시면 됩니다. 그 후 토큰을 생성하는데 꼭 기억해주셔야합니다.

![](/assets/images/sdlcautomation/03/sonarqube-token.png)


#### gradle에 sonarqube와 jacoco 추가
제 프로젝트는 멀티모듈 프로젝트로 아래는  `root`의 `build.gradle`파일입니다. 프로젝트는 아래와 같은 구성으로 되어있습니다.

- root
  - api(sub project)

실제 코드는 아래 링크에 있습니다.

- root project의 `build.gradle` : [GitHub](https://github.com/cocoding-ss/apjung-backend/blob/develop/build.gradle)
- `api` subproject의 `build.gradle` :  [GitHub](https://github.com/cocoding-ss/apjung-backend/blob/develop/api/build.gradle)

```groovy
// root의 build.gradle
plugins {
    id "org.sonarqube" version "2.7"
}

sonarqube {
    properties {
        property "sonar.sourceEncoding", "UTF-8"
    }
}

allprojects {
    group 'me.apjung.backend'
    version '1.0'
}

subprojects {
    apply plugin: 'java'

    sonarqube {
        properties {
            property "sonar.sources", "src/main/java"
            property "sonar.tests", "src/test/java"
            property "sonar.coverage.jacoco.xmlReportPaths", "build/jacoco/jacoco.xml"
        }
    }
}

project('api') {
}
```

```groovy
// api의 build.gradle
plugins {
    id "org.asciidoctor.convert" version "2.4.0"
    id "com.google.cloud.tools.jib" version "2.5.0"
    id "jacoco"
    id "idea"
    id "org.flywaydb.flyway" version "7.0.3"
}
// ...
test {
    jacoco {
        destinationFile = file("$buildDir/jacoco/jacoco.exec")
    }

    useJUnitPlatform {
        includeEngines 'junit-jupiter'
    }

    finalizedBy 'jacocoTestReport'
}
// ...
/*************************
 * jacoco 코드 커버리지 분석
 *************************/

jacoco {
    toolVersion = '0.8.5'
}

jacocoTestReport {
    reports {
        html.enabled false
        xml.enabled true
        csv.enabled false

        html.destination file("$buildDir/jacoco/jacoco.html")
        xml.destination file("$buildDir/jacoco/jacoco.xml")
    }

//    이 설정을 열어주면 Code Coverage가 일정 이상이어야 Build됨
//    finalizedBy 'jacocoTestCoverageVerification'
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            // 'element'가 없으면 프로젝트의 전체 파일을 합친 값을 기준으로 한다.
            limit {
                // 'counter'를 지정하지 않으면 default는 'INSTRUCTION'
                // 'value'를 지정하지 않으면 default는 'COVEREDRATIO'
                minimum = 0.30
            }
        }

        rule {
            // 룰을 간단히 켜고 끌 수 있다.
            enabled = true

            // 룰을 체크할 단위는 클래스 단위
            element = 'CLASS'

            // 브랜치 커버리지를 최소한 50% 만족시켜야 한다.
            limit {
                counter = 'BRANCH'
                value = 'COVEREDRATIO'
                minimum = 0.50
            }

            // 라인 커버리지를 최소한 50% 만족시켜야 한다.
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.50
            }

            // 빈 줄을 제외한 코드의 라인수를 최대 300라인으로 제한한다.
            limit {
                counter = 'LINE'
                value = 'TOTALCOUNT'
                maximum = 300
//              maximum = 8
            }

            // 커버리지 체크를 제외할 클래스들
            excludes = [
                    '*.test.*',
            ]
        }
    }
}
```

#### 테스트
아래 명령을 사용하여 test할 수 있습니다. `sonar.login`에는 위의 토큰을 넣어주시고 `sonar.host.url`에는 소나큐브 서버의 url을 입력해주시면 됩니다.
```bash
$ gradlew sonarqube -Dfile.encoding=UTF-8 test sonarqube \
  -Dsonar.projectKey=apjung:backend \
  -Dsonar.host.url=${SONAR_HOST_URL} \
  -Dsonar.login=${SONAR_AUTH_TOKEN}
```

#### Jenkins Pipeline
Jenkins에서 SonarQube Plugin을 설치하신 후 withSonarQubeEnv를 통해 환경변수로 사용하실 수 있습니다.
```groovy
stage('SonarQube Analysis') {
    steps {
        script {
            withSonarQubeEnv() {
                withCredentials([string(credentialsId: 'github-api-token', variable: 'GITHUB_TOKEN')]) {
                    sh (
                        script: """#!/bin/bash
                            ./gradlew -Pprofile=test -Dfile.encoding=UTF-8 test sonarqube --stacktrace \
                              -Dsonar.projectKey=apjung:backend \
                              -Dsonar.host.url=${SONAR_HOST_URL} \
                              -Dsonar.login=${SONAR_AUTH_TOKEN}
                        """
                    )
                }
            }
        }
    }
}
```

#### 동작 확인
gradle명령 이후 소나큐브 사이트로 접속해 현재 개발

# [4] ArgoCD 구축과 도입

#### 환경
- Kubernetes Cluster 1.19.2
- ArgoCD : [GitHub](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/argocd/install.yaml)

### 개요
이번 포스트에서는 ArgoCD를 구축해 보겠습니다. ArgoCD는 쿠버네티스를 위한 선언적인 GitOps CD툴로 컨셉이 재미있습니다.

- Why ArgoCD?

애플리케이션의 정의, 설정, 환경들은 선언적이고 버전 컨트롤 되어야합니다. 애플리케이션 배치와 라이프사이클 관리는 자동이어야하고 감사 가능하고 이해하기 쉬워야 합니다.

![](/assets/images/sdlcautomation/04/argocd_architecture.png)

ArgoCD는 GitOps Pattern을 따르고 쿠버네티스 스펙은 다음 방법으로 정의될 수 있습니다.

- kustomize
- helm
- ksonnet
- jsonnet
- plain YAML/json

정리

쉽게 말하면 ArgoCD는 GitOps Repo에 있는 정의된 Kubernetes Spec(yaml 등)과 현재 Kubernetes Cluster에 Deploy 되어있는 오브젝트들의 스펙을 Sync 시키는 툴입니다.

#### 주요 컨셉
ArgoCD의 주요 컨셉이니 읽어보시면 이해하시는데 도움이 많이 됩니다.

- **Application** A group of Kubernetes resources as defined by a manifest. This is a Custom Resource Definition (CRD).
- **Application source type** Which Tool is used to build the application.
- **Target state** The desired state of an application, as represented by files in a Git repository.
- **Live state** The live state of that application. What pods etc are deployed.
- **Sync status** Whether or not the live state matches the target state. Is the deployed application the same as Git says it should be?
- **Sync** The process of making an application move to its target state. E.g. by applying changes to a Kubernetes cluster.
- **Sync operation status** Whether or not a sync succeeded.
- **Refresh** Compare the latest code in Git with the live state. Figure out what is different.
- **Health** The health of the application, is it running correctly? Can it serve requests?
- **Tool** A tool to create manifests from a directory of files. E.g. Kustomize or Ksonnet. See Application Source Type.
- **Configuration management tool** See Tool.
- **Configuration management plugin** A custom tool.

### 실습

#### 설치
자세한 설명은 [여기](https://argoproj.github.io/argo-cd/getting_started/) 에서 확인하실 수 있습니다. 아래는 제가 한 과정입니다.

```bash
$ kubectl create namespace argocd
$ kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

위에가 ArgoCD에서 제공하는 스펙인데, 저는 다운로드받아 조금 수정해주었습니다. 제 스펙은 [여기](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/argocd/install.yaml) 에서 확인하실 수 있습니다.

ArgoCD의 데이터는 etcd클러스터에 저장되기 때문에 클러스터가 재부팅되도 남아있습니다. 즉 PV 등을 마운트할 필요가 없습니다.

ArgoCD-Server를 Service Type Load Balancer 혹은 Ingress를 통해 노출시켜주시면 웹페이지 접속이 가능합니다. 초기 계정은 admin / admin(?) ... 일 것 같습니다. GitHub Credentials를 설정해주신 후 좌측 상단의 New APP 버튼을 누르시고 앱을 등록해주세요.


![](/assets/images/sdlcautomation/04/argocd_newapp.png)

![](/assets/images/sdlcautomation/04/argocd_apps.png)

Out Of Sync 상태라면 Sync 버튼을 눌러주시면 자동으로 GitOps Repo의 스펙에 맞게 Deploy 됩니다. 여러 배포전략(Canary, Rolling)등을 사용하실 수 있고 APP DETAILS에서 Sync Policy를 설정해주실 수 있습니다.

![](/assets/images/sdlcautomation/04/argocd_syncpolicy.png)

- Automated : Out Of Sync상태라면 자동으로 Re Deploy 합니다
- Prune Resources : Resource들을 Prune 합니다
- Self Heal : 간혹 발생하는 snowflake 컨테이너나 다운된 컨테이너를 desire에 맞게 자동으로 deploy 합니다. Automated를 켜놓아도 Out Of Sync 상태일 때 한번 Sync할 뿐 반복적으로 Sync되는 것이 아닙니다.

위 설정들을 ArgoCD 매뉴얼에서 한번 읽어보시는 것을 권장드립니다. 보통은 Automated와 Self Heal을 Enable 시켜놓는 것이 좋습니다. Automated는 현재 App이 가지고있는 Digest와 GitOps Repo의 Digest를 비교하여 Sync상태를 결정하고 Deploy를 **한번**만 해줍니다.

Etc...

App 생성하실 때 Recursive 어쩌구 옵션이 있는데 그거 활성화하시면 하위디렉토리의 스펙들까지 한번에 Sync됩니다.

#### 동작 확인
Sync Policy Automated를 활성화 시켜놓으신 상태에서 Git Ops Repo의 스펙을 변경해보세요. ArgoCD가 Git Repo를 Polling하고 있다가 감지되면 다시 Deploy됩니다. 대략 이 시간은 1분인 것 같 습니다.

# [5] CI/CD 파이프라인 구축

#### 환경
- Kubernetes Cluster 1.19.2
- SpringBoot(JDK 11), JPA(ORM, Entity Validation), Flyway(DB Migration), JIB(Image Build & Push)
- ArgoCD : [GitHub](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/argocd/install.yaml)
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- Jenkins Inbount Agent Image :  [DockerHub](https://hub.docker.com/r/jenkins/inbound-agent)
- Sonarqube Image : [DockerHub](https://hub.docker.com/_/sonarqube)


### 개요
이번 포스트에서는 CICD 과정을 구축해볼 예정입니다. CI에서는 브랜치의 코드가 정상적으로 동작하는지 테스트하고 애플리케이션과 컨테이너를 빌드합니다. CD에서는 변경된 서버 스펙으로 애플리케이션 서버들을 다시 배포해줍니다.

CICD란 Continuos Integration, Continuos Delivery, Continuos Deployment의 약자로, Contiuos Delivery와 Continuos Deployment의 차이는 프로덕션에 배포할 때 Manual Approval가 있는지 없는지의 차이 같습니다. 자세한 설명은 아래링크에 잘 나와있습니다.

- [RedHat Blog](https://www.redhat.com/ko/topics/devops/what-is-ci-cd)

AWS DevOps 자격증을 공부하던 중 아래 블로그에서 영감을 받았습니다.
- [AWS DevOps Blog](https://aws.amazon.com/ko/blogs/devops/validating-aws-codecommit-pull-requests-with-aws-codebuild-and-aws-lambda/)

CI 부분에서의 브랜치테스트와 GitHub Branch Rule에 의한 Reject를 구성하여 PR을 관리하고 Jenkins에서 GitOps 레포를 자동으로 업데이트하여 ArgoCD에 의해 서버가 Deploy 되도록 구성했습니다. Prod환경에서는 Manual Deploy를 해보았습니다.

![](/assets/images/sdlcautomation/05/cicd.png)

#### 유용하게 사용한 Jenkins Plugin

- GitHub Pull Request Builder : GitHub을 이용하고 Pull Request Trigger가  필요하다면? 반드시 써야하는 갓갓 플러그인
- Generic WebHook Trigger : 웹훅을 다양한 작업으로 트리거할 수 있습니다. 웹훅과 함께 날라오는 파라미터로 임의의 변수 설정이 가능합니다.
- HttpRequest : Github API를 사용하기 위한 플러그인, 기존 Jenkins의 sh curl을 사용하니 json데이터가 이스케이프가 되는지 잘 되지 않아서 이것을 사용했습니다.
- Jenkins Blue Ocean : 여기서 Pipeline을 만들지는 않았고 UI용으로만 사용했습니다.

#### Credentials 관리
- GitHub에 대한 Access 정보는 SSH Key를 이용, Jenkins Credentials에 저장했습니다.
- GitHub Api에 대한 Access Token 정보는 Secret Text Jenkins Credentials에 저장했습니다.
- Pipeline에서 withCredentials로 활용했습니다.
- 애플리케이션에서 사용하는 Credentials는 Kubernetes Secret에서 환경변수로 주입했습니다.

### 실습
#### Pull Request Jenkins Pipeline
원본 파일은 [링크](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/jenkins/jenkinsfiles/apjung-backend-branch-build-test) 에서 확인하실 수 있습니다. 다음과 같은 순서로 진행되며 종료 후 GitHub Commit Status를 업데이트합니다. (Jenkins의 GitHub Pull Request Builder Plugin)

1. Branch Checkout
2. Build test

테스트가 종료되면 다음과 같이 Pull Request가 Merge 가능한 상태로 변경됩니다.

![](/assets/images/sdlcautomation/05/commistatusnot.png)

![](/assets/images/sdlcautomation/05/commitstatus.png)

```groovy
pipeline {
    agent {
        label 'apjung-backend-branch-test-agent'
    }
    stages {
        stage('Git checkout') {
            steps {
                git branch: "${ghprbSourceBranch}",
                    credentialsId: 'apjung-backend',
                    url: 'git@github.com:cocoding-ss/apjung-backend.git'
            }
        }
        stage('Gradle build') {
            steps {
                script {
                    sh (
                            script: '''#!/bin/bash
                                ./gradlew -Dfile.encoding=UTF-8 clean build -Pprofile=test
                            ''',
                            returnStdout: true
                    )
                }
            }
        }
    }
}
```

#### 개발서버 Jenkins Pipeline
원본파일은 [링크](https://github.com/cocoding-ss/apjung-gitops/blob/master/onprem/infra/jenkins/jenkinsfiles/apjung-backend-dev-build) 에서 확인하실 수 있습니다. 순서는 다음과 같습니다. GitOps Repo가 업데이트된 후 ArgoCD의 Automatic Deploy에 의해 서버가 재구성됩니다.

1. Develop Branch Checkout
2. DB 마이그레이션 (Flyway)
3. Build Test
4. Container Image Build & Push (JIB)
5. ArgoCD GitOps Repo Patch

```groovy
pipeline {
    agent {
        label 'apjung-backend-dev-agent'
    }
    stages {
        stage('Git checkout') {
            steps {
                git branch: "develop",
                    credentialsId: 'apjung-backend',
                    url: 'git@github.com:cocoding-ss/apjung-backend.git'
            }
        }
        stage('Dev Server Build Test') {
            steps {
                retry(2) {
                    withCredentials([string(credentialsId: 'MYSQL_DEV_PASSWORD', variable: 'MYSQL_ROOT_PASSWORD')]) {
                    sh """#!/bin/bash

                    ./gradlew -Pprofile=dev flywayClean
                    ./gradlew -Pprofile=dev flywayMigrate
                    ./gradlew -Dfile.encoding=UTF-8 -Pprofile=dev clean build
                    """
                    }
                }
            }
        }
        stage('Docker Build & Push') {
            steps {
                retry(3) {
                    withCredentials([string(credentialsId: 'dockerhub-username', variable: 'USERNAME'), string(credentialsId: 'dockerhub-password', variable: 'PASSWORD')]) {
                    sh """#!/bin/bash
                    ./gradlew -Pprofile=dev jib \
                       -Djib.to.image=labyu/apjung-backend:dev-${BUILD_NUMBER} \
                       -Djib.to.auth.username=${USERNAME} \
                       -Djib.to.auth.password='${PASSWORD}' --stacktrace
                    """
                    }
                }
            }
        }
        stage('clean up workspace') {
            steps {
                step([$class: 'WsCleanup'])
            }
        }
        stage('GitOps Checkout') {
            steps {
                git branch: "master",
                    credentialsId: 'apjung-gitops',
                    url: 'git@github.com:cocoding-ss/apjung-gitops.git'
            }
        }
        stage('GitOps Patch') {

            steps {
                sshagent(credentials : ['apjung-gitops']) {
                    sh '''#!/bin/bash
                    cd onprem/application/apjung-backend/dev
                    BEFORE=$(cat Deploy.yaml | grep 'image: labyu/apjung-backend:dev-' | sed -e 's/^ *//g' -e 's/ *$//g')
                    sed -i "s@$BEFORE@image: labyu/apjung-backend:dev-$BUILD_NUMBER@g" Deploy.yaml

                    git config user.name labyu
                    git config user.email labyu2020@gmail.com
                    git add .
                    git commit -m "Jenkins Dev Server Version Up $BUILD_NUMBER"
                    git push origin master
                    '''
                }

            }
        }
    }
}
```

![](/assets/images/sdlcautomation/05/pipeline.png)

# [6] Notification 구축

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

