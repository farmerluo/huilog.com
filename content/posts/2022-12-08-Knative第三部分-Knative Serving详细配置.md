---
title: Knative第三部分：Knative Serving详细配置
author: 阿辉
date: 2022-12-08T15:15:14+00:00
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
knative 详细配置


#  配置服务的yaml

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  # 这是service名
  name: hello
spec:
  template:
    metadata:
      # 这是revision名，格式为{service-name}-{revision-name}
      name: hello-world-two
      # 软并发限制，并发瞬间变大时，可能超过此值
      annotations:
        # 自动扩缩容的类 "kpa.autoscaling.knative.dev" or "hpa.autoscaling.knative.dev"
        autoscaling.knative.dev/class: "kpa.autoscaling.knative.dev"
        # 自动扩缩容指标，"concurrency" ,"rps"或"cpu" cpu指标仅在具有 HPA,依赖上一个
        autoscaling.knative.dev/metric: "concurrency"
        # 上一个配置为concurrency表示并发超过200时进行扩容，上一个配置为rps表示超过200请求每秒时进行扩容，上一个配置为cpu表示cpu百分比
        autoscaling.knative.dev/target: "200"
        # 目标突发容量
        autoscaling.knative.dev/targetBurstCapacity: "200"
        # 目标利用率 允许达到硬限制的80%时进行扩建，但另外20%的流量还是发到此服务上
        autoscaling.knative.dev/targetUtilizationPercentage: "80"
        # 局部ingress入口类，会以局部为准，没有局部时用全局配置ConfigMap/config-network中的配置
        networking.knative.dev/ingress.class: <ingress-type>
        # 局部证书类入口，会以局部为准，没有局部时用全局配置ConfigMap/config-network中的配置，默认cert-manager.certificate.networking.knative.dev
        networking.knative.dev/certifcate.class: <certificate-provider>
        # 将流量逐步推出到修订版 先从1%开始，之后是与18%递增推进，是基于时间的，不与自动缩放子系统交互
        serving.knative.dev/rolloutDuration: "380s"
        # 每个修订版应具有的最小副本数  默认值： 0 如果启用缩放到零并使用类 KPA
        autoscaling.knative.dev/minScale: "3"
        # 每个修订应具有的最大副本数  0表示无限制
        autoscaling.knative.dev/maxScale: "3"
        # 修订在创建后必须立即达到的初始目标 创建 Revision 时，会自动选择初始比例和下限中较大的一个作为初始目标比例。默认： 1
        autoscaling.knative.dev/initialScale: "0"
        # 缩减延迟指定一个时间窗口，在应用缩减决策之前，该时间窗口必须以降低的并发性通过。
        autoscaling.knative.dev/scaleDownDelay: "15m"
        # 自动缩放配置模式--稳定窗口 在缩减期间，只有在稳定窗口的整个持续时间内没有任何流量到达修订版后，才会删除最后一个副本。
        autoscaling.knative.dev/window: "40s"
        # 自动缩放配置模式--紧急窗口 评估历史数据的窗口将如何缩小例如，值为10.0意味着在恐慌模式下，窗口将是稳定窗口大小的 10%。1.0~00.0
        autoscaling.knative.dev/panicWindowPercentage: "20.0"
        # 恐慌模式阈值 定义 Autoscaler 何时从稳定模式进入恐慌模式。 流量的百分比
        autoscaling.knative.dev/panicThresholdPercentage: "150.0"
        
    spec:
      # 硬限制，流量过大时，将多余的流量转到缓存层上
      containerConcurrency: 50
      containers:
        - image: gcr.io/knative-samples/helloworld-go
          ports:
            - containerPort: 8080
          env:
            - name: TARGET
              value: "World"
          resources:
            requests:
              cpu: 100m
              memory: 640M
            limits:
              cpu: 1
    traffic:
  - latestRevision: true
    percent: 80
  # hello-world为revision名
  - revisionName: hello-world
    percent: 20
    # 通过tag进行访问，访问地址 staging-<route name>.<namespace>.<domain>
    tag: staging

