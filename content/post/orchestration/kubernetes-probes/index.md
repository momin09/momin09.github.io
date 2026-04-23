---
title: "Kubernetes Probe 정리: liveness, readiness, startup"
date: 2026-04-09T10:40:00+09:00
draft: false
description: "Kubernetes의 livenessProbe, readinessProbe, startupProbe가 각각 언제 어떤 목적로 동작하는지 정리합니다."
tags: ["개념정리", "Kubernetes", "Probe", "Readiness", "Liveness"]
---

## 배경

Kubernetes에서 애플리케이션 상태를 볼 때 가장 자주 보게 되는 설정이 `probe`다.

그런데 막상 쓰다 보면 아래가 은근 헷갈린다.

- `livenessProbe`와 `readinessProbe`는 뭐가 다른가
- `startupProbe`는 언제 써야 하는가
- 세 개가 동시에 있으면 누가 먼저 동작하는가

특히 `startupProbe`는 많이들 "최초 1번만 도는 체크"처럼 이해하는데, 정확히는 **초기 기동 구간에서만 따로 도는 probe**라고 보면 된다.

## 환경

이 글은 Kubernetes Pod의 container probe 기준으로 정리한다.

대표적으로 아래 세 가지를 다룬다.

- `livenessProbe`
- `readinessProbe`
- `startupProbe`

## 핵심 내용

### 1. livenessProbe

`livenessProbe`는 말 그대로 **컨테이너가 살아 있는지** 본다.

이 probe가 실패하면 Kubernetes는:

- 컨테이너가 비정상 상태라고 판단하고
- 해당 컨테이너를 재시작한다

즉, 목적은 **죽었거나 더 이상 정상 상태로 보기 어려운 컨테이너를 다시 띄우는 것**에 가깝다.

이 probe는 애플리케이션이 떠 있는 동안 계속 주기적으로 체크한다.

### 2. readinessProbe

`readinessProbe`는 **지금 트래픽을 받아도 되는 상태인지** 확인한다.

이 probe가 실패하면 Kubernetes는:

- Pod를 죽이지는 않고
- Service endpoint 목록에서 제외한다

즉, 목적은 **트래픽을 붙일지 말지 결정하는 것**이다.

애플리케이션이 살아는 있지만:

- 초기화가 덜 끝났거나
- DB 연결이 아직 준비되지 않았거나
- 잠시 요청을 받을 수 없는 상태라면

`readinessProbe`를 실패시켜서 트래픽만 빼 둘 수 있다.

이 probe도 실행 중에는 계속 주기적으로 체크한다.

### 3. startupProbe

`startupProbe`는 **느리게 뜨는 애플리케이션의 시작 구간만 따로 보호하기 위한 probe**다.

핵심은 이거다.

- 컨테이너 시작 직후부터 동작한다
- 성공하기 전까지는 `livenessProbe`와 `readinessProbe`가 사실상 대기한다
- 최초 성공 1회 이후에는 더 이상 실행되지 않는다

즉, `startupProbe`는 **시작 구간에서만 반복 체크**하고, 한 번 성공하면 역할이 끝난다.

말투를 조금 실무적으로 줄이면 이렇게 이해하면 된다.

> 초기 구간에서 성공할 때까지 체크하고, 최초 성공 1회 이후에는 다시 체크하지 않는다

이 구조가 중요한 이유는, 앱이 뜨는 데 오래 걸리는데 `livenessProbe`가 먼저 들어오면 정상 기동 중인 컨테이너를 죽여 버릴 수 있기 때문이다.

`startupProbe`는 그 초기 구간을 따로 분리해서 보호해 준다.

## 동작 차이 요약

| Probe | 목적 | 실패 시 동작 | 실행 시점 |
|---|---|---|---|
| `livenessProbe` | 살아 있는지 확인 | 컨테이너 재시작 | 실행 중 계속 |
| `readinessProbe` | 트래픽 받을 준비 확인 | Service endpoint 제외 | 실행 중 계속 |
| `startupProbe` | 초기 기동 완료 확인 | 기동 실패로 판단 | 시작 구간에서만 |

진짜 핵심만 다시 적으면:

- `livenessProbe`: 계속 헬스체크
- `readinessProbe`: 계속 헬스체크
- `startupProbe`: 시작 구간에서만 체크, 성공 후 종료

## 실전에서 중요한 점

### 1. startupProbe가 있으면 초기 기동을 따로 보호할 수 있다

앱이 느리게 뜨는 경우 `livenessProbe`만 두면, 아직 뜨는 중인 앱을 비정상으로 오해할 수 있다.

이때 `startupProbe`를 두면:

- 애플리케이션이 뜨는 동안은 startup probe가 먼저 보고
- 최초 성공 이후부터 liveness/readiness가 계속 이어진다

그래서 Java, Spring Boot, 대형 모델 로딩, 캐시 warm-up 같은 느린 시작 구간에서 특히 유용하다.

### 2. liveness와 readiness는 역할이 다르다

둘 다 헬스체크이긴 한데, 쓰는 목적은 다르다.

- `livenessProbe`는 재시작 판단
- `readinessProbe`는 트래픽 유입 판단

그래서 같은 endpoint를 쓰더라도 의미는 따로 생각하는 게 좋다.

### 3. startupProbe를 "딱 1회 실행"으로 이해하면 조금 틀릴 수 있다

이건 운영에서 진짜 자주 헷갈린다.

`startupProbe`는:

- 컨테이너 시작 후
- `periodSeconds` 간격으로 반복 체크하고
- 성공하면 종료된다

즉, "처음에만 쓰는 probe"는 맞다. 다만 **최초 성공 1회 이후엔 안 돈다**는 뜻이지, 시작하자마자 딱 1번만 실행된다는 뜻은 아니다.

## 예시

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5

startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 5
  failureThreshold: 30
```

위 설정이면:

- `startupProbe`가 먼저 초기 기동 구간을 확인하고
- 최초 성공한 뒤부터
- `livenessProbe`와 `readinessProbe`가 지속적으로 상태를 본다

## 요약

- `livenessProbe`는 컨테이너가 살아 있는지 계속 확인한다.
- `readinessProbe`는 트래픽을 받을 준비가 되었는지 계속 확인한다.
- `startupProbe`는 초기 기동 구간에서만 동작하고, 최초 성공 1회 이후에는 더 이상 실행되지 않는다.
- 그래서 느리게 뜨는 애플리케이션에서는 `startupProbe`가 매우 중요하다.
- 운영에서는 `startupProbe = 시작 구간 보호`, `liveness = 재시작`, `readiness = 트래픽 제어`로 기억하면 가장 깔끔하다.
