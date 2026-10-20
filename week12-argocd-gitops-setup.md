
# 12주차 Hands-on Labs: Argo CD 기반 GitOps 배포 환경 구축

## 실습 환경

| 항목 | 구성 |
|---|---|
| 클러스터 작업 | AWS CloudShell (서울 리전 `ap-northeast-2`) · `kubectl` · `eksctl` |
| Git 작업 | Windows 10/11 + VSCode 터미널(Git Bash) |
| EKS | 클러스터 `kcu-eks` / 노드 `t3.medium` × 2 (Spot) |
| Argo CD | 네임스페이스 `argocd` · 공식 `stable` 매니페스트 설치 · UI는 LoadBalancer로 노출 |
| 저장소 | 애플리케이션 저장소 `kcu-git-lab` (10~11주차) / GitOps 저장소 `kcu-gitops` (이번 주 생성, Public) |
| 이미지 | 11주차 CI가 만든 ECR `webapp:<Commit SHA>` |

> [!WARNING]
> **EKS는 프리티어가 없는 유료 서비스**입니다. 1강~3강 실습을 가능하면 하루에 이어서 진행하고, 마지막의 **리소스 정리**를 반드시 수행합니다.
>
> 중간에 멈춰야 한다면 리소스 정리 순서대로 삭제한 뒤 다음에 다시 생성합니다.

---

## 12주 1강 Hands-on Labs : Argo CD 실습 환경 구성

> CloudShell에서 EKS 클러스터를 만들고 Argo CD를 설치
>
> `argocd-server`를 LoadBalancer로 노출해 웹 UI에 admin으로 로그인
>
> CloudShell → eksctl(EKS 생성) → kubectl(Argo CD 설치) → 브라우저(Argo CD UI)

### 1단계: CloudShell과 eksctl 준비

- AWS 콘솔 오른쪽 위 리전이 **아시아 태평양(서울)** 인지 확인 → **CloudShell** 실행
- 계정과 kubectl 확인

```bash
aws sts get-caller-identity
kubectl version --client
```

- eksctl 설치 (8주차에 설치했다면 `eksctl version` 만 확인)

```bash
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz"
tar -xzf eksctl_Linux_amd64.tar.gz
mkdir -p $HOME/bin && mv eksctl $HOME/bin/
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
eksctl version
```

### 2단계: EKS 클러스터 생성

- 8주차와 같은 설정으로 클러스터 생성 (약 15~20분)

```bash
eksctl create cluster \
  --name kcu-eks \
  --region ap-northeast-2 \
  --nodegroup-name kcu-nodegroup \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 2 \
  --managed \
  --spot
```

> [!NOTE]
> 생성 중에는 CloudShell 창을 닫지 않습니다.
>
> 오류가 나면 **CloudFormation** 콘솔의 `eksctl-kcu-eks-*` 스택 이벤트에서 원인을 확인합니다.
>
> Spot 용량 부족 오류가 나면 `--spot` 옵션을 빼고 다시 생성합니다.

### 3단계: 클러스터 연결 확인

- kubeconfig 갱신 (eksctl이 자동 설정하지만 한 번 더 실행)

```bash
aws eks update-kubeconfig --region ap-northeast-2 --name kcu-eks
```

- Worker Node 2대가 Ready인지 확인

```bash
kubectl get nodes
kubectl get namespaces
```

```text
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-xx-xx.ap-northeast-2.compute.internal    Ready    <none>   3m    v1.xx
ip-192-168-yy-yy.ap-northeast-2.compute.internal    Ready    <none>   3m    v1.xx
```

> [!NOTE]
> 다음 날 CloudShell에 다시 접속해 실습을 이어갈 때도 이 단계부터 실행합니다.

### 4단계: Argo CD 설치

- Argo CD는 여러 개의 구성 요소가 Pod로 나뉘어 argocd 네임스페이스에서 동작합니다.

