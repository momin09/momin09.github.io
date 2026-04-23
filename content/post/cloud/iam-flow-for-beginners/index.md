---
title: "IAM 흐름으로 이해하기: Root, User, Policy, Group, Role"
date: 2023-04-23T10:00:00+09:00
draft: false
description: "AWS IAM을 처음 볼 때 헷갈리는 Root Account, IAM User, Policy, Group, Role, Trust Relationship 흐름을 한 번에 정리합니다."
tags: ["AWS", "IAM", "Policy", "Role", "Security"]
---

## 배경

IAM은 처음 보면 용어가 비슷비슷해서 흐름이 잘 안 잡힌다.

특히 아래가 자주 섞인다.

- `Root Account`와 `IAM User`
- `Policy`와 `Group`
- `Role`과 `Switch Role`
- `권한 부여`와 `신뢰 관계`

그래서 이 글은 기능을 하나씩 나열하기보다는, **IAM이 실제로 어떤 순서로 등장하는지** 기준으로 정리한다.

흐름만 먼저 적어보면 이렇다.

```text
Root Account
-> IAM User 생성
-> Policy로 권한 부여
-> 비슷한 권한은 Group으로 묶기
-> AWS 서비스 권한은 Role로 분리
-> 다른 계정/주체가 Role을 쓰려면 Trust Relationship 필요
```

## 환경

이 글은 IAM 입문 관점에서 정리한다.

다루는 범위는 아래다.

- `Root Account`
- `IAM User`
- `IAM Policy`
- `IAM Group`
- `IAM Role`
- `Switch Role`
- `Trust Relationship`

그리고 설명을 쉽게 하기 위해 일부 상황은 단순화해서 적지만, 기술적으로 틀리기 쉬운 부분은 같이 바로잡아 둔다.

## 핵심 내용

### 1. 처음 시작은 Root Account다

AWS에 회원가입하면 가장 먼저 생기는 건 `Root Account`다.

이 계정은 말 그대로 계정의 주인이라서 권한이 가장 강하다.  
그래서 실무에서는 Root를 계속 쓰기보다, 초기에 필요한 작업만 하고 **일반 작업용 IAM User나 Role로 내려오는 것**이 기본이다.

즉, 시작점은 Root지만 운영의 주체가 Root가 되는 건 보안상 좋지 않다.

### 2. IAM User는 "사람이 쓰는 계정"으로 이해하면 편하다

그 다음 자주 등장하는 게 `IAM User`다.

이건 보통:

- 콘솔에 로그인하는 사람
- CLI를 쓰는 사람
- 오래된 방식의 액세스 키를 쓰는 주체

를 표현할 때 많이 쓴다.

중요한 건, **새로 만든 IAM User는 기본적으로 아무 권한도 없다**는 점이다.

즉:

- User를 만들었다
- 그런데 아무것도 못 한다

이 흐름이 정상이다.

권한은 따로 붙여줘야 한다.

### 3. Policy가 실제 권한 내용이다

IAM에서 진짜 중요한 건 `Policy`다.

정책은 결국 아래를 적는 문서다.

- 누가
- 어떤 리소스에 대해
- 어떤 액션을
- 허용할지 또는 거부할지

