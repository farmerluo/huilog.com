---
title: Knative第二部分-Knative Serving主要功能研究
author: 阿辉
date: 2022-12-08T15:10:14+08:00
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

# 1. 灰度发布

修改helloworld-go.yaml，将"Go Sample v1"改成"Go Sample v2":
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
          value: "Go Sample v2"
```

<!--more-->

让上面的配置生效，查看service的变化：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f helloworld-go.yaml 
service.serving.knative.dev/helloworld-go configured
[root@sh-saas-k8stest-master-dev-01 knative]# kn service describe  -n example helloworld-go
Name:       helloworld-go
Namespace:  example
Age:        14d
URL:        http://helloworld-go.example.example.com

Revisions:  
     ?  helloworld-go-00002 (latest created) [2] (12s)
        Image:  registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8 (at 032b80)
  100%  @latest (helloworld-go-00001) [1] (14d)
        Image:     registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8 (at 032b80)
        Replicas:  0/0

Conditions:  
  OK TYPE                   AGE REASON
  ?? Ready                  12s 
  ?? ConfigurationsReady    12s 
  ++ RoutesReady            14d 
[root@sh-saas-k8stest-master-dev-01 knative]# kn service describe  -n example helloworld-go
Name:       helloworld-go
Namespace:  example
Age:        14d
URL:        http://helloworld-go.example.example.com

Revisions:  
  100%  @latest (helloworld-go-00002) [2] (24s)
        Image:     registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8 (at 032b80)
        Replicas:  1/1

Conditions:  
  OK TYPE                   AGE REASON
  ++ Ready                  11s 
  ++ ConfigurationsReady    12s 
  ++ RoutesReady            11s 
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m1.707s
user    0m0.003s
sys     0m0.002s
```
可以看到Knative Service会自动做滚动发布。


kservice会自动生成一个revision:helloworld-go-00002，最终将流量从revision helloworld-go-00001切换到helloworld-go-00002：
```yaml
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get revisions.serving.knative.dev -n example 
NAME                  CONFIG NAME     K8S SERVICE NAME      GENERATION   READY   REASON   ACTUAL REPLICAS   DESIRED REPLICAS
helloworld-go-00001   helloworld-go   helloworld-go-00001   1            True             0                 0
helloworld-go-00002   helloworld-go   helloworld-go-00002   2            True             0                 0

[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get kservice helloworld-go -n example 
NAME            URL                                        LATESTCREATED         LATESTREADY           READY   REASON
helloworld-go   http://helloworld-go.example.example.com   helloworld-go-00002   helloworld-go-00002   True    
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl get kservice helloworld-go -n example  -o yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  creationTimestamp: "2021-08-10T10:34:14Z"
  name: helloworld-go
  namespace: example
  resourceVersion: "138660057"
  selfLink: /apis/serving.knative.dev/v1/namespaces/example/services/helloworld-go
  uid: 9522cb07-a15e-4995-a3e2-0389855e434d
spec:
  template:
    metadata:
      creationTimestamp: null
    spec:
      containerConcurrency: 0
      containers:
      - env:
        - name: TARGET
          value: Go Sample v2
        image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4dc8
        name: user-container
        readinessProbe:
          successThreshold: 1
          tcpSocket:
            port: 0
        resources: {}
      enableServiceLinks: false
      timeoutSeconds: 300
  traffic:
  - latestRevision: true
    percent: 100
status:
  address:
    url: http://helloworld-go.example.svc.cluster.local
  conditions:
  - lastTransitionTime: "2021-08-25T07:54:31Z"
    status: "True"
    type: ConfigurationsReady
  - lastTransitionTime: "2021-08-25T07:54:32Z"
    status: "True"
    type: Ready
  - lastTransitionTime: "2021-08-25T07:54:32Z"
    status: "True"
    type: RoutesReady
  latestCreatedRevisionName: helloworld-go-00002
  latestReadyRevisionName: helloworld-go-00002
  observedGeneration: 2
  traffic:
  - latestRevision: true
    percent: 100
    revisionName: helloworld-go-00002
  url: http://helloworld-go.example.example.com
```


# 1.1 按流量比例灰度发布

修改helloworld-go.yaml：
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
          value: "Go Sample v2"
  traffic:
  - percent: 50 
    revisionName: helloworld-go-00001
  - percent: 50 
    revisionName: helloworld-go-00002
```

测试：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# cat helloworld-go.yaml 
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
          value: "Go Sample v2"
  traffic:
  - percent: 50 
    revisionName: helloworld-go-00001
  - percent: 50 
    revisionName: helloworld-go-00002
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m4.322s
user    0m0.002s
sys     0m0.002s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m0.015s
user    0m0.003s
sys     0m0.002s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m20.436s
user    0m0.002s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m0.007s
user    0m0.001s
sys     0m0.003s
```


