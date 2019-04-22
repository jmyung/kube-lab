#  서비스 (Services)

본 랩에서는 `쿠버네티스 서비스`에 대해 알아보겠습니다.

## 1. 인바인드 노드 포트

- Master Node:
  - TCP 6443 : Kubernetes API Server
  - TCP 2379~2380 : etcd server client API
  - TCP 10250 : Kubelet API
  - TCP 10251 : kube-scheduler
  - TCP 10252 : kube-controller-manager
  - TCP 10255 : Read-only Kubelet API
- Worker Nodes:
  - TCP 10250 : Kubelet API
  - TCP 10255 : Read-only Kubelet API
  - TCP 30000~32767 : NodePort Services

## 2. 서비스 타입

### 2-1. 서비스 란
  - 컨테이너를 파드 내에서 동작. 어플리케이션을 대표하는 가장 간단한 쿠버네티스 오브젝트
  - 파드는 보통 디플로이먼트에 의해 관리됨
  - 서비스는 디플로이먼트를 노출(expose)시킴
  - 서드파티 벤더(GCP, AWS 등)는 로드밸런싱 서비스타입을 제공

### 2-2. 서비스 타입

- kubeproxy : 클러스터의 모든 노드에 존재. 노드 포트로 들어온 트래픽을 클러스터 내의 적절한 파드로 리다이렉팅합니다.
- ClusterIP : 클러스터 내부 IP에 서비스를 노출합니다. 이 값을 선택하면 클러스터 내에서만 서비스에 접근 할 수 있습니다. 이것은 기본 ServiceType입니다. internal LoadBalancer. 트래픽을 내부 파드 ip에 전달합니다.
- NodePort : 정적 포트 (NodePort)에서 각 노드의 IP에 서비스를 노출합니다. NodePort 서비스가 라우트할 ClusterIP 서비스가 자동으로 생성됩니다. <NodeIP>:<NodePort>를 요청하여 클러스터 외부에서 NodePort 서비스에 접속할 수 있습니다.
- LoadBalancer : 클라우드 벤더의 로드밸런서를 사용하여 외부로 서비스를 노출합니다. 외부 로드밸런서가 라우트 할 NodePort 및 ClusterIP 서비스가 자동으로 생성됩니다. 어플리케이션의 일부 (예 : 프론트 엔드)의 경우 외부 (클러스터 외부) IP 주소로 서비스를 노출 할 수 있습니다.


## 3. (먼저) 디플로이먼트

- GKE 클러스터 올리기
```sh
gcloud config set compute/zone us-central1-a
gcloud container clusters create lab10 --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

- 명령형
```sh
kubectl run nginx --image=nginx:1.7.9 --replicas=3
```

- 선언형
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

- nginx에 요청
```sh
k run -it alpine --image=alpine -- sh
# apk add curl
# ???
```

## 4. ClusterIp

[![01](https://d33wubrfki0l68.cloudfront.net/27b2978647a8d7bdc2a96b213f0c0d3242ef9ce0/e8c9b/images/docs/services-iptables-overview.svg?raw=true)]()


```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: nginx-pod
  ports:
  - protocol: TCP
    port: 8080
    targetPort: 80
```

또는

```sh
kubectl expose deployment nginx-deployment --port=8080 --target-port=80
```

```
selector:
  app: nginx-pod
```
`selector`가 없는 경우에는 endpoint를 직접 만들어줘야합니다.
참고 : https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors

```sh
sed -i 's/Thank/Sorry/g' /usr/share/nginx/html/index.html
```

## 5. NodePort

```sh
$ kubectl edit svc my-service # type: NodePort 로 변경
service/my-service edited
```

```sh
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP   10.27.240.1     <none>        443/TCP          1h
my-service   NodePort    10.27.242.221   <none>        8080:30755/TCP   13m
```

GKE 에서 NodePort 테스트 해보기

## 6. LoadBalancer

```sh
$ kubectl edit svc my-service # type: LoadBalancer 로 변경
service/my-service edited
```

```sh
$ kubectl get svc
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.27.240.1     <none>        443/TCP          1h
my-service   LoadBalancer   10.27.242.221   <pending>     8080:30755/TCP   22m
```

## 7. CKA 문제

#### 7-1. Create a service that uses an external load balancer and points to a 3 pod cluster running nginx.

#### 7-2. Create a service that manually requires endpoint creation - and create that too

#### 7-3. Write a service that exposes nginx on a nodeport

- a. Change it to use a cluster port
- b. Scale the service
- c. Change it to use an external IP
- d. Change it to use a load balancer
