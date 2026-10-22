
# 6주차 Hands-on Labs: 클라우드 인프라 아키텍처 이해

## 실습 환경

| 항목 | 구성 |
|---|---|
| 실습 도구 | AWS Management Console, MobaXterm |
| AWS 리전 | 아시아 태평양(서울) `ap-northeast-2` |
| VPC | `lab-vpc` · `10.0.0.0/16` |
| 가용 영역 | `ap-northeast-2a`, `ap-northeast-2c` |
| 서브넷 | Public 2개 · Private 2개 |
| EC2 | Amazon Linux 2023 · `t3.micro` |
| 웹 서버 | Apache HTTP Server (`httpd`) · HTTP 80 |
| 접속 | SSH 22 · Key Pair `devops-key` |

> [!WARNING]
> AWS는 사용한 리소스에 따라 요금이 발생할 수 있습니다.
>
> 계정의 무료 이용 자격과 크레딧을 먼저 확인하고, 실습이 끝나면 EC2와 VPC 등 생성한 리소스를 반드시 삭제합니다.
>
> **NAT Gateway는 이번 웹 서버 실습에 필요하지 않으므로 생성하지 않습니다.** NAT Gateway는 별도 비용이 발생합니다.

---

## 6주 1강 Hands-on Labs : AWS 계정 보안

### 1단계: AWS 계정 가입

**준비물**

- 본인 이메일 주소
- 휴대폰 번호
- 본인 명의 신용카드 또는 체크카드
- MFA 인증을 위한 스마트폰

**① AWS 가입 페이지 접속**