# 1.2 基于Tag(子域名)金丝雀灰度发布

修改helloworld-go.yaml，增加tag：
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
          value: "Go Sample v2"
  traffic:
  - percent: 50 
    revisionName: helloworld-go-00001
    tag: a
  - percent: 50 
    revisionName: helloworld-go-00002
    tag: b
```

应用配置测试，可以发现正常的域名访问，还是基于流量比例来的：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f ./helloworld-go.yaml 
service.serving.knative.dev/helloworld-go configured
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m15.506s
user    0m0.003s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m20.202s
user    0m0.002s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m0.007s
user    0m0.001s
sys     0m0.003s
```
除了上面的访问方式，还可以使用"tag名-域名"的访问来访问，这样就可以使用特定的域名来测试特定的版本。如：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: a-helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m0.006s
user    0m0.002s
sys     0m0.002s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: b-helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m0.007s
user    0m0.001s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: c-helloworld-go.example.example.com" http://10.248.132.176

real    0m0.018s
user    0m0.002s
sys     0m0.002s
```

# 1.3 基于http header金丝雀灰度发布

基于http header金丝雀灰度发布的需要Knative 0.16以上版本才支持，并且默认是关的，需要使用以下命令手动开启：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl patch cm config-features -n knative-serving -p '{"data":{"tag-header-based-routing":"Enabled"}}'
configmap/config-features patched
```

helloworld-go.yaml跟上面一样，不需要做任何修改：
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
          value: "Go Sample v2"
  traffic:
  - percent: 50 
    revisionName: helloworld-go-00001
    tag: a
  - percent: 50 
    revisionName: helloworld-go-00002
    tag: b
```
我们下面直接测试就行：

通过下面的测试，我们发现基于tag-子域名的访问方式依然是可以使用的。
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: b-helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v2!

real    0m16.141s
user    0m0.002s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" http://10.248.132.176
Hello Go Sample v1!

real    0m20.507s
user    0m0.002s
sys     0m0.003s
```

通过下面的测试，我们发现可以通过http header头来访问，注意http header的名称为：Knative-Serving-Tag

```bash
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" -H "Knative-Serving-Tag:a" http://10.248.132.176
Hello Go Sample v1!

real    0m0.007s
user    0m0.002s
sys     0m0.002s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.example.example.com" -H "Knative-Serving-Tag:b" http://10.248.132.176
Hello Go Sample v2!

real    0m0.006s
user    0m0.002s
sys     0m0.002s
```

# 2. 配置自定义域名



# 3. 通过VirtualService来配置更多域名的访问

我们也可以通过编写下面的VirtualService，配置自己的域名及header等条件，实现灰度的功能或访问

## 3.1 使用rewrite模式

我们推荐使用VirtualService rewrite的功能来实现不同的域名，不同的headers及url等来实现不同的路由功能。
如下面的配置，我们通过header wid实现了一个灰度功能，并且使用了一个外部域名。
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: helloworld-go-rewrite-ingress
  namespace: example
spec:
  gateways:
  - knative-serving/knative-ingress-gateway
  #- knative-serving/knative-local-gateway
  hosts:
  - helloworld-go.weimobb.com
  http:
  - match:
    - headers:
        wid:
          prefix: a
    rewrite:
      authority: a-helloworld-go.example.example.com
    route:
      - destination:
          host: istio-ingressgateway.istio-system.svc.cluster.local
          port:
            number: 80
        weight: 100
  - match:
    - headers:
        wid:
          prefix: b
    rewrite:
      authority: b-helloworld-go.example.example.com
    route:
      - destination:
          host: istio-ingressgateway.istio-system.svc.cluster.local
          port:
            number: 80
        weight: 100
```

我们看看测试效果，可以发现是符合预期的。
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# kubectl apply -f ./helloworld-go-rewrite.yaml 
virtualservice.networking.istio.io/helloworld-go-rewrite-ingress configured
[root@sh-saas-k8stest-master-dev-01 knative]# time curl  -i -L -H "Host: helloworld-go.weimobb.com" -H "wid: a"  http://10.248.132.176
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8
date: Thu, 02 Sep 2021 03:05:50 GMT
x-envoy-upstream-service-time: 10342
server: istio-envoy

Hello Go Sample v1!

real    0m10.364s
user    0m0.002s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl  -i -L -H "Host: helloworld-go.weimobb.com" -H "wid: b"  http://10.248.132.176
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8
date: Thu, 02 Sep 2021 03:06:07 GMT
x-envoy-upstream-service-time: 20031
server: istio-envoy

Hello Go Sample v2!

real    0m20.052s
user    0m0.001s
sys     0m0.004s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl  -i -L -H "Host: helloworld-go.weimobb.com" -H "wid: b"  http://10.248.132.176
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8
date: Thu, 02 Sep 2021 03:06:25 GMT
x-envoy-upstream-service-time: 3
server: istio-envoy

Hello Go Sample v2!

real    0m0.009s
user    0m0.001s
sys     0m0.004s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl  -i -L -H "Host: helloworld-go.weimobb.com" -H "wid: a"  http://10.248.132.176
HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8
date: Thu, 02 Sep 2021 03:06:27 GMT
x-envoy-upstream-service-time: 2
server: istio-envoy

Hello Go Sample v1!

real    0m0.009s
user    0m0.004s
sys     0m0.002s
```

