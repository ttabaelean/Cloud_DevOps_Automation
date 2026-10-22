
# 14주차 Hands-on Labs: 클라우드 DevOps 자동화 운영 및 과정 리뷰

## 실습 환경

| 항목 | 구성 |
|---|---|
| 실습 범위 | 1강 : 파이프라인 운영 점검 / 2강 : 장애 재현과 해결 / 3강 : 실습 없음 (과정 정리) |
| 클러스터 작업 | AWS CloudShell (서울 리전 `ap-northeast-2`) · `kubectl` · `eksctl` |
| Git 작업 | Windows 10/11 + VSCode 터미널(Git Bash) |
| 환경 | 13주차 환경을 다시 생성 : `~/kcu-infra/cluster.yaml` · ECR `petclinic` · Argo CD · `apps/petclinic.yaml` |
| 저장소 | 애플리케이션 저장소 `kcu-petclinic` / GitOps 저장소 `kcu-gitops` (13주차 그대로 사용) |

> [!WARNING]
> **EKS는 프리티어가 없는 유료 서비스**입니다. 1강 · 2강 실습을 가능하면 하루에 이어서 진행하고, 2강 마지막의 **리소스 정리**를 반드시 수행합니다.
>
> **참고:** 13주차 실습을 하지 않았다면 13주차 Hands-on Labs를 먼저 진행합니다. 비용이 걱정되면 강의 화면과 이 페이지의 예상 출력으로 흐름을 이해하는 것만으로도 충분합니다.

---

## 14주 1강 Hands-on Labs : 파이프라인 운영 점검

13주차 파일(`cluster.yaml` · `apps/petclinic.yaml`)로 환경을 다시 만들고, 파이프라인을 단계별 정상 기준으로 점검합니다.

**환경 복원 → CI 점검 → 이미지 · 보안 점검 → 배포 점검 → 서비스 · 비용 점검**

### 1단계: 12주차까지 진행했던 실습 환경 구성(사전 구성 완료)

> [!NOTE]
> **확인 포인트**
>
> - 13주차에 만든 **설정 파일** 덕분에 같은 환경을 명령 몇 줄로 다시 만들 수 있음 (IaC의 장점)
> - 13주차 정리 때 삭제한 **인증 정보**(Access Key · Token · Secrets)와 **비활성화한 Workflow**는 다시 준비

- **CloudShell** : EKS · ECR · Argo CD 다시 만들기 (EKS 약 15~20분)
  - 리전이 **서울**인지 확인 (CloudShell 홈 폴더는 리전별로 따로 있음)
  - `cat cluster.yaml`로 13주차 파일이 남아 있는지 먼저 확인

```bash
cd ~/kcu-infra
cat cluster.yaml
eksctl create cluster -f cluster.yaml

aws ecr create-repository \
  --repository-name petclinic \
  --image-scanning-configuration scanOnPush=true \
  --region ap-northeast-2

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 로드밸런서 구분용 Name 태그 (13주차 1강 5단계와 동일, 외부 노출 전에 설정)
kubectl annotate svc argocd-server -n argocd \
  service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags='Name=kcu-argocd-lb' \
  --overwrite
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get nodes
kubectl get pods -n argocd
```

- **AWS 콘솔 · GitHub 웹** : 인증 정보 재등록 (13주차 2강 1~2단계와 동일)
  - IAM 사용자 `github-actions-ci` **다시 생성** (13주차 정리 때 삭제함 · 정책 `AmazonEC2ContainerRegistryPowerUser`) → Access Key 발급
  - Fine-grained Token `kcu-gitops-token` 재발급 (Only select : `kcu-gitops` · Contents : Read and write · 30 days)
  - `kcu-petclinic` → Settings → Secrets and variables → Actions : `AWS_ACCESS_KEY_ID` · `AWS_SECRET_ACCESS_KEY` · `GITOPS_TOKEN` 등록
  - `kcu-petclinic` → **Actions → PetClinic CI → [Enable workflow]**

- **내 PC** : 빈 Commit으로 CI 다시 실행 (새 ECR에 이미지 생성 + kcu-gitops 태그 갱신)

```bash
cd ~/kcu-petclinic
git pull
git commit --allow-empty -m "Rebuild for week 14"
git push
```

- **내 PC** : CI 성공 후 kcu-gitops에서 CI가 갱신한 이미지 태그 · readinessProbe 확인 (13주차 2강 4단계에서 작성)

