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

```shell
vi pod.yaml
```

pod.yaml
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
```

Pod 띄우기
```shell
k apply -f pod.yaml
k get po
```



## 2-2. 특정 노드에서 띄워보기

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
