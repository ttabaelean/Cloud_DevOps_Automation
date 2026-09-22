# 9주차: VPC·RDS·ECR·EKS 구축 및 PetClinic 배포

---

## 실습 개요

- PetClinic을 3-Tier(VPC-EKS-RDS) 구조로 AWS에 직접 구축하고 배포한다.
    
    <img width="3584" height="1920" alt="Image" src="https://github.com/user-attachments/assets/7ee1e4a9-291a-427a-a0d5-e8cd414a2048" />
    
    | 항목 | 구성 |
    | --- | --- |
    | VPC | petclinic-vpc (10.0.0.0/16) |
    | RDS | petclinic-db (MySQL) |
    | ECR | petclinic |
    | EKS | petclinic-cluster |
    | 관리 환경 | AWS CloudShell |

# 9주 2강 — 인프라 구축 및 이미지 준비

## 1단계: 인프라 아키텍처 구성 단계

### STEP 01: VPC 만들기

- PetClinic 서비스를 운영하기 위한 전용 네트워크를 구성합니다.
    
    <img width="2536" height="1924" alt="Image" src="https://github.com/user-attachments/assets/99d9b058-5ef9-435c-ac37-03d4cbf18ae7" />
    
- vpc name: `petclinic-vpc`
- Public Subnet 2개 (AZ a, c)
- Private Subnet 4개 (AZ a, c)
    
    
    | Subnet | CIDR | AZ | 용도 |
    | --- | --- | --- | --- |
    | Public Subnet 1 | 10.0.1.0/24 | ap-northeast-2a | NAT GW |
    | Public Subnet 2 | 10.0.2.0/24 | ap-northeast-2c | - |
    | Private Subnet 1 | 10.0.3.0/24 | ap-northeast-2a | EKS Node |
    | Private Subnet 2 | 10.0.4.0/24 | ap-northeast-2c | EKS Node |
    | Private Subnet 3 | 10.0.5.0/24 | ap-northeast-2a | RDS |
    | Private Subnet 4 | 10.0.6.0/24 | ap-northeast-2c | RDS |
- 퍼블릭 IPv4 자동 활성화
    - Public Subnet 1 선택 후 작업(Actions) →  서브넷 설정 편집 → **퍼블릭 IPv4 주소 자동 할당** 체크
    - Public Subnet 2 선택 후 작업(Actions) →  서브넷 설정 편집 → **퍼블릭 IPv4 주소 자동 할당** 체크

### STEP 02: CloudShell에서 EKS 만들기

- Amazon EKS 아키텍처
    
    <img width="2536" height="1924" alt="Image" src="https://github.com/user-attachments/assets/6d60c784-e46e-437d-a5b0-82dae7bad854" />
    
- 별도 관리 서버 없이 CloudShell에서 진행합니다.
- eksctl 설치
    
    ```bash
    ARCH=amd64
    PLATFORM=$(uname -s)_$ARCH
    curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$PLATFORM.tar.gz"
    tar -xzf eksctl_$PLATFORM.tar.gz
    mkdir -p $HOME/bin
    mv eksctl $HOME/bin/
    export PATH=$HOME/bin:$PATH
    ```
    
- 설치 확인 (aws cli, kubectl은 CloudShell에 기본 설치되어 있음)
    
    ```bash
    eksctl version
    aws --version
    kubectl version --client
    ```
    
- Subnet ID 확인
    
    ```bash
    aws ec2 describe-subnets \
      --filters "Name=vpc-id,Values=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=petclinic-vpc --query 'Vpcs[0].VpcId' --output text)" \
      --query 'Subnets[].{ID:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}' --output table
    ```
    
- 
- 위에서 확인한 subnet-id 를 아래 명령에 채워서 eks cluster를 생성
    
    
    - EKS cluster 이름: `petclinic-cluster`
    - 노드 그룹 이름: `petclinic-nodegroup`
    - 위치: `Private Subnet 1, 2`
    - 워커 노드 :
        - 타입: `t3.medium`
        - 요구 : `2`
        - 최소 : `1`
        - 최대 `3`
        - `SPOT`
    
    ```bash
    eksctl create cluster \
      --name petclinic-cluster \
      --region ap-northeast-2 \
      --vpc-public-subnets=<Public-1-ID>,<Public-2-ID> \
      --vpc-private-subnets=<Private-1-ID>,<Private-2-ID> \
      --nodegroup-name petclinic-nodegroup \
      --node-type t3.medium --nodes 2 --nodes-min 1 --nodes-max 3 \
      --managed --spot
    ```
    
    - 15분 정도 소요됨

- 인증서 설정 및 동작 확인
    
    ```bash
    aws eks update-kubeconfig --region ap-northeast-2 --name petclinic-cluster
    kubectl get nodes -o wide
    eksctl get cluster --region ap-northeast-2
    ```
    

### STEP 03: ECR Repository 만들기

- ECR → Private repositories → 리포지토리 생성
- 이름 : `petclinic`
- 이미지 태그 변경 가능성: Mutable

### STEP 04: Amazon RDS MySQL 구성

- PetClinic 데이터 저장을 위한 데이터베이스를 구성합니다.
    
    !image.png
    
