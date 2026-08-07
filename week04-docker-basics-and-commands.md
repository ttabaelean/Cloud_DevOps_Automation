
4주차 Hands-on Labs 실습 가이드
4주 1강 Hands-on Labs
1. Rocky Linux에 Docker Engine 설치
Bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker version
2. Docker 관리자 설정
Bash
sudo usermod -a -G docker admin
newgrp docker
id
4주 2강 Hands-on Labs: 컨테이너 기반 웹 서버 운영
실습 순서:

Docker Hub에서 Apache(httpd) 이미지를 다운로드합니다.

Apache(httpd) 컨테이너를 백그라운드로 실행하고 80번 포트를 연결합니다.

실행 중인 컨테이너를 확인합니다.

컨테이너 내부에 접속하여 기본 웹 페이지를 생성합니다.

웹 브라우저 또는 curl 명령으로 웹 페이지가 정상 출력되는지 확인합니다.

Apache 컨테이너 로그를 확인합니다.

웹서비스로 여러 번 접속하여 Access Log가 실시간으로 출력되는지 확인합니다.

실습이 끝나면 컨테이너를 중지하고 삭제합니다.

Bash
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
4주 3강 Hands-on Labs: 컨테이너 기반 웹 서버 운영 (Volume & Network)
실습 순서:

웹 페이지를 저장할 디렉터리를 생성합니다.

Volume을 연결하여 Apache(httpd) 컨테이너를 실행합니다.

웹 브라우저 또는 curl 명령으로 웹 페이지를 확인합니다.

Host에서 웹 페이지를 수정한 후 변경 사항이 즉시 반영되는지 확인합니다.

Docker Network를 생성하고 확인합니다.

기존 컨테이너를 삭제한 후 web-network에 연결하여 다시 실행합니다.

생성된 Network 정보를 확인합니다.

웹 서비스가 정상 동작하는지 확인합니다.

실습 정리: 모든 컨테이너와 네트워크를 삭제합니다.

Bash
mkdir -p ~/website
echo "<h1>Docker Volume Test</h1>" > ~/website/index.html

docker run -d --name webserver \
  -p 80:80 -v ~/website:/usr/local/apache2/htdocs httpd

curl http://localhost

echo "<h1>Volume Success</h1>" > ~/website/index.html
curl http://localhost

docker network create web-network
docker network ls

docker stop webserver
docker rm webserver
docker run -d --name webserver \
  --network web-network -p 80:80 \
  -v ~/website:/usr/local/apache2/htdocs httpd

docker network inspect web-network
curl http://localhost
docker ps

docker stop webserver
docker rm webserver
docker network rm web-network
[끝]
