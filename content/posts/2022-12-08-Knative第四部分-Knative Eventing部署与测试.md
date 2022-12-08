---
title: Knative第四部分：Knative Eventing部署与测试
author: 阿辉
date: 2022-12-08T15:30:14+00:00
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
# Knative Eventing部署
下载YAML部署Eventing CRD以及Core:
```bash
wget https://github.com/knative/eventing/releases/download/v0.24.0/eventing-crds.yaml
wget https://github.com/knative/eventing/releases/download/v0.24.0/eventing-core.yaml
sed -i 's/gcr.io/gcr.tencentcloudcr.com/g' eventing-core.yaml 
kubectl apply -f ./eventing-crds.yaml 
kubectl apply -f ./eventing-core.yaml
```
<!--more-->

查看运行状态：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get all -n knative-eventing 
NAME                                       READY   STATUS    RESTARTS   AGE
pod/eventing-controller-55d94667f5-s8fpv   1/1     Running   0          53s
pod/eventing-webhook-5885bdd79c-nx4qw      1/1     Running   0          53s

NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/eventing-webhook   ClusterIP   10.248.156.175   <none>        443/TCP   7m30s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eventing-controller     1/1     1            1           7m30s
deployment.apps/eventing-webhook        1/1     1            1           7m30s
deployment.apps/pingsource-mt-adapter   0/0     0            0           7m30s

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/eventing-controller-55d94667f5     1         1         1       53s
replicaset.apps/eventing-controller-7dfc7f88b7     0         0         0       7m30s
replicaset.apps/eventing-webhook-5885bdd79c        1         1         1       53s
replicaset.apps/eventing-webhook-5c94fccb6f        0         0         0       7m30s
replicaset.apps/pingsource-mt-adapter-54f7b45844   0         0         0       53s
replicaset.apps/pingsource-mt-adapter-6f4585dd     0         0         0       7m30s

NAME                                                   REFERENCE                     TARGETS          MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/eventing-webhook   Deployment/eventing-webhook   <unknown>/100%   1         5         1          7m30s
```

安装默认的消息通道层，支持的Channel有：Kafka Channel,Google Cloud pub/sub Channel,In-Memory,NATS Channel。

这里我们选用最简单的In-Memory试用：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# wget https://github.com/knative/eventing/releases/download/v0.24.0/in-memory-channel.yaml
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g' in-memory-channel.yaml 
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g' in-memory-channel.yaml 
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f in-memory-channel.yaml 
namespace/knative-eventing unchanged
serviceaccount/imc-controller created
clusterrolebinding.rbac.authorization.k8s.io/imc-controller created
rolebinding.rbac.authorization.k8s.io/imc-controller created
serviceaccount/imc-dispatcher created
clusterrolebinding.rbac.authorization.k8s.io/imc-dispatcher created
configmap/config-imc-event-dispatcher created
configmap/config-observability unchanged
configmap/config-tracing unchanged
deployment.apps/imc-controller created
service/inmemorychannel-webhook created
service/imc-dispatcher created
deployment.apps/imc-dispatcher created
customresourcedefinition.apiextensions.k8s.io/inmemorychannels.messaging.knative.dev created
clusterrole.rbac.authorization.k8s.io/imc-addressable-resolver created
clusterrole.rbac.authorization.k8s.io/imc-channelable-manipulator created
clusterrole.rbac.authorization.k8s.io/imc-controller created
clusterrole.rbac.authorization.k8s.io/imc-dispatcher created
role.rbac.authorization.k8s.io/knative-inmemorychannel-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/inmemorychannel.eventing.knative.dev created
validatingwebhookconfiguration.admissionregistration.k8s.io/validation.inmemorychannel.eventing.knative.dev created
secret/inmemorychannel-webhook-certs created
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get all -n knative-eventing 
NAME                                       READY   STATUS    RESTARTS   AGE
pod/eventing-controller-55d94667f5-s8fpv   1/1     Running   0          7m40s
pod/eventing-webhook-5885bdd79c-nx4qw      1/1     Running   0          7m40s
pod/imc-controller-864bf555d9-hn24k        1/1     Running   0          2m47s
pod/imc-dispatcher-898665c89-jzjdx         1/1     Running   0          2m47s

NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/eventing-webhook          ClusterIP   10.248.156.175   <none>        443/TCP   14m
service/imc-dispatcher            ClusterIP   10.248.212.43    <none>        80/TCP    2m47s
service/inmemorychannel-webhook   ClusterIP   10.248.168.40    <none>        443/TCP   2m47s

NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/eventing-controller     1/1     1            1           14m
deployment.apps/eventing-webhook        1/1     1            1           14m
deployment.apps/imc-controller          1/1     1            1           2m47s
deployment.apps/imc-dispatcher          1/1     1            1           2m47s
deployment.apps/pingsource-mt-adapter   0/0     0            0           14m

NAME                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/eventing-controller-55d94667f5     1         1         1       7m40s
replicaset.apps/eventing-controller-7dfc7f88b7     0         0         0       14m
replicaset.apps/eventing-webhook-5885bdd79c        1         1         1       7m40s
replicaset.apps/eventing-webhook-5c94fccb6f        0         0         0       14m
replicaset.apps/imc-controller-864bf555d9          1         1         1       2m47s
replicaset.apps/imc-dispatcher-898665c89           1         1         1       2m47s
replicaset.apps/pingsource-mt-adapter-54f7b45844   0         0         0       7m40s
replicaset.apps/pingsource-mt-adapter-6f4585dd     0         0         0       14m

NAME                                                   REFERENCE                     TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/eventing-webhook   Deployment/eventing-webhook   8%/100%   1         5         1          14m
```

