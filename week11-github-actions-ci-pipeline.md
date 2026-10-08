

# 11주차 Hands-on Labs: GitHub Actions를 활용한 CI 파이프라인 구축

## 실습 환경

| 항목 | 구성 |
|---|---|
| 실습 PC | Windows 10/11 + VSCode 터미널(Git Bash) |
| 실습 저장소 | 로컬 `~/kcu-git-lab` ↔ GitHub `kcu-git-lab` (10주차 저장소) |
| Runner | GitHub-hosted Runner `ubuntu-latest` |
| Workflow 파일 | `.github/workflows/` (`hello.yml` → `docker-build.yml` → `container-ci.yml`) |
| AWS | Region `ap-northeast-2` / ECR Repository `webapp` |

> [!WARNING]
> 내 PC에는 Docker를 설치하지 않아도 됩니다. Docker Build·Push는 모두 GitHub의 Runner에서 실행됩니다.
>
> 3강에서 만드는 **AWS Access Key는 절대 파일에 적거나 Commit하지 않습니다.**

---

## 11주 1강 Hands-on Labs : 첫 번째 GitHub Actions Workflow 실행

> 10주차 `kcu-git-lab` 저장소에 가장 간단한 Workflow(`hello.yml`)를 추가하고 Push
>
> Push 이벤트로 GitHub Actions가 자동 실행되는 것을 Actions 탭에서 확인
>
> local PC(hello.yml 작성) → git push → GitHub(Push Event) → Runner(ubuntu) 실행

### 1단계: 실습 저장소 준비

> [!NOTE]
> **명령어 정리**
>
> - `git switch main` : main 브랜치로 이동
> - `git pull` : GitHub의 최신 Commit을 내 PC로 가져오기
> - `git clone <주소>` : 로컬 폴더가 없을 때 GitHub 저장소를 새로 복제

- 저장소로 이동 후 최신 상태로 맞추기

```bash
cd ~/kcu-git-lab
git switch main
git pull
ls
git status
```

```text
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

### 2단계: Workflow 파일 작성 (hello.yml)

> [!NOTE]
> **용어 정리**
>
> - `.github/workflows/` : GitHub가 Workflow 파일을 찾는 위치 (폴더 이름이 정확해야 실행됨)
> - `name` : Workflow 이름 (Actions 탭에 표시)
> - `on: push` : 실행 조건(Event) — 모든 브랜치의 push
> - `jobs` → `runs-on` : 작업 단위와 실행 환경(Runner)
> - `steps` → `uses` : 만들어진 Action 사용 / `run` : 명령어 직접 실행

- Workflow 폴더 생성

```bash
mkdir -p .github/workflows
```

- `hello.yml` 작성

```bash
cat > .github/workflows/hello.yml << 'EOF'
name: Hello CI

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest

    steps:
      # 저장소 코드를 Runner(가상 머신)로 가져오기
      - name: Checkout
        uses: actions/checkout@v4

      # 인사 문구를 로그에 출력
      - name: Hello
        run: echo "Hello GitHub Actions"
