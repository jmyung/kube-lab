# 파드 띄우기 실습

본 랩에서는 배포의 최소단위인 파드를 띄워본다.

### 1. GKE를 이용한 쿠버네티스 클러스터 생성

##### 1-1. project zone 셋팅
```
gcloud config set compute/zone us-central1-a
```


##### 1-2. 클러스터 생성
```
gcloud container clusters create lab02 --num-nodes 3 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

결과

```
WARNING: Starting in 1.12, new clusters will have basic authentication disabled by default. Basic authentication can be enabled (or disabled) manually using the `--[no-]enable-basic-auth` flag.
WARNING: Starting in 1.12, new clusters will not have a client certificate issued. You can manually enable (or disable) the issuance of the client certificate using the `--[no-]issue-client-certificate` flag.
WARNING: Currently VPC-native is not the default mode during cluster creation. In the future, this will become the default mode and can be disabled using `--no-enable-ip-alias` flag. Use `--[no-]enable-ip-alias` flag to suppress this warning.
WARNING: Starting in 1.12, default node pools in new clusters will have their legacy Compute Engine instance metadata endpoints disabled by default. To create a cluster with legacy instance metadata endpoints disabled in the default node pool, run `clusters create` with the flag `--metadata disable-legacy-endpoints=true`.
This will enable the autorepair feature for nodes. Please see https://cloud.google.com/kubernetes-engine/docs/node-auto-repair for more information on node autorepairs.
WARNING: The behavior of --scopes will change in a future gcloud release: service-control and service-management scopes will no longer be added to what is specified in --scopes. To use these scopes, add them explicitly to --scopes. To use the new behavior, set container/new_scopes_behavior property (gcloud config set container/new_scopes_behavior true).
WARNING: Starting in Kubernetes v1.10, new clusters will no longer get compute-rw and storage-ro scopes added to what is specified in --scopes (though the latter will remain included in the default --scopes). To use these scopes, add them explicitly to --scopes. To use the new behavior, set container/new_scopes_behavior property (gcloud config set container/new_scopes_behavior true).

Creating cluster lab02 in us-central1-a...done.
Created [https://container.googleapis.com/v1/projects/qwiklabs-gcp-7b7d7e6f2dcce3e8/zones/us-central1-a/clusters/lab02].
To inspect the contents of your cluster, go to: https://console.cloud.google.com/kubernetes/workload_/gcloud/us-central1-a/lab02?project=qwiklabs-gcp-7b7d7e6f2dcce3e8
kubeconfig entry generated for lab02.
```

참고 : https://labs.play-with-k8s.com

### 2. 클러스터 기본정보 확인해 보기

##### 2-1. 버전확인
```
kubectl version
```

##### 2-2. 클러스터 정보 확인
```
kubectl cluster-info
```

##### 2-3. 노드 확인
```
kubectl get nodes
```

### 3. 컨테이너 실행 및 배포

##### 3-1. nginx 컨테이너 실행

```
kubectl run nginx --image=nginx:1.10.0
```

##### 3-2. 파드 정보 확인
```
kubectl get pods
```

##### 3-3. 외부에서 접근할 수 있도록 서비스 추가
```
kubectl expose deployment nginx --port 80 --type LoadBalancer
```

##### 3-4. 서비스 확인
```
kubectl get services
```

##### 3-5. 외부에서 붙기
```
curl http://<External IP>:80
```

### 4. html 수정해보기

##### 4-1. 컨테이너 내부 접속
```
kubectl get pods
k exec -it [파드이름] sh
```

##### 4-2. 소스수정
```
cd /usr/share/nginx/html
apt-install vim
```

### 다음시간은
- pod yaml 파일 직접 만들어서 배포해보기
- Docker build 를 이용 수정된 소스를 반영하여 재배포해보기 

### 숙제
- 미니큐브로 로컬에 클러스터 환경 설치하고, 오늘 한 것 복습해보기
- 미니큐브 설치가이드 : https://kubernetes.io/ko/docs/setup/minikube/
