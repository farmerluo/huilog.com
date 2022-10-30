---
title: 通过coredns跨k8s集群解析DNS
author: 阿辉
date: 2021-12-13T11:02:05+00:00
categories:
- Kubernetes
- Coredns
tags:
- Kubernetes
- Coredns
keywords:
- Kubernetes
- Coredns
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---

最近在弄AlertManager,发现其集群模式需要知道各个AlertManager节点的IP地址，而我们现在需要在2个K8s集群同时部署AlertManager上来创建一套AlertManager，以确保在某一个K8s集群挂了后报警还能发出来，同时确保当2个K8s集群都正常的时候，一个告警只发一次出来。

但怎么在A集群上找到B集群上的POD的IP，这是个问题。通过研究，我们发现CoreDns可以很好的解决这个问题。

假设我们现在有2套集群，2套集群使用的DNS后缀都是cluster.local，容器网段及service网段都不一样。

我们现在在A集群上，先看旧的CoreDNS的配置：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log . {combined} {
          class error
        }
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        reload
        loadbalance
        #rewrite name saas-fe-zhan.internal.com public-fe-zhan-node-dev.public-fe-node-dev.svc.cluster.local
    }
    
```

<!--more-->

可以看到，只配置了默认根域，我们可以再增加一个B集群的域的配置。修改CoreDNS的配置如下：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        log . {combined} {
          class error
        }
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        reload
        loadbalance
        #rewrite name saas-fe-zhan.internal.com public-fe-zhan-node-dev.public-fe-node-dev.svc.cluster.local
    }
    cluster.dev.:53 {
        log . {combined} {
          class error
        }
        errors
        health
        ready
        kubernetes cluster.dev {
          kubeconfig /etc/coredns/RemoteK8sConfig context1
        }
        cache 30
        reload
        loadbalance
    }

  RemoteK8sConfig: |
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JS==
        server: https://k8s-apiserver.internal.com:8443
      name: cluster1
    contexts:
    - context:
        cluster: cluster1
        user: user1
      name: context1
    current-context: context1
    kind: Config
    preferences: {}
    users:
    - name: user1
      user:
        client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JS=
        client-key-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JS=
```

需要注意的是`kubeconfig /etc/coredns/RemoteK8sConfig context1`中的context1表示kubeconfig文件中的上下文名，这个参数官方文档显示为可选，实际上按照我的测试，不写会出现如下报错：
```
plugin/kubernetes: ./coreconf:24 - Error during parsing: Wrong argument count or unexpected line ending after '/etc/coredns/RemoteK8sConfig'
```

将以上配置生效，修改CoreDns的Depolyment，增加RemoteK8sConfig配置加载：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: kube-dns
    kubernetes.io/name: CoreDNS
  name: coredns
  namespace: kube-system
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-dns
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: kube-dns
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: k8s-app
                  operator: In
                  values:
                  - kube-dns
              topologyKey: kubernetes.io/hostname
            weight: 100
      containers:
      - args:
        - -conf
        - /etc/coredns/Corefile
        image: coredns/coredns:1.8.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 5
        name: coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /ready
            port: 8181
            scheme: HTTP
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        securityContext:
          allowPrivilegeEscalation: false
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - all
          readOnlyRootFilesystem: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /etc/coredns
          name: config-volume
          readOnly: true
      dnsPolicy: Default
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-cluster-critical
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      serviceAccount: coredns
      serviceAccountName: coredns
      terminationGracePeriodSeconds: 30
      tolerations:
      - key: CriticalAddonsOnly
        operator: Exists
      volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
          - key: RemoteK8sConfig
            path: RemoteK8sConfig
          name: coredns
        name: config-volume

```

CoreDns的配置重启后，我们现在测试和验证效果：

通过查询DNS `kubernetes.default.svc.cluster.local`和`kubernetes.default.svc.cluster.dev`可以看到解析出来的IP是不一样的。说明配置是有生效的。

```bash
[root@sh-saas-k8stest-master-dev-01 yaml]# dig @10.248.1.193 kubernetes.default.svc.cluster.local

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> @10.248.1.193 kubernetes.default.svc.cluster.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49170
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubernetes.default.svc.cluster.local. IN A

;; ANSWER SECTION:
kubernetes.default.svc.cluster.local. 5	IN A	10.248.252.1

;; Query time: 0 msec
;; SERVER: 10.248.1.193#53(10.248.1.193)
;; WHEN: Mon Dec 13 18:48:25 CST 2021
;; MSG SIZE  rcvd: 117

[root@sh-saas-k8stest-master-dev-01 yaml]# dig @10.248.1.193 kubernetes.default.svc.cluster.dev

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> @10.248.1.193 kubernetes.default.svc.cluster.dev
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59137
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;kubernetes.default.svc.cluster.dev. IN	A

;; ANSWER SECTION:
kubernetes.default.svc.cluster.dev. 5 IN A	10.253.252.1

;; Query time: 0 msec
;; SERVER: 10.248.1.193#53(10.248.1.193)
;; WHEN: Mon Dec 13 18:48:29 CST 2021
;; MSG SIZE  rcvd: 113
```


再查一下B集群上的一个有状态服务的POD DNS，也没问题。

```bash
[root@sh-saas-k8stest-master-dev-01 yaml]# dig @10.248.1.193 vmstorage-victoria-metrics-k8s-stack-0.vmstorage-victoria-metrics-k8s-stack.victoria.svc.cluster.dev

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-9.P2.el7 <<>> @10.248.1.193 vmstorage-victoria-metrics-k8s-stack-0.vmstorage-victoria-metrics-k8s-stack.victoria.svc.cluster.dev
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 15816
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;vmstorage-victoria-metrics-k8s-stack-0.vmstorage-victoria-metrics-k8s-stack.victoria.svc.cluster.dev. IN A

;; ANSWER SECTION:
vmstorage-victoria-metrics-k8s-stack-0.vmstorage-victoria-metrics-k8s-stack.victoria.svc.cluster.dev. 5	IN A 10.253.7.107

;; Query time: 0 msec
;; SERVER: 10.248.1.193#53(10.248.1.193)
;; WHEN: Mon Dec 13 18:47:57 CST 2021
;; MSG SIZE  rcvd: 245

[root@sh-saas-k8s1-master-dev-01 ~]# kubectl get pod vmstorage-victoria-metrics-k8s-stack-0 -n victoria -o wide
NAME                                     READY   STATUS    RESTARTS   AGE     IP             NODE          NOMINATED NODE   READINESS GATES
vmstorage-victoria-metrics-k8s-stack-0   2/2     Running   1          5d23h   10.253.7.107   10.12.97.33   <none>           <none>
```

最后，留一个问题，2个K8s集群配置的域名后缀都是cluster.local，为什么通过后缀cluster.dev却能查出来结果？