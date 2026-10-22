# 7주차 Hands-on Labs: 클라우드 확장 서비스 및 애플리케이션 아키텍처

## 실습 환경

| 항목 | 구성 |
|---|---|
| 실습 도구 | AWS Management Console, MobaXterm(또는 EC2 Instance Connect) |
| AWS 리전 | 서울 `ap-northeast-2` |
| 네트워크 | 6주차 `lab-vpc` (`10.0.0.0/16`), Public 2개·Private 2개, AZ 2개 |
| ALB | `webserver-alb` (Internet-facing, HTTP 80) |
| Target Group | `webserver-tg` (HTTP 80, Health Check `/`) |
| EC2 | `web01`, `web02` (Amazon Linux 2023, `t3.micro`) |
| ASG | `webserver-asg` (Desired 2, Min 1, Max 5, CPU 목표 50%) |
| Launch Template | `webserver-lt` (Amazon Linux 2023, `t3.micro`) |
| RDS | MySQL, `kcu-rds`, `db.t3.micro`, 20 GiB, 초기 DB `kcu` |

> [!WARNING]
> ALB, EC2, RDS, Public IPv4 주소 등은 과금될 수 있습니다. 특히 **RDS와 ALB는 무료 이용 조건을 확인**하고 실습을 진행하세요. 실습 후 마지막 **리소스 정리**를 반드시 수행합니다.
>
> 6주차에 `lab-vpc`를 삭제했다면 먼저 6주차 노트를 참고해 같은 네트워크를 다시 생성합니다. 이번 실습에서는 NAT Gateway를 만들지 않습니다.

---

## 7주 1강 Hands-on Labs: 로드밸런서(ALB) 구성

### 실습 아키텍처

```text
                         사용자 브라우저
                               |
                           HTTP : 80
                               |
                      Internet-facing ALB
                         webserver-alb
                     Public Subnet A / C
                               |
                      Listener HTTP : 80
                               |
                     Target Group webserver-tg
                          /          \
                  EC2 web01        EC2 web02
                 10.0.1.0/24      10.0.2.0/24
                  Web Server 01    Web Server 02
```

### 1단계: 실습 네트워크 확인

- AWS 콘솔에서 **서울 리전** 선택
- **VPC → 내 VPC**에서 `lab-vpc` (`10.0.0.0/16`) 확인
- **VPC → 서브넷**에서 다음 Public Subnet 확인
  - AZ-a: `10.0.1.0/24`
  - AZ-c: `10.0.2.0/24`
- 두 Public Subnet의 라우팅 테이블에 `0.0.0.0/0 → Internet Gateway` 경로가 있는지 확인

> [!NOTE]
> 이 노트는 6주차 `lab-vpc`를 사용합니다. **Default VPC를 선택하지 않습니다.**

### 2단계: 보안 그룹 생성

**① ALB 보안 그룹**

- **EC2 → 보안 그룹 → 보안 그룹 생성**
- 이름: `webserver-alb-sg`
- VPC: `lab-vpc`
- 인바운드: **HTTP / TCP 80 / Anywhere-IPv4 (`0.0.0.0/0`)**
- 아웃바운드: 기본 허용 유지

**② 웹 서버 보안 그룹**

- 이름: `webserver-sg`
- VPC: `lab-vpc`
- 인바운드 규칙

| 유형 | 포트 | 소스 | 목적 |
|---|---|---|---|
| HTTP | 80 | `webserver-alb-sg` | ALB에서 웹 서버로 전달 |
| SSH | 22 | My IP | 관리자 접속 (선택) |

> [!NOTE]
> 웹 서버의 HTTP 포트는 전체 인터넷에 열지 않고 **ALB 보안 그룹에서 오는 요청만 허용**합니다. SSH 접속을 사용한다면 자신의 공인 IP만 허용합니다.

### 3단계: EC2 웹 서버 2대 생성

**EC2 → 인스턴스 → 인스턴스 시작**에서 아래 설정으로 각각 생성합니다.

