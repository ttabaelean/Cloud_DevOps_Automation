
# 13주차 Hands-on Labs: 클라우드 DevOps 자동화 구현

## 실습 환경

| 항목 | 구성 |
|---|---|
| 클러스터 작업 | AWS CloudShell (서울 리전 `ap-northeast-2`) · `kubectl` · `eksctl` |
| Git 작업 | Windows 10/11 + VSCode 터미널(Git Bash) |
| EKS | 설정 파일 `~/kcu-infra/cluster.yaml`로 생성 · 클러스터 `kcu-eks` / 노드 `t3.medium` · `t3a.medium` × 2 (Spot) |
| ECR | 리포지토리 `petclinic` (scanOnPush 사용) · 태그 = Git Commit SHA |
| Argo CD | 네임스페이스 `argocd` · 공식 `stable` 매니페스트 · Application은 YAML(`apps/petclinic.yaml`)로 생성 |
| 저장소 | 애플리케이션 저장소 `kcu-petclinic` (이번 주 생성, Public) / GitOps 저장소 `kcu-gitops` (12주차, Public) |
| 애플리케이션 | Spring PetClinic (Java 17 · Spring Boot · 포트 8080) |

> [!WARNING]
> **EKS는 프리티어가 없는 유료 서비스**입니다. 1강~3강 실습을 가능하면 하루에 이어서 진행하고, 마지막의 **리소스 정리**를 반드시 수행합니다.
>
> 중간에 멈춰야 한다면 리소스 정리 순서대로 삭제한 뒤 다음에 `cluster.yaml`로 다시 생성합니다.

---

## 13주 1강 Hands-on Labs : EKS 기반 인프라 구축

### 1단계: CloudShell 준비

- AWS 콘솔 오른쪽 위 리전이 **아시아 태평양(서울)** 인지 확인 → **CloudShell** 실행
- 계정 확인 후 계정 ID를 변수에 저장

```bash
aws sts get-caller-identity
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo $ACCOUNT_ID
```

- eksctl · kubectl 확인

```bash
eksctl version
kubectl version --client
```

- eksctl이 없으면 설치 (12주차 1단계와 동일)

```bash
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz
mkdir -p $HOME/bin && mv eksctl $HOME/bin/
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
eksctl version
```

> [!NOTE]
> CloudShell의 홈 폴더(`$HOME`)는 세션이 끝나도 유지됩니다. `ACCOUNT_ID` 같은 변수는 세션이 끝나면 사라지므로 다시 접속하면 위 export 명령을 다시 실행합니다.

### 2단계: EKS 인프라 구축을 위한 cluster.yaml 작성

- **ClusterConfig** : eksctl이 읽는 클러스터 설정 파일. 명령 옵션 대신 YAML로 클러스터를 선언 (IaC)
- 작업 폴더 생성 후 설정 파일 작성

```bash
mkdir -p ~/kcu-infra && cd ~/kcu-infra

cat > cluster.yaml << 'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: kcu-eks
  region: ap-northeast-2
managedNodeGroups:
  - name: kcu-nodegroup
    instanceTypes: ["t3.medium", "t3a.medium"]
    spot: true
    desiredCapacity: 2
    minSize: 1
    maxSize: 2
    volumeSize: 20
EOF

cat cluster.yaml
```

> [!NOTE]
> 이 파일은 14주차에서도 사용하므로 지우지 않습니다.

### 3단계: EKS 클러스터 생성

- 설정 미리 보기 (실제로 생성하지 않음, 기본값까지 채워진 최종 설정 확인)

```bash
eksctl create cluster -f cluster.yaml --dry-run | head -20
```

- 클러스터 생성 (약 15~20분)

```bash
eksctl create cluster -f cluster.yaml
```

```text
... EKS cluster "kcu-eks" in "ap-northeast-2" region is ready
```

- Worker Node 2대 Ready · Spot 확인

```bash
kubectl get nodes -L eks.amazonaws.com/capacityType
```

```text
NAME                                               STATUS   ROLES    AGE   VERSION   CAPACITYTYPE
ip-192-168-xx-xx.ap-northeast-2.compute.internal   Ready    <none>   3m    v1.xx     SPOT
ip-192-168-yy-yy.ap-northeast-2.compute.internal   Ready    <none>   3m    v1.xx     SPOT
```

