---
title: "EKS 구축 with Fargate, ALB Controller, ArgoCD"
date: 2026-04-09T00:00:00+09:00
draft: false
toc: true
comments: false
homeProject: true
homeLabel: "Cloud"
homeSummary: "EKS 생성부터 Fargate, ALB, ArgoCD까지 연결한 초기 부트스트랩 구성."
homeTags: ["EKS", "Fargate", "ArgoCD"]
aliases:
    - /page/project/eks-bootstrap/
---

## 구성 요소

이번 프로젝트에서 다룬 핵심 구성은 아래와 같다.

- EKS Cluster
- Fargate Profile
- AWS Load Balancer Controller
- ArgoCD
- ALB Ingress

즉, 클러스터 생성에서 끝나는 것이 아니라 **기본 실행 환경 + 외부 노출 + GitOps 진입점**까지 한 번에 묶어서 정리한 구성이다.

## 구축 순서

### 1. 사전 도구 준비

먼저 아래 도구가 필요하다.

- `awscli`
- `kubectl`
- `eksctl`
- `helm`
- `argocd`

이 단계는 단순 설치처럼 보이지만, 실제로는 이후 명령 대부분이 이 도구들에 걸려 있기 때문에 가장 먼저 맞춰 두는 편이 좋다.

### 2. EKS Cluster Role 생성

EKS 클러스터는 AWS 리소스를 관리하기 위해 전용 IAM Role이 필요하다.

기본적으로는:

1. EKS가 Assume 할 수 있는 trust policy를 만들고
2. Role을 생성한 뒤
3. `AmazonEKSClusterPolicy`를 연결한다

핵심 포인트는 **클러스터 생성 전에 Role을 먼저 준비해야 한다**는 점이다.

예시 trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "eks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### 3. EKS Cluster 생성

클러스터는 AWS CLI로 생성했다.

이 단계에서 중요한 입력값은 아래다.

- 클러스터 이름
- 리전
- Kubernetes 버전
- Cluster Role ARN
- 서브넷 정보

생성 이후에는 `describe-cluster`로 상태를 확인하고, `update-kubeconfig`로 로컬 `kubectl` 컨텍스트를 연결한다.

즉, 클러스터 생성 자체보다도 **생성 후 운영자가 바로 접속 가능한 상태까지 만드는 것**이 이 단계의 끝이다.

## 실행 환경 구성

### 1. Fargate Profile 구성

Fargate 기반으로 Pod를 실행하려면 별도의 Pod Execution Role과 Fargate Profile이 필요하다.

여기서 핵심은 두 가지다.

- 실행 Role이 있어야 하고
- 어떤 namespace/pod selector가 Fargate에서 동작할지 명시해야 한다

특히 초기 단계에서는 `kube-system` 관련 Fargate Profile이 중요했다. CoreDNS 같은 기본 컴포넌트가 정상적으로 올라오려면 해당 namespace를 커버하는 프로파일이 필요하기 때문이다.

### 2. CoreDNS 재시작 확인

Fargate Profile 생성 이후에는 CoreDNS가 정상적으로 재스케줄되는지 확인해야 한다.

```bash
kubectl rollout restart deployment/coredns -n kube-system
```

이 단계는 작아 보여도, DNS가 흔들리면 이후 모든 컴포넌트가 연쇄적으로 꼬일 수 있어서 중요하다.

## 외부 노출 구성

### 1. AWS Load Balancer Controller 사전 작업

Ingress를 ALB로 노출하려면 먼저 AWS Load Balancer Controller를 위한 IAM 구성이 필요하다.

실제로는 아래 순서로 진행했다.

1. IAM Policy 생성
2. OIDC Provider 연결
3. IAM Service Account 생성

여기서 핵심은 **Kubernetes Service Account와 AWS IAM Role을 연결하는 흐름**이다.  
즉, 단순히 Helm 설치만 하는 게 아니라 그 전에 IAM 기반 권한 위임 경로를 먼저 맞춰야 한다.

### 2. Helm으로 Controller 배포

