---
title: "JVM GC 로그와 Runtime Metrics를 함께 볼 때 확인할 것들"
slug: "jvm-gc-runtime-metrics-checkpoints"
date: 2026-04-10T16:45:00+09:00
draft: false
description: "GC 로그와 JVM Runtime Metrics를 함께 보고 old gen 포화, Full GC, Humongous Allocation 관점에서 무엇을 확인하면 좋을지 정리합니다."
tags: ["JVM", "Java", "GC", "G1GC", "Observability", "Troubleshooting"]
---

## 배경

JVM Runtime Metrics 화면 2장과 GC 로그 CSV 1개를 함께 보고, 현재 메모리 상황을 어떤 관점으로 해석하면 좋을지 정리한 메모다.

핵심은 단순히 `GC 횟수`가 아니라 아래를 같이 보는 것이다.

- old gen이 얼마나 빨리 차는지
- Full GC가 실제로 메모리를 줄여주는지
- non-heap, classes, thread 쪽도 같이 이상 신호가 있는지
- 큰 객체 할당이나 evacuation failure 같은 경고 패턴이 보이는지

## 확인한 자료

- Runtime Metrics 스크린샷 2장
- GC 로그 CSV 1개
  - `extract-2026-04-10T07_20_07.322Z.csv`

CSV 기준으로 보면 로그는 한 JVM만 있는 것이 아니라 여러 JVM 인스턴스 로그가 섞여 있는 것으로 보인다. `Using G1` 시작 로그가 여러 번 나오기 때문에, 실제 원인 분석 단계에서는 pod 또는 instance 단위로 다시 분리해서 보는 것이 좋다.

## 핵심 내용

### 1. 가장 먼저 보이는 신호는 old gen 포화다

스크린샷에서는 여러 JVM 인스턴스가 old gen을 빠르게 `5GB` 이상까지 채우는 흐름이 보인다.

GC 로그에서도 아래와 같이 거의 힙 상한선까지 차 있는 구간이 반복된다.

```text
[161.778s][info][gc] GC(126) Pause Full (G1 Compaction Pause) 5454M->4984M(5476M) 4105.226ms
[168.103s][info][gc] GC(143) Pause Full (G1 Compaction Pause) 5468M->5201M(5476M) 4184.930ms
[884.323s][info][gc] GC(320) Pause Full (G1 Compaction Pause) 5473M->5473M(5476M) 5982.989ms
```

이 패턴은 단순히 GC가 자주 돈다는 수준이 아니라, old gen이 거의 꽉 찬 상태에서 Full GC까지 갔는데도 회복 폭이 작거나 아예 회복이 안 되는 상태에 가깝다.

### 2. Full GC가 너무 많고 너무 길다

CSV를 집계해 보면 다음과 같은 특징이 있다.

- 전체 로그 수: `3637`
- `Pause Young`: `1808`
- `Concurrent Mark Cycle`: `850`
- `Pause Full`: `690`
- `Humongous Allocation`: `14`

특히 `Pause Full`은 평균 약 `6098ms`, 최대 약 `8679ms` 수준이다.

이 정도면 단순 내부 이벤트가 아니라 애플리케이션 응답 시간이나 배치 처리 시간에 실제 영향을 줄 수 있는 수준으로 봐야 한다.

### 3. Evacuation Failure 이후 Full GC로 들어간다

문제가 커지는 초반 흐름도 비교적 선명하다.

```text
[157.672s][info][gc] GC(125) Pause Young (Normal) (G1 Evacuation Pause) (Evacuation Failure) 5430M->5454M(5476M) 25.678ms
[161.778s][info][gc] GC(126) Pause Full (G1 Compaction Pause) 5454M->4984M(5476M) 4105.226ms
```

즉, evacuation failure가 먼저 보이고 곧바로 Full GC로 이어진다. 이는 G1이 객체를 정상적으로 이동시키기 어려울 정도로 여유 영역이 부족했음을 시사한다.

### 4. Humongous Allocation도 같이 봐야 한다

로그에는 `G1 Humongous Allocation`도 보인다.

