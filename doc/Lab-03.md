# 파드 띄우기 실습

본 랩에서는 파드를 정의하는 YAML을 직접 작성해봅니다.

## 1. 에디터 설치

- 본랩에서는 vi를 사용합니다.

alias
```shell
alias k=kubectl
```

## 2. YAML 작성

## 2-1. nginx 컨테이터 이미지를 Pod으로 띄워보기

참고 : https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#step-two-add-a-nodeselector-field-to-your-pod-configuration

##### 야믈 파일 작성

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
k apply -f pod.yaml
k get po
k get po --show-labels
```

##### 상태 조회
```sh
k describe po 파드이름
```

##### 로그 조회

실행중인 인스턴스에서 로그를 다운로드합니다.

```sh
k logs 파드이름
```


##### 파드 삭제
```sh
k delete po 파드이름
```

파드가 삭제될 때는 30초의 유예기간을 갖습니다. 더이상 요청을 받지 않습니다.
또한, 해당 파드의 컨테이너 데이터도 삭제됩니다.

## 2-2. 특정 노드에서 띄워보기

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
k label node 이름 disktype=ssd
```

## 2-3. 네임스페이스 지정해보기

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
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```
