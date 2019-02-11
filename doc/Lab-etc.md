## Kubernetes graphical dashboard

### grant cluster level permissions

```sh
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```

### create a new dashboard service

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

### Change type: ClusterIP to type: NodePort

```sh
kubectl -n kube-system edit service kubernetes-dashboard
```

### Token

To login into Kubernetes dashboard, you must authenticate using a token. Use a token allocated to a service account, such as the namespace-controller. To get the token value, run the following command:

```sh
kubectl -n kube-system describe $(kubectl -n kube-system \
get secret -n kube-system -o name | grep namespace) | grep token:
```

```sh
kubectl proxy --port 8081
```
