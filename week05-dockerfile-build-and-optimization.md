# 애플리케이션 컨테이너 이미지 빌드


## 1. 도커 이미지 빌드

- `docker build` 명령어는 Dockerfile에 기술된 지시어를 순차적으로 실행하여 도커 이미지(Image)를 생성하는 명령어
- 기본 구문
    
    `docker build [옵션] <빌드 컨텍스트 경로>`
    

### 1.1 간단한 컨테이너 이미지 빌드 및 실행

- **docker commit으로 간단한 웹 서버 만들기**
    - debian 리눅스 컨테이너 실행
    - nginx 설치
    - index.html 수정 : 컨테이너의  IP 주소, 컨테이너 이를 출력
    - commit 명령으로 컨테이너 이미지 생성
    
    ```bash
    # 데비안(Debian) 리눅스 환경의 컨테이너를 실행하고 내부로 접속
    docker run -it --name test-commit debian:latest bash
    
    # 컨테이너 내부에서웹 서버(Nginx) 설치 및 설정
    apt-get update && apt-get install -y nginx
    
    # index.html 수정
    echo "<h1>Container Web Server1</h1>" > /var/www/html/index.html
    
    # Nginx 실행 확인 후 컨테이너 내부에서 나오기
    exit
    ```
    
    ```bash
    # commit 명령으로 이미지 생성
    # docker commit [컨테이너명] [새이미지명]:[태그]
    docker commit test-commit webserver:v1
    
    # 생성된 이미지 확인
    docker images
    ```
    
    ```bash
    #생성한 컨테이너 실행 TEST
    # -d: 백그라운드 실행
    # -p 80:80: 호스트 80포트를 컨테이너 80포트로 연결
    docker run -d --name web1 -p 80:80 webserver:v1 nginx -g 'daemon off;'
    docker ps
    ```
    
    - 웹 브라우저로 접속 TEST
        
        <img width="506" height="149" alt="Image" src="https://github.com/user-attachments/assets/0d562134-7f41-427a-9b86-ab5926d639bc" />
        
    
    ```bash
    # 동작중인 컨테이너 삭제
    docker rm -f $(docker ps -aq)
    ```
    

- Docker 파일을 이용해서 생성하는 컨테이너 이미지
    - 컨테이너 빌드 디렉토리 생성
        
        ```bash
        # 작업 디렉토리를 생성하고 Dockerfile을 작성
        mkdir ~/build-test
        cd ~/build-test
        ```
        
    - Dockerfile 작성
        - Dockerfile에 기술된 지시어를 순차적으로 실행하여 도커 이미지(Image)를 생성
        
        ```bash
        cat << 'EOF' > Dockerfile
        FROM debian:latest
        
        RUN apt-get update && apt-get install -y nginx
        RUN echo "<h1>Container Web Server2</h1>" > /var/www/html/index.html
        
        CMD ["nginx", "-g", "daemon off;"]
        EOF
        ```
        
    - 컨테이너 빌드
        
        ```bash
        docker build -t webserver:v2 .
        ```
        
    - 생성된 컨테이너 이미지 확인 후 컨테이너 실행
        
        ```bash
        docker images
        ```
        
        ```bash
        
        IMAGE           ID             DISK USAGE   CONTENT SIZE   EXTRA
        debian:latest   35b8ff74ead4        184MB         52.5MB
        webserver:v1    c461c24eeb20        237MB         70.6MB
        webserver:v2    1fd52afdfaaf        237MB         70.6MB
        ```
        
    - 생성한 컨테이너 실행 TEST
        
        ```bash
        # -d: 백그라운드 실행
        # -p 80:80: 호스트 80포트를 컨테이너 80포트로 연결
        docker run -d --name web2 -p 8080:80 webserver:v2
        docker ps
        ```
        
        웹 브라우저를 통한 연결 TEST  : http://192.168.100.10
        
        <img width="559" height="171" alt="Image" src="https://github.com/user-attachments/assets/1efc20d4-34c6-4b84-9fbf-a138084ee3c1" />
        