예를 들면 이런 식이다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ]
    }
  ]
}
```

핵심은 이거다.

> IAM에서 실제 권한은 User나 Group이 아니라 Policy에 들어 있다

User, Group, Role은 결국 그 Policy를 붙여서 권한을 받는 그릇에 가깝다.

### 4. 같은 권한이 반복되면 Group으로 묶는다

사용자가 한 명일 때는 User에 직접 Policy를 붙여도 된다.

그런데 비슷한 권한이 필요한 사람이 많아지면 같은 작업을 계속 반복하게 된다.  
이럴 때 쓰는 게 `IAM Group`이다.

예를 들면:

- S3 읽기/쓰기 권한이 필요한 사용자 여러 명
- 읽기 전용 사용자 여러 명

이런 경우 User마다 따로 붙이는 대신 Group에 Policy를 붙이고, User를 Group에 넣는 식으로 관리한다.

즉:

- 권한 내용은 Policy
- 사람 여러 명 묶는 건 Group

이렇게 나누면 된다.

### 5. AWS 서비스 권한은 User보다 Role로 보는 게 맞다

여기서 IAM이 조금 더 어려워지는 지점이 `Role`이다.

많이 헷갈리는 부분이 이거다.

> EC2가 S3에 접근해야 하는데, EC2에 User를 붙이나?

그건 아니다.

AWS 서비스가 다른 AWS 서비스에 접근할 때는 보통 `IAM Role`을 쓴다.

대표적인 예:

- EC2가 S3에 접근
- Lambda가 DynamoDB에 접근
- EKS Pod가 AWS API를 호출

이런 경우는 User보다 Role이 맞다.

왜냐하면 Role은 **임시 자격 증명** 기반으로 동작하고, 사람이 로그인하는 계정보다 서비스 권한 위임에 훨씬 적합하기 때문이다.

즉:

- 사람이 직접 쓰는 건 User
- 서비스나 워크로드가 쓰는 건 Role

이렇게 기억하면 흐름이 훨씬 편하다.

### 6. Role도 결국 Policy를 받아서 권한을 가진다

Role이라고 해서 따로 마법처럼 권한이 생기는 건 아니다.

Role도 마찬가지로:

1. Role을 만들고
2. Policy를 연결하고
3. 그 Role을 EC2, Lambda 같은 주체가 사용한다

이 구조다.

즉, Role은 "권한이 담기는 대상"이지, 권한 정의 그 자체는 아니다.

예를 들어 EC2가 특정 S3 bucket에만 업로드하게 하고 싶다면:

- S3 전체 권한을 줄 필요는 없고
- 특정 bucket의 `ListBucket`, `PutObject` 정도만 허용한 Policy를 만들고
- 그 Policy를 Role에 연결한 뒤
- 해당 Role을 EC2에 붙이면 된다

이 흐름이 가장 기본적인 서비스 권한 부여 방식이다.

### 7. Switch Role은 "다른 권한 세트로 잠깐 들어가는 것"이다

이번엔 사람 기준으로 다시 보자.

IAM User가 평소에는 큰 권한이 없는데, 잠깐 더 높은 권한이 필요할 때가 있다.  
이럴 때 나오는 개념이 `Switch Role`이다.

쉽게 말하면:

- 평소 계정은 그대로 있고
- 잠깐 특정 Role을 assume해서
- 그 Role이 가진 권한으로 작업하는 것

이라고 보면 된다.

발표 자료 식으로 표현하면 "완장" 비유를 써도 이해는 쉽다.  
다만 문서로 적을 때는 **임시로 다른 Role의 권한을 사용한다** 정도로 적는 게 더 정확하다.

### 8. 여기서 중요한 게 Trust Relationship이다

Role은 Policy만 붙였다고 끝이 아니다.

Role은 누가 그 Role을 사용할 수 있는지도 따로 정의해야 한다.  
그게 `Trust Relationship`, 즉 신뢰 관계다.

이건 Role의 trust policy에서 관리한다.

예를 들어:

- 특정 AWS 서비스만 이 Role을 사용할 수 있게 할지
- 특정 계정의 사용자만 AssumeRole 할 수 있게 할지

를 여기서 정한다.

핵심은 아래 두 가지다.

1. 호출하는 쪽에 `sts:AssumeRole` 권한이 필요할 수 있다
2. 대상 Role 쪽 trust policy에 호출 주체가 허용되어 있어야 한다

그래서 현업에서 흔히 말하는 "왜 Switch Role이 안 되지?"는 대부분:

- 호출 주체 권한 문제이거나
- trust relationship 문제이거나
- 둘 다인 경우

가 많다.

여기서 중요한 건 `무조건 양방향 Allow`라고 외우는 것보다,  
**"AssumeRole은 호출 권한과 trust policy를 같이 봐야 한다"**고 이해하는 쪽이 훨씬 정확하다는 점이다.

## 전체 흐름을 한 번에 보면

정말 흐름만 놓고 다시 적으면 이렇다.

### 1. Root Account

- AWS 계정의 시작점
- 너무 강해서 일상 운영에는 잘 안 씀

### 2. IAM User

- 사람용 계정
- 처음엔 권한 없음

### 3. Policy

- 실제 권한 내용
- Action / Resource / Effect 정의

### 4. Group

- 비슷한 권한이 필요한 User 묶음
- 권한 관리를 반복하지 않게 해 줌

### 5. Role

- 서비스나 워크로드에 권한 위임할 때 주로 사용
- 임시 자격 증명 기반

### 6. Switch Role / AssumeRole

- 다른 권한 세트로 잠깐 들어가는 방식
- 사람이나 서비스가 Role을 사용할 때 등장

### 7. Trust Relationship

- 누가 그 Role을 쓸 수 있는지 정의
- Role 사용 가능 여부를 결정하는 핵심

## 실전에서 중요한 점

### 1. User에 장기 권한을 계속 쌓아 올리는 건 피하는 게 좋다

처음 IAM을 배우면 User에 권한을 계속 붙이게 된다.

그런데 실무에선:

- 사람 권한은 가능한 한 줄이고
- 서비스 권한은 Role로 분리하고
- 필요할 때만 AssumeRole로 올려 쓰는 구조

가 더 안전하다.

### 2. Role은 "서비스 전용"이라고만 보면 반만 맞다

Role은 EC2, Lambda 같은 서비스에 많이 붙지만, 거기서 끝은 아니다.

- 다른 계정 사용자
- federated user
- AWS 서비스

도 Role을 assume할 수 있다.

즉, Role은 "서비스가 쓰는 권한"이라고 시작하면 이해는 쉽지만, 실제로는 더 넓은 개념이다.

### 3. IAM은 결국 Policy와 Trust를 같이 봐야 한다

권한이 안 될 때는 보통 둘 중 하나만 보고 끝내기 쉽다.

그런데 실제로는:

- Policy는 붙어 있는데 trust가 안 맞거나
- trust는 맞는데 AssumeRole 권한이 없거나
- Resource 범위가 안 맞거나

이런 경우가 많다.

그래서 IAM은 항상:

- 누가 호출하는지
- 어떤 Policy가 붙어 있는지
- trust relationship이 어떻게 되어 있는지

를 같이 봐야 한다.

## 요약

- IAM의 시작점은 Root Account지만, 운영의 중심은 User와 Role로 내려오는 게 보통이다.
- User는 사람용 계정이고, Role은 서비스나 다른 주체에게 권한을 위임할 때 많이 쓴다.
- 실제 권한 내용은 Policy에 들어 있다.
- 비슷한 권한이 반복되면 Group으로 묶는다.
- Switch Role은 다른 Role의 권한을 임시로 사용하는 방식이다.
- Role을 쓰려면 Policy만 아니라 Trust Relationship도 같이 맞아야 한다.
- IAM은 `User / Policy / Group / Role / Trust` 흐름으로 보면 생각보다 훨씬 덜 헷갈린다.