```text
[166.075s][info][gc] GC(29) Pause Young (Concurrent Start) (G1 Humongous Allocation) 231M->171M(488M) 7.441ms
[246.397s][info][gc] GC(168) Pause Young (Concurrent Start) (G1 Humongous Allocation) 5431M->5271M(5476M) 137.389ms
```

이는 큰 `byte[]`, `char[]`, 대형 문자열 버퍼, 큰 JSON payload, 파일 처리 버퍼 같은 객체가 원인일 수 있다. 횟수 자체는 많지 않더라도 old gen 압박이 큰 시점과 겹치면 영향이 커질 수 있다.

### 5. non-heap, classes, thread는 우선순위가 상대적으로 낮아 보인다

스크린샷 기준으로는 아래 항목들이 가장 큰 이상 신호로 보이지는 않는다.

- `Non-Heap Usage`
- `Number of Classes Loaded`
- `Thread Count`

따라서 현재 우선순위는 metaspace 누수나 thread leak보다 heap old gen 문제 쪽이 더 높아 보인다.

## 무엇을 확인하면 좋을까

### 1. 여러 JVM 로그가 섞였는지 먼저 분리한다

지금 CSV는 여러 인스턴스 로그가 섞여 있는 것처럼 보인다. 먼저 아래 기준으로 분리해서 보는 것이 좋다.

- pod 이름
- instance ID
- hostname
- 컨테이너 재시작 시점

### 2. Full GC 직후에도 내려가지 않는지 본다

다음 패턴이면 우선순위를 높여야 한다.

- Full GC 이후 사용량이 계속 `5GB` 안팎에 머문다
- `5473M->5473M(5476M)`처럼 회복이 거의 없다
- 시간이 갈수록 GC 시간만 길어진다

### 3. heap dump를 가장 의심스러운 시점에 뜬다

heap dump는 아래 시점 중 하나에서 확보하는 것이 좋다.

- old gen이 `4.5GB` 이상 올라간 시점
- Full GC 직후에도 거의 안 내려간 시점
- `Humongous Allocation` 직후

heap dump에서 우선 볼 항목은 아래와 같다.

- `byte[]`
- `char[]`
- `java.lang.String`
- `HashMap`
- `ArrayList`
- 애플리케이션 캐시 객체
- 대형 응답/직렬화 버퍼

### 4. 캐시 정책을 확인한다

운영 환경에서 생각보다 흔한 원인은 누수보다 캐시 설정이다.

- local cache 사용 여부
- max size 설정 여부
- expire 설정 여부
- refresh 전략
- key cardinality가 과도하게 커지지 않는지

### 5. 대용량 객체를 만드는 코드 경로를 확인한다

`Humongous Allocation`이 보였기 때문에 아래 코드 경로도 같이 점검하면 좋다.

- 큰 JSON 응답 생성
- 파일 다운로드/업로드 버퍼
- 이미지/PDF 처리
- 압축/복호화 버퍼
- 대량 문자열 결합
- 대량 조회 결과를 한 번에 메모리에 적재하는 코드

### 6. 컨테이너 메모리 제한과 Xmx 설정을 같이 본다

로그상 힙 상한은 자주 `5476M`으로 보인다. 이 값이 컨테이너 memory limit와 너무 가깝다면 JVM이 회복 여유 없이 끝까지 몰릴 수 있다.

- 컨테이너 memory limit
- `-Xmx`
- `-XX:MaxRAMPercentage`
- pod OOMKill 이력

## 요약

이번 자료에서 가장 먼저 의심해야 할 것은 `old gen 포화와 Full GC 반복`이다. 반면 non-heap, classes, thread 쪽은 우선순위가 낮아 보인다.

다음 순서로 확인하면 효율적이다.

1. pod 또는 instance 기준으로 로그와 메트릭을 분리
2. 가장 빨리 포화되는 인스턴스 확인
3. Full GC 직후에도 안 내려가는 시점의 heap dump 확보
4. dominator tree로 장수 객체, 캐시, 대형 배열/문자열 확인
5. 캐시 정책과 대용량 객체 생성 코드 경로 점검