<br>


### 1.2. Dockerfile 문법 이해

- 간단한 웹 서버 만들기 : FROM, RUN, CMD
    - Dockerfile 만들기
        
        ```bash
        mkdir ~/build-test/lab1
        cd ~/build-test/lab1/
        ```
        
        ```bash
        cat << 'EOF' > Dockerfile
        # base image 
        FROM debian:latest
        
        # 컨테이너 구성(웹서버 설치)
        RUN apt-get update
        RUN apt-get install -y nginx
        RUN echo "Webserver test1" > /var/www/html/index.html
        
        # container 시작시 실행할 명령어
        CMD ["nginx", "-g", "daemon off;"]
        EOF
        ```
        
        - 컨테이너 빌드
        
        ```bash
        docker build -t lab1:v1 .
        ```
        
        - 컨테이너 레이어 수는?
        
    - 컨테이너 레이어 수 줄이기
        
        ```bash
        cat << 'EOF' > Dockerfile
        # base image 
        FROM debian:latest
        
        # 컨테이너 구성(웹서버 설치)
        RUN apt-get update && apt-get install -y nginx
        RUN echo "Webserver test1" > /var/www/html/index.html
        
        # container 시작시 실행할 명령어
        CMD ["nginx", "-g", "daemon off;"]
        EOF
        ```
        
        - 컨테이너 빌드
            
            ```bash
            docker build -t lab1:v2 .
            ```
            
            - 컨테이너 빌드 속도는?
            
            ```bash
            docker build --no-cache -t lab1:v3 .
            ```
            
            - 컨테이너 빌드 속도는?

- 간단한 웹 서버 만들기 : FROM, RUN, CMD, COPY, ENV, WORKDIR
    - Dockerfile 만들기
        
        ```bash
        mkdir ~/build-test/lab2
        cd ~/build-test/lab2/
        echo "TEST WEB CONTAINER" > index.html
        ```
        
        ```bash
        cat << 'EOF' > Dockerfile
        FROM debian:latest  
        LABEL maintainer="seongmi.lee@gmail.com"
        
        RUN apt-get update && apt-get install -y nginx
        
        ENV DOC_ROOT=/var/www/html
        WORKDIR $DOC_ROOT
        
        COPY index.html .
        EXPOSE 80 
        
        CMD ["nginx", "-g", "daemon off;"] 
        EOF
        ```
        
        - 컨테이너 빌드
        
        ```bash
        docker build -t lab2:v1 .
        ```
        
        ```bash
        docker images
        ```
        

