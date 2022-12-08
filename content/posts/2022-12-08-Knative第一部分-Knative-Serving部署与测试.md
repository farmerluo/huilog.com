---
title: Knative第一部分-Knative Serving部署与测试
author: 阿辉
date: 2022-12-08T15:00:14+08:00
categories:
- Serverless
tags:
- Serverless
- Knative
keywords:
- Serverless
- Knative
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
{{< toc >}}

# Knative部署与测试
## 1. Knative简介

[knative](https://github.com/knative) 是谷歌牵头的 serverless 架构方案，旨在提供一套简单易用的 serverless 开源方案，把 serverless 标准化和平台化。目前参与 knative 项目的公司主要有： Google、Pivotal、IBM、Red Hat和SAP。

早期版本有以下Knative组件：

- Build - 源到容器的构建编排
- Eventing - 管理和交付事件
- Serving - 请求驱动的计算，可以扩展到零

因某些原因，2019年Build组年已经发展成一个独立项目Tekton，所以Knative现在主要由两个部分组成，Knative Serving以及Knative Eventing.

<!--more-->

## 2. 安装前准备

本次安装的Knative为最新的0.24版本，部署要求为：

- Kubernetes v1.18或以上版本，我们这次是基于Kubernetes v1.19版本进行测试
- Kubernetes集群每节点最低有2 CPUs, 4 GB内存, 20 GB以上的磁盘空间
- 选择一种Knative的网络层，可选的有Kourier,Ambassador,Contour,Istio，我们使用Istio来进行测试

## 3. 安装Knative Serving

### 3.1 部署Knative Serving

通过下面的操作，我们可以先安装Knative Serving:

```bash
wget https://github.com/knative/serving/releases/download/v0.24.0/serving-crds.yaml
wget https://github.com/knative/serving/releases/download/v0.24.0/serving-core.yaml
kubectl apply -f ./serving-crds.yaml 
kubectl apply -f ./serving-core.yaml 
```

查看各个资源是否已经正常：

```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get all -n knative-serving 
NAME                                         READY   STATUS             RESTARTS   AGE
pod/activator-dfc4f7578-265fn                0/1     ImagePullBackOff   0          15h
pod/autoscaler-756797655b-nc8ln              0/1     ImagePullBackOff   0          15h
pod/controller-7bccdf6fdb-6tznw              0/1     ImagePullBackOff   0          15h
pod/domain-mapping-65fd554865-9zv4t          0/1     ImagePullBackOff   0          15h
pod/domainmapping-webhook-7ff8f59965-p9hvc   0/1     ImagePullBackOff   0          15h
pod/webhook-568c4d697-xvrhc                  0/1     ImagePullBackOff   0          15h

NAME                            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                           AGE
service/activator-service       ClusterIP   10.248.231.247   <none>        9090/TCP,8008/TCP,80/TCP,81/TCP   15h
service/autoscaler              ClusterIP   10.248.128.26    <none>        9090/TCP,8008/TCP,8080/TCP        15h
service/controller              ClusterIP   10.248.129.16    <none>        9090/TCP,8008/TCP                 15h
service/domainmapping-webhook   ClusterIP   10.248.155.59    <none>        9090/TCP,8008/TCP,443/TCP         15h
service/webhook                 ClusterIP   10.248.172.13    <none>        9090/TCP,8008/TCP,443/TCP         15h

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/activator               0/1     1            0           15h
deployment.apps/autoscaler              0/1     1            0           15h
deployment.apps/controller              0/1     1            0           15h
deployment.apps/domain-mapping          0/1     1            0           15h
deployment.apps/domainmapping-webhook   0/1     1            0           15h
deployment.apps/webhook                 0/1     1            0           15h

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/activator-dfc4f7578                1         1         0       15h
replicaset.apps/autoscaler-756797655b              1         1         0       15h
replicaset.apps/controller-7bccdf6fdb              1         1         0       15h
replicaset.apps/domain-mapping-65fd554865          1         1         0       15h
replicaset.apps/domainmapping-webhook-7ff8f59965   1         1         0       15h
replicaset.apps/webhook-568c4d697                  1         1         0       15h

NAME                                            REFERENCE              TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/activator   Deployment/activator   <unknown>/100%   1         20        1          15h
horizontalpodautoscaler.autoscaling/webhook     Deployment/webhook     <unknown>/100%   1         5         1          15h
```

我们可以发现，POD没有起来，原因为ImagePullBackOff。主要是国内拉不到海外镜像的原因，下面的命令通过更换国内加速镜像地址解决：

```bash
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g'   serving-core.yaml 
[root@sh-saas-k8stest-master-dev-01 knative]# grep "gcr.tencentcloudcr.com" serving-core.yaml 
  image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/queue@sha256:6c6fdac40d3ea53e39ddd6bb00aed8788e69e7fac99e19c98ed911dd1d2f946b
  queueSidecarImage: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/queue@sha256:6c6fdac40d3ea53e39ddd6bb00aed8788e69e7fac99e19c98ed911dd1d2f946b
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/activator@sha256:bd48057f8d6e5f95f389225979c89d2d99d52a4876d2342fc033c909096d4698
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/autoscaler@sha256:d3d6f5ecfff577a6d4d5a0549379fa1acf1d66d7422a58b1756e40eba555285c
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/controller@sha256:e9b9cc7a2a091194161a7a256d6fc342317080f1a31e1f1a33993a5e1993fdb5
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/domain-mapping@sha256:fafa1afcc9f2c6e79b096ffbc67e14ff3f34e3591d887335811ca15970b3b837
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/domain-mapping-webhook@sha256:1335e57427a372d15e8a90a3370eaa6896c9cb3fd7cb6f2a63021cceaea2e8e6
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/serving/cmd/webhook@sha256:e9503135b4b46a3700d91bdb32df60923575aaedbcc5192bdc3e41b64591ee50
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f ./serving-core.yaml
```

现在查看POD状态，都正常了：

```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get pod -n knative-serving
NAME                                     READY   STATUS    RESTARTS   AGE
activator-bbb849d6-ch9lq                 1/1     Running   0          28s
autoscaler-6fcf6994dd-ts26g              1/1     Running   0          28s
controller-7b586546c8-4rtfq              1/1     Running   0          28s
domain-mapping-c464c4dfc-2w4zg           1/1     Running   0          27s
domainmapping-webhook-6f947f9f89-dn4kv   1/1     Running   0          27s
webhook-7945667f7-kfv58                  1/1     Running   0          26s
```

如果没有部署istio,需先部署istio，我这边已经部署好istio 1.10.3，就跳过这步了。

### 3.2 部署Knative Istio controller

```bash
[root@sh-saas-k8stest-master-dev-01 knative]# wget https://github.com/knative/net-istio/releases/download/v0.24.0/net-istio.yaml
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g'  net-istio.yaml 
[root@sh-saas-k8stest-master-dev-01 knative]# grep "gcr.tencentcloudcr.com"  net-istio.yaml 
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/net-istio/cmd/controller@sha256:41f58d7e8319e473dbf27dec3d329b631f03ec0899c98ba2dd3d3166a6fb7a42
          image: gcr.tencentcloudcr.com/knative-releases/knative.dev/net-istio/cmd/webhook@sha256:49bf045db42aa0bfe124e9c5a5c511595e8fc27bf3a77e3900d9dec2b350173c
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f net-istio.yaml 
clusterrole.rbac.authorization.k8s.io/knative-serving-istio created
gateway.networking.istio.io/knative-ingress-gateway created
gateway.networking.istio.io/knative-local-gateway created
service/knative-local-gateway created
configmap/config-istio created
peerauthentication.security.istio.io/webhook created
peerauthentication.security.istio.io/domainmapping-webhook created
peerauthentication.security.istio.io/net-istio-webhook created
deployment.apps/net-istio-controller created
deployment.apps/net-istio-webhook created
secret/net-istio-webhook-certs created
service/net-istio-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/webhook.istio.networking.internal.knative.dev created
validatingwebhookconfiguration.admissionregistration.k8s.io/config.webhook.istio.networking.internal.knative.dev created
[root@sh-saas-k8stest-master-dev-01 knative]# 

```

## 4. 测试Knative Serving

编写一个helloworld-go.yaml：
```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: helloworld-go
  namespace: example
spec:
  template:
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
        env:
        - name: TARGET
          value: "Go Sample v1"
```

创建上面的Kn Service:
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl create namespace example
namespace/example created
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f helloworld-go.yaml 
service.serving.knative.dev/helloworld-go created
```

查看Knative Serving创建的相关资源：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get all -n example
NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP                                            PORT(S)                                      AGE
service/helloworld-go                 ExternalName   <none>           knative-local-gateway.istio-system.svc.cluster.local   80/TCP                                       2m2s
service/helloworld-go-00001           ClusterIP      10.248.140.210   <none>                                                 80/TCP                                       2m13s
service/helloworld-go-00001-private   ClusterIP      10.248.227.128   <none>                                                 80/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   2m13s

NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helloworld-go-00001-deployment   0/0     0            0           2m13s

NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/helloworld-go-00001-deployment-7754b949cb   0         0         0       2m13s

NAME                                      URL                                        READY   REASON
route.serving.knative.dev/helloworld-go   http://helloworld-go.example.example.com   True    

NAME                                        URL                                        LATESTCREATED         LATESTREADY           READY   REASON
service.serving.knative.dev/helloworld-go   http://helloworld-go.example.example.com   helloworld-go-00001   helloworld-go-00001   True    

NAME                                               CONFIG NAME     K8S SERVICE NAME      GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
revision.serving.knative.dev/helloworld-go-00001   helloworld-go   helloworld-go-00001   1            True             0                 0

NAME                                              LATESTCREATED         LATESTREADY           READY   REASON
configuration.serving.knative.dev/helloworld-go   helloworld-go-00001   helloworld-go-00001   True    
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get svc istio-ingressgateway --namespace istio-system
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                                                                                      AGE
istio-ingressgateway   NodePort   10.248.132.176   <none>        80:20080/TCP,443:20443/TCP,15020:37042/TCP,15029:31676/TCP,15030:26910/TCP,15031:30084/TCP,15032:30347/TCP,15443:27715/TCP,31400:25312/TCP   4d7h

```

访问一下，可以看到第一次访问时间花了1.45秒，原因是流量来了，自动扩容新起来了一个POD：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m1.457s
user    0m0.001s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get all -n example

NAME                                                  READY   STATUS    RESTARTS   AGE
pod/helloworld-go-00001-deployment-7754b949cb-4zsl5   2/2     Running   0          7s

NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP                                            PORT(S)                                      AGE
service/helloworld-go                 ExternalName   <none>           knative-local-gateway.istio-system.svc.cluster.local   80/TCP                                       11m
service/helloworld-go-00001           ClusterIP      10.248.140.210   <none>                                                 80/TCP                                       12m
service/helloworld-go-00001-private   ClusterIP      10.248.227.128   <none>                                                 80/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   12m

NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helloworld-go-00001-deployment   1/1     1            1           12m

NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/helloworld-go-00001-deployment-7754b949cb   1         1         1       12m

NAME                                        URL                                        LATESTCREATED         LATESTREADY           READY   REASON
service.serving.knative.dev/helloworld-go   http://helloworld-go.example.example.com   helloworld-go-00001   helloworld-go-00001   True    

NAME                                               CONFIG NAME     K8S SERVICE NAME      GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
revision.serving.knative.dev/helloworld-go-00001   helloworld-go   helloworld-go-00001   1            True             1                 1

NAME                                              LATESTCREATED         LATESTREADY           READY   REASON
configuration.serving.knative.dev/helloworld-go   helloworld-go-00001   helloworld-go-00001   True    

NAME                                      URL                                        READY   REASON
route.serving.knative.dev/helloworld-go   http://helloworld-go.example.example.com   True  
```

再次访问，会发现现在访问速度快了很多：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m0.007s
user    0m0.001s
sys     0m0.003s
```


大约一分钟后，自动缩容到了0:
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get all -n example
NAME                                                  READY   STATUS        RESTARTS   AGE
pod/helloworld-go-00001-deployment-7754b949cb-whsqm   1/2     Terminating   0          66s

NAME                                  TYPE           CLUSTER-IP       EXTERNAL-IP                                            PORT(S)                                      AGE
service/helloworld-go                 ExternalName   <none>           knative-local-gateway.istio-system.svc.cluster.local   80/TCP                                       7m11s
service/helloworld-go-00001           ClusterIP      10.248.140.210   <none>                                                 80/TCP                                       7m22s
service/helloworld-go-00001-private   ClusterIP      10.248.227.128   <none>                                                 80/TCP,9090/TCP,9091/TCP,8022/TCP,8012/TCP   7m22s

NAME                                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/helloworld-go-00001-deployment   0/0     0            0           7m22s

NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/helloworld-go-00001-deployment-7754b949cb   0         0         0       7m22s

NAME                                        URL                                        LATESTCREATED         LATESTREADY           READY   REASON
service.serving.knative.dev/helloworld-go   http://helloworld-go.example.example.com   helloworld-go-00001   helloworld-go-00001   True    

NAME                                               CONFIG NAME     K8S SERVICE NAME      GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
revision.serving.knative.dev/helloworld-go-00001   helloworld-go   helloworld-go-00001   1            True             0                 0

NAME                                              LATESTCREATED         LATESTREADY           READY   REASON
configuration.serving.knative.dev/helloworld-go   helloworld-go-00001   helloworld-go-00001   True    

NAME                                      URL                                        READY   REASON
route.serving.knative.dev/helloworld-go   http://helloworld-go.example.example.com   True 
```

## 5. Knative Serving相关资源

通过以下命令可以查看Knative Serving的所有CRD资源：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl api-resources | grep knative
metrics                                           autoscaling.internal.knative.dev   true         Metric
podautoscalers                    kpa,pa          autoscaling.internal.knative.dev   true         PodAutoscaler
images                            img             caching.internal.knative.dev       true         Image
certificates                      kcert           networking.internal.knative.dev    true         Certificate
clusterdomainclaims               cdc             networking.internal.knative.dev    false        ClusterDomainClaim
ingresses                         kingress,king   networking.internal.knative.dev    true         Ingress
serverlessservices                sks             networking.internal.knative.dev    true         ServerlessService
configurations                    config,cfg      serving.knative.dev                true         Configuration
domainmappings                    dm              serving.knative.dev                true         DomainMapping
revisions                         rev             serving.knative.dev                true         Revision
routes                            rt              serving.knative.dev                true         Route
services                          kservice,ksvc   serving.knative.dev                true         Service
```

## 参考

官方文档：
https://knative.dev/docs/admin/install/