## 3.2 使用redirect模式

也可以通过redirect来做，如以下VirtualService配置：
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld-go-redirect-ingress
  namespace: example
spec:
  gateways:
  - knative-serving/knative-ingress-gateway
  - knative-serving/knative-local-gateway
  hosts:
  - helloworld-go.weimoba.com
  http:
  - match:
    - authority:
        prefix: helloworld-go.weimoba.com 
    redirect:
      authority: a-helloworld-go.example.example.com
```
看看效果：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# echo "10.248.132.176 a-helloworld-go.example.example.com" >> /etc/hosts
[root@sh-saas-k8stest-master-dev-01 knative]# time curl  -i -L -H "Host: helloworld-go.weimoba.com"  http://10.248.132.176
HTTP/1.1 301 Moved Permanently
location: http://a-helloworld-go.example.example.com/
date: Wed, 01 Sep 2021 10:30:23 GMT
server: istio-envoy
content-length: 0

HTTP/1.1 200 OK
content-length: 20
content-type: text/plain; charset=utf-8
date: Wed, 01 Sep 2021 10:30:23 GMT
x-envoy-upstream-service-time: 2
server: istio-envoy

Hello Go Sample v1!

real    0m0.019s
user    0m0.003s
sys     0m0.002s
```

## 3.3 使用直接到应用的模式

我们也可以通过配置自己的域名及header等条件直接转发到应用对应的服务上，以实现灰度的功能或访问：
```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: helloworld-go-weimob-com-ingress
  namespace: example
spec:
  gateways:
  - knative-serving/knative-ingress-gateway
  - knative-serving/knative-local-gateway
  hosts:
  - helloworld-go.weimob.com
  http:
  - match:
    - authority:
        prefix: helloworld-go.weimob.com
      gateways:
      - knative-serving/knative-ingress-gateway
      headers:
        wid:
          exact: a
    retries: {}
    route:
    - destination:
        host: helloworld-go-00001.example.svc.cluster.local
        port:
          number: 80
      headers:
        request:
          set:
            Knative-Serving-Namespace: example
            Knative-Serving-Revision: helloworld-go-00001
      weight: 100
  - match:
    - authority:
        prefix: helloworld-go.weimob.com
      gateways:
      - knative-serving/knative-ingress-gateway
      headers:
        wid:
          exact: b
    retries: {}
    route:
    - destination:
        host: helloworld-go-00002.example.svc.cluster.local
        port:
          number: 80
      headers:
        request:
          set:
            Knative-Serving-Namespace: example
            Knative-Serving-Revision: helloworld-go-00002
      weight: 100
```

查看效果，可以看到是预期内的：
```bash
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.weimob.com" -H "wid:b" http://10.248.132.176
Hello Go Sample v2!

real    0m20.578s
user    0m0.004s
sys     0m0.002s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.weimob.com" -H "wid:a" http://10.248.132.176
Hello Go Sample v1!

real    0m4.722s
user    0m0.002s
sys     0m0.003s
[root@sh-saas-k8stest-master-dev-01 knative]# time curl -H "Host: helloworld-go.weimob.com"  http://10.248.132.176

real    0m0.006s
user    0m0.002s
sys     0m0.003s
```



PS: 解释一下VirtualService中authority的意思，它是访问URI中//到/之间的部分。如URI为"http://helloworld-go.weimob.com:8080/aaa"，则http authority为helloworld-go.weimob.com:8080。

根据rfc3986规范的组成语法，URI 进行拆解如下：
```
foo://example.com:8042/over/there?name=ferret#nose
\_/   \______________/\_________/ \_________/ \__/
 |           |            |            |        |
scheme   authority       path        query   fragment
 |   _____________________|__
/ \ /                        \
urn:example:animal:ferret:nose
```

参考：https://zhuanlan.zhihu.com/p/216488873

参考：https://github.com/knative/docs/blob/mkdocs/docs/serving/samples/knative-routing-go/routing-internal.yaml