> [!NOTE]
> 생성 중에는 CloudShell 창을 닫지 않습니다.
>
> 오류가 나면 **CloudFormation** 콘솔의 `eksctl-kcu-eks-*` 스택 이벤트에서 원인을 확인합니다.
>
> Spot 용량 부족 오류가 나면 `eksctl delete cluster -f cluster.yaml`로 지운 뒤 `spot: true` 줄을 삭제하고 다시 생성합니다.
>
> 다음 날 이어서 할 때는 `aws eks update-kubeconfig --region ap-northeast-2 --name kcu-eks`로 연결을 다시 설정합니다.

### 4단계: ECR 리포지토리 생성

- `petclinic` 리포지토리 생성
  - `--image-scanning-configuration scanOnPush=true`
    - 이미지가 Push될 때마다 ECR이 이미지 안의 OS 패키지를 검사해서, 알려진 보안 취약점(CVE)이 있는지 스캔
    - 기본 스캔은 무료
    - CI/CD에 보안 점검을 자동으로 끼워 넣는 것(DevSecOps)의 가장 쉬운 예

```bash
aws ecr create-repository \
  --repository-name petclinic \
  --image-scanning-configuration scanOnPush=true \
  --region ap-northeast-2
```

- 리포지토리 URI 확인

```bash
aws ecr describe-repositories --repository-names petclinic \
  --query 'repositories[0].repositoryUri' --output text
```

```text
123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic
```

> [!NOTE]
> AWS 콘솔 **ECR → 프라이빗 리포지토리**에서도 확인할 수 있습니다.

### 5단계: Argo CD 설치 · 로그인

- Argo CD 설치 (12주차와 동일)

```bash
# argocd namespace 생성
kubectl create namespace argocd

# argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

watch -n 1 kubectl get pods -n argocd
# 종료는 Ctrl + C
```

- UI 외부 노출 · 접속 주소 확인 (주소가 열리기까지 2~3분)

```bash
# argocd server에 외부 접속 가능하도록 LoadBalancer 타입으로 수정
# EC2 > 로드밸런서 생성
kubectl get svc -n argocd

# 로드밸런서를 구분할 Name 태그 지정 (외부 노출 전에 설정)
kubectl annotate svc argocd-server -n argocd \
  service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags='Name=kcu-argocd-lb' \
  --overwrite

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# argocd 접속 URL 수집 및 확인
ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo https://$ARGOCD_URL
```

- Argo CD 관리자(admin) 초기 비밀번호 확인

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

- 브라우저에서 `https://<접속 주소>` → 인증서 경고 **[고급]** → **[계속 진행]**
- **Username** : `admin` / **Password** : 위에서 출력된 값 → **[SIGN IN]**

---

## 13주 2강 Hands-on Labs : PetClinic CI 파이프라인 구축

PetClinic 소스를 Push하면 GitHub Actions가 JAR 빌드 → 이미지 빌드 · ECR Push → kcu-gitops 이미지 태그 갱신까지 자동 수행

**① IAM · Access Key → ② 저장소 · Token · Secrets → ③ 소스 · Dockerfile → ④ kcu-gitops 매니페스트 → ⑤ ci.yml → ⑥ Push → ⑦ 결과 확인**

### 1단계: IAM 사용자 · Access Key (AWS 콘솔)

- **IAM → 사용자 → [사용자 생성]**
  - 사용자 이름 : `github-actions-ci` (콘솔 로그인 체크 안 함)
  - 권한 옵션 : **직접 정책 연결** → `AmazonEC2ContainerRegistryPowerUser`
- 생성한 사용자 → **보안 자격 증명 → [액세스 키 만들기]**
  - 사용 사례 : **AWS 외부에서 실행되는 애플리케이션**
  - **[.csv 파일 다운로드]**로 Access Key ID · Secret access key 보관

> [!WARNING]
> Secret access key는 생성 화면에서 **한 번만** 표시됩니다. 파일에 적거나 Commit하지 않습니다.
>
> 이번 실습은 교육용 Access Key 방식입니다. 실무에서는 장기 액세스 키 대신 GitHub OIDC와 IAM Role을 사용하는 방식을 권장합니다.

