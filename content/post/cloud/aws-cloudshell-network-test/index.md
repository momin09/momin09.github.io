---
title: "CloudShell로 특정 환경에서 네트워크 테스트하기"
date: 2026-04-09T09:10:00+09:00
draft: false
description: "AWS CloudShell을 특정 네트워크 환경에 맞춰 세팅한 뒤 telnet과 curl로 간단히 연결 테스트하는 방법을 정리합니다."
tags: ["AWS", "CloudShell", "Network", "RDS"]
---

## 배경

특정 환경에서 네트워크가 실제로 열려 있는지 확인해야 할 때가 있다.

이럴 때 로컬 PC 대신 **AWS CloudShell**을 테스트 기준점으로 두면 편하다.  
CloudShell을 테스트하려는 네트워크 환경에 맞게 세팅해 두면, 해당 위치에서 바로 도메인과 포트 연결 여부를 확인할 수 있다.

이번 글은 아래처럼 아주 짧은 흐름만 정리한다.

1. CloudShell을 테스트할 네트워크 환경으로 세팅
2. 네트워크 확인용 도구 설치
3. `telnet`, `curl`로 도메인/포트 테스트

## 환경

테스트 전제는 아래와 같다.

- AWS CloudShell 사용
- 테스트 대상 네트워크 환경이 이미 준비되어 있음
- 확인 대상은 RDS endpoint 또는 특정 domain/IP + port

핵심은 **CloudShell이 어느 네트워크 위치에서 동작하느냐**를 먼저 맞추는 것이다.

## 핵심 내용

### 1. CloudShell을 테스트할 네트워크 환경으로 맞춘다

먼저 CloudShell을 실제로 테스트하고 싶은 네트워크 환경 기준으로 세팅한다.

예를 들어:

- 특정 VPC 기준으로 접근 가능한지 확인하고 싶은 경우
- 특정 보안 그룹/라우팅 환경에서 통신이 되는지 보고 싶은 경우
- 내부 도메인 또는 RDS endpoint 접근 가능 여부를 보고 싶은 경우

이 단계가 먼저 맞아야 이후 결과도 의미가 있다.

### 2. 네트워크 툴 설치

이후 아래 명령어로 테스트 도구를 설치한다.

```bash
sudo yum install telnet
```

### 3. RDS 또는 특정 endpoint 포트 확인

RDS endpoint와 port 연결 여부를 볼 때는 아래처럼 확인할 수 있다.

```bash
telnet {rds domain} {port}
```

예:

```bash
telnet mydb.xxxxxxxxxxxx.us-east-1.rds.amazonaws.com 3306
```

이 방식은 해당 도메인과 포트로 TCP 연결이 가능한지 빠르게 볼 때 유용하다.

### 4. domain 또는 IP 기준으로 curl 테스트

HTTP 계열 응답이나 단순 endpoint 접근 여부를 볼 때는 `curl`도 같이 쓸 수 있다.

```bash
curl {domain or ip} {port}
```

예:

```bash
curl example.internal 8080
curl 10.0.10.15 8080
```

상황에 따라 `domain` 기준과 `ip` 기준을 나눠서 보면 DNS 문제인지, 순수 네트워크 문제인지 구분하는 데 도움이 된다.

## 요약

- CloudShell을 먼저 테스트 대상 네트워크 환경 기준으로 맞춘다.
- 이후 `telnet`을 설치해서 TCP 연결 여부를 확인한다.
- RDS는 `telnet {rds domain} {port}` 형태로 빠르게 테스트할 수 있다.
- 추가로 `curl {domain or ip} {port}` 를 사용하면 endpoint 응답이나 도메인/IP 기준 차이도 함께 볼 수 있다.
