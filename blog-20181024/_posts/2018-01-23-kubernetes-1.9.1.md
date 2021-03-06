---
layout: post
title: kubernetes 1.9.1
categories: kubernetes
description: kubernetes 1.9.1
keywords: kubernetes
feature-img: "assets/img/pexels/desk-top.jpeg"
---

# kubernetes 1.9.1

> 基于 二进制 文件部署
> 本地化 kube-apiserver, kube-controller-manager , kube-scheduler
> 我这边配置  既是 master 也是 nodes


## 环境说明

> 这里配置2个Master  1个node,   Master-64 只做 Master,  Master-65 既是 Master 也是 Node,  node-66 只做单纯 Node

```
kubernetes-64: 172.16.1.64
kubernetes-65: 172.16.1.65
kubernetes-66: 172.16.1.66
```


## 初始化环境

```
hostnamectl --static set-hostname hostname

kubernetes-64: 172.16.1.64
kubernetes-65: 172.16.1.65
kubernetes-66: 172.16.1.66
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

172.16.1.64  kubernetes-64
172.16.1.65  kubernetes-65
172.16.1.66  kubernetes-66
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

```


```
# config.json 文件

vi  config.json


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

vi csr.json

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


[root@kubernetes-64 ssl]# ls -lt
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
cp *.pem /etc/kubernetes/ssl
cp ca.csr /etc/kubernetes/ssl

# 这里要将文件拷贝到所有的k8s 机器上

scp *.pem 172.16.1.65:/etc/kubernetes/ssl/
scp *.csr 172.16.1.65:/etc/kubernetes/ssl/

scp *.pem 172.16.1.66:/etc/kubernetes/ssl/
scp *.csr 172.16.1.66:/etc/kubernetes/ssl/

```



# 安装 docker

> 所有服务器预先安装 docker-ce ，官方1.9 中提示， 目前 k8s 支持最高 Docker versions 1.11.2, 1.12.6, 1.13.1, and 17.03.1

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

# 查看yum 版本

yum list docker-ce.x86_64  --showduplicates |sort -r



# 安装指定版本 docker-ce 17.03 被 docker-ce-selinux 依赖, 不能直接yum 安装 docker-ce-selinux

wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.1.ce-1.el7.centos.noarch.rpm


rpm -ivh docker-ce-selinux-17.03.1.ce-1.el7.centos.noarch.rpm


yum -y install docker-ce-17.03.1.ce


# 查看安装

docker version
Client:
 Version:      17.03.1-ce
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   f5ec1e2
 Built:        Tue Jun 27 02:21:36 2017
 OS/Arch:      linux/amd64

```
 

## 更改docker 配置


```
# 添加配置

vi /etc/systemd/system/docker.service