### 2단계: GitHub Repository · Token · Secrets (GitHub 웹) 생성

> [!NOTE]
> **GITHUB_TOKEN** : Workflow 실행 시 자동 제공. Workflow가 실행된 저장소(kcu-petclinic)에만 권한
>
> **Fine-grained Token** : 저장소 · 권한 · 만료를 좁게 지정하는 개인 토큰 → kcu-gitops에 Push할 때 사용

- **GitHub Repository(애플리케이션 저장소) 생성** : GitHub Repositories → **[New]**
  - Repository name : `kcu-petclinic`
  - Choose Visibility : **Public**
  - Add README **체크 안 함** (빈 저장소)
  - **[Create repository]**

- **Fine-grained Token 생성**
  - GitHub 저장소에 접근할 때 사용하는 인증 토큰
  - CI가 `kcu-gitops`를 수정할 때 사용할 인증 정보로 사용
  - 오른쪽 상단 프로필 → **Settings → Developer settings → Personal access tokens → Fine-grained tokens → [Generate new token]**

| 항목 | 입력값 | 의미 |
|---|---|---|
| Token name | `kcu-gitops-token` | |
| Expiration | `30 days` | 30일 후 만료되어 사용할 수 없음 |
| Repository access | **Only select repositories** → `kcu-gitops` | 선택한 저장소에만 접근 허용 · CI가 수정할 대상 저장소 |
| Permissions | Repository permissions → **Contents : Read and write** → Metadata : Read-only | 저장소 파일 읽기와 Commit·Push 허용 · 저장소 기본 정보 조회 |

- **[Generate token]** → `github_pat_...` 값 복사 (한 번만 표시)

- **Secrets 등록** : `kcu-petclinic` → **Settings → Secrets and variables → Actions → [New repository secret]**

| Name | Secret |
|---|---|
| `AWS_ACCESS_KEY_ID` | 1단계 Access Key ID |
| `AWS_SECRET_ACCESS_KEY` | 1단계 Secret access key |
| `GITOPS_TOKEN` | Fine-grained Token (`github_pat_...`) |

### 3단계: PetClinic 소스 · Dockerfile (내 PC - VSCode)

- 원본 PetClinic을 내려받아 내 저장소로 초기화
  - `.git` 삭제 : 원본 Commit 이력 제거
  - `.github` 삭제 : 원본 Workflow가 우리 저장소에서 실행되지 않도록 제거

```bash
cd ~

# 원본 PetClinic을 내려받아 내 저장소로 복제
git clone https://github.com/spring-projects/spring-petclinic.git kcu-petclinic

cd kcu-petclinic

# 원본 PetClinic에 포함된 github 구성 정보 삭제
rm -rf .git .github

# 현재 폴더를 새로운 Git 저장소로 만들고, Origin 브랜치 이름을 main으로 지정
git init -b main
ls
code -r .
```

- Dockerfile 작성 (프로젝트 최상위, `pom.xml`과 같은 위치)

```bash
cat > Dockerfile << 'EOF'
FROM eclipse-temurin:17-jre
WORKDIR /app

#JAR은 CI Runner에서 Maven으로 빌드하고, Dockerfile은 만들어진 JAR만 복사
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
EOF
```

### 4단계: kcu-gitops에 PetClinic 매니페스트 추가 (내 PC VSCode)

- 12주차 kcu-gitops 최신화 후 폴더 생성 (`apps`는 3강에서 사용)

```bash
cd ~/kcu-gitops
git pull
mkdir -p petclinic apps
ls
```

- deployment.yaml 작성

```bash
cat > petclinic/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: petclinic
spec:
  replicas: 2
  selector:
    matchLabels:
      app: petclinic
  template:
    metadata:
      labels:
        app: petclinic
    spec:
      containers:
        - name: petclinic
          # 임시값. CI가 실행되면 실제 ECR 주소(Commit SHA 태그)로 자동 변경
          image: petclinic:initial
          ports:
            - containerPort: 8080
          # Pod가 트래픽을 받을 준비가 되었는지 확인 (Spring Boot Actuator)
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              memory: 1Gi
EOF
```

