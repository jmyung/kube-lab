# helm 맛보기

## 1. helm 설치

```sh
$ sudo su -
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get > /tmp/get_helm.sh
% Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                               Dload  Upload   Total   Spent    Left  Speed
100  7234  100  7234    0     0  53986      0 --:--:-- --:--:-- --:--:-- 54390
$ chmod 700 /tmp/get_helm.sh
$ kubectl --namespace=kube-system create clusterrolebinding add-on-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:default
clusterrolebinding.rbac.authorization.k8s.io/add-on-cluster-admin created
```

## 2. helm install

```sh
$ mkdir charts
$ cd charts/
[charts]$ helm create httpd
Creating httpd
[charts]$ ll
total 0
drwxr-xr-x. 4 root root 93 Mar 17 10:27 httpd
[charts]$ cd httpd/
[charts/httpd]$ ll
total 8
drwxr-xr-x. 2 root root    6 Mar 17 10:27 charts
-rw-r--r--. 1 root root  101 Mar 17 10:27 Chart.yaml
drwxr-xr-x. 2 root root  106 Mar 17 10:27 templates
-rw-r--r--. 1 root root 1020 Mar 17 10:27 values.yaml
[charts/httpd]$ vi values.yaml
[charts/httpd]$ vi Chart.yaml
[charts/httpd]$ cd charts/

[charts]$ helm install --name my-httpd ./httpd/
NAME:   my-httpd
LAST DEPLOYED: Sun Mar 17 10:54:40 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Service
NAME      TYPE      CLUSTER-IP     EXTERNAL-IP  PORT(S)       AGE
my-httpd  NodePort  10.102.77.133  <none>       80:31321/TCP  0s

==> v1beta2/Deployment
NAME      DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
my-httpd  1        1        1           0          0s

==> v1/Pod(related)
NAME                       READY  STATUS             RESTARTS  AGE
my-httpd-5464544898-285fl  0/1    ContainerCreating  0         0s


NOTES:
1. Get the application URL by running these commands:
  export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-httpd)
  export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT

[charts]$ k get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        1h
my-httpd     NodePort    10.102.77.133   <none>        80:31321/TCP   56s

[charts]$   export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services my-httpd)
[charts]$   export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
[charts]$ echo http://$NODE_IP:$NODE_PORT
http://10.0.1.213:31321
[charts]$ k get po
NAME                        READY     STATUS    RESTARTS   AGE
my-httpd-5464544898-285fl   1/1       Running   0          3m
[charts]$ k get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        1h
my-httpd     NodePort    10.102.77.133   <none>        80:31321/TCP   3m
```
