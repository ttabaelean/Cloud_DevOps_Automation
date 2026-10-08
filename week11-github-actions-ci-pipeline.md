11주차 Hands-on Labs: GitHub Actions를 활용한 CI 파이프라인 구축
학습 목표: GitHub 저장소에 코드를 Push하면 GitHub Actions가 자동으로 실행되어 Docker 이미지를 빌드하고 Amazon ECR에 저장하는 CI 파이프라인을 구성합니다.

실습 환경
항목	구성
실습 PC	Windows 10/11, VSCode 터미널(Git Bash)
로컬 / 원격 저장소	~/kcu-git-lab / GitHub kcu-git-lab (10주차 실습 연계)
Runner	GitHub-hosted ubuntu-latest
Workflow	.github/workflows/ (hello.yml → docker-build.yml → container-ci.yml)
AWS	서울 리전 ap-northeast-2, ECR 리포지토리 webapp


[!IMPORTANT]
내 PC에 Docker를 설치할 필요는 없습니다. Docker Build 및 Push는 GitHub Runner에서 실행합니다. AWS Access Key를 소스 파일이나 Git 저장소에 기록하거나 Commit하지 마세요.

11주 1강. 첫 번째 GitHub Actions Workflow 실행
실습 흐름: Workflow 작성 → Git Commit & Push → GitHub Push Event → Runner 실행 → Actions 로그 확인
1단계. 실습 저장소 준비
cd ~/kcu-git-lab
git switch main
git pull
ls
git status
정상 상태 예시:
On branch main
Your branch is up to date with 'origin/main'.
nothing to commit, working tree clean
[!NOTE]
- git switch main: main 브랜치로 이동
- git pull: 원격 저장소의 변경 사항을 가져와 현재 브랜치에 반영
- 로컬 저장소가 없다면 git clone <저장소 URL>로 복제한 후 폴더로 이동합니다.

2단계. Workflow 파일 작성 (hello.yml)
mkdir -p .github/workflows
cat > .github/workflows/hello.yml << 'EOF'
name: Hello CI

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      - name: Hello
        run: echo "Hello GitHub Actions"
EOF

cat .github/workflows/hello.yml
[!NOTE]
- .github/workflows/: GitHub Actions Workflow 파일 위치
- name: Actions 탭에 표시할 Workflow 이름
- on: push: 모든 브랜치의 Push 이벤트에 반응
- jobs: 실행할 작업 정의 / runs-on: Runner 지정
- uses: 준비된 Action 사용 / run: Runner에서 명령어 실행

3단계. Commit & Push
git add .github/workflows/hello.yml
git commit -m "Add hello workflow"
git push
[!NOTE]
git add → git commit → git push 순서로 변경 내용을 GitHub에 올립니다. Push 자체가 Workflow 실행 이벤트가 됩니다.

4단계. Actions 탭에서 실행 결과 확인
1. GitHub kcu-git-lab → Actions → Hello CI로 이동합니다.
2. Add hello workflow 실행 항목 → hello Job을 선택합니다.
3. Checkout, Hello Step의 성공 여부와 Hello 로그를 확인합니다.
Hello GitHub Actions
[!NOTE]
- 노란색: 실행 중 / 초록색 체크: 성공 / 빨간색 X: 실패
- Set up job, Complete job 등은 GitHub가 자동 수행하는 준비·정리 단계입니다.

5단계. Workflow 수정 후 자동 실행 재확인
hello.yml의 steps 아래에 Show Info Step을 추가합니다. (VSCode 또는 vi 사용)
      - name: Show Info
        run: |
          echo "Branch : ${{ github.ref_name }}"
          echo "Commit : ${{ github.sha }}"
[!IMPORTANT]
위 코드는 기존 steps 아래에 추가합니다. - name: Show Info의 들여쓰기는 - name: Hello와 같아야 합니다.

git add .
git commit -m "Add show info step"
git push
Actions → Hello CI → Show Info 로그에서 브랜치와 Commit ID를 확인하고 다음 명령의 결과와 비교합니다.
git log -1 --format=%H
[!NOTE]
- ${{ github.ref_name }}: 실행을 유발한 브랜치 이름
- ${{ github.sha }}: 실행과 연결된 Commit SHA (3강에서 Docker 이미지 태그로 사용)
- run: |: 여러 줄 명령을 실행

