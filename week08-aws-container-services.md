# 8주차 **클라우드 컨테이너 서비스**

---

## 8주 1강 **Hands-on Labs : 컨테이너 이미지 생성 후 ECR에 저장**

---

### **1단계: 프라이빗 리포지토리 생성**

- **프라이빗 리포지토리 생성**
    - 리포지토리 이름 : `kcu-web`
    - 이미지 태그 설정: Mutable
    - 생성

### **2단계: 컨테이너 빌드**

- 간단한 웹 페이지 생성
    
    ```bash
    mkdir ~/kcu-web
    cd ~/kcu-web
    echo " <h1>KCU Cloud DevOps</h1> " > index.html
    ```
    
- Dockerfile 생성
    
    ```bash
    cat > Dockerfile <<'EOF'
    FROM nginx:alpine
    COPY index.html /usr/share/nginx/html/index.html
    EXPOSE 80
    EOF
    
    cat Dockerfile
    ```
    
- 컨테이너 빌드
    
    ```bash
    docker build -t kcu-web:latest .
    docker images
    ```
    

### 3단계: ECR에 Image 저장

- 생성한 리포지토리 선택 후 [푸시 명령 보기] 선택
    
    ```bash
    1. 인증 토큰을 검색하고 레지스트리에 대해 Docker 클라이언트를 인증합니다. 다음 AWS CLI을(를) 사용하세요.
    aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 609417967491.dkr.ecr.ap-northeast-2.amazonaws.com
    참고: AWS CLI을(를) 사용하는 중 오류가 발생하면 최신 버전의 AWS CLI 및 Docker가 설치되어 있는지 확인하세요.
    
    2. 다음 명령을 사용하여 도커 이미지를 빌드합니다. 도커 파일을 처음부터 새로 빌드하는 방법에 대한 자세한 내용은 여기  지침을 참조하십시오. 이미지를 이미 빌드한 경우에는 이 단계를 건너뛸 수 있습니다.
    docker build -t kcu-web .
    
    3. 빌드가 완료되면 이미지에 태그를 지정하여 이 리포지토리에 푸시할 수 있습니다.
    docker tag kcu-web:latest 609417967491.dkr.ecr.ap-northeast-2.amazonaws.com/kcu-web:latest
    docker images
    
    4. 다음 명령을 실행하여 이 이미지를 새로 생성한 AWS 리포지토리로 푸시합니다.
    docker push 609417967491.dkr.ecr.ap-northeast-2.amazonaws.com/kcu-web:latest
    ```
    

### **4단계: ECR에 저장된 컨테이너 이미지 확인**

- AWS Console → **ECR**
- `kcu-web` 선택
- 저장된 이미지 확인
    - Image tag: `latest`
    - Image URI 확인

## 8주 2강 **Hands-on lab: EKS 클러스터 구축**

---

### **1단계: eksctl 설치 및 실습 환경 확인**

- AWS Console에서 **CloudShell** 실행
- 리전 / AWS 계정 정보 확인
    
    ```
    aws configure get region
    aws sts get-caller-identity
    ```
    
- `eksctl` 최신 버전 다운로드 및 설치
    
    ```bash
    ARCH=amd64
    PLATFORM=$(uname -s)_$ARCH
    
    curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
    
    tar -xzf eksctl_$PLATFORM.tar.gz
    
    mkdir -p $HOME/bin
    mv eksctl $HOME/bin/
    
    export PATH=$HOME/bin:$PATH
    ```
    
- 기본 도구 설치 여부 확인
    
    ```bash
    aws --version
    kubectl version --client
    eksctl version
    ```
    

### **2단계: EKS 클러스터 생성**

