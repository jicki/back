---
layout: post
title: Kubernetes Helm Charts
categories: kubernetes
description: Kubernetes Helm Charts
keywords: kubernetes
feature-img: "assets/img/pexels/desk-top.jpeg"
---

# Kubernetes Helm Charts

> Helm 是由 Deis 发起的一个开源工具，有助于简化部署和管理 Kubernetes 应用。
官方 github 地址 https://github.com/kubernetes/helm


## Helm 组件

> Helm 采用客户端/服务器架构 [ C/S 架构 ]

* Helm CLI 是 Helm 客户端，可以在本地执行。

* Tiller 是服务器端组件，在 Kubernetes 群集上运行，并管理 Kubernetes 应用程序的生命周期。

* Chart：一个 Helm 包，其中包含了运行一个应用所需要的镜像、依赖和资源定义等，还可能包含 Kubernetes 集群中的服务定义，类似 Homebrew 中的 formula，APT 的 dpkg 或者 Yum 的 rpm 文件。

* Repository 是 Chart 仓库，Helm客户端通过HTTP协议来访问仓库中Chart的索引文件和压缩包。


## 安装 Helm 

> 查看官方 releases 版本 https://github.com/kubernetes/helm/releases


```
# 下载文件
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.7.0-linux-amd64.tar.gz

tar zxvf helm-v2.7.0-linux-amd64.tar.gz 

mv linux-amd64/helm /usr/local/bin/helm

helm help

```

## 部署 Github Charts repository

> 官方提示 Charts 可以使用任何提供 YAML 和 tar 文件存储的 http 服务器, 并且可以回答 Get 请求, 可以自建 http 服务器, 使用 Amazon S3 存储, 或者 Github Pages . 官方文档 https://github.com/kubernetes/helm/blob/master/docs/chart_repository.md



```
# 创建  Github Pages

两种方式: 
1. 通过配置一个项目来提供其docs/目录的内容
2. 通过配置一个项目来为特定的分支提供服务

# 这里使用 docs/ 的方式。

首先登陆 github.com  创建一个 New repository

https://github.com/new

1. 创建一个 名为 charts 为名称的 仓库

2. 然后初始化一下 (git init)

# echo "# My chart to helm" > README.md
# mkdir docs
# echo "hello word!" > docs/index.html
# git init
# git add -A
# git commit -m "first commit"
# git remote add origin https://github.com/jicki520/chart.git
# git push -u origin master

3. 点击WEB UI 中你的 repo Settings

# GitHub Pages
# Source 中 选择 master branch /docs folder
# 保存以后显示如下
# Your site is ready to be published at https://jicki520.github.io/charts/.
# Enforce HTTPS

# 尝试访问  https://jicki520.github.io/charts/


4. 登陆 helm 所在服务器

git clone https://github.com/jicki520/chart.git

cd chart
helm create mychart
helm package mychart
mv mychart-0.1.0.tgz docs
helm repo index docs --url https://jicki520.github.io/chart
git add -A
git commit -m 'ADD'
git push origin master

```


```
# 添加一个 package 
# 官方 github package 地址 https://github.com/kubernetes/charts



```




## 部署 Tiller

> 安装 Tiller 到 kubernetes 集群 POD 中。

```
helm init --upgrade

# 初始化配置的时候， Helm 会去 gcr.io 中拉取 tiller 的镜像， 而且会将 "https://kubernetes-charts.storage.googleapis.com" 做为 stable repository 地址, 以后下载都会从这里拉取，国内被墙的话，很痛苦，所以我们用自己的。

# helm init -i 指定镜像下载地址，--stable-repo-url 配置缺省 stable repository 地址

helm init --upgrade -i jicki/tiller:v2.7.0 --stable-repo-url https://jicki520.github.io/charts/

# 输出如下信息

tes.oss-cn-hangzhou.aliyuncs.com/charts
Creating /root/.helm 
Creating /root/.helm/repository 
Creating /root/.helm/repository/cache 
Creating /root/.helm/repository/local 
Creating /root/.helm/plugins 
Creating /root/.helm/starters 
Creating /root/.helm/cache/archive 
Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://jicki520.github.io/charts/ 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been upgraded to the current version.
Happy Helming!

```


```
# 查看 kubernetes 服务

kubectl get po -n kube-system |grep tiller
tiller-deploy-69bc7b6f75-g9zmh                1/1       Running   0          59s

```

```
# 卸载 tiller 

helm reset

# 或者

kubectl 删除 deployment

```



