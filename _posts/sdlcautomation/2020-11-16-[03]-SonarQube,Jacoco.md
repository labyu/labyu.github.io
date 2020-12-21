---
title: "[SDLC Automation.03] Sonarqube, Jacoco"
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
  og_image: "/assets/images/sdlcautomation/03/sonarqube.png"
---

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
gradle명령 이후 소나큐브 사이트로 접속해 현재 개발중인 코드가 반영되었는지 확인해주세요.