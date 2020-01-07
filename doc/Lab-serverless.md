# 서버리스 구성 on k8s

[nuclio](https://github.com/nuclio/nuclio)를 사용하여, k8s 상에서 서버리스 환경을 구성합니다.

## 1. prerequisite
k8s는 1.15.3으로 클러스터 구성

- kubelet v.1.15.3
```sh
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet
```

> 1.16 부터는 k9s api가 deplicated 되어, 펑션 컨테이너가 생성이 안되는 문제가 발생함. [관련이슈](https://github.com/nuclio/nuclio/issues/1364)

## 2. nuclio 설치

### namaspace 생성
```sh
kubectl create ns nuclio
```

### 시크릿 생성

registry는 도커 허브 사용
```sh
kubectl create secret docker-registry registry-credentials --namespace nuclio \
    --docker-server=https://index.docker.io \
    --docker-username={본인계정} \
    --docker-password=$mypassword \
    --docker-email={본인이메일}
```

### RBAC 생성
```sh
kubectl apply -f https://raw.githubusercontent.com/nuclio/nuclio/master/hack/k8s/resources/nuclio-rbac.yaml
```

### nuclio 배포
```sh
kubectl apply -f https://raw.githubusercontent.com/nuclio/nuclio/master/hack/k8s/resources/nuclio.yaml
```

### 포트포워딩
```sh
kubectl port-forward --address 0.0.0.0 -n nuclio $(kubectl get pods -n nuclio -l nuclio.io/app=dashboard -o jsonpath='{.items[0].metadata.name}') 8070:8070
```
> --address 0.0.0.0 를 붙여야 public ip 접속가능


## 2. NATS 큐 구성

![alt text](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LqMYcZML1bsXrN3Ezg0%2F-LqMZac7AGFpQY7ewbGi%2F-LqMZeqHRi1BPHIRKAMs%2Fqueue.svg?generation=1570206036633351&alt=media)

### 2.1 작성중
