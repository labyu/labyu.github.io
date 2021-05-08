---
title: "[Solved] Java SecureRandom issue"
categories: 
  - Solved
tags:
  - java
  - blocking
last_modified_at: 2020-11-01T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/solved/01/jenkinsslow.png"
---

### 개요
잘돌아가던 Jenkins의 CI (gradle build) 과정이 엄청나게 느려지는 현상이 갑자기 발생했습니다. 기존에는 5분이내로 끝나던 작업이 갑자기 6~8시간 이상 걸리게 되었습니다. 물론 MAC에서는 정상적으로 build 시간이 5분 이내로 나왔습니다. 😢

![](/assets/images/solved/01/jenkinsslow.png)

### 해결

Gradle Task에 debug 옵션을 주고 실행해보니 다음과 같은 로그가 나왔습니다.

```bash
00:55:42.265 [DEBUG] [org.gradle.cache.internal.DefaultFileLockManager] Lock acquired on daemon addresses registry.
00:55:42.266 [DEBUG] [org.gradle.cache.internal.DefaultFileLockManager] Releasing lock on daemon addresses registry.
00:55:42.266 [DEBUG] [org.gradle.cache.internal.DefaultFileLockManager] Waiting to acquire shared lock on daemon
```

jstack으로 Gradle Task의 덤프를 떠보니 아래와 같이 나왔습니다. 해당 ci과정이 Kubernetes Cluset의 Jenkins Inboud Agent 컨테이너에서 동작하고 있었기 때문에 부담없이 jstack 등을 설치했습니다.
```bash
"Test worker" #12 prio=5 os_prio=0 cpu=6944.52ms elapsed=525.31s tid=0x00007ff530b52800 nid=0x7d0 runnable  [0x00007ff509cf5000]
   java.lang.Thread.State: RUNNABLE
        at java.io.FileInputStream.readBytes(java.base@11.0.9/Native Method) <- 문제가 되는 부분
        at java.io.FileInputStream.read(java.base@11.0.9/FileInputStream.java:279)
        at java.io.FilterInputStream.read(java.base@11.0.9/FilterInputStream.java:133)
        at sun.security.provider.NativePRNG$RandomIO.readFully(java.base@11.0.9/NativePRNG.java:424)
        at sun.security.provider.NativePRNG$RandomIO.ensureBufferValid(java.base@11.0.9/NativePRNG.java:526)
        at sun.security.provider.NativePRNG$RandomIO.implNextBytes(java.base@11.0.9/NativePRNG.java:545)
        - locked <0x00000000e0923b38> (a java.lang.Object)
        at sun.security.provider.NativePRNG$Blocking.engineNextBytes(java.base@11.0.9/NativePRNG.java:268)
        at java.security.SecureRandom.nextBytes(java.base@11.0.9/SecureRandom.java:751)
        at java.security.SecureRandom.next(java.base@11.0.9/SecureRandom.java:808)
        at java.util.Random.nextInt(java.base@11.0.9/Random.java:390)
        at java.util.Random.internalNextInt(java.base@11.0.9/Random.java:279)
        at java.util.Random$RandomIntsSpliterator.tryAdvance(java.base@11.0.9/Random.java:1029)
        at java.util.stream.IntPipeline.forEachWithCancel(java.base@11.0.9/IntPipeline.java:163)
        at java.util.stream.AbstractPipeline.copyIntoWithCancel(java.base@11.0.9/AbstractPipeline.java:502)
        at java.util.stream.AbstractPipeline.copyInto(java.base@11.0.9/AbstractPipeline.java:488)
        at java.util.stream.AbstractPipeline.wrapAndCopyInto(java.base@11.0.9/AbstractPipeline.java:474)
        at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(java.base@11.0.9/ReduceOps.java:913)
        at java.util.stream.AbstractPipeline.evaluate(java.base@11.0.9/AbstractPipeline.java:234)
        at java.util.stream.IntPipeline.collect(java.base@11.0.9/IntPipeline.java:508)
        at me.apjung.backend.component.RandomStringBuilder.RandomStringBuilder.generateAlphaNumeric(RandomStringBuilder.java:17)
```

위의 덤프를 보면 SecureRandom객체의 작업에서 진행이 안되는 것을 알 수 있는데 문제가 된 코드는 아래와 같았습니다.
```java
Random random = SecureRandom.getInstanceStrong();
            return random.ints(leftLimit, rightLimit + 1)
                    .filter(i -> (i <= 57 || 65 <= i) && (i <= 90 || 97 <= i))
                    .limit(length)
                    .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
                    .toString();
```

Linux의 /dev/random을 사용할 때의 파일 블로킹 이슈였습니다. 꽤 유명한 이슈였던 것 같네요 .. lambda와 블로킹의 괴랄한 조합이었군요 ㅎㅎ..

- [link](https://inspirit941.tistory.com/entry/Tomcat-%EC%84%9C%EB%B2%84-%EA%B5%AC%EB%8F%99%EC%8B%9C-Creation-of-SecureRandom-instance-for-session-ID-generation-Warning-%ED%95%B4%EA%B2%B0)
- [link](https://gampol.tistory.com/entry/Tomcat-%EA%B5%AC%EB%8F%99-%EC%8B%9C-devurandom-%EB%B8%94%EB%A1%9C%ED%82%B9-%EC%9D%B4%EC%8A%88%EC%A7%80%EC%97%B0%EC%8B%9C%EC%9E%91-%EB%AC%B8%EC%A0%9C)

로컬 환경(맥북)에서 빌드할 때에는 전혀 이런 현상이 나타나지 않아 서버의 문제라고 생각하여 Kubernetes Cluster와 XenServer VM 퍼포먼스 최적화, Gradle 프로퍼티 설정 등 다양하게 접근해보았는데 설마 코드의 문제라고는 생각하지 못했습니다. 몇일동안 고생했습니다 ㅎㅎ. ㅠ

해결한 코드는 아래와 같습니다. (SecureRandom > SplittableRandom 객체로 교체) 빌드시간이 1분 30초가 되었다..!
```java
SplittableRandom random = new SplittableRandom();
        return random.ints(leftLimit, rightLimit + 1)
                .filter(i -> (i <= 57 || 65 <= i) && (i <= 90 || 97 <= i))
                .limit(length)
                .collect(StringBuilder::new, StringBuilder::appendCodePoint, StringBuilder::append)
                .toString();
```