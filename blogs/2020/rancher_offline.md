---
title: 记录一次 k3s 离线安装高可用 rancher
date: '2020-09-05 18:25:28'
sidebar: false
categories:
 - 技术
tags:
 - k3s
 - k8s
 - rancher
publish: true
---


生产环境用rancher部署k8s，首先你需要一个rancher，不能通过单节点docker run 起来，因为这不高可用

所以官方推荐使用 rke 或者 k3s 搭建，k3s 貌似是主推产品，轻量级 k8s .

> k3s: 轻量级 Kubernetes。安装简单，内存只有一半，所有的二进制都不到 100MB。



所以我尝试用k3s 搭建rancher，由于国内安装地址挂了，而直接安装又很慢，所以我采用[离线安装的方式](https://rancher2.docs.rancher.cn/docs/k3s/installation/airgap/_index)

把对应的tar包和k3s bin 文件上传到对应的服务器之后，安装以下步骤走

步骤：

1. `sudo mkdir -p /var/lib/rancher/k3s/agent/images/`
2. `sudo cp k3s /usr/local/bin/`
3. `sudo cp k3s-airgap-images-amd64.tar  /var/lib/rancher/k3s/agent/images/`
4. `sudo INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_EXEC='server  --tls-san="47.110.92.7"' ./install.sh` 这里注意如果不配置 `--tls-san="47.110.92.7"`的话，你没有办法在外部通过 kubeconfig 访问集群
5.  `cat /var/lib/rancher/k3s/server/node-token` 获取agent注册的token
6. `INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://172.23.96.246:6443 K3S_TOKEN=K106babcefe122d96865e08181fe37773af9dbe600b67798de9645b2389b29de15b::server:076884928668d11df16dde90ef62b1a9 ./install.sh` 注册 agent 也就是node节点
7. 按照文档安装cert-manager，以及rancher  https://rancher2.docs.rancher.cn/docs/rancher2/installation/k8s-install/helm-rancher/_index



汇总

```shell
sudo mkdir -p /var/lib/rancher/k3s/agent/images/

sudo cp k3s /usr/local/bin/

sudo cp k3s-airgap-images-amd64.tar  /var/lib/rancher/k3s/agent/images/

sudo INSTALL_K3S_SKIP_DOWNLOAD=true INSTALL_K3S_EXEC='server  --tls-san="47.110.92.7"' ./install.sh

//在master 节点执行
TOKEN=$(cat /var/lib/rancher/k3s/server/node-token)
echo $TOKEN

// 在node节点执行
INSTALL_K3S_SKIP_DOWNLOAD=true K3S_URL=https://172.23.96.246:6443 K3S_TOKEN=$TOKEN ./install.sh

helm repo add rancher-stable http://rancher-mirror.oss-cn-beijing.aliyuncs.com/server-charts/stable

helm install rancher rancher-stable/rancher \
 --namespace cattle-system \
 --set hostname=rancher.whoops-cn.club \
 --set ingress.tls.source=letsEncrypt \
 --set letsEncrypt.email=1025434218@qq.com

```



出现的问题

2020/09/05 10:00:33 [ERROR] ClusterController local [cluster-deploy] failed with : waiting for server-url setting to be set

有可能的 cert-manager 版本不对，没有指定版本 v0.15.0 导致一直报错