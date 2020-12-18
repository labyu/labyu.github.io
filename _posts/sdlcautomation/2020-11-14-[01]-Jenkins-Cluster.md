---
title: "[SDLC Automation.01] Jenkins Cluster"
categories: 
  - SDLC Automation
tags:
  - kubernetes
  - jenkins
last_modified_at: 2020-11-01T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
---
#### 환경
- Kubernetes Cluster 1.19.2
- Jenkins Master Image : [DockerHub](https://hub.docker.com/r/jenkins/jenkins)
- Jenkins Inbount Agent Image :  [DockerHub](https://hub.docker.com/r/jenkins/inbound-agent)

### 개요
이번 포스트에서는 Kubernetes에서 Jenkins Master - Agent로 구성된 Cluster를 구축해볼 예정입니다. Jenkins Agent는 일반적으로 JNLP(JAVA Web Start)방식으로 노드들간 통신을 합니다. 또한 파이프라인에서 도커 컨테이너 혹은 쿠버네티스 파드를 프로비저닝하면서 에이전트로서 사용(pod Template 등)할 수 있지만 그 에이전트 스펙을 확실하게 잡기 위해서 일단 미리 프로비저닝된 쿠버네티스 파드를 통해 구성할 예정입니다.

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