```bash
cd ~/kcu-gitops
git pull
grep -n -E "image:|readinessProbe|path:" petclinic/deployment.yaml
```

```text
...:            image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:3f9c1a7e...(40자리)
...:            readinessProbe:
...:                path: /actuator/health
```

- **CloudShell** : CI 성공 후 Application 생성

```bash
cd ~/kcu-gitops
git pull
kubectl apply -f apps/petclinic.yaml
kubectl get applications -n argocd
```

```text
NAME        SYNC STATUS   HEALTH STATUS
petclinic   Synced        Healthy
```

- **CloudShell** : Argo CD UI 접속 주소 · admin 비밀번호 확인 (새로 설치되어 바뀜)

```bash
ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo https://$ARGOCD_URL

kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

> [!NOTE]
> CloudShell에 `~/kcu-gitops`가 없으면 `git clone https://github.com/<Username>/kcu-gitops.git` 후 진행합니다.

### 2단계: CI 점검 (GitHub 웹 · 내 PC)

> [!NOTE]
> **🎯 목적**
>
> 코드 Push 후 CI가 **정상으로 끝났는지**, 그리고 CI가 만든 이미지가 **배포 대상(kcu-gitops)에 제대로 반영됐는지** 확인합니다.
>
> "코드를 바꿨는데 화면이 안 바뀐다"는 장애가 나면 가장 먼저 보는 곳입니다.

- `kcu-petclinic` → **Actions → PetClinic CI** : 실행 이력 · 성공/실패 · 소요 시간 확인
  - 확인 포인트 : 최근 실행이 초록 체크인지, 평소(약 3~5분)보다 갑자기 오래 걸리지 않는지 → 실패가 반복되거나 시간이 크게 늘면 빌드 문제의 신호
- 최신 실행 → **build** Job : Step별 소요 시간 확인 (보통 Maven 빌드가 가장 오래 걸림)
  - 확인 포인트 : 어느 Step이 시간을 가장 많이 쓰는지 알아 두면, 나중에 CI가 느려졌거나 실패했을 때 원인 Step을 바로 찾을 수 있음

<img width="550" height="554" alt="GitHub Actions PetClinic CI 실행 단계별 점검" src="https://github.com/user-attachments/assets/851bb688-2296-41f6-b2a3-c62fce7029ac" />

- GitOps 매니페스트의 이미지 태그와 소스 Commit SHA 비교
  - **"지금 배포하려는 이미지가 내가 방금 Push한 코드로 만든 이미지가 맞는가?"** 를 확인
  - CI는 소스 Commit SHA를 이미지 태그로 붙이고, 그 태그를 kcu-gitops의 deployment.yaml에 기록함 → 두 값이 같으면 CI → GitOps 연결이 정상
  - 주의 : kcu-petclinic과 kcu-gitops는 서로 다른 저장소라 **Commit SHA 자체는 다름**. 비교 대상은 소스 SHA ↔ 이미지 태그 (또는 Deploy 메시지의 7자리)

```bash
# ① 소스 저장소의 최신 Commit SHA(40자리) = CI가 이미지 태그로 쓴 값
git -C ~/kcu-petclinic log -1 --format=%H

# ② CI가 kcu-gitops에 만든 태그 갱신 Commit을 내 PC로 받아오기
# 배포 매니페스트에 기록된 이미지 주소 → 끝의 태그가 ①과 같아야 정상
git -C ~/kcu-gitops pull
grep image: ~/kcu-gitops/petclinic/deployment.yaml

# kcu-gitops 최근 이력 → 맨 위가 CI(github-actions[bot])가 만든 "Deploy petclinic <소스 SHA 7자리>" 인지 확인
git -C ~/kcu-gitops log --oneline -3
```

### 3단계: 이미지 · 보안 점검 (CloudShell)

> [!NOTE]
> **목적**
>
> - CI가 ECR에 올린 이미지가 **실제로 저장됐는지**, 그리고 그 이미지에 **알려진 보안 취약점(CVE)이 얼마나 있는지** 확인합니다.
> - ECR repository 생성 시 `scanOnPush`를 구성. 이미지가 Push될 때마다 ECR이 자동으로 스캔합니다.
>
> ```bash
> aws ecr create-repository \
>   --repository-name petclinic \
>   --image-scanning-configuration scanOnPush=true \
>   --region ap-northeast-2
> ```

**① 이미지 목록 · Push 시각**

