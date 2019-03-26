# 젠킨스 CI/CD

본 랩에서는 젠킨스 CI/CD 에 대해 알아보겠습니다.

## 1. GCP API 확인

- Kubernetes Engine API : Enable
- Cloud Build API : Enable


## 2. 쿠버네티스 클러스터 생성

- zone 설정
```sh
gcloud config set compute/zone us-central1-f
```

- 클러스터 생성
```sh
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/projecthosting,cloud-platform"
```

- 생성 확인
```sh
$ gcloud container clusters list
NAME        LOCATION       MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
jenkins-cd  us-central1-f  1.11.7-gke.12   111.222.333.444  n1-standard-2  1.11.7-gke.12  2          RUNNING
```

- kubeconfig 가져오기
```sh
gcloud container clusters get-credentials jenkins-cd
```


## 3. 레파지토리 클론

- 깃 클론
```sh
git clone https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git
cd continuous-deployment-on-kubernetes
```


## 4. Helm 패키지 매니저를 이용한 Jenkins 설치

- helm 설치
```sh
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
tar zxfv helm-v2.9.1-linux-amd64.tar.gz
cp linux-amd64/helm .
```

- 내 계정(who)을 `cluster-admin`(what)역할로 Role Binding

이렇게 해야 클러스터 내 젠킨스 퍼미션을 줄 수 있습니다.
```sh
$ kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)

clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-binding created
```

- 결과 확인

```sh
$ kubectl get ClusterRoleBinding cluster-admin-binding -oyaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: 2019-03-24T13:38:32Z
  name: cluster-admin-binding
  resourceVersion: "4850"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/cluster-admin-binding
  uid: xxx
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: jesang.myung@gmail.com
```

- tiller 서비스 계정(who) 생성 후 `cluster-admin`(what)역할로 Role Binding
```sh
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```
위를 수행하지 않으면 tiller 서버가 helm install 명령어로 jenkins를 설치할 수 없습니다.

- 결과 확인

```sh
$ k get  serviceaccount/tiller -oyaml -n kube-system
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2019-03-23T10:54:19Z
  name: tiller
  namespace: kube-system
  resourceVersion: "5357"
  selfLink: /api/v1/namespaces/kube-system/serviceaccounts/tiller
  uid: 024fe35a-4d5a-11e9-98ab-42010a800078
secrets:
- name: tiller-token-r7qsn
```

- `tiller` 서비스 계정으로 helm 초기화
```sh
./helm init --service-account=tiller
./helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

- `tiller` 서버 디플로이먼트 확인
```sh
$ k get deploy -n kube-system
NAME                     DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
tiller-deploy            1         1         1            1           2m
```

```sh
$ ./helm version
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

```sh
$ ./helm search jenkins
NAME            CHART VERSION   APP VERSION     DESCRIPTION
stable/jenkins  0.35.2          lts             Open source continuous integration server. It s...
```

```sh
$ cat jenkins/values.yaml
Master:
  InstallPlugins:
    - kubernetes:1.12.6
    - workflow-job:2.31
    - workflow-aggregator:2.5
    - credentials-binding:1.16
    - git:3.9.3
    - google-oauth-plugin:0.7
    - google-source-plugin:0.3
  Cpu: "1"
  Memory: "3500Mi"
  JavaOpts: "-Xms3500m -Xmx3500m"
  ServiceType: ClusterIP
Agent:
  Enabled: true
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "512Mi"
Persistence:
  Size: 100Gi
NetworkPolicy:
  ApiVersion: networking.k8s.io/v1
rbac:
  install: true
  serviceAccountName: cd-jenkins
```

```sh
$ ./helm install -n cd stable/jenkins -f jenkins/values.yaml --version 0.16.6 --wait
NAME:   cd
LAST DEPLOYED: Sat Mar 23 20:15:31 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME              TYPE       CLUSTER-IP    EXTERNAL-IP  PORT(S)    AGE
cd-jenkins-agent  ClusterIP  11.11.11.11  <none>       50000/TCP  5s
cd-jenkins        ClusterIP  11.11.11.11  <none>       8080/TCP   5s

==> v1beta1/Deployment
NAME        DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
cd-jenkins  1        1        1           0          5s

==> v1/Pod(related)
NAME                         READY  STATUS    RESTARTS  AGE
cd-jenkins-5dc9cd6487-bs8c9  0/1    Init:0/1  0         5s

==> v1/Secret
NAME        TYPE    DATA  AGE
cd-jenkins  Opaque  2     5s

==> v1/ConfigMap
NAME              DATA  AGE
cd-jenkins        4     5s
cd-jenkins-tests  1     5s

==> v1/PersistentVolumeClaim
NAME        STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
cd-jenkins  Bound   pvc-f866d118-4d5c-11e9-98ab-42010a800078  100Gi     RWO           standard      5s

==> v1/ServiceAccount
NAME        SECRETS  AGE
cd-jenkins  1        5s

==> v1beta1/ClusterRoleBinding
NAME                     AGE
cd-jenkins-role-binding  5s


NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace default cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "component=cd-jenkins-master" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080
  kubectl port-forward $POD_NAME 8080:8080

3. Login with the password from step 1 and the username: admin

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
Configure the Kubernetes plugin in Jenkins to use the following Service Account name cd-jenkins using the following steps:
  Create a Jenkins credential of type Kubernetes service account with service account name cd-jenkins
  Under configure Jenkins -- Update the credentials config in the cloud section to use the service account credential you created in the step above.
```

