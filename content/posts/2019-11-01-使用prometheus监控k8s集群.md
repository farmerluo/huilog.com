---
title: 使用prometheus监控k8s集群
author: 阿辉
date: 2019-11-01T10:17:20+00:00
categories:
- Kubernetes
- Prometheus
tags:
- Kubernetes
- Prometheus
keywords:
- Kubernetes
- Prometheus
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

# 1. Prometheus 简介
以下是网上看到的简介：

Prometheus 是一套开源的系统监控报警框架。它启发于 Google 的 borgmon 监控系统，由工作在 SoundCloud 的 google 前员工在 2012 年创建，作为社区开源项目进行开发，并于 2015 年正式发布。2016 年，Prometheus 正式加入 Cloud Native Computing Foundation，成为受欢迎度仅次于 Kubernetes 的项目。

作为新一代的监控框架，Prometheus 具有以下特点：

- 强大的多维度数据模型：
- 时间序列数据通过 metric 名和键值对来区分。
- 所有的 metrics 都可以设置任意的多维标签。
- 数据模型更随意，不需要刻意设置为以点分隔的字符串。
- 可以对数据模型进行聚合，切割和切片操作。
- 支持双精度浮点类型，标签可以设为全 unicode。
- 灵活而强大的查询语句（PromQL）：在同一个查询语句，可以对多个 metrics 进行乘法、加法、连接、取分数位等操作。
- 易于管理： Prometheus server 是一个单独的二进制文件，可直接在本地工作，不依赖于分布式存储。
- 高效：平均每个采样点仅占 3.5 bytes，且一个 Prometheus server 可以处理数百万的 metrics。
- 使用 pull 模式采集时间序列数据，这样不仅有利于本机测试而且可以避免有问题的服务器推送坏的 metrics。
- 可以采用 push gateway 的方式把时间序列数据推送至 Prometheus server 端。
- 可以通过服务发现或者静态配置去获取监控的 targets。
- 有多种可视化图形界面。
- 易于伸缩。
需要指出的是，由于数据采集可能会有丢失，所以 Prometheus 不适用对采集数据要 100% 准确的情形。但如果用于记录时间序列数据，Prometheus 具有很大的查询优势，此外，Prometheus 适用于微服务的体系架构。

<!--more-->

# 2. Prometheus部署
先生成prometheus的命名空间和prometheus配置文件：
prometheus_configmap.yaml:
```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: prometheus

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: prometheus
data:
  prometheus.yml: |
    global:
      scrape_interval:     15s
      evaluation_interval: 15s
    scrape_configs:

    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https

    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

    - job_name: kubernetes-node-exporters
      honor_timestamps: true
      scrape_interval: 30s
      scrape_timeout: 30s
      metrics_path: /metrics
      scheme: http
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - separator: ;
        regex: __meta_kubernetes_node_label_(.+)
        replacement: $1
        action: labelmap
      - source_labels: [__meta_kubernetes_node_name]
        separator: ;
        regex: (.+)
        target_label: __address__
        replacement: ${1}:9100
        action: replace

    - job_name: 'kubernetes-cadvisor'
      kubernetes_sd_configs:
      - role: node
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor

    - job_name: 'kubernetes-service-endpoints'
      kubernetes_sd_configs:
      - role: endpoints
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_name

    - job_name: 'kubernetes-services'
      kubernetes_sd_configs:
      - role: service
      metrics_path: /probe
      params:
        module: [http_2xx]
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__address__]
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-ingresses'
      kubernetes_sd_configs:
      - role: ingress
      relabel_configs:
      - source_labels: [__meta_kubernetes_ingress_annotation_prometheus_io_probe]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_ingress_scheme,__address__,__meta_kubernetes_ingress_path]
        regex: (.+);(.+);(.+)
        replacement: ${1}://${2}${3}
        target_label: __param_target
      - target_label: __address__
        replacement: blackbox-exporter.example.com:9115
      - source_labels: [__param_target]
        target_label: instance
      - action: labelmap
        regex: __meta_kubernetes_ingress_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_ingress_name]
        target_label: kubernetes_name

    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
```

