---
layout: post
title: kubernetes 1.7.3 + calico 多 Master
categories: kubernetes
description: kubernetes 1.7.3 + calico 多 Master
keywords: kubernetes
feature-img: "assets/img/pexels/desk-top.jpeg"
---

# kubernetes 1.7.3 

> 基于 二进制 文件部署
> 本地化 kube-apiserver, kube-controller-manager , kube-scheduler
> 我这边配置  既是 master 也是 nodes

## 环境说明

```
k8s-master-1: 10.6.0.140
k8s-master-2: 10.6.0.187
k8s-master-3: 10.6.0.188
```


## 初始化环境

```
hostnamectl --static set-hostname hostname

10.6.0.140 - k8s-master-1
10.6.0.187 - k8s-master-2
10.6.0.188 - k8s-master-3
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

10.6.0.140 k8s-master-1
10.6.0.187 k8s-master-2
10.6.0.188 k8s-master-3
```


# 创建 验证

> 这里使用 CloudFlare 的 PKI 工具集 cfssl 来生成 Certificate Authority (CA) 证书和秘钥文件。


## 安装 cfssl


```
mkdir -p /opt/local/cfssl

cd /opt/local/cfssl

wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 cfssl-certinfo

chmod +x *
```


## 创建 CA 证书配置

```
mkdir /opt/ssl

cd /opt/ssl

/opt/local/cfssl/cfssl print-defaults config > config.json

/opt/local/cfssl/cfssl print-defaults csr > csr.json


```


```
# config.json 文件

{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}

```



```
# csr.json 文件

{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}


```

## 生成 CA 证书和私钥

```

cd /opt/ssl/

/opt/local/cfssl/cfssl gencert -initca csr.json | /opt/local/cfssl/cfssljson -bare ca


[root@k8s-master-1 ssl]# ls -lt
总用量 20
-rw-r--r-- 1 root root 1005 7月   3 17:26 ca.csr
-rw------- 1 root root 1675 7月   3 17:26 ca-key.pem
-rw-r--r-- 1 root root 1363 7月   3 17:26 ca.pem
-rw-r--r-- 1 root root  210 7月   3 17:24 csr.json
-rw-r--r-- 1 root root  292 7月   3 17:23 config.json

```


## 分发证书


```
# 创建证书目录
mkdir -p /etc/kubernetes/ssl

# 拷贝所有文件到目录下
cp * /etc/kubernetes/ssl

# 这里要将文件拷贝到所有的k8s 机器上

scp * 10.6.0.187:/etc/kubernetes/ssl/

scp * 10.6.0.188:/etc/kubernetes/ssl/

```



# 安装 docker


```
# 导入 yum 源

# 安装 yum-config-manager

yum -y install yum-utils

# 导入
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo


# 更新 repo
yum makecache

# 安装

yum install docker-ce -y
```
 

## 更改docker 配置

```
# 修改配置

vi /usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS $DOCKER_OPTS $DOCKER_DNS_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

```

```
# 修改其他配置

mkdir -p /usr/lib/systemd/system/docker.service.d/


vi /usr/lib/systemd/system/docker.service.d/docker-options.conf

# 添加如下 :   (注意 environment 必须在同一行，如果出现换行会无法加载)


[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 --graph=/opt/docker --registry-mirror=http://b438f72b.m.daocloud.io --disable-legacy-registry --iptables=false"

```





```
# 重新读取配置，启动 docker 
systemctl daemon-reload
systemctl start docker
systemctl enable docker

```





# etcd 集群

> etcd 是k8s集群的基础组件


## 安装 etcd

```
yum -y install etcd
```

## 创建 etcd 证书


```
cd /opt/ssl/

vi etcd-csr.json

{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.6.0.140",
    "10.6.0.187",
    "10.6.0.188"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

```


```
# 生成 etcd   密钥

/opt/local/cfssl/cfssl gencert -ca=/opt/ssl/ca.pem \
  -ca-key=/opt/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes etcd-csr.json | /opt/local/cfssl/cfssljson -bare etcd

```


```
# 查看生成

[root@k8s-master-1 ssl]# ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem



# 拷贝到etcd服务器

# etcd-1 
cp etcd*.pem /etc/kubernetes/ssl/

# etcd-2
scp etcd* 10.6.0.187:/etc/kubernetes/ssl/

# etcd-3
scp etcd* 10.6.0.188:/etc/kubernetes/ssl/



# 如果 etcd 非 root 用户，读取证书会提示没权限

chmod 644 /etc/kubernetes/ssl/etcd-key.pem

```




## 修改 etcd 配置



> 修改 etcd 启动文件 /usr/lib/systemd/system/etcd.service

```
# etcd-1


vi /usr/lib/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \
  --name=etcd1 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://10.6.0.140:2380 \
  --listen-peer-urls=https://10.6.0.140:2380 \
  --listen-client-urls=https://10.6.0.140:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://10.6.0.140:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://10.6.0.140:2380,etcd2=https://10.6.0.187:2380,etcd3=https://10.6.0.188:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

``` 

```
# etcd-2


vi /usr/lib/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \
  --name=etcd2 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://10.6.0.187:2380 \
  --listen-peer-urls=https://10.6.0.187:2380 \
  --listen-client-urls=https://10.6.0.187:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://10.6.0.187:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://10.6.0.140:2380,etcd2=https://10.6.0.187:2380,etcd3=https://10.6.0.188:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

``` 

```
# etcd-3


vi /usr/lib/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/usr/bin/etcd \
  --name=etcd3 \
  --cert-file=/etc/kubernetes/ssl/etcd.pem \
  --key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \
  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls=https://10.6.0.188:2380 \
  --listen-peer-urls=https://10.6.0.188:2380 \
  --listen-client-urls=https://10.6.0.188:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://10.6.0.188:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://10.6.0.140:2380,etcd2=https://10.6.0.187:2380,etcd3=https://10.6.0.188:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

``` 