- helm 으로 설치된 Jenkins 리소스 확인
```sh
kubectl get svc --show-labels
kubectl get deployments --show-labels
kubectl get pods --show-labels
```

## 5. Jenkins 접속

### 포트 포워딩으로 로컬 외부 포트 허용
```sh
export POD_NAME=$(kubectl get pods -l "component=cd-jenkins-master" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```


### 접속 계정
- username : admin
- password
```sh
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```


## 6. 어플리케이션 배포

[![01](https://gcpstaging-qwiklab-website-prod.s3.amazonaws.com/bundles/assets/3f54f9241596a6b03888b7ff079f8ee9ab3ba2d2c4db960175ee7b8336704b3e.png)]()

```sh
cd sample-app
kubectl create ns production
kubectl apply -f k8s/production -n production
kubectl apply -f k8s/canary -n production
kubectl apply -f k8s/services -n production
kubectl scale deployment gceme-frontend-production -n production --replicas 4
```

```sh
$ k get po -n production --show-labels
NAME                                         READY     STATUS    RESTARTS   AGE       LABELS
gceme-backend-canary-59dcdd58c5-jk94k        1/1       Running   0          6m        app=gceme,env=canary,pod-template-hash=1587881471,role=backend
gceme-backend-production-7fd77cf554-4bbh7    1/1       Running   0          6m        app=gceme,env=production,pod-template-hash=3983379110,role=backend
gceme-frontend-canary-c9846cd9c-dhfq5        1/1       Running   0          6m        app=gceme,env=canary,pod-template-hash=754027857,role=frontend
gceme-frontend-production-6976ccd9cd-6pq4h   1/1       Running   0          6m        app=gceme,env=production,pod-template-hash=2532778578,role=frontend
gceme-frontend-production-6976ccd9cd-fmn59   1/1       Running   0          6m        app=gceme,env=production,pod-template-hash=2532778578,role=frontend
gceme-frontend-production-6976ccd9cd-tz45q   1/1       Running   0          6m        app=gceme,env=production,pod-template-hash=2532778578,role=frontend
gceme-frontend-production-6976ccd9cd-xds8s   1/1       Running   0          6m        app=gceme,env=production,pod-template-hash=2532778578,role=frontend
```

- 버전 확인
```sh
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
curl http://$FRONTEND_SERVICE_IP/version
1.0.0
```

## 7. 젠킨스 파이프라인 만들기

### 7-1. Github 내 `s1` repository 생성

### 7-2. origin master 에 push
```sh
$ git init
Initialized empty Git repository in /home/jesang_myung/dev/continuous-deployment-on-kubernetes/sample-app/.git/
$ git config --global user.email "jesang.myung@gmail.com"
$ git config --global user.name "jmyung"
$ git remote add origin https://github.com/jmyung/s1.git
$ git remote -v
origin  https://github.com/jmyung/s2.git (fetch)
origin  https://github.com/jmyung/s2.git (push)
$ git add .
$ git commit -m "initial"
[master (root-commit) bd919d6] initial
 31 files changed, 2434 insertions(+)
 create mode 100644 Dockerfile
 create mode 100644 Gopkg.lock
 create mode 100644 Gopkg.toml
 create mode 100644 Jenkinsfile
 create mode 100644 html.go
 create mode 100644 k8s/canary/backend-canary.yaml
 create mode 100644 k8s/canary/frontend-canary.yaml
 create mode 100644 k8s/dev/backend-dev.yaml
 create mode 100644 k8s/dev/default.yml
 create mode 100644 k8s/dev/frontend-dev.yaml
 create mode 100644 k8s/production/backend-production.yaml
 create mode 100644 k8s/production/frontend-production.yaml
 create mode 100644 k8s/services/backend.yaml
 create mode 100644 k8s/services/frontend.yaml
 create mode 100644 main.go
 create mode 100644 main_test.go
 create mode 100644 vendor/cloud.google.com/go/AUTHORS
 create mode 100644 vendor/cloud.google.com/go/CONTRIBUTORS
 create mode 100644 vendor/cloud.google.com/go/LICENSE
 create mode 100644 vendor/cloud.google.com/go/compute/metadata/metadata.go
 create mode 100644 vendor/golang.org/x/net/AUTHORS
 create mode 100644 vendor/golang.org/x/net/CONTRIBUTORS
 create mode 100644 vendor/golang.org/x/net/LICENSE
 create mode 100644 vendor/golang.org/x/net/PATENTS
 create mode 100644 vendor/golang.org/x/net/context/context.go
 create mode 100644 vendor/golang.org/x/net/context/ctxhttp/ctxhttp.go
 create mode 100644 vendor/golang.org/x/net/context/ctxhttp/ctxhttp_pre17.go
 create mode 100644 vendor/golang.org/x/net/context/go17.go
 create mode 100644 vendor/golang.org/x/net/context/go19.go
 create mode 100644 vendor/golang.org/x/net/context/pre_go17.go
 create mode 100644 vendor/golang.org/x/net/context/pre_go19.go
$ git push origin master
Username for 'https://github.com': jmyung
Password for 'https://jmyung@github.com':
Counting objects: 48, done.
Compressing objects: 100% (43/43), done.
Writing objects: 100% (48/48), 27.03 KiB | 0 bytes/s, done.
Total 48 (delta 11), reused 0 (delta 0)
remote: Resolving deltas: 100% (11/11), done.
To https://github.com/jmyung/s2.git
 * [new branch]      master -> master
```



### 7-3. 젠킨스 잡 생성

#### step 1. Jenkins > New Item

#### step 2. 멀티 브랜치 선택
- item name : sample-app
- Multibranch Pipeline 선택 > OK

#### step 3. Branch Sources
- Add Source > git 선택
- Project Repository : 위에서 생성한 github repo 기입
  - 예 : https://github.com/유저명/s1.git

#### step 4. Build Configuration
- Jenkinsfile 유지
- 해당 `Jenkinsfile`에서 빌드 파이프라인 정의

#### step 5. Scan Multibranch Pipeline Triggers
- Periodically if not otherwise run 선택
- Interval : 1 minute

#### step 6. Save 버튼 클릭
Console Output 확인하면, Error 발생하는 것을 확인할 수 있습니다.

- 에러내용 (Build and push image with Container Builder)

```
+ PYTHONUNBUFFERED=1 gcloud builds submit -t gcr.io/REPLACE_WITH_YOUR_PROJECT_ID/gceme:master.1 .
Creating temporary tarball archive of 31 file(s) totalling 77.3 KiB before compression.
Uploading tarball of [.] to [gs://jenkins-cd-test_cloudbuild/source/1553442444.29-b06db329928943d99ce6626469c27f69.tgz]
ERROR: (gcloud.builds.submit) INVALID_ARGUMENT: invalid build: invalid image name "gcr.io/REPLACE_WITH_YOUR_PROJECT_ID/gceme:master.1" contains uppercase letters
```

## 8. 개발 환경 구성하기

`dev` 브랜치를 생성하여, 깃허브 서버에 푸시한 후, 젠킨스가 배포하도록 구성해보겠습니다.

### 8-1. `dev` 브랜치 생성

```sh
git checkout -b dev
```

### 8-2. 소스 수정

#### 8-2-1. Jenkinsfile 수정

- PROJECT_ID 수정
```sh
gcloud config get-value project
```
위 명령어로 확인후, `REPLACE_WITH_YOUR_PROJECT_ID` 를 `PROJECT_ID` 로 수정

#### 8-2-2. html.go 수정

```sh
vi html.go
```

- `blue`를 `orange`로 수정 (37, 81 라인)
```html
<div class="card orange">
```

#### 8-2-3. main.go 수정

```sh
vi main.go
```

- `1.0.0` 에서 `2.0.0` 으로 수정 (46 라인)
```go
const version string = "2.0.0"
```

## 9. 개발계 배포

- 커밋 & 푸시
```sh
git commit -am "Version 2.0.0"
git push origin dev
```

[![01](https://github.com/jmyung/kube-lab/blob/master/doc/img/jenkins-01.png?raw=true)]()

[![01](https://github.com/jmyung/kube-lab/blob/master/doc/img/jenkins-02.png?raw=true)]()

- 서비스 오브젝트 확인
```sh
jesang_myung@cloudshell:~ (jenkins-cd-test)$ kubectl get svc -n dev
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
gceme-backend    ClusterIP   10.11.255.22    <none>        8080/TCP   16m
gceme-frontend   ClusterIP   10.11.252.179   <none>        80/TCP     16m
```

- 2.0.0 배포 확인
```sh
k run -it alpine --image=alpine -- sh
# apk add curl
# curl gceme-frontend.dev.svc.cluster.local/version
2.0.0
```

## 9. 카나리 릴리스

```sh
git checkout -b canary
git push origin canary
```

[![01](https://github.com/jmyung/kube-lab/blob/master/doc/img/jenkins-03.png?raw=true)]()


[![01](https://github.com/jmyung/kube-lab/blob/master/doc/img/jenkins-04.png?raw=true)]()


```sh
export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
```

## 10. 운영계 배포

```sh
git checkout master
git merge canary
git push origin master
```

```sh
export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
2.0.0
2.0.0
2.0.0
2.0.0
2.0.0
2.0.0
```
