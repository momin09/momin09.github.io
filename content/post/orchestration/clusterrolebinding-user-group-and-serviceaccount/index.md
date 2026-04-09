---
title: "ClusterRoleBinding의 User/Group과 ServiceAccount의 차이"
date: 2026-04-09T09:00:00+09:00
draft: false
description: "Kubernetes RBAC에서 User, Group, ServiceAccount가 어디서 오고 어떻게 매칭되는지 정리합니다."
tags: ["개념정리", "Kubernetes", "RBAC", "ServiceAccount", "EKS"]
---

## 배경

RBAC를 보다 보면 `ClusterRoleBinding`의 `subjects`에 아래 세 가지가 함께 등장한다.

- `User`
- `Group`
- `ServiceAccount`

여기서 가장 많이 헷갈리는 지점은 이거다.

> ServiceAccount는 `kubectl get sa`로 보이는데, User와 Group은 도대체 어디에 있는가?

결론부터 말하면, **User와 Group은 쿠버네티스 안에 "저장된 객체"로 존재하지 않는다.**  
이 차이를 이해하면 RBAC 구조가 훨씬 명확해진다.

## 핵심 개념

### 1. ServiceAccount는 쿠버네티스가 직접 관리하는 리소스다

`ServiceAccount`는 파드가 API 서버와 통신할 때 사용하는 **클러스터 내부 신원**이다.

- 네임스페이스에 소속된다
- 리소스로 존재한다
- `kubectl get sa`로 조회할 수 있다
- 파드에 연결할 수 있다

예를 들어 아래 파드는 `my-app` ServiceAccount로 동작한다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: my-app
```

즉, ServiceAccount는 "실체가 있는 쿠버네티스 리소스"다.

### 2. User와 Group은 쿠버네티스 리소스가 아니다

반면 `User`와 `Group`은 쿠버네티스 내부에 저장된 객체가 아니다.

- `kubectl get users` 같은 API는 없다
- `kubectl create user` 같은 명령도 없다
- 오타가 있어도 리소스 생성 자체는 실패하지 않는다

이유는 인증(Authentication)을 쿠버네티스가 직접 관리하지 않고, **외부 인증 결과를 받아서 쓰는 구조**이기 때문이다.

API 서버는 요청이 들어오면 먼저 "이 요청자가 누구인가"를 확인한다.  
그 결과로 아래 같은 값이 들어온다.

```text
username = homin-kim
groups   = [dev, system:authenticated]
```

그리고 RBAC은 이 값을 `RoleBinding` 또는 `ClusterRoleBinding`의 `subjects`와 문자열 기준으로 비교한다.

즉:

- `User`는 username 문자열
- `Group`은 groups 문자열 배열

일 뿐이다.

### 3. ClusterRoleBinding은 "객체 조회"가 아니라 문자열 매칭을 한다

아래 바인딩이 있다고 해보자.

```yaml
subjects:
  - kind: User
    name: homin-kim
  - kind: Group
    name: dev
