---
title: weave scope的安装配置以及traefik ingress basic auth的实现
author: 阿辉
date: 2018-12-28T10:47:56+00:00
categories:
- Kubernetes
- traefik
tags:
- Kubernetes
- traefik
- Weave Scope
keywords:
- Kubernetes
- traefik
- Weave Scope
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---

{{< toc >}}

Weave Scope 能自动生成应用程序的映射，使你能够直观地了解、监控并控制你的微服务容器应用。

## 1. Weave Scope的简介
### 1.1. 实时了解Docker容器状态
查看容器基础设施的概况，或者专注于一个特殊的微服务。从而轻松发现并纠正问题，确保你的容器化应用的稳定与性能。
![](/wp-content/uploads/2018/12/7885921b966f07bd293625e9cb36f359.png)


### 1.2. 内部细节与深度链接
查看容器的指标、标签和元数据。在一个可扩展、可排序的列表内，从容器内的进程到容器运行的主机之间轻松切换。对于指定的主机或者服务，很容易找高负载（CPU或内存）的容器。
![](/wp-content/uploads/2018/12/a812a46b4ee711a838d397591750153e.png)

### 1.3. 容器的交互与管理
直接与容器交互：暂停、重启或者停止容器，以及启动命令行。这些都在Scope的浏览器内进行。
![](/wp-content/uploads/2018/12/109f98aa669d33b2a6fb870a00f02e44.png)

<!--more-->

## 2. Weave Scope的安装
通过以下命令安装Weave Scope：
`kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"`
运行后会发现已经起来几个pod:

```bash
[root@sh-saas-k8s1-master-dev-01 ~]# kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"

namespace "weave" created
serviceaccount "weave-scope" created
clusterrole.rbac.authorization.k8s.io "weave-scope" created
clusterrolebinding.rbac.authorization.k8s.io "weave-scope" created
deployment.apps "weave-scope-app" created
service "weave-scope-app" created
daemonset.extensions "weave-scope-agent" created
[root@sh-saas-k8s1-master-dev-01 ~]# 
[root@sh-saas-k8s1-master-dev-01 ~]# kubectl get pod -n weave  -o wide
NAME                               READY     STATUS    RESTARTS   AGE       IP             NODE
weave-scope-agent-8z2st            1/1       Running   0          31s       10.12.97.11    10.12.97.11
weave-scope-agent-f94sh            1/1       Running   0          31s       10.12.97.23    10.12.97.23
weave-scope-agent-h8vmh            1/1       Running   0          31s       10.12.97.12    10.12.97.12
weave-scope-agent-kqrm6            1/1       Running   0          31s       10.12.97.13    10.12.97.13
weave-scope-agent-pqgt4            1/1       Running   0          31s       10.12.97.21    10.12.97.21
weave-scope-agent-szrh4            1/1       Running   0          31s       10.12.97.22    10.12.97.22
weave-scope-app-5bdf799d4b-mjbjq   1/1       Running   0          32s       10.253.3.176   10.12.97.21
```
这样一条命令就可以完成Weave Scope的安装，是不是很简单？

## 3. Weave Scope的配置
Weave Scope安装好了，怎么访问呢？

我们可以看下，安装时已经创建了k8s service：
```bash
[root@sh-saas-k8s1-master-dev-01 dev]# kubectl get service -n weave -o wide
NAME              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE       SELECTOR
weave-scope-app   ClusterIP   10.253.255.240   <none>        80/TCP    1h        app=weave-scope,name=weave-scope-app,weave-cloud-component=scope,weave-scope-component=app

[root@sh-saas-k8s1-master-dev-01 dev]# kubectl describe service weave-scope-app -n weave 
Name:         weave-scope-app
Namespace:    weave
Labels:       app=weave-scope
              name=weave-scope-app
              weave-cloud-component=scope
              weave-scope-component=app
Annotations:  cloud.weave.works/launcher-info={
  "original-request": {
    "url": "/k8s/scope.yaml?k8s-version=Q2xpZW50IFZlcnNpb246IHZlcnNpb24uSW5mb3tNYWpvcjoiMSIsIE1pbm9yOiIxMCIsIEdpdFZlcnNpb246InYxLjEwLjExIiwgR2...
                   kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"cloud.weave.works/launcher-info":"{\n  \"original-request\": {\n    \"url\": \"/k8s/sco...
Selector:          app=weave-scope,name=weave-scope-app,weave-cloud-component=scope,weave-scope-component=app
Type:              ClusterIP
IP:                10.253.255.240
Port:              app  80/TCP
TargetPort:        4040/TCP
Endpoints:         10.253.3.176:4040
Session Affinity:  None
Events:            <none>
```
居然已经有service了，我们可以创建一个ingress让外部可以访问：

`vim weave.yml`
```ymal
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: weave-scope-app-ingress
  namespace: weave
spec:
  rules:
  - host: "scope.k8s1.dev.huilog.com"
    http:
      paths:
      - backend:
          serviceName: weave-scope-app
          servicePort: 80
```
应用以上ingress：
```bash
kubectl apply -f weave.yml
```
现在使用访问浏览器打开以上域名来使用weave scope了。

PS：
Q:如果你的k8s前面还有使用nginx等做代理转发，可能会发现weave scope一直在连接。这个问题我就有碰到。
A:通过使用浏览器开发者工具调试，我们发现weave scope有一个请求的返回值是400,查看这个请求头，发现其需要使用websocket。于是对nginx的配置进行如下修改：
```bash
server {
    listen       80;
    charset utf-8;
    server_name  *.k8s1.dev.huilog.com;

    access_log  /data/log/nginx/k8s1.dev.huilog.com.access.log  ha;
    error_log   /data/log/nginx/k8s1.dev.huilog.com.error.log;

    client_body_timeout 5s;
    client_header_timeout 3s;
    req_status server_reqstat_monitor;
    location = /favicon.ico { access_log off; log_not_found off; }
    location ~ /\.          { access_log off; log_not_found off; }

    location / {
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host:$server_port;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_pass   http://k8s1_nodes;
    }

}
```
websocket需要的是这三个配置：
```bash
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
```
nginx重载生效,Weave Scope就可以正常使用了。

## 4. traefik basic auth的配置
Weave Scope的页面默认是没有用户管理的认证的。基于安全考虑，我们需要给这个管理后面配置一个http认证的功能。
由于我k8s的ingress使用的是traefik，下面我们在traefik的配置一个basic auth。

先创建一个认证文件：
```bash
[root@sh-saas-k8s1-master-dev-01 dev]# htpasswd -c auth admin
New password:
Re-type new password:
Adding password for user admin
```

然后再利用这个认证文件创建一个k8s的secret:
```bash
[root@sh-saas-k8s1-master-dev-01 dev]# kubectl create secret generic authsecret --from-file auth --namespace=weave
secret "authsecret" created
```

最后更新ingress配置，配置上basic auth：
```bash
[root@sh-saas-k8s1-master-dev-01 dev]# vim weave.yml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: weave-scope-app-ingress
  namespace: weave
  annotations:
    kubernetes.io/ingress.class: traefik
    ingress.kubernetes.io/auth-type: "basic"
    ingress.kubernetes.io/auth-secret: "authsecret"
spec:
  rules:
  - host: "scope.k8s1.dev.huilog.com"
    http:
      paths:
      - backend:
          serviceName: weave-scope-app
          servicePort: 80

[root@sh-saas-k8s1-master-dev-01 dev]# kubectl apply -f weave.yml
ingress.extensions "weave-scope-app-ingress" configured
```
重新打开weave scope的页面，可以看到必须输入用户名和密码才能登陆了。