安装代理层(broker layer)，可选的有Apache Kafka Broker, MT-Channel-based,RabbitMQ Broker。

这里我们选用MT-Channel-based来测试，MT-Channel-based是一个支持多租户且简单的broker layer实现:
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# wget https://github.com/knative/eventing/releases/download/v0.24.0/mt-channel-broker.yaml
--2021-08-18 17:06:42--  https://github.com/knative/eventing/releases/download/v0.24.0/mt-channel-broker.yaml
Resolving github.com (github.com)... 13.229.188.59
Connecting to github.com (github.com)|13.229.188.59|:443... connected.
Unable to establish SSL connection.
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g' mt-ch^C
[root@sh-saas-k8stest-master-dev-01 knative]# wget https://github.com/knative/eventing/releases/download/v0.24.0/mt-channel-broker.yaml
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g' mt-channel-broker.yaml 
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f mt-channel-broker.yaml 
clusterrole.rbac.authorization.k8s.io/knative-eventing-mt-channel-broker-controller created
clusterrole.rbac.authorization.k8s.io/knative-eventing-mt-broker-filter created
serviceaccount/mt-broker-filter created
clusterrole.rbac.authorization.k8s.io/knative-eventing-mt-broker-ingress created
serviceaccount/mt-broker-ingress created
clusterrolebinding.rbac.authorization.k8s.io/eventing-mt-channel-broker-controller created
clusterrolebinding.rbac.authorization.k8s.io/knative-eventing-mt-broker-filter created
clusterrolebinding.rbac.authorization.k8s.io/knative-eventing-mt-broker-ingress created
deployment.apps/mt-broker-filter created
service/broker-filter created
deployment.apps/mt-broker-ingress created
service/broker-ingress created
deployment.apps/mt-broker-controller created
horizontalpodautoscaler.autoscaling/broker-ingress-hpa created
horizontalpodautoscaler.autoscaling/broker-filter-hpa created
```


```bash
[root@sh-saas-k8stest-master-dev-01 knative]# wget https://raw.githubusercontent.com/knative/docs/master/docs/install/collecting-metrics/collector.yaml
[root@sh-saas-k8stest-master-dev-01 knative]# sed -i 's/gcr.io/gcr.tencentcloudcr.com/g' collector.yaml
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl create namespace metrics
namespace/metrics created
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f collector.yaml 
configmap/otel-collector-config created
deployment.apps/otel-collector created
service/otel-collector created
service/otel-export created
```
