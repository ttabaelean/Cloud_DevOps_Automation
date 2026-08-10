# 애플리케이션 컨테이너 이미지 빌드

## **1. Docker Image와 Dockerfile 이해**

### docker build 명령어

- `docker build` 명령어는 Dockerfile에 기술된 지시어를 순차적으로 실행하여 **도커 이미지(Image)**를 생성하는 명령어
- 기본 구문
    
    docker build [옵션] <빌드 컨텍스트 경로>
    

### **Hands-on Labs : 간단한 컨테이너 이미지 빌드 및 실행**

- **Build Context 구성**
    
    ```bash
    # 실습 디렉터리를 생성
    mkdir -p ~/build/myweb
    cd ~/build/myweb
    
    #웹 페이지를 생성
    cat > index.html <<'EOF'
    <h1>Hello Docker!</h1>
    <h2>My First Docker Image</h2>
    EOF
    
    #현재 파일을 확인
    tree ~/build/
    ```
    
- **Dockerfile 작성**
    
    ```bash
    # httpd 이미지를 기반으로 새로운 이미지를 생성
    cat > Dockerfile <<'EOF'
    FROM rockylinux/rockylinux:10
    RUN dnf install -y httpd
    COPY index.html /var/www/html/index.html
    EXPOSE 80
    CMD ["/usr/sbin/httpd", "-D", "FOREGROUND"]
    EOF
    
    # Dockerfile 확인
    cat Dockerfile
    tree ~/build/
    ```
    
- **Docker Image 빌드**
    
    ```bash
    # 현재 디렉터리를 Build Context로 지정하여 이미지를 생성
    docker build -t myweb:v1 .
    
    # 빌드 된 이미지를 확인
    docker images myweb:v1
    
    # 이미지의 Layer를 확인
    docker history myweb:v1
    ```
    
- **Container 실행**
    - 컨테이너 이름: myweb
    - 포트매핑 : -p 80:80
    
    ```bash
    docker run -d  --name myweb -p 80:80 myweb:v1
    
    # 실행 상태 확인
    docker ps
    
    # 웹 브라우저에서도 접속
    curl http://localhost
    
    # 컨테이너 종료
    docker rm -f myweb
    ```
    

## **2. 애플리케이션 컨테이너 이미지 빌드**

### 웹 서버 컨테이너 이미지 빌드

- Dockerfile 만들기
    
    ```bash
    mkdir -p ~/build/webserver
    cd ~/build/webserver/
    echo "<h1> Web Site </h1>" > index.html
    
    ```
    
    ```bash
    cat << 'EOF' > Dockerfile
    FROM debian:latest  
    LABEL maintainer="seongmi.lee@gmail.com"
    
    RUN apt-get update && \
        apt-get install -y nginx
    
    ENV DOC_ROOT=/var/www/html
    WORKDIR $DOC_ROOT
    
    COPY index.html .
    EXPOSE 80 
    
    CMD ["nginx", "-g", "daemon off;"] 
    EOF
    ```
    
- 컨테이너 빌드
    
    ```bash
    docker build -t webserver:v1  .
    docker images
    ```
    
- 컨테이너 실행
    
    ```bash
    docker run -d --name webserver -p 80:80  webserver:v1
    docker ps
    curl http://localhost
    ```
    
- 컨테이너 삭제
    
    ```bash
    docker rm -f webserver
    docker ps -a
    ```
    

### Node.js 애플리케이션 이미지 빌드

- app.js 소스 코드
    
    ```bash
    mkdir -p ~/build/app-node
    cd ~/build/app-node/
    ```
    
    ```bash
    cat > app.js << "EOF"
    const http = require('http');
    const os = require('os');
    http.createServer((req, res) => {
      res.writeHead(200, { 'Content-Type': 'text/html; charset=utf-8' });
      res.end(`
        <h1>🚀 Node.js App is Running</h1>
        <h2>Host: ${os.hostname()}</h2> 
       <p>Verified by Weplat</p>
      `);
    }).listen(8080);
    EOF
    ```
    
- Dockerfile 만들기
    
    ```bash
    cat << EOF > Dockerfile
    FROM node:22-alpine
    
    LABEL maintainer="seongmi lee"
    LABEL description="Node.js 실무 예제"
    
    WORKDIR /app
    
    # --chown 옵션으로 권한 관리 (root 계정 사용 지양)
    COPY --chown=node:node app.js .
    USER node
    
    ENV PORT=8080
    EXPOSE 8080
    
    ENTRYPOINT ["node"]
    CMD ["app.js"]
    EOF
    ```
    
- 컨테이너 빌드 및 결과 확인
    
    ```bash
    # 이미지 빌드
    docker build -t node-app:v1 .
    
    # 컨테이너 실행 (포트 포워딩 적용)
    docker run -d --name my-node-app -p 8080:8080 node-app:v1
    
    # 결과 확인
    curl localhost:8080
    ```
    
- 브라우저를 통해 확인 : http://192.168.100.10:8080
- 컨테이너 종료
    
    ```bash
    docker ps  -a
    docker stop my-node-app
    docker rm -f my-node-app
    ```
    

### Hands-on  labs: Spring Petclinic 애플리케이션 이미지 빌드

- 실습 목표
    - Spring PetClinic 소스 코드를 다운로드한다.
    - Maven을 이용하여 실행 가능한 JAR 파일을 생성한다.
    - Dockerfile을 작성하여 PetClinic Container Image를 생성한다.
    - 생성된 Image를 Container로 실행한다.