| 항목 | web01 | web02 |
|---|---|---|
| 이름 | `web01` | `web02` |
| AMI | Amazon Linux 2023 | Amazon Linux 2023 |
| 인스턴스 유형 | `t3.micro` | `t3.micro` |
| VPC | `lab-vpc` | `lab-vpc` |
| 서브넷 | Public `10.0.1.0/24` | Public `10.0.2.0/24` |
| Public IPv4 | 활성화 | 활성화 |
| 보안 그룹 | `webserver-sg` | `webserver-sg` |
| 키 페어 | 6주차 `devops-key` 또는 기존 키 | 동일 |

**사용자 데이터(User data)**에 각각 다음 스크립트를 입력하면 최초 부팅 시 Apache와 페이지가 자동 설치됩니다.

**web01**

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo '<h1>Web Server 01</h1>' > /var/www/html/index.html
```

**web02**

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo '<h1>Web Server 02</h1>' > /var/www/html/index.html
```

> [!NOTE]
> PPT의 수동 설치 명령과 동일한 동작을 User data로 자동화했습니다. 수동 설치를 실습하려면 SSH로 각각 접속하여 아래 명령을 실행해도 됩니다.
>
> ```bash
> sudo dnf install -y httpd
> sudo systemctl enable --now httpd
> echo '<h1>Web Server 01</h1>' | sudo tee /var/www/html/index.html
> ```
>
> web02에서는 `Web Server 02`로 바꿉니다.

### 4단계: Target Group 생성

- **EC2 → 대상 그룹(Target Groups) → 대상 그룹 생성**
- 대상 유형: **인스턴스(Instances)**
- 이름: `webserver-tg`
- 프로토콜: **HTTP**, 포트: **80**
- VPC: `lab-vpc`
- 상태 검사(Health check): HTTP, 경로 `/`
- **다음** → `web01`, `web02` 선택 → **아래에 보류 중인 것으로 포함(Include as pending below)** → **대상 그룹 생성**

**확인:** 대상 그룹 → **대상(Targets)** 탭에서 두 인스턴스가 `healthy`가 될 때까지 기다립니다. 아직 ALB를 만들지 않았다면 Health Check 상태가 지연될 수 있습니다.

### 5단계: Application Load Balancer 생성

- **EC2 → 로드 밸런서 → 로드 밸런서 생성**
- **Application Load Balancer** 선택
- 이름: `webserver-alb`
- 체계(Scheme): **인터넷 경계(Internet-facing)**
- IP 주소 유형: IPv4
- 네트워크 매핑
  - VPC: `lab-vpc`
  - AZ-a Public Subnet `10.0.1.0/24`
  - AZ-c Public Subnet `10.0.2.0/24`
- 보안 그룹: `webserver-alb-sg`
- 리스너: **HTTP : 80**
- 기본 작업: **대상 그룹 `webserver-tg`로 전달**
- **로드 밸런서 생성**

### 6단계: ALB 동작 확인

1. **EC2 → 대상 그룹 → `webserver-tg` → 대상**에서 두 인스턴스가 `healthy`인지 확인합니다.
2. **EC2 → 로드 밸런서 → `webserver-alb`**에서 **DNS 이름**을 복사합니다.
3. 브라우저에서 `http://<ALB-DNS-이름>` 접속 후 여러 번 새로고침합니다.
4. `Web Server 01`, `Web Server 02` 페이지가 각각 표시되는지 확인합니다.

> [!NOTE]
> ALB는 요청을 여러 정상 대상에 분산하지만, 새로고침할 때 반드시 01·02가 번갈아 나타나는 것은 아닙니다. 브라우저 연결 재사용 등의 영향이 있습니다. 필요하면 시크릿 창이나 여러 요청으로 확인합니다.

**1강 완료 확인**

- [ ] EC2 2대 Running
- [ ] ALB HTTP 80 리스너 생성
- [ ] 대상 그룹에 두 인스턴스 Healthy
- [ ] ALB DNS로 웹 페이지 접속 성공

---

## 7주 2강 Hands-on Labs: Auto Scaling Group 구성

### 실습 목표