EOF
```

- 파일 확인

```bash
cat .github/workflows/hello.yml
```

### 3단계: Commit & Push

> [!NOTE]
> **명령어 정리**
>
> - `git add` → `git commit` → `git push` : 10주차와 같은 흐름
> - push가 곧 GitHub Actions의 **Event**가 됨

- Workflow 파일을 Commit 후 GitHub로 Push

```bash
git add .github/workflows/hello.yml
git commit -m "Add hello workflow"
git push
```

### 4단계: Actions 탭에서 실행 결과 확인

> [!NOTE]
> **확인 포인트**
>
> - 🟡 노란 원 : 실행 중 / ✅ 초록 체크 : 성공 / ❌ 빨간 X : 실패
> - `Set up job` · `Complete job` 은 GitHub가 자동으로 추가하는 Step (Runner 준비·정리)

- GitHub 저장소 → **Actions** 탭 → **Hello CI** → 실행 항목(Commit 메시지 `Add hello workflow`) 클릭
- **hello** Job 클릭 → Step이 모두 성공했는지 확인

```text
✅ Set up job
✅ Checkout
✅ Hello
✅ Post Checkout
✅ Complete job
```

- **Hello** Step을 펼쳐 로그 확인

```text
Hello GitHub Actions
```

### 5단계: Workflow 수정 후 자동 실행 다시 확인

> [!NOTE]
> **용어 정리**
>
> - `${{ github.ref_name }}` : Workflow를 실행시킨 브랜치 이름
> - `${{ github.sha }}` : Workflow를 실행시킨 Commit ID (3강에서 이미지 Tag로 사용)
> - `run: |` : 여러 줄 명령을 차례로 실행

- `hello.yml` 맨 아래에 **Show Info** Step을 추가해 파일을 다시 작성 (VSCode에서 직접 추가해도 됨, `- name` 들여쓰기는 위 Step과 같은 6칸)

```bash
vi .github/workflows/hello.yml
```

```yaml
name: Hello CI

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      # 저장소 코드를 Runner(가상 머신)로 가져오기
      - name: Checkout
        uses: actions/checkout@v4

      # 인사 문구를 로그에 출력
      - name: Hello
        run: echo "Hello GitHub Actions"

      # 추가-실행을 일으킨 브랜치 이름과 Commit ID 출력
      - name: Show Info
        run: |
          echo "Branch : ${{ github.ref_name }}" # 브랜치 ID
          echo "Commit : ${{ github.sha }}" # Commit ID
```

- 저장 후 Push → Actions 탭에 **새 실행 항목이 자동으로 추가**되는지 확인

```bash
git add .
git commit -m "Add show info step"
git push
```

- **Show Info** Step 로그 확인 → 로컬 Commit ID와 비교

```text
Branch : main                   <- Workflow를 실행시킨 브랜치 이름 (github.ref_name)
Commit : 3f9c1a7e...(40자리)     <- Workflow를 실행시킨 Commit ID (github.sha), 아래 git log 결과와 같아야 함
```

```bash
git log -1 --format=%H
```

---

## 11주 2강 Hands-on Labs : Docker 이미지 Build Workflow 작성

> Dockerfile(httpd 기반)과 index.html을 준비하고, main 브랜치에 push하면 Runner가 Docker 이미지를 자동으로 Build
>
> docker-build.yml : Checkout → Docker 확인 → Docker Build
>
> feature 브랜치 push로 `branches: main` 조건 동작 확인

### 1단계: 저장소 구조 확인

> [!NOTE]
> **명령어 정리**
>
> - `ls -a` : 숨김 파일·폴더까지 포함해 목록 보기
> - `find . -path ./.git -prune -o -type f -print` : `.git` 을 제외한 전체 파일 목록

- 현재 저장소 파일 확인

```bash
cd ~/kcu-git-lab

ls
```

```text
about.html  contact.html  index.html  README.md
```

```bash
ls .github/workflows/
```

```text
hello.yml
```

- 이번 강의에서 추가할 파일

```text
kcu-git-lab/
├── Dockerfile                    ← 추가
├── index.html                    ← 수정
└── .github/
    └── workflows/
        ├── hello.yml
        └── docker-build.yml      ← 추가
```

### 2단계: Dockerfile 작성

> [!NOTE]
> **용어 정리**
>
> - `FROM httpd:latest` : Apache 웹 서버 이미지를 기반으로 사용
> - `COPY index.html /usr/local/apache2/htdocs/` : 웹 페이지를 Apache 문서 폴더로 복사
> - `EXPOSE 80` : 컨테이너가 80번 포트를 사용함을 표시

- Dockerfile 작성 (파일 이름 대소문자 주의 : `Dockerfile`)

```bash
cat > Dockerfile << 'EOF'
FROM httpd:latest

COPY index.html /usr/local/apache2/htdocs/