- `eksctl`을 이용하여 EKS 클러스터 생성
    
    비용 절감을 위해 Worker Node를 **Spot Instance**로 생성
    
    ```
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
    
- 생성 완료까지 시간이 소요(10분 정도)되므로 진행 상태를 확인한다.
- 촬영을 위해 미리 만들었습니다.

### **3단계: kubectl을 이용한 EKS 클러스터 확인**

- 생성된 EKS 클러스터 확인
    
    ```
    eksctl get cluster
    ```
    
- EKS와 `kubectl` 연결 정보 설정
    
    ```
    aws eks update-kubeconfig --region ap-northeast-2 --name kcu-eks
    ```
    
- Worker Node 확인
    
    ```
    kubectl get nodes
    kubectl get nodes -o wide
    ```
    

### **4단계: EKS 클러스터 연결 및 리소스 확인**

- **EKS → Clusters → `kcu-eks`**
- Cluster 확인
    - Cluster Name : `kcu-eks`
    - Status : `Active`
    - Kubernetes Version 확인
    - Networking 정보 확인
- Managed Node Group 확인
    - EKS → `kcu-eks`
    - **Compute → Node groups**
    - `kcu-nodegroup` 선택
- EC2 Worker Node 확인
    - **EC2 → Instances**
    - EKS에서 생성된 EC2 인스턴스 확인

## 8주 3강 : Hands-on lab: ECR Image를 EKS에 배포하여 웹서비스 운영

### 1단계: ECR Image 주소 확인

- ECR 리포지토리에 저장된 컨테이너 이미지 확인

```
AWS_REGION=ap-northeast-2
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

IMAGE=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/kcu-web:latest
echo $IMAGE
```

### 2단계: Deployment 생성

- ECR에 저장된 `kcu-web:latest` 이미지를 실행하는 Deployment YAML 생성
- Deployment Name: `kcu-webserver-dep`
- Pod 수: `2`
    
    ```bash
    cat > kcu-deployment.yaml <<EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: kcu-webserver-dep
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: kcu-webserver
      template:
        metadata:
          labels:
            app: kcu-webserver
        spec:
          containers:
            - name: kcu-webserver
              image: ${IMAGE}
              ports:
                - containerPort: 80
    EOF
    ```
    
- 웹 서버 배포
    
    ```bash
    kubectl apply -f kcu-deployment.yaml
    ```
    
- 배포 확인
    
    ```bash
    kubectl get deployment
    kubectl get pods
    
    kubectl get pods -o wide
    ```
    

### 3단계: LoadBalancer Service 생성

- 외부에서 웹서버에 접근할 수 있도록 Service 생성
- Service Name: `kcu-webserver-svc`
    
    ```bash
    cat > kcu-service.yaml <<'EOF'
    apiVersion: v1
    kind: Service
    metadata:
      name: kcu-webserver-svc
    spec:
      selector:
        app: kcu-webserver
      ports:
        - protocol: TCP
          port: 80
          targetPort: 80
      type: LoadBalancer
    EOF
    ```
    
- service 생성
    
    ```bash
    kubectl apply -f kcu-service.yaml
    ```
    

### 4단계: 웹 서비스 접속 확인

- **EC2 → Load Balancers**
- Kubernetes Service에 의해 생성된 Load Balancer 확인
- CloudShell에서 Service 주소 확인
    
    ```
    kubectl get svc
    ```
    
- 웹 브라우저에서 확인

## Hands-on lab: 8주차에 생성한 aws 리소스 삭제

### 1단계: Kubernetes Resource 삭제

- Service 삭제
    
    ```bash
    kubectl delete service kcu-webserver-svc
    ```
    
- Deployment 삭제
    
    ```bash
    kubectl delete service kcu-webserver-svc
    ```
    
- 삭제 확인
    
    ```bash
    kubectl delete deployment kcu-webserver-dep
    ```
    

### 2단계: EKS Cluster 삭제

- EKS 삭제
    
    ```
    eksctl delete cluster \
      --name kcu-eks \
      --region ap-northeast-2
    ```
    
- 삭제 여부 확인

### 3단계: ECR Repository 삭제

- ECR → Repositories → `kcu-web` → Delete