- 시작 템플릿(Launch Template)으로 EC2 생성 설정 정의
- ASG가 최소·원하는·최대 인스턴스 수를 관리하는지 확인
- CPU 대상 추적 정책으로 Scale Out 관찰

> [!IMPORTANT]
> **1강에서 수동 생성한 `web01`, `web02`는 ASG가 관리하는 인스턴스가 아닙니다.** 2강에서는 같은 ALB를 사용하되, ASG가 새로 만드는 인스턴스를 별도로 확인합니다.
>
> 실습을 명확히 하려면 `webserver-tg`에서 기존 `web01`, `web02`를 **등록 해제(Deregister)** 한 뒤 ASG를 연결하세요. 두 인스턴스는 RDS 접속 실습을 위해 유지할 수 있습니다.

### 1단계: 시작 템플릿 생성

- **EC2 → 시작 템플릿(Launch Templates) → 시작 템플릿 생성**

| 항목 | 설정값 |
|---|---|
| 시작 템플릿 이름 | `webserver-lt` |
| 템플릿 버전 설명 | `V1` |
| AMI | Amazon Linux 2023 (x86_64) |
| 인스턴스 유형 | `t3.micro` |
| 키 페어 | 포함하지 않음 |
| 보안 그룹 | `webserver-sg` |
| 서브넷 | 시작 템플릿에서 지정하지 않음 |

**고급 세부 정보 → 사용자 데이터(User data)**에 아래 내용을 입력합니다.

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo '<h1>Auto Scaling Web Server</h1>' > /var/www/html/index.html
```

- **시작 템플릿 생성** 선택

> [!NOTE]
> 사용자 데이터는 새 EC2가 처음 시작할 때 Apache를 설치하고 기본 웹 페이지를 생성합니다. 키 페어를 생략하므로 ASG 인스턴스에 직접 SSH로 접속하지 않는 구성을 기본으로 합니다.

### 2단계: Auto Scaling Group 생성

- **EC2 → Auto Scaling 그룹 → Auto Scaling 그룹 생성**

| 항목 | 설정값 |
|---|---|
| ASG 이름 | `webserver-asg` |
| 시작 템플릿 | `webserver-lt` (기본 버전) |
| VPC | `lab-vpc` |
| 가용 영역 및 서브넷 | Public `10.0.1.0/24`, `10.0.2.0/24` |
| 로드 밸런싱 | 기존 로드 밸런서 대상 그룹에 연결 |
| 대상 그룹 | `webserver-tg` |
| ELB 상태 검사 | 사용 권장 (상태 검사 유예 기간 설정) |
| 원하는 용량(Desired) | 2 |
| 최소 용량(Min) | 1 |
| 최대 용량(Max) | 5 |
| 크기 조정 정책 | 대상 추적(Target tracking) |
| 지표 | 평균 CPU 사용률 |
| 목표값 | 50% |

- 나머지 옵션 확인 후 **Auto Scaling 그룹 생성**
- **ASG → 활동(Activity)**에서 인스턴스 생성 기록 확인
- **EC2 → 인스턴스**에서 ASG가 생성한 인스턴스 2대 확인
- **대상 그룹 `webserver-tg` → 대상**에서 ASG 인스턴스가 `healthy`인지 확인

> [!NOTE]
> 이 노트는 6주차의 `lab-vpc`를 사용하므로 PPT에 표시된 **Default VPC** 대신 `lab-vpc`를 선택합니다.
>
> Public Subnet에 생성한 ASG 인스턴스가 패키지를 설치하려면 Public IPv4 주소와 인터넷 경로가 필요합니다. 서브넷의 **퍼블릭 IPv4 자동 할당** 설정을 확인하세요. 이 설정이 꺼져 있으면 시작 템플릿 또는 네트워크 설정에서 적절히 활성화해야 합니다.

### 3단계: ASG 동작 확인

**① 자동 복구 확인**

1. ASG의 **인스턴스 관리(Instance management)**에서 관리 중인 인스턴스 1대 확인
2. 해당 인스턴스를 EC2 콘솔에서 종료(Terminate)
3. ASG **활동(Activity)**에서 인스턴스 종료 및 새 인스턴스 시작 기록 확인
4. 잠시 후 원하는 용량 `2`로 복구되는지 확인

> [!WARNING]
> **ASG 관리 인스턴스만** 종료합니다. 1강에서 만든 수동 인스턴스 `web01`, `web02`는 ASG가 자동으로 다시 만들지 않습니다.

**② CPU 부하로 Scale Out 테스트 (선택 실습)**

PPT의 CPU 부하 명령:

```bash
sudo dnf install -y stress-ng
sudo stress-ng --cpu 2 --timeout 300s
```

> [!IMPORTANT]
> 위 명령은 **부하를 발생시킬 EC2 내부에서 실행**해야 합니다. 그러나 이번 시작 템플릿은 키 페어를 포함하지 않으므로 기본 구성만으로는 SSH 접속할 수 없습니다.
>
> 실습하려면 별도로 SSM Session Manager 접근 권한·연결을 구성하거나, 시작 템플릿에 접속 가능한 키 페어를 설정하고 새 인스턴스를 시작해야 합니다. `stress-ng` 설치 가능 여부도 AMI 저장소에 따라 확인이 필요합니다.
>
> CPU 목표값 50%는 **50%를 넘는 순간 즉시 5대로 늘어난다는 의미가 아닙니다.** CloudWatch 지표와 조정 정책에 따라 점진적으로 확장되며, 300초 부하만으로 확장이 반드시 관찰되는 것은 아닙니다.

**③ 관찰 항목**

- **Auto Scaling 그룹 → 모니터링**: 평균 CPU 사용률, 인스턴스 수
- **Auto Scaling 그룹 → 활동**: Scale Out 활동 이력
- **EC2 → 인스턴스**: 새 인스턴스 생성 여부
- **대상 그룹**: 새 인스턴스가 Healthy로 등록되는지 확인

**2강 완료 확인**

- [ ] Launch Template 생성
- [ ] ASG Desired 2, Min 1, Max 5 설정
- [ ] 대상 그룹에 ASG 인스턴스 등록
- [ ] 관리 인스턴스 종료 시 ASG가 새 인스턴스 생성
- [ ] (선택) CPU 부하와 Scale Out 활동 확인

---

## 7주 3강 Hands-on Labs: Amazon RDS(MySQL) 구축

### 실습 아키텍처

```text
EC2 web01 (Public Subnet)
        |
        | MySQL TCP 3306
        v