```

<!--more-->

# 配置自动扩容相关的yaml

ConfigMap/config-autoscaler

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: config-autoscaler
 namespace: knative-serving
data:
 # 全局并发默认数 100
 container-concurrency-target-default: "100"
 # 全局并发百分比
 container-concurrency-target-percentage: "0.7"
 # 修订在创建后必须立即达到的初始目标 需结合allow-zero-initial-scale
 initial-scale: "0"
 # 只有在使用 KnativePodAutoscaler (KPA) 时才能启用缩放为零
 enable-scale-to-zero: "true"
 # 放大率 确定所需 Pod 与现有 Pod 的最大比率。例如，具有的值2.0，修改只能从扩展N到2*N
 max-scale-up-rate: "1000"
 # 缩减率 确定现有 Pod 与所需 Pod 的最大比率。例如，具有的值2.0，修改只能从扩展N到N/2
 max-scale-down-rate: "2"
 # 扩展到零宽上限时间限制，标志决定了在 Autoscaler 决定将 pod 缩放为零后最后一个 pod 将保持活动状态的最长时间。
 scale-to-zero-grace-period: "30s"
 # 标志决定了在 Autoscaler 决定将 pod 缩放为零后最后一个 pod 将保持活动状态的最短时间。 "1m5s"
 scale-to-zero-pod-retention-period: "0s"
 # 自动缩放配置模式--稳定窗口 在缩减期间，只有在稳定窗口的整个持续时间内没有任何流量到达修订版后，才会删除最后一个副本。
 stable-window: "60s"
 # 自动缩放配置模式--紧急窗口 评估历史数据的窗口将如何缩小例如，值为10.0意味着在恐慌模式下，窗口将是稳定窗口大小的 10%。1.0~00.0
 panic-window-percentage: "10"
 # 恐慌模式阈值 定义 Autoscaler 何时从稳定模式进入恐慌模式。 流量的百分比
 panic-threshold-percentage: "200"
 target-burst-capacity: "200"
 # 配置每秒请求数 (RPS) 目标
 requests-per-second-target-default: "200"
 # 每个修订应具有的最大副本数
 max-scale: "3"
 max-scale-limit: "100"
 # 缩减延迟指定一个时间窗口，在应用缩减决策之前，该时间窗口必须以降低的并发性通过。
 scale-down-delay: "15m"

```

# 配置资源部署的yaml

ConfigMap/config-deployment

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: config-deployment
  namespace: knative-serving
  labels:
    serving.knative.dev/release: devel
  annotations:
    knative.dev/example-checksum: "fa67b403"
data:
  # This is the Go import path for the binary that is containerized
  # and substituted here.
  # 队列镜像
  queue-sidecar-image: ko://knative.dev/serving/cmd/queue
  # List of repositories for which tag to digest resolving should be skipped
  # 应跳过其标记的存储库列表,跳过标签解析
  registries-skipping-tag-resolving: "kind.local,ko.local,dev.local"
  # digest-resolution-timeout is the maximum time allowed for an image's
  # digests to be resolved.
  # 解析图像摘要所允许的最长时间
  digest-resolution-timeout: "10s"
  # progress-deadline is the duration we wait for the deployment to
  # be ready before considering it failed.
  # 在认为部署失败之前等待部署就绪的时间,初始化时间，pod达到规模的时间
  progress-deadline: "600s"
  # queue-sidecar-cpu-request is the requests.cpu to set for the queue proxy sidecar container.
  # If omitted, a default value (currently "25m"), is used.
  # 为队列代理sidecar容器设置的requests.cpu
  queue-sidecar-cpu-request: "25m"
  # queue-sidecar-cpu-limit is the limits.cpu to set for the queue proxy sidecar container.
  # If omitted, no value is specified and the system default is used.
  # 为队列代理sidecar容器设置的limits.cpu
  queue-sidecar-cpu-limit: "1000m"
  # queue-sidecar-memory-request is the requests.memory to set for the queue proxy container.
  # If omitted, no value is specified and the system default is used.
  # 要为队列代理容器设置的requests.memory
  queue-sidecar-memory-request: "400Mi"
  # queue-sidecar-memory-limit is the limits.memory to set for the queue proxy container.
  # If omitted, no value is specified and the system default is used.
  # 要为队列代理容器设置的limits.memory
  queue-sidecar-memory-limit: "800Mi"
  # queue-sidecar-ephemeral-storage-request is the requests.ephemeral-storage to
  # set for the queue proxy sidecar container.
  # If omitted, no value is specified and the system default is used.
  # 要为队列代理sidecar容器设置的requests.ephemeral-storage
  queue-sidecar-ephemeral-storage-request: "512Mi"
  # queue-sidecar-ephemeral-storage-limit is the limits.ephemeral-storage to set
  # for the queue proxy sidecar container.
  # If omitted, no value is specified and the system default is used.
  # 要为队列代理sidecar容器设置的limit.ephemeral-storage
  queue-sidecar-ephemeral-storage-limit: "1024Mi"

```



# 配置网络的yaml

ConfigMap/config-network

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: config-network
 namespace: knative-serving
data:
  # 将流量逐步推出到修订版,受影响的配置目标首先部署到 1% 的流量，然后以相同的增量步骤分配给其余的分配流量。基于时间的，不与自动缩放子系统交互。
  rollout-duration: "380s"  # Value in seconds.

```

# 配置使用kubernetes的属性

ConfigMap/config-features

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/part-of: knative-serving
    app.kubernetes.io/version: 1.0.0
    serving.knative.dev/release: v1.0.0
  name: config-features
  namespace: knative-serving
data:
  # 多容器支持，一个pod里多个容器
  multi-container: "enabled"
  # 节点亲和性
  kubernetes.podspec-affinity: "disabled"
  # host别名,添加hostts映射
  kubernetes.podspec-hostaliases: "disabled"
  # 节点选择器
  kubernetes.podspec-nodeselector: "disabled"
  # 容忍度
  kubernetes.podspec-tolerations: "disabled"
  # Kubernetes FieldRef支持，容器使用pod的spec属性依赖
  kubernetes.podspec-fieldref: "disabled"
  # Kubernetes RuntimeClassName支持
  kubernetes.podspec-runtimeclassname: "disabled"
  # 用户在Pod的SecurityContext上设置
  kubernetes.podspec-securitycontext: "disabled"
  # Kubernetes PriorityClassName支持
  kubernetes.podspec-priorityclassname: "disabled"
  # Kubernetes SchedulerName支持
  kubernetes.podspec-schedulername: "disabled"
  # 允许最终用户在Pod的SecurityContext上添加功能子集
  kubernetes.containerspec-addcapabilities: "disabled"
  # 试运行，验证webhook的podspec
  kubernetes.podspec-dryrun: "allowed"
  # 基于标记标头的路由功能
  tag-header-based-routing: "disabled"
  # http2自动检测
  autodetect-http2: "disabled"
  # 对EmptyDir的卷支持
  kubernetes.podspec-volumes-emptydir: "disabled"
  # 初始化容器支持
  kubernetes.podspec-init-containers: "disabled"

```



# 配置默认值

ConfigMap/config-defaults

```yaml
apiVersion:  v1
kind:  ConfigMap
metadata:
  name:  config-defaults
  namespace:  knative-serving
data:
  # 修订超时秒数,修订超时秒数包含用于修订的每个请求超时的默认秒数
  revision-timeout-seconds: "300"
  # 最大修订超时秒数，如果该值增加，则激活器的 terminateGraceTimeSeconds 也应增加以防止进行中的请求被中断
  max-revision-timeout-seconds: "600"
  # 修订 CPU 条件，包含默认情况下分配给修订的cpu分配
  revision-cpu-request: "400m"
  # 修订内存条件，包含默认情况下分配给修订的内存分配
  revision-memory-request: "100M"
  # 修订存储空间，包含默认情况下分配给修订的存储分配
  revision-ephemeral-storage-request: "500M"
  # 修订cpu限制包含默认情况下限制修订的cpu分配
  revision-cpu-limit: "1000m"
  # 修订内存限制包含默认情况下限制修订的内存分配
  revision-memory-limit: "200M"
  # 修订临时存储限制包含临时存储分配，以在默认情况下限制修订
  revision-ephemeral-storage-limit: "750M"
  # 容器名称模板 {{.Name}}
  container-name-template: "user-container"
  # 容器并发性指定容器一次可以处理的最大请求数，超过此阈值的请求将排队。将值设置为零将禁用此限制，并允许通过pod接收到的请求数
  container-concurrency: "0"
  # “容器并发最大限制”是一个运算符设置，确保单个修订不能具有任意大的并发值或自动缩放目标。容器并发默认设置必须等于或低于此值。
  container-concurrency-max-limit: "1000"
  # 允许容器并发零控制用户是否可以为containerConcurrency指定0（即无界）
  allow-container-concurrency-zero: "true"
  # k8s的service连接到容器
  enable-service-links: "false"

```

