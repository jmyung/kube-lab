# 쿠버네티스 클러스터 설치

클러스터를 설치하는 방법에는 여러가지가 있습니다.

1. Heptio
2. kubeadm
3. kube-spray
4. cluster-api
5. kops


본 랩에서는 kubeadm 으로 단일 마스터 클러스터를 구성해봅니다.

- 참고 : https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

## 1. VM 생성

한 개의 마스터와 두개의 워커로 사용될 AWS EC2 서버를 띄워 보겠습니다.

### 1-2. VPC 생성

- 서울 리전 선택
- VPC > VPC 만들기
  - 단일 퍼블릭 서브넷이 있는 VPC
    - 노드를 퍼블릭 서블릭에 구성하는 것은 보안에 취약함 (교육 목적상 구성)
  - VPC 이름 : `vpc-kube`
  - 가용 영역 : `ap-northeast-2a`
  - 서브넷 이름 : `public subnet-kube`
  - 나머지는 디폴트
- 서브넷 확인
  - public subnet-kube 서브넷의 라우팅 테이블 : igw 로 시작하는 것 확인
- 보안 그룹 생성
  - Security group name : `sgkube`
  - Description : `sgkube`
  - VPC : `vpc-kube`
  - 생성 클릭
- 보안 그룹 인바운드 룰 수정
  - Inbound Rules 탭 > Edit rules 버튼 클릭
  - 규칙 추가
    - 유형 : 모든 TCP
    - 소스 : 위치 무관
    - Save rules 버튼 클릭
    - 참고 : 실제 사용시에는 사용할 포트만 선별하여 명시해야 합니다.

### 1-3. VM 서버 생성

총 3개를 생성합니다.

- EC2 > 인스턴스 시작
- 단계 1: Amazon Machine Image(AMI) 선택
  - `Ubuntu Server 18.04 LTS (HVM), SSD Volume Type` 선택
- 다음
- 단계 3: 인스턴스 세부 정보 구성
  - 인스턴스 개수 : `3`
  - 네트워크 : `vpc-kube`
  - 퍼블릭 IP 자동 할당 : 활성화
- 다음 ...
- 단계 6: 보안 그룹 구성
  - 기존 보안 그룹 선택 : sgkube 선택
- 검토 및 시작
  - 새 키 페어 생성
  - 키 페어 이름 : `kube-key`
  - 키 페이 다운로드
  - `인스턴스 시작` 버튼 클릭
  - 인스턴스 보기 버튼 클릭
- Name 태킹
  - 위에서 부터 Name 기입
    - master
    - worker1
    - worker2
- SSH 연결
  - 위에서 부터 마우스 오른쪽 버튼 클릭하여 SSH 연결
  - 예:
  ```sh
  ssh -i "kube-key.pem" ubuntu@ec2-100-200-300-400.ap-northeast-2.compute.amazonaws.com
  ```

## 2. 설치 레파지토리 추가

### 2-1. 도커 레파지토리 추가 (3개 서버 모두)
```sh
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

### 2-2. 쿠버네티스 레파지토리 추가 (3개 서버 모두)
```sh
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```

## 3. 쿠버네티스 클러스터 설치


### 3-1. Docker, Kubeadm, Kubelet, Kubectl 설치
```sh
sudo apt-get update
sudo apt-get install -y docker-ce=18.06.1~ce~3-0~ubuntu kubelet kubeadm kubectl
sudo apt-mark hold docker-ce kubelet kubeadm kubectl
```

### 3-2. `net.bridge.bridge-nf-call-iptables` 활성화
```sh
echo "net.bridge.bridge-nf-call-iptables=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3-3. 클러스터 초기화 (마스터노드만)
```sh
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

### 3-4. kubectl 구성 (마스터노드만)
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 3-5. flannel 네트워크 플러그인 설치 (마스터노드만)

참고 : https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#pod-network

```sh
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
```

### 3-6. 워커 노드 부트스트래핑
```sh
sudo kubeadm join $controller_private_ip:6443 --token $token --discovery-token-ca-cert-hash $hash
```

### 3-7. 노드 조회
```sh
kubectl get nodes
```