11주 2강. Docker 이미지 Build Workflow 작성
실습 흐름: Dockerfile 준비 → main에 Push → Runner에서 Docker Build → 브랜치 조건 확인
1단계. 저장소 구조 확인
cd ~/kcu-git-lab
ls
ls .github/workflows/
이번 강의에서 추가할 파일:
kcu-git-lab/
├── Dockerfile
├── index.html
└── .github/
    └── workflows/
        ├── hello.yml
        └── docker-build.yml
[!NOTE]
ls -a: 숨김 파일 포함 목록 확인 / find . -path ./.git -prune -o -type f -print: .git을 제외한 파일 목록 확인

2단계. Dockerfile 작성
cat > Dockerfile << 'EOF'
FROM httpd:latest
COPY index.html /usr/local/apache2/htdocs/
EXPOSE 80
EOF
[!NOTE]
- FROM httpd:latest: Apache HTTP Server 기반 이미지
- COPY: 웹 페이지를 Apache 문서 디렉터리로 복사
- EXPOSE 80: 컨테이너의 서비스 포트 표시
- 파일 이름은 정확히 Dockerfile로 작성합니다.

3단계. index.html 수정
echo "<h1>GitHub Actions CI Lab</h1>" > index.html
cat index.html
4단계. Docker Build Workflow 작성 (docker-build.yml)
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
      - name: Checkout
        uses: actions/checkout@v4
      - name: Check Docker
        run: docker --version
      - name: Build Docker Image
        run: docker build -t webapp:latest .
EOF
[!NOTE]
- branches: - main: main 브랜치에 Push할 때만 실행
- docker --version: Runner의 Docker 설치 여부 확인
- docker build -t webapp:latest .: 현재 디렉터리의 Dockerfile로 이미지 빌드

5단계. Commit & Push
git add .
git status
git commit -m "Add Docker build CI"
git push
[!NOTE]
기존 hello.yml은 모든 Push에 반응하므로 이번에는 Hello CI와 Docker Build CI가 모두 실행됩니다.

6단계. Actions 결과 확인
GitHub → Actions → Docker Build CI → build Job에서 다음 Step을 확인합니다.
Checkout → Check Docker → Build Docker Image
빌드 로그에서 webapp:latest 이미지가 생성되었는지 확인합니다.
[!NOTE]
이미지는 내 PC가 아니라 일회성 GitHub-hosted Runner에 생성됩니다. Workflow 종료 후 Runner가 제거되므로 3강에서 이미지를 ECR로 Push하여 보관합니다.

7단계. 브랜치 조건 동작 확인
git switch -c feature-ci
git branch

echo "<p>feature test</p>" >> about.html
cat about.html

git add about.html
git commit -m "Test feature branch"
git push -u origin feature-ci
Actions 탭에서 Hello CI만 새로 실행되고 Docker Build CI는 실행되지 않는지 확인합니다.
git switch main
[!NOTE]
hello.yml은 모든 브랜치의 Push에 반응하지만 docker-build.yml은 main Push에만 반응합니다. 이 실습에서 feature-ci를 main으로 병합하지 않습니다.

11주 3강. GitHub Actions → Amazon ECR Push
실습 흐름: AWS 인증 → ECR Login → Docker Build → Docker Push → ECR 이미지 확인
1단계. AWS 준비 (IAM Access Key / ECR Repository)
① IAM 사용자 및 Access Key 생성
1. AWS 콘솔 → IAM → 사용자 → 사용자 생성
2. 사용자 이름: github-actions-ci (콘솔 액세스 선택하지 않음)
3. 직접 정책 연결 → AmazonEC2ContainerRegistryPowerUser
4. 생성한 사용자 → 보안 자격 증명 → 액세스 키 만들기
5. 사용 사례 AWS 외부에서 실행되는 애플리케이션 선택 후 키 생성
6. Access Key ID와 Secret Access Key를 안전하게 보관합니다.
[!WARNING]
이 방식은 교육용입니다. 장기 Access Key를 사용하는 대신 실무에서는 GitHub OIDC + AWS IAM Role을 사용하여 임시 자격 증명을 발급받는 구성을 권장합니다. 교육용 정책도 ECR 리포지토리별 최소 권한보다 넓습니다. Access Key는 Git 저장소에 저장하거나 Commit하지 않습니다.

