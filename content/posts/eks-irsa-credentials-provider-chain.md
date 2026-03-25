---
title: "라이브러리 버전 호환 이슈로 돌아본 EKS IRSA와 AWS Credentials Provider Chain"
date: 2025-04-26
tags: ["AWS EKS", "IRSA", "Spring Boot", "Troubleshooting"]
categories: ["Troubleshooting"]
summary: "AWS EKS에 배포한 앱이 IRSA 대신 노드의 EC2 Role을 사용하던 문제를 SDK 버전 차이에서 찾아 해결한 경험"
draft: false
---

## 문제상황

AWS EKS에 배포한 앱이 배포 환경에서만 AWS Secrets Manager 접근이 불가하였습니다.

- 개발 환경: Spring boot 3.2, Kotlin 2.1
- 배포 환경: AWS EKS (IRSA를 통한 assume role 사용)
- 사용한 라이브러리: `io.awspring.cloud:spring-cloud-starter-aws-secrets-manager-config:2.4.4`

## 문제 분석 과정

### 로컬 테스트

우선 AWS Secrets Manager의 생성 여부와 오탈자를 확인하기 위해 로컬 환경에서 테스트를 진행했습니다. 개발 환경에서는 정상적으로 Secrets 값을 불러올 수 있었고, 로컬에서는 문제가 없음을 확인했습니다.

### EKS pod에서의 IRSA 사용

EKS에 배포된 Pod가 IRSA를 통해 올바르게 Role을 Assume하는지 확인했습니다.

```bash
kubectl exec -it <pod-name> -n <namespace> -- aws sts get-caller-identity
```

실행 결과, 기대한 IAM Role로 정상적으로 Assume된 것을 확인할 수 있었습니다.

```json
{
  "UserId": "AROAxxxxxxxxxxxx:sample-service-account",
  "Account": "<AWS_ACCOUNT_ID>",
  "Arn": "arn:aws:sts::<AWS_ACCOUNT_ID>:assumed-role/<IAM_ROLE_NAME>/sample-service-account"
}
```

Pod 레벨에서 IRSA가 정상적으로 작동하는 것을 확인한 후, 문제의 원인이 AWS IAM Trust Policy나 EKS 설정이 아니라 다른 부분에 있을 것으로 판단했습니다.

## 최종 해결 방법

AWS 리소스 설정과 EKS 환경 자체에는 문제가 없다는 것을 확인한 후, 애플리케이션 로그를 집중적으로 살펴보기 시작했습니다.

그 과정에서, 의도하지 않은 Assume Role이 사용되고 있다는 경고 로그를 발견했습니다!

```
2025-04-26T10:45:00.169Z  WARN 1 --- [xxx-yyy-api] [           main] i.a.c.s.AwsSecretsManagerPropertySources : Unable to load AWS secret from some-namespace/some-secrets-name. User: arn:aws:sts::123456789012:assumed-role/eks-ap-northeast-2-some-namespace-node/i-0e12345b6b789a012 is not authorized to perform: secretsmanager:GetSecretValue on resource: eks-namespace/xxx-yyy-api because no identity-based policy allows the secretsmanager:GetSecretValue action (Service: AWSSecretsManager; Status Code: 400; Error Code: AccessDeniedException; Request ID: 2613802c-661a-44b2-addd-fa5d3c2f9dc1; Proxy: null)
```

로그를 통해 애플리케이션이 **Pod의 IRSA Role** 대신 **노드의 EC2 Role**을 통해 SecretsManager에 접근하려 한다는 것을 알게 되었습니다.

Pod 자체에서는 IRSA Role을 사용하고 있었기 때문에, 문제를 바로 찾지 못하고 있던 중 팀장님께서 `build.gradle.kts`에서 Vulnerability Warning이 뜨는 라이브러리를 확인해보라고 조언해주셨습니다.

기존에 사용하던 라이브러리는 `aws-java-sdk-core:1.12.395`를 의존하고 있었고, AWS SDK for Java v1과 v2 간의 **Default Credentials Provider Chain 동작 방식**이 다르다는 점을 알게 되었습니다.

결국, SDK 버전 차이로 인해 IRSA Role보다 노드 Role이 먼저 사용되는 이슈가 발생했던 것입니다.

이를 해결하기 위해 `io.awspring.cloud:spring-cloud-aws-starter-secrets-manager`으로 교체하여 SDK 버전을 업그레이드하고 문제를 해결했습니다.

## 마무리

이번 트러블슈팅을 통해서 아래의 것들을 배울 수 있게 되었습니다:

- AWS SDK for Java의 버전에 따른 Default credentials provider chain 동작 차이
- EKS IRSA의 Role Assume 과정을 실질적으로 검증하는 방법

이번 트러블슈팅을 통해 새로운 지식들을 얻게 되었지만, 라이브러리 버전 및 호환성 검토, 빌드 경고(Warnings) 확인과 같은 기본적인 부분을 소홀히 하여 쉽게 해결할 수 있었던 문제를 어렵게 돌아가게 된 점이 아쉬웠습니다.

특히, 최근 스터디하는 개발 서적에서 중요하다고 강조하던 기본 원칙들을 스스로 간과하고 있었다는 사실이 부끄럽게 느껴졌습니다.

이번 경험을 교훈 삼아, 앞으로는 이러한 실수를 반복하지 않기 위해 문제를 더 꼼꼼히 점검하고 기본을 철저히 지키려 합니다. :)

## 참고

- [AWS SDK for Java v2: Default credentials provider chain](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/credentials-chain.html)
- [AWS SDK for Java v1: Using the Default Credentials Provider Chain](https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html#credentials-default)