```bash
# argocd 네임스페이스 생성 후 설치
kubectl create namespace argocd

# argocd 설치 운영
# --server-side --force-conflicts : 공식 문서 권장 방식. Argo CD의 큰 CRD를 크기 제한 없이 적용
# stable : 최신 안정 버전을 가리킴 (운영에서는 v3.x.y 처럼 버전 고정 권장)
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- 1~2분 후 Pod가 모두 Running인지 확인
  - API Server : 웹 UI · CLI 요청 처리, 로그인 인증 · 권한 관리
  - Repo Server : Git 저장소를 Clone해 쿠버네티스 매니페스트 생성
  - Application Controller : 원하는 상태와 실제 상태 비교 → Sync 실행

```bash
# Argo CD 구성 요소들이 각각 Pod로 실행되는지 확인
kubectl get pods -n argocd
```

```text
NAME                                                READY   STATUS
argocd-application-controller-0                     1/1     Running
argocd-applicationset-controller-xxxxxxxxxx-xxxxx   1/1     Running
argocd-dex-server-xxxxxxxxxx-xxxxx                  1/1     Running
argocd-notifications-controller-xxxxxxxxxx-xxxxx    1/1     Running
argocd-redis-xxxxxxxxxx-xxxxx                       1/1     Running
argocd-repo-server-xxxxxxxxxx-xxxxx                 1/1     Running
argocd-server-xxxxxxxxxx-xxxxx                      1/1     Running
```

### 5단계: Argo CD UI 외부 노출

- argocd-server Service 타입을 ClusterIP → LoadBalancer로 변경

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

- 접속 주소 출력

```bash
kubectl get svc argocd-server -n argocd

ARGOCD_URL=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo https://$ARGOCD_URL
```

- 로드밸런서 주소가 실제로 열리기까지 **2~3분** 걸립니다. 접속이 안 되면 잠시 후 다시 시도합니다.
- 브라우저에서 `https://<접속 주소>` 열기
- 인증서 경고 화면 → **[고급]** → **[계속 진행]**

> [!NOTE]
> Argo CD가 스스로 만든 자체 서명(Self-signed) 인증서를 사용하므로 경고가 나오는 것이 정상입니다.

### 6단계: admin 로그인

- 초기 admin 비밀번호 확인

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

- **Username** : `admin`
- **Password** : 위에서 출력된 값 → **[SIGN IN]**
- 비어 있는 **Applications** 화면이 나오면 로그인 성공

### 7단계: Argo CD UI 둘러보기

- **Applications** : 배포 단위 목록 (현재 비어 있음)
- **Settings → Clusters** : `in-cluster` · `https://kubernetes.default.svc` (Argo CD가 설치된 kcu-eks 자신 = 배포 대상)
- **Settings → Repositories** : 등록된 저장소 없음 (Public 저장소는 등록 불필요, Private 저장소는 여기서 인증 등록)
- **Settings → Projects** : `default` 프로젝트 확인
- **User Info → [UPDATE PASSWORD]** 로 admin 비밀번호 변경

---

## 12주 2강 Hands-on Labs : Argo CD로 webapp 배포

> GitOps 저장소(`kcu-gitops`)에 Deployment·Service 매니페스트를 작성하고 Push
>
> Argo CD UI에서 Application을 만들어 Manual Sync로 11주차 ECR 이미지를 EKS에 배포
>
> ① ECR 이미지 URI → ②③④ kcu-gitops 작성·push → ⑤ Application 생성 → ⑥ Sync → ⑦ 웹 접속

### 1단계: ECR 이미지 URI 확인 (CloudShell)

