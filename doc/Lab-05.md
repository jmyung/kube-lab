# 디플로이먼트

본 랩에서는 어플리케이션을 배포하는 디플로이먼트에 대해 알아보겠습니다.

- 롤링 업데이트 (Rolling update)
- 카나리 디플로이먼트 (Canary deployments)
- 블루-그린 디플로이먼트 (Blue-green deployments)



## 0. 클러스터 설치
```
gcloud container clusters create lab05 --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```


## 1. Deployment 맛보기


### 1-1. deployment 오브젝트

```sh
kubectl explain deployment
kubectl explain deployment --recursive
kubectl explain deployment.metadata.name
```

```sh
git clone https://github.com/googlecodelabs/orchestrate-with-kubernetes.git
```

#### deployments/auth.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: auth
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:2.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: "10Mi"
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

```sh
kubectl create -f deployments/auth.yaml
```

```sh
kubectl get deployments
kubectl get replicasets
kubectl get pods
```



#### services/auth.yaml

```yaml
kind: Service
apiVersion: v1
metadata:
  name: "auth"
spec:
  selector:
    app: "auth"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

```sh
kubectl create -f services/auth.yaml
```

#### hello 배포

```sh
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
```

```sh
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
kubectl get services frontend
curl -ks https://<EXTERNAL-IP>
```

### 1-2. 스케일링

#### 파드 개수를 5개로 늘리기

```sh
kubectl scale deployment hello --replicas=5
kubectl get pods | grep hello- | wc -l
```

#### 파드 개수를 3개로 줄이기

```sh
kubectl scale deployment hello --replicas=3
kubectl get pods | grep hello- | wc -l
```


## 2. 롤링 업데이트 (Rolling update)

![rolling](https://gcpstaging-qwiklab-website-prod.s3.amazonaws.com/bundles/assets/b9ce83f6343906592ff307ff71c11c4f7e4bc9f2831f3ee4169b08e281d499bd.png)

```sh
kubectl edit deployment hello
```
```sh
...
containers:
- name: hello
  image: kelseyhightower/hello:2.0.0
...
```
```sh
kubectl get replicaset
kubectl rollout history deployment/hello
kubectl rollout status deployment/hello
```

## 3. 카나리 디플로이먼트 (Canary deployments)

## 4. 블루-그린 디플로이먼트 (Blue-green deployments)