- Spring PetClinic 다운로드
    
    ```
    # git과 java 설치
    sudo dnf install -y git java-21-openjdk-devel
    
    # 소스코드 다운로드
    cd ~/build
    git clone https://github.com/spring-projects/spring-petclinic.git
    cd spring-petclinic
    ```
    
- Spring PetClinic Build
    
    ```
    # Maven으로 애플리케이션을 빌드합니다.
    ./mvnw clean package -DskipTests
    
    # 생성된 JAR 확인
    ls -lh target/*.jar
    ```
    
- Dockerfile 작성
    
    ```bash
    cat > Dockerfile <<'EOF'
    FROM eclipse-temurin:21-jdk
    WORKDIR /app
    COPY . .
    RUN ./mvnw clean package -DskipTests && \
        cp target/*.jar app.jar
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "app.jar"]
    EOF
    ```
    
- Container Image Build
    
    ```
    docker build -t petclinic:v1 .
    
    # Image 확인
    docker images petclinic
    ```
    
- Container 실행
    
    ```
    docker run -d --name petclinic \
       -p 8080:8080 \
       petclinic:v1
    
    # Container 상태 확인
    docker ps -a
    
    # 로그 확인
    docker logs -f petclinic
    ```
    
- PetClinic 접속 확인 : http://192.168.100.10:8080
- Container 종료
    
    ```
    docker rm -f petclinic
    ```
    

## 3. **Docker 이미지 최적화**

### **경량의 컨테이너 이미지 빌드**

- 애플리케이션 준비
    
    ```bash
    mkdir -p ~/build/nodeapp-multi/src
    cd ~/build/nodeapp-multi
    
    cat > src/app.ts <<'EOF'
    import * as http from 'http';
    import * as os from 'os';
    
    http.createServer((req, res) => {
      res.writeHead(200, { 'Content-Type': 'text/plain' });
      res.end(
        `Node.js App is Running\nHost: ${os.hostname()}\nSoongsil Cyber University`
      );
    }).listen(8080);
    EOF
    
    # package.json을 작성
    cat > package.json <<'EOF'
    {
      "name": "nodeapp",
      "version": "1.0.0",
      "scripts": {
        "build": "tsc"
      },
      "devDependencies": {
        "@types/node": "^22.0.0",
        "typescript": "^5.0.0"
      }
    }
    EOF
    
    # tsconfig.json을 작성
    cat > tsconfig.json <<'EOF'
    {
      "compilerOptions": {
        "target": "ES2020",
        "module": "CommonJS",
        "outDir": "dist"
      },
      "include": ["src"]
    }
    EOF
    
    ```
    
- 일반 Image Build
    
    ```bash
    cat > Dockerfile <<'EOF'
    FROM node:22
    
    WORKDIR /app
    
    COPY package.json .
    RUN npm install
    
    COPY src ./src
    COPY tsconfig.json .
    
    RUN npm run build
    
    EXPOSE 8080
    
    CMD ["node", "dist/app.js"]
    EOF
    
    docker build -t nodeapp:v1 .
    ```
    
- 경량 Base Image Build
    
    ```bash
    cat > Dockerfile <<'EOF'
    FROM node:22-alpine
    
    WORKDIR /app
    
    COPY package.json .
    RUN npm install
    
    COPY src ./src
    COPY tsconfig.json .
    
    RUN npm run build
    
    EXPOSE 8080
    
    CMD ["node", "dist/app.js"]
    EOF
    
    docker build -t nodeapp:v2 .
    ```
    
- Multi-stage Image Build
    
    ```bash
    cat > Dockerfile <<'EOF'
    # Stage 1. Builder
    FROM node:22 AS builder
    
    WORKDIR /app
    
    COPY package.json .
    RUN npm install
    
    COPY src ./src
    COPY tsconfig.json .
    
    RUN npm run build
    
    # Stage 2. Runtime
    FROM node:22-alpine
    
    WORKDIR /app
    
    COPY --from=builder /app/dist ./dist
    
    EXPOSE 8080
    
    CMD ["node", "dist/app.js"]
    EOF
    
    docker build -t nodeapp:multi .
    ```
    

### Hands-on labs : **Petclinic - multi-stage 이미지 빌드 실행**

- 실습 목표
    - Maven을 Host에 직접 사용하지 않고 **Docker Build 과정 안에서 Spring PetClinic을 Build**
    - 
- Multi-stage Dockerfile 작성
    
    ```bash
    cat > Dockerfile <<'EOF'
    FROM eclipse-temurin:21-jdk AS builder
    WORKDIR /build
    COPY . .
    RUN ./mvnw clean package -DskipTests
    
    # Stage 2. Runtime
    FROM eclipse-temurin:21-jre
    WORKDIR /app
    COPY --from=builder /build/target/*.jar app.jar
    EXPOSE 8080
    ENTRYPOINT ["java", "-jar", "app.jar"]
    EOF
    ```
    
- Multi-stage Image Build
    
    ```
    docker build -t petclinic:multi .
    
    # Image 확인
    docker images petclinic
    ```
    
- Multi-stage Container 실행
    
    ```
    docker run -d --name petclinic -p 8080:8080 \
       petclinic:multi
      
    # 확인
    docker ps
    docker logs -f petclinic
    ```
    
- 애플리케이션 접속 - 웹 브라우저 : http://192.168.100.10:8080
- Multi-stage Container 종료
    
    ```
    docker rm -f petclinic
    ```
