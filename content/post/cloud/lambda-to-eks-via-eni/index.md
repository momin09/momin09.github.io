---
title: "외부망이 막힌 환경에서 Lambda가 EKS에 접근할 때 VPC Endpoint가 필요했던 이유"
date: 2026-04-09T10:00:00+09:00
draft: false
description: "us-east-1의 외부망 차단 환경에서 Lambda가 EKS에 접근할 때, Hyperplane ENI만으로는 충분하지 않았고 EKS VPC Endpoint가 필요했던 흐름을 정리합니다."
tags: ["개념정리", "AWS", "Lambda", "EKS", "VPC Endpoint"]
---

## 배경

`us-east-1`에서 외부망 접근이 막힌 환경에서 Lambda가 같은 Account의 EKS를 호출해야 하는 상황이 있었다.

처음에는 아래처럼 이해하기 쉬웠다.

- Lambda는 VPC에 붙어 있다
- Hyperplane ENI도 정상적으로 붙는다
- 같은 Account의 EKS니까 VPC 내부로 바로 갈 것 같다

그런데 실제로는 원하는 대로 동작하지 않았다.

Lambda 내부에서 Kubernetes 관련 라이브러리로 API를 호출했을 때, 요청 경로를 따라가 보면 `https://eks.us-east-1.amazonaws.com` 을 향하고 있었고, 이 경로는 단순히 `Lambda ENI -> EKS private endpoint`로 끝나는 구조가 아니었다.

결론적으로 이 문제는 **Hyperplane ENI의 문제는 아니었고, 목적지가 VPC 내부 사설 엔드포인트가 아니라 AWS의 regional EKS service endpoint였기 때문에 발생한 문제**였다.

## 전제

이 문서는 아래 조건을 기준으로 정리한다.

- 리전: `us-east-1`
- Lambda와 EKS가 같은 AWS Account에 있음
- Lambda는 VPC에 연결되어 있음
- Lambda 쪽 Hyperplane ENI는 정상적으로 생성 및 동작함
- 외부 인터넷(IGW/NAT 경유) 접근은 막혀 있음

여기서 중요한 건, **Lambda가 VPC에 붙어 있다는 사실만으로 모든 AWS API가 사설 경로로 바뀌는 것은 아니라는 점**이다.

## 핵심 구분

이 문제는 아래 두 경로를 구분해야 이해가 된다.

### 1. Lambda의 VPC 연결 경로

Lambda가 VPC에 연결되면, Lambda 서비스는 선택한 subnet/security group 조합으로 Hyperplane ENI를 사용해 VPC 내부에 붙는다.

즉, 출발지는 정상적으로 VPC 내부 private source가 된다.

```text
Lambda
    -> Hyperplane ENI
    -> VPC 내부 사설 네트워크
```

여기까지는 정상이다.

### 2. 실제 호출 대상 경로

문제는 Lambda 애플리케이션이 무엇을 호출하느냐였다.

이번 케이스에서는 Kubernetes 라이브러리 로직 안에서 최종적으로 `https://eks.us-east-1.amazonaws.com` 을 향하는 호출이 발생했다.

이 주소는 **클러스터 private endpoint 자체가 아니라 AWS EKS regional service endpoint**로 봐야 한다.

즉, 우리가 기대한 경로:

```text
Lambda -> ENI -> EKS private endpoint
```

가 아니라, 실제로는 아래에 가까웠다.

```text
Lambda
    -> Hyperplane ENI
    -> VPC 내부 경로
    -> AWS EKS regional service endpoint (eks.us-east-1.amazonaws.com)
```

외부망이 막힌 환경에서는 이 호출이 완결되지 않았다.

## 실제로 관찰한 흐름

운영 기준으로 보면 흐름은 이렇게 정리할 수 있다.