RDS MySQL kcu-rds
  ├── Private Subnet A : 10.0.3.0/24
  └── Private Subnet C : 10.0.4.0/24
      (DB Subnet Group)
```

> [!NOTE]
> RDS는 Private Subnet에 배치하고 외부 공개 접근을 사용하지 않습니다. 같은 VPC의 EC2에서 MySQL 클라이언트로 접속합니다. **DB Subnet Group에 AZ 2개를 포함해도 RDS를 Multi-AZ로 생성했다는 의미는 아닙니다.** 이번 실습은 단일 DB 인스턴스입니다.

### 1단계: DB Subnet Group 생성

- **RDS → 서브넷 그룹(Subnet groups) → DB 서브넷 그룹 생성**

| 항목 | 설정값 |
|---|---|
| 이름 | `webserver-subgrp` |
| 설명 | `Private subnets for KCU RDS` |
| VPC | `lab-vpc` |
| 가용 영역 | `ap-northeast-2a`, `ap-northeast-2c` |
| 서브넷 | Private `10.0.3.0/24`, `10.0.4.0/24` |

- **생성** 선택
- 서로 다른 AZ의 Private Subnet 2개가 포함되었는지 확인

### 2단계: RDS 보안 그룹 생성

- **EC2 → 보안 그룹 → 보안 그룹 생성**
- 이름: `kcu-rds-sg`
- VPC: `lab-vpc`
- 인바운드 규칙:

| 유형 | 포트 | 소스 |
|---|---|---|
| MySQL/Aurora | TCP 3306 | `webserver-sg` |

> [!WARNING]
> RDS 3306 포트를 `0.0.0.0/0`으로 공개하지 않습니다. 웹 서버가 사용하는 `webserver-sg`를 소스로 지정하여 해당 보안 그룹을 사용하는 EC2에서만 접근하도록 설정합니다.

### 3단계: RDS MySQL 인스턴스 생성

- **RDS → 데이터베이스 → 데이터베이스 생성**
- **표준 생성(Standard create)** 선택

| 항목 | 설정값 |
|---|---|
| 엔진 | MySQL |
| 템플릿 | 샌드박스(표시되는 경우) 또는 실습에 맞는 개발/테스트 옵션 |
| DB 인스턴스 식별자 | `kcu-rds` |
| 마스터 사용자 이름 | `admin` |
| 비밀번호 | 본인이 정한 강력한 비밀번호 |
| 인스턴스 클래스 | `db.t3.micro` (제공되는 경우) |
| 스토리지 | 범용 SSD, 20 GiB |
| VPC | `lab-vpc` |
| DB Subnet Group | `webserver-subgrp` |
| 퍼블릭 액세스 | **아니요(No)** |
| VPC 보안 그룹 | 기존 `kcu-rds-sg` |
| 초기 데이터베이스 이름 | `kcu` |
| 다중 AZ 배포 | 아니요 (단일 인스턴스) |
| 자동 백업 | 교육용 실습에서 비활성화 가능하면 비활성화 |

- 설정 확인 후 **데이터베이스 생성**
- 상태가 **사용 가능(Available)** 이 될 때까지 대기

> [!WARNING]
> PPT에는 예시 비밀번호가 있으나 공개 GitHub 노트에는 공통 비밀번호를 사용하지 않습니다. 각자 강력한 비밀번호를 설정하고 안전하게 보관하세요. 생성 화면에서 자동 백업 해제나 특정 인스턴스 유형이 허용되지 않으면 콘솔의 실제 지원 옵션을 확인합니다. 비용이 발생할 수 있습니다.

### 4단계: RDS Endpoint 확인

- **RDS → 데이터베이스 → `kcu-rds` → 연결 및 보안**
- **엔드포인트(Endpoint)** 복사
- 포트 `3306` 확인
- 연결된 보안 그룹이 `kcu-rds-sg`인지 확인

```text
예시 Endpoint: kcu-rds.xxxxxxxxxxxx.ap-northeast-2.rds.amazonaws.com
Port: 3306
```

### 5단계: EC2에서 MySQL 접속

- 1강에서 만든 `web01`에 SSH 접속 (MobaXterm 또는 EC2 Instance Connect)
- MySQL 호환 클라이언트 설치

```bash
sudo dnf install -y mariadb105
```

> [!NOTE]
> PPT에는 `sudo dnf update -y`도 포함되어 있습니다. 전체 패키지 업데이트는 시간이 걸릴 수 있어 이 노트에서는 필수 명령으로 넣지 않았습니다. `mariadb105` 패키지 설치가 실패하면 AMI에서 제공하는 MariaDB 클라이언트 패키지 이름을 확인하세요.

**RDS 접속**

```bash
mysql -h <RDS-엔드포인트> -u admin -p
```

- 비밀번호 입력 프롬프트가 나오면 생성 시 설정한 비밀번호 입력
- MySQL에 접속한 뒤 데이터베이스 확인

```sql
SHOW DATABASES;
USE kcu;
SELECT DATABASE();
EXIT;
```

예상 확인 결과:

```text
Database
information_schema
kcu
mysql
performance_schema
sys
```

> [!NOTE]
> 데이터베이스 목록은 엔진 버전과 권한에 따라 다를 수 있습니다. `kcu` 데이터베이스가 보이고 `USE kcu;`가 성공하면 실습 목표를 달성한 것입니다.

**접속 실패 시 확인**

- RDS 상태가 `Available`인지
- Endpoint 주소와 포트가 맞는지
- RDS와 EC2가 같은 `lab-vpc`인지
- `kcu-rds-sg`가 **3306 / 소스 `webserver-sg`**를 허용하는지
- EC2의 보안 그룹 아웃바운드가 DB 통신을 허용하는지
- 비밀번호가 올바른지

---

## 리소스 정리 (순서 중요)

> [!WARNING]
> **RDS → ASG → 수동 EC2 → ALB → Target Group → 시작 템플릿 → 보안 그룹 → VPC** 순서로 정리합니다. 실제 콘솔에서는 일부 삭제 작업이 비동기로 진행되므로 **삭제 완료를 확인한 후** 다음 단계로 이동합니다.

### ① RDS 삭제

- **RDS → 데이터베이스 → `kcu-rds` → 작업 → 삭제**
- 교육용 데이터가 필요하지 않다면 최종 스냅샷을 만들지 않는 옵션 선택
- 삭제 방지 기능이 활성화되어 있으면 먼저 비활성화
- 삭제 완료 확인

> [!NOTE]
> 최종 스냅샷을 보존하면 스냅샷 스토리지 비용이 발생할 수 있습니다. 실제 업무 데이터에는 이 삭제 절차를 적용하지 않습니다.

### ② DB Subnet Group 삭제

- **RDS → 서브넷 그룹 → `webserver-subgrp` → 삭제**
- DB 삭제가 완료된 후 진행

### ③ Auto Scaling Group 삭제

- **EC2 → Auto Scaling 그룹 → `webserver-asg`**
- 그룹 삭제를 선택하고 삭제 완료 확인
- ASG가 생성한 인스턴스가 종료되었는지 확인

### ④ 수동 생성 EC2 삭제

- **EC2 → 인스턴스**에서 `web01`, `web02` 선택
- **인스턴스 상태 → 인스턴스 종료(Terminate)**
- 종료 완료 확인

### ⑤ ALB · Target Group 삭제

- **EC2 → 로드 밸런서 → `webserver-alb` → 삭제**
- ALB 삭제 완료 확인
- **EC2 → 대상 그룹 → `webserver-tg` → 삭제**

### ⑥ 시작 템플릿 삭제

- **EC2 → 시작 템플릿 → `webserver-lt` → 삭제**

### ⑦ 보안 그룹 삭제

- **EC2 → 보안 그룹**에서 `kcu-rds-sg`, `webserver-sg`, `webserver-alb-sg` 삭제
- 보안 그룹 간 참조나 리소스 연결이 남아 있으면 먼저 제거

### ⑧ VPC 삭제 (실습 종료 시)

- **VPC → 내 VPC → `lab-vpc` → 삭제**
- 6주차 네트워크를 다음 실습에서 계속 사용할 계획이면 보존 가능하나, 불필요한 리소스와 비용은 확인
- `devops-key` 키 페어는 이후 실습을 위해 보존 가능

### ⑨ 최종 확인

- [ ] RDS `kcu-rds` 없음
- [ ] DB Subnet Group `webserver-subgrp` 없음
- [ ] ASG `webserver-asg` 없음
- [ ] EC2 `web01`, `web02` 및 ASG 인스턴스 없음
- [ ] ALB `webserver-alb` 없음
- [ ] Target Group `webserver-tg` 없음
- [ ] Launch Template `webserver-lt` 없음
- [ ] 실습용 보안 그룹 정리
- [ ] 불필요한 VPC·Public IPv4·Elastic IP·스냅샷 없음

---

## 실습 완료 확인

| 강의 | 확인 항목 | 정상 결과 |
|---|---|---|
| 1강 | Target Group | `web01`, `web02` Healthy |
| 1강 | ALB | DNS로 웹 페이지 접속 성공 |
| 2강 | ASG | Desired 2, Min 1, Max 5 |
| 2강 | 자동 복구 | 관리 인스턴스 종료 후 대체 인스턴스 생성 |
| 2강 | CPU 기반 확장 | 정책 설정 및 활동 기록 확인 (부하 테스트 선택) |
| 3강 | RDS | `kcu-rds` Available |
| 3강 | DB 접속 | EC2에서 MySQL 접속 및 `kcu` DB 확인 |
| 종료 | 리소스 정리 | 과금 리소스 삭제 완료 |