- 가장 최근에 Push된 이미지의 전체 URI 출력

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
TAG=$(aws ecr describe-images --repository-name webapp \
  --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags[0]' --output text)
echo ${ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:${TAG}
```

```text
123456789012.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:3f9c1a7e....(40자리)
```

- 출력된 **이미지 URI 전체를 복사**해 둠

> [!NOTE]
> AWS 콘솔 **ECR → webapp → Images** 에서 최신 이미지의 **[URI 복사]** 를 사용해도 됩니다.

### 2단계: GitOps 저장소 생성 (GitHub · 내 PC)

> [!NOTE]
> **용어 정리**
>
> - **애플리케이션 저장소** `kcu-git-lab` : 소스 코드 · Dockerfile · CI Workflow
> - **GitOps 저장소** `kcu-gitops` : 배포 설정 YAML만 보관, Argo CD가 감시할 공간

- GitHub 오른쪽 위 **[+]** → **New repository**
  - Repository name : `kcu-gitops`
  - Visibility : **Public**
  - ☑ **Add README** (이번에는 GitHub에서 먼저 만들고 Clone)

- 내 PC VSCode 터미널(Git Bash)에서 Clone

```bash
cd ~
git clone https://github.com/<Username>/kcu-gitops.git
cd kcu-gitops
mkdir webapp
code -r .
```

### 3단계: deployment.yaml 작성

> [!NOTE]
> **확인 포인트**
>
> - `<< EOF` (따옴표 없음) : `${IMAGE}` 자리에 변수 값(실제 이미지 URI)이 들어감
> - YAML은 **스페이스 2칸** 들여쓰기 (탭 사용 금지)

- 1단계에서 복사한 이미지 URI를 변수에 저장

```bash
# echo ${ACCOUNT_ID}.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:${TAG}
# 609417967491.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:1a9e31258c851093799e603a54d5e713d4d82018
# IMAGE=609417967491.dkr.ecr.ap-northeast-2.amazonaws.com/webapp:1a9e31258c851093799e603a54d5e713d4d82018
IMAGE=<1단계에서 복사한 이미지 URI>
echo $IMAGE
```

- deployment.yaml 작성

```bash
cat > webapp/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp
          image: ${IMAGE}
          ports:
            - containerPort: 80
EOF
```

### 4단계: service.yaml 작성 후 Push

- service.yaml 작성 (치환할 변수가 없으므로 `'EOF'`)

```bash
cat > webapp/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
EOF
```

- image 줄 확인 → `${IMAGE}` 가 아니라 **실제 ECR 주소**가 보여야 함

```bash
cat webapp/deployment.yaml
```

- Commit & Push

```bash
git add .
git commit -m "Add webapp manifests"
git push
```

```text
kcu-gitops/
├── README.md
└── webapp/
    ├── deployment.yaml
    └── service.yaml
```

### 5단계: Application 생성 (Argo CD UI)

- **Application** : 무엇을(Source) · 어디에(Destination) · 어떻게(Sync Policy) 배포할지 정의한 Argo CD 리소스
  - **Revision** `HEAD` : 저장소 기본 브랜치(main)의 최신 Commit
  - **AUTO-CREATE NAMESPACE** : `CreateNamespace=true`, 네임스페이스가 없으면 자동 생성

- **Applications → [+ NEW APP]**

| 항목 | 입력값 |
|---|---|
| Application Name | `webapp` |
| Project Name | `default` |
| SYNC POLICY | `Manual` |
| SYNC OPTIONS | ☑ `AUTO-CREATE NAMESPACE` |
| Repository URL | `https://github.com/<Username>/kcu-gitops.git` |
| Revision | `HEAD` |
| Path | `webapp` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `webapp` |

- **[CREATE APPLICATION]** → webapp 카드 상태 확인 : `OutOfSync` · `Missing` (Git에는 있지만 클러스터에는 아직 없음)

- 같은 내용을 YAML로 표현하면 (참고용)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: webapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<Username>/kcu-gitops.git
    targetRevision: HEAD
    path: webapp
  destination:
    server: https://kubernetes.default.svc
    namespace: webapp
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### 6단계: Sync 실행

> [!NOTE]
> **확인 포인트**
>
> - Sync Status : `OutOfSync` → `Synced`
> - Health Status : `Missing` → `Progressing` → `Healthy`

- webapp 카드 → **[SYNC]** → **[SYNCHRONIZE]**

<img width="376" height="278" alt="Argo CD Sync 실행" src="https://github.com/user-attachments/assets/6aa9c69d-277f-41a9-b3ff-ae44f7fd53cc" />

- webapp 카드 클릭 → 리소스 트리 확인 : `webapp-svc(svc)` · `webapp(deploy)` → `rs` → `pod × 2`

<img width="1059" height="389" alt="Argo CD 리소스 트리" src="https://github.com/user-attachments/assets/7f0040e4-376a-4b9e-86bb-fb8c926c01cf" />

- CloudShell에서도 확인

```bash
kubectl get applications -n argocd
kubectl get all -n webapp
```

### 7단계: 웹 서비스 접속 확인

- Service의 외부 주소 확인

```bash
kubectl get svc webapp-svc -n webapp
```

```text
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP                                   PORT(S)
webapp-svc   LoadBalancer   10.100.xx.xx    xxxx.ap-northeast-2.elb.amazonaws.com         80:3xxxx/TCP
```

- 브라우저에서 `http://<EXTERNAL-IP>` 접속 (2~3분 소요) → **GitHub Actions CI Complete!** 확인 (11주차 3강에서 수정한 index.html)

<img width="784" height="169" alt="webapp 서비스 접속 확인" src="https://github.com/user-attachments/assets/da519269-9935-4426-9069-473076099e7b" />

- Argo CD UI → `webapp(deploy)` 클릭 → **LIVE MANIFEST** 에서 배포된 image 태그 확인

---

## 12주 3강 Hands-on Labs : GitOps 자동 배포 · 복구 · 롤백

> Auto Sync · Prune · Self Heal을 켜고 Git Commit만으로 배포를 운영
>
> replicas 변경(자동 배포) → kubectl scale(자동 복구) → 새 이미지 태그(새 버전 배포) → git revert(롤백)
>
> 마지막에 리소스를 순서대로 정리

### 1단계: Auto Sync 활성화 (Argo CD UI)

- `automated` : Git 변경을 감지하면 자동 Sync (기본 약 3분 주기 Polling, `[REFRESH]` 로 즉시 비교)
- `prune` : Git에서 삭제한 리소스를 클러스터에서도 삭제
- `selfHeal` : kubectl 등으로 직접 바꾼 상태(Drift)를 Git 상태로 되돌림

- webapp → **[DETAILS]** → **SYNC POLICY**
  - **[ENABLE AUTO-SYNC]** → **[OK]**
  - **PRUNE RESOURCES** → **[ENABLE]**
  - **SELF HEAL** → **[ENABLE]**

<img width="274" height="238" alt="Argo CD Auto Sync 설정" src="https://github.com/user-attachments/assets/b41831a3-1291-4cec-bd9c-5bcbecdd7ac2" />

- CloudShell에서 설정 확인

```bash
kubectl get application webapp -n argocd -o jsonpath="{.spec.syncPolicy}"; echo
```

```text
{"automated":{"prune":true,"selfHeal":true},"syncOptions":["CreateNamespace=true"]}
```

### 2단계: Git으로 replicas 변경 (내 PC - vscode)

- Pod 개수를 2 → 3으로 변경 후 Push (VSCode에서 직접 수정해도 됨)

```bash
cd ~/kcu-gitops
git pull
sed -i 's/replicas: 2/replicas: 3/' webapp/deployment.yaml
cat webapp/deployment.yaml

git diff
git commit -am "Scale webapp to 3"
git push
```

- **SYNC 버튼을 누르지 않고** 자동 반영 확인 (CloudShell)

```bash
kubectl get pods -n webapp
```

```text
webapp-xxxxxxxxxx-aaaaa   1/1   Running
webapp-xxxxxxxxxx-bbbbb   1/1   Running
webapp-xxxxxxxxxx-ccccc   1/1   Running
```

<img width="1371" height="444" alt="replicas 3개 자동 반영" src="https://github.com/user-attachments/assets/b0687899-6e0f-408b-b894-8f984812c7d9" />

### 3단계: Self-Heal 확인 (CloudShell)

- **Self-Heal : Argo CD가 Drift를 감지해 Git에 선언된 상태로 되돌림**
- **결과 : 클러스터에서 직접 변경한 내용은 자동 복구되므로, 지속적으로 유지할 변경 사항은 Git을 통해 관리해야 함**

- kubectl로 직접 Pod 개수 변경 → Drift 발생(**Git과 클러스터가 달라진 상태**)

```bash
kubectl scale deployment webapp -n webapp --replicas=5
kubectl get pods -n webapp -w
```

- 5개로 늘었다가 **수 초 후 Git 기준 3개로 복구**되는지 확인 → `Ctrl + C` 로 종료

```bash
kubectl get pods -n webapp
```

- Deployment를 삭제해도 다시 생성되는지 확인

```bash
kubectl delete deployment webapp -n webapp
kubectl get pods -n webapp -w

kubectl get deployment -n webapp
```

- Argo CD UI에서 `OutOfSync` → `Synced` 로 바뀌는 과정 확인

<img width="772" height="324" alt="Argo CD Self-Heal" src="https://github.com/user-attachments/assets/a1a84f48-4772-4ac2-85af-6e8c25b8503c" />

### 4단계: 새 이미지 빌드 (내 PC · CI)

- 애플리케이션 저장소의 웹 페이지 수정 후 Push → 11주차 **Container CI** 실행

```bash
cd ~/kcu-git-lab
git pull
echo "<h1>Argo CD GitOps v2</h1>" > index.html
git commit -am "Update page to v2"
git push
```

- GitHub **Actions** 탭 → **Container CI** 5개 Step 성공 확인
- ECR `webapp` 에 새 태그가 생겼는지 확인 → 새 태그 = 아래 Commit ID

```bash
git log -1 --format=%H
```

### 5단계: 이미지 태그 변경 → 자동 배포

- 새 태그를 변수에 저장하고 GitOps 저장소의 image 태그 변경

```bash
NEW_TAG=$(git -C ~/kcu-git-lab log -1 --format=%H)
cd ~/kcu-gitops
sed -i "s|webapp:.*|webapp:${NEW_TAG}|" webapp/deployment.yaml
git diff
git commit -am "Deploy webapp v2"
git push
```

- Rolling Update 확인 (CloudShell) → 브라우저 새로고침 → **Argo CD GitOps v2**

```bash
kubectl get pods -n webapp -w
kubectl get deployment webapp -n webapp \
  -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

<img width="776" height="157" alt="새 이미지 자동 배포" src="https://github.com/user-attachments/assets/94c6bf38-7508-4812-a743-a0542532f803" />

### 6단계: git revert로 롤백

- `git revert HEAD` : 마지막 Commit의 변경을 취소하는 **새 Commit** 생성 (이력이 지워지지 않음)
- Argo CD UI의 **HISTORY AND ROLLBACK** 은 Auto Sync가 켜져 있으면 Rollback 버튼을 사용할 수 없음 → GitOps에서는 Git으로 롤백

- 직전 배포 Commit 되돌리기

```bash
cd ~/kcu-gitops
git revert HEAD --no-edit
git log --oneline
git push
```

```text
xxxxxxx (HEAD -> main) Revert "Deploy webapp v2"
xxxxxxx Deploy webapp v2
xxxxxxx Scale webapp to 3
xxxxxxx Add webapp manifests
xxxxxxx Initial commit
```

- 자동 Sync 후 브라우저 새로고침 → **GitHub Actions CI Complete!** (이전 버전)

<img width="774" height="139" alt="GitOps 롤백 결과" src="https://github.com/user-attachments/assets/f7b4a46c-6e3d-4553-86f6-b46f0782937f" />

- Argo CD UI → **[HISTORY AND ROLLBACK]** → 배포 이력과 Revision(Commit) 확인

---

## 리소스 정리 (순서 중요)

> [!WARNING]
> 로드밸런서(ELB)가 남아 있으면 VPC 삭제가 막혀 **EKS 클러스터 삭제가 실패**할 수 있습니다. 반드시 아래 순서대로 진행합니다.

### ① Application 삭제 (Argo CD UI)

- webapp → **[DELETE]** → 앱 이름 `webapp` 입력 → Propagation Policy **Foreground** → **[OK]**
- Deployment · Pod · Service(ELB)가 함께 삭제되었는지 확인

```bash
kubectl get all -n webapp
```

### ② Argo CD UI용 로드밸런서 · 네임스페이스 삭제

```bash
kubectl delete svc argocd-server -n argocd
kubectl delete namespace argocd webapp
```

### ③ EKS 클러스터 삭제 (약 10~15분)

```bash
eksctl delete cluster --name kcu-eks --region ap-northeast-2
```

### ④ 11주차 리소스 정리

[11주차 GitHub Actions CI 실습 노트](week11-github-actions-ci-pipeline.md)의 **12주차 실습 종료 후 일괄 삭제** 참고

- GitHub `kcu-git-lab` → Actions → **Container CI** → **Disable workflow**
- GitHub Secrets `AWS_ACCESS_KEY_ID` · `AWS_SECRET_ACCESS_KEY` 삭제
- Amazon ECR `webapp` 리포지토리 삭제
- IAM 사용자 `github-actions-ci` 액세스 키 비활성화 → 삭제 → 사용자 삭제

### ⑤ 정리 확인

- EC2 → **로드 밸런서** · **인스턴스** 가 남아 있지 않은지 확인
- EKS 클러스터 목록 비어 있음 · CloudFormation `eksctl-kcu-eks-*` 스택 삭제 완료

> [!NOTE]
> GitHub `kcu-git-lab` · `kcu-gitops` 저장소는 비용이 들지 않으므로 그대로 유지합니다.

