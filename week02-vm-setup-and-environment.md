# 2주차 Hands-on Labs: 가상 컴퓨터 환경 이해 및 가상머신 구축

이 문서는 **2주차 강의 자료**에 수록된 가상머신(VM) 생성, OS 설치, 멀티 노드 환경 구축 및 스냅샷 관리 실습 절차를 정리한 가이드입니다.

---

## LAB 1: VMware Workstation 설치 및 가상머신(server1) 생성

### 1. VMware Workstation Pro 설치
- 설치 마법사를 실행하여 초기 설정을 완료합니다.

### 2. 가상머신(`server1`) 생성 (New Virtual Machine)
- **CPU 코어 수**: 2
- **Memory (RAM)**: 4 GiB
- **Virtual Disk (HDD)**: 20 GB
- **ISO 이미지 매핑**: Rocky Linux ISO 파일 연동

### 3. 리눅스 운영체제(OS) 설치 및 초기 설정
- **언어 설정**: 한국어
- **계정 생성**:
  - `root` 비밀번호 설정 (`password`)
  - 사용자 계정 생성 (`admin` / `admin123!`)
- **네트워크 설정**:
  - Hostname: `server1`
  - IP Address: `192.168.100.10/24`
  - Default Gateway: `192.168.100.2`
  - DNS: `8.8.8.8`

---

## LAB 2: VM 복제 및 멀티 노드 환경 구축

### 1. 가상머신 복제 (Clone)
- `server1` 가상머신을 복제하여 `server2`를 생성합니다.

### 2. `server2` 서버 환경 구성
- `server2` 전원을 켜고 로그인합니다 (`root` / `password`).
- **IP 주소 변경**: `192.168.100.20`
- **Hostname 변경**: `server2`
- **네트워크 설정 및 인터넷 연결 확인**:
  ```bash
  ip addr
  ping -c 3 8.8.8.8