## 启动 etcd

> 分别启动 所有节点的 etcd 服务

```
systemctl enable etcd

systemctl start etcd

systemctl status etcd
```


```
# 如果报错 请使用
journalctl -f -t etcd  和 journalctl -u etcd 来定位问题

```




## 验证 etcd 集群状态

> 查看 etcd 集群状态：

```
etcdctl --endpoints=https://10.6.0.140:2379 \
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        cluster-health

member 29262d49176888f5 is healthy: got healthy result from https://10.6.0.188:2379
member d4ba1a2871bfa2b0 is healthy: got healthy result from https://10.6.0.140:2379
member eca58ebdf44f63b6 is healthy: got healthy result from https://10.6.0.187:2379
cluster is healthy
```


> 查看 etcd 集群成员：

```
etcdctl --endpoints=https://10.6.0.140:2379 \
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        member list


29262d49176888f5: name=etcd3 peerURLs=https://10.6.0.188:2380 clientURLs=https://10.6.0.188:2379 isLeader=false
d4ba1a2871bfa2b0: name=etcd1 peerURLs=https://10.6.0.140:2380 clientURLs=https://10.6.0.140:2379 isLeader=true
eca58ebdf44f63b6: name=etcd2 peerURLs=https://10.6.0.187:2380 clientURLs=https://10.6.0.187:2379 isLeader=false

```

# 安装 kubectl 工具


## Master 端


```
# 首先安装 kubectl

wget https://dl.k8s.io/v1.7.3/kubernetes-client-linux-amd64.tar.gz

tar -xzvf kubernetes-client-linux-amd64.tar.gz

cp kubernetes/client/bin/* /usr/local/bin/

chmod a+x /usr/local/bin/kube*


# 验证安装

kubectl version
Client Version: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.3", GitCommit:"2c2fe6e8278a5db2d15a013987b53968c743f2a1", GitTreeState:"clean", BuildDate:"2017-08-03T07:00:21Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}
The connection to the server localhost:8080 was refused - did you specify the right host or port?

```

## 创建 admin 证书

> kubectl 与 kube-apiserver 的安全端口通信，需要为安全通信提供 TLS 证书和秘钥。


```
cd /opt/ssl/

vi admin-csr.json


{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

```


```
# 生成 admin 证书和私钥
cd /opt/ssl/

/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/config.json \
  -profile=kubernetes admin-csr.json | /opt/local/cfssl/cfssljson -bare admin


# 查看生成

[root@k8s-master-1 ssl]# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem

cp admin*.pem /etc/kubernetes/ssl/

scp admin*.pem 10.6.0.187:/etc/kubernetes/ssl/

scp admin*.pem 10.6.0.188:/etc/kubernetes/ssl/
```


## 配置 kubectl kubeconfig 文件


> server  配置为 本机IP  各自连接本机的 Api

```
# 配置 kubernetes 集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://10.6.0.140:6443


# 配置 客户端认证

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
  


kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin


kubectl config use-context kubernetes

```


## kubectl config 文件

```
# kubeconfig 文件在 如下:

/root/.kube

```



# 部署 Kubernetes Master 节点

> Master 需要部署 kube-apiserver , kube-scheduler , kube-controller-manager 这三个组件。
kube-scheduler 作用是调度pods分配到那个node里，简单来说就是资源调度。
kube-controller-manager 作用是 对 deployment controller , replication controller, endpoints controller, namespace controller, and serviceaccounts controller等等的循环控制，与kube-apiserver交互。


## 安装 组件


```
# 从github 上下载版本

cd /tmp

wget https://dl.k8s.io/v1.7.3/kubernetes-server-linux-amd64.tar.gz

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/

```


## 创建 kubernetes 证书

```
cd /opt/ssl

vi kubernetes-csr.json

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.6.0.140",
    "10.6.0.187",
    "10.6.0.188",
    "10.254.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}


## 这里 hosts 字段中 三个 IP 分别为 127.0.0.1 本机， 10.6.0.140 和 10.6.0.187 为 Master 的IP，多个Master需要写多个 10.254.0.1 为 kubernetes SVC 的 IP， 一般是 部署网络的第一个IP , 如: 10.254.0.1 ， 在启动完成后，我们使用   kubectl get svc ， 就可以查看到

```


## 生成 kubernetes 证书和私钥


```
/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/config.json \
  -profile=kubernetes kubernetes-csr.json | /opt/local/cfssl/cfssljson -bare kubernetes

# 查看生成

[root@k8s-master-1 ssl]# ls -lt kubernetes*
-rw-r--r-- 1 root root 1245 7月   4 11:25 kubernetes.csr
-rw------- 1 root root 1679 7月   4 11:25 kubernetes-key.pem
-rw-r--r-- 1 root root 1619 7月   4 11:25 kubernetes.pem
-rw-r--r-- 1 root root  436 7月   4 11:23 kubernetes-csr.json


# 拷贝到目录
cp -r kubernetes* /etc/kubernetes/ssl/

scp -r kubernetes* 10.6.0.187:/etc/kubernetes/ssl/

scp -r kubernetes* 10.6.0.188:/etc/kubernetes/ssl/

```

## 配置 kube-apiserver

> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token  一致，如果一致则自动为 kubelet生成证书和秘钥。


```
# 生成 token

[root@k8s-master-1 ssl]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
10b459a82af1e16663f25061372fdab4


# 创建 token.csv 文件

cd /opt/ssl

vi token.csv

10b459a82af1e16663f25061372fdab4,kubelet-bootstrap,10001,"system:kubelet-bootstrap"


# 拷贝

cp token.csv /etc/kubernetes/

scp token.csv 10.6.0.187:/etc/kubernetes/

scp token.csv 10.6.0.188:/etc/kubernetes/
```


## 创建 kube-apiserver.service 文件

