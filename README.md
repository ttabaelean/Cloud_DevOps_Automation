
# ☁️ 클라우드 DevOps 자동화

### Cloud DevOps Automation

**숭실사이버대학교 AI·컴퓨터공학과 | 2026학년도 2학기**

**담당 교수 : 이성미**

---

## 📖 과정 소개

클라우드 환경에서 애플리케이션을 개발하고 배포하는 방법은 빠르게 변화하고 있습니다.

기존에는 서버를 직접 구축하고 애플리케이션을 수동으로 배포했다면, 오늘날에는 **컨테이너, Kubernetes, CI/CD, GitOps**를 활용하여 애플리케이션의 빌드부터 배포 및 운영까지 자동화하고 있습니다.

본 과정에서는 Linux와 Docker의 기초부터 AWS 클라우드 인프라, Kubernetes, GitHub Actions, Argo CD까지 단계적으로 학습합니다.

최종적으로 **GitHub에 소스 코드를 Push하면 컨테이너 이미지가 자동으로 빌드되고, Amazon ECR에 저장된 후 Amazon EKS에 배포되는 DevOps 자동화 파이프라인**을 구현합니다.

이론 학습과 Hands-on Labs를 병행하여 클라우드 DevOps의 핵심 개념과 실제 동작 원리를 이해하는 것을 목표로 합니다.

---

## 🎯 학습 목표

본 과정을 마치면 다음과 같은 역량을 갖출 수 있습니다.

1. **Linux 운영 환경 이해**
   - Linux 기본 명령어와 시스템 관리
   - 사용자, 권한, 프로세스 및 서비스 관리

2. **컨테이너 기반 애플리케이션 운영**
   - Docker 이미지와 컨테이너의 이해
   - Dockerfile을 활용한 컨테이너 이미지 생성

3. **AWS 클라우드 인프라 구축**
   - AWS VPC, EC2, ALB, ASG, RDS 이해
   - Amazon ECR 및 EKS를 활용한 컨테이너 배포

4. **CI/CD 자동화**
   - Git과 GitHub를 활용한 형상 관리
   - GitHub Actions를 이용한 CI 파이프라인 구축
   - Argo CD 기반 GitOps 자동 배포

5. **DevOps 파이프라인 운영**
   - 배포 상태 점검
   - 장애 원인 분석 및 복구
   - 자동화 파이프라인의 전체 운영 흐름 이해

---

## 🛠️ 주요 학습 기술

| 분야 | 기술 및 서비스 |
|---|---|
| Operating System | Linux |
| Container | Docker, Dockerfile |
| Cloud | AWS |
| AWS Infrastructure | VPC, EC2, ALB, ASG, RDS |
| Container Registry | Amazon ECR |
| Container Orchestration | Kubernetes, Amazon EKS |
| Version Control | Git, GitHub |
| CI | GitHub Actions |
| CD / GitOps | Argo CD |
| Application | Spring PetClinic |
| Automation | CI/CD Pipeline, GitOps |

---

## 📚 주차별 강의 구성

| 주차 | 학습 주제 | 주요 학습 내용 |
|---|---|---|
| 1주차 | Linux 기초 | Linux 환경 및 기본 명령어 |
| 2주차 | Linux 시스템 관리 | 파일 시스템, 사용자 및 권한 관리 |
| 3주차 | Linux 운영 관리 | 프로세스, 서비스 및 시스템 관리 |
| 4주차 | Docker 기초 | 컨테이너 이해 및 Docker 사용 |
| 5주차 | Dockerfile | 컨테이너 이미지 생성 및 관리 |
| 6주차 | 클라우드 인프라 아키텍처 이해 | AWS 계정 보안, VPC, EC2 |
| 7주차 | 클라우드 확장 서비스 및 애플리케이션 아키텍처 | ALB, Auto Scaling, RDS |
| 8주차 | AWS 컨테이너 서비스 | Amazon ECR, EKS, Kubernetes Service |
| 9주차 | 3-Tier 애플리케이션 아키텍처 | EKS, RDS, Ingress 기반 애플리케이션 배포 |
| 10주차 | Git과 GitHub를 활용한 형상 관리 | Git 명령어, GitHub 원격 저장소 |
| 11주차 | GitHub Actions를 활용한 CI 파이프라인 구축 | Workflow, 이미지 빌드 및 ECR Push |
| 12주차 | Argo CD를 활용한 GitOps CD | GitOps, Argo CD, Kubernetes 자동 배포 |
| 13주차 | 클라우드 DevOps 자동화 구현 | GitHub Actions + ECR + Argo CD + EKS |
| 14주차 | 클라우드 DevOps 자동화 운영 및 과정 리뷰 | 파이프라인 점검, 장애 분석 및 과정 정리 |

