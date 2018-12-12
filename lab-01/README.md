# 컨테이너 맛보기

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

##### 3-2. 웹서버 도커 이미지 빌드
```
sudo docker build -t py-web-server:v1 .
```

##### 3-3. 웹서버 도커 이미지 실행

```
sudo docker run -d -p 8888:8888 --name py-web-server -h my-web-server py-web-server:v1
```

##### 3-4. 테스트

```
curl http://localhost:8888
sudo docker rm -f py-web-server
```

### 4. 도커 이미지 푸시 및 실행

##### 4-1. 도커 이미지 이름 변경

```
sudo docker tag [before] [after]
```

##### 4-2. 도커 사이트 가입

- docker.com 가입
- 로그인
```
sudo docker login
로긴정보 입력
```

##### 4-3. 도커 이미지 푸시

```
sudo docker push [after]
```

##### 4-4. 도커 레지스트리 이미지로 실행

```
docker run -d -p 8888:8888 -h my-web-server 도커ACCOUNT/py-web-server:v1
```



## Docker composer 로 워드프레스 띄워보기
