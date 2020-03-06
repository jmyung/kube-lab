# 인그레스 (Ingress)


## 1. nginx 인그레스 컨트롤러 구성

> 참고사이트 : [nginx ingress](https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/)

### 매니페스트 다운로드
```sh
git clone https://github.com/nginxinc/kubernetes-ingress/
cd kubernetes-ingress/deployments
```

### 네임스페이스, 서비스 생성

```sh
kubectl apply -f common/ns-and-sa.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-ingress
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
```

### RBAC 생성
```sh
kubectl apply -f rbac/rbac.yaml
```

### 디폴트 시크릿 및 컨피그맵 생성
```sh
kubectl apply -f common/default-server-secret.yaml
kubectl apply -f common/nginx-config.yaml
```


### 인그레스 컨트롤러 생성
```sh
kubectl apply -f deployment/nginx-ingress.yaml
```
- 내용
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress
  template:
    metadata:
      labels:
        app: nginx-ingress
     #annotations:
       #prometheus.io/scrape: "true"
       #prometheus.io/port: "9113"
    spec:
      serviceAccountName: nginx-ingress
      containers:
      - image: nginx/nginx-ingress:edge
        imagePullPolicy: Always
        name: nginx-ingress
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
       #- name: prometheus
         #containerPort: 9113
        securityContext:
          allowPrivilegeEscalation: true
          runAsUser: 101 #nginx
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        args:
          - -nginx-configmaps=$(POD_NAMESPACE)/nginx-config
          - -default-server-tls-secret=$(POD_NAMESPACE)/default-server-secret
         #- -v=3 # Enables extensive logging. Useful for troubleshooting.
         #- -report-ingress-status
         #- -external-service=nginx-ingress
         #- -enable-leader-election
         #- -enable-prometheus-metrics
```
- 확인
```sh
kubectl get pods --namespace=nginx-ingress
```
- 결과
```
NAME                             READY     STATUS    RESTARTS   AGE
nginx-ingress-5c8f7c4f4f-c4b8b   1/1       Running   0          27s
```

### 서비스 노드포트로 띄우기

- 30000대 포트 지정 (수정 필요)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress
  namespace: nginx-ingress
spec:
  type: NodePort
  ports:
  - name: http
    nodePort: 30080
    port: 80
    protocol: TCP
    targetPort: 80
  - name: https
    nodePort: 30081
    port: 443
    protocol: TCP
    targetPort: 443
  selector:
    app: nginx-ingress
```
- 서비스 생성
```sh
kubectl create -f service/nodeport.yaml
```

## 2. 인그레스 구성

### 어플리케이션 배포 및 서비스 생성

- foo
```sh
kubectl run foo --image nginx
kubectl expose deploy foo --port=4200 --target-port=80
kubectl exec -it <foo pod_name> -- sed -i 's/nginx/foo/g' /usr/share/nginx/html/index.html
```

- bar
```sh
kubectl run bar --image nginx
kubectl expose deploy bar --port=8080 --target-port=80
kubectl exec -it <bar pod_name> -- sed -i 's/nginx/bar/g' /usr/share/nginx/html/index.html
```

### 인그레스 배포

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.test.io
    http:
      paths:
      - path: /
        backend:
          serviceName: foo
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: bar
          servicePort: 8080
```

### 테스트

```sh
curl http://localhost:30080/ -H 'Host: app.test.io'
curl app.test.io:30080 # /etc/hosts 에 먼저 추가 필요
```

## 3. SSL 적용

참고 : [인그레스 TLS 가이드](https://kubernetes.github.io/ingress-nginx/user-guide/tls/)

### SSL 인증서 생성

#### 환경변수
```
KEY_FILE="tls.key"
CERT_FILE="tls.crt"
HOST="*.test.io"
CERT_NAME="tls-test"
```

#### openssl 이용 ssl 인증서 생성
```sh
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ${KEY_FILE} -out ${CERT_FILE} -subj "/CN=${HOST}/O=${HOST}"
```

### 시크릿 생성
```sh
kubectl create secret tls ${CERT_NAME} --key ${KEY_FILE} --cert ${CERT_FILE}
secret/tls-test created
```
- 확인
```sh
kubectl describe secret tls-test
```

- 내용
```
Name:         tls-test
Namespace:    default
Labels:       <none>
Annotations:  <none>
Type:  kubernetes.io/tls
Data
====
tls.key:  1708 bytes
tls.crt:  1147 bytes
```

### 인그래스 재적용
- tls 속성 추가
```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.test.io
    http:
      paths:
      - path: /
        backend:
          serviceName: foo
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: bar
          servicePort: 8080
  tls:
  - hosts:
    - app.test.io
    secretName: tls-test
```

```sh
curl -k https://localhost:30081/ -H 'Host: app.test.io'
curl -k https://app.test.io:30081 # /etc/hosts 에 먼저 추가 필요
curl https://localhost:30081/ -H 'Host: app.test.io' --cacert tls.crt
```


## 참고) SSL 적용
먼저, SSL 적용을 위해 키를 생성합니다.

- apache2 설치

- 인증서 생성
```sh
openssl genrsa -out private.key 1024
openssl rsa -in private.key -pubout -out public.key
openssl req -new -key private.key -out private.csr
openssl req -x509 -key private.key -in private.csr -out private.crt -days 3650
```
> CN에 도메인 정보를 기입합니다.

- 결과

**파일** | **종류**
:---: | :---
private.key | 개인키
public.key | 공개키
private.crt | 인증서 (rootCA 인증)
rootCA.key | rootCA 개인키
rootCA.pem | rootCA 인증요청서 (CRT)

- 이동
```sh
mv * /etc/apache2/ssl/
```

- 설정파일 수정
  - 대상 파일
```sh
vi /etc/apache2/sites-available/default-ssl.conf
```
  - 수정내용
```sh
SSLCertificateFile /etc/apache2/ssl/public.key
SSLCertificateKeyFile /etc/apache2/ssl/private.pem
SSLCACertificateFile /etc/apache2/ssl/rootCA.pem
```
- 포트설정 수정
  - 대상 파일
```sh
vi /etc/apache2/ports.conf
```
  - 내용 : 80, 443으로 수정

- 테스트
```sh
curl --capath /etc/apache2/ssl/ https://test.io:443
```
