---
title: "Cross-Account ECR Sync Architecture"
date: 2026-04-01
draft: false
toc: true
comments: false
---

## 아키텍처

<img
  src="sync-architecture.svg"
  alt="Cross-account ECR sync architecture"
  loading="lazy"
  style="display:block;width:100%;max-width:902px;height:auto;margin:0 auto 1.5rem;" />

이 구조는 `us-east-1`에서 발생한 ECR `CreateRepository` 이벤트를 수집해서, `Account A`의 `ap-northeast-2`에서 메타데이터 동기화를 수행하는 흐름으로 설계되어 있습니다. 핵심은 이벤트는 `us-east-1`에서 감지하지만, 실제 동기화 작업은 `ap-northeast-2`의 메인 계정에서 실행된다는 점입니다.

## 전체 흐름

1. 여러 계정에서 `us-east-1` ECR 리포지토리가 생성되면 CloudTrail이 `CreateRepository` 이벤트를 기록합니다.
2. 이 이벤트는 Event Bus를 거쳐 `eventName: CreateRepository` 조건의 EventBridge Rule에 매칭됩니다.
3. 매칭된 이벤트는 cross-region, cross-account 경로를 통해 `Account A`의 `ap-northeast-2` Event Bus로 전달됩니다.
4. `Account A`의 `ap-northeast-2`에서 다시 EventBridge Rule이 이벤트를 받고, Lambda가 메타데이터 동기화를 수행합니다.
5. Lambda는 대상 ECR에 태그와 Lifecycle Policy를 반영해 리포지토리 메타데이터를 맞춥니다.

## 영역별 해석

### 1. 상단: 이벤트 수집 구간

상단 박스는 `AWS Cloud / us-east-1`에서 일어나는 이벤트 수집 구간입니다. 다이어그램상 `All Accounts (Main + Others)` 범위 안에서 다음 순서가 보입니다.

- `ECR CreateRepository`
- `CloudTrail`
- `Event Bus`
- `EventBridge Rule`

즉, 리포지토리 생성 자체를 직접 폴링하지 않고 CloudTrail 이벤트 기반으로 받아서 EventBridge로 연결하는 구조입니다.

### 2. 중단: 실제 동기화 구간

중단 박스는 `Main Account - ap-northeast-2`에서 실제 동기화가 실행되는 영역입니다.

- `Event Bus (ap-northeast-2)`가 전달된 이벤트를 수신합니다.
- `EventBridge Rule`이 다시 한 번 이벤트를 필터링합니다.
- `Lambda`가 메타데이터 동기화를 수행합니다.
- 마지막으로 `ECR (ap-northeast-2)`에 `TagResource`, `PutLifecyclePolicy` 성격의 작업을 적용합니다.

즉, 이 구간은 “이벤트 전달”이 아니라 “실제 반영” 레이어입니다.

### 3. 하단: 루프 방지 구간

하단의 빨간 박스는 `ap-northeast-2 자체 이벤트 (루프백 차단)` 영역입니다. 여기서는 `ap-northeast-2` 안에서 다시 발생한 생성 이벤트가 Lambda를 다시 호출하지 않도록 차단하는 의도가 표현되어 있습니다.

이 부분은 운영상 매우 중요합니다. 만약 대상 리전에서 다시 발생한 이벤트까지 동일하게 처리하면 다음 문제가 생길 수 있습니다.

- 자기 자신이 만든 변경을 다시 이벤트로 받아 재처리하는 루프
- 중복 태깅, 중복 정책 적용
- 불필요한 Lambda 실행 증가

따라서 하단 영역은 “동기화 대상 리전의 자체 이벤트는 제외한다”는 안전장치로 읽는 게 맞습니다.

## 이 아키텍처의 핵심 포인트

- 이벤트 감지는 `us-east-1`
- 동기화 실행은 `Account A / ap-northeast-2`
- 동기화 대상은 ECR 메타데이터
- 대상 리전 자체 이벤트는 루프 방지를 위해 차단

## 정리

이 구조는 단순 복제가 아니라, `CreateRepository` 이벤트를 기준으로 태그와 Lifecycle Policy 같은 메타데이터를 다른 리전으로 맞춰 주는 이벤트 기반 동기화 아키텍처입니다. 이벤트 수집, cross-account 전달, 중앙 처리, 루프 방지까지 분리되어 있어서 운영 관점에서도 의도가 명확한 편입니다.