EXPOSE 80
EOF
```

### 3단계: index.html 수정

- 웹 페이지 내용 변경

```bash
echo "<h1>GitHub Actions CI Lab</h1>" > index.html
cat index.html
```

### 4단계: Docker Build Workflow 작성 (docker-build.yml)

> [!NOTE]
> **용어 정리**
>
> - `on: push: branches: - main` : **main 브랜치에 push될 때만** 실행
> - `docker --version` : Runner에 Docker가 이미 설치되어 있는지 확인
> - `docker build -t webapp:latest .` : 현재 폴더의 Dockerfile로 `webapp:latest` 이미지 생성

- `docker-build.yml` 작성

```bash
cat > .github/workflows/docker-build.yml << 'EOF'
name: Docker Build CI

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # 저장소 코드(Dockerfile, index.html)를 Runner로 가져오기
      - name: Checkout
        uses: actions/checkout@v4

      # Runner에 Docker가 설치되어 있는지 버전 확인
      - name: Check Docker
        run: docker --version

      # Dockerfile로 webapp:latest 이미지 빌드
      - name: Build Docker Image
        run: docker build -t webapp:latest .
EOF
```

### 5단계: Push

- Dockerfile, index.html, Workflow를 함께 Commit 후 Push

```bash
git add .
git status
git commit -m "Add Docker build CI"
git push
```

> **참고**: `hello.yml` 도 `on: push` 이므로 이번 push에서는 **Hello CI와 Docker Build CI 두 개가 함께 실행**됩니다. Workflow 파일마다 따로 실행되는 것이 정상입니다.

### 6단계: Actions 결과 확인

> [!NOTE]
> **확인 포인트**
>
> - Docker Build는 내 PC가 아니라 **GitHub-hosted Runner(Ubuntu)** 에서 실행됨
> - Workflow가 끝나면 Runner가 삭제되므로 빌드한 이미지도 함께 사라짐 → 3강에서 ECR에 Push해 보관

- **Actions** 탭 → **Docker Build CI** → **build** Job
- Step 성공 여부 확인 : Checkout → Check Docker → Build Docker Image
- **Build Docker Image** 로그 마지막 부분 확인

```text
#X naming to docker.io/library/webapp:latest done
```

- **Workflow가 끝나면 Runner가 삭제되므로 빌드한 이미지도 함께 사라짐** → 3강에서 ECR에 Push해 보관

### 7단계: 브랜치 조건(branches) 동작 확인

> [!NOTE]
> **확인 포인트**
>
> - feature 브랜치 push → `on: push` 인 **Hello CI만 실행**
> - `branches: - main` 인 Docker Build CI는 실행되지 않음

- feature 브랜치를 만들어 Push

```bash
git switch -c feature-ci
git branch

echo "<p>feature test</p>" >> about.html
cat about.htm