- service.yaml 작성

```bash
cat > petclinic/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: petclinic-svc
  annotations:
    # AWS 로드밸런서에 구분용 Name 태그 추가
    service.beta.kubernetes.io/aws-load-balancer-additional-resource-tags: "Name=kcu-petclinic-lb"
spec:
  selector:
    app: petclinic
  ports:
    - port: 80
      targetPort: 8080
  type: LoadBalancer
EOF
```

- Commit & Push

```bash
git add .
git commit -m "Add petclinic manifests"
git push
```

### 5단계: ci.yml 작성 (내 PC · kcu-petclinic)

> [!NOTE]
> **Workflow 구성 (Job 1개 · Step 8개)**
>
> - ①~③ JAR 빌드 : Checkout → JDK 17 → Maven Build
> - ④~⑥ 이미지 : AWS 인증 → ECR 로그인 → Docker Build · Push (태그 = Commit SHA)
> - ⑦~⑧ 태그 갱신 : kcu-gitops Checkout(GITOPS_TOKEN) → deployment.yaml image 변경 → Commit · Push

- Workflow 폴더 생성

```bash
cd ~/kcu-petclinic
mkdir -p .github/workflows
```

- `.github/workflows/ci.yml` 작성

```bash
cat > .github/workflows/ci.yml << 'EOF'
name: PetClinic CI

# main 브랜치에 Push하면 Workflow 실행
on:
  push:
    branches: [ main ]

env:
  AWS_REGION: ap-northeast-2
  ECR_REPOSITORY: petclinic
  GITOPS_REPO: ${{ github.repository_owner }}/kcu-gitops

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # ①~③ 소스를 내려받고 Java 환경을 준비한 뒤 실행 JAR 생성
      # ① kcu-petclinic 소스를 Runner의 작업 폴더로 내려받기
      - name: Checkout
        uses: actions/checkout@v6

      # ② Temurin JDK 17 설치 및 Maven 의존성 캐시 설정
      - name: Set up JDK 17
        uses: actions/setup-java@v6
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      # ③ Maven Wrapper에 실행 권한을 부여하고 소스를 빌드
      # 테스트 실행은 생략하고 target/ 폴더에 JAR 파일 생성
      - name: Build JAR with Maven
        run: |
          chmod +x mvnw
          ./mvnw -B package -DskipTests

      # ④~⑥ AWS 인증 후 컨테이너 이미지를 빌드하여 ECR에 저장
      # ④ GitHub Secrets의 Access Key로 AWS CLI 인증 환경 설정
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v6
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # ⑤ Docker가 이미지를 Push할 수 있도록 Amazon ECR에 로그인
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # ⑥ Dockerfile로 이미지를 빌드하고 Commit SHA를 태그로 붙여 ECR에 Push
      # 이미지 전체 주소를 GITHUB_ENV에 기록하여 다음 Step에서도 사용
      - name: Build and push image
        env:
          REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          IMAGE=$REGISTRY/$ECR_REPOSITORY:$GITHUB_SHA
          docker build -t $IMAGE .
          docker push $IMAGE
          echo "IMAGE=$IMAGE" >> $GITHUB_ENV

      # ⑦~⑧ GitOps 저장소의 배포 이미지 주소를 변경하고 Commit · Push
      # ⑦ GITOPS_TOKEN으로 인증하여 kcu-gitops를 gitops/ 폴더에 내려받기
      - name: Checkout GitOps repo
        uses: actions/checkout@v6
        with:
          repository: ${{ env.GITOPS_REPO }}
          token: ${{ secrets.GITOPS_TOKEN }}
          path: gitops

      # ⑧ deployment.yaml의 image를 새 ECR 이미지 주소로 변경
      # Commit 작성자를 설정하고, 변경 사항이 있으면 Commit · Push
      # 이후 Argo CD가 Git 변경을 감지하여 EKS에 자동 배포
      - name: Update image tag
        working-directory: gitops
        run: |
          # 배포 YAML의 이미지 주소를 새 ECR 이미지 주소로 변경
          sed -i "s|image: .*|image: $IMAGE|" petclinic/deployment.yaml
          # Commit 작성자 이름 설정
          git config user.name "github-actions[bot]"
          # Commit 작성자 이메일 설정
          git config user.email "github-actions@github.com"
          # 변경 사항이 없으면 성공으로 종료
          git diff --quiet && exit 0
          # 수정된 파일을 소스 Commit SHA 앞 7자리를 포함한 메시지로 Commit
          git commit -am "Deploy petclinic ${GITHUB_SHA::7}"
          # GitHub의 kcu-gitops 저장소에 Push
          git push
EOF
```