```bash
# ECR petclinic 리포지토리의 이미지 태그와 Push 시각을 표로 출력
# → 2단계에서 본 소스 SHA 태그가 있고, Push 시각이 방금 CI 실행 시각이면 정상
aws ecr describe-images --repository-name petclinic \
  --query 'imageDetails[*].[imageTags[0],imagePushedAt]' --output table
```

```text
----------------------------------------------------------------------------------
|                                 DescribeImages                                 |
+-------------------------------------------+------------------------------------+
|  5b6f7b337a71cef323dd1829a4e48ac7a07ff551 |  2026-10-21T21:09:07.858000+00:00  |
|  276fca025cec6182b1c53d16cf7c935077427636 |  2026-10-21T20:48:42.098000+00:00  |
+-------------------------------------------+------------------------------------+
```

**② 최신 이미지의 스캔 상태 · 취약점 심각도별 개수**

- 이미지가 Push될 때마다 ECR이 이미지 안의 OS 패키지를 검사해서, 알려진 보안 취약점(CVE)이 있는지 스캔
- 스캔 결과 확인

| 항목 | 의미 |
|---|---|
| **CVE ID** | 발견된 취약점의 식별 번호 |
| **심각도(Severity)** | 취약점의 위험 수준 : CRITICAL(치명적) → HIGH(높음) → MEDIUM(중간) → LOW(낮음) |
| **패키지 이름·버전** | 취약점이 발견된 OS 패키지와 설치 버전 |
| **설명** | 어떤 보안 문제가 발생할 수 있는지 설명 |

```bash
# ① 가장 최근에 Push된 이미지의 태그를 TAG 변수에 저장 (Push 시각으로 정렬 → 마지막 1개)
TAG=$(aws ecr describe-images --repository-name petclinic \
  --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags[0]' --output text)
echo $TAG    # 2단계의 소스 SHA와 같은지 눈으로 확인

# ② 그 이미지의 스캔 상태(scanStatus)와 심각도별 취약점 개수(counts)만 골라서 출력
#    scanStatus : COMPLETE = 스캔 끝 / IN_PROGRESS = 아직 스캔 중(잠시 후 다시)
#    counts     : CRITICAL · HIGH · MEDIUM · LOW 등급별 취약점 개수
aws ecr describe-image-scan-findings \
  --repository-name petclinic \
  --image-id imageTag="$TAG" \
  --region ap-northeast-2 \
  --query '{scanStatus:imageScanStatus.status,counts:imageScanFindings.findingSeverityCounts}'
```

```text
{
    "scanStatus": "COMPLETE",
    "counts": {
        "HIGH": 2,
        "MEDIUM": 9,
        "LOW": 4
    }
}
```

**③ 콘솔 ECR → petclinic → 이미지 클릭 → 취약성에서 패키지별 목록 확인 가능**

<img width="1303" height="491" alt="Amazon ECR PetClinic 이미지 취약점 스캔 결과" src="https://github.com/user-attachments/assets/ab87d856-2379-4dbd-b760-a29eced47975" />

- `상태`가 `완료`인지 먼저 확인합니다.
- 이미지 빌드 성공과 보안상 안전함은 별개입니다. ECR 스캔으로 알려진 취약점을 확인하고, 위험도가 높은 항목부터 수정합니다.
- 현재 실습의 CI는 스캔 결과에 따라 배포를 차단하도록 구성하지 않았으므로, 취약점이 발견되어도 배포는 계속 진행됩니다.
- 발견 건수가 0이어도 모든 보안 문제가 없다는 보장은 아닙니다. 기본 스캔은 애플리케이션 소스 코드나 Java 의존성 전체를 검사하는 기능은 아닙니다.
- Critical이 없다고 이미지 전체가 안전하다는 뜻은 아니며, Critical · High가 있으면 조치 필요 여부를 검토합니다.
- 참고: [Amazon ECR 이미지 스캔 공식 문서](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)

### 4단계: 배포 점검 (CloudShell · Argo CD UI)

> [!NOTE]
> **🎯 목적**
>
> - Argo CD가 Git(kcu-gitops)에 적힌 대로 클러스터에 **배포를 끝냈는지**, 그리고 **지금 실행 중인 Pod가 정말 새 이미지로 떠 있는지** 확인합니다.
> - Git(원하는 상태)과 클러스터(실제 상태)가 같은지 보는 단계입니다.

- 참고 : kubectl 자동완성 기능 설정

