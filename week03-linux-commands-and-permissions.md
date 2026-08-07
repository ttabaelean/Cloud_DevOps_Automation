# 3주차 Hands-on Labs: 리눅스 기본 명령어 및 운영체제 조작

이 문서는 **3주차 강의 자료**에 수록된 SSH 원격 접속, 파일 시스템 관리, 로그 모니터링, 그리고 서비스 및 데몬 관리 실습 절차를 정리한 가이드입니다.

---

## LAB 1: SSH 원격 접속 및 시스템 정보 확인
### 1. MobaXterm을 이용한 SSH 접속
- **Session** -> **SSH** 선택 후 접속 정보 설정:
  - **Remote host**: `192.168.100.10`
  - **Specify username**: `admin`
  - **Password**: `admin123!`

### 2. 시스템 정보 확인 명령어 실행
```bash
whoami      # 현재 로그인 사용자 확인
hostname    # 서버 이름 확인
pwd         # 현재 작업 디렉터리 위치 확인
date        # 현재 날짜 및 시간 확인
ip a        # 네트워크 인터페이스 및 IP 확인
df -h       # 디스크 사용량 확인
free -h     # 메모리 사용량 확인
```

---

# LAB 2: 파일 시스템 관리 및 로그 모니터링

## 개발팀 프로젝트 디렉터리 구성 및 실시간 로그 모니터링

### 1. 프로젝트 디렉터리 생성 및 이동
```bash
mkdir ~/project
cd ~/project
pwd
```

### 2. 파일 생성
```bash
touch app.log
touch config.txt
ls -l
```

### 3. 백업 파일 복사
```bash
cp config.txt config.bak
ls -l
```

### 4. 백업 디렉터리 생성 및 파일 이동
```bash
mkdir backup
mv config.bak backup/
ls -l
ls backup
```

### 5. 파일 내용 작성 및 확인
```bash
echo "server.port=8080" > config.txt
echo "spring.application.name=demo" >> config.txt
cat config.txt
less config.txt
```

### 6. 실시간 로그 모니터링 (`tail -f`)
```bash
tail -f app.log
```

### 7. (다른 터미널 창) 로그 데이터 추가 및 확인
```bash
cd ~/project
echo "Application Started" >> app.log
echo "User Login Success" >> app.log
echo "ERROR: Database Connection Failed" >> app.log
```
> **참고**: `tail -f` 실행 중인 터미널은 `Ctrl + C` 키를 입력하여 종료합니다.

---

# LAB 3: 서비스(Service) 및 데몬(Daemon) 관리

## Apache(httpd) 웹 서버 설치, 서비스 자동 등록 및 방화벽 설정

### 1. Apache 패키지 설치
```bash
sudo dnf install -y httpd
```

### 2. 서비스 상태 확인 및 시작
```bash
systemctl status httpd
sudo systemctl start httpd
systemctl status httpd
```

### 3. 기본 웹 페이지 생성 및 로컬 동작 확인
```bash
echo "<h1> Web Site </h1>" | sudo tee /var/www/html/index.html
curl http://localhost
```

### 4. 부팅 시 자동 실행 설정 및 상태 확인
```bash
sudo systemctl enable httpd
systemctl is-enabled httpd
```

### 5. 방화벽 HTTP 서비스 허용
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### 6. 외부 접속 확인
- 호스트 PC의 웹 브라우저 주소창에 `http://192.168.100.10` 입력 후 접속을 확인합니다.


