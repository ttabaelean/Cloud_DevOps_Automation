# 4주차 Hands-on Labs 실습 가이드

## 4주 1강 Hands-on Labs

### 1. Rocky Linux에 Docker Engine 설치
```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker version

```

### 2. Docker 관리자 설정

```bash
sudo usermod -a -G docker admin
newgrp docker
id

```

---

## 4주 2강 Hands-on Labs: 컨테이너 기반 웹 서버 운영

### 1. 웹 서버 실행 및 접속 테스트

```bash
docker pull httpd
docker run -d --name webserver -p 80:80 httpd
docker ps

docker exec -it webserver bash
echo "<h1>Web Site</h1>" > /usr/local/apache2/htdocs/index.html
exit

curl http://localhost
docker logs -f webserver
curl http://localhost

docker stop webserver
docker rm webserver

```

---

## 4주 3강 Hands-on Labs: 컨테이너 기반 웹 서버 운영 (Volume & Network)

### 1. 볼륨 연결 및 네트워크 구성

```bash
mkdir -p ~/website
echo "<h1>Docker Volume Test</h1>" > ~/website/index.html

docker run -d --name webserver -p 80:80 -v ~/website:/usr/local/apache2/htdocs httpd

curl http://localhost

echo "<h1>Volume Success</h1>" > ~/website/index.html
curl http://localhost

docker network create web-network
docker network ls

docker stop webserver
docker rm webserver
docker run -d --name webserver --network web-network -p 80:80 -v ~/website:/usr/local/apache2/htdocs httpd

docker network inspect web-network
curl http://localhost
docker ps

docker stop webserver
docker rm webserver
docker network rm web-network

```

```

```