```bash
sudo yum install -y bash-completion

# 명령어 하나씩 따로 실행
source /usr/share/bash-completion/bash_completion

source <(kubectl completion bash)
```

**① Application · 리소스 상태**

```bash
# Argo CD가 보는 상태
#   SYNC STATUS : Git과 클러스터가 같은가 (Synced = 같음)
#   HEALTH      : 배포된 앱이 정상인가 (Healthy = 정상, Progressing = 배포 진행 중)
kubectl get applications -n argocd

# Deployment → ReplicaSet → Pod 를 한 번에 조회 (-o wide : Pod가 실행 중인 노드 · IP까지 표시)
#   → Deployment READY 2/2, Pod 2개 Running · READY 1/1 · RESTARTS 0 이면 정상
kubectl get deploy,rs,pods -n petclinic -o wide
```

**② 실행 중인 이미지 · 배포 이력 · 최근 Events**

```bash
# 지금 실행 중인 이미지 주소 → 태그가 2단계의 소스 SHA와 같아야 '새 버전이 실제로 배포됨'
kubectl get deploy petclinic -n petclinic -o jsonpath='{..image}'; echo

# Deployment 배포 이력(REVISION) → 이미지가 바뀔 때마다 번호가 하나씩 늘어남
kubectl rollout history deploy/petclinic -n petclinic

# 네임스페이스에서 최근 일어난 사건 5개 (시간순)
#   → 지금은 Pulled · Created · Started 같은 Normal만 보이면 정상
#   → 2강에서 장애를 만들면 여기에 Warning(Failed, Unhealthy 등)이 나타남
kubectl get events -n petclinic --sort-by=.lastTimestamp | tail -5
```

- Argo CD UI → `petclinic` → **[APP DETAILS]** : SYNC STATUS · HEALTH · Revision 확인
  - 확인 포인트 : Revision(Git Commit)이 kcu-gitops 최신 Commit과 같으면 Argo CD가 최신 Git을 배포했다는 뜻

### 5단계: 서비스 · 비용 점검

> [!NOTE]
> **목적**
>
> 사용자 입장에서 **서비스가 실제로 동작하는지** 확인하고, 운영에서 놓치기 쉬운 **비용 · 토큰 만료**까지 점검합니다.
>
> Pod가 Running이어도 앱 내부가 비정상일 수 있어서 상태 확인 주소로 한 번 더 봅니다.

**① 애플리케이션 동작 상태 확인 (CloudShell)**

```bash
# PetClinic Service(로드밸런서)의 외부 주소를 LB 변수에 저장
LB=$(kubectl get svc petclinic-svc -n petclinic \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo http://$LB      # 이 주소를 브라우저에서 열어 화면도 확인

# Spring Boot Actuator 상태 확인 주소 호출 → status UP 이면 애플리케이션 내부까지 정상
#   (readinessProbe도 같은 주소를 사용 → 사람은 curl로, 쿠버네티스는 Probe로 확인)
curl -s http://$LB/actuator/health; echo
```

```text
{"status":"UP","groups":["liveness","readiness"]}
```

**② 애플리케이션 로그 · 노드 유형**

```bash
# 애플리케이션 로그 최근 20줄 (Pod 이름 대신 deploy/petclinic 으로 지정)
#   → 'Started PetClinicApplication' 이 보이고 ERROR 가 없으면 정상 기동
kubectl logs deploy/petclinic -n petclinic --tail=20

# 노드 목록 + 용량 유형 열 추가
#   → CAPACITYTYPE 이 SPOT 이면 cluster.yaml 설정대로 할인 인스턴스 사용 중 (비용 절감)
kubectl get nodes -L eks.amazonaws.com/capacityType
```

**③ AWS 비용 점검**

- AWS 콘솔 **과금정보 및 비용 관리** → 이번 달 EKS · EC2 · Elastic Load Balancing 요금 확인
  - 목적 : EKS(클러스터 시간당 요금) · EC2(노드) · ELB(로드밸런서)는 켜져 있는 동안 계속 과금 → 실습 비용이 얼마나 나오는지 확인하는 습관

**④ GitHub Token 만료일 확인**

- GitHub 프로필 → Settings → Developer settings → **Fine-grained tokens** → `kcu-gitops-token` 만료일 확인
  - 목적 : CI가 kcu-gitops에 Push할 때 쓰는 토큰(30일). 만료되면 어느 날 갑자기 CI의 태그 갱신 Step이 403으로 실패 → 만료 전에 재발급

