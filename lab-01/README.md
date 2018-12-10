# 컨테이너 다루기

본 랩에서는 도커파일을 생성해보고, Docker hub 레지스트리에 푸시한 후 컨테이너를 실행해본다.

- 도커 이미지 빌드
- 도커 레지스트리에 푸시
- 도커 컨테이너 실행

### 1. VM 띄우기

- GCP 또는 AWS 상에서 VM을 하나 띄운다.
- 도커 설치
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

### 2. 수작업으로 웹서버 띄워보기

##### 2-1. 프로젝트 가져오기
```
git clone https://github.com/jmyung/py-docker-demo
cd py-docker-demo
```

##### 2-2. 의존성 라이브러리 설치
```
sudo apt-get install -y python3 python3-pip
pip3 install tornado
```

##### 2-3. 백그라운드로 파이썬 앱 실행
```
python3 web-server.py &
```

##### 2-4. 테스트
```
python3 web-server.py &
```

##### 2-5. 웹서버 종료
```
curl http://localhost:8888
```


### 3. 도커 이미지 빌드 및 실행

##### 3-1. 도커파일 살펴보기

```
cat Dockerfile
```
