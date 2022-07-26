# AWS EC2 Pinpoint 설치

## EC2 구성
* Hbase, Collector, Web 하나의 EC2에 설치
* AMI - **Amazon Linux 2 AMI (HVM) - Kernel 5.10, SSD Volume Type 선택**
* Security Group InBound Open Port: 8000 - 9999

## Pinpoint 설치
java 8 설치(Hbase 버전 지원으로 인해 java 8설치

![images/java-support-by-release-line.png](images/java-support-by-release-line.png)

```bash
# Java 8 설치
sudo yum install -y java-1.8.0-openjdk-devel.x86_64

readlink -f /usr/bin/java
# /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.amzn2.0.2.x86_64/jre/bin/java
# java 경로까지만 처리

sudo vim /etc/profile
# export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.amzn2.0.2.x86_64
# 해당 파일 제일 하단에 입력 후 재접속

# 확인
echo $JAVA_HOME
# /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.amzn2.0.2.x86_64

```

Hbase 설치

```bash
# HBase 다운로드
wget https://archive.apache.org/dist/hbase/1.4.14/hbase-1.4.14-bin.tar.gz

# 압축해제
tar xvf hbase-1.4.14-bin.tar.gz

# create symbolic link
ln -s hbase-1.4.14 hbase

# Start hbase
hbase/bin/start-hbase.sh

# Pinpoint 에서 제공하는 Hbase 스크립트 다운로드
wget https://raw.githubusercontent.com/pinpoint-apm/pinpoint/master/hbase/scripts/hbase-create.hbase

# 스크립트 실행
hbase/bin/hbase shell hbase-create.hbase
```

## Pinpoint Collector 설치

```bash
# Download Collector 
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.3.3/pinpoint-collector-boot-2.3.3.jar

# 실행권한 부여
chmod +x pinpoint-collector-boot-2.3.3.jar

# 실행
nohup java -jar -Dpinpoint.zookeeper.address=localhost pinpoint-collector-boot-2.3.3.jar >/dev/null 2>&1 &
```

## Pinpoint Web 설치

```bash
# Download Web
wget https://github.com/pinpoint-apm/pinpoint/releases/download/v2.3.3/pinpoint-web-boot-2.3.3.jar

# 실행권한 부여
chmod +x pinpoint-web-boot-2.3.3.jar

# 실행
nohup java -jar -Dpinpoint.zookeeper.address=localhost pinpoint-web-boot-2.3.3.jar >/dev/null 2>&1 &
```

## Reference
* [https://jojoldu.tistory.com/573](https://jojoldu.tistory.com/573)
* [https://hbase.apache.org/book.html](https://hbase.apache.org/book.html)
* [https://pinpoint-apm.gitbook.io/pinpoint/](https://pinpoint-apm.gitbook.io/pinpoint/)