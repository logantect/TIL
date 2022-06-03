### Attached & Detached 컨테이너 이해하기
```shell
# detached mode default
$ docker start [컨테이너ID}

# attached mode default
$ docker run [이미지ID]

# 기본 연결 모드로 실행 로그 확인 가능
$ docker run -p 3000:80 7f21a3339122

# 기본 비 연결 모드로 실행 로그 확인 안됨
$ docker start [컨테이너ID]

# 컨테이너에 기록된 로그 확인
$ docker logs musing_franklin

# 컨테이너에 기록된 로그 실시간 확인
$ docker logs -f musing_franklin

```