- 인증서 기반의 웹 서버 만들기
    
    ```bash
    mkdir -p ~/build-test/lab3-ssl
    cd ~/build-test/lab3-ssl
    ```
    
    - 호스트에서 사설 인증서 만들기
        
        ```bash
        # openssl로 사설 인증서 생성 (키와 인증서 2개 생성)
        openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
          -keyout server.key -out server.crt \
          -subj "/C=KR/ST=Seoul/L=Seoul/O=Weplat/CN=localhost"
        ```
        
    - Nginx 설정 파일 생성 (default.conf)
        
        ```bash
        cat << 'EOF' > default.conf
        server {
            listen 80;
            listen 443 ssl;
            server_name localhost;
        
            ssl_certificate /etc/nginx/ssl/server.crt;
            ssl_certificate_key /etc/nginx/ssl/server.key;
        
            location / {
                root   /usr/share/nginx/html;
                index  index.html;
            }
        }
        EOF
        ```
        
    - 웹 소스 작성
        
        ```bash
        # index.html
        cat << 'EOF' > index.html
        <h1>Secure Web Server is running with SSL Certificate</h1>
        <p>Verified by Weplat Security Lab</p>
        <script src='script.js'></script>
        EOF
        
        # script.js
        cat << 'EOF' > script.js
        document.getElementById('msg').innerHTML = 'JS File Loaded Successfully via HTTPS!';
        EOF
        
        #압축하기
        tar -cvzf web_src.tar.gz index.html script.js
        ```
        
    - Dockerfile 만들기
        
        ```bash
        FROM debian:latest
        LABEL maintainer="seongmi.lee@gmail.com"
        
        # Nginx 설치
        RUN apt-get update && apt-get install -y nginx
        
        # 인증서 배치
        RUN mkdir -p /etc/nginx/ssl
        COPY server.key /etc/nginx/ssl/server.key
        COPY server.crt /etc/nginx/ssl/server.crt
        
        # nginx 설정 파일 전달
        COPY default.conf /etc/nginx/sites-available/default
        
        # 웹소스 전달. ADD로 자동 압축 해제후 전달
        ADD web_src.tar.gz /usr/share/nginx/html/
        
        EXPOSE 80 443
        
        CMD ["nginx", "-g", "daemon off;"]
        ```
        
    - `.dockerignore` 파일 만들기
        - `default.conf`, `index.html`, `script.js`를 개별적으로 사용하지 않고, **압축한 `web_src.tar.gz` 파일**을 사용
        
        ```bash
        cat << 'EOF' > .dockerignore
        # 빌드에 포함할 필요 없는 원본 파일들 제외
        index.html
        script.js
        
        # 임시 파일 및 로그 제외
        *.log
        .git
        .DS_Store
        EOF
        ```
        
    - 컨테이너 빌드 및 실행 TEST
        
        ```bash
        docker build -t webserver-tls:v1 .
        
        docker images
        ```
        
    - 컨테이너 실행
        
        ```bash
        docker run -d --name secure-web -p 80:80 -p 443:443 webserver-tls:v1
        ```
        
    - 결과 확인
        - 내부 TEST : curl
            
            ```bash
            # -k 옵션은 사설 인증서의 보안 경고를 무시하고 접속하게 해줍니다.
            curl -k https://localhost
            ```
            
        - 외부 접속 TEST : https://ServerIP
            
            <img width="977" height="692" alt="Image" src="https://github.com/user-attachments/assets/c1fbd058-a550-468d-8e9c-7c85f1c9357f" />
            

- 모든 컨테이너 서비스 종료
    
    ```bash
    crm
    ```
    

<br>


### 1.3 🧑‍💻  퀴즈: Rocky Linux 9 기반 보안 웹 서버 구축

- Rocky Linux 9 환경에서 인증서 기반(HTTPS) 웹 서버를 구축합니다.
- 각 단계를 차근차근 따라하며, **Step 5**의 Dockerfile 빈칸을 채워 빌드를 완성하세요.
- **Step 1: 실습 전용 디렉토리 생성**
    - 모든 작업물은 관리의 편의를 위해 지정된 디렉토리에서 진행합니다.
    
    ```bash
    mkdir -p ~/build-test/quiz-container
    cd ~/build-test/quiz-container
    ```
    
- **Step 2: 사설 인증서 생성**
    - 보안 통신을 위한 키와 인증서를 생성합니다.
    - 인증서 파일 이름: webserver.crt, webserver.key
    
    ```bash
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
      -keyout webserver.key -out webserver.crt \
      -subj "/C=KR/ST=Seoul/L=Seoul/O=MyCOMPANY/CN=localhost"
    ```
    
- **Step 3: Nginx 구성 파일 작성 (`nginx.conf`)**
    - SSL 설정을 포함한 서버 설정 파일을 미리 준비합니다.
    
    ```bash
    cat << 'EOF' > nginx.conf
    server {
        listen 80;
        listen 443 ssl;
        server_name localhost;
    
        ssl_certificate /etc/nginx/ssl/webserver.crt;
        ssl_certificate_key /etc/nginx/ssl/webserver.key;
    
        location / {
            root   /usr/share/nginx/html;
            index  index.html;
        }
    }
    EOF
    ```
    
- **Step 4: 나를 소개하는 웹 문서 생성 (`index.html`)**
    
    ```bash
    cat << 'EOF' > index.html
    
    ....
    
    EOF
    ```
    