```

# 自定义 系统 service 文件一般存于 /etc/systemd/system/ 下
# 配置为 各自的本地 IP

vi /etc/systemd/system/kube-apiserver.service

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --advertise-address=10.6.0.140 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/lib/audit.log \
  --authorization-mode=RBAC \
  --bind-address=10.6.0.140 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
  --etcd-servers=https://10.6.0.140:2379,https://10.6.0.187:2379,https://10.6.0.188:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=10.6.0.140 \
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --experimental-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```



```
# 这里面要注意的是 --service-node-port-range=30000-32000
# 这个地方是 映射外部端口时 的端口范围，随机映射也在这个范围内映射，指定映射端口必须也在这个范围内。
```





 
## 启动 kube-apiserver

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver

```


## 配置 kube-controller-manager

> master 配置为 各自 本地 IP

```
# 创建 kube-controller-manager.service 文件

vi /etc/systemd/system/kube-controller-manager.service


[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://10.6.0.140:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.254.0.0/16 \
  --cluster-cidr=10.233.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target


```



## 启动 kube-controller-manager

```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```


## 配置 kube-scheduler

> master 配置为 各自 本地 IP

```
# 创建 kube-cheduler.service 文件

vi /etc/systemd/system/kube-scheduler.service


[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://10.6.0.140:8080 \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```

## 启动 kube-scheduler

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler

```
 

## 验证 Master 节点

```
[root@k8s-master-1 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 



[root@k8s-master-2 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-2               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}  



[root@k8s-master-3 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   

```
 


## 部署 Master Node 部分


> Node 部分 需要部署的组件有 docker  calico  kubectl  kubelet  kube-proxy 这几个组件。


## 配置 kubelet

> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求(certificatesigningrequests)。


```
# 先创建认证请求
# user 为 master 中 token.csv 文件里配置的用户
# 只需创建一次就可以

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

```

## 创建 kubelet kubeconfig 文件

> server 配置为 master 本机 IP

```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://10.6.0.140:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=10b459a82af1e16663f25061372fdab4 \
  --kubeconfig=bootstrap.kubeconfig


# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
  
  
# 配置默认关联
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 拷贝生成的 bootstrap.kubeconfig 文件

mv bootstrap.kubeconfig /etc/kubernetes/

```


## 创建 kubelet.service 文件

```
# 创建 kubelet 目录

> 配置为 node 本机 IP

mkdir /var/lib/kubelet

vi /etc/systemd/system/kubelet.service


[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --address=10.6.0.140 \
  --hostname-override=10.6.0.140 \
  --pod-infra-container-image=jicki/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --require-kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.254.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --v=2
ExecStopPost=/sbin/iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 4194 -j ACCEPT
ExecStopPost=/sbin/iptables -A INPUT -s 172.16.0.0/12 -p tcp --dport 4194 -j ACCEPT
ExecStopPost=/sbin/iptables -A INPUT -s 192.168.0.0/16 -p tcp --dport 4194 -j ACCEPT
ExecStopPost=/sbin/iptables -A INPUT -p tcp --dport 4194 -j DROP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```


```
# 如上配置:
10.6.0.140       为本机的IP
10.254.0.2       预分配的 dns 地址
cluster.local.   为 kubernetes 集群的 domain
jicki/pause-amd64:3.0  这个是 pod 的基础镜像，既 gcr 的 gcr.io/google_containers/pause-amd64:3.0 镜像， 下载下来修改为自己的仓库中的比较快。
```

## 启动 kubelet

```

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet

```


## 配置 TLS 认证

```
# 查看 csr 的名称

[root@k8s-master-1 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-LkH2ZX9b2kACKmkNbp9PnK6BYAa5fMeh7nWtrGCipYc   58s       kubelet-bootstrap   Pending
node-csr-cxXvnzvukInZrSXT1EJTFaDzERFsuwsR2hCcgWyYZ2o   1m        kubelet-bootstrap   Pending
node-csr-jcQdD_haTRkPMTXwcHeyjQZUt2lb1S4rDeTgKUeQwgM   1m        kubelet-bootstrap   Pending


# 增加 认证

[root@k8s-master-1 ~]# kubectl certificate approve node-csr-LkH2ZX9b2kACKmkNbp9PnK6BYAa5fMeh7nWtrGCipYc
certificatesigningrequest "node-csr-LkH2ZX9b2kACKmkNbp9PnK6BYAa5fMeh7nWtrGCipYc" approved
[root@k8s-master-1 ~]# 
[root@k8s-master-1 ~]# kubectl certificate approve node-csr-cxXvnzvukInZrSXT1EJTFaDzERFsuwsR2hCcgWyYZ2o
certificatesigningrequest "node-csr-cxXvnzvukInZrSXT1EJTFaDzERFsuwsR2hCcgWyYZ2o" approved
[root@k8s-master-1 ~]# 
[root@k8s-master-1 ~]# kubectl certificate approve node-csr-jcQdD_haTRkPMTXwcHeyjQZUt2lb1S4rDeTgKUeQwgM
certificatesigningrequest "node-csr-jcQdD_haTRkPMTXwcHeyjQZUt2lb1S4rDeTgKUeQwgM" approved
```




## 验证 nodes

```
[root@k8s-master-1 ~]# kubectl get nodes
NAME         STATUS    AGE       VERSION
10.6.0.140   Ready     27s       v1.7.3
10.6.0.187   Ready     20s       v1.7.3
10.6.0.188   Ready     37s       v1.7.3


# 成功以后会自动生成配置文件与密钥

# 配置文件

ls /etc/kubernetes/kubelet.kubeconfig   
/etc/kubernetes/kubelet.kubeconfig


# 密钥文件

ls /etc/kubernetes/ssl/kubelet*
/etc/kubernetes/ssl/kubelet-client.crt  /etc/kubernetes/ssl/kubelet.crt
/etc/kubernetes/ssl/kubelet-client.key  /etc/kubernetes/ssl/kubelet.key

```


## 配置 kube-proxy


## 创建 kube-proxy 证书

```
# 证书方面由于我们node端没有装 cfssl
# 我们回到 master 端 机器 去配置证书，然后拷贝过来

[root@k8s-master-1 ~]# cd /opt/ssl


vi kube-proxy-csr.json

{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

```

## 生成 kube-proxy 证书和私钥

```
/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/config.json \
  -profile=kubernetes  kube-proxy-csr.json | /opt/local/cfssl/cfssljson -bare kube-proxy
  
# 查看生成
ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem

# 拷贝到目录
cp kube-proxy*.pem /etc/kubernetes/ssl/

scp kube-proxy*.pem 10.6.0.187:/etc/kubernetes/ssl/

scp kube-proxy*.pem 10.6.0.188:/etc/kubernetes/ssl/
```


## 创建 kube-proxy kubeconfig 文件

> server 配置为各自 本机IP

```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://10.6.0.140:6443 \
  --kubeconfig=kube-proxy.kubeconfig


# 配置客户端认证

kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
  
  
# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig



# 配置默认关联
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 拷贝到目录
mv kube-proxy.kubeconfig /etc/kubernetes/

```


## 创建 kube-proxy.service 文件


> 配置为 各自的 IP

```
# 创建 kube-proxy 目录

mkdir -p /var/lib/kube-proxy


vi /etc/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=10.6.0.140 \
  --hostname-override=10.6.0.140 \
  --cluster-cidr=10.254.0.0/16 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```


## 启动 kube-proxy

```

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy

```


#  部署 Node 节点 

> Node 节点 基于 Nginx 负载 API 做 Master HA

```
# master 之间除 api server 以外其他组件通过 etcd 选举，api server 默认不作处理；在每个 node 上启动一个 nginx，每个 nginx 反向代理所有 api server，node 上 kubelet、kube-proxy 连接本地的 nginx 代理端口，当 nginx 发现无法连接后端时会自动踢掉出问题的 api server，从而实现 api server 的 HA
```


![ HAMaster][2]



```
cd /tmp

wget https://dl.k8s.io/v1.7.3/kubernetes-server-linux-amd64.tar.gz

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-proxy,kubelet} /usr/local/bin/

```


```
# ALL node

mkdir -p /etc/kubernetes/ssl/

scp ca.pem kube-proxy.pem kube-proxy-key.pem  node-*:/etc/kubernetes/ssl/


```



```
# kubelet

# 首先 创建 kubelet kubeconfig 文件


kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig


# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=10b459a82af1e16663f25061372fdab4 \
  --kubeconfig=bootstrap.kubeconfig


# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
  
  
# 配置默认关联
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 拷贝生成的 bootstrap.kubeconfig 文件

mv bootstrap.kubeconfig /etc/kubernetes/

```


```

# 创建 kube-proxy kubeconfig 文件


kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-proxy.kubeconfig


# 配置客户端认证

kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
  
  
# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig



# 配置默认关联
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 拷贝到目录
mv kube-proxy.kubeconfig /etc/kubernetes/
```


## 创建Nginx 代理

> 在每个 node 都必须创建一个 Nginx 代理， 这里特别注意， 当 Master 也做为 Node 的时候 不需要配置 Nginx-proxy 

```
# 创建配置目录
mkdir -p /etc/nginx

# 写入代理配置

cat << EOF >> /etc/nginx/nginx.conf
error_log stderr notice;

worker_processes auto;
events {
  multi_accept on;
  use epoll;
  worker_connections 1024;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 10.6.0.140:6443;
        server 10.6.0.187:6443;
        server 10.6.0.188:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

```


```
# 配置 Nginx 基于 docker 进程，然后配置 systemd 来启动

cat << EOF >> /etc/systemd/system/nginx-proxy.service

[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 6443:6443 \\
                              -v /etc/nginx:/etc/nginx \\
                              --name nginx-proxy \\
                              --net=host \\
                              --restart=on-failure:5 \\
                              --memory=512M \\
                              nginx:1.13.3-alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF

```



```
# 启动 Nginx

systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy


# 重启 Node 的 kubelet 与 kube-proxy

systemctl restart kubelet
systemctl status kubelet

systemctl restart kube-proxy
systemctl status kube-proxy

```


## Master 配置 TLS 认证

```
# 查看 csr 的名称

[root@k8s-master-1 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-LkH2ZX9b2kACKmkNbp9PnK6BYAa5fMeh7nWtrGCipYc   58s       kubelet-bootstrap   Pending
node-csr-cxXvnzvukInZrSXT1EJTFaDzERFsuwsR2hCcgWyYZ2o   1m        kubelet-bootstrap   Pending
node-csr-jcQdD_haTRkPMTXwcHeyjQZUt2lb1S4rDeTgKUeQwgM   1m        kubelet-bootstrap   Pending


# 增加 认证

[root@k8s-master-1 ~]# kubectl certificate approve node-csr-LkH2ZX9b2kACKmkNbp9PnK6BYAa5fMeh7nWtrGCipYc
certificatesigningrequest "node-csr-LkH2ZX9b2kACKmkNbp9PnK6BYAa5fMeh7nWtrGCipYc" approved
[root@k8s-master-1 ~]# 
[root@k8s-master-1 ~]# kubectl certificate approve node-csr-cxXvnzvukInZrSXT1EJTFaDzERFsuwsR2hCcgWyYZ2o
certificatesigningrequest "node-csr-cxXvnzvukInZrSXT1EJTFaDzERFsuwsR2hCcgWyYZ2o" approved
[root@k8s-master-1 ~]# 
[root@k8s-master-1 ~]# kubectl certificate approve node-csr-jcQdD_haTRkPMTXwcHeyjQZUt2lb1S4rDeTgKUeQwgM
certificatesigningrequest "node-csr-jcQdD_haTRkPMTXwcHeyjQZUt2lb1S4rDeTgKUeQwgM" approved
```


# Calico 网络


## 修改  kubelet.service

```
vi /etc/systemd/system/kubelet.service

# 增加 如下配置

  --network-plugin=cni \


# 重新加载配置
systemctl daemon-reload
systemctl restart kubelet.service
systemctl status kubelet.service

```

## 修改  kube-proxy.service

```
# 重新加载配置
systemctl daemon-reload
systemctl restart kube-proxy.service
systemctl status kube-proxy.service

```


## 安装 Calico 

> 官网地址 http://docs.projectcalico.org/v2.3/getting-started/kubernetes/installation/hosted/hosted

```
# 下载 yaml 文件

wget http://docs.projectcalico.org/v2.3/getting-started/kubernetes/installation/hosted/calico.yaml

wget http://docs.projectcalico.org/v2.3/getting-started/kubernetes/installation/rbac.yaml


# 下载 镜像

# 国外镜像 有墙
quay.io/calico/node:v1.3.0
quay.io/calico/cni:v1.9.1
quay.io/calico/kube-policy-controller:v0.6.0


# 国内镜像
jicki/node:v1.3.0
jicki/cni:v1.9.1
jicki/kube-policy-controller:v0.6.0


```

## 配置 calico

```
vi calico.yaml

# 注意修改如下选项:


  etcd_endpoints: "https://10.6.0.140:2379,https://10.6.0.187:2379,https://10.6.0.188:2379"
  
    etcd_ca: "/calico-secrets/etcd-ca"  
    etcd_cert: "/calico-secrets/etcd-cert"
    etcd_key: "/calico-secrets/etcd-key"  


# 这里面要写入 base64 的信息



data:
  etcd-key: (cat /etc/kubernetes/ssl/etcd-key.pem | base64 | tr -d '\n')
  etcd-cert: (cat /etc/kubernetes/ssl/etcd.pem | base64 | tr -d '\n')
  etcd-ca: (cat /etc/kubernetes/ssl/ca.pem | base64 | tr -d '\n')


    - name: CALICO_IPV4POOL_CIDR
      value: "10.233.0.0/16"
      


```


## 导入 yaml 文件

```
[root@k8s-master-1 ~]# kubectl apply -f calico.yaml 
configmap "calico-config" created
secret "calico-etcd-secrets" created
daemonset "calico-node" created
deployment "calico-policy-controller" created
serviceaccount "calico-policy-controller" created
serviceaccount "calico-node" created


[root@k8s-master-1 ~]# kubectl apply -f rbac.yaml
clusterrole "calico-policy-controller" created
clusterrolebinding "calico-policy-controller" created
clusterrole "calico-node" created
clusterrolebinding "calico-node" created

```

## 验证 Calico

```

[root@k8s-master-1 ~]# kubectl get pods -n kube-system 
NAME                                       READY     STATUS    RESTARTS   AGE
calico-node-2dsq4                          2/2       Running   0          6m
calico-node-9ktvk                          2/2       Running   0          6m
calico-node-gwmx5                          2/2       Running   0          6m
calico-policy-controller-458850194-pn65p   1/1       Running   0          6m


```


## 安装 Calicoctl

```
cd /usr/local/bin/

wget -c  https://github.com/projectcalico/calicoctl/releases/download/v1.3.0/calicoctl

chmod +x calicoctl



## 创建 calicoctl 配置文件

# 配置文件， 在 安装了 calico 网络的 机器下

mkdir /etc/calico

vi /etc/calico/calicoctl.cfg


apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "https://10.6.0.140:2379,https://10.6.0.187:2379,https://10.6.0.188:2379"
  etcdKeyFile: "/etc/kubernetes/ssl/etcd-key.pem"
  etcdCertFile: "/etc/kubernetes/ssl/etcd.pem"
  etcdCACertFile: "/etc/kubernetes/ssl/ca.pem"




# 查看 calico 状态

[root@k8s-master-1 ~]# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.6.0.188   | node-to-node mesh | up    | 10:11:59 | Established |
| 10.6.0.187   | node-to-node mesh | up    | 10:16:32 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

```


# 测试集群


```
# 创建一个 nginx deplyment

apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports: 
            - containerPort: 80
            
---

apiVersion: v1 
kind: Service
metadata: 
  name: nginx-svc 
spec: 
  ports: 
    - port: 80
      targetPort: 80
      protocol: TCP 
  selector: 
    name: nginx

```

```
[root@k8s-master-1 ~]# kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
nginx-dm-2214564181-lxff5   1/1       Running   0          14m
nginx-dm-2214564181-qm1bp   1/1       Running   0          14m


[root@k8s-master-1 ~]# kubectl get deployment
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-dm   2         2         2            2           14m


[root@k8s-master-1 ~]# kubectl get svc
NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.254.0.1      <none>        443/TCP   4h
nginx-svc    10.254.129.54   <none>        80/TCP    15m

```


```
# 在 node 里 curl

[root@k8s-node-1 ~]# curl 10.254.129.54
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```



# 配置 KubeDNS

> 官方 github yaml 相关 https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns


## 下载镜像

```
# 官方镜像
gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4
gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4


# 我的镜像
jicki/k8s-dns-sidecar-amd64:1.14.4
jicki/k8s-dns-kube-dns-amd64:1.14.4
jicki/k8s-dns-dnsmasq-nanny-amd64:1.14.4

```



## 下载 yaml 文件


```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-cm.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-sa.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-controller.yaml.base


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-svc.yaml.base


# 修改后缀

mv kubedns-controller.yaml.base kubedns-controller.yaml

mv kubedns-svc.yaml.base kubedns-svc.yaml

```


## 系统预定义的 RoleBinding

> 预定义的 RoleBinding system:kube-dns 将 kube-system 命名空间的 kube-dns ServiceAccount 与 system:kube-dns Role 绑定， 该 Role 具有访问 kube-apiserver DNS 相关 API 的权限；


```
[root@k8s-master-1 kubedns]# kubectl get clusterrolebindings system:kube-dns -o yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2017-07-04T04:15:13Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-dns
  resourceVersion: "106"
  selfLink: /apis/rbac.authorization.k8s.io/v1beta1/clusterrolebindings/system%3Akube-dns
  uid: 60c1e0e1-606f-11e7-b212-d4ae52d1f0c9
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-dns
subjects:
- kind: ServiceAccount
  name: kube-dns
  namespace: kube-system
  
```

## 修改 kubedns-svc.yaml

```
# kubedns-svc.yaml 中 clusterIP: __PILLAR__DNS__SERVER__ 修改为我们之前定义的 dns IP 10.254.0.2

cat kubedns-svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
    
```


## 修改 kubedns-controller.yaml

```
1. # 修改 --domain=__PILLAR__DNS__DOMAIN__.   为 我们之前 预定的 domain 名称 --domain=cluster.local.

2. # 修改 --server=/__PILLAR__DNS__DOMAIN__/127.0.0.1#10053  中 domain 为我们之前预定的 --server=/cluster.local./127.0.0.1#10053

3. # 修改 --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__, 中的 domain 为我们之前预定的  --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local.,

4. # 修改 --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,  中的 domain 为我们之前预定的  --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local.,

```

## 导入 yaml 文件

```
# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *

# 导入

[root@k8s-master-1 kubedns]# kubectl create -f .
configmap "kube-dns" created
deployment "kube-dns" created
serviceaccount "kube-dns" created
service "kube-dns" created

```

## 查看 kubedns 服务

```
[root@k8s-master-1 kubedns]# kubectl get all --namespace=kube-system
NAME                                          READY     STATUS    RESTARTS   AGE
po/calico-node-2dsq4                          2/2       Running   0          15h
po/calico-node-9ktvk                          2/2       Running   0          15h
po/calico-node-gwmx5                          2/2       Running   0          15h
po/calico-policy-controller-458850194-pn65p   1/1       Running   0          15h
po/kube-dns-1511229508-jxkvs                  3/3       Running   0          4m

NAME           CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
svc/kube-dns   10.254.0.2   <none>        53/UDP,53/TCP   4m

NAME                              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/calico-policy-controller   1         1         1            1           15h
deploy/kube-dns                   1         1         1            1           4m

NAME                                    DESIRED   CURRENT   READY     AGE
rs/calico-policy-controller-458850194   1         1         1         15h
rs/kube-dns-1511229508                  1         1         1         4m

```


## 验证 dns 服务

> 在验证 dns 之前，在 dns 未部署之前创建的 pod 与 deployment 等，都必须删除，重新部署，否则无法解析


```
# 导入之前的 nginx-dm yaml文件

[root@k8s-master-1 ~]# kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP              NODE
nginx-dm-2214564181-0ctcx   1/1       Running   0          27s       10.233.168.1    10.6.0.188
nginx-dm-2214564181-brz79   1/1       Running   0          3m        10.233.196.2    10.6.0.140
nginx-dm-2214564181-z8whk   1/1       Running   0          3m        10.233.182.65   10.6.0.187

[root@k8s-master-1 ~]# kubectl get svc -o wide
NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes   10.254.0.1     <none>        443/TCP   16h       <none>
nginx-svc    10.254.140.2   <none>        80/TCP    3m        name=nginx


# 创建一个 pods 来测试一下 nameserver

apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  containers:
  - name: alpine
    image: alpine
    command:
    - sh
    - -c
    - while true; do sleep 1; done



# 查看 pods
[root@k8s-master-1 ~]# kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
alpine                      1/1       Running   0          5s
nginx-dm-2214564181-0ctcx   1/1       Running   0          7m
nginx-dm-2214564181-brz79   1/1       Running   0          10m
nginx-dm-2214564181-z8whk   1/1       Running   0          10m



# 测试

[root@k8s-master-1 ~]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.140.2 nginx-svc.default.svc.cluster.local


[root@k8s-master-1 ~]# kubectl exec -it alpine ping nginx-svc    
PING nginx-svc (10.254.140.2): 56 data bytes

```


# 部署 Ingress 与 Dashboard



## 部署 dashboard && heapster


> 官方 dashboard 的github https://github.com/kubernetes/dashboard


## 下载 dashboard 镜像

```
# 官方镜像
gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.3

# 国内镜像
jicki/kubernetes-dashboard-amd64:v1.6.3
```


## 下载 yaml 文件

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dashboard/dashboard-controller.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dashboard/dashboard-service.yaml



# 因为开启了 RBAC 所以这里需要创建一个 RBAC 认证

vi dashboard-rbac.yaml


apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: dashboard
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

```


## 下载 heapster 镜像

```
# 官方文件
gcr.io/google_containers/heapster-amd64:v1.3.0
gcr.io/google_containers/heapster-influxdb-amd64:v1.1.1


# 本地文件
jicki/heapster-amd64:v1.3.0
jicki/heapster-influxdb-amd64:v1.1.1


```


## 下载 heapster 文件

> 官方网站 https://github.com/kubernetes/heapster/blob/master/docs/influxdb.md

```
curl -O https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/influxdb.yaml

curl -O https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/influxdb/heapster.yaml

curl -O https://raw.githubusercontent.com/kubernetes/heapster/master/deploy/kube-config/rbac/heapster-rbac.yaml


```

## 修改 influxdb 配置

```
# influxdb.yaml 记录存储data数据

# 如下:

        volumeMounts:
        - mountPath: /data
          name: influxdb-storage
      volumes:
      - name: influxdb-storage
        emptyDir: {}


# volumes 请自行修改为 共享存储 或者 本地目录

```






## 导入 yaml

```
# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *




# dashboard-controller.yaml 增加 rbac 授权


# 在第二个 spec 下面 增加

    spec:
      serviceAccountName: dashboard



# 导入文件

[root@k8s-master-1 dashboard]# kubectl apply -f .
deployment "kubernetes-dashboard" created
serviceaccount "dashboard" created
clusterrolebinding "dashboard" created
service "kubernetes-dashboard" created



[root@k8s-master-1 heapster]# kubectl apply -f .
clusterrolebinding "heapster" created
serviceaccount "heapster" created
deployment "heapster" created
service "heapster" created
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created



# 查看 svc 与 pod

[root@k8s-master-1 ~]# kubectl get svc -n kube-system
NAME                  CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
heapster              10.254.231.18    <none>        80/TCP          13s
kube-dns              10.254.0.2       <none>        53/UDP,53/TCP   1h
monitoring-influxdb   10.254.240.245   <none>        8086/TCP        13s

```



## 部署 Nginx Ingress

> Kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress； 什么是 Ingress ? Ingress 就是利用 Nginx Haproxy 等负载均衡工具来暴露 Kubernetes 服务。
>
>
> 官方 Nginx Ingress github https://github.com/kubernetes/ingress/tree/master/examples/deployment/nginx



## 配置 调度 node

```
# ingress 有多种方式 1.  deployment 自由调度 replicas
                     2.  daemonset 全局调度 分配到所有node里


#  deployment 自由调度过程中，由于我们需要 约束 controller 调度到指定的 node 中，所以需要对 node 进行 label 标签


# 默认如下:
[root@k8s-master-1 ingress]# kubectl get nodes
NAME         STATUS    AGE       VERSION
10.6.0.140   Ready     18h       v1.7.3
10.6.0.187   Ready     18h       v1.7.3
10.6.0.188   Ready     18h       v1.7.3


# 对 140 与 187 打上 label

[root@k8s-master-1 ingress]# kubectl label nodes 10.6.0.140 ingress=proxy
node "10.6.0.140" labeled
[root@k8s-master-1 ingress]# kubectl label nodes 10.6.0.187 ingress=proxy
node "10.6.0.187" labeled


# 打完标签以后

[root@k8s-master-1 ingress]# kubectl get nodes --show-labels
NAME         STATUS    AGE       VERSION   LABELS
10.6.0.140   Ready     18h       v1.7.3    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=10.6.0.140
10.6.0.187   Ready     18h       v1.7.3    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=10.6.0.187
10.6.0.188   Ready     18h       v1.7.3    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=10.6.0.188


```




```
# 下载镜像

# 官方镜像
gcr.io/google_containers/defaultbackend:1.0
gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.11

# 国内镜像
jicki/defaultbackend:1.0
jicki/nginx-ingress-controller:0.9.0-beta.11

```


```
# 部署 Nginx  backend , Nginx backend 用于统一转发 没有的域名 到指定页面。

curl -O https://raw.githubusercontent.com/kubernetes/ingress/master/examples/deployment/nginx/default-backend.yaml


# 直接导入既可, 这里不需要修改

[root@k8s-master-1 ingress]# kubectl apply -f default-backend.yaml 
deployment "default-http-backend" created
service "default-http-backend" created



# 查看服务
[root@k8s-master-1 ingress]# kubectl get deployment -n kube-system default-http-backend
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
default-http-backend   1         1         1            1           36s

```


```
# 部署 Ingress RBAC 认证

curl -O https://raw.githubusercontent.com/kubernetes/ingress/master/examples/rbac/nginx/nginx-ingress-controller-rbac.yml


# 修改 namespace

sed -i 's/namespace: nginx-ingress/namespace: kube-system/g' nginx-ingress-controller-rbac.yml


# 导入 yaml 文件

[root@k8s-master-1 ingress]# kubectl apply -f nginx-ingress-controller-rbac.yml 
namespace "nginx-ingress" created
serviceaccount "nginx-ingress-serviceaccount" created
clusterrole "nginx-ingress-clusterrole" created
role "nginx-ingress-role" created
rolebinding "nginx-ingress-role-nisa-binding" created
clusterrolebinding "nginx-ingress-clusterrole-nisa-binding" created


```




```
# 部署 Ingress Controller 组件

# 下载 yaml 文件

curl -O https://raw.githubusercontent.com/kubernetes/ingress/master/examples/deployment/nginx/nginx-ingress-controller.yaml

# 上面 对 两个 node 打了 label 所以配置 replicas: 2
# 修改 yaml 文件 增加 rbac 认证 , hostNetwork  还有 nodeSelector, 第二个 spec 下 增加。

spec:
  replicas: 2
  ....
    spec:
      hostNetwork: true
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        ingress: proxy
    ....


```


```
# 导入 yaml 文件

[root@k8s-master-1 ingress]# kubectl apply -f nginx-ingress-controller.yaml
deployment "nginx-ingress-controller" created


# 查看服务，可以看到这两个 pods 被分别调度到 140 与 187 中
[root@k8s-master-1 ingress]# kubectl get pods -n kube-system -o wide
NAME                                       READY     STATUS    RESTARTS   AGE       IP             NODE
nginx-ingress-controller-190167013-9j5kd   1/1       Running   0          45s       10.6.0.140     10.6.0.140
nginx-ingress-controller-190167013-n66qd   1/1       Running   0          45s       10.6.0.187     10.6.0.187


```

```
# 查看我们原有的 svc

[root@k8s-master-1 dashboard]# kubectl get svc -n kube-system kubernetes-dashboard
NAME                   CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   10.254.243.198   <none>        80/TCP    21s


# 创建 yaml 文件

vi dashboard-ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kube-system
spec:
  rules:
  - host: dashboard.jicki.me
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 80



# 导入 yaml

[root@k8s-master-1 dashboard]# kubectl apply -f dashboard-ingress.yaml 
ingress "dashboard-ingress" created



# 查看 ingress

[root@k8s-master-1 dashboard]# kubectl get ingress -n kube-system -o wide
NAME                HOSTS                ADDRESS                 PORTS     AGE
dashboard-ingress   dashboard.jicki.me   10.6.0.140,10.6.0.187   80        1h


# 测试访问

[root@k8s-master-1 dashboard]# curl -I dashboard.jicki.me
HTTP/1.1 200 OK
Server: nginx/1.13.2
Date: Thu, 06 Jul 2017 06:32:00 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 848
Connection: keep-alive
Accept-Ranges: bytes
Cache-Control: no-store
Last-Modified: Tue, 16 May 2017 12:53:01 GMT

```

![ Dashboard][1]


# 破坏性测试

> 破坏性测试，手动重启服务器，查看各服务间的恢复以及问题


```
[root@k8s-master-1 ~]# kubectl get nodes
NAME         STATUS    AGE       VERSION
10.6.0.140   Ready     20h       v1.7.3
10.6.0.187   Ready     20h       v1.7.3
10.6.0.188   Ready     20h       v1.7.3



[root@k8s-master-1 nginx]# etcdctl --endpoints=https://10.6.0.140:2379 \
>         --cert-file=/etc/kubernetes/ssl/etcd.pem \
>         --ca-file=/etc/kubernetes/ssl/ca.pem \
>         --key-file=/etc/kubernetes/ssl/etcd-key.pem \
>         member list
2017-08-08 02:36:38.187672 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
29262d49176888f5: name=etcd3 peerURLs=https://10.6.0.188:2380 clientURLs=https://10.6.0.188:2379 isLeader=false
d4ba1a2871bfa2b0: name=etcd1 peerURLs=https://10.6.0.140:2380 clientURLs=https://10.6.0.140:2379 isLeader=true
eca58ebdf44f63b6: name=etcd2 peerURLs=https://10.6.0.187:2380 clientURLs=https://10.6.0.187:2379 isLeader=false


[root@k8s-master-1 ~]# reboot
PolicyKit daemon disconnected from the bus.
We are no longer a registered authentication agent.

```



```
# 服务器重启

[root@k8s-master-2 ~]# kubectl get nodes
NAME         STATUS     AGE       VERSION
10.6.0.140   NotReady   20h       v1.7.3
10.6.0.187   Ready      20h       v1.7.3
10.6.0.188   Ready      20h       v1.7.3



[root@k8s-master-2 ~]# etcdctl --endpoints=https://10.6.0.187:2379 \
>         --cert-file=/etc/kubernetes/ssl/etcd.pem \
>         --ca-file=/etc/kubernetes/ssl/ca.pem \
>         --key-file=/etc/kubernetes/ssl/etcd-key.pem \
>         member list
2017-08-08 02:34:00.132653 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
29262d49176888f5: name=etcd3 peerURLs=https://10.6.0.188:2380 clientURLs=https://10.6.0.188:2379 isLeader=false
d4ba1a2871bfa2b0: name=etcd1 peerURLs=https://10.6.0.140:2380 clientURLs=https://10.6.0.140:2379 isLeader=false
eca58ebdf44f63b6: name=etcd2 peerURLs=https://10.6.0.187:2380 clientURLs=https://10.6.0.187:2379 isLeader=true



# 等待服务器重启以后

[root@k8s-master-2 ~]# kubectl get nodes
NAME         STATUS    AGE       VERSION
10.6.0.140   Ready     20h       v1.7.3
10.6.0.187   Ready     20h       v1.7.3
10.6.0.188   Ready     20h       v1.7.3


[root@k8s-master-2 ~]# kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP              NODE
alpine                      1/1       Running   1          4h        10.233.196.8    10.6.0.140
nginx-dm-2214564181-0ctcx   1/1       Running   0          4h        10.233.168.4    10.6.0.188
nginx-dm-2214564181-brz79   1/1       Running   1          4h        10.233.196.7    10.6.0.140
nginx-dm-2214564181-z8whk   1/1       Running   0          4h        10.233.182.65   10.6.0.187


[root@k8s-master-2 ~]# etcdctl --endpoints=https://10.6.0.140:2379 \
>         --cert-file=/etc/kubernetes/ssl/etcd.pem \
>         --ca-file=/etc/kubernetes/ssl/ca.pem \
>         --key-file=/etc/kubernetes/ssl/etcd-key.pem \
>         member list
2017-08-08 02:39:52.830072 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
29262d49176888f5: name=etcd3 peerURLs=https://10.6.0.188:2380 clientURLs=https://10.6.0.188:2379 isLeader=true
d4ba1a2871bfa2b0: name=etcd1 peerURLs=https://10.6.0.140:2380 clientURLs=https://10.6.0.140:2379 isLeader=false
eca58ebdf44f63b6: name=etcd2 peerURLs=https://10.6.0.187:2380 clientURLs=https://10.6.0.187:2379 isLeader=false

```


# Update Version


## Master Or Node Update 二进制文件

```
cd /tmp

wget https://dl.k8s.io/v1.7.5/kubernetes-server-linux-amd64.tar.gz

systemctl stop kube-apiserver

systemctl stop kube-controller-manager

systemctl stop kube-scheduler

systemctl stop kubelet

systemctl stop kube-proxy

tar zxvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/


systemctl start kube-apiserver

systemctl start kube-controller-manager

systemctl start kube-scheduler

systemctl start kubelet

systemctl start kube-proxy

systemctl status kube-apiserver

systemctl status kube-controller-manager

systemctl status kube-scheduler

systemctl status kubelet

systemctl status kube-proxy

cd ..

rm -rf kubernetes*

```


## Node Update 二进制文件

```

cd /tmp

wget https://dl.k8s.io/v1.7.5/kubernetes-server-linux-amd64.tar.gz

systemctl stop kubelet

systemctl stop kube-proxy

tar zxvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes


cp -r server/bin/{kubectl,kube-proxy,kubelet} /usr/local/bin/


systemctl start kubelet

systemctl start kube-proxy


systemctl status kubelet

systemctl status kube-proxy

cd ..

rm -rf kubernetes*



```



  [1]: https://jicki.me/images/posts/kubernetes/dashboard.png
  [2]: https://jicki.me/images/posts/kubernetes/hamaster.jpg



