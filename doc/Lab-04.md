# 쿠버네티스 클러스터 설치

본 랩에서는 kubeadm 으로 단일 마스터 클러스터를 구성해봅니다.
참고 : https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

## 설치 레파지토리 추가

### 도커 레파지토리 추가 (3개 서버 모두)
```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

### 쿠버네티스 레파지토리 추가 (3개 서버 모두)
```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

### Docker, Kubeadm, Kubelet, Kubectl 설치
```sh
sudo apt-get update
sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet=1.12.2-00 kubeadm=1.12.2-00 kubectl=1.12.2-00
sudo apt-mark hold docker-ce kubelet kubeadm kubectl
```

### `net.bridge.bridge-nf-call-iptables` 활성화
```sh
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 클러스터 초기화 (마스터노드만)
```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

### kubectl 구성 (마스터노드만)
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### flannel 네트워크 플러그인 설치 (마스터노드만)
```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

### 워커노드 부트스트래핑
```sh
sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash
```

### 노드 조회
```sh
kubectl get nodes
```