创建prometheus所需的SA和RABC，Deployment等配置文件：
prometheus.yaml:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-serviceaccount
  namespace: prometheus

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus-serviceaccount
  namespace: prometheus

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    name: prometheus-deployment
  name: prometheus
  namespace: prometheus
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      serviceAccountName: prometheus-serviceaccount
      containers:
      - image: prom/prometheus:v2.13.1
        name: prometheus
        command:
        - "/bin/prometheus"
        args:
        - "--config.file=/prometheus/conf/prometheus.yml"
        - "--storage.tsdb.path=/prometheus/data"
        - "--storage.tsdb.retention=30d"
        ports:
        - containerPort: 9090
          protocol: TCP
        volumeMounts:
        - mountPath: "/prometheus/data"
          name: data
        - mountPath: "/prometheus/conf"
          name: config-volume
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
          limits:
            cpu: 5000m
            memory: 2500Mi
      volumes:
      - emptyDir: {}
        name: data
      - configMap:
          name: prometheus-config
        name: config-volume

---
apiVersion: v1
kind: Service
metadata:
  annotations:
  name: prometheus
  namespace: prometheus
  labels:
    name: prometheus-service
spec:
  ports:
  - name: http
    port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
  sessionAffinity: None
  type: NodePort
```

部署prometheus:
```bash
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl apply -f prometheus_configmap.yaml 
namespace "prometheus" created
configmap "prometheus-config" created
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl apply -f prometheus.yaml 
serviceaccount "prometheus-serviceaccount" created
clusterrole.rbac.authorization.k8s.io "prometheus" created
clusterrolebinding.rbac.authorization.k8s.io "prometheus" created
deployment.extensions "prometheus" created
service "prometheus" created
```
部署成功，可以看看运行有没有什么问题：
```bash
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl get pod -n prometheus 
NAME                          READY     STATUS    RESTARTS   AGE
prometheus-77fd78b574-mrrz6   1/1       Running   0          10s
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl get pod -n prometheus  -o wide
NAME                          READY     STATUS    RESTARTS   AGE       IP            NODE
prometheus-77fd78b574-mrrz6   1/1       Running   0          17s       10.248.1.12   10.19.0.21
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl logs -n prometheus prometheus-77fd78b574-mrrz6 
level=warn ts=2019-11-01T09:27:44.501481603Z caller=main.go:295 deprecation_notice="\"storage.tsdb.retention\" flag is deprecated use \"storage.tsdb.retention.time\" instead."
level=info ts=2019-11-01T09:27:44.501537053Z caller=main.go:302 msg="Starting Prometheus" version="(version=2.7.1, branch=HEAD, revision=62e591f928ddf6b3468308b7ac1de1c63aa7fcf3)"
level=info ts=2019-11-01T09:27:44.501559636Z caller=main.go:303 build_context="(go=go1.11.5, user=root@f9f82868fc43, date=20190131-11:16:59)"
level=info ts=2019-11-01T09:27:44.501578409Z caller=main.go:304 host_details="(Linux 4.4.194-1.el7.elrepo.x86_64 #1 SMP Sat Sep 21 09:30:26 EDT 2019 x86_64 prometheus-77fd78b574-mrrz6 (none))"
level=info ts=2019-11-01T09:27:44.501597155Z caller=main.go:305 fd_limits="(soft=1048576, hard=1048576)"
level=info ts=2019-11-01T09:27:44.501615568Z caller=main.go:306 vm_limits="(soft=unlimited, hard=unlimited)"
level=info ts=2019-11-01T09:27:44.502652402Z caller=main.go:620 msg="Starting TSDB ..."
level=info ts=2019-11-01T09:27:44.502696381Z caller=web.go:416 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2019-11-01T09:27:44.507526594Z caller=main.go:635 msg="TSDB started"
level=info ts=2019-11-01T09:27:44.507566584Z caller=main.go:695 msg="Loading configuration file" filename=/prometheus/conf/prometheus.yml
level=info ts=2019-11-01T09:27:44.508687279Z caller=kubernetes.go:201 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2019-11-01T09:27:44.509311487Z caller=kubernetes.go:201 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2019-11-01T09:27:44.509814197Z caller=kubernetes.go:201 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2019-11-01T09:27:44.510303946Z caller=kubernetes.go:201 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2019-11-01T09:27:44.510798593Z caller=kubernetes.go:201 component="discovery manager scrape" discovery=k8s msg="Using pod service account via in-cluster config"
level=info ts=2019-11-01T09:27:44.511231031Z caller=main.go:722 msg="Completed loading of configuration file" filename=/prometheus/conf/prometheus.yml
level=info ts=2019-11-01T09:27:44.511242928Z caller=main.go:589 msg="Server is ready to receive web requests."
```
找到node port:
```bash
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl get node -o wide
NAME         STATUS    ROLES     AGE       VERSION    EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
10.19.0.11   Ready     master    14d       v1.10.13   <none>        CentOS Linux 7 (Core)   4.4.194-1.el7.elrepo.x86_64   docker://18.6.2
10.19.0.12   Ready     master    14d       v1.10.13   <none>        CentOS Linux 7 (Core)   4.4.194-1.el7.elrepo.x86_64   docker://18.6.2
10.19.0.21   Ready     node      14d       v1.10.13   <none>        CentOS Linux 7 (Core)   4.4.194-1.el7.elrepo.x86_64   docker://18.6.2
10.19.0.22   Ready     node      14d       v1.10.13   <none>        CentOS Linux 7 (Core)   4.4.194-1.el7.elrepo.x86_64   docker://18.6.2
[root@sh-saas-k8stest-master-dev-01 yaml]# kubectl get -n prometheus service prometheus -o wide
NAME         TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE       SELECTOR
prometheus   NodePort   10.248.255.13   <none>        9090:31304/TCP   6m        app=prometheus
```

现在可以通过http://10.19.0.21:31304/来访问了：
![](http://www.huilog.com/wp-content/uploads/2019/11/18d1a50e6d7420e91fc70ba607d9913d.png)

这个时候我们会发现，pods,ingress,deployment等监控数据都没有进来。这是因为这些监控是依赖另一个独立的应用：[kube-state-metrics](https://github.com/kubernetes/kube-state-metrics "kube-state-metrics")

# 3.部署kube-state-metrics
部署kube-state-metrics需要注意一下版本的兼容性：
![](http://www.huilog.com/wp-content/uploads/2019/11/f8e8d2d5418f3a583a685a07d8c39461.png)
我的k8s版本是1.10，这个版本可以使用kube-state-metrics 1.5的版本：

yaml被墙，从github上下载不下来，我干脆clone下来，clone可以，然后换到tag v1.5.0:
```
[root@sh-saas-k8stest-master-dev-01 yaml]# git clone https://github.com/kubernetes/kube-state-metrics.git
Cloning into 'kube-state-metrics'...
remote: Enumerating objects: 30, done.
remote: Counting objects: 100% (30/30), done.
remote: Compressing objects: 100% (22/22), done.
Receiving objects: 100% (16155/16155), 14.14 MiB | 194.00 KiB/s, done.
remote: Total 16155 (delta 10), reused 14 (delta 4), pack-reused 16125
Resolving deltas: 100% (9962/9962), done.
[root@sh-saas-k8stest-master-dev-01 yaml]# cd kube-state-metrics/
[root@sh-saas-k8stest-master-dev-01 kube-state-metrics]# git checkout 1.5.0
error: pathspec '1.5.0' did not match any file(s) known to git.
[root@sh-saas-k8stest-master-dev-01 kube-state-metrics]# git checkout v1.5.0
Note: checking out 'v1.5.0'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by performing another checkout.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -b with the checkout command again. Example:

  git checkout -b new_branch_name

