---
title: 在K8S 的ingress上配置HTTP认证的方法
author: 阿辉
date: 2018-04-26T07:42:42+00:00
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
在K8S 的ingress上配置HTTP认证的方法如下：

1 . 使用htpasswd创建一个auth文件：
```bash
htpasswd -c ./auth myusername
cat auth
myusername:$apr1$78Jyn/1K$ERHKVRPPlzAX8eBtLuvRZ0
```
2. 创建一个K8S的secret:
```bash
kubectl create secret generic mysecret --from-file auth --namespace=monitoring 
kubectl --namespace=monitoring get secret mysecret 
NAME      TYPE    DATA    AGE 
mysecret Opaque   1      106d
```
<!--more-->
3. 通过以下参数将创建的secret与ingress关联起来:
```yaml
ingress.kubernetes.io/auth-type: "basic"
ingress.kubernetes.io/auth-secret: "mysecret"
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: prometheus-dashboard
 namespace: monitoring
 annotations:
   kubernetes.io/ingress.class: traefik
   ingress.kubernetes.io/auth-type: "basic"
   ingress.kubernetes.io/auth-secret: "mysecret"
spec:
 rules:
 - host: dashboard.prometheus.example.com
   http:
     paths:
     - backend:
         serviceName: prometheus
         servicePort: 9090
```
最后创建就可以了：
```bash
kubectl create -f prometheus-ingress.yaml -n monitoring
```
参考：

https://docs.traefik.io/user-guide/kubernetes/

https://docs.traefik.io/configuration/backends/kubernetes/