---

## 14주 2강 Hands-on Labs : 장애 재현과 해결

> [!NOTE]
> **목표**
>
> 장애 3가지를 일부러 만들고 → 증상 확인 → 원인 분석 → Git으로 복구
>
> - 장애 ①② : kcu-gitops 수정 → Argo CD
> - 장애 ③ : kcu-petclinic 수정 → GitHub Actions

> [!NOTE]
> **Pod 상태별 주요 원인 · 확인 명령**
>
> - `Pending` : 노드 자원 부족 · 노드 없음 → `kubectl describe pod` (Events)
> - `ImagePullBackOff` : 이미지 주소 · 태그 · 권한 오류 → `kubectl describe pod` (Events)
> - `CrashLoopBackOff` : 컨테이너가 반복 종료되어 재시작 간격이 늘어난 상태 (주요 원인 : 앱 시작 실패 · 설정 · 메모리)
>   → `kubectl logs <Pod> --previous` + `kubectl describe pod` (Last State · Reason · Exit Code)
> - `Running 0/1` : 실행 중이지만 Ready 아님 (이번 실습에서는 Readiness Probe 경로 오류) → `kubectl describe pod` (Events)

### 1단계: 장애 ① 이미지 태그 오류

**① 내 PC : 이미지 태그 오류 발생**

- `image` 줄 끝에 `-typo`를 붙여 ECR에 없는 태그로 변경

```bash
cd ~/kcu-gitops
git pull

sed -i 's|\(image: .*\)|\1-typo|' petclinic/deployment.yaml
grep image petclinic/deployment.yaml

git diff
git commit -am "Incident 1: wrong image tag"
git push
```

**② CloudShell : 증상 확인**

- 빨리 보려면 Argo CD UI **[REFRESH]**

```bash
kubectl get pods -n petclinic
```

```text
NAME                         READY   STATUS             RESTARTS
petclinic-xxxxxxxxxx-aaaaa   1/1     Running            0
petclinic-xxxxxxxxxx-bbbbb   1/1     Running            0
petclinic-yyyyyyyyyy-ccccc   0/1     ImagePullBackOff   0
```

- 처음에는 `ErrImagePull`, 잠시 뒤 `ImagePullBackOff`로 표시됩니다.

**③ 원인 확인**

- 새 Pod 이름으로 변경

```bash
kubectl describe pod <새 Pod 이름> -n petclinic | tail -8
```

```text
Warning  Failed   ...  Failed to pull image ".../petclinic:3f9c1a7e...-typo": ... not found
Normal   BackOff  ...  Back-off pulling image ".../petclinic:3f9c1a7e...-typo"
```

- 브라우저에서 PetClinic 새로고침 → **화면 정상** (기존 Pod 2개가 계속 서비스)
- Argo CD UI → `petclinic` : Sync **Synced** / Health **Progressing**

**④ 복구 (내 PC)**

```bash
git revert HEAD --no-edit
git push
```

**⑤ 복구 확인**

- Argo CD에서 복구 Commit이 반영되어 **Synced · Healthy**가 되고, `kubectl get pods -n petclinic`에서 **기존 정상 Pod 2개만 READY 1/1**로 남은 것을 확인한 후 다음 장애를 진행합니다.
- Sync가 계속 진행 중이면 Argo CD UI에서 진행 중인 작업과 오류 메시지를 먼저 확인합니다.

### 2단계: 장애 ② Readiness Probe 실패

**① 내 PC : Readiness Probe 경로 오류 발생**

- Probe 경로를 존재하지 않는 주소로 변경 (13주차 2강 4단계 `deployment.yaml`의 `readinessProbe.httpGet.path`)

```bash
cd ~/kcu-gitops
code -r .

git pull
sed -i 's|/actuator/health|/actuator/healthz|' petclinic/deployment.yaml

git diff
git commit -am "Incident 2: wrong probe path"
git push
```

**② CloudShell : 증상 확인**

- 새 Pod가 Running이지만 계속 **0/1**

```bash
kubectl get pods -n petclinic
```

**③ 원인 확인**

```bash
kubectl describe pod <새 Pod 이름> -n petclinic | grep -i readiness
```

```text
Readiness:  http-get http://:8080/actuator/healthz delay=20s ...
Warning  Unhealthy  ...  Readiness probe failed: HTTP probe failed with statuscode: 404
```

**④ 복구 (내 PC)**

```bash
git revert HEAD --no-edit
git push
```

