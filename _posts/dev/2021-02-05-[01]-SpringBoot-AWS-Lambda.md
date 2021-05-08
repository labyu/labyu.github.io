---
title: "[Dev] SpringBoot AWS Lambda Integration (Feat. Asynchronous Crawling)"
categories: 
  - Dev
tags:
  - springboot
  - aws lambda
  - asynchronous
  - java
last_modified_at: 2020-11-01T00:00:00+09:00
author_profile: true
sitemap :
  changefreq : daily
  priority : 1.0
header:
  og_image: "/assets/images/dev/01/lambda.png"
---

### 개요
새로 진행하고 있는 토이프로젝트에서 크롤링 작업이 필요했습니다. 요구사항은 간단하게 다음과 같았습니다.

- 특정 엔티티 A가 있습니다.
- 특정 엔티티 A를 생성할때 프론트에서 주는 정보는 30% 정도밖에 안됩니다
- 특정 엔티티 A의 나머지 70% 정보는 크롤링을 통해 획득할 수 있습니다
- 서비스를 위해서는 특정 엔티티의 100% 정보가 필요합니다.

이왁 같은 상황일떄 서비스에서 크롤링 과정은 near real time에 가깝게 동작해야하고 빠르게 엔티티의 정보를 업데이트 해주어야 합니다. 크롤링을 무겁고 특수한 작업(Heavy Job)이라 정의하면 요구사항을 다음과 같이 정의할 수 있습니다.

- **특수한 작업이 SpringBoot의 API와 연동되어 동작해야함**

저는 이것을 해결하기 위해 `AWS Lambda`와 `@Async` 어노테이션을 통해 해결했습니다.

AWS Lambda는 왜 사용했는가
- 크롤링 작업은 SpringBoot (Java)에서 하는 것보다 Python에서 하는 것이 더욱 쉽게 개발할 수 있을 것이라 생각했습니다.
- AWS SDK를 통해 Java에서 쉽게 Invoke할 수 있습니다.

`@Async`는 왜 사용했는가
- 무거운 작업은 동기적으로 동작할 경우 사용자가 API를 요청했을 때 기다려야하는 시간이 길어질 것 같았습니다.
- AWS Lambda를 Invoke하는 것을 비동기로 처리하면 near real time에 가깝게 서비스할 수 있을 것 같았습니다.

### 실습

#### AWS Lambda 개발
AWS Lambda는 CloudFormation을 사용하는 AWS SAM(Serverless Application Model)을 이용해 개발했습니다. sam cli를 통해 쉽게 Local Invoke 및 테스팅을 진행할 수 있어서 편리하게 개발할 수 있었습니다. [Example Source](https://github.com/baruntravel/baruntravel/tree/develop/backend/lambda/baruntravel)

- SAM Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
Globals:
  Function:
    Timeout: 3

Resources:
  KakaoMapCralwer:
    Type: AWS::Serverless::Function
      FunctionName: KakaoMapCralwer
      CodeUri: KakaoMapCralwer/
      Handler: app.lambda_handler
      Runtime: python3.6

Outputs:
  KakaoMapCralwer:
    Description: "KakaoMapCralwer"
    Value: !GetAtt KakaoMapCralwer.Arn
  KakaoMapCralwerIamRole:
    Description: "Implicit IAM Role created for KakaoMapCralwer function"
    Value: !GetAtt KakaoMapCralwerRole.Arn
```

- SAM CLI

```bash
$ sam init
$ sam build && sam local invoke -e event/event.json
$ sam deploy --guided
```

#### SpringBoot Lambda Invoke [Example Source](https://github.com/baruntravel/baruntravel/tree/develop/backend/api/src/main/java/me/travelplan/service/kakaomap)
먼저 aws dependencies를 추가해줍니다.

- `build.gradle`

```groovy
  implementation platform('software.amazon.awssdk:bom:2.13.33')
  implementation 'software.amazon.awssdk:lambda'
```

이후에 AWS Client를 추가해주어야합니다. Crednetails를 설정하는 방법은 다음을 참고해주세요. [Using Credentails](https://docs.aws.amazon.com/ko_kr/sdk-for-java/latest/developer-guide/credentials.html) 저는 Envrionment를 통해 설정해주었습니다.

```bash
export AWS_ACCESS_KEY_ID=
export AWS_SECRET_ACCESS_KEY=
```

```java
@Configuration
public class AwsConfig {
    @Bean
    public LambdaClient lambdaClient() {
        return LambdaClient.builder()
                .region(Region.AP_NORTHEAST_2)
                .credentialsProvider(EnvironmentVariableCredentialsProvider.create())
                .build();
    }
}
```

이후에 다음과같이 Lambda를 Invoke하는 메소드를 작성했습니다. 동작확인 코드라서 지저분한 점은 이해해주세요 ㅎㅎ.. [AWS Lambda](https://docs.aws.amazon.com/ko_kr/sdk-for-java/latest/developer-guide/examples-lambda.html)

```java
@RequiredArgsConstructor
@Service
public class KakaoMapService {
    private final LambdaClient lambdaClient;
    private final ObjectMapper objectMapper;

    public KakaoMapPlace getKakaoMapPlace(Long placeId) {
        try {
            HashMap<String, Object> payloadMap = new HashMap<>();
            payloadMap.put("body", "{\"placeId\": " + placeId + "}");
            String payloadStr = objectMapper.writeValueAsString(payloadMap);
            SdkBytes payload = SdkBytes.fromUtf8String(payloadStr);

            InvokeRequest request = InvokeRequest.builder()
                    .functionName("KakaoMapCralwer")
                    .payload(payload)
                    .build();

            InvokeResponse res = lambdaClient.invoke(request);
            String json = res.payload().asUtf8String();

            Map<String, String> responseMap = objectMapper.readValue(json, Map.class);
            String body = objectMapper.writeValueAsString(responseMap.get("body"));
            return objectMapper.readValue(body, KakaoMapPlace.class);
        } catch (LambdaException | JsonProcessingException ex) {
            // ignore
            return null;
        }
    }
}
```

이후에 이 메소드를 호출하는 서비스에서 `@Async` 어노테이션을 붙여주었습니다. 그런데 생각보다 크롤링작업이 빨라서 일단 빼놓았습니다.
```java
    @Async
    public void updateDetail(Long id) {
        // ..
        KakaoMapPlace kakaoPlace = kakaoMapService.getKakaoMapPlace(id);
        // ..
    }
```
