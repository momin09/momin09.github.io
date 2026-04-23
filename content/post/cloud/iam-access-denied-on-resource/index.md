---
title: "`User: [ARN] is not authorized to perform ... on resource ...` 에러 정리"
slug: "iam-access-denied-on-resource"
date: 2023-06-18T10:00:00+09:00
draft: false
description: "AWS에서 `User: [ARN] is not authorized to perform ... on resource ...` 형태의 AccessDenied 에러가 나올 때 어떻게 읽고 어떤 순서로 확인하면 되는지 정리합니다."
tags: ["AWS", "IAM", "AccessDenied", "Policy", "Security"]
---

## 배경

AWS를 쓰다 보면 아래와 같은 에러를 자주 보게 된다.

```text
User: arn:aws:iam::123456789012:user/example-user is not authorized to perform: s3:GetObject on resource: arn:aws:s3:::example-bucket/path/file.txt
```

처음 보면 그냥 "권한이 없구나" 정도로 지나가기 쉽지만, 이 메시지에는 실제로 꽤 많은 힌트가 들어 있다.

특히 뒤쪽에 `on resource ...`가 붙어 있으면, **어떤 리소스에 어떤 액션이 막혔는지**까지 같이 알려주기 때문에 원인을 좁히기가 훨씬 쉬워진다.

이 글은 이 에러를 볼 때 무엇을 읽어야 하고, 어떤 순서로 확인하면 좋은지 기준을 정리한다.

## 환경

이 글은 AWS IAM 기본 개념을 이미 조금 알고 있다는 전제로 쓴다.

주로 아래 상황을 염두에 둔다.

- 콘솔 또는 CLI에서 특정 API 호출이 실패하는 경우
- IAM User 또는 IAM Role이 특정 리소스 접근에 실패하는 경우
- `AccessDenied`, `not authorized to perform`, `is not authorized to perform` 메시지를 만난 경우

## 핵심 내용

### 1. 에러 메시지는 세 덩어리로 읽으면 된다

예를 들어 아래 에러가 있다고 하자.

```text
User: arn:aws:sts::123456789012:assumed-role/app-role/session-name is not authorized to perform: secretsmanager:GetSecretValue on resource: arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/db-password
```

이 메시지는 보통 아래 세 부분으로 나눠 읽으면 된다.

- `누가`
  - `User: arn:aws:sts::...:assumed-role/...`
- `무엇을 하려다가`
  - `secretsmanager:GetSecretValue`
- `어느 리소스에서 막혔는지`
  - `arn:aws:secretsmanager:...:secret:prod/db-password`

즉, 이 에러는 단순히 "권한이 없다"가 아니라:

> 이 주체가, 이 액션을, 이 리소스에 대해 수행할 권한이 없다는 뜻이다

### 2. `resource`가 보인다는 건 리소스 범위까지 맞춰서 봐야 한다는 뜻이다

`resource`가 명시된 에러는 아래 같은 경우가 많다.

- 액션 자체는 맞지만 `Resource` 범위가 다름
- 특정 리소스 ARN을 정책이 포함하지 않음
- 와일드카드 범위가 생각보다 좁음
- 버킷 ARN과 오브젝트 ARN을 혼동함

예를 들어 S3에서는 아래 둘이 다르다.

- `arn:aws:s3:::example-bucket`
- `arn:aws:s3:::example-bucket/*`

버킷 목록 조회와 오브젝트 읽기는 필요한 리소스 ARN이 서로 다를 수 있다.  
그래서 액션은 맞게 적었는데도 `resource` 범위를 잘못 써서 막히는 경우가 자주 나온다.

### 3. 가장 먼저 보는 건 identity-based policy다

기본적으로 먼저 봐야 하는 것은 해당 User 또는 Role에 붙은 IAM 정책이다.

확인 포인트는 아래다.

- `Action`에 필요한 API가 들어 있는가
- `Resource`에 실제 대상 ARN이 포함되는가
- `Effect`가 `Allow`인가