**⑤ 복구 확인**

- Argo CD **Synced · Healthy**
- 기존 정상 Pod 2개만 **READY 1/1**인 것을 확인한 후 다음 장애를 진행합니다.

### 3단계: 장애 ③ CI 빌드 실패

**① 내 PC : Dockerfile 오류 발생**

- Dockerfile의 COPY 경로를 잘못 변경 (Maven은 `target/`에 JAR 생성)

```bash
cd ~/kcu-petclinic
code -r .

git pull

sed -i 's|COPY target/|COPY build/|' Dockerfile
git diff

git commit -am "Incident 3: broken Dockerfile"
git push
```

**② GitHub Actions에서 오류 확인**

- GitHub → **Actions → PetClinic CI** : 빨간 X → **Build and push image** Step 로그 확인
- `build` 경로를 찾을 수 없다는 오류 (문구는 Docker 버전에 따라 조금 다름)
- 이후 Step(GitOps Checkout · 태그 갱신) **실행 안 됨** → kcu-gitops 변경 없음 → **서비스 영향 없음**

<img width="920" height="404" alt="GitHub Actions Dockerfile COPY 경로 오류로 CI 빌드 실패" src="https://github.com/user-attachments/assets/25124119-8b64-46c7-8e14-55b19abba210" />

**③ 복구 (내 PC)**

- CI 성공 → 새 태그 Commit → Argo CD 자동 배포

```bash
git revert HEAD --no-edit
git push
```

### 4단계: 복구 확인 · 사후 분석

**① 최종 상태 확인**

- **Synced · Healthy**
- Pod 2개 모두 **READY 1/1**

```bash
kubectl get applications -n argocd
kubectl get pods -n petclinic
```

**② 장애 · 복구 이력 확인 (내 PC)**

```bash
git -C ~/kcu-gitops pull
git -C ~/kcu-gitops log --oneline -6
```

```text
xxxxxxx Deploy petclinic e5f6a7b
xxxxxxx Revert "Incident 2: wrong probe path"
xxxxxxx Incident 2: wrong probe path
xxxxxxx Revert "Incident 1: wrong image tag"
xxxxxxx Incident 1: wrong image tag
xxxxxxx Deploy petclinic 3f9c1a7    ← 1강 환경 복원 때 CI가 만든 Commit
```

**③ Argo CD 배포 이력 확인**

- Argo CD UI → **[HISTORY AND ROLLBACK]** : 장애 · 복구 배포 이력 확인

---

## 리소스 정리 (순서 중요)

> [!WARNING]
> 로드밸런서(ELB)가 남아 있으면 VPC 삭제가 막혀 **EKS 클러스터 삭제가 실패**할 수 있습니다. 반드시 아래 순서대로 진행합니다.

### ① Application 삭제 (CloudShell)

13주차 Application에 설정한 `resources-finalizer.argocd.argoproj.io`에 의해 Deployment · Pod · Service(ELB) 함께 삭제

```bash
kubectl delete application petclinic -n argocd
kubectl get all -n petclinic
```

### ② Argo CD UI용 로드밸런서 · 네임스페이스 삭제

```bash
kubectl delete svc argocd-server -n argocd
kubectl delete namespace argocd petclinic
```

### ③ EKS 클러스터 삭제 (약 10~15분)

AWS 콘솔 **EC2 → 로드 밸런서**에서 `kcu-petclinic-lb` · `kcu-argocd-lb` (Name 태그)가 삭제된 것을 확인한 후 실행

```bash
eksctl delete cluster -f ~/kcu-infra/cluster.yaml
```

### ④ ECR 리포지토리 삭제 (이미지 포함)

```bash
aws ecr delete-repository --repository-name petclinic --force --region ap-northeast-2
```

### ⑤ 인증 정보 정리

- IAM 사용자 `github-actions-ci` → 액세스 키 비활성화 → 삭제 → 사용자 삭제
- GitHub → Fine-grained tokens → `kcu-gitops-token` **[Delete]**
- `kcu-petclinic` → Secrets 3개 삭제 · Actions → PetClinic CI → **Disable workflow**

### ⑥ 정리 확인

- EC2 → 이번 실습에서 생성한 **로드 밸런서** · **인스턴스** 없음
- EKS 클러스터 목록 비어 있음 · CloudFormation `eksctl-kcu-eks-*` 스택 삭제 완료
- ECR `petclinic` 리포지토리 없음

---