HEAD is now at c888603... Merge branch 'master' into release-1.5
[root@sh-saas-k8stest-master-dev-01 kube-state-metrics]# cd kubernetes/
[root@sh-saas-k8stest-master-dev-01 kubernetes]# ls
kube-state-metrics-cluster-role-binding.yaml  kube-state-metrics-deployment.yaml    kube-state-metrics-role.yaml             kube-state-metrics-service.yaml
kube-state-metrics-cluster-role.yaml          kube-state-metrics-role-binding.yaml  kube-state-metrics-service-account.yaml
[root@sh-saas-k8stest-master-dev-01 kubernetes]# 
```

k8s官方镜像库也被墙了，需要换成一个镜像地址，镜像在docker官方的仓库里，地址为：https://hub.docker.com/r/mirrorgooglecontainers. 这是k8s官方仓库的一个镜像.
修改后的kube-state-metrics-deployment.yaml文件如下：
```bash
apiVersion: apps/v1
# Kubernetes versions after 1.9.0 should use apps/v1
# Kubernetes versions before 1.8.0 should use apps/v1beta1 or extensions/v1beta1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: kube-state-metrics
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: kube-state-metrics
    spec:
      serviceAccountName: kube-state-metrics
      containers:
      - name: kube-state-metrics
        image: mirrorgooglecontainers/kube-state-metrics:v1.5.0
        ports:
        - name: http-metrics
          containerPort: 8080
        - name: telemetry
          containerPort: 8081
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
      - name: addon-resizer
        image: mirrorgooglecontainers/addon-resizer:1.8.3
        resources:
          limits:
            cpu: 150m
            memory: 50Mi
          requests:
            cpu: 150m
            memory: 50Mi
        env:
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        command:
          - /pod_nanny
          - --container=kube-state-metrics
          - --cpu=100m
          - --extra-cpu=1m
          - --memory=100Mi
          - --extra-memory=2Mi
          - --threshold=5
          - --deployment=kube-state-metrics
