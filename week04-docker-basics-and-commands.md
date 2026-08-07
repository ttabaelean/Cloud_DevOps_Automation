
# 4주차 Hands-on Labs 실습 가이드

## 4주 1강 Hands-on Labs

### 1. Rocky Linux에 Docker Engine 설치
```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker version
```