1. Lambda의 Hyperplane ENI는 정상적으로 동작했다.
2. 즉, Lambda가 VPC에 붙지 못해서 실패한 것은 아니었다.
3. 하지만 Lambda 내부에서 Kubernetes 라이브러리 호출 시 `https://eks.us-east-1.amazonaws.com` 방향의 요청이 발생했다.
4. 이 경로는 단순 사설 EKS endpoint 접근과 달라서, 외부망이 막힌 상태에서는 통신이 되지 않았다.
5. `VPC Endpoint`를 생성한 뒤 동작이 정상화되었다.

즉, 실패 원인은 "Lambda가 VPC에 못 붙음"이 아니라 **AWS 서비스 엔드포인트에 대한 사설 경로가 없었음**에 더 가까웠다.

## 왜 VPC Endpoint가 필요했는가

이번 케이스에서는 Lambda가 사설 네트워크 안에서 출발하더라도, 목적지인 `eks.us-east-1.amazonaws.com` 에 도달하는 별도 경로가 필요했다.

그래서 Interface VPC Endpoint를 만들어 아래처럼 경로를 사설화해야 했다.

```text
Lambda
    -> Hyperplane ENI
    -> Interface VPC Endpoint
    -> EKS regional service
```

이렇게 되면 Lambda는 인터넷을 거치지 않고도 VPC 내부 사설 경로로 EKS 관련 API를 호출할 수 있다.

즉, 이번 구조의 핵심은:

- Hyperplane ENI는 정상
- 하지만 그것만으로는 부족
- `eks.us-east-1.amazonaws.com` 호출을 private하게 종결할 수 있는 VPC Endpoint가 추가로 필요

라는 점이다.

## 정리하면 트래픽 경로는 이렇게 봐야 했다

이번 사례를 문서로 남길 때는 아래처럼 적는 편이 정확하다.

### 기대했던 경로

```text
Lambda
    -> Hyperplane ENI
    -> EKS private endpoint
    -> Kubernetes API
```

### 실제로 문제를 일으킨 경로

```text
Lambda
    -> Hyperplane ENI
    -> eks.us-east-1.amazonaws.com
    -> 외부망 차단으로 실패
```

### 최종 동작한 경로

```text
Lambda
    -> Hyperplane ENI
    -> EKS Interface VPC Endpoint
    -> eks.us-east-1.amazonaws.com
    -> 필요한 EKS API 처리
```

즉, **Lambda의 ENI는 출발지 문제를 해결해 줬고, VPC Endpoint는 목적지 문제를 해결해 줬다**고 보는 게 가장 정확하다.

## 문서화할 때 주의할 점

이 케이스를 단순히 "Lambda가 ENI로 EKS에 붙었다"라고만 쓰면 중요한 맥락이 빠진다.

정확히는 아래처럼 나누어 적는 편이 좋다.

- Lambda의 VPC 연결 자체는 Hyperplane ENI로 정상 동작했다
- 하지만 Kubernetes 라이브러리 호출 과정에서 `eks.us-east-1.amazonaws.com` 으로 향하는 요청이 있었다
- 외부망이 차단된 환경에서는 이 regional service endpoint에 대한 private 경로가 없어서 실패했다
- EKS용 Interface VPC Endpoint 생성 후 통신이 완료되었다

이렇게 적어야 "왜 ENI가 있는데도 안 됐는가"가 설명된다.

## 정리

- 이번 문제는 Hyperplane ENI 장애가 아니었다.
- Lambda는 정상적으로 VPC 내부에 붙어 있었다.
- 실제 병목은 `https://eks.us-east-1.amazonaws.com` 호출 경로였다.
- 외부망이 막힌 환경에서는 이 AWS regional service endpoint를 private하게 접근할 수 있도록 VPC Endpoint가 필요했다.
- 따라서 이번 사례는 `Lambda ENI만으로 충분한 구조`가 아니라, `Lambda ENI + EKS VPC Endpoint`가 함께 필요했던 구조로 이해하는 것이 맞다.
