# 파드 띄우기 실습

본 랩에서는 파드를 정의하는 매니패스트를 직접 작성해봅니다.

## 1. 클러스터 올리기

### 1-1. project zone 셋팅
```
gcloud config set compute/zone us-central1-a
```

### 1-2. 클러스터 생성
```
gcloud container clusters create lab03 --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

### 1-3. alias 설정
```shell
alias k=kubectl
```

## 2. 파드 매니페스트 생성 / 실행

- 본랩에서는 vi를 사용합니다.

### 2-1. nginx 컨테이터 이미지를 Pod으로 띄워보기

참고 : https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#step-two-add-a-nodeselector-field-to-your-pod-configuration

##### 매니페스트 파일 (YAML) 작성

쿠버네티스 오브젝트의 스펙을 기술해 봅니다.

```shell
vi pod1.yaml
```

pod2-1.yaml
```yaml
apiVersion: v1 # 이 오브젝트를 생성하기 위해 사용하고 있는 쿠버네티스 API 버전이 어떤 것인지
kind: Pod # 어떤 종류의 오브젝트를 생성하고자 하는지
metadata:  # 이름, 네임스페이스 를 포함하여 오브젝트를 유일하게 구분지어 줄 데이터
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

##### Pod 띄우기
```shell
kubectl apply -f pod.yaml
kubectl get po
kubectl get po --show-labels
```

##### 세부사항 조회

- 파드 기본정보
- 실행중인 컨테이너 정보
- 이벤트 정보

```sh
kubectl describe po 파드이름
```

##### 파드 삭제
```sh
kubectl delete po 파드이름
```

파드가 삭제될 때는 30초의 유예기간을 갖습니다. 더이상 요청을 받지 않습니다.
또한, 해당 파드의 컨테이너 데이터도 삭제됩니다.

### 2-2. 특정 노드에서 띄워보기

pod2-2.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```

파드 생성이 안된다...

```sh
kubectl label node 이름 disktype=ssd
```

### 2-3. 네임스페이스 지정해보기

- **네임스페이스** : 클러스터 내부의 객체를 관리.
  - 각 네임스페이스는 객체 집합을 담고 있는 폴더로 생각할 수 있습니다.
  - 명시를 안하면, default namespace


pod2-3.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: kuba
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
```

## 3. 포트 포워딩

참고 : https://github.com/kubernetes-up-and-running/kuard

### 3-1. kuard-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kuard
spec:
  containers:
  - name: kuard
    image: gcr.io/kuar-demo/kuard-amd64:1
    ports:
    - containerPort: 8080
      name: http
      protocol: TCP
```

### 3-2. 파드 실행

```shell
kubectl apply -f kuard-pod.yaml
```

### 3-3. 포트 포워딩

```shell
kubectl port-forward kuard 8080:8080
```

http://localhost:8080 확인


### 3-4. 로그 확인

##### 로그 조회

실행중인 인스턴스에서 로그를 다운로드합니다.

```sh
kubectl logs kuard
```

## 4. 하나의 파드, 두개의 컨테이너

https://kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/#creating-a-pod-that-runs-two-containers

two-container-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
spec:

  restartPolicy: Never

  volumes:
  - name: shared-data
    emptyDir: {}

  containers:

  - name: nginx-container
    image: nginx
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html

  - name: debian-container
    image: debian
    volumeMounts:
    - name: shared-data
      mountPath: /pod-data
    command: ["/bin/sh"]
    args: ["-c", "echo Hello from the debian container > /pod-data/index.html"]
```

컨테이너 쉘 접속

```sh
kubectl exec -it two-containers -c nginx-container -- /bin/bash

root@two-containers:/ apt-get update
root@two-containers:/ apt-get install curl procps
root@two-containers:/ ps aux

root@two-containers:/ curl localhost
```

포트포워딩 후 로컬에서 접속

```
how?
```

파드 상태를 확인해봅시다.

```sh
kubectl get po

NAME             READY     STATUS      RESTARTS   AGE
two-containers   1/2       Completed   0          20m
```

왜 하나만 살아있을까요?

나머지 하나도 계속 running 상태가 되도록 한번 변경해봅시다.

## 5. 클러스터 삭제

`gcloud container clusters delete lab03
`
## 6. 미니큐브 띄워보기

https://kubernetes.io/ko/docs/tutorials/hello-minikube/
