# 디플로이먼트 두번째

본 랩에서는 어플리케이션을 배포하는 디플로이먼트에 대해 알아보겠습니다.

- 카나리 디플로이먼트 (Canary deployments)
- 블루-그린 디플로이먼트 (Blue-green deployments)


## 0. 클러스터 설치

- 클러스터 설치
```
gcloud container clusters create lab05 --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

- 프로젝트 설정
```gcloud config set project PROJECT_ID```



## 1. Deployment (Lab-05)

- 일괄 수행 (Lab-05)
```
kubectl create -f deployments/auth.yaml
kubectl create -f services/auth.yaml
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
```

- IP 확인
```sh
kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"
curl -ks https://IP
```


## 2. 카나리 디플로이먼트 (Canary deployments)

![rolling](https://gcpstaging-qwiklab-website-prod.s3.amazonaws.com/bundles/assets/a92ae020fe45c962846f03a4dcf30f0002c9b50a09a04a6024c5706ae65a668c.png)

- 카나리 앱 배포 (이미지 버전 2.0.0)
```sh
kubectl create -f deployments/hello-canary.yaml
```


- hello 앱 확인하기
```sh
$ kubectl get pods | grep hello
NAME                            READY     STATUS    RESTARTS   AGE
hello-84f68fb667-4dprx          1/1       Running   0          13m
hello-84f68fb667-bttxg          1/1       Running   0          13m
hello-84f68fb667-sdjxd          1/1       Running   0          13m
hello-canary-66f697bdf6-f7bc7   1/1       Running   0          7s
```
기배포된 hello 3개는 1.0.0, 새롭게 배포된 hello-canary 1개는 2.0.0


- OUTPUT
  - 연속해서 이하 커맨드 수행
```sh
curl -ks https://IP/version
```
  - 결과 확인 : 25%의 비율로 카나리 App(2.0.0)에 요청하는 것을 확인할 수 있습니다.
```sh
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"1.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
```

- 이상이 없으면 1.0.0 디플로이먼트를 삭제하고 2.0.0의 레플리카를 늘려줍니다.

- 만약 클라이언트에서 한가지 버전만 보고 싶다면 (헷갈리지 않도록) `sessionAffinity: ClientIP` 속성 정의
```sh
kubectl edit svc hello
```
```sh
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: 2019-02-18T14:03:06Z
  name: hello
  namespace: default
  resourceVersion: "7956"
  selfLink: /api/v1/namespaces/default/services/hello
  uid: ea644406-3385-11e9-a09b-42010a800063
spec:
  clusterIP: 10.51.240.113
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: hello
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
  type: ClusterIP
status:
  loadBalancer: {}
```
```sh
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
$ curl -ks https://35.193.9.190/version
{"version":"2.0.0"}
```
어떤 사람은 1.0.0 나올 수도 있습니다.





## 3. 블루-그린 디플로이먼트 (Blue-green deployments)

![blue-green](https://gcpstaging-qwiklab-website-prod.s3.amazonaws.com/bundles/assets/3ce5bc436e3b64a358fb21c86abb42b2812ef0c02287b938bb5b70bac0b1ea9c.png)

### 3-1. 블루 서비스 올리기 (이미지 버전 1.0.0)

- 기존 1.0.0 에 붙음

```sh
kubectl apply -f services/hello-blue.yaml
```

```sh
selector:
  app: "hello"
  version: 1.0.0
```

- 테스트

```sh
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

### 3-2. 그린 디플로이먼트 올리기 (이미지 버전 2.0.0)

- 새로운 버전 (2.0.0)의 그린 디플로이먼트를 올립니다.

`hello-green.yaml`

```sh
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: hello-green
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: hello
        track: stable
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: 10Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1
```

- 배포
```sh
kubectl create -f deployments/hello-green.yaml
```
```sh
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```
하지만, 여전히 1.0.0


- 그린 서비스 올리기
```sh
kubectl apply -f services/hello-green.yaml
```
```
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```


### 3-3. 롤백 (이미지 버전 1.0.0 으로 다시)

```sh
kubectl apply -f services/hello-blue.yaml
```
- 결과 확인
```sh
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```