---

## 🧑‍💻 Hands-on Labs

본 저장소에는 강의에서 사용하는 실습 자료가 정리되어 있습니다.

실습은 Linux와 컨테이너 기초부터 AWS 인프라, Kubernetes, CI/CD 자동화까지 단계적으로 진행됩니다.

### 실습 자료 바로가기

| 주차 | Hands-on Lab | 주요 실습 |
|---|---|---|
| 6주차 | [AWS VPC · EC2 실습](https://github.com/ttabaelean/Cloud_DevOps_Automation/search?q=week06&type=code) | AWS 계정 보안, VPC 및 EC2 웹 서버 |
| 7주차 | [ALB · ASG · RDS 실습](https://github.com/ttabaelean/Cloud_DevOps_Automation/search?q=week07&type=code) | 로드밸런서, 오토스케일링, 데이터베이스 |
| 8주차 | [ECR · EKS 실습](https://github.com/ttabaelean/Cloud_DevOps_Automation/search?q=week08&type=code) | 컨테이너 이미지 저장 및 EKS 배포 |
| 9주차 | [3-Tier 애플리케이션 실습](https://github.com/ttabaelean/Cloud_DevOps_Automation/search?q=week09&type=code) | EKS, RDS, Ingress |
| 10주차 | [Git · GitHub 실습](https://github.com/ttabaelean/Cloud_DevOps_Automation/search?q=week10&type=code) | Git 형상 관리 및 원격 저장소 |
| 11주차 | [GitHub Actions CI 실습](https://github.com/ttabaelean/Cloud_DevOps_Automation/search?q=week11&type=code) | 컨테이너 이미지 빌드 및 ECR Push |
| 12주차 | [Argo CD GitOps 실습](week12-argocd-gitops-setup.md) | GitOps 기반 자동 배포 |
| 13주차 | [End-to-End CI/CD 실습](week13-end-to-end-cicd-pipeline.md) | 전체 DevOps 자동화 파이프라인 |
| 14주차 | [DevOps 파이프라인 운영 실습](week14-devops-pipeline-summary.md) | 운영 점검, 장애 재현 및 복구 |

> **참고**
>
> 1~5주차는 Linux 및 Docker 기초 학습을 중심으로 진행합니다.
>
> 일부 실습에서는 AWS 계정이 필요하며, 사용한 AWS 리소스에 따라 비용이 발생할 수 있습니다.
>
> 실습 종료 후에는 각 주차의 리소스 정리 절차를 반드시 수행하시기 바랍니다.

---

## 🔄 DevOps 자동화 파이프라인

본 과정의 최종 목표는 애플리케이션의 코드 변경부터 Kubernetes 배포까지 이어지는 자동화 파이프라인을 이해하고 구현하는 것입니다.

### 전체 아키텍처

```text
                Developer
                    │
                    │ git push
                    ▼
             GitHub Repository
               (kcu-petclinic)
                    │
                    ▼
              GitHub Actions
               CI Pipeline
                    │
              Build & Test
                    │
               Docker Build
                    │
                    ▼
               Amazon ECR
             Container Image
                    │
                    │ Image Tag
                    ▼
             GitOps Repository
                (kcu-gitops)
                    │
                    │ Git 변경 감지
                    ▼
                  Argo CD
               CD / GitOps
                    │
                    │ 자동 동기화
                    ▼
                Amazon EKS
                  Kubernetes
                    │
                    ▼
             Spring PetClinic
                Application
                    │
                    ▼
                   User
```

### CI와 CD의 역할

| 구분 | CI | CD |
|---|---|---|
| 사용 기술 | GitHub Actions | Argo CD |
| 실행 계기 | 소스 코드 Push | GitOps 저장소 변경 |
| 주요 작업 | 빌드, 이미지 생성, ECR Push | 변경 감지, Kubernetes 배포 |
| 결과 | 새로운 컨테이너 이미지 | 새로운 애플리케이션 버전 배포 |

### 자동화 흐름

1. 개발자가 애플리케이션 소스를 수정하고 GitHub에 Push합니다.
2. GitHub Actions가 Workflow를 실행합니다.
3. 애플리케이션을 빌드하고 컨테이너 이미지를 생성합니다.
4. 생성한 이미지를 Amazon ECR에 Push합니다.
5. GitOps 저장소의 Kubernetes 매니페스트에 새로운 이미지 태그를 기록합니다.
6. Argo CD가 GitOps 저장소의 변경 사항을 감지합니다.
7. Amazon EKS에 새로운 애플리케이션 버전을 배포합니다.
8. 사용자는 변경된 애플리케이션을 이용할 수 있습니다.

> **핵심 개념**
>
> **CI는 애플리케이션을 배포할 수 있는 이미지로 만드는 과정이고, CD는 그 이미지를 실제 운영 환경에 배포하는 과정입니다.**
>
> GitOps에서는 Git 저장소를 배포 설정의 기준(Source of Truth)으로 사용합니다.

---

## 🏗️ 최종 종합 실습

### Spring PetClinic End-to-End CI/CD

13주차에서는 지금까지 학습한 기술을 연결하여 전체 DevOps 자동화 파이프라인을 구축합니다.

**실습 구성**

- **GitHub** : Spring PetClinic 소스 코드 관리
- **GitHub Actions** : CI 파이프라인 자동 실행
- **Docker** : 애플리케이션 컨테이너 이미지 생성
- **Amazon ECR** : 컨테이너 이미지 저장
- **GitOps Repository** : Kubernetes 배포 매니페스트 관리
- **Argo CD** : Git 변경 감지 및 자동 배포
- **Amazon EKS** : Kubernetes 애플리케이션 실행

**실습 목표**

소스 코드 변경 후 GitHub에 Push하는 것만으로 새로운 애플리케이션 버전이 자동으로 배포되는 과정을 확인합니다.

📘 [13주차 종합 실습 바로가기](week13-end-to-end-cicd-pipeline.md)

---

## 🔍 DevOps 파이프라인 운영

14주차에서는 구축한 자동화 파이프라인의 운영 상태를 점검하고, 장애를 직접 재현하여 해결하는 방법을 학습합니다.

**운영 점검**

- GitHub Actions Workflow 실행 결과
- Amazon ECR 이미지 및 보안 취약점
- Argo CD 동기화 상태
- Kubernetes Deployment 및 Pod 상태
- 애플리케이션 서비스 상태
- AWS 리소스 및 비용

**장애 재현 및 복구**

- 잘못된 이미지 태그로 인한 `ImagePullBackOff`
- Readiness Probe 오류로 인한 Pod 준비 실패
- Dockerfile 오류로 인한 CI 빌드 실패
- Git Revert를 활용한 복구

📘 [14주차 운영 실습 바로가기](week14-devops-pipeline-summary.md)

---

## ⚠️ 실습 유의 사항

### AWS 비용 관리

- AWS 서비스는 사용량에 따라 요금이 발생할 수 있습니다.
- EKS, EC2, Load Balancer, RDS 등은 실습 후 삭제해야 합니다.
- 각 Hands-on Lab의 **리소스 정리** 절차를 확인합니다.
- AWS 콘솔의 **Billing and Cost Management**에서 비용을 점검합니다.

### 보안 관리

- AWS Access Key와 Secret Access Key를 GitHub에 업로드하지 않습니다.
- GitHub Token 및 인증 정보를 소스 코드에 직접 작성하지 않습니다.
- 실습용 인증 정보는 사용 후 삭제하거나 폐기합니다.
- 실제 운영 환경에서는 GitHub OIDC와 AWS IAM Role 등 단기 자격 증명 방식을 권장합니다.

---

## 📌 학습 안내

- 실습 자료는 강의 내용을 이해하기 위한 교육용 예제입니다.
- AWS 콘솔 화면과 서비스 설정은 업데이트에 따라 일부 달라질 수 있습니다.
- 실습은 각 주차의 순서에 따라 진행하는 것을 권장합니다.
- AWS 비용이 부담되는 경우 강의 영상과 실습 자료를 통해 동작 흐름을 학습할 수 있습니다.
- 강의 교안과 실습 노트를 함께 참고하시기 바랍니다.

---

## 👩‍🏫 강의 정보

| 항목 | 내용 |
|---|---|
| 강의명 | 클라우드 DevOps 자동화 |
| 영문명 | Cloud DevOps Automation |
| 학과 | AI·컴퓨터공학과 |
| 학기 | 2026학년도 2학기 |

---

### ☁️ From Linux to Cloud Native DevOps

**Linux → Docker → AWS → Kubernetes → GitHub Actions → Argo CD → DevOps Automation**

클라우드 인프라의 기초부터 애플리케이션 자동 배포와 운영까지, 단계적으로 학습합니다.

---

**© 2026 클라우드 DevOps 자동화 | 이성미**