이후 `eks-charts`를 Helm repo로 등록하고, `aws-load-balancer-controller`를 배포했다.

이 단계의 목적은 명확하다.

- Kubernetes Ingress 리소스를
- AWS ALB로 연결할 수 있게 만드는 것

즉, EKS에서 외부 트래픽을 다루는 기반을 여는 단계다.

## GitOps 진입점 구성

### 1. ArgoCD 배포 전 준비

ArgoCD도 Fargate 기반으로 올릴 예정이었기 때문에, `argocd` namespace용 Fargate Profile을 추가로 생성했다.

그리고 실제로는 **서브넷 태깅 작업**이 중요했다.  
이걸 빼먹으면 Ingress 생성 시 다음과 같은 오류가 발생한다.

```text
Failed build model due to couldn't auto-discover subnets: unable to resolve at least one subnet (0 match VPC and tags)
```

즉, ALB Controller는 단순히 클러스터 안 리소스만 보는 게 아니라, AWS 쪽 서브넷 태그 정보까지 보고 대상 서브넷을 선택한다.

필요했던 태깅 규칙은 아래였다.

```text
Private Subnet => kubernetes.io/role/internal-elb: 1
Public Subnet  => kubernetes.io/role/elb: 1
```

### 2. ArgoCD 배포

ArgoCD는 두 가지 방식으로 배포 가능했다.

- Helm 사용
- `kubectl apply` 사용

이번 정리에서는 둘 다 기록해 두었지만, 운영 일관성을 생각하면 한 가지 방식으로 정착하는 편이 더 좋다.

### 3. ArgoCD Ingress 구성

ArgoCD를 외부에서 접근할 수 있도록 ALB Ingress를 생성했다.

이때 확인 포인트는 아래였다.

- `ingressClassName: alb`
- HTTPS 리스너 설정
- ACM 인증서 ARN
- `alb.ingress.kubernetes.io/target-type: ip`
- 로그인 페이지 기준 health check 경로

즉, ArgoCD 자체 배포보다도 **외부 접근을 위한 Ingress 정합성**이 실제 체감 난이도를 좌우했다.

## 트러블슈팅 메모

### 1. Subnet auto-discovery 실패

문제:

- ALB 생성 시 대상 서브넷을 자동으로 찾지 못했다.

원인:

- 서브넷 태그가 없거나 기대하는 형식과 맞지 않았다.

해결:

- Public/Private Subnet에 필요한 태그를 명시적으로 추가했다.

### 2. ArgoCD 로그인 리다이렉트 이슈

문제:

- HTTPS로 접속할 때는 정상 로그인되는데, HTTP로 접근 시 리다이렉트 흐름이 기대대로 동작하지 않았다.

현재 판단:

- Ingress annotation 또는 redirect 설정 정합성 문제일 가능성이 높다.

이 부분은 아직 "완전 해결"보다는 **후속 확인 포인트가 남아 있는 메모**로 두는 편이 맞다.  
프로젝트 문서에서는 이런 미해결 상태도 숨기지 않고 남겨 두는 것이 나중에 다시 볼 때 더 유용하다.

## 정리

이 프로젝트는 단순히 EKS 클러스터를 띄우는 데서 끝나지 않았다.

- EKS 클러스터 생성
- Fargate 실행 기반 준비
- ALB Controller 배포
- ArgoCD 배포
- 외부 접근용 Ingress 구성

까지 이어지는, 비교적 실제 운영에 가까운 초기 부트스트랩 흐름을 한 번에 다뤘다.

특히 다시 봐야 할 포인트는 아래 두 가지다.

- AWS 리소스와 Kubernetes 리소스의 연결 지점은 대부분 IAM/OIDC/태깅에서 막힌다.
- "설치는 끝났는데 접속이 안 된다"는 문제는 대체로 Ingress, 인증서, 서브넷 선택 조건에서 나온다.

앞으로 이 문서를 기반으로 각 단계별 글을 더 쪼개면, `EKS 생성`, `Fargate 운영`, `ALB Controller`, `ArgoCD Ingress 트러블슈팅` 식으로 후속 글로 확장하기 좋다.