- 요구사항
    - 보안그룹 생성
        - 이름: `petclinic-db-sg`
        - VPC: `petclinic-vpc`
        - 인바운드 규칙(⚠️주의: Pod가 RDS에 3306포트로 접속 가능해야 합니다. 위에서 확인한 EKS 클러스터 보안 그룹을 Source로 지정하세요.)
            
            
            | Type | Port | Source |
            | --- | --- | --- |
            | MYSQL/Aurora | 3306 | EKS 클러스터 보안 그룹
            **eks-cluster-sg-petclinic-cluster-XXXXX** |
    - DB Subnet Group 생성
        - name: `petclinic-db-subnetgroup`
        - VPC: `petclinic-vpc`
        - 서브넷 추가
            - 가용영역 : A, C
            - 서브넷: Private Subnet 3, 4
    - Amazon RDS MySQL 생성
        - name: `petclinic-db`
        - 엔진: `mysql`
        - 템플릿: **`샌드박스`(단일 AZ, 인스턴스 1개)** (***클라우드 운영비용 절감 목적***)
        - 설정
            - 마스터 사용자 이름: `petclinic`
            - 암호: `petclinic1!`
        - 인스턴스 구성
            - 인스턴스 : `db.t3.micro`
        - 연결
            - VPC: `petclinic-vpc`
            - VPC 보안 그룹: `petclinic-db-sg`
        - 추가 구성
            - 초기 데이터베이스 이름: `petclinic`
            - 자동 백업 활성화 : 해제

## 2단계: PetClinic 애플리케이션 컨테이너 빌드 및 업로드

### STEP 05: PetClinic 이미지 빌드 & ECR 업로드

- CloudShell에 Java 설치
    
    ```bash
    sudo dnf install -y java-17-amazon-corretto-devel
    java -version
    ```
    
- 소스 다운로드
    
    ```bash
    git clone https://github.com/spring-projects/spring-petclinic.git
    cd spring-petclinic
    ```
    
- MySQL 정보 기록
    
    ```bash
    cat > src/main/resources/application-mysql.properties << 'EOF'
    database=mysql
    spring.datasource.url=${MYSQL_URL:jdbc:mysql://<DB-ENDPOINT>:3306/petclinic}
    spring.datasource.username=${MYSQL_USER:petclinic}
    spring.datasource.password=${MYSQL_PASS:petclinic1!}
    spring.sql.init.mode=always
    EOF
    
    cat src/main/resources/application-mysql.properties
    ```
    
- Dockerfile 작성
    
    ```bash
    # 컴파일후 binary jar 파일 생성
    ./mvnw clean package -DskipTests
    
    cat > Dockerfile << 'EOF'
    FROM eclipse-temurin:17-jdk-jammy
    WORKDIR /app
    COPY target/*.jar app.jar
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=mysql", "app.jar"]
    EOF
    
    docker build -t petclinic .
    ```
    
- 이미지 빌드 & ECR Push
    
    ```bash
    AWS_REGION=ap-northeast-2
    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    
    aws ecr get-login-password --region $AWS_REGION \
    | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
    
    # docker build -t petclinic .
    docker images
    docker tag petclinic:latest ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/petclinic:latest
    docker push ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/petclinic:latest
    ```
    

> 2강 완료 : VPC/Subnet · RDS(Available) · ECR(petclinic:latest) · EKS(Node Ready), Petclinic Container build 후 ECR에 저장
> 

---

# 3강 — EKS 배포 및 서비스 운영

## 3단계:  Petclinic 애플리케이션 배포

### STEP 06: Petclinic 애플리케이션 배포

- 애플리케이션 운영을 위한 Namespace 생성
    
    ```bash
    kubectl create namespace petclinic
    kubectl config set-context --current --namespace=petclinic
    ```
    
- Deployment 생성
    
    ```bash
    AWS_REGION=ap-northeast-2
    ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    RDS_ENDPOINT=$(aws rds describe-db-instances \
      --db-instance-identifier petclinic-db \
      --region $AWS_REGION \
      --query 'DBInstances[0].Endpoint.Address' --output text)
    
    cat > deployment.yaml << EOF
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: petclinic-dep
      namespace: petclinic
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
            image: ${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/petclinic:latest
            ports:
            - containerPort: 8080
    EOF
    
    kubectl apply -f deployment.yaml
    kubectl get pods -n petclinic -o wide
    ```
    
- Service(NLB) 생성
    
    ```bash
    cat > service.yaml << 'EOF'
    apiVersion: v1
    kind: Service
    metadata:
      name: petclinic-svc
      namespace: petclinic
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    spec:
      type: LoadBalancer
      selector:
        app: petclinic
      ports:
      - name: http
        port: 80
        targetPort: 8080
    EOF
    
    kubectl apply -f service.yaml
    kubectl get svc petclinic-svc -n petclinic
    ```
    

### STEP 07: 배포 확인 · 웹 브라우저 접속

- 로드밸런서 DNS로 접속
    
    ```bash
    NLB_DNS=$(kubectl get svc petclinic-svc -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
    curl -I http://$NLB_DNS
    ```
    
- 브라우저에서 `http://$NLB_DNS` 접속 → Spring PetClinic 화면 확인
- Petclinic 고객 등록 TEST

- 전체 흐름 : `사용자 → NLB → Service → PetClinic Pod → RDS MySQL`

---

# 실습 자원 정리

> 과금 방지를 위해 반드시 이 순서로 삭제한다.
> 

```bash
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete namespace petclinic

eksctl delete cluster --name petclinic-cluster --region ap-northeast-2

aws ecr delete-repository --repository-name petclinic --force --region ap-northeast-2
```

- RDS `petclinic-db` 삭제
- NAT Gateway 삭제
- Elastic IP 해제
- VPC 삭제 (Subnet/Route Table/IGW 포함)
