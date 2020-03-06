# 서버리스 구성 on k8s

[nuclio](https://github.com/nuclio/nuclio)를 사용하여, k8s 상에서 서버리스 환경을 구성합니다.

## 1. prerequisite
k8s는 1.15.3으로 클러스터 구성합니다.

- kubelet v.1.15.3 다운로드
```sh
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.15.3/bin/linux/amd64/kubelet
```

> 1.16 부터는 k9s api가 deplicated 되어, 펑션 컨테이너가 생성이 안되는 문제가 발생함. [관련이슈](https://github.com/nuclio/nuclio/issues/1364)

- kubeadm 설치
```sh
kubeadm init --pod-network-cidr=10.244.0.0/16 --kubernetes-version v1.15.3
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/2140ac876ef134e0ed5af15c65e414cf26827915/Documentation/kube-flannel.yml
kubectl taint nodes ${hostname} node-role.kubernetes.io/master:NoSchedule-
```


## 2. nuclio 설치

### 2-1. namaspace 생성
```sh
kubectl create ns nuclio
```

### 2-2. 시크릿 생성

- registry는 도커 허브 사용
```sh
kubectl create secret docker-registry registry-credentials --namespace nuclio \
    --docker-server=https://index.docker.io \
    --docker-username={본인계정} \
    --docker-password=$mypassword \
    --docker-email={본인이메일}
```

### 2-3. RBAC 생성
```sh
kubectl apply -f https://raw.githubusercontent.com/nuclio/nuclio/master/hack/k8s/resources/nuclio-rbac.yaml
```

### 2-4. nuclio 배포
```sh
kubectl apply -f https://raw.githubusercontent.com/nuclio/nuclio/master/hack/k8s/resources/nuclio.yaml
```

### 2-5. 포트포워딩
```sh
kubectl port-forward --address 0.0.0.0 -n nuclio $(kubectl get pods -n nuclio -l nuclio.io/app=dashboard -o jsonpath='{.items[0].metadata.name}') 8070:8070 &
```
> --address 0.0.0.0 를 붙여야 public ip 접속가능


## 3. NATS 큐 구성

![alt text](https://blobscdn.gitbook.com/v0/b/gitbook-28427.appspot.com/o/assets%2F-LqMYcZML1bsXrN3Ezg0%2F-LqMZac7AGFpQY7ewbGi%2F-LqMZeqHRi1BPHIRKAMs%2Fqueue.svg?generation=1570206036633351&alt=media)

### 3.1 NATS Server 띄우기

- 설치 및 기동
```sh
{
  curl -L https://github.com/nats-io/nats-server/releases/download/v2.0.0/nats-server-v2.0.0-linux-amd64.zip -o nats-server.zip
  unzip nats-server.zip -d nats-server
  cp nats-server /usr/local/bin
  nats-server &
}
```

- nats 서버 정보 기재
nats://{IP주소}:4222

### 3.2 NATS Publisher 띄우기

- [샘플](https://github.com/gregwhitaker/nats-queue-example) : CPU, 메모리 메트릭 정보 전송
```sh
sudo apt-get install openjdk-8-jdk
git clone https://github.com/gregwhitaker/nats-queue-example
./gradlew :queue-service:run
```

- 결과
```
...
...
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"fb595716-ea38-4f4b-9561-5913c8e7a5f5","cpuPercentage":0.21,"totalPhysicalMemory":1962.0,"freePhysicalMemory":627.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"48ac6cfe-c69e-4390-b1e6-cc3beafc1d5d","cpuPercentage":0.16,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"a30a16b3-247c-4e10-9858-8bc65f54a38d","cpuPercentage":0.22,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"3f101798-1fe4-4364-a50b-0936145b7a4b","cpuPercentage":0.24,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"b001a27b-fa30-47e1-bbe0-a1d5e5099864","cpuPercentage":0.23,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"f399d771-000d-45cd-a25b-72359b391c70","cpuPercentage":0.23,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"dfd8ab03-16dc-4424-90a4-d427dc0201ac","cpuPercentage":0.23,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"c20ba28e-52fa-4f20-a328-115c02131147","cpuPercentage":0.24,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
[Timer-0] INFO nats.example.queue.service.MetricsPublishTask - Publishing: {"id":"53f7652b-6b6d-49b0-9a38-9622e2c50216","cpuPercentage":0.23,"totalPhysicalMemory":1962.0,"freePhysicalMemory":629.0}
<=========----> 75% EXEC
```

- queue 이름은 소스 내에 이하로 기재됨
  - metrics-queue

## 4. Queue를 Subcribe 하는 Function 띄우기

### 4-1. nuctl 설치 : nuclio CLI 도구

```sh
wget https://github.com/nuclio/nuclio/releases/download/1.3.5/nuctl-1.3.5-linux-amd64 -O nuctl
chmod +x nuctl
mv nuctl /usr/local/bin/
```

### 4-2. **파드 목록 호출** 펑션 작성
- 파일명 : my_function.py

```py
from kubernetes import client, config
import os

def my_entry_point(context, event):
        context.logger.info_with('Got invoked',
                trigger_kind=event.trigger.kind,
                event_body=event.body,
                some_env=os.environ.get('MY_ENV_VALUE'))

        if event.trigger.kind == 'nats':
            config.load_kube_config()
            v1=client.CoreV1Api()
            print("Listing pods with their IPs:")
            ret = v1.list_pod_for_all_namespaces(watch=False)
            for i in ret.items:
                print("%s\t%s\t%s" % (i.status.pod_ip, i.metadata.namespace, i.metadata.name))
        else:
            return 'It wasnt invoked from nats'
```

### 4-3. 배포 yaml 정의

- secret
```
kubectl create secret generic kconfig --from-file=.kube/config -n nuclio
```

- 파일명 : function.yaml
```yaml
apiVersion: "nuclio.io/v1"
kind: NuclioFunction
metadata:
  name: my-function
  namespace: nuclio
spec:
  env:
  - name: MY_ENV_VALUE
    value: my value
  handler: my_function:my_entry_point
  runtime: python:3.6
  build:
    commands:
    - pip3.6 install kubernetes
  triggers:
    myNatsTopic:
      kind: "nats"
      url: "nats://{NATS서버 IP}:4222"
      attributes:
        "topic": "metrics-queue"
  volumes:
  - volume:
      name: "config"
      secret:
        secretName: "kconfig"
    volumeMount:
      name: "config"
      mountPath: "/root/.kube"
```

### 4-4. 펑션 배포
```sh
nuctl deploy --path ./ \
  --namespace nuclio \
  --env MY_ENV_VALUE='myung value' \
  --registry docker.io/{본인계정}
```


### 4-5. 결과 확인

- 펑션 배포 확인
```sh
$ nuctl get function -n nuclio
  NAMESPACE |    NAME     | PROJECT | STATE | NODE PORT | REPLICAS
  nuclio    | my-function | default | ready |     32092 | 1/1
```

- 파드 배포 확인
```sh
$ kubectl get po -n nuclio
NAME                                 READY   STATUS    RESTARTS   AGE
my-function-7557f4cf7d-9mstp         1/1     Running   0          35m
nuclio-controller-7c77d68cb9-22zth   1/1     Running   4          3d
nuclio-dashboard-54d5f44b4d-8t7xm    1/1     Running   4          3d
```

- Subcribe 로그 확인
 - 명령
```sh
kubectl logs my-function-7557f4cf7d-9mstp -n nuclio -f
```
 - 결과

```sh
...
...
Listing pods with their IPs:
10.244.0.126    default mypod2
10.244.0.129    default mypod3
10.244.0.127    default mypod5
10.244.0.128    kube-system     coredns-bf7759867-mksn8
10.244.0.124    kube-system     coredns-bf7759867-vv6hj
172.31.32.81    kube-system     etcd-sdspaas1c.mylabserver.com
172.31.32.81    kube-system     kube-apiserver-sdspaas1c.mylabserver.com
172.31.32.81    kube-system     kube-controller-manager-sdspaas1c.mylabserver.com
172.31.32.81    kube-system     kube-flannel-ds-amd64-8tdtc
172.31.32.81    kube-system     kube-proxy-zqz8b
172.31.32.81    kube-system     kube-scheduler-sdspaas1c.mylabserver.com
10.244.0.130    nuclio  my-function-7557f4cf7d-9mstp
10.244.0.131    nuclio  nuclio-controller-7c77d68cb9-22zth
10.244.0.132    nuclio  nuclio-dashboard-54d5f44b4d-8t7xm
20.01.21 08:12:41.677 sor.http.w0.python.logger (D) Processing event {"name": "my-function", "version": -1, "eventID": "d9d5ce74-b081-4703-a496-14a7dfb6ddd4"}
20.01.21 08:12:41.677 sor.http.w0.python.logger (D) Sending event to wrapper {"size": 0}
20.01.21 08:12:41.678            processor.http (W) 'NoneType' object has no attribute 'decode'
20.01.21 08:12:41.678            processor.http (I) Got invoked {"some_env": "myung value", "trigger_kind": "http", "event_body": null}
 ```




### 기타 명령

```sh
nuctl deploy nuclio-function-test -n nuclio \
    --path ./StartJobHandler.java \
    --onbuild-image quay.io/nuclio/handler-builder-java-onbuild:1.1.30-amd64 \
    --file ./function-build.yaml \
    --runtime java \
    --handler StartJobHandler \
    --no-pull \
    --verbose \
    --offline \
    --registry docker.io/human537 \
    --triggers '{"myNatsTopic": {"kind": "nats", "maxWorkers": 1, "url": "nats://172.17.0.3:4222", "attributes": {"topic": "startJob", "queueName": "a-batch"}}}'
```


```sh
nuctl invoke java-nats -n nuclio
```


---------------------------------

### NUCLIO FUNCTION 설정
maven {url 'http://13.126.46.165:8081/repository/maven-releases/'}
group: 'io.kubernetes', name: 'client-java', version: '4.0.0'
http://13.126.46.165:8081/repository/maven-releases/