```

쿠버네티스는 여기서 `homin-kim`이라는 사용자가 실제로 존재하는지 확인하지 않는다.

그 대신 요청이 들어왔을 때:

1. 인증 단계가 `username=homin-kim`, `groups=[dev]`를 넘기고
2. RBAC이 바인딩의 `name`과 비교해서
3. 일치하면 권한을 부여한다

이게 전부다.

그래서 `User/Group`은 **인가 시점에만 의미를 가지는 일회성 문자열**에 가깝다.

### 4. User/Group은 어디서 오는가

`User`와 `Group` 문자열의 출처는 사용하는 인증 방식에 따라 달라진다.

대표적인 예시는 아래와 같다.

#### x509 클라이언트 인증서

- 인증서의 `CN(Common Name)` -> User
- 인증서의 `O(Organization)` -> Group

예를 들어 kubeadm 환경의 `admin.conf`는 보통 이 구조를 쓴다.

- `CN = kubernetes-admin`
- `O = system:masters`

그래서 `system:masters` 그룹에 매칭되어 강한 권한을 갖게 된다.

#### OIDC

사내 SSO나 외부 IdP를 붙이면 토큰의 claim에서 username과 groups를 가져온다.

- Dex
- Keycloak
- Google
- Okta

같은 구성이 여기에 해당한다.

#### 클라우드 IAM 연동

관리형 Kubernetes에서는 IAM 인증 결과를 Kubernetes username/groups로 매핑하는 방식이 자주 쓰인다.

EKS를 예로 들면, 결국 핵심은 동일하다.

- IAM 사용자 또는 역할로 인증하고
- 그 결과가 Kubernetes의 username/groups로 해석되고
- RBAC은 그 문자열을 기준으로 권한을 판단한다

즉, **EKS라고 해서 RBAC의 원리가 달라지는 것은 아니다.**

### 5. 시스템이 자동으로 붙이는 그룹도 있다

일부 그룹은 시스템이 자동으로 부여한다.

- `system:authenticated`
- `system:unauthenticated`
- `system:masters`
- `system:nodes`

이 값들도 결국 RBAC에서는 일반 Group 문자열처럼 동작한다.

## ServiceAccount를 조금 더 자세히 보면

### 1. 모든 파드는 ServiceAccount를 하나 가진다

파드는 항상 하나의 ServiceAccount를 사용한다.

- 명시하면 해당 SA 사용
- 명시하지 않으면 네임스페이스의 `default` SA 사용

그래서 ServiceAccount는 사실상 **파드의 기본 신원**이다.

### 2. ServiceAccount는 토큰을 통해 자신을 증명한다

파드가 API 서버에 요청할 때는 ServiceAccount 토큰을 사용한다.

Kubernetes 1.24 이후에는 보안상 이유로, 예전의 장기 Secret 토큰 대신 **짧은 수명의 projected token**이 기본 방식이 되었다.

보통 파드 안에서는 아래 경로로 마운트된다.

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

필요하면 직접 토큰을 발급해서 확인할 수도 있다.

```bash
kubectl create token my-app --duration=1h
```

### 3. ServiceAccount도 내부적으로는 username/groups를 가진다

흥미로운 점은 ServiceAccount 역시 인증 시점에는 내부적으로 username/groups 형태로 표현된다는 점이다.

예를 들어:

```text
username = system:serviceaccount:production:my-app
groups   = [system:serviceaccounts, system:serviceaccounts:production]
```

그래서 아래처럼 그룹 기반 권한 부여도 가능하다.

- 특정 네임스페이스의 모든 ServiceAccount에 권한 부여
- 모든 ServiceAccount 공통 권한 부여

즉, `ServiceAccount`는 실체가 있는 리소스이지만, 인증/인가 단계에 들어가면 결국 문자열 신원으로 변환되어 처리된다.

## 실전에서 중요한 점

### 1. User/Group은 조회 대상이 아니라 "인증 결과"다

User/Group을 찾으려고 `kubectl` 리소스를 뒤지는 순간부터 헷갈리기 쉽다.

봐야 하는 곳은 보통 아래 둘 중 하나다.

- 클러스터의 인증 방식 설정
- 실제 요청이 어떤 username/groups로 들어오는지

간단한 확인에는 아래 명령이 유용하다.

```bash
kubectl auth whoami
```

이 명령은 현재 인증된 주체가 어떤 username과 groups로 보이는지 확인할 때 도움이 된다.

### 2. ServiceAccount는 워크로드 권한, User/Group은 사람 또는 외부 시스템 권한으로 보면 편하다

실무에서는 이렇게 나누면 이해가 빠르다.

- `ServiceAccount`: 파드, 컨트롤러, 오퍼레이터 같은 클러스터 내부 워크로드
- `User/Group`: 사람, CI, 외부 인증 시스템을 통해 들어오는 주체

물론 내부 구현상 모두 문자열 신원으로 귀결되지만, 운영 관점에서는 이 구분이 매우 유용하다.

### 3. default ServiceAccount에 권한을 붙이는 건 피하는 편이 좋다

앱별로 ServiceAccount를 따로 만들고 필요한 권한만 부여하는 것이 기본이다.

특히 아래 습관은 피하는 편이 좋다.

- `default` SA에 광범위한 권한 부여
- 모든 워크로드가 같은 SA 공유
- API 호출이 필요 없는 파드에도 토큰 자동 마운트 유지

토큰이 필요 없는 경우에는 아래 옵션으로 마운트를 끌 수 있다.

```yaml
automountServiceAccountToken: false
```

## 요약

- `User`와 `Group`은 쿠버네티스 리소스가 아니라 인증 단계에서 들어오는 문자열이다.
- `ClusterRoleBinding`은 그 문자열을 실제 사용자 객체와 대조하는 것이 아니라, 요청 시점의 `username/groups`와 단순 매칭한다.
- `ServiceAccount`는 쿠버네티스가 직접 관리하는 리소스이며, 파드의 신원으로 사용된다.
- 하지만 ServiceAccount도 인증 단계에 들어가면 결국 username/groups 형태로 표현된다.
- RBAC를 이해할 때는 `ServiceAccount는 실체가 있는 내부 신원`, `User/Group은 외부 인증이 넘겨주는 문자열 라벨`이라고 구분하면 가장 깔끔하다.
