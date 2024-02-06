# Sơ đồ bài LAB

![setup.png](setup.png?raw=true "setup.png")

# Dựng Cluster Load Balancing

Tạo ssh key

```
ssh-keygen -t rsa
```

Khởi tạo Nodes

```
cd Cluster-Loadbalancing
vagrant up
```

Sau khi tạo thành công các Node thì SSH vào từng Node để có thể dùng ansible-playbook tự động cài đặt các Tool cần thiết cho Node và dựng Cluster

Cài đặt Load Balancer

```
ansible-playbook -i playbook/lb_inventory.yml playbook/lb_playbook.yml --extra-vars "cluster_vip=172.16.0.16"
```

SSH vào IP 172.16.0.16 để kiểm tra trạng thái

```
ssh ci@172.16.0.16
sudo systemctl status haproxy
```

Cài đặt Kubernetes Cluster

```
ansible-playbook -i playbook/cluster_inventory.yml playbook/cluster_playbook.yml --extra-vars "cluster_vip=172.16.0.16"
```

Kéo File config K8s về máy Local

```
scp ci@172.16.1.11:/home/ci/.kube/config ~/.kube/config
kubectl get no
```

# 1. Chạy Trên Máy Local

## 1.1 Cài ArgoCD

```
kubectl create ns argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Dùng Port-Forward để chạy ArgoCD ở local

```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Lấy password của ArgoCD

```
argocd admin initial-password -n argocd
```

Login bằng CLI

```
argocd login <IP:Port>
```

Đổi password ArgoCD

```
argocd account update-password
```

## 1.2 Cài Đặt Longhorn Để Lưu Trữ Dữ Liệu

Thêm Repo của Longhorn

```
helm repo add longhorn https://charts.longhorn.io
helm repo update
```

Cài đặt Longhorn

```
helm install longhorn longhorn/longhorn --namespace longhorn-system --create-namespace --version 1.5.3
```

Xem các Pod của Longhorn

```
kubectl -n longhorn-system get pod
```

Truy cập vào giao diện UI thông qua Port-forward

```
kubectl port-forward svc/longhorn-frontend -n longhorn-system 8080:80
```

## 1.3 Sử Dụng CircleCI Để Test-Build-Push Code

Login vào CircleCI bằng tk Github, chọn Repo cần CI và tạo File config.yaml.

Chọn Projects -> chọn Repo -> Set Up Project -> Project Settings -> Enviroment Variables -> thêm tài khoản Docker với [Name: DOCKER_PASSWORD, Value: "password docker"], [Name: DOCKER_USER, Value: "user name"]

## 1.4 Cài Nginx Ingress Controller

Cài đặt Nginx Ingress Controller

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/baremetal/deploy.yaml
kubectl get po -n ingress-nginx
```

Chuyển tiếp các yêu cầu từ bên ngoài đến dịch vụ trong Cluster

```
kubectl -n ingress-nginx patch svc ingress-nginx-controller --patch '{"spec": { "type": "NodePort", "ports": [ { "port": 80, "nodePort": 30100 }, { "port": 443, "nodePort": 30101 } ] } }'
kubectl get svc -n ingress-nginx
```

Cài configmap

```
kubectl -n ingress-nginx patch configmap ingress-nginx-controller --patch-file patch-configmap.yaml
```

## 1.5 ArgoCD

Chạy File argocd-application.yaml để tạo App trên ArgoCD
```
kubectl apply -f argocd/argocd-application.yaml
```

##

# 2. Public Ra Domain

## 2.1 Cài 1 Cert-manager.

Mục đích chính là để tạo SSL cho domain.

```
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl create ns cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.9.1/cert-manager.crds.yaml
helm install my-release -n cert-manager --version v1.9.1 jetstack/cert-manager
```

## 2.2 Cài Nginx Ingress Controller

Nginx Ingress Controller để quản lý việc External IP Loadbalancer ra bên ngoài.

```
kubectl create ns ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install nginx-ingress -n ingress-nginx ingress-nginx/ingress-nginx --set controller.publishService.enabled=true
kubectl get svc -n ingress-nginx -o wide
```








