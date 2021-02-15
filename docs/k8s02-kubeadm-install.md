---
layout: default
title: k8s 시리즈 01 - kubeadm 설치
nav_order: 6
---

# kubeadm 설치

설치 스크립트 추가 ( 마지막에 Ctrl+C )

```bash
$ cat > install.sh
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

스왑 제거 

```bash
$ swapoff -a
$ sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

쿠버네티스 설치

```bash
$ bash install.sh
```

호스트네임 확인

- 노드끼리 호스트네임이 같으면 제대로 설치되지 않음.
- 확인후 같은 호스트네임이 있으면 변경해주고 모든 노드를 재부팅한다

쿠버네티스 init (마스터만)

```bash
$ kubeadm init
```

kubectl 일반 계정에 설치 

```bash
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

노드 추가 

설치후 나오는 마지막 두줄 각 노드별로 실행 

```bash
$ kubeadm join 10.0.2.15:6443 --token eaic08.6b7epowm0ci8mo3y \
    --discovery-token-ca-cert-hash sha256:6e6e8fa5780858012d42711fdb559134bd1cd45a5c04ad774ce6e435fdf224e6
```

wave net 추가 (일반 계정에서 kubectl로 ) 

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```