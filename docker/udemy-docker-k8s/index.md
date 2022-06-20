# Docker & Kubernetes: The Practical Guide [2022 Edition]
Udemy [Docker & Kubernetes: The Practical Guide [2022 Edition]](https://www.udemycom/course/docker-kubernetes-the-practical-guide/) 강의 메모입니다.

## 29. Attached & Detached 컨테이너 이해하기
```shell
# detached mode default
$ docker start [컨테이너ID]

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


## 32. 이미지 & 컨테이너 삭제하기

---

```bash
$ docker rm [컨테이너 ID]

$ docker images

# 이미지 제거
$ docker rmi [이미지 ID]

# 여러개 이미지 제거
$ docker rmi [이미지 ID] [이미지 ID] [이미지 ID]
```

이미지가 더 이상 컨테이너에서 사용되지 않고  중지된 컨테이너에 포함된 경우에만

이미지를 제거할 수 있다는 겁니다.

중지된 컨테이너가 있는 경우 그 컨테이너에서 사용 중인 이미지를 제거할 수 없습니다.

먼저 그 컨테이너를 제거해야 합니다.

즉, 컨테이너가 시작되거나 중지되더라도 그 컨테이너에 속한 이미지는 제거할 수 없습니다.

우선적으로 컨테이너를 제거해야 합니다.

## 33. 중지된 컨테이너 자동 제거하기

---

```bash
$ docker run -p 3000:80 -d -rm 2ddf2ede7d6c
```

## 34. 작동 배경 살펴보기: 이미지 검사

---

```bash
$ docker image inspect [이미지 ID]
```

## 35. 컨테이너에/컨테이너로 부터 파일 복사하기

## Reference
* [Docker & Kubernetes: The Practical Guide [2022 Edition]](https://www.udemycom/course/docker-kubernetes-the-practical-guide/)