- **Step 5: Dockerfile 생성**
    - 다음의 괄호를 채워서 Dockerfile을 완성하세요.
    
    ```bash
    # base image 
    [   A   ] rockylinux:9
    
    # maintainer 수정하기
    LABEL maintainer="seongmi.lee@gmail.com"
    
    #패키지 설치 (Nginx 설치)
    RUN [   B   ] install -y nginx
    
    # 인증서 배치를 위한 폴더 생성
    RUN mkdir -p /etc/nginx/ssl
    
    #호스트의 인증서 파일을 컨테이너 내부로 복사
    [   C   ] webserver.key /etc/nginx/ssl/webserver.key
    [   C   ] webserver.crt /etc/nginx/ssl/webserver.crt
    
    # [ D ] Nginx 설정 파일 교체 (Rocky Linux 경로 기준)
    COPY nginx.conf /etc/nginx/conf.d/default.conf
    
    # [ E ] 웹 소스 파일 복사
    COPY index.html /usr/share/nginx/html/index.html
    
    EXPOSE 80 443
    
    # 컨테이너 실행 명령어
    CMD ["nginx", "-g", "daemon off;"]
    ```
    
- Step 6: `.dockerignore` 파일 생성
    - 불필요한 파일이 빌드 컨텍스트에 포함되지 않도록 설정합니다.
    
    ```bash
    cat << 'EOF' > .dockerignore
    *.log
    .git
    .DS_Store
    EOF
    ```
    
- **Step 7: 컨테이너 빌드하기**
    - **컨테이너 이미지 이름:** `my-intro-web:v1`
    
    ```bash
    
    ```
    
- **Step 8: 이미지 확인 및 동작 TEST**
    - **이미지 리스트 확인 및 컨테이너 실행**
    - container name : quiz-web
    
    ```bash
    docker images
    docker run -d --name quiz-web -p 80:80 -p 443:443 my-intro-web:v1
    ```
    
    - 로컬호스트에서 확인
    
    ```bash
    curl -k https://localhost
    ```
    
    - 웹브라우저를 통해 외부 접속 확인: https://IP_Address


---
<br>

## **2. 애플리케이션 소스 코드를 포함한 컨테이너 빌드하기**

### 2.1 Node.js 애플리케이션 컨테이너 빌드

- node.js 소스 코드
    
    ```bash
    mkdir ~/build-test/app-node
    cd ~/build-test/app-node/
    ```
    
    - node source code : **app.js**
    
    ```bash
    const http = require('http');
    const os = require('os');
    
    const handler = function(req, res) {
      res.setHeader('Content-Type', 'text/html; charset=utf-8');
      res.writeHead(200);
      
      // 스타일을 한 줄로 넣어서 길이를 대폭 줄였습니다.
      res.end(`
        <body style="text-align:center; padding-top:50px; font-family:sans-serif;">
          <h1 style="color:navy;">🚀 Node.js App is Running</h1>
          <h2 style="background:yellow; display:inline-block; padding:10px;">
            Host(Container) name: ${os.hostname()}
          </h2>
          <p style="color:gray;">Verified by Weplat</p>
        </body>
      `);
    };
    
    http.createServer(handler).listen(8080);
    ```
    
- Dockerfile 만들기
    - dockerfile
    
    ```bash
    # 베이스 이미지: 보안과 용량을 위해 alpine 사용
    FROM node:20-alpine
    
    # 메타데이터 추가
    LABEL maintainer="seongmi.lee@weplat.co.kr"
    LABEL description="Node.js 실무 예제"
    
    # 작업 디렉토리 설정 (루트 경로 사용 지양)
    WORKDIR /app
    
    # 소스 코드 복사 (소유권 설정을 통해 보안 강화)
    # --chown 옵션으로 권한 관리 (root 계정 사용 지양)
    COPY --chown=node:node app.js .
    
    # 보안: root 대신 node 일반 사용자 계정 사용
    USER node
    
    # 환경 변수 설정
    ENV PORT=8080
    
    # 포트 명시
    EXPOSE 8080
    
    # 실행 명령어 (ENTRYPOINT와 CMD의 조합)
    ENTRYPOINT ["node"]
    CMD ["app.js"]
    ```
    
