# AWS EC2 Pinpoint 설치 with docker

## EC2 구성
* AMI - **Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type 선택**

## 도커 및 Git 설치

```bash
# docker 설치
sudo amazon-linux-extras install docker

# docker 실행
sudo service docker start

# docker 사용자 ec2-user 그룹에 추가
sudo usermod -a -G docker ec2-user

# Git 설치
sudo yum install git -y

# 재부팅
sudo reboot
```

## 도커 컴포즈 설치

```bash
# docker-compose 설치
sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

# 권한 수정
sudo chmod +x /usr/local/bin/docker-compose

# Verify
docker-compose version
```

## Pinpoint 설치

```bash
git clone https://github.com/naver/pinpoint-docker.git
cd pinpoint-docker
docker-compose pull && docker-compose up -d
```

## 위 명령어 실행시 아래 와 같이 오류 발생하여 권한처리 후 정상동작 확인

```bash
# docker-compose pull && docker-compose up -d 실행 시 오류 발생!!
Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info": dial unix /var/run/docker.sock: connect: permission denied

# docker.sock 권한 변경
sudo chmod 666 /var/run/docker.sock

# 다시 실행
docker-compose pull && docker-compose up -d
```

## Pinpoint Web 접속

- http://[Pinpoint-Web-host]:8079
- Sample App(quickapp) 앱 확인

![pinpoint-demo](images/demo.png)

## Reference
* [https://gist.github.com/npearce/6f3c7826c7499587f00957fee62f8ee9](https://gist.github.com/npearce/6f3c7826c7499587f00957fee62f8ee9)
* [https://pinpoint-apm.gitbook.io/pinpoint/](https://pinpoint-apm.gitbook.io/pinpoint/)
* [https://newbedev.com/shell-error-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socket-at-unix-var-run-docker-sock-get-http-2fvar-2frun-2fdocker-sock-v1-24-info-dial-unix-var-run-docker-sock-connect-permission-denied-code-example](https://newbedev.com/shell-error-got-permission-denied-while-trying-to-connect-to-the-docker-daemon-socket-at-unix-var-run-docker-sock-get-http-2fvar-2frun-2fdocker-sock-v1-24-info-dial-unix-var-run-docker-sock-connect-permission-denied-code-example)