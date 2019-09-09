# HA 마스터 클러스터 구성

본 랩에서는 `HA 마스터 클러스터 구성`에 대해 알아보겠습니다.
- 참고 : [Creating Highly Available clusters with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

## 1. 준비 : VM 띄우기

### 1-1. 프로젝트 생성

- Select a project > New Project
- `Project name` 기입 후 `CREATE` 버튼 클릭
- `Cloud Shell` 버튼 클릭하여 이하 진행 (https://cloud.google.com/shell/)
- 프로젝트ID 확인 (기입한 프로젝트명과 다를 수 있음)

### 1-2. GCP 환경정보 구성

#### 1-2-1. 리전 설정 및 GCP API 활성화

- 프로젝트ID 변수화
```sh
PROJECT_ID="프로젝트ID"
```

- 리전 설정 및 GCP API 활성화
```sh
{
   gcloud config set project ${PROJECT_ID}
   gcloud config set compute/region us-west2
   gcloud config set compute/zone us-west2-c
   gcloud services enable compute.googleapis.com
}
```

#### 1-2-2. (OPTION) 환경정보 구성 shell 생성

- **config-update.sh** 생성
```sh
mkdir setup
cd setup
vi config-update.sh
```
- 내용
```sh
PROJECT_ID="프로젝트ID"
gcloud config set project ${PROJECT_ID}
gcloud config set compute/region us-west2
gcloud config set compute/zone us-west2-c
```
- 저장후
```sh
chmod +x config-update.sh
```

> 세션이 끊기는 경우, gcloud shell 내에서 수시로 해당 **config-update.sh** 를 실행하여 환경정보를 재구성할 수 있습니다.

### 1-3. VM 생성

#### 1-3-1. 네트워크 생성

- VPC 생성, 서브넷 생성, 방화벽 해제
```sh
{
   gcloud compute networks create k8s-ha-lab-network --subnet-mode custom
   gcloud compute networks subnets create k8s-ha-lab-subnet \
     --network k8s-ha-lab-network \
     --range 10.240.0.0/24
   gcloud compute firewall-rules create k8s-ha-lab-fw-allow \
     --allow tcp,udp,icmp \
     --network k8s-ha-lab-network \
     --source-ranges 0.0.0.0/0
}
```
> 편의를 위해 실습 목적으로 방화벽에서 모든 포트를 개방하였습니다.

#### 1-3-2. 마스터 VM 생성

- 컨트롤러 용도로 사용될 VM을 3개 생성합니다.
```sh
for i in 0 1 2; do
  gcloud compute instances create controller-${i} \
    --async \
    --boot-disk-size 10GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.1${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-ha-lab-subnet \
    --tags k8s-ha-lab,controller
done
```

#### 1-3-3. 워커 VM 생성

- 워커 용도로 사용될 VM을 3개 생성합니다.
```sh
for i in 0 1 2; do
  gcloud compute instances create worker-${i} \
    --async \
    --boot-disk-size 10GB \
    --can-ip-forward \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud \
    --machine-type n1-standard-1 \
    --private-network-ip 10.240.0.2${i} \
    --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
    --subnet k8s-ha-lab-subnet \
    --tags k8s-ha-lab,worker
done
```

#### 1-3-4. 로드밸런서 VM 생성

- 로드밸런서 용도로 사용될 VM을 한개 생성합니다.
```sh
gcloud compute instances create load-balancer \
  --async \
  --boot-disk-size 10GB \
  --can-ip-forward \
  --image-family ubuntu-1804-lts \
  --image-project ubuntu-os-cloud \
  --machine-type n1-standard-1 \
  --private-network-ip 10.240.0.30 \
  --scopes compute-rw,storage-ro,service-management,service-control,logging-write,monitoring \
  --subnet k8s-ha-lab-subnet \
  --tags k8s-ha-lab,load-balancer
```
- 정적 Public IP 생성
```sh
gcloud compute addresses create k8s-ha-lab \
  --region $(gcloud config get-value compute/region)
```
> LB의 Public IP는 고정될 필요가 있습니다.

- IP 확인
```sh
gcloud compute addresses list --filter="name=('k8s-ha-lab')"
```
- 정적 IP를 로드밸런서 VM에 붙이기
```sh
gcloud compute instances delete-access-config load-balancer --access-config-name "external-nat"
gcloud compute instances add-access-config load-balancer --access-config-name "external-nat" --address [바로위에 확인된 IP 기입]
```

#### 1-3-5. 생성된 VM 확인
- 생성된 VM의 개수가 맞는지 확인합니다.
```sh
gcloud compute instances list
```
- VM 개수
| VM  | 컨트롤러 | 워커 | 로드밸런서 |
| --- | ------ | -- | -------- |
| 개수 | 3      | 3  | 1        |


## 2. 로드 밸런서 설정

nginx proxy_pass 기능을 이용하여, LB를 생성합니다. 참고로, 쿠버네티스 문서에는 [HAProxy](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/#create-load-balancer-for-kube-apiserver)를 대안으로 제시합니다.


### 2-1. nginx 설치

- 먼저 load-balancer 인스턴스에 로그인합니다.
```
gcloud compute ssh load-balancer
```

- nginx를 설치합니다.
```sh
{
   sudo apt-get install -y nginx
   sudo systemctl enable nginx
}
```

### 2-2. 세개의 마스터 노드에서 쿠버네티스 API 트래픽 밸런싱 구성

- **nginx.conf** 편집
```sh
sudo mkdir -p /etc/nginx/tcpconf.d
sudo vi /etc/nginx/nginx.conf
```

  - 마지막 줄에 추가
```sh
include /etc/nginx/tcpconf.d/*;
```



- 구성 파일을 만들어 쿠버네티스 API 로드밸런싱 구성
  - 환경 변수 설정
```
CONTROLLER0_INTERNAL_IP=10.240.0.10
CONTROLLER1_INTERNAL_IP=10.240.0.11
CONTROLLER2_INTERNAL_IP=10.240.0.12
```
> 10.240.0.10~2 는 컨트롤러 VM의 private ip 입니다.

  - proxy_pass 설정
```sh
cat << EOF | sudo tee /etc/nginx/tcpconf.d/kubernetes.conf
stream {
    upstream kubernetes {
        server ${CONTROLLER0_INTERNAL_IP}:6443;
        server ${CONTROLLER1_INTERNAL_IP}:6443;
        server ${CONTROLLER2_INTERNAL_IP}:6443;
    }

    server {
        listen 6443;
        listen 443;
        proxy_pass kubernetes;
    }
}
EOF
```
  - nginx 설정 리로드
```sh
sudo nginx -s reload
```

## 3. 클러스터 구성

### 3-1. 첫번째 컨트롤 플레인 노드 구성

gcloud compute ssh controller-0

- LOAD_BALANCER_DNS={로드밸런서 IP주소}
- LOAD_BALANCER_PORT=6443

- **kubeadm-config.yaml** 파일 생성
```
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: stable
controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"
```

```
{
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo apt install docker.io -y
}
```


- 컨트롤 플레인 초기화
```
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs
또는
sudo kubeadm init --config=kubeadm-config.yaml --upload-certs --ignore-preflight-errors=NumCPU
```




```
jesang_myung2@controller-0:~$ sudo kubeadm init --config=kubeadm-config.yaml --upload-certs --ignore-preflight-errors=NumCPU
[init] Using Kubernetes version: v1.15.3
[preflight] Running pre-flight checks
	[WARNING NumCPU]: the number of available CPUs 1 is less than the required 2
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'

[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [controller-0 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.240.0.10 34.94.10.36]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [controller-0 localhost] and IPs [10.240.0.10 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [controller-0 localhost] and IPs [10.240.0.10 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 24.023911 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.15" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
d3dcf77134564d71eb6cb52b05362bf2b0aeed79b5dacb195345f17e5b2c8db3
[mark-control-plane] Marking the node controller-0 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node controller-0 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: gq3lgh.za2a78q3luhq39om
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 34.94.10.36:6443 --token gq3lgh.za2a78q3luhq39om \
    --discovery-token-ca-cert-hash sha256:ac86400dee49e7610ab759915a7d52afb0656b45b7e9d0bd6f9614968644f788 \
    --control-plane --certificate-key d3dcf77134564d71eb6cb52b05362bf2b0aeed79b5dacb195345f17e5b2c8db3

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 34.94.10.36:6443 --token gq3lgh.za2a78q3luhq39om \
    --discovery-token-ca-cert-hash sha256:ac86400dee49e7610ab759915a7d52afb0656b45b7e9d0bd6f9614968644f788
```


- gcloud compute ssh controller-1, gcloud compute ssh controller-2

```
sudo kubeadm join 34.94.10.36:6443 --token gq3lgh.za2a78q3luhq39om \
    --discovery-token-ca-cert-hash sha256:ac86400dee49e7610ab759915a7d52afb0656b45b7e9d0bd6f9614968644f788 \
    --control-plane --certificate-key d3dcf77134564d71eb6cb52b05362bf2b0aeed79b5dacb195345f17e5b2c8db3 \
    --ignore-preflight-errors=NumCPU
```


```
[preflight] Running pre-flight checks
	[WARNING Service-Docker]: docker service is not enabled, please run 'systemctl enable docker.service'
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
	[WARNING NumCPU]: the number of available CPUs 1 is less than the required 2
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[download-certs] Downloading the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [controller-1 localhost] and IPs [10.240.0.11 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [controller-1 localhost] and IPs [10.240.0.11 127.0.0.1 ::1]
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [controller-1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 10.240.0.11 34.94.10.36]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Checking that the etcd cluster is healthy
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.15" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[etcd] Announced new etcd member joining to the existing etcd cluster
[etcd] Wrote Static Pod manifest for a local etcd member to "/etc/kubernetes/manifests/etcd.yaml"
[etcd] Waiting for the new etcd member to join the cluster. This can take up to 40s
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node controller-1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node controller-1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.
```

jesang_myung2@controller-2:~$ kubectl get no
NAME           STATUS     ROLES    AGE   VERSION
controller-0   NotReady   master   10m   v1.15.3
controller-1   NotReady   master   84s   v1.15.3
controller-2   NotReady   master   41s   v1.15.3


- WOKER

sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl


echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
sudo modprobe br_netfilter

echo '1' > sudo tee /proc/sys/net/ipv4/ip_forward



sudo kubeadm join 34.94.10.36:6443 --token gq3lgh.za2a78q3luhq39om \
    --discovery-token-ca-cert-hash sha256:ac86400dee49e7610ab759915a7d52afb0656b45b7e9d0bd6f9614968644f788 \
    --ignore-preflight-errors=NumCPU


    jesang_myung2@controller-0:~$ kubectl get no
    NAME           STATUS     ROLES    AGE     VERSION
    controller-0   NotReady   master   27m     v1.15.3
    controller-1   NotReady   master   18m     v1.15.3
    controller-2   NotReady   master   17m     v1.15.3
    worker-0       NotReady   <none>   2m10s   v1.15.3
    worker-1       NotReady   <none>   2m10s   v1.15.3
    worker-2       NotReady   <none>   2m11s   v1.15.3




    jesang_myung2@controller-0:~$ kubectl get no
    NAME           STATUS   ROLES    AGE     VERSION
    controller-0   Ready    master   29m     v1.15.3
    controller-1   Ready    master   19m     v1.15.3
    controller-2   Ready    master   19m     v1.15.3
    worker-0       Ready    <none>   3m49s   v1.15.3
    worker-1       Ready    <none>   3m49s   v1.15.3
    worker-2       Ready    <none>   3m50s   v1.15.3


```
controllers/nginx-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```


    jesang_myung2@controller-0:~$ k expose deploy/nginx-deployment --type=NodePort
  service/nginx-deployment exposed
  jesang_myung2@controller-0:~$ k get svc
  NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
  kubernetes         ClusterIP   10.96.0.1       <none>        443/TCP        40m
  nginx-deployment   NodePort    10.102.234.90   <none>        80:30552/TCP   3s