- [AWS 공식 홈페이지](https://aws.amazon.com/ko/) 접속
- **[AWS 계정 생성]** 선택
- 이메일 주소와 AWS 계정 이름 입력
- 이메일 인증 및 Root 사용자 비밀번호 설정

**② 계정 정보 등록**

- 개인 계정 또는 비즈니스 계정 선택
- 연락처 정보 입력
- 결제 정보 등록
- 전화번호 또는 문자메시지를 통한 본인 확인
- 지원 플랜 선택 후 가입 완료

> [!NOTE]
> 신규 고객에게 제공되는 무료 계정 및 크레딧 정책은 가입 시점과 계정 유형에 따라 달라질 수 있습니다.
>
> 가입 화면에서 무료 계정 조건, 크레딧, 만료일을 확인합니다. 무료 이용 자격이 있어도 모든 AWS 서비스가 무료인 것은 아닙니다.

### 2단계: AWS Root 계정 로그인

**① AWS Management Console 접속**

- [AWS Management Console](https://console.aws.amazon.com/) 접속
- **Root 사용자** 선택
- 가입할 때 사용한 이메일 주소 입력
- 비밀번호 입력 후 로그인

**② Root 계정 확인**

Root 계정은 AWS 계정의 모든 리소스와 결제 정보에 접근할 수 있는 최고 권한 계정입니다.

- AWS 계정 설정 및 결제 정보 관리
- IAM 사용자와 권한 관리
- AWS 리소스 생성 및 삭제

> [!WARNING]
> Root 계정은 평상시 EC2, VPC 실습에 사용하지 않는 것이 좋습니다.
>
> Root 계정에 MFA를 설정하고, 일반적인 실습은 필요한 권한을 부여받은 IAM 사용자 또는 IAM Identity Center 사용자로 진행합니다.

### 3단계: Root 계정 MFA 설정

**MFA(Multi-Factor Authentication)**

비밀번호 외에 추가 인증 수단을 사용하여 계정 탈취 위험을 줄이는 보안 기능입니다.

**① MFA 설정 메뉴 이동**

- AWS 콘솔 오른쪽 위 계정 메뉴 선택
- **보안 자격 증명(Security credentials)** 선택
- **Multi-factor authentication (MFA)** 항목 확인
- **[MFA 디바이스 할당]** 선택

**② MFA 디바이스 등록**

- 디바이스 이름 입력 (예: `root-mfa`)
- MFA 방식 선택
  - 인증 앱(Authenticator app)
  - 보안 키 또는 Passkey
  - 하드웨어 TOTP 토큰

이번 실습에서는 스마트폰 인증 앱을 사용합니다.

**③ 인증 앱 연결**

- 스마트폰에 인증 앱 설치
- AWS 화면에 표시된 QR 코드 스캔
- 인증 앱에서 생성된 MFA 코드 입력
- 화면 안내에 따라 인증 완료

**④ 설정 확인**

- MFA 디바이스가 등록되었는지 확인
- 로그아웃 후 다시 로그인
- 비밀번호 외에 MFA 인증을 요청하는지 확인

> [!NOTE]
> Root MFA 설정은 계정 보안의 기본입니다.
>
> MFA 인증 수단을 분실하면 계정 복구 절차가 필요할 수 있으므로, AWS의 권장 방식에 따라 복구 수단을 안전하게 관리합니다.

---

## 6주 2강 Hands-on Labs : VPC 네트워크 구축

### 실습 아키텍처

```text
AWS Region: ap-northeast-2
└── VPC: lab-vpc (10.0.0.0/16)
    │
    ├── Availability Zone: ap-northeast-2a
    │   ├── Public Subnet  : 10.0.1.0/24
    │   └── Private Subnet : 10.0.3.0/24
    │
    └── Availability Zone: ap-northeast-2c
        ├── Public Subnet  : 10.0.2.0/24
        └── Private Subnet : 10.0.4.0/24

Internet
   │
Internet Gateway
   │
Public Route Table
   ├── Public Subnet (2a)
   └── Public Subnet (2c)
```

**실습 목표**

- VPC 생성
- 2개 가용 영역에 Public/Private Subnet 구성
- Internet Gateway와 라우팅 테이블 확인
- EC2 웹 서버를 배포할 네트워크 준비

### 1단계: AWS 리전 설정

- AWS Management Console 로그인
- 오른쪽 위 리전 선택
- **아시아 태평양(서울) / ap-northeast-2** 선택

> [!NOTE]
> VPC는 리전 단위 리소스입니다. 이후 EC2 실습도 같은 서울 리전에서 진행합니다.

### 2단계: VPC 생성

**① VPC 콘솔 이동**

- AWS 검색창에서 `VPC` 검색
- **VPC → VPC 생성(Create VPC)** 선택
- 생성 옵션에서 **VPC 등(VPC and more)** 선택

**② VPC 설정**

| 항목 | 설정값 |
|---|---|
| 생성할 리소스 | VPC 등 |
| 이름 태그 자동 생성 | `lab` |
| IPv4 CIDR | `10.0.0.0/16` |
| IPv6 CIDR | 없음 |
| 테넌시 | 기본값(Default) |
| 가용 영역 수 | 2 |
| AZ | `ap-northeast-2a`, `ap-northeast-2c` |
| 퍼블릭 서브넷 수 | 2 |
| 프라이빗 서브넷 수 | 2 |
| NAT Gateway | 없음 |
| VPC Endpoint | 없음 |

> [!IMPORTANT]
> 이름 태그 자동 생성 값을 `lab`로 설정하면 실제 VPC 이름은 `lab-vpc`로 생성되는지 미리 보기에서 확인합니다.
>
> AWS 콘솔의 이름 생성 방식은 화면 버전에 따라 다를 수 있습니다.

**③ 서브넷 CIDR 설정**

VPC 생성 화면의 서브넷 CIDR 사용자 지정 항목에서 다음 값을 지정합니다.

| 가용 영역 | 유형 | CIDR |
|---|---|---|
| ap-northeast-2a | Public | `10.0.1.0/24` |
| ap-northeast-2c | Public | `10.0.2.0/24` |
| ap-northeast-2a | Private | `10.0.3.0/24` |
| ap-northeast-2c | Private | `10.0.4.0/24` |

- 설정 확인
- **[VPC 생성]** 선택
- 리소스 생성 완료까지 대기

### 3단계: VPC 및 서브넷 확인

**① VPC 확인**

- VPC → **내 VPC**
- `lab-vpc` 선택

확인 항목:

- VPC 이름 : `lab-vpc`
- IPv4 CIDR : `10.0.0.0/16`
- 상태 : Available

**② 서브넷 확인**

- VPC → **서브넷**
- `lab-vpc`에 속한 서브넷 4개 확인

확인 항목:

- Public Subnet 2개
- Private Subnet 2개
- 각 서브넷의 가용 영역과 CIDR

> [!NOTE]
> 서브넷 이름만으로 Public/Private 여부가 결정되는 것은 아닙니다.
>
> 인터넷 게이트웨이로 연결되는 기본 경로가 있는 라우팅 테이블에 연결되어 있어야 Public Subnet으로 사용할 수 있습니다.

### 4단계: Internet Gateway 확인

**① Internet Gateway 확인**

- VPC → **인터넷 게이트웨이**
- `lab-vpc`에 연결된 Internet Gateway 선택

확인 항목:

- 상태 : Attached
- 연결된 VPC : `lab-vpc`

**② Internet Gateway 역할**

Internet Gateway는 VPC와 인터넷 사이의 통신 경로를 제공합니다.

Public EC2가 인터넷과 통신하려면 다음 조건이 필요합니다.

1. Internet Gateway가 VPC에 연결되어 있어야 함
2. Public Subnet 라우팅 테이블에 인터넷 경로가 있어야 함
3. EC2에 Public IPv4 주소가 있어야 함
4. Security Group에서 필요한 통신이 허용되어야 함

### 5단계: 라우팅 테이블 확인

- VPC → **라우팅 테이블**
- `lab-vpc`의 Public Route Table 선택
- **라우팅(Routes)** 탭 확인

**Public Route Table**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |
| `0.0.0.0/0` | Internet Gateway |

**Private Route Table (NAT 미생성)**

| Destination | Target |
|---|---|
| `10.0.0.0/16` | local |

> [!NOTE]
> `local`은 VPC 내부 네트워크 통신 경로입니다.
>
> `0.0.0.0/0 → Internet Gateway`는 인터넷으로 향하는 IPv4 트래픽의 경로입니다.
>
> 이번 실습에서는 NAT Gateway를 생성하지 않으므로 Private Subnet에 인터넷으로 향하는 기본 경로가 없습니다.

### 6단계: 서브넷 연결 확인

- Public Route Table 선택
- **서브넷 연결(Subnet associations)** 탭 확인
- Public Subnet 2개가 연결되어 있는지 확인

Private Subnet은 Internet Gateway로 향하는 기본 경로가 없는 라우팅 테이블에 연결되어 있는지 확인합니다.

**2강 완료 확인**

- [ ] `lab-vpc` 생성
- [ ] Public Subnet 2개 생성
- [ ] Private Subnet 2개 생성
- [ ] Internet Gateway 연결
- [ ] Public Route Table에 인터넷 경로 설정
- [ ] NAT Gateway 미생성

---

## 6주 3강 Hands-on Labs : EC2 웹 서버 배포 및 운영

### 실습 구성

```text
사용자 웹 브라우저
       │
       │ HTTP : 80
       ▼
    Internet
       │
 Internet Gateway
       │
  lab-vpc (10.0.0.0/16)
       │
 Public Subnet (10.0.1.0/24)
       │
 EC2 : web-server
 ├── Amazon Linux 2023
 ├── t3.micro
 ├── Apache HTTP Server
 ├── Public IPv4
 └── Security Group
     ├── SSH 22  : My IP
     └── HTTP 80 : Anywhere
```

### 1단계: EC2 Key Pair 생성

**① EC2 콘솔 이동**

- AWS 검색창에서 `EC2` 검색
- EC2 → **네트워크 및 보안 → 키 페어**
- **[키 페어 생성]** 선택

**② Key Pair 설정**

| 항목 | 설정값 |
|---|---|
| 이름 | `devops-key` |
| 키 페어 유형 | RSA |
| 프라이빗 키 파일 형식 | `.pem` |

- **[키 페어 생성]** 선택
- `devops-key.pem` 파일 다운로드

> [!WARNING]
> 개인키(`.pem`)는 최초 생성 시 다운로드합니다.
>
> 다른 사람과 공유하거나 GitHub에 업로드하지 않습니다. 안전한 폴더에 보관합니다.

### 2단계: Security Group 생성

**① 보안 그룹 생성**

- EC2 → **네트워크 및 보안 → 보안 그룹**
- **[보안 그룹 생성]** 선택

| 항목 | 설정값 |
|---|---|
| 보안 그룹 이름 | `webserver-sg` |
| 설명 | `Security group for web server` |
| VPC | `lab-vpc` |

**② 인바운드 규칙**

| 유형 | 프로토콜 | 포트 | 소스 |
|---|---|---|---|
| SSH | TCP | 22 | My IP |
| HTTP | TCP | 80 | Anywhere-IPv4 (`0.0.0.0/0`) |

- 아웃바운드 규칙은 실습을 위해 기본 허용 설정 유지
- **[보안 그룹 생성]** 선택

> [!NOTE]
> SSH(22)는 관리자 접속용이므로 My IP로 제한합니다.
>
> HTTP(80)는 외부 브라우저에서 웹 페이지를 확인하기 위해 허용합니다.
>
> 실제 운영 환경에서는 필요한 포트와 출발지 주소만 허용하는 것이 원칙입니다.

### 3단계: EC2 인스턴스 생성

**① 인스턴스 시작**

- EC2 → **인스턴스**
- **[인스턴스 시작]** 선택

**② EC2 설정**

| 항목 | 설정값 |
|---|---|
| 이름 | `web-server` |
| AMI | Amazon Linux 2023 |
| 아키텍처 | 64비트(x86) |
| 인스턴스 유형 | `t3.micro` |
| 키 페어 | `devops-key` |
| VPC | `lab-vpc` |
| 서브넷 | Public Subnet `10.0.1.0/24` |
| 퍼블릭 IP 자동 할당 | 활성화 |
| 보안 그룹 | 기존 `webserver-sg` 선택 |
| 스토리지 | gp3 SSD, 20 GiB |

- **[인스턴스 시작]** 선택

> [!NOTE]
> `t3.micro`는 계정의 무료 이용 조건에 따라 비용이 발생할 수 있습니다.
>
> Public IPv4 주소도 과금 대상일 수 있으므로 실습 종료 후 인스턴스를 삭제합니다.

### 4단계: EC2 상태 확인

- EC2 → 인스턴스 목록
- `web-server` 선택

확인 항목:

- 인스턴스 상태 : Running
- 상태 검사 : 2/2 통과
- VPC : `lab-vpc`
- Public IPv4 주소 할당
- 보안 그룹 : `webserver-sg`

**Public IPv4 주소 복사**

```text
예시: 3.xxx.xxx.xxx
```

> [!NOTE]
> Public IPv4 주소는 인스턴스를 중지했다가 다시 시작하면 변경될 수 있습니다.

### 5단계: MobaXterm으로 EC2 접속

**① MobaXterm 실행**

- MobaXterm 실행
- **Session → SSH** 선택

**② SSH 접속 설정**

| 항목 | 설정값 |
|---|---|
| Remote host | EC2 Public IPv4 |
| Specify username | `ec2-user` |
| Port | 22 |
| Advanced SSH settings | Use private key |
| Private key | `devops-key.pem` |

- **[OK]** 선택
- SSH 접속

**③ 접속 확인**

```bash
whoami
hostname
cat /etc/os-release
```

예상 확인 결과:

```text
ec2-user
Amazon Linux 2023
```

> [!NOTE]
> SSH 접속이 실패하면 다음 항목을 확인합니다.
>
> - EC2 상태가 Running인지
> - Public IPv4 주소가 맞는지
> - 보안 그룹 SSH(22) 소스가 현재 내 IP인지
> - 올바른 `devops-key.pem`을 선택했는지
> - 사용자 이름이 `ec2-user`인지

### 6단계: Apache 웹 서버 설치

**① Apache 설치**

```bash
sudo dnf install -y httpd
```

**② Apache 서비스 시작 및 부팅 시 자동 실행**

```bash
sudo systemctl enable --now httpd
```

**③ Apache 상태 확인**

```bash
systemctl status httpd
```

확인할 상태:

```text
Active: active (running)
```

> [!NOTE]
> `systemctl enable --now httpd`
>
> - `enable` : 부팅 시 자동 시작
> - `--now` : 현재 즉시 서비스 시작

### 7단계: 웹 페이지 작성

**① index.html 작성**

```bash
echo "Hello AWS EC2" | sudo tee /var/www/html/index.html
```

**② 파일 내용 확인**

```bash
cat /var/www/html/index.html
```

예상 출력:

```text
Hello AWS EC2
```

**③ EC2 내부에서 HTTP 접속 확인**

```bash
curl http://localhost
```

예상 출력:

```text
Hello AWS EC2
```

### 8단계: 웹 브라우저 접속

내 PC의 웹 브라우저에서 EC2 Public IPv4 주소 입력

```text
http://<EC2-Public-IP>
```

브라우저 화면:

```text
Hello AWS EC2
```

**확인 사항**

- EC2가 정상 실행 중인가?
- Apache가 정상 동작하는가?
- Security Group에서 HTTP(80)를 허용했는가?
- Public IPv4 주소로 외부 접속이 가능한가?

> [!NOTE]
> 이번 실습은 EC2에 직접 Apache를 설치하여 웹 서비스를 운영하는 가장 기본적인 형태입니다.
>
> 이후 Docker, Kubernetes, EKS를 학습하면서 동일한 웹 서비스가 컨테이너 기반으로 배포되는 구조를 이해하게 됩니다.

---

## 리소스 정리 (순서 중요)

> [!WARNING]
> 실습 종료 후 생성한 AWS 리소스를 삭제합니다.
>
> EC2를 중지(Stop)하는 것만으로는 EBS 등의 비용이 모두 없어지지 않습니다.

### ① EC2 인스턴스 삭제

- EC2 → **인스턴스**
- `web-server` 선택
- **인스턴스 상태 → 인스턴스 종료(Terminate)**
- 종료 확인

인스턴스 상태가 종료 상태로 변경될 때까지 기다립니다.

### ② NAT Gateway 확인

- VPC → **NAT 게이트웨이**
- `lab-vpc`에 연결된 NAT Gateway가 있는지 확인

이번 실습 지침대로 생성했다면 NAT Gateway는 없습니다.

만약 생성했다면:

- NAT Gateway 선택
- **작업 → NAT 게이트웨이 삭제**
- 삭제 완료 확인

### ③ Elastic IP 확인

- EC2 → **탄력적 IP**
- 이번 실습에서 할당한 Elastic IP가 있는지 확인
- 사용하지 않는 Elastic IP는 연결 해제 후 **탄력적 IP 주소 릴리스**

> [!NOTE]
> 이번 실습에서는 Elastic IP를 직접 생성하지 않습니다.
>
> NAT Gateway를 생성했다면 연결된 Elastic IP가 있을 수 있으므로 삭제 후 확인합니다.

### ④ VPC 삭제

- VPC → **내 VPC**
- `lab-vpc` 선택
- **작업 → VPC 삭제**
- 삭제 대상과 연결된 리소스 확인
- 삭제 실행

> [!NOTE]
> EC2, NAT Gateway 등 VPC에 종속된 리소스가 남아 있으면 삭제가 실패할 수 있습니다.
>
> VPC 삭제 화면에서 함께 삭제되는 리소스를 확인하고, 별도 삭제가 필요한 종속 리소스가 있다면 먼저 정리합니다.

### ⑤ Key Pair 보존

- EC2 → **키 페어**
- `devops-key` 확인

이번 실습에서는 삭제하지 않고 보존합니다.

Key Pair 자체에는 별도 사용 요금이 발생하지 않으며, 이후 EC2 실습에서 재사용할 수 있습니다.

### ⑥ 최종 확인

- [ ] EC2 `web-server` 종료 완료
- [ ] VPC `lab-vpc` 삭제 완료
- [ ] NAT Gateway 없음
- [ ] 불필요한 Elastic IP 없음
- [ ] 실습에서 생성한 보안 그룹 등 종속 리소스 정리
- [ ] Key Pair `devops-key` 보존

---


---

