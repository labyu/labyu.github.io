---

title: "DIY Kubernetes"
categories: 
  - tech
tags:
  - kubernetes
last_modified_at: 2021-10-11T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/devetc/03/HomeServer.png"
---

![](/assets/images/devetc/03/HomeServer.png)
### 개요
이번 포스트에서는 집에서 Wifi 설정과 데스크톱을 이용해 Kubernetes Cluster를 구축해볼 예정입니다. 매번 토이프로젝트를할 떄 애플리케이션 서버를 어떻게할지 고민이었는데, 또한 추가로 Redis, ElasticSearch, MySQL, Jenkins  등 여러 리소스들을 편리하게 사용하기 위해 쿠버네티스를 구축해 관리하고 배포할 예정입니다.

저는 다음과같이 설정했습니다.
- 집에 남는 데스크탑을 Cytrix Xen Hypervisor로 가상화
- VM을 5개 생성한 후 Kubernetes Cluster 구축 (Control Plain 1, Worker 3)

## 실습

### 네트워크 설정
현재 집은 LGU+ 공유기를 사용중인데 DHCP 범위가 자동으로 192.168.219.0/24로 잡혀있었습니. 공유기 설정 페이지에서도 찾아보았지만 이를 바꿀 수가 없는 듯 합니다. 일단 RFC의 private IP range를 보고 대략적으로 다음과 같이 나누어주었습니다.

**RFC 1918 Private IPv4 address**

|RFC1918 name|IP address range|Number of addresses|
|:---:|:---:|:---:|
|24-bit block|10.0.0.0 – 10.255.255.255|16777216|
|20-bit block|172.16.0.0 – 172.31.255.255|1048576|
|16-bit block|192.168.0.0 – 192.168.255.255|65536|

- 기존에 집에서 사용해야하는 IP들 (무선 연결 등) 192.168.219.1(gateway) - 192.168.219.99
- Xen Server 192.168.219.100
- VM IP 192.168.219.150 - 192.168.219.199
- DNS & Util Server 192.168.219.200
- NFS 192.168.219.210
- Kubernetes IP Range 10.0.0.0/16

### SSH
SSH 연결은 SSH Jump로 손쉽게 됩니다. MobaXTerm에서 이를 지원해주기 때문에 매우 편하게 안쪽의 VM까지 SSH 연결을 할 수 있었습니다. Mac용으로는 Terminus로 설정해놓았습니다.

### CentOS/RHEL Kubernetes Cluster 구축
XenServer의 VM으로 만든 CentOS에 쿠버네티스 클러스터를 구축했습니다. Xen Center의 Console창으로도 가능하지만 불편해서 MobaXTerm에서 SSH 연결을 구성해서 사용하는 것이 편합니다. 구축은 다음 글을 참고 했습니다 (kubeadm)

