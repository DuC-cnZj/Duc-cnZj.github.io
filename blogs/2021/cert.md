---
title: 数字证书原理
date: '2021-12-19 12:28:39'
sidebar: false
categories:
 - 技术
tags:
 - 证书
publish: true
---

>  以下两篇文章都讲的很到位，建议细品

1. [数字证书原理-赵化冰的博客 | Zhaohuabing Blog](https://zhaohuabing.com/post/2020-03-19-pki/)
2. [一文带你彻底厘清 Kubernetes 中的证书工作机制-赵化冰的博客 | Zhaohuabing Blog](https://zhaohuabing.com/post/2020-05-19-k8s-certificate/)

CN 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端证书则为证书申请者的姓名；

简称：O 字段，对于 SSL 证书，一般为网站域名；而对于代码签名证书则为申请单位名称；而对于客户端单位证书则为证书申请者所在单位名称；

所在城市 (Locality)| 简称：L 字段 
所在省份 (State/Provice)| 简称：ST 字段 
所在国家 (Country)| 简称：C 字段，只能是国家字母缩写，如中国：CN

## 证书中的 hosts

> [kube-apiserver 证书中的 hosts 受限问题 · Issue #185 · opsnull/follow-me-install-kubernetes-cluster · GitHub](https://github.com/opsnull/follow-me-install-kubernetes-cluster/issues/185)

我简单的把证书分为三类：

- server 证书：server auth；
- client 证书：client auth；
- peer 证书：server auth + client auth。

经过我的实践和理解：client 证书可以不用指定 `hosts` 字段，server 证书和 peer 证书必须要指定 `hosts` 字段，否则客户端访问时会提示错误：

```
Unable to connect to the server: x509: cannot validate certificate for 192.168.1.x because it doesn't contain any IP SANs
```

确实有这个问题，用 go 试了下，hosts 必须包含服务的地址不然就会出问题

但是当我在做 HA master （我使用 Keepalived 新增了一个 VIP，并且使用 HAProxy 代理 https 的 kube-apiserver）时，就会存在一个问题：由于 VIP 并不在 kube-apiserver 的 `hosts` 中，导致所有客户端都没有办法通过 VIP 来访问 kube-apiserver （提示 VIP 不在 SANS 中）。

## golang 自签证书实现 https

[Go HTTPS Demo: golang 生成https 自签名证书，及服务端、客户端双向认证代码](https://toscode.gitee.com/huy_admin/Go-HTTPS-Demo)

## k8s 集群 Group 和 用户

1. 准备集群绑定 yaml
   ```yaml
   apiVersion: rbac.authorization.k8s.io/v1
   kind: ClusterRoleBinding
   metadata:
     name: custom-reader
   subjects:
   - kind: Group
     name: reader
     apiGroup: rbac.authorization.k8s.io
   roleRef:
     kind: ClusterRole
     name: view
     apiGroup: rbac.authorization.k8s.io
   ```

2. 准备证书所需的配置文件
    ```toml
    [ req ]
    default_bits = 2048
    prompt = no
    default_md = sha256
    req_extensions = req_ext
    distinguished_name = dn

    [ dn ]
    C = CN
    ST = ZJ
    L = HangZhou
    # CN 是集群中的 user
    CN = duc
    # O：事先在集群创建了 Group, 用户使用事先创建好的群组就会有这个群组的权限
    # 总之这个是和 Group 关联的
    O = reader

    [ req_ext ]
    subjectAltName = @alt_names

    [ alt_names ]
    IP.1 = 127.0.0.1

    [ v3_ext ]
    authorityKeyIdentifier=keyid,issuer:always
    basicConstraints=CA:FALSE
    keyUsage=keyEncipherment,dataEncipherment
    extendedKeyUsage=serverAuth,clientAuth
    subjectAltName=@alt_names
    ```

3. 先创建一个用户
   ```shell
   openssl genrsa -out duc.key 2048
   
   openssl req -new -key duc.key -out duc.csr -config cfg
   
   # ca 证书需要自己拷贝出来
   openssl x509 -req -in duc.csr -CA pki/ca.crt -CAkey pki/ca.key \
   -CAcreateserial -out duc.crt -days 10000 \
   -extensions v3_ext -extfile cfg
   ```

4. 设置用户
   ```shell
   kc config set-credentials duc --client-certificate duc.crt --client-key duc.key
   kc config set-context duc --cluster docker-desktop --user duc
   kc config use-context duc
   ```

5. 测试
   ```shell
   kc auth can-i edit po
   kc auth can-i get deploy
   kc auth can-i create ns
   kc auth can-i get ns
   ```
