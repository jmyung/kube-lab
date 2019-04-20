#  서비스 (Services)

본 랩에서는 `쿠버네티스 서비스`에 대해 알아보겠습니다.


## 1.

- ClusterIp
- NodePort
- LoadBalancer


## 2.

- 명령형
```sh
kubectl run nginx --image=nginx
```

- 선언형
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
...
```

## 3. 기억해야 할 것

- 컨테이너를 파드 내에서 동작. 어플리케이션을 대표하는 가장 간단한 쿠버네티스 오브젝트
- 파드는 보통 디플로이먼트에 의해 관리됨
- 서비스는 디플로이먼트를 노출(expose)시킴
- 서드파티 벤더(GCP, AWS 등)는 로드밸런싱 서비스타입을 제공

## 4. 인바인드 노드 포트 요구사항

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

## 5.

```sh
kubectl get pods -o wide
kubectl get deployments
kubectl get deployment webhead --type="NodePort" --port 80 # 외부에서 접근할 수 있도록 서비스를 노출
kubectl get services
curl localhost:32516 # 모든 노드의 32516 포트로 포트매핑
```

- kubeproxy : 클러스터의 모든 노드에 존재. 노드 포트로 들어온 트래픽을 클러스터 내의 적절한 파드로 리다이렉팅
- NodePort : 모든 노드의 3만번 포트로 포트매핑
- ClusterIp : internal LoadBalancer. 트래픽을 내부 pod ip에 전달한다


## 6. Ingress