- 주요 항목
  - `cache: maven` : Maven 라이브러리를 캐시해 다음 실행부터 빌드 시간 단축
  - `chmod +x mvnw` : Windows에서 만든 저장소는 실행 권한 정보가 빠질 수 있어 추가
  - `$GITHUB_ENV` : 앞 Step에서 만든 `IMAGE` 값을 다음 Step에서 사용
  - `git diff --quiet && exit 0` : 바뀐 내용이 없으면(같은 Commit 재실행) 그대로 성공 종료

### 6단계: Push → CI 실행 (내 PC)

- 포함된 파일 확인 후 Commit · 원격 저장소 연결 · Push

```bash
cd ~/kcu-petclinic

git add .
git status
git commit -m "Add PetClinic source, Dockerfile and CI"
git remote add origin https://github.com/<Username>/kcu-petclinic.git
git push -u origin main
```

- GitHub `kcu-petclinic` → **Actions** 탭 → **PetClinic CI** 실행 확인 (약 3~5분, 첫 실행은 더 오래 걸림)

### 7단계: 결과 확인

- Actions → **build** Job → 모든 Step 초록 체크

<img width="930" height="710" alt="PetClinic CI GitHub Actions 실행 성공" src="https://github.com/user-attachments/assets/cfd1856e-3cd5-4f89-81b6-c2a3d7fe4a05" />

- 이미지 태그 확인 (내 PC VSCode)

```bash
git log -1 --format=%H
```

- ECR `petclinic` → 이미지 태그가 Commit SHA인지 확인 → 이미지 클릭 → **취약성** 스캔 결과 확인

<img width="925" height="281" alt="ECR PetClinic 이미지 태그와 스캔 결과" src="https://github.com/user-attachments/assets/2014b1ce-e8a4-412a-ae22-75dac2878923" />

- GitHub `kcu-gitops` → 최신 Commit이 `Deploy petclinic xxxxxxx` (작성자 github-actions[bot]) 인지 확인

<img width="942" height="671" alt="GitOps 저장소 자동 Commit 확인" src="https://github.com/user-attachments/assets/b8adce1e-b652-4238-879a-38aa3a7dd545" />

- 내 PC의 kcu-gitops에 CI 변경 내용 받기

```bash
cd ~/kcu-gitops
git pull
grep image: petclinic/deployment.yaml
```

```text
image: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/petclinic:3f9c1a7e....(40자리)
```

---

## 13주 3강 Hands-on Labs : PetClinic GitOps 자동 배포

- Application을 YAML로 만들어 PetClinic을 EKS에 배포
- 코드 Push → CI → 태그 Commit → Argo CD 자동 배포 → 롤백까지 확인
- 마지막에 리소스를 순서대로 정리

### 1단계: Application YAML 작성 (내 PC)

- 반드시 `git pull` 먼저 실행해서 CI가 만든 Commit 받기

```bash
cd ~/kcu-gitops
git pull
cat petclinic/deployment.yaml
```

- Argo CD의 **Application YAML 작성**

```bash
cat > apps/petclinic.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: petclinic
  namespace: argocd
  # Application을 삭제하면 Argo CD가 만든 Deployment · Service(ELB)도 함께 삭제
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<Username>/kcu-gitops.git
    targetRevision: HEAD
    path: petclinic
  destination:
    server: https://kubernetes.default.svc
    namespace: petclinic
  syncPolicy:
    # Auto Sync, Prune, Self Heal 사용
    automated: { prune: true, selfHeal: true }
    syncOptions: [ CreateNamespace=true ]
EOF
```

