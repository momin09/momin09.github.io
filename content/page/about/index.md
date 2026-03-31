---
title: About
description: Cloud | DevOps | SRE Engineer - KIM HOMIN
date: 2026-03-30
lastmod: 2026-03-30
menu:
    main: 
        weight: -90
        params:
            icon: user
---

# KIM HOMIN

**Cloud | DevOps | SRE**

> [!INFO] Contact
> - 📧 khm0909z@gmail.com
> - 📞 010-6284-2911
> - 📍 서울시 마포구 망원동

---

## PROFESSIONAL SUMMARY

> **"대규모 엔터프라이즈 트래픽을 처리하는 대한항공 DCO 프로젝트의 SRE/Cloud 엔지니어"**
>
> AWS Solutions Architect Professional 및 CKA 자격을 보유한 3년 차 DevOps 엔지니어입니다. Enterprise급 아키텍처 설계를 기반으로 인프라를 빈틈없이 구현하고, 운영 효율성을 고려하여 설계를 기술적으로 보완하는 역량을 보유하고 있습니다. 특히 YAML 기반의 Terraform 프레임워크를 독자 개발하여 IaC 진입 장벽을 낮추고, 데이터 기반의 성능 최적화를 통해 서비스 안정성을 확보하는 데 주력합니다.

---

## WORK EXPERIENCE

### (주)비즈테크아이 | 대한항공 DCO 클라우드 운영
**2023.03 – Present**
*DevOps Engineer (SRE) / Cloud Infrastructure Team*

대한항공 DCO(DataCenter Operation) 프로젝트에서 AWS 클라우드 인프라 구축 및 운영 전반을 수행합니다. 설계된 아키텍처를 실제 환경에 최적화하여 구현하고, Python/Terraform을 활용한 자동화 환경 구축에 기여합니다.

#### Key Achievements

1. **YAML 기반 Terraform 프레임워크 개발 및 레거시 자원 표준화 (2025)**
   - `Context`: Terraform HCL의 높은 진입 장벽으로 인한 DevOps 병목 현상 및 비표준 자원 증가 문제.
   - `Solution`: HCL 지식 없이 YAML 설정만으로 리소스를 생성하는 **추상화 모듈(Abstraction Module)**을 개발하고, Naming/Tagging 정책을 코드로 강제화.
   - `Impact`: 개발팀의 Self-Service 인프라 환경을 구축하고, 기존 레거시 자원의 **100% IaC 이관 및 거버넌스 체계** 확립.

2. **Terraform Plan 파이프라인 구축 및 리뷰 프로세스 개선 (2025)**
   - `Context`: YAML 모듈 도입 후 급증한 변경 요청의 리스크를 관리하고, 리뷰어 검토 시간 단축 필요.
   - `Solution`: Jenkins 파이프라인에 Plan 검증을 통합하고, **PR 코멘트(상세)와 Google Chat(알림)을 연동한 ChatOps 체계** 구축.
   - `Impact`: 리뷰어의 심리적 부담 제거 및 변경 사항에 대한 **실시간 가시성(Real-time Visibility)** 확보.

3. **Terraform Migration 진행 (2025 - 2026)**
   - `Context`: 콘솔에서 생성된 대규모 리소스(ECR 등)가 관리 사각지대에 있어 변경 추적이 어렵고, 수동 이관 시 **과도한 반복 작업(Toil) 및 휴먼 에러 발생** 우려.
   - `Solution`: **Python 자동화 스크립트**를 개발하여 기존 리소스를 Terraform State로 신속하게 **Import 및 코드화**. (서비스 중단 없이 EFS 이관 완료, ECR 대규모 마이그레이션 수행)
   - `Impact`: 수작업 대비 **이관 공수를 획기적으로 단축**하고, 모든 인프라 자산을 IaC 파이프라인에 편입하여 **구성 정합성(Consistency)** 확보.

4. **AWS Enterprise 인프라 구현 및 아키텍처 고도화 (2023 – Present)**
   - `Context`: 10여 개 신규 대규모 서비스의 아키텍처 설계서를 토대로, 실제 운영 가능한 고가용성 인프라 구축.
   - `Solution`: 설계 단계의 보안/효율성 취약점을 분석하여 **기술적 보완책을 역제안**하고, Datadog 메트릭 기반 스케일링 전략 수립.
   - `Impact`: 잠재적 결함을 사전 제거하여 Re-work 비용을 최소화하고 **성공적인 서비스 런칭** 완수.

5. **Helm Template 구조 개선 및 표준화 (2024)**
   - `Context`: 100여 개 마이크로서비스의 개별적인 Helm Chart 관리로 인한 설정 중복 발생.
   - `Solution`: 공통 설정을 집약한 **Library Chart**를 도입하여 중복을 제거하고, 언어별 특성에 맞춘 표준 템플릿 모듈화.
   - `Impact`: 신규 서비스 배포를 위한 설정 시간을 **기존 대비 70% 단축**하고 배포 안정성 확보.

#### Main Responsibilities
- Terraform(YAML 추상화 모듈, 레거시 이관), AWS SAM(Serverless 배포), Helm(Library Chart)
- EKS Fargate 프로파일 관리 및 Pod 최적화, Python/Lambda 기반의 재기동 자동화, HPA 비용 최적화
- Multi-Account(Organizations) 및 IAM 정책 관리, Route53/CloudFront 운영, DR 구축 지원
- Jenkins 유지보수, 빌드/배포 스크립트 최적화
- Datadog(APM, Log) 및 CloudWatch 모니터링 대시보드 고도화, 장애 알람 임계치 최적화
- 24/7 On Call 대기 및 지원

---

## TECH SKILLS

| Category | Skills |
| :--- | :--- |
| **Cloud** | **AWS** (EKS, Lambda, S3, CloudFront, Route53, IAM, Organizations) |
| **Orchestration** | **Kubernetes** (EKS, Fargate), Helm |
| **IaC** | **Terraform** (Module Dev), **AWS SAM** |
| **Script** | **Python** |
| **CI/CD & DevOps** | **Jenkins**, AWS CodePipeline, Git |
| **Observability** | **Datadog** (APM, Log), AWS CloudWatch |

---

## CERTIFICATIONS

- **AWS Certified Solutions Architect – Professional (SAP)** | 2024.06.29
- **Certified Kubernetes Administrator (CKA)** | 2023.09.02
- **AWS Certified Solutions Architect – Associate (SAA)** | 2023.06.24
- **정보처리기사** | 2022.11.25
- **네트워크 관리사 2급** | 2022.07.12
- **AWS Certified Cloud Practitioner (CCP)** | 2021.06.25

---

## EDUCATION

- **가톨릭대학교** | 2016.03 - 2023.02
  - 컴퓨터정보공학부 학사
- **구름 쿠버네티스 전문가 양성 과정** | 2021.07 - 2021.11
  - AWS, CI/CD Pipeline, Kubernetes