[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker-storage-setup.service
Wants=docker-storage-setup.service

[Service]
Type=notify
Environment=GOTRACEBACK=crash
ExecReload=/bin/kill -s HUP $MAINPID
Delegate=yes
KillMode=process
ExecStart=/usr/bin/dockerd \
          $DOCKER_OPTS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $DOCKER_DNS_OPTIONS \
          $INSECURE_REGISTRY
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=1min
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

```

```
# 修改其他配置


# 低版本内核， kernel 3.10.x  配置使用 overlay2


vi /etc/docker/daemon.json

{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}




mkdir -p /etc/systemd/system/docker.service.d/


vi /etc/systemd/system/docker.service.d/docker-options.conf

# 添加如下 :   (注意 environment 必须在同一行，如果出现换行会无法加载)


[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 \
    --data-root=/opt/docker --log-opt max-size=50m --log-opt max-file=5"




vi /etc/systemd/system/docker.service.d/docker-dns.conf


# 添加如下 : 

[Service]
Environment="DOCKER_DNS_OPTIONS=\
    --dns 10.254.0.2 --dns 114.114.114.114  \
    --dns-search default.svc.cluster.local --dns-search svc.cluster.local  \
    --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2"
    
```





```
# 重新读取配置，启动 docker 
systemctl daemon-reload
systemctl start docker
systemctl enable docker

```


```
# 如果报错 请使用
journalctl -f -t docker  和 journalctl -u docker 来定位问题

```




# etcd 集群

> etcd 是k8s集群最重要的组件， etcd 挂了，集群就挂了


## 安装 etcd

> 官方地址 https://github.com/coreos/etcd/releases

```
# 下载 二进制文件

wget https://github.com/coreos/etcd/releases/download/v3.2.14/etcd-v3.2.14-linux-amd64.tar.gz

tar zxvf etcd-v3.2.14-linux-amd64.tar.gz

cd etcd-v3.2.14-linux-amd64

mv etcd  etcdctl /usr/bin/

```

## 创建 etcd 证书

> etcd 证书这里，默认配置三个，后续如果需要增加，更多的 etcd 节点
> 这里的认证IP 请多预留几个，以备后续添加能通过认证，不需要重新签发

```
cd /opt/ssl/

vi etcd-csr.json

{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.16.1.64",
    "172.16.1.65",
    "172.16.1.66"
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

[root@kubernetes-64 ssl]# ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem



# 拷贝到etcd服务器

# etcd-1 
cp etcd*.pem /etc/kubernetes/ssl/

# etcd-2
scp etcd*.pem 172.16.1.65:/etc/kubernetes/ssl/

# etcd-3
scp etcd*.pem 172.16.1.66:/etc/kubernetes/ssl/



# 如果 etcd 非 root 用户，读取证书会提示没权限

chmod 644 /etc/kubernetes/ssl/etcd-key.pem

```



## 修改 etcd 配置

> 由于 etcd 是最重要的组件，所以 --data-dir 请配置到其他路径中


```
# 创建 etcd data 目录， 并授权

useradd etcd

mkdir -p /opt/etcd

chown -R etcd:etcd /opt/etcd


```

```
# etcd-1


vi /etc/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
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
  --initial-advertise-peer-urls=https://172.16.1.64:2380 \
  --listen-peer-urls=https://172.16.1.64:2380 \
  --listen-client-urls=https://172.16.1.64:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.16.1.64:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://172.16.1.64:2380,etcd2=https://172.16.1.65:2380,etcd3=https://172.16.1.66:2380 \
  --initial-cluster-state=new \
  --data-dir=/opt/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

``` 

```
# etcd-2


vi /etc/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
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
  --initial-advertise-peer-urls=https://172.16.1.65:2380 \
  --listen-peer-urls=https://172.16.1.65:2380 \
  --listen-client-urls=https://172.16.1.65:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.16.1.65:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://172.16.1.64:2380,etcd2=https://172.16.1.65:2380,etcd3=https://172.16.1.66:2380 \
  --initial-cluster-state=new \
  --data-dir=/opt/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

``` 

```
# etcd-3


vi /etc/systemd/system/etcd.service


[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/opt/etcd/
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
  --initial-advertise-peer-urls=https://172.16.1.66:2380 \
  --listen-peer-urls=https://172.16.1.66:2380 \
  --listen-client-urls=https://172.16.1.66:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://172.16.1.66:2379 \
  --initial-cluster-token=k8s-etcd-cluster \
  --initial-cluster=etcd1=https://172.16.1.64:2380,etcd2=https://172.16.1.65:2380,etcd3=https://172.16.1.66:2380 \
  --initial-cluster-state=new \
  --data-dir=/opt/etcd/
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

``` 



## 启动 etcd

> 分别启动 所有节点的 etcd 服务

```
systemctl daemon-reload
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
etcdctl --endpoints=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        cluster-health

member 35eefb8e7cc93b53 is healthy: got healthy result from https://172.16.1.66:2379
member 4576ff5ed626a66b is healthy: got healthy result from https://172.16.1.64:2379
member bf3bd651ec832339 is healthy: got healthy result from https://172.16.1.65:2379
cluster is healthy
```


> 查看 etcd 集群成员：

```
etcdctl --endpoints=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        member list


35eefb8e7cc93b53: name=etcd3 peerURLs=https://172.16.1.66:2380 clientURLs=https://172.16.1.66:2379 isLeader=false
4576ff5ed626a66b: name=etcd1 peerURLs=https://172.16.1.64:2380 clientURLs=https://172.16.1.64:2379 isLeader=true
bf3bd651ec832339: name=etcd2 peerURLs=https://172.16.1.65:2380 clientURLs=https://172.16.1.65:2379 isLeader=false

```

# 配置 Kubernetes 集群

> kubectl 安装在所有需要进行操作的机器上


## Master and Node 

> Master 需要部署 kube-apiserver , kube-scheduler , kube-controller-manager 这三个组件。
kube-scheduler 作用是调度pods分配到那个node里，简单来说就是资源调度。
kube-controller-manager 作用是 对 deployment controller , replication controller, endpoints controller, namespace controller, and serviceaccounts controller等等的循环控制，与kube-apiserver交互。


### 安装组件

```
# 从github 上下载版本

cd /tmp

wget https://dl.k8s.io/v1.9.1/kubernetes-server-linux-amd64.tar.gz

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl} /usr/local/bin/


scp server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} 172.16.1.65:/usr/local/bin/


scp server/bin/{kube-proxy,kubelet} 172.16.1.66:/usr/local/bin/

```


### 创建 admin 证书

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
  -config=/opt/ssl/config.json \
  -profile=kubernetes admin-csr.json | /opt/local/cfssl/cfssljson -bare admin


# 查看生成

[root@kubernetes-64 ssl]# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem

cp admin*.pem /etc/kubernetes/ssl/

scp admin*.pem 172.16.1.65:/etc/kubernetes/ssl/

```


### 配置 kubectl kubeconfig 文件


>  生成证书相关的配置文件存储与 /root/.kube 目录中

```
# 配置 kubernetes 集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443


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




### 创建 kubernetes 证书

```
cd /opt/ssl

vi kubernetes-csr.json

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.16.1.64",
    "172.16.1.65",
    "172.16.1.66",
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


## 这里 hosts 字段中 三个 IP 分别为 127.0.0.1 本机， 172.16.1.64 和 172.16.1.65 为 Master 的IP，多个Master需要写多个。  10.254.0.1 为 kubernetes SVC 的 IP， 一般是 部署网络的第一个IP , 如: 10.254.0.1 ， 在启动完成后，我们使用   kubectl get svc ， 就可以查看到

```


### 生成 kubernetes 证书和私钥


```
/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes kubernetes-csr.json | /opt/local/cfssl/cfssljson -bare kubernetes

# 查看生成

[root@kubernetes-64 ssl]# ls -lt kubernetes*
-rw-r--r-- 1 root root 1261 11月 16 15:12 kubernetes.csr
-rw------- 1 root root 1679 11月 16 15:12 kubernetes-key.pem
-rw-r--r-- 1 root root 1635 11月 16 15:12 kubernetes.pem
-rw-r--r-- 1 root root  475 11月 16 15:12 kubernetes-csr.json


# 拷贝到目录
cp kubernetes*.pem /etc/kubernetes/ssl/

scp kubernetes*.pem 172.16.1.65:/etc/kubernetes/ssl/

```

### 配置 kube-apiserver

> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token  一致，如果一致则自动为 kubelet生成证书和秘钥。


```
# 生成 token

[root@kubernetes-64 ssl]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
df3b158fbdc425ae2ac70bbef0688921


# 创建 token.csv 文件

cd /opt/ssl

vi token.csv

df3b158fbdc425ae2ac70bbef0688921,kubelet-bootstrap,10001,"system:kubelet-bootstrap"


# 拷贝

cp token.csv /etc/kubernetes/

scp token.csv 172.16.1.65:/etc/kubernetes/

```


```
# 生成高级审核配置文件

cd /etc/kubernetes


cat >> audit-policy.yaml <<EOF
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF




# 拷贝

scp audit-policy.yaml 172.16.1.65:/etc/kubernetes/

```



### 创建 kube-apiserver.service 文件

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
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --advertise-address=172.16.1.64 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --secure-port=6443 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
  --etcd-servers=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=127.0.0.1 \
  --insecure-port=8080 \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.254.0.0/18 \
  --service-node-port-range=30000-32000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --enable-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/token.csv \
  --v=1
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```



```
# k8s 1.8 开始需要 添加 --authorization-mode=Node
# k8s 1.8 开始需要 添加 --admission-control=NodeRestriction
# k8s 1.8 开始需要 添加 --audit-policy-file=/etc/kubernetes/audit-policy.yaml

# 这里面要注意的是 --service-node-port-range=30000-32000
# 这个地方是 映射外部端口时 的端口范围，随机映射也在这个范围内映射，指定映射端口必须也在这个范围内。
```

 
### 启动 kube-apiserver

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver

```


### 配置 kube-controller-manager


```
# 创建 kube-controller-manager.service 文件

vi /etc/systemd/system/kube-controller-manager.service


[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.254.0.0/18 \
  --cluster-cidr=10.254.64.0/18 \
  --cluster-name=kubernetes \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --leader-elect=true \
  --v=1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target


```



### 启动 kube-controller-manager

```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```


### 配置 kube-scheduler


```
# 创建 kube-cheduler.service 文件

vi /etc/systemd/system/kube-scheduler.service


[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --address=0.0.0.0 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```

### 启动 kube-scheduler

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler

```
 

### 验证 Master 节点

```
[root@kubernetes-64 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-2               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 



[root@kubernetes-65 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-2               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}  


```
 

### 配置 kubelet

> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求(certificatesigningrequests)。

```

# 先创建认证请求
# user 为 master 中 token.csv 文件里配置的用户
# 只需创建一次就可以

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

```

### 创建 kubelet kubeconfig 文件


```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=df3b158fbdc425ae2ac70bbef0688921 \
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


### 创建 kubelet.service 文件


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
  --cgroup-driver=cgroupfs \
  --hostname-override=kubernetes-64 \
  --pod-infra-container-image=jicki/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.254.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --fail-swap-on=false \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --v=1

[Install]
WantedBy=multi-user.target

```


```
# 如上配置:
kubernetes-64    本机hostname
10.254.0.2       预分配的 dns 地址
cluster.local.   为 kubernetes 集群的 domain
jicki/pause-amd64:3.0  这个是 pod 的基础镜像，既 gcr 的 gcr.io/google_containers/pause-amd64:3.0 镜像， 下载下来修改为自己的仓库中的比较快。
```

### 启动 kubelet

```

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet

```

```
# 如果报错 请使用
journalctl -f -t kubelet  和 journalctl -u kubelet 来定位问题

```


### 配置 TLS 认证

```
# 查看 csr 的名称

[root@kubernetes-64 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-Pu4QYp3NAwlC6o8AG8iwdCl52CiqhjiSyrso3335JTs   1m        kubelet-bootstrap   Pending
node-csr-poycCHd7B8YPxc12EBgI3Rwe0wnDJah5uIGvQHzghVY   2m        kubelet-bootstrap   Pending


# 增加 认证

kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

```



### 验证 nodes

```
[root@kubernetes-64 ~]# kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
kubernetes-64   Ready     <none>    12s       v1.9.1
kubernetes   Ready     <none>    17s       v1.9.1


# 成功以后会自动生成配置文件与密钥

# 配置文件

ls /etc/kubernetes/kubelet.kubeconfig   
/etc/kubernetes/kubelet.kubeconfig


# 密钥文件  这里注意如果 csr 被删除了，请删除如下文件，并重启 kubelet 服务

ls /etc/kubernetes/ssl/kubelet*
/etc/kubernetes/ssl/kubelet-client.crt  /etc/kubernetes/ssl/kubelet.crt
/etc/kubernetes/ssl/kubelet-client.key  /etc/kubernetes/ssl/kubelet.key

```


## 配置 kube-proxy


### 创建 kube-proxy 证书

```
# 证书方面由于我们node端没有装 cfssl
# 我们回到 master 端 机器 去配置证书，然后拷贝过来

[root@kubernetes-64 ~]# cd /opt/ssl


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
  -config=/opt/ssl/config.json \
  -profile=kubernetes  kube-proxy-csr.json | /opt/local/cfssl/cfssljson -bare kube-proxy
  
# 查看生成
ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem

# 拷贝到目录

cp kube-proxy* /etc/kubernetes/ssl/

scp kube-proxy* 172.16.1.65:/etc/kubernetes/ssl/

scp kube-proxy* 172.16.1.66:/etc/kubernetes/ssl/
```


### 创建 kube-proxy kubeconfig 文件

```
# 配置集群

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

# 拷贝到需要的 node 端里

scp kube-proxy.kubeconfig 172.16.1.65:/etc/kubernetes/

scp kube-proxy.kubeconfig 172.16.1.66:/etc/kubernetes/
```


### 创建 kube-proxy.service 文件

>  1.9  官方 ipvs 已经 beta , 尝试开启 ipvs 测试一下, 官方 --feature-gates=SupportIPVSProxyMode=false 默认是 false 的， 需要打开
--feature-gates=SupportIPVSProxyMode=true
> --masquerade-all 必须添加这项配置，否则 创建 svc 在 ipvs 不会添加规则

> 打开 ipvs 需要安装 ipvsadm 软件， 在 node 中安装  
> yum install ipvsadm -y

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
  --bind-address=172.16.1.65 \
  --hostname-override=kubernetes-65 \
  --cluster-cidr=10.254.64.0/18 \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=1
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```


### 启动 kube-proxy

```

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy

```

```
# 检查  ipvs

[root@kubernetes-65 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr persistent 10800
  -> 172.16.1.64:6443             Masq    1      0          0         
  -> 172.16.1.65:6443             Masq    1      0          0  

```



```
# 如果报错 请使用
journalctl -f -t kube-proxy  和 journalctl -u kube-proxy 来定位问题

```

#### 至此 Master 端 与 Master and Node 端的安装完毕




##  Node 端 

> 单 Node 部分 需要部署的组件有 docker  calico  kubelet  kube-proxy 这几个组件。
Node 节点 基于 Nginx 负载 API 做 Master HA


```
# master 之间除 api server 以外其他组件通过 etcd 选举，api server 默认不作处理；在每个 node 上启动一个 nginx，每个 nginx 反向代理所有 api server，node 上 kubelet、kube-proxy 连接本地的 nginx 代理端口，当 nginx 发现无法连接后端时会自动踢掉出问题的 api server，从而实现 api server 的 HA
```


![ HAMaster][2]


### 发布证书


```
# ALL node

mkdir -p /etc/kubernetes/ssl/

scp ca.pem kube-proxy.pem kube-proxy-key.pem  node-*:/etc/kubernetes/ssl/

```


### 创建Nginx 代理

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
        server 172.16.1.64:6443;
        server 172.16.1.65:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

# 更新权限
chmod +r /etc/nginx/nginx.conf

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
ExecStart=/usr/bin/docker run -p 127.0.0.1:6443:6443 \\
                              -v /etc/nginx:/etc/nginx \\
                              --name nginx-proxy \\
                              --net=host \\
                              --restart=on-failure:5 \\
                              --memory=512M \\
                              nginx:1.13.7-alpine
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
systemctl status nginx-proxy

```

### 配置 Kubelet.service 文件


```
# 创建目录

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
  --cgroup-driver=cgroupfs \
  --hostname-override=kubernetes-66 \
  --pod-infra-container-image=jicki/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.254.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --fail-swap-on=false \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --v=1

[Install]
WantedBy=multi-user.target

```


```
# 启动 kubelet 与 kube-proxy

systemctl daemon-reload
systemctl start kubelet
systemctl status kubelet
```



### 配置 kube-proxy.service


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
  --bind-address=172.16.1.66 \
  --hostname-override=kubernetes-66 \
  --cluster-cidr=10.254.64.0/18 \
  --masquerade-all \
  --feature-gates=SupportIPVSProxyMode=true \
  --proxy-mode=ipvs \
  --ipvs-min-sync-period=5s \
  --ipvs-sync-period=5s \
  --ipvs-scheduler=rr \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


```


```
#  启动

systemctl start kube-proxy
systemctl status kube-proxy
```



### Master 配置 TLS 认证

```
# 查看 csr 的名称

[root@kubernetes-64 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-Fla2k7UdFJrN9d8Jjlw588zu0ESUycXHcir1f7bAh5U   14s       kubelet-bootstrap   Pending
node-csr-Pu4QYp3NAwlC6o8AG8iwdCl52CiqhjiSyrso3335JTs   19m       kubelet-bootstrap   Approved,Issued
node-csr-poycCHd7B8YPxc12EBgI3Rwe0wnDJah5uIGvQHzghVY   20m       kubelet-bootstrap   Approved,Issued



# 增加 认证

[root@kubernetes-64 ~]# kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

```

```
[root@kubernetes-64 ~]# kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
kubernetes-64   Ready     <none>    17m       v1.9.1
kubernetes   Ready     <none>    18m       v1.9.1
kubernetes-66   Ready     <none>    26s       v1.9.1

```


### 限制 POD 的调度

> 由于 master-64 只做 master 不做 pod 调度，所以禁止调度到 master-64中,  Pod 的调度是通过 kubelet 服务来启动的，但是不启动 kubelet 的话，节点在 node 里是不可见的。

```
[root@kubernetes-64 ~]# kubectl cordon  kubernetes-64
node "kubernetes-64" cordoned


[root@kubernetes-64 ~]# kubectl get nodes            
NAME            STATUS                     ROLES     AGE       VERSION
kubernetes-64   Ready,SchedulingDisabled   <none>    19m       v1.9.1
kubernetes   Ready                      <none>    19m       v1.9.1
kubernetes-66   Ready                      <none>    1m        v1.9.1

```



# 配置 Flannel 网络

> flannel 网络只部署在 kube-proxy 相关机器

> 个人 百度盘 下载 https://pan.baidu.com/s/1eStojia

```
rpm -ivh flannel-0.9.1-1.x86_64.rpm

```

```
# 配置 flannel

# 由于我们docker更改了 docker.service.d 的路径

# 所以这里把 flannel.conf 的配置拷贝到 这个目录去

mv /usr/lib/systemd/system/docker.service.d/flannel.conf /etc/systemd/system/docker.service.d

```

```
# 配置 flannel 网段

etcdctl --endpoints=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        set /flannel/network/config \ '{"Network":"10.254.64.0/18","SubnetLen":24,"Backend":{"Type":"host-gw"}}'

```

```
# 修改 flanneld 配置

vi /etc/sysconfig/flanneld


# Flanneld configuration options  

# etcd 地址

FLANNEL_ETCD_ENDPOINTS="https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379"

# 配置为上面的路径 flannel/network
FLANNEL_ETCD_PREFIX="/flannel/network"

# 其他的配置，可查看 flanneld --help,这里添加了 etcd ssl 认证
FLANNEL_OPTIONS="-ip-masq=true -etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/etcd.pem -etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem -iface=em1"

```


```
# 启动 flannel 

systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld

```


```

# 如果报错 请使用
journalctl -f -t flanneld  和 journalctl -u flanneld 来定位问题

```


```
# 配置完毕，重启 docker

systemctl daemon-reload
systemctl restart docker
systemctl status docker


```

```
# 重启 kubelet

systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet

```



```
# 验证 网络

ifconfig  查看  docker0 网络 是否已经更改为配置IP网段 


```




## 测试集群


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
[root@kubernetes-64 ~]# kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP            NODE
nginx-dm-84f8f49555-dzpm9   1/1       Running   0          6s        10.254.90.2   kubernetes-65
nginx-dm-84f8f49555-qbnvv   1/1       Running   0          6s        10.254.66.2   k8s-master-66



[root@kubernetes-64 ~]# kubectl get svc -o wide    
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP   2h        <none>
nginx-svc    ClusterIP   10.254.41.39   <none>        80/TCP    1m


```


```
# 在 安装了 Flannel 网络的节点 里 curl

[root@kubernetes-64 ~]# curl 10.254.51.137
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


```
# 查看 ipvs 规则

[root@kubernetes-65 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr persistent 10800
  -> 172.16.1.64:6443             Masq    1      0          0         
  -> 172.16.1.65:6443             Masq    1      0          0         
TCP  10.254.41.39:80 rr
  -> 10.254.66.2:80               Masq    1      0          0         
  -> 10.254.90.2:80               Masq    1      0          1  

```



# 配置 CoreDNS

> 官方 地址  https://coredns.io


## 下载镜像

```
# 官方镜像
coredns/coredns:1.0.4


# 我的镜像
jicki/coredns:1.0.4

```



## 下载 yaml 文件

```
wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed

mv coredns.yaml.sed coredns.yaml


```


```
# vi coredns.yaml

...
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local 10.254.0.0/18 {
          pods insecure
          upstream /etc/resolv.conf
          fallthrough in-addr.arpa ip6.arpa
        }
...        
        image: jicki/coredns:1.0.4
...        
  clusterIP: 10.254.0.2
  

```


```
# 配置说明 


# 这里 kubernetes cluster.local 为 创建 svc 的 IP 段

kubernetes cluster.local 10.254.0.0/18 

# clusterIP  为 指定 DNS 的 IP

clusterIP: 10.254.0.2

```






## 导入 yaml 文件

```
# 导入

[root@kubernetes-64 coredns]# kubectl apply -f coredns.yaml 
serviceaccount "coredns" created
clusterrole "system:coredns" created
clusterrolebinding "system:coredns" created
configmap "coredns" created
deployment "coredns" created
service "coredns" created

```

## 查看 kubedns 服务

```
[root@kubernetes-64 coredns]# kubectl get pod,svc -n kube-system
NAME                          READY     STATUS    RESTARTS   AGE
po/coredns-6bd7d5dbb5-jh4fj   1/1       Running   0          19s

NAME          TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
svc/coredns   ClusterIP   10.254.0.2   <none>        53/UDP,53/TCP   19s
```

## 检查日志

```
[root@kubernetes-64 coredns]# kubectl logs -n kube-system coredns-6bd7d5dbb5-jh4fj

.:53
CoreDNS-1.0.1
linux/amd64, go1.9.2, 99e163c3
2017/12/20 09:34:24 [INFO] CoreDNS-1.0.1
2017/12/20 09:34:24 [INFO] linux/amd64, go1.9.2, 99e163c3
```



## 验证 dns 服务

> 在验证 dns 之前，在 dns 未部署之前创建的 pod 与 deployment 等，都必须删除，重新部署，否则无法解析


```

# 创建一个 pods 来测试一下 dns 

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



# 查看 创建的服务

[root@kubernetes-64 yaml]# kubectl get pods,svc 
NAME                           READY     STATUS    RESTARTS   AGE
po/alpine                      1/1       Running   0          19s
po/nginx-dm-84f8f49555-tmqzm   1/1       Running   0          23s
po/nginx-dm-84f8f49555-wdk67   1/1       Running   0          23s

NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
svc/kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP   5h
svc/nginx-svc    ClusterIP   10.254.40.179   <none>        80/TCP    23s



# 测试

[root@kubernetes-64 ~]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.40.179 nginx-svc.default.svc.cluster.local


[root@kubernetes-64 yaml]# kubectl exec -it alpine nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local

```


# 部署 Ingress 与 Dashboard



## 部署 dashboard && heapster


> 官方 dashboard 的github https://github.com/kubernetes/dashboard


## 下载 dashboard 镜像

```
# 官方镜像
k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.2

# 国内镜像
jicki/kubernetes-dashboard-amd64:v1.8.2
```


## 下载 yaml 文件

```
curl -O https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml

```


## 导入 yaml

```
# 替换所有的 images

sed -i 's/k8s\.gcr\.io/jicki/g' *


# 导入文件

[root@kubernetes-64 dashboard]# kubectl apply -f kubernetes-dashboard.yaml 
secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created




[root@kubernetes-64 ~]# kubectl get pods,svc -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
po/coredns-5984fb8cbb-77dl4                1/1       Running   0          3h
po/coredns-5984fb8cbb-9hdwt                1/1       Running   0          3h
po/kubernetes-dashboard-78bcdc4d64-x6fhq   1/1       Running   0          14s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
svc/kube-dns               ClusterIP   10.254.0.2      <none>        53/UDP,53/TCP   3h
svc/kubernetes-dashboard   ClusterIP   10.254.18.143   <none>        443/TCP         14s

```



## 部署 Nginx Ingress

> Kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress； 什么是 Ingress ? Ingress 就是利用 Nginx Haproxy 等负载均衡工具来暴露 Kubernetes 服务。
>
>
> 官方 Nginx Ingress github:  https://github.com/kubernetes/ingress-nginx/



## 配置 调度 node

```
# ingress 有多种方式 1.  deployment 自由调度 replicas
                     2.  daemonset 全局调度 分配到所有node里


#  deployment 自由调度过程中，由于我们需要 约束 controller 调度到指定的 node 中，所以需要对 node 进行 label 标签


# 默认如下:
[root@kubernetes-64 ingress]# kubectl get nodes
NAME            STATUS                     ROLES     AGE       VERSION
kubernetes-64   Ready,SchedulingDisabled   <none>    10d       v1.9.1
kubernetes-65   Ready                      <none>    10d       v1.9.1
kubernetes-66   Ready                      <none>    10d       v1.9.1


# 对 65 与 66 打上 label

[root@kubernetes-64 ingress]# kubectl label nodes kubernetes-65 ingress=proxy
node "kubernetes-65" labeled
[root@kubernetes-64 ingress]# kubectl label nodes kubernetes-66 ingress=proxy
node "kubernetes-66" labeled


# 打完标签以后

[root@kubernetes-64 ingress]# kubectl get nodes --show-labels
NAME            STATUS                     ROLES     AGE       VERSION   LABELS
kubernetes-64   Ready,SchedulingDisabled   <none>    10d       v1.9.1    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=kubernetes-64
kubernetes-65   Ready                      <none>    10d       v1.9.1    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=kubernetes-65
kubernetes-66   Ready                      <none>    10d       v1.9.1    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=kubernetes-66

```




```
# 下载镜像

# 官方镜像
gcr.io/google_containers/defaultbackend:1.4
quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.10.0

# 国内镜像
jicki/defaultbackend:1.4
jicki/nginx-ingress-controller:0.10.0

```


```
# 下载 yaml 文件

# 部署 Nginx  backend , Nginx backend 用于统一转发 没有的域名 到指定页面。

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml


# 部署 Ingress RBAC 认证

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml


# 部署 Ingress Controller 组件


curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml


# tcp-service 与 udp-service, 由于 ingress 不支持 tcp 与 udp 的转发，所以这里配置了两个基于 tcp 与 udp 的 service ,通过 --tcp-services-configmap 与 --udp-services-configmap 来配置 tcp 与 udp 的转发服务


# tcp 例子

apiVersion: v1
kind: ConfigMap
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  9000: "default/tomcat:8080"
  
#  以上配置， 转发 tomcat:8080 端口 到 ingress 节点的 9000 端口中

  
# udp 例子

apiVersion: v1
kind: ConfigMap
metadata:
  name: udp-services
  namespace: ingress-nginx
data:
  53: "kube-system/kube-dns:53"
```  



```
# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *
sed -i 's/quay\.io\/kubernetes-ingress-controller/jicki/g' *


# 上面 对 两个 node 打了 label 所以配置 replicas: 2
# 修改 yaml 文件 增加 rbac 认证 , hostNetwork  还有 nodeSelector, 第二个 spec 下 增加。

vi with-rbac.yaml



spec:
  replicas: 2
  ....
    spec:
      serviceAccountName: nginx-ingress-serviceaccount
      hostNetwork: true
      nodeSelector:
        ingress: proxy
    ....
          # 这里添加一个 other 端口做为后续tcp转发
          ports:
          - name: http
            containerPort: 80
          - name: https
            containerPort: 443
          - name: other
            containerPort: 8888


```


```
# 导入 yaml 文件

[root@kubernetes-64 ingress]# kubectl apply -f namespace.yaml 
namespace "ingress-nginx" created


[root@kubernetes-64 nginx-ingress]# kubectl apply -f .
configmap "nginx-configuration" created
deployment "default-http-backend" created
service "default-http-backend" created
namespace "ingress-nginx" configured
serviceaccount "nginx-ingress-serviceaccount" created
clusterrole "nginx-ingress-clusterrole" created
role "nginx-ingress-role" created
rolebinding "nginx-ingress-role-nisa-binding" created
clusterrolebinding "nginx-ingress-clusterrole-nisa-binding" created
configmap "tcp-services" created
configmap "udp-services" created
deployment "nginx-ingress-controller" created



# 查看服务，可以看到这两个 pods 被分别调度到 65 与 66 中
[root@kubernetes-64 ingress]# kubectl get pods -n ingress-nginx -o wide
NAME                                        READY     STATUS    RESTARTS   AGE       IP             NODE
default-http-backend-76f7d74455-kxbr2       1/1       Running   0          5m        10.254.126.4   kubernetes-65
nginx-ingress-controller-8476958f94-8fh5h   1/1       Running   0          5m        172.16.1.66    kubernetes-66
nginx-ingress-controller-8476958f94-qfhhp   1/1       Running   0          5m        172.16.1.65    kubernetes-65
```


```
# 查看我们原有的 svc

[root@kubernetes-64 ingress]# kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
alpine                      1/1       Running   0          24m
nginx-dm-84f8f49555-tmqzm   1/1       Running   0          24m
nginx-dm-84f8f49555-wdk67   1/1       Running   0          24m


```


```
# 创建一个 基于 nginx-dm 的 ingress

vi nginx-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.jicki.me
    http:
      paths:
      - backend:
          serviceName: nginx-svc
          servicePort: 80



# 查看服务

[root@kubernetes-64 ingress]# kubectl get ingress
NAME            HOSTS            ADDRESS   PORTS     AGE
nginx-ingress   nginx.jicki.me             80        6s
```

```
# 测试访问

[root@kubernetes-64 ingress]# curl nginx.jicki.me
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



```
# 创建一个基于 dashboard 的 https 的 ingress
# 新版本的 dashboard 默认就是 ssl ,所以这里使用 tcp 代理到 443 端口


# 查看 dashboard svc

[root@kubernetes-64 dashboard]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.254.0.2      <none>        53/UDP,53/TCP   4h
kubernetes-dashboard   ClusterIP   10.254.18.143   <none>        443/TCP         57m



# 修改 tcp-services-configmap.yaml 文件

vi tcp-services-configmap.yaml


kind: ConfigMap
apiVersion: v1
metadata:
  name: tcp-services
  namespace: ingress-nginx
data:
  8888: "kube-system/kubernetes-dashboard:443"



# 导入文件

[root@kubernetes-64 dashboard]# kubectl apply -f tcp-services-configmap.yaml 
configmap "tcp-services" created


# 查看服务

[root@kubernetes-64 dashboard]# kubectl get configmap/tcp-services -n ingress-nginx
NAME           DATA      AGE
tcp-services   1         11m



[root@kubernetes-64 dashboard]# kubectl describe configmap/tcp-services -n ingress-nginx
Name:         tcp-services
Namespace:    ingress-nginx
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","data":{"8888":"kube-system/kubernetes-dashboard:443"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"tcp-services","namesp...

Data
====
8888:
----
kube-system/kubernetes-dashboard:443
Events:  <none>




# 测试访问

[root@kubernetes-64 dashboard]# curl -I -k https://dashboard.jicki.me:8888

HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: no-store
Content-Length: 990
Content-Type: text/html; charset=utf-8
Last-Modified: Mon, 15 Jan 2018 13:10:36 GMT
Date: Tue, 23 Jan 2018 09:12:08 GMT


```

```
# 登录认证

# 首先创建一个 dashboard rbac 超级用户

vi dashboard-admin-rbac.yaml

---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kube-system


# 导入文件

[root@kubernetes-64 dashboard]# kubectl apply -f dashboard-admin-rbac.yaml 
serviceaccount "kubernetes-dashboard-admin" created
clusterrolebinding "kubernetes-dashboard-admin" created



# 查看超级用户的 token 名称

[root@kubernetes-64 dashboard]# kubectl -n kube-system get secret | grep kubernetes-dashboard-admin
kubernetes-dashboard-admin-token-mnhdz   kubernetes.io/service-account-token   3         1m


# 查看 token 部分

[root@kubernetes-64 dashboard]# kubectl describe -n kube-system secret/kubernetes-dashboard-admin-token-mnhdz
Name:         kubernetes-dashboard-admin-token-mnhdz
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=kubernetes-dashboard-admin
              kubernetes.io/service-account.uid=dc14511d-0020-11e8-b47b-44a8420b9988

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1363 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1tbmhkeiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImRjMTQ1MTFkLTAwMjAtMTFlOC1iNDdiLTQ0YTg0MjBiOTk4OCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.Vg7vYBIaBICYFCX_XORvoUjkYAKdQoAuT2sy8o4y8Z6DmMaCQXijOBGCWsS40-n_qiBhlrSwLeN0RvjCOfLmcH4gUSjPBkSmc-S6SHh09ErzrHjCQSblCCZgXjyyse2w1LwWw87CiAiwHCb0Jm7r0lhm4DjhXeLpUhdXoqOltHlBoJqxzDwb9qKgtY-nsQ2Y9dhV405GeqB9RLOxSKHWx6K1lXP_0tLUGgIatJx6f-EMurFbmODJfex9mT2LTq9pblblegw9EG9j2IhfHQSnwR8hPMT3Tku-XEf3vtV-1eFqetZHRJHS23machhvSvuppFjmPAd_ID3eETBt7ncNmQ



# 登录 web ui 选择 令牌登录

```
![ dashboard ][3]








# k8s 运维相关


## 基础维护

```
# 当需要对主机进行维护升级时，首先将节点主机设置成不可调度模式： 

kubectl cordon［nodeid］  

# 然后需要将主机上正在运行的容器驱赶到其它可用节点： 
 
kubectl drain ［nodeid］

# 给予900秒宽限期优雅的调度
kubectl drain node1.k8s.novalocal --grace-period=120


# 当容器迁移完毕后，运维人员可以对该主机进行操作，配置升级性能参数调优等等。当对主机的维护操作完毕后， 再将主机设置成可调度模式： 

kubectl uncordon [nodeid]  

```




# Other



## 特殊 env


```
# yaml 中的一些 特殊 env


    env:
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: MY_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP



```


  [1]: https://jicki.me/images/posts/kubernetes/dashboard.png
  [2]: https://jicki.me/images/posts/kubernetes/hamaster.jpg
  [3]: https://jicki.me/images/posts/kubernetes/dashboard-new.jpeg

