---
title: "PowerShell로 Windows 방화벽 규칙 조회/추가/삭제하기"
slug: "powershell-windows-firewall-rule-commands"
date: 2026-04-09T17:00:00+09:00
draft: false
description: "PowerShell로 Windows 방화벽 규칙을 조회하고, 포트 규칙을 추가하거나 삭제할 때 자주 쓰는 명령을 변수 기반으로 정리합니다."
tags: ["PowerShell", "Windows", "Firewall", "WSL"]
---

## 배경

Windows에서 WSL이나 특정 로컬 서비스 포트를 열어야 할 때가 있다.

이럴 때 PowerShell로 방화벽 규칙을 바로 조회하거나 추가/삭제해 두면 편하다.  
이번 글은 아래처럼 자주 쓰는 명령만 짧게 정리한다.

- 특정 규칙 이름 패턴으로 조회
- 특정 포트 범위 허용 규칙 추가
- 특정 규칙의 포트 정보 확인
- 특정 규칙 삭제

## 환경

기준 환경은 아래다.

- Windows PowerShell 또는 PowerShell 7
- Windows Defender Firewall
- 관리자 권한 PowerShell 권장

특히 `New-NetFirewallRule`, `Remove-NetFirewallRule` 같은 명령은 관리자 권한에서 실행하는 편이 안전하다.

## 핵심 내용

### 0. 공통 변수 먼저 정리

먼저 아래처럼 이름, 패턴, 포트, 방향, 프로토콜, 액션을 변수로 빼 두고 시작하면 편하다.

```powershell
$RuleNamePattern      = "<RULE_NAME_PATTERN>"
$RuleDisplayName      = "<RULE_DISPLAY_NAME>"
$Direction            = "Inbound"
$LocalPortRange       = "<LOCAL_PORT_OR_RANGE>"
$Protocol             = "TCP"
$RuleAction           = "Allow"
```

즉, 아래 명령은 전부 이 변수들만 바꿔서 재사용하는 방식으로 보면 된다.

### 1. 특정 이름 패턴으로 방화벽 규칙 조회

규칙 이름이 일정한 패턴으로 묶여 있다면, 아래처럼 `DisplayName` 기준으로 조회하면 된다.

```powershell
Get-NetFirewallRule |
    Where-Object { $_.DisplayName -like $RuleNamePattern } |
    Select-Object DisplayName, Enabled, Action
```

이 명령은:

- `DisplayName`이 특정 패턴과 맞는 규칙만 찾고
- 규칙 이름, 활성화 여부, 허용/차단 액션만 간단히 보여준다

### 2. 특정 포트 범위를 허용하는 규칙 추가

WSL 서비스나 로컬 애플리케이션 포트 범위를 열 때는 아래처럼 변수 기반으로 정리해 두면 재사용하기 좋다.

```powershell
New-NetFirewallRule `
    -DisplayName $RuleDisplayName `
    -Direction $Direction `
    -LocalPort $LocalPortRange `
    -Protocol $Protocol `
    -Action $RuleAction
```

핵심은 이거다.

- 규칙 이름도 변수
- 방향도 변수
- 포트 범위도 변수
- 프로토콜과 액션도 변수

이렇게 빼 두면 나중에 포트 범위나 이름만 바꿔서 바로 재사용할 수 있다.

### 3. 특정 규칙의 포트 정보 확인

규칙은 있는데 실제 어떤 포트가 걸려 있는지 보고 싶을 때는 `Get-NetFirewallPortFilter`를 같이 보면 된다.

```powershell
Get-NetFirewallRule -DisplayName $RuleDisplayName |
    Get-NetFirewallPortFilter
```

이 명령은 규칙 본체가 아니라, 그 규칙에 연결된 포트 필터 정보를 보여준다.

즉:

- 어떤 프로토콜인지
- 어떤 로컬 포트인지
- 어떤 원격 포트 조건이 있는지

같은 걸 볼 때 유용하다.

### 4. 특정 규칙 삭제

기존 규칙을 정리할 때는 아래처럼 `DisplayName`을 변수로 빼서 삭제하면 된다.

```powershell
Remove-NetFirewallRule -DisplayName $RuleDisplayName
```

이건 이름이 정확히 맞아야 한다.  
그래서 보통은 바로 지우기 전에 먼저 한 번 조회해 보고 삭제하는 편이 덜 위험하다.

예:

```powershell
Get-NetFirewallRule -DisplayName $RuleDisplayName
Remove-NetFirewallRule -DisplayName $RuleDisplayName
```

## 실전에서 중요한 점

### 1. 이름 규칙을 먼저 정해 두는 게 좋다

방화벽 규칙은 나중에 쌓이면 찾기 꽤 귀찮아진다.

그래서 이름을 아래처럼 일정하게 맞춰 두는 게 좋다.

- `<TEAM> <HOST> <SERVICE> <PORT>`
- `<PRODUCT> <SERVICE_GROUP>`
- `<APP_NAME>`

즉, **누가 봐도 어떤 용도인지 보이는 이름**으로 두는 게 제일 낫다.

### 2. 포트 범위는 문자열 변수로 두는 게 편하다

단일 포트든 범위 포트든 `-LocalPort`에는 문자열로 넣는 편이 정리하기 좋다.

예:

```powershell
$LocalPortRange = "<SINGLE_PORT>"
$LocalPortRange = "<PORT_RANGE>"
```

이렇게 해 두면 단일 포트와 범위를 같은 패턴으로 다룰 수 있다.

### 3. 삭제는 조회 후 실행하는 습관이 좋다

`Remove-NetFirewallRule`은 바로 반영되기 때문에, 이름이 애매하면 먼저 조회하고 지우는 쪽이 안전하다.

이건 특히 비슷한 이름 규칙이 여러 개 있을 때 중요하다.

## 요약

- `Get-NetFirewallRule` + `Where-Object`로 이름 패턴 기반 조회를 할 수 있다.
- `New-NetFirewallRule`은 이름, 방향, 포트, 프로토콜, 액션을 변수로 빼 두면 재사용하기 좋다.
- `Get-NetFirewallPortFilter`로 특정 규칙의 실제 포트 정보를 확인할 수 있다.
- `Remove-NetFirewallRule`로 규칙 삭제가 가능하고, 삭제 전 조회하는 습관이 안전하다.
- 실무에서는 규칙 이름과 포트 범위를 변수화해 두는 방식이 가장 관리하기 편하다.