- [Link](https://gruuuuu.github.io/cloud/k8s-install/)

대략 다음과 같은 스펙으로 구축했습니다

- 설치도구 : Kubeadm
- CNI : Calico
- Ingress Controller : Nginx

노드 스펙은 다음과같이 설정했습니다

- Master Node : 2 vCPU, 6G RAM
- Worker Node : 4 vCPU, 6G RAM

### kubeadm으로 node join
원래 마스터노드 kubeadm init할때 마지막에 node join하는 명령이 나오는데 이것을 잃어버렸을때 아래와 같이 다시 할 수 있습니다.

```bash
# TOKEN 리스트 확인
$ kubeadm token list

# 없다면 TOKEN 생성
$ kubeadm token create

# HASH 확인
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# kubeadm 노드 조인
$ kubeadm join {APISERVER}:{PORT} --token {TOKEN} --discovery-token-ca-cert-hash sha256:{HASH}
```

최종적으로 다음과 같은 그림이 됩니다.
```bash
$ kubectl get node
NAME          STATUS     ROLES    AGE   VERSION
k8s-master    Ready      master   25h   v1.18.8
k8s-node-01   Ready      <none>   25h   v1.18.8
k8s-node-02   NotReady   <none>   20s   v1.18.8
k8s-node-03   NotReady   <none>   7s    v1.18.8

$ kubectl get all -n kube-system
NAME                                          READY   STATUS    RESTARTS   AGE
pod/calico-kube-controllers-75d555c48-gzcsk   1/1     Running   1          15h
pod/calico-node-9c9fk                         1/1     Running   0          14h
pod/calico-node-mcnds                         1/1     Running   1          15h
pod/coredns-66bff467f8-c76mz                  1/1     Running   1          15h
pod/coredns-66bff467f8-nrdgl                  1/1     Running   1          15h
pod/etcd-k8s-master                           1/1     Running   1          15h
pod/kube-apiserver-k8s-master                 1/1     Running   1          15h
pod/kube-controller-manager-k8s-master        1/1     Running   1          15h
pod/kube-proxy-f9tz8                          1/1     Running   0          14h
pod/kube-proxy-ls2v2                          1/1     Running   1          15h
pod/kube-scheduler-k8s-master                 1/1     Running   1          15h

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   15h

NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
daemonset.apps/calico-node   2         2         2       2            2           beta.kubernetes.io/os=linux   15h
daemonset.apps/kube-proxy    2         2         2       2            2           kubernetes.io/os=linux        15h

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/calico-kube-controllers   1/1     1            1           15h
deployment.apps/coredns                   2/2     2            2           15h

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/calico-kube-controllers-75d555c48   1         1         1       15h
replicaset.apps/coredns-66bff467f8                  2         2         2       15h

$ kubectl get all -n ingress-nginx
NAME                                           READY   STATUS      RESTARTS   AGE
pod/ingress-nginx-admission-create-qx642       0/1     Completed   0          3h26m
pod/ingress-nginx-admission-patch-b8h8m        0/1     Completed   0          3h26m
pod/ingress-nginx-controller-9d6f6c9d7-dshnk   1/1     Running     0          3h26m

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             NodePort    10.103.85.174   <none>        80:32360/TCP,443:31208/TCP   3h26m
service/ingress-nginx-controller-admission   ClusterIP   10.97.157.206   <none>        443/TCP                      3h26m

NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ingress-nginx-controller   1/1     1            1           3h26m

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/ingress-nginx-controller-9d6f6c9d7   1         1         1       3h26m

NAME                                       COMPLETIONS   DURATION   AGE
job.batch/ingress-nginx-admission-create   1/1           12s        3h26m
job.batch/ingress-nginx-admission-patch    1/1           15s        3h26m
```

### Jenkins 구축
일단 jenkins를 위해서 namespace를 만들고 Deployment, Service, PV, PVC를 설정해주었습니다.
```bash
$ k -n jenkins get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/jenkins-deployment-5754bd847d-v48z8   1/1     Running   0          113m

NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
service/jenkins-service   ClusterIP   10.99.68.224   <none>        8080/TCP,50000/TCP   116m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/jenkins-deployment   1/1     1            1           113m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/jenkins-deployment-5754bd847d   1         1         1       113m

$ k -n jenkins get pv
NAME             CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                 STORAGECLASS   REASON   AGE
jenkins-volume   5Gi        RWX            Retain           Bound    jenkins/pvc-jenkins   manual                  117m

$ k -n jenkins get pvc
NAME          STATUS   VOLUME           CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-jenkins   Bound    jenkins-volume   5Gi        RWX            manual         117m
```

또한 jenkins를 Proxy 하기 위해서 Ingress를 만들고 서비스에 붙여 놓았습니다.
```bash
$ k -n jenkins get ing
NAME              CLASS    HOSTS               ADDRESS           PORTS   AGE
jenkins-ingress   <none>   jenkins.****.**     192.168.219.151   80      117m
```

#### DNS에서 Public IP로 도메인 연결
Route53에서 Ingress Controller의 노드로의 IP로 A레코드를 성정해주어 연결했습니다.

![](/assets/images/devetc/03/tempsnip.png)

#### Jenkins 연결 성공

![](/assets/images/devetc/03/jenkins.png)

### 인그레스에 TLS 설정 (letsencrypt 사용)
default에 certbot-auto를 이용해 tls secret을 만든 후 각 네임스페이스에 복사, 업데이트해주는 스크립트를 작성하고 crontab으로 등록했습니다

```bash
#!/bin/bash

# 시크릿 스펙
SECRET=tls-secret
KEY_PATH=/etc/letsencrypt/archive/****.**/privkey1.pem
CERT_PATH=/etc/letsencrypt/archive/****.**/cert1.pem

# 업데이트해야할 네임스페이스들
ARGOCD=argocd
JENKINS=jenkins

# cert 재발금
certbot-auto renew

# 기존 시크릿 제거
kubectl delete secret $SECRET
kubectl -n $ARGOCD delete secret $SECRET
kubectl -n $JENKINS delete secret $SECRET


# default에 시크릿생성
kubectl create secret tls $SECRET --key $KEY_PATH --cert $CERT_PATH

# Argocd로 시크릿복사
kubectl get secret $SECRET --namespace=default -oyaml | grep -v namespace | kubectl apply --namespace=$ARGOCD -f -

# Jenkins로 시크릿복사
kubectl get secret $SECRET --namespace=default -oyaml | grep -v namespace | kubectl apply --namespace=$JENKINS -f -
```


### DNS서버 구축과 연결
위의 RHEL/Centos7 쿠버네티스 클러스터 구축 예제를 보면 각 노드의 아이피를 /etc/hosts에 입력해주고 있다. 그래서 dnsmasq를 이용해 DNS서버를 구축했습니다. 이외에 별도로 필요할일이 있을 때 활용할 수 있을 것 같습니다

- 1차 : DNS서버의 /etc/hosts
- 2차 : 8.8.8.8 (google dns)

```bash
### DNS 서버
$ yum install -y dnsmasq

$ vi /etc/hosts
192.168.219.150 k8s-master
192.168.219.151 k8s-node-01 jenkins.****.** argocd.****.**
192.168.219.152 k8s-node-02
192.168.219.153 k8s-node-03

$ vi /etc/dnsmasq.conf
...
# 2차 네임서버로 사용할 것 구글 dns 설정해놓음
resolv-file=/etc/resolv.dnsmasq
```

각 노드에서는 네임서버만 바꾸어주면 동작합니다.
```bash
### 클러스터 노드
$ vi /etc/resolve.conf
nameserver 192.168.219.200
```

### NFS 서버 구축과 연결
스토리지 용량이 큰 CentOS7 VM을 하나 만들어서 nfs-server를 구축하고 /storage 경로로 모두 마운트 해 놓았습니다. 권한설정은 no_all_squash, no_root_squash로 해놓았다. 이로써 쿠버네티스의 PersistenceVolume이 hostPath 또는 nfs로 사용할 수 있게 되었습니다.

### 추후 개선해야할 점
- 현재 마스터노드가 하나인데, SPOF를 제거하기 위해 이중화를 해주어야한다. (마스터노드 HA)
- 만약 서비스가 생기고 트래픽이 많아지면 Route 53의 가중치기반, 지연시간기반등을 통해 클라우드의 쿠버네티스 클러스터와 분산화를 해줄 예정이다. (멀티 클러스터)