② Amazon ECR Repository 생성
1. AWS 콘솔에서 리전을 **서울 (ap-northeast-2)**로 설정합니다.
2. Amazon ECR → Private registry → Repositories → 리포지토리 생성
3. 이름: webapp / 나머지는 기본값
4. 생성 후 URI를 확인합니다.
123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/webapp
[!NOTE]
위 URI의 123456789012는 예시 AWS 계정 ID입니다. 실제 실습에서는 본인 계정의 URI를 사용합니다.

2단계. GitHub Secrets 등록
GitHub kcu-git-lab → Settings → Secrets and variables → Actions → New repository secret에서 다음 두 값을 각각 등록합니다.
Name	등록 값
AWS_ACCESS_KEY_ID	IAM에서 생성한 Access Key ID
AWS_SECRET_ACCESS_KEY	IAM에서 생성한 Secret Access Key


[!NOTE]
- ${{ secrets.AWS_ACCESS_KEY_ID }}처럼 Workflow에서 Secret을 참조합니다.
- 등록한 Secret 값은 GitHub UI에서 다시 조회할 수 없습니다.
- Secret 값은 로그에서 일반적으로 마스킹되지만, 의도적으로 출력하거나 변형해 노출하지 마세요.

3단계. ECR Push Workflow 작성 (container-ci.yml)
기존 학습용 Workflow 두 개를 삭제합니다.
cd ~/kcu-git-lab
git switch main
git rm .github/workflows/docker-build.yml
git rm .github/workflows/hello.yml
새 Workflow를 작성합니다.
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
      - name: Checkout
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v5
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build Docker Image
        run: |
          docker build -t ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }} .

      - name: Push Docker Image
        run: |
          docker push ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
EOF
[!NOTE]
- env: Workflow에서 공통으로 사용하는 변수
- configure-aws-credentials@v5: Runner의 AWS 인증 설정
- amazon-ecr-login@v2: ECR 레지스트리 로그인
- ${{ steps.login-ecr.outputs.registry }}: 로그인된 ECR 레지스트리 주소
- ${{ github.sha }}: 40자리 Commit SHA를 이미지 태그로 사용
- 이 단계에서는 EKS에 배포하지 않습니다.

4단계. 소스 수정 후 Push (CI 실행)
echo "<h1>GitHub Actions CI Complete!</h1>" > index.html
cat index.html

git add .
git status
git status에서 새 Workflow, 기존 Workflow 삭제, index.html 수정이 Staging되었는지 확인합니다.
git commit -m "Add container CI to ECR"
git push
[!NOTE]
이제 개발자는 Push만 하면 됩니다. Checkout → AWS 인증 → ECR Login → Docker Build → Docker Push는 GitHub Actions가 실행합니다.

5단계. Actions 결과 확인
GitHub → Actions → Container CI → build Job을 열어 다음 다섯 Step이 성공했는지 확인합니다.
Checkout
Configure AWS credentials
Login to Amazon ECR
Build Docker Image
Push Docker Image
Docker Push 성공 로그 예시:
<Commit SHA>: digest: sha256:xxxxxxxx... size: 2xxx
[!NOTE]
- Credentials could not be loaded: GitHub Secrets 이름과 등록 여부 확인
- repository ... does not exist: ECR 리포지토리 이름 및 리전 확인
- IAM 권한 오류: 사용자 정책과 인증 정보를 확인

6단계. Amazon ECR에서 이미지 확인
1. AWS 콘솔 → Amazon ECR → Repositories → webapp → Images
2. 이미지 태그가 40자리 Commit SHA인지 확인합니다.
3. 로컬 저장소의 최신 Commit SHA와 비교합니다.
git log -1 --format=%H
[!IMPORTANT]
11주차 CI의 종료 지점은 Amazon ECR입니다. EKS 배포를 위한 CD는 다음 주 Argo CD 실습에서 다룹니다.

리소스 정리 (12주차 연계)
[!IMPORTANT]
11주차 종료 직후에는 ECR 이미지와 인증 설정을 삭제하지 않습니다. 12주차 Argo CD / EKS 연계 실습에서 사용할 수 있도록 유지합니다. 단, 키가 노출된 것으로 의심되면 즉시 비활성화하고 교체합니다.

11주차 종료 시 유지
- ECR webapp 리포지토리 및 이미지
- IAM 사용자 github-actions-ci 및 실습용 Access Key
- GitHub Secrets 2개
- GitHub kcu-git-lab 저장소 및 container-ci.yml