- VSCode에서 `apps/petclinic.yaml`의 `<Username>`을 내 GitHub ID로 변경

```bash
code -r .
```

- Git Push

```bash
grep repoURL apps/petclinic.yaml

git add apps/petclinic.yaml
git commit -m "Add petclinic Application"
git push
```

### 2단계: Application 생성 (CloudShell)

- kcu-gitops를 Clone해서 Application 적용 (Public 저장소라 인증 불필요)

```bash
cd ~
git clone https://github.com/<Username>/kcu-gitops.git
```

> [!NOTE]
> 이미 Clone했다면 `cd ~/kcu-gitops && git pull` 후 apply 합니다.
>
> SYNC 버튼을 누르지 않아도 Auto Sync로 바로 배포됩니다.

```bash
# Application 적용
kubectl apply -f kcu-gitops/apps/petclinic.yaml

kubectl get applications -n argocd
```

```text
NAME        SYNC STATUS   HEALTH STATUS
petclinic   Synced        Healthy
```

### 3단계: 배포 · 접속 확인

- Argo CD UI → `petclinic` → 리소스 트리 : `petclinic-svc` · `petclinic(deploy)` → `rs` → `pod × 2` · **Synced / Healthy**

<img width="1520" height="523" alt="Argo CD PetClinic Application 배포 상태" src="https://github.com/user-attachments/assets/557bddfc-d73b-4e06-a166-c26cf39313b4" />

- Pod Ready 확인 (Java 기동 약 30초~1분, 0/1 → 1/1)

```bash
kubectl get pods -n petclinic
```

```text
petclinic-xxxxxxxxxx-aaaaa   0/1   Running   0   20s
petclinic-xxxxxxxxxx-aaaaa   1/1   Running   0   45s
```

- 접속 주소 확인 → 브라우저에서 `http://<EXTERNAL-IP>` (2~3분 소요)

```bash
kubectl get svc petclinic-svc -n petclinic
```

- PetClinic 화면에서 **FIND OWNERS** · **VETERINARIANS** 메뉴 확인 (내장 H2 DB 예제 데이터)

<img width="1073" height="506" alt="Spring PetClinic 웹 서비스 접속 확인" src="https://github.com/user-attachments/assets/25eef0c4-c866-4799-81a8-1137a2c773aa" />

### 4단계: PetClinic 소스 코드 수정 후 Push (내 PC)

> [!NOTE]
> **동작 흐름**
>
> PetClinic 소스 수정 → kcu-petclinic에 Commit & Push → CI 자동 실행
>
> 1. CI가 새 이미지를 빌드 → ECR에 새 태그로 Push
> 2. CI가 `kcu-gitops`의 `deployment.yaml`을 새 이미지 태그로 수정 → Commit·Push
>
> Argo CD가 `kcu-gitops` 변경 감지 → 새 버전 이미지로 배포

- PetClinic 소스 수정
  - VSCode에서 `src/main/resources/templates/welcome.html` 열기

```bash
cd ~/kcu-petclinic
code -r .
```

- `<h2 th:text="#{welcome}">Welcome</h2>` 바로 아래에 한 줄 추가 후 저장

```html
<h2 th:text="#{welcome}">Welcome</h2>
<h3>v2 : GitHub Actions + Argo CD</h3>
```

- Commit & Push → CI 자동 실행

```bash
git diff
git commit -am "Update welcome page v2"
git push
```

### 5단계: 자동 배포 흐름 확인

> [!NOTE]
> **동작 흐름**
>
> PetClinic 소스 수정 → kcu-petclinic에 Commit & Push → CI 자동 실행
>
> 1. CI가 새 이미지를 빌드 → ECR에 새 태그로 Push
> 2. CI가 `kcu-gitops`의 `deployment.yaml`을 새 이미지 태그로 수정 → Commit·Push
>
> Argo CD가 `kcu-gitops` 변경 감지 → 새 버전 이미지로 배포

- ① GitHub **Actions** → PetClinic CI 성공 (약 3~5분)
- ② GitHub **kcu-gitops** → `Deploy petclinic xxxxxxx` Commit 생성 (CI 작성)
- ③ Argo CD UI → `petclinic` : `OutOfSync` → `Synced` (늦으면 **[REFRESH]**)
- ④ Rolling Update 관찰 (CloudShell) → 새 Pod Ready 후 기존 Pod 종료