git add about.html
git commit -m "Test feature branch"
git push -u origin feature-ci
```

- Actions 탭에서 **Hello CI만 새로 실행**되었는지 확인
- main 브랜치로 돌아오기

```bash
git switch main
```

---

## 11주 3강 Hands-on Labs : GitHub Actions → Amazon ECR Push

> 2강 Workflow를 확장해 AWS 인증 → ECR Login → Docker Build → Docker Push까지 자동화
>
> - AWS : IAM 사용자 Access Key 생성, ECR Repository(webapp) 생성
> - GitHub : Secrets에 Access Key 등록, container-ci.yml 작성
>
> index.html 수정 후 push → ECR에 Commit SHA 태그 이미지 확인

### 1단계: AWS 준비 (IAM Access Key · ECR Repository)

> [!NOTE]
> **용어 정리**
>
> - `IAM 사용자` : GitHub Actions가 AWS에 접근할 때 사용할 전용 계정
> - `AmazonEC2ContainerRegistryPowerUser` : ECR 이미지 Push·Pull 권한 (삭제·관리 권한 제외)
> - `Access Key ID` / `Secret Access Key` : 프로그램이 AWS에 로그인할 때 쓰는 아이디·비밀번호
> - `ECR Repository` : Docker 이미지를 저장하는 AWS 저장소 (8주차 실습과 동일)

#### ① IAM 사용자와 Access Key 생성

- AWS 콘솔 → **IAM** → **사용자** → **[사용자 생성]**
  - 사용자 이름 : `github-actions-ci`
  - AWS Management Console 액세스 : **선택하지 않음**
- 권한 설정 → **직접 정책 연결** → `AmazonEC2ContainerRegistryPowerUser` 선택 → **[사용자 생성]**

> ⚠️ 교육용 설정입니다. 이 정책은 계정의 모든 ECR 리포지토리에 권한을 주므로, 실무에서는 필요한 리포지토리로 범위를 좁힌 최소 권한을 적용해서 사용해야 합니다.

- 생성한 사용자 클릭 → **보안 자격 증명** 탭 → **[액세스 키 만들기]**
  - 사용 사례 : **AWS 외부에서 실행되는 애플리케이션** → [다음] → [액세스 키 만들기]
- **Access key ID** 와 **Secret access key** 를 복사하거나 **[.csv 파일 다운로드]**

> ⚠️ Secret access key는 이 화면에서 **한 번만** 보입니다. 메모장 등에 잠시 보관하고, 저장소 폴더 안에 저장하거나 Commit하지 않습니다.

#### ② ECR Repository 생성

- AWS 콘솔 → 리전 **아시아 태평양(서울) ap-northeast-2** 확인
- **Amazon ECR** → **Private registry** → **Repositories** → **[리포지토리 생성]**
  - 리포지토리 이름 : `webapp`
  - 나머지 설정은 기본값 → **[생성]**
- 리포지토리 URI 확인

```text
# 609417967491.dkr.ecr.ap-northeast-2.amazonaws.com/webapp
123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/webapp
```

### 2단계: GitHub Secrets 등록

> [!NOTE]
> **용어 정리**
>
> - `Secrets` : 비밀번호·키를 GitHub에 암호화해 저장하는 기능 (등록 후에는 값을 다시 볼 수 없음)
> - `${{ secrets.이름 }}` : Workflow에서 Secret 값을 꺼내 쓰는 방법 (로그에는 `***` 로 표시)

- GitHubb `kcu-git-lab`저장소 → **Settings** → **Secrets and variables** → **Actions** → **[New repository secret]**
- 아래 2개를 각각 등록 (Name은 **대소문자까지 정확히** 입력)

| Name | Secret |
|---|---|
| `AWS_ACCESS_KEY_ID` | 1단계에서 만든 Access key ID |
| `AWS_SECRET_ACCESS_KEY` | 1단계에서 만든 Secret access key |

- **Repository secrets** 목록에 2개가 보이면 완료

### 3단계: ECR Push Workflow 작성 (container-ci.yml)

> [!NOTE]
> **용어 정리**
>
> - `env` : Workflow 전체에서 쓰는 변수 (`${{ env.AWS_REGION }}`)
> - `aws-actions/configure-aws-credentials@v5` : Secrets의 Access Key로 Runner에 AWS 인증 설정
> - `aws-actions/amazon-ecr-login@v2` : ECR에 docker login (`id: login-ecr`)
> - `${{ steps.login-ecr.outputs.registry }}` : ECR 주소 (`<계정ID>.dkr.ecr.ap-northeast-2.amazonaws.com`)
> - `${{ github.sha }}` : 이미지 Tag로 사용할 Commit ID (40문자 전체)

- 2강 Workflow를 확장한 파일이므로 기존 `docker-build.yml`, `hello.yml` 은 삭제

```bash
cd ~/kcu-git-lab
git rm .github/workflows/docker-build.yml
git rm .github/workflows/hello.yml
```

- `container-ci.yml` 작성

```bash
cat > .github/workflows/container-ci.yml << 'EOF'
name: Container CI

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-northeast-2
  ECR_REPOSITORY: webapp

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      # 저장소 코드(Dockerfile, index.html)를 Runner로 가져오기
      - name: Checkout
        uses: actions/checkout@v4

      # Secrets의 Access Key로 Runner에 AWS 인증 설정
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      # ECR에 docker login
      # 로그인에 성공하면 ECECR 레지스트리 주소를 login-ecr Step의 결과값(outputs.registry)으로 내보냄
      # 예: 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # 이미지 빌드
      # 123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:<Commit SHA>
      - name: Build Docker Image
        run: |
          docker build -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} .

      # 빌드한 이미지를 ECR webapp 리포지토리로 Push
      - name: Push Docker Image
        run: |
          docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