예를 들어 아래 정책은 특정 시크릿 읽기를 허용한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/db-password*"
    }
  ]
}
```

여기서 `Resource`가 다른 시크릿 ARN을 가리키거나, suffix 패턴이 맞지 않으면 같은 액션이어도 거부될 수 있다.

### 4. Allow가 있어도 다른 정책이 막을 수 있다

IAM에서는 단순히 Allow 하나 있다고 끝나지 않는다.

아래 요소들이 중간에서 막을 수 있다.

- 명시적 `Deny`
- Resource-based policy
- Permission boundary
- Session policy
- Organizations SCP

즉, "정책에 Allow 넣었는데도 안 돼요"라는 상황은 꽤 흔하다.

이럴 때는 단순히 정책 한 장만 보지 말고, **최종 권한 평가에 영향을 주는 다른 정책도 같이** 봐야 한다.

### 5. resource-based policy가 있는 서비스는 반드시 같이 본다

일부 서비스는 리소스 쪽 정책도 함께 본다.

대표적으로 아래가 자주 나온다.

- S3 bucket policy
- KMS key policy
- SQS queue policy
- SNS topic policy
- Secrets Manager resource policy

예를 들어 KMS는 IAM 정책에서 Allow가 있어도, key policy 쪽이 맞지 않으면 접근이 안 될 수 있다.

즉:

- IAM policy는 허용
- 하지만 resource policy가 허용하지 않음
- 결과는 AccessDenied

이 흐름이 충분히 가능하다.

### 6. AssumeRole 환경이면 실제 호출 주체를 정확히 봐야 한다

에러에 `arn:aws:sts::...:assumed-role/...`가 보이면, 실제 호출 주체는 원본 사용자보다 **현재 AssumeRole로 받은 세션**이다.

이 경우 확인해야 할 것은 아래다.

- 누가 어떤 Role을 assume 했는지
- 그 Role에 필요한 권한이 있는지
- Trust relationship이 맞는지
- 세션 정책이 추가로 제한하고 있지 않은지

즉, 원래 IAM User 정책만 보고 있으면 놓칠 수 있다.

### 7. 흔한 원인은 ARN 오타보다 "범위 불일치"다

실무에서 자주 보는 패턴은 이런 쪽이다.

- 리전이 다름
- 계정 ID가 다름
- 리소스 이름 접두사만 맞고 실제 ARN은 다름
- 버킷 ARN만 넣고 object ARN은 빠짐
- secret 이름 뒤 suffix를 고려하지 않음
- KMS key ARN 대신 alias ARN만 보고 있음

그래서 에러 메시지에 찍힌 `resource ARN`을 그대로 복사해서 정책 `Resource`와 나란히 비교해 보는 것이 가장 빠르다.

## 확인 순서

### 1. 에러 메시지에서 세 가지를 먼저 뽑는다

- 주체 ARN
- 액션
- 리소스 ARN

이 세 줄만 따로 적어도 분석 속도가 빨라진다.

### 2. 해당 주체에 연결된 IAM 정책을 본다

- inline policy
- managed policy
- role policy

여기서 `Action`, `Resource`, `Effect`를 확인한다.

### 3. 명시적 Deny가 있는지 본다

Allow보다 Deny가 우선한다.

- IAM policy 내 Deny
- permission boundary 제한
- session policy 제한
- SCP 제한

### 4. resource-based policy가 있는 서비스인지 확인한다

특히 아래는 자주 같이 봐야 한다.

- S3
- KMS
- SQS
- SNS
- Secrets Manager

### 5. AssumeRole인지 확인한다

에러 주체가 `assumed-role`이면 현재 세션 기준으로 봐야 한다.

### 6. Policy Simulator나 CloudTrail로 최종 확인한다

애매할 때는 아래가 도움이 된다.

- IAM Policy Simulator
- CloudTrail event history

CloudTrail에서는 누가 어떤 API를 어떤 리소스에 대해 호출했고 어떤 에러가 났는지 교차 확인할 수 있다.

## 자주 헷갈리는 예시

### S3

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::example-bucket"
}
```

이 정책은 `GetObject` 기준으로는 보통 부족하다.  
오브젝트 접근이면 아래처럼 object ARN 범위가 필요하다.

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::example-bucket/*"
}
```

### Secrets Manager

시크릿 이름 뒤에 랜덤 suffix가 붙는 경우가 있어서, 콘솔에 보이는 이름만 보고 정확히 일치한다고 생각하면 틀릴 수 있다.

그래서 아래처럼 suffix를 감안한 패턴이 필요할 수 있다.

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/db-password*"
}
```

### KMS

KMS는 IAM 정책뿐 아니라 key policy도 같이 봐야 한다.  
특히 다른 서비스가 내부적으로 KMS를 호출하는 경우, 생각보다 KMS 쪽에서 막히는 경우가 많다.

## 요약

`User: [ARN] is not authorized to perform ... on resource ...` 에러는 그냥 "권한 없음" 정도로 넘기기보다 아래 순서로 읽는 것이 좋다.

1. 누가 막혔는지 본다
2. 어떤 액션이 막혔는지 본다
3. 어떤 리소스에서 막혔는지 본다
4. IAM policy의 `Action`과 `Resource`를 맞춰 본다
5. 명시적 Deny, resource policy, SCP, boundary까지 확인한다

특히 `resource`가 에러에 찍혀 있다면, 많은 경우 원인은 액션 이름보다 **리소스 범위 불일치**에 있다.