```bash
kubectl get pods -n petclinic -w
```

> [!NOTE]
> Pod 교체 과정을 확인한 후 `Ctrl + C`를 눌러 종료합니다.

- ⑤ 배포된 이미지 태그 확인 → 4단계 Commit ID와 비교

```bash
kubectl get deploy petclinic -n petclinic -o jsonpath='{..image}'; echo
```

- ⑥ 브라우저 새로고침 → **v2 : GitHub Actions + Argo CD** 문구 확인

<img width="827" height="530" alt="PetClinic v2 GitHub Actions Argo CD 자동 배포 결과" src="https://github.com/user-attachments/assets/7ab79a0f-051c-4496-bc56-53035f4ddcbb" />

### 6단계: git revert로 롤백 (내 PC)

> [!NOTE]
> **롤백: 두 번째 작업 변경 취소**
>
> - `kcu-gitops`에서 `git revert` : deployment.yaml의 image 태그가 이전 값으로 복구 → Push
> - Argo CD가 변경 감지 → **이전 이미지로 다시 배포**
> - `kcu-petclinic`의 **소스 코드는 그대로 유지**

- PetClinic 소스코드 수정 → kcu-petclinic Push
  - CI 동작 → Image 생성 → ECR에 new tag Push
  - CI 동작 → kcu-gitops의 deployment.yaml의 image 태그 수정
- Argo CD 동작 → 새 버전 이미지로 배포
- kcu-gitops의 Commit 버전 이전으로 되돌리기
  - CI가 만든 태그 변경 Commit을 되돌림 → 이전 이미지로 복귀 (CI 재실행 없음)

```bash
cd ~/kcu-gitops
git pull
git log --oneline -3
git revert HEAD --no-edit
git push
```

```text
xxxxxxx (HEAD -> main) Revert "Deploy petclinic b2c3d4e"
xxxxxxx Deploy petclinic b2c3d4e
xxxxxxx Add petclinic Application
```

- Argo CD 자동 Sync → 브라우저 새로고침 → v2 문구가 없는 이전 화면
- Argo CD UI → **[HISTORY AND ROLLBACK]** → 배포 이력과 Revision 확인

> [!NOTE]
> 소스 저장소(kcu-petclinic)는 여전히 v2입니다. 다음에 kcu-petclinic에 Push하면 CI가 새 이미지를 다시 배포합니다.

---

## 리소스 정리 (순서 중요)

> [!WARNING]
> 로드밸런서(ELB)가 남아 있으면 VPC 삭제가 막혀 **EKS 클러스터 삭제가 실패**할 수 있습니다. 반드시 아래 순서대로 진행합니다.

### ① Application 삭제 (CloudShell)

- finalizer로 Deployment · Pod · Service(ELB) 함께 삭제

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

```bash
eksctl delete cluster -f ~/kcu-infra/cluster.yaml
```

### ④ ECR 리포지토리 삭제

- AWS 콘솔 → **프라이빗 레지스트리 → 리포지토리**에서 `petclinic` 선택
- 이미지 삭제
- `petclinic` 리포지토리 **삭제**

### ⑤ 인증 정보 정리

- IAM 사용자 `github-actions-ci` → 액세스 키 비활성화 → 삭제 → 사용자 삭제
- GitHub 프로필 → Settings → Developer settings → Fine-grained tokens → `kcu-gitops-token` **[Delete]**
- `kcu-petclinic` → Settings → Secrets and variables → Actions → Secret 3개 삭제 · Actions → PetClinic CI → **Disable workflow**

### ⑥ 정리 확인

- EC2 → **로드 밸런서** · **인스턴스**가 남아 있지 않은지 확인
- EKS 클러스터 목록 비어 있음 · CloudFormation `eksctl-kcu-eks-*` 스택 삭제 완료
- ECR `petclinic` 리포지토리 없음

> [!NOTE]
> GitHub `kcu-petclinic` · `kcu-gitops` 저장소는 비용이 들지 않으므로 그대로 유지합니다.

---