EOF
```

### 4단계: 소스 수정 후 Push (CI 실행)

- index.html 수정

```bash
echo "<h1>GitHub Actions CI Complete!</h1>" > index.html
cat index.html
```

- Stage 후 **Commit 전에** 변경 내용 확인

```bash
git add .
git status
```

```text
Changes to be committed:
        new file:   .github/workflows/container-ci.yml
        deleted:    .github/workflows/docker-build.yml
        modified:   index.html
```

- Commit & Push

```bash
git commit -m "Add container CI to ECR"
git push
```

> **참고**: Commit 이후에 `git status` 를 다시 보면 `nothing to commit, working tree clean` 이 나오는 것이 정상입니다.
>
> **참고**: 이제 개발자는 push만 하고, Checkout → AWS 인증 → ECR Login → Docker Build → Docker Push는 GitHub Actions가 수행합니다.

### 5단계: Actions 결과 확인

> [!NOTE]
> **확인 포인트**
>
> - 5개 Step 모두 ✅ : Checkout → Configure AWS credentials → Login to Amazon ECR → Build → Push
> - 로그에 Access Key 값은 `***` 로 가려져 표시됨

- **Actions** 탭 → **Container CI** → **build** Job → Step 확인
- **Push Docker Image** 로그 마지막 줄 확인

```text
<Commit SHA>: digest: sha256:xxxxxxxx... size: 2xxx
```

> 💡 `Credentials could not be loaded` 오류 : Secret 이름 오타 또는 미등록 → 2단계 재확인
>
> 💡 `name unknown: The repository with name 'webapp' does not exist` : ECR 리포지토리 이름·리전 확인 (ap-northeast-2)

### 6단계: Amazon ECR에서 이미지 확인

- AWS 콘솔 → **Amazon ECR** → **Repositories** → **webapp** → **Images**
- 이미지 태그가 **Commit SHA(40자리)** 인지 확인
- 로컬 최신 Commit ID와 비교 → 같은 값이면 "이 Commit으로 만든 이미지"임을 추적 가능

```bash
git log -1 --format=%H
```

> **참고**: 11주차 CI는 **Amazon ECR까지**가 종료 지점입니다. EKS에 실제로 배포하는 CD는 다음 주 **Argo CD**에서 다룹니다.

---

### 리소스 정리 (12주차 연계)

> ⚠️ 11주차에 만든 ECR 이미지와 인증 설정은 **12주차 Argo CD · EKS 배포 실습까지 유지**합니다. 아래 삭제는 **12주차 실습이 끝난 뒤** 진행합니다.

- **유지할 리소스** : ECR `webapp` 리포지토리·이미지, IAM 사용자 `github-actions-ci` · Access Key, GitHub Secrets, Container CI Workflow
  - 이미지 1개(수십 MB)를 보관하는 비용은 매우 적습니다.
  - Access Key가 노출된 것 같으면 즉시 **IAM → 액세스 키 비활성화** 후 새 키를 발급해 Secrets를 다시 등록합니다.

> ⚠️ GitHub `kcu-git-lab` 저장소와 Workflow 파일은 **다음 주 Argo CD 실습에서 계속 사용**하므로 삭제하지 않습니다.
