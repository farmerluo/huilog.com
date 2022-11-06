---
title: kubernetes traefik ingress 安装及配置
author: 阿辉
date: 2018-05-08T15:09:05+00:00
categories:
- Kubernetes
- Traefik
tags:
- Kubernetes
- Traefik
keywords:
- Kubernetes
- Traefik
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---
Traefik是一款开源的反向代理与负载均衡工具。它最大的优点是能够与常见的微服务系统直接整合，可以实现自动化动态配置。目前支持Docker, Swarm, Mesos/Marathon, Mesos, Kubernetes, Consul, Etcd, Zookeeper, BoltDB, Rest API等等后端模型。
以下是架构图:
![](/wp-content/uploads/2018/05/architecture.png)
需要指出的是，ingress-controllers其实是kubernetes的一部分，ingress就是从kubernetes集群外访问集群的入口，将用户的URL请求转发到不同的service上。Ingress相当于nginx、apache等负载均衡方向代理服务器，其中还包括规则定义，即URL的路由信息，路由信息得的刷新由Ingress controller来提供。

Ingress Controller 实质上可以理解为是个监视器，Ingress Controller 通过不断地跟 kubernetes API 打交道，实时的感知后端 service、pod 等变化，比如新增和减少 pod，service 增加与减少等；当得到这些变化信息后，Ingress Controller 再结合下文的 Ingress 生成配置，然后更新反向代理负载均衡器，并刷新其配置，达到服务发现的作用。

<!--more-->

## 1. 部署traefik
traefik的yaml文件可以从 https://github.com/containous/traefik/tree/master/examples/k8s 下载，无需修改可直接部署在kubernetes上。需要注意的是其提供了DaemonSet及deployment两种部署方式 ，我使用的是DaemonSet部署方式：
```bash
mkdir traefik
cd traefik/
wget https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-rbac.yaml
wget https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/traefik-ds.yaml

kubectl create -f traefik-ds.yaml
serviceaccount "traefik-ingress-controller" created
daemonset.extensions "traefik-ingress-controller" created
service "traefik-ingress-service" created

kubectl create -f traefik-rbac.yaml
clusterrole.rbac.authorization.k8s.io "traefik-ingress-controller" created
clusterrolebinding.rbac.authorization.k8s.io "traefik-ingress-controller" created
```
traefik有traefik-ingress-controller及traefik-web-ui两部分，后者为一个WEB的管理界面，可以使用traefik-ingress将其配置成集群外WEB访问.
```bash
wget https://raw.githubusercontent.com/containous/traefik/master/examples/k8s/ui.yaml
#修改host: traefik-ui.minikube改为你的域名
kubectl create -f ui.yaml
```
好了，现在可以用刚刚配置的域名来访问traefik web ui了。
![](/wp-content/uploads/2018/05/traefik.jpg)

## 2. traefik使用测试
编辑一个yaml文件：
```yaml
vim nginx.yaml 
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-router
  namespace: test
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx-router
    spec:
      containers:
      - name: nginx-router
        image: 172.21.248.242/base/nginx
        ports:
        - containerPort: 80
		
---
kind: Service
apiVersion: v1
metadata:
  name: nginx-router
  namespace: test
spec:
  selector:
    app: nginx-router
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
	  
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-router-ingress
  namespace: test
  annotations:
    kubernetes.io/ingress.class: traefik
spec:
  rules:
  - host: "nginx.k8s.dev.huilog.com"
    http:
      paths:
      - backend:
          serviceName: nginx-router
          servicePort: 80
		  
# 创建部署及服务：
[root@bs-ops-test-docker-dev-01 dev]# kubectl create -f nginx.yaml 
deployment.extensions "nginx-router" created
service "nginx-router" created
ingress.extensions "nginx-router-ingress" created
[root@bs-ops-test-docker-dev-01 dev]#
```
测试：
```shell
[root@bs-ops-test-docker-dev-01 dev]# kubectl -n=test get deploy -o wide
NAME           DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS     IMAGES                      SELECTOR
nginx-router   2         2         2            2           2d        nginx-router   172.21.248.242/base/nginx   app=nginx-router
[root@bs-ops-test-docker-dev-01 dev]# kubectl -n=test get pod  -o wide | grep nginx-router
nginx-router-6d85cfb7cd-25mnb   1/1       Running   0          2d        10.244.2.16   bs-ops-test-docker-dev-03
nginx-router-6d85cfb7cd-2xhw6   1/1       Running   0          2d        10.244.1.3    bs-ops-test-docker-dev-02
[root@bs-ops-test-docker-dev-01 dev]# curl -x 172.21.251.111:80 http://nginx.k8s.dev.huilog.com
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
我们通过curl -x 172.21.251.111:80 http://nginx.k8s.dev.huilog.com
 测试，可以看到是可以正常得到结果的。后面可以在Node的前端配置一台LB（nginx或硬件LB都可以），把这个域名配置到nginx上，后端发到k8s的traefik的节点上。
需要注意的是，在kubernetes前配置lb需要注意把host带过来，例如nginx配置一定要如下参数：
proxy_set_header Host $host;
否则将会返回404的错误，原因是以IP去kubernetes访问，而不是域名,所以返回的是traefik默认的404页。

## 3. 参考资料
- https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-controllers
- https://jimmysong.io/kubernetes-handbook/practice/traefik-ingress-installation.html
- https://docs.traefik.io/