- 컨테이너 빌드 및 TEST
    
    ```bash
    # 이미지 빌드
    docker build -t node-app:v1 .
    
    # 컨테이너 실행 (포트 포워딩 적용)
    docker run -d --name my-node-app -p 8080:8080 node-app:v1
    
    # 결과 확인
    curl localhost:8080
    ```
    
- 브라우저를 통해 확인
    
    <img width="870" height="269" alt="Image" src="https://github.com/user-attachments/assets/298cc960-4bbc-464a-8910-cc789fdff149" />
    
- 컨테이너 종료
    
    ```bash
    crm
    ```
    
<br>

### 2.2 🧑‍💻  퀴즈: Python 게임 애플리케이션 이미지 빌드 및 배포

- 직접 작성한 파이썬 소스 코드를 Dockerfile을 통해 이미지로 만들고, 컨테이너로 배포합니다.

**Step 1: 실습 전용 디렉토리 생성**

- 새로운 실습을 위해 깔끔한 공간을 먼저 마련합니다.
    
    ```bash
    mkdir -p ~/build-test/app-python && cd ~/build-test/app-python
    ```
    

**Step 2: Python 게임 소스 작성 (`box_game.py`)**

- 아래 명령어를 복사하여 게임 서버 코드를 생성합니다. (내용을 읽어보며 80번 포트가 사용됨을 확인하세요.)
    
    ```bash
    cat << 'EOF' > box_game.py
    from http.server import BaseHTTPRequestHandler, HTTPServer
    import os
    
    GAME_TITLE = "Weplat Python Game (Build Ver)"
    BOX_SPEED = 5
    SPAWN_RATE = 800
    
    GAME_HTML = f"""
    <!DOCTYPE html>
    <html>
    <head>
        <meta charset="UTF-8">
        <title>{GAME_TITLE}</title>
        <style>
            body {{ background: #222; color: white; text-align: center; font-family: sans-serif; }}
            #game {{ width: 400px; height: 500px; border: 2px solid white; margin: 20px auto; position: relative; overflow: hidden; }}
            #player {{ width: 30px; height: 30px; background: #4caf50; position: absolute; bottom: 10px; left: 185px; }}
            .box {{ width: 40px; height: 40px; background: #f44336; position: absolute; }}
        </style>
    </head>
    <body>
        <h1>{GAME_TITLE} 🐍</h1>
        <p>Container Hostname: {os.uname()[1]}</p>
        <div id="game"><div id="player"></div></div>
        <script>
            const speed = {BOX_SPEED}; const spawnRate = {SPAWN_RATE};
            const game = document.getElementById("game");
            const player = document.getElementById("player");
            let px = 185; let gameOver = false;
            document.addEventListener("keydown", (e) => {{
                if(gameOver) return;
                if(e.key === "ArrowLeft" && px > 0) px -= 20;
                if(e.key === "ArrowRight" && px < 370) px += 20;
                player.style.left = px + "px";
            }});
            function createBox() {{
                if(gameOver) return;
                const b = document.createElement("div");
                b.className = "box"; b.style.left = Math.random() * 360 + "px";
                game.appendChild(b);
                let by = 0;
                const move = setInterval(() => {{
                    if(gameOver) {{ clearInterval(move); b.remove(); return; }}
                    by += speed; b.style.top = by + "px";
                    const pRect = player.getBoundingClientRect();
                    const bRect = b.getBoundingClientRect();
                    if (bRect.bottom >= pRect.top && bRect.top <= pRect.bottom && bRect.left <= pRect.right && bRect.right >= pRect.left) {{
                        gameOver = true; alert("💥 GAME OVER!"); location.reload();
                    }}
                    if(by > 500) {{ b.remove(); clearInterval(move); }}
                }}, 20);
            }}
            setInterval(createBox, spawnRate);
        </script>
    </body>
    </html>
    """
    
    class Handler(BaseHTTPRequestHandler):
        def do_GET(self):
            self.send_response(200)
            self.send_header('Content-type', 'text/html; charset=utf-8')
            self.end_headers()
            self.wfile.write(GAME_HTML.encode('utf-8'))
    
    print(f"[{GAME_TITLE}] Server starting on port 80...")
    HTTPServer(('0.0.0.0', 80), Handler).serve_forever()
    EOF
    ```
    

