---
title: 在国内利用 kubeadm 在 Ubuntu 中安装单 master k8s 集群
date: 2019-08-31 15:26:29
tags:
---

本文主要参考 Kubernetes 官方文档 [Creating a single control-plane cluster with kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)，但由于国内实在是访问 Google 困难，之前的教程都是从国内各种镜像中拉取 k8s 镜像，再 tag 成官方的，实在是不太优雅。好在 K8S 新版支持了指定镜像源，可以方便的指定国内源。

# 0. 安装 docker 和 k8s 相关命令行工具

## 0.1 安装 docker

利用 [daocloud](http://get.daocloud.io/#install-docker) 上的 docker 安装方法在国内安装 docker，由于默认会从官方拉，我们可以考虑利用 Aliyun 的镜像

```
curl -sSL https://get.daocloud.io/docker | sh -s -- --mirror Aliyun
```

然后配置 docker 加速器，这里我们选择 [azure 的源](http://mirror.azk8s.cn/help/docker-registry-proxy-cache.html)，非常良心

```
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s https://dockerhub.azk8s.cn/
```

根据提示重启 docker

```
sudo systemctl restart docker.service
```

## 0.2 安装 k8s 相关命令行工具——`kubelet`, `kubeadm`, `kubectl`

在这里我们利用 [中科大 (UTSC)](http://mirrors.ustc.edu.cn) 的 ubuntu k8s 源下载 k8s 相关包

```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb http://mirrors.ustc.edu.cn/kubernetes/apt/ kubernetes-xenial main
EOF

sudo apt install -y kubelet=1.15.4-00 kubeadm=1.15.4-00 kubectl=1.15.4-00
```

# 1. 初始化 Kubernetes

## 1.1 初始化 k8s master

K8s 1.11 支持了 `kubeadm` 传入配置文件，我们可以新建下述配置文件 `kubeadm.conf`

```yaml
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
etcd:
  # one of local or external
  local:
    imageRepository: "gcr.azk8s.cn/google-containers" # origin is "k8s.gcr.io"
    imageTag: "3.3.10"
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"    # for flannel
  dnsDomain: "cluster.local"
kubernetesVersion: "v1.15.4"
imageRepository: "gcr.azk8s.cn/google-containers" # origin is "k8s.gcr.io"
```

这里我们依然用到了 [azure 的 k8s 源](https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md#22-container-registry-proxy)，巨硬可以说是很良心了。上面的 `kubernetesVersion` 写明自己的 kubeadm 版本，不然会去 google 官方请求版本，可能会卡住。然后运行

```bash
kubeadm init --config kubeadm.conf
```

即可初始化集群。

完成之后，初始化成功，会输出类似于下面的信息

```
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 172.17.49.197:6443 --token zq59ch.wrr80vejg49091mw --discovery-token-ca-cert-hash sha256:197810ac44d29e4f311a639b22f1dac7dced81604c489be390a1fa5cf0b96f8d
```

执行

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 1.2 添加网络配置

上述信息做好了之后，在 master 上加入 `flannel` 插件

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml
```

如若慢的话，建议下载该文件再 kubectl apply：

```bash
curl -L -o /tmp/kube-flannel.yml https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml
kubectl apply -f /tmp/kube-flannel.yml
```

注意上述命令只需要在主节点执行。可能会遇到拉取不到镜像的问题，可能需要手动在各个节点拉取镜像，可以使用 [azure 的 k8s 源](https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md#22-container-registry-proxy) 快速拉取：

```bash
docker pull quay.azk8s.cn/coreos/flannel:v0.11.0-amd64
docker tag quay.azk8s.cn/coreos/flannel:v0.11.0-amd64 quay.io/coreos/flannel:v0.11.0-amd64
```

注意从属节点也需要自己拉取下述的镜像。

## 1.3 节点加入

在 `kubeadm init`成功后，命令行会显示 Join 命令，命令格式如下：

```
kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
```

但如果之后忘记了命令，可以用下述命令获取 `<token>`：

```
kubeadm token list
```

然后可以用下述命令获得 `<hash>`

```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