```
然后再批量部署，就安装完成了：
```bash
[root@sh-saas-k8stest-master-dev-01 kubernetes]# kubectl apply -f ./
clusterrolebinding.rbac.authorization.k8s.io "kube-state-metrics" created
clusterrole.rbac.authorization.k8s.io "kube-state-metrics" created
deployment.apps "kube-state-metrics" created
rolebinding.rbac.authorization.k8s.io "kube-state-metrics" created
role.rbac.authorization.k8s.io "kube-state-metrics-resizer" created
serviceaccount "kube-state-metrics" created
service "kube-state-metrics" created
```

过一会就可以发现之前没有的监控指标都有了。
![](http://www.huilog.com/wp-content/uploads/2019/11/ad027554fa4ffc6d2e0e8f9dbe6e1fd4.png)

# 4. k8s节点node exporter部署
生成下面的daemonset配置文件：
vim node-exporter-ds.yml
```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
    k8s-app: node-exporter
    kubernetes.io/cluster-service: "true"
    version: v0.18.1
  name: node-exporter
  namespace: kube-system
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: node-exporter
      version: v0.18.1
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: node-exporter
        version: v0.18.1
    spec:
      containers:
      - args:
        - --log.level=info
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/rootfs
        - --collector.vmstat
        - --collector.vmstat.fields=.*
        - --collector.netstat
        - --collector.netstat.fields=.*
        - --collector.filesystem.ignored-mount-points=^/(proc|tmpfs|shm|sys|var/lib/docker/.+)($|/)
        - --collector.filesystem.ignored-fs-types=^(autofs|tmpfs|shm|binfmt_misc|bpf|cgroup2?|configfs|debugfs|devpts|devtmpfs|fusectl|hugetlbfs|mqueue|nsfs|overlay|proc|procfs|pstore|rpc_pipefs|securityfs|selinuxfs|squashfs|sysfs|tracefs)$
        image: prom/node-exporter:v0.18.1
        imagePullPolicy: IfNotPresent
        name: prometheus-node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: metrics
          protocol: TCP
        resources:
          limits:
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 50Mi
        securityContext:
          privileged: true
          runAsUser: 0
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /host/proc
          name: proc
          readOnly: true
        - mountPath: /host/sys
          name: sys
          readOnly: true
        - mountPath: /rootfs
          name: root
          readOnly: true
      dnsPolicy: ClusterFirst
      hostIPC: true
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      tolerations:
      - effect: NoSchedule
        key: kubernetes.io/role
        value: master
      volumes:
      - hostPath:
          path: /proc
          type: ""
        name: proc
      - hostPath:
          path: /sys
          type: ""
        name: sys
      - hostPath:
          path: /
          type: ""
        name: root
  templateGeneration: 16
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 1
    type: RollingUpdate
```
创建daemonset:
```bash
[root@sh-saas-k8stest-master-dev-01 prometheus]# kubectl apply -f node-exporter-ds.yml 
daemonset.extensions "node-exporter" created
```
过一会就能看到node exporter的监控信息了。