Step 3: Dockerfile 생성

- base image: python:3.9-slim
- CMD : python3  box_game.py
- Expose port : 80

Step 4: 이미지 빌드 및 컨테이너 실행

- **이미지 빌드** (이름: `python-game`, 태그: `v1`)
- **컨테이너 실행** (호스트 포트 **8089**를 컨테이너 포트 **80**에 연결하세요)

**Step 5: 결과 확인 및 업데이트 실습**

- **결과 확인**: 웹 브라우저에서 `http://서버IP:8089`에 접속하여 게임이 돌아가는지 확인합니다.


---
<br>

## **3. Best Practice 집약형 실무 Dockerfile 실습**

### 3.1 Dockerfile에 Best Practice 예

- 이미지 경량화 및 보안 (Rule 1, 5 적용)
    
    ```bash
    FROM node:20
    WORKDIR /app
    COPY app.js .
    # 기본적으로 root 계정으로 실행됨
    CMD ["node", "app.js"]
    ```
    
    - base image의 크기 줄이기 : node:20-alpine
    - 기본적으로 root 계정으로 실행되지 않도록 구성하기
    
    ```bash
    # 1. 경량화된 공식 이미지 사용 (Rule 1)
    FROM node:20-alpine
    WORKDIR /app
    
    # 2. 보안을 위해 일반 사용자 계정 사용 (Rule 5)
    COPY --chown=node:node app.js .
    USER node
    CMD ["node", "app.js"]
    ```

<br>
    
- 레이어 최적화 및 캐싱 활용 (Rule 2 적용)
    
    ```bash
    FROM node:20-alpine
    WORKDIR /app
    # 소스 전체를 먼저 복사하면, 코드 한 줄만 바꿔도 npm install이 매번 다시 실행됨
    COPY . .
    RUN npm install
    CMD ["node", "app.js"]
    ```
    
    - 빌드 속도 최적화
    - 소스 코드가 수정되어도 라이브러리 설치(`npm install`) 과정이 재사용되도록 레이어 순서를 최적화
    
    ```bash
    FROM node:20-alpine
    WORKDIR /app
    # 1. 잘 변하지 않는 종속성 파일을 먼저 복사 (Rule 2)
    COPY package.json .
    RUN npm install
    # 2. 자주 변하는 소스 코드는 마지막에 복사
    COPY . .
    CMD ["node", "app.js"]
    ```
    
<br>

- 유연한 명령어 실행 (Rule 6 적용)
    
    ```bash
    FROM python:3.9-slim
    WORKDIR /app
    COPY . .
    # 실행 명령어가 통째로 고정되어 인자값 변경이 어려움
    CMD ["python", "app.js"]
    ```
    
    - 컨테이너 실행 시 기본 실행 파일은 고정하되, 인자값만 바꿔서 유연하게 대응할 수 있도록 수정
    
    ```bash
    FROM python:3.9-slim
    WORKDIR /app
    COPY . .
    # 1. 실행 바이너리는 ENTRYPOINT로 고정 (Rule 6)
    ENTRYPOINT ["python"]
    # 2. 기본 실행 파일은 CMD로 제공 (사용자가 변경 가능)
    CMD ["app.js"]
    ```

<br>

- Multi-stage Build 적용 (Rule 7 적용)
    
    ```bash
    # 빌드 도구가 포함된 큰 이미지가 그대로 최종 이미지가 됨
    FROM node:20
    WORKDIR /app
    COPY . .
    RUN npm install && npm run build
    CMD ["node", "dist/main.js"]
    ```
    
    - Multi-stage Build
    
    ```bash
    # 1. Build Stage: 빌드 도구가 포함된 이미지 사용 (Rule 7)
    FROM node:20 AS builder
    WORKDIR /app
    COPY . .
    RUN npm install && npm run build
    
    # 2. Run Stage: 실제 실행에 필요한 최소 환경만 구성
    FROM node:20-alpine
    WORKDIR /app
    # 빌드 결과물만 쏙 빼오기
    COPY --from=builder /app/dist ./dist
    COPY --from=builder /app/node_modules ./node_modules
    USER node
    CMD ["node", "dist/main.js"]
    ```
    
<br>

### 3.2 컨테이너 빌드

- **실습 환경 준비**
    
    ```bash
    mkdir -p ~/build-test/compare-build && cd ~/build-test/compare-build
    
    # 간단한 앱 소스
    cat << 'EOF' > app.js
    const http = require('http');
    http.createServer((req, res) => {
      res.writeHead(200);
      res.end("Build Comparison Test\n");
    }).listen(8080);
    console.log("Server running on port 8080");
    EOF
    
    # 패키지 정보 (빌드 과정 재현용)
    cat << 'EOF' > package.json
    {
      "name": "compare-build",
      "version": "1.0.0",
      "main": "app.js",
      "dependencies": {
        "express": "^4.18.2"
      }
    }
    EOF
    ```
    
- Dockerfile 생성  (`Dockerfile.single`)
    
    ```bash
    # Dockerfile.single
    FROM node:20
    
    WORKDIR /app
    COPY package.json .
    
    # 빌드 도구와 라이브러리가 설치됨
    RUN npm install 
    
    COPY . .
    
    EXPOSE 8080
    CMD ["node", "app.js"]
    ```
    
- 빌드 실행
    
    ```bash
    docker build -f Dockerfile.single -t node-app:single .
    ```
    
<br>

### 3.3 Best Practice 적용된 컨테이너 빌드

- Dockerfile 생성  (`Dockerfile.multi`)
    
    ```bash
    # Dockerfile.multi
    
    # --- Stage 1: Build (빌드 도구 포함) ---
    FROM node:20 AS builder
    WORKDIR /build
    COPY package.json .
    RUN npm install
    COPY . .
    
    # --- Stage 2: Run (실행에 필요한 최소 환경) ---
    FROM node:20-alpine
    WORKDIR /app
    # 빌드 스테이지에서 설치된 결과물만 쏙 빼옴!
    COPY --from=builder /build /app
    
    EXPOSE 8080
    USER node
    CMD ["node", "app.js"]
    ```
    
- 빌드 실행
    
    ```bash
    docker build -f Dockerfile.multi -t node-app:multi .
    ```
    
<br>

### 3.4 결과 확인

- 빌드가 완료되면 아래 명령어로 용량 차이를 확인
    
    ```bash
    docker images | grep node-app
    ```
    
    ```bash
    node-app:multi     6627b7095927        196MB         49.1MB
    node-app:single    c50e62ff3327       1.57GB          401MB
    ```
    
- 컨테이너 동작 확인
    
    ```bash
    # 1. 일반 빌드 컨테이너 실행 (8081 포트)
    docker run -d --name app-single -p 8081:8080 node-app:single
    
    # 2. 멀티 스테이지 빌드 컨테이너 실행 (8082 포트)
    docker run -d --name app-multi -p 8082:8080 node-app:multi
    ```
    
- 웹 브라우저를 통한 접속 TEST
    
   <img width="423" height="103" alt="Image" src="https://github.com/user-attachments/assets/4dde3f9a-9414-4e9b-a321-1cce516004ff" />
   <img width="401" height="129" alt="Image" src="https://github.com/user-attachments/assets/71311610-f30b-4447-bbd1-85d898f1f3b8" />
    


