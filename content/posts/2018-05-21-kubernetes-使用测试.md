---
title: kubernetes 日常使用及操作测试
author: 阿辉
date: 2018-05-21T06:02:37+00:00
categories:
- Kubernetes
tags:
- Kubernetes
keywords:
- Kubernetes
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false

---
{{< toc >}}

前文已经安装好了一套kubernetes 1.10，下面我们来进行日常使用测试
## 1. 创建部署及服务
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
<!--more-->

## 2. 测试部署及服务
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
我们通过curl -x 172.21.251.111:80 http://nginx.k8s.dev.huilog.com 测试，可以看到是可以正常得到结果的。后面可以在Node的前端配置一台LB（nginx或硬件LB都可以），把这个域名配置到nginx上，后端发到k8s的traefik的节点上。

## 3. 健康检测机制
kubernetes提供了liveness及readiness两种pod健康检测探针：

- Kubelet使用liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，liveness探针将捕获到deadlock，重启处于该状态下的容器。
- Kubelet使用readiness probe（就绪探针）来确定容器是否已经完全启动就绪并可以接受流量。只有当Pod中的容器都处于就绪状态时kubelet才会认定该Pod处于就绪状态。该信号的作用是控制哪些Pod应该作为service的后端。如果Pod处于非就绪状态，那么pod将不会进入service的load balancer中。
- 注意：liveness及readiness两种pod健康检测探针的配置方式是一样的，可参考3.1中的例子。
- [官方参考](http://https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/ "官方参考")

### 3.1 probe方式
kubernotes提供了三种probe方式，分别为：

- httpGet
httpGet方式是通过定时去请求一个http URL来确定是检测状态，如果返回的http 状态代码在200~400之间为成功，否则为失败。以下为httpGet的例子：
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```
httpGet可配置的参数如下：

- host：连接的主机名，默认连接到pod的IP。你可能想在http header中设置”Host”而不是使用IP。
- scheme：连接使用的schema，默认HTTP。
- path: 访问的HTTP server的path。
- httpHeaders：自定义请求的header。HTTP运行重复的header。
- port：访问的容器的端口名字或者端口号。端口号必须介于1和65525之间。

对于HTTP探测器，kubelet向指定的路径和端口发送HTTP请求以执行检查。 Kubelet将probe发送到容器的IP地址，除非地址被httpGet中的可选host字段覆盖。 在大多数情况下，你不想设置主机字段。 有一种情况下你可以设置它。 假设容器在127.0.0.1上侦听，并且Pod的hostNetwork字段为true。 然后，在httpGet下的host应该设置为127.0.0.1。 如果你的pod依赖于虚拟主机，这可能是更常见的情况，你不应该用host，而是应该在httpHeaders中设置Host头。

- exec
exec方式为定时去执行一个检测脚本或命令行，如果检测脚本或命令行执行后，退出码为0为成功，否则如果非0退出码为失败。
```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```
- tcpSocket
tcpSocket为通过探测tcp端口来确定存活状态，如果tcp端口通为成功，否则为失败。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```
### 3.2 通用的probe参数
- initialDelaySeconds：容器启动后第一次执行探测是需要等待多少秒。
- periodSeconds：执行探测的频率。默认是10秒，最小1秒。
- timeoutSeconds：探测超时时间。默认1秒，最小1秒。
- successThreshold：探测失败后，最少连续探测成功多少次才被认定为成功。默认是1。对于liveness必须是1。最小值是1。
- failureThreshold：探测成功后，最少连续探测失败多少次才被认定为失败。默认是3。最小值是1。

