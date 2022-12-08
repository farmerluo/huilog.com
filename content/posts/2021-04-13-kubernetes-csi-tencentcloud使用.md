---
title: kubernetes-csi-tencentcloud使用
author: 阿辉
date: 2021-04-13T16:10:14+08:00
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
---
{{< toc >}}
# 1.概述
kubernetes-csi-tencentcloud官方地址：[https://github.com/TencentCloud/kubernetes-csi-tencentcloud](https://github.com/TencentCloud/kubernetes-csi-tencentcloud)

官方存储类参考：[https://kubernetes.io/zh/docs/concepts/storage/storage-classes/](https://kubernetes.io/zh/docs/concepts/storage/storage-classes/)

目前支持CBS,CFS,COS这三种存储类型，其中CBS不支持共享模式，CFS支持共享模式。

<!--more-->

# 2. CBS存储 CSI

## 2.1 CBS存储配置

namespace.yaml:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: k8s-csi-tencentcloud
```

secret.yaml:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-tencentcloud
  namespace: k8s-csi-tencentcloud
data:
  # value need base64 encoding
  #   echo -n "<SECRET_ID>" | base64
  TENCENTCLOUD_CBS_API_SECRET_ID: "xxxxxxx"
  TENCENTCLOUD_CBS_API_SECRET_KEY: "xxxxxx="
```

csidriver.yaml:
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: csidrivers.csi.storage.k8s.io
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  group: csi.storage.k8s.io
  names:
    kind: CSIDriver
    plural: csidrivers
  scope: Cluster
  validation:
    openAPIV3Schema:
      properties:
        spec:
          description: Specification of the CSI Driver.
          properties:
            attachRequired:
              description: Indicates this CSI volume driver requires an attach operation,
                and that Kubernetes should call attach and wait for any attach operation
                to complete before proceeding to mount.
              type: boolean
            podInfoOnMountVersion:
              description: Indicates this CSI volume driver requires additional pod
                information (like podName, podUID, etc.) during mount operations.
              type: string
  version: v1alpha1
```

csi-controller-rbac.yaml:
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: csi-controller-sa
  namespace: k8s-csi-tencentcloud
---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-provisioner-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["csinodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots"]
    verbs: ["get", "list"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents"]
    verbs: ["get", "list"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-provisioner-binding
subjects:
  - kind: ServiceAccount
    name: csi-controller-sa
    namespace: k8s-csi-tencentcloud
roleRef:
  kind: ClusterRole
  name: csi-provisioner-role
  apiGroup: rbac.authorization.k8s.io

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-attacher-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["csi.storage.k8s.io"]
    resources: ["csinodeinfos"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["volumeattachments"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-attacher-binding
subjects:
  - kind: ServiceAccount
    name: csi-controller-sa
    namespace: k8s-csi-tencentcloud
roleRef:
  kind: ClusterRole
  name: csi-attacher-role
  apiGroup: rbac.authorization.k8s.io

---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-snapshotter-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents"]
    verbs: ["create", "get", "list", "watch", "update", "delete"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["apiextensions.k8s.io"]
    resources: ["customresourcedefinitions"]
    verbs: ["create", "list", "watch", "delete"]
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshotcontents/status"]
    verbs: ["update"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "watch", "list", "delete", "update", "create"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-snapshotter-binding
subjects:
  - kind: ServiceAccount
    name: csi-controller-sa
    namespace: k8s-csi-tencentcloud
roleRef:
  kind: ClusterRole
  name: csi-snapshotter-role
  apiGroup: rbac.authorization.k8s.io
---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-resizer-role
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "update", "patch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims/status"]
    verbs: ["update", "patch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-resizer-binding
subjects:
  - kind: ServiceAccount
    name: csi-controller-sa
    namespace: k8s-csi-tencentcloud
roleRef:
  kind: ClusterRole
  name: csi-resizer-role
  apiGroup: rbac.authorization.k8s.io

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-secret-role
  namespace: k8s-csi-tencentcloud
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-secret-binding
  namespace: k8s-csi-tencentcloud
subjects:
  - kind: ServiceAccount
    name: csi-controller-sa
    namespace: k8s-csi-tencentcloud
roleRef:
  kind: ClusterRole
  name: csi-secret-role
  apiGroup: rbac.authorization.k8s.io
```

csi-node-rbac.yaml:
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: csi-node-sa
  namespace: k8s-csi-tencentcloud

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-node-role
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: csi-node-binding
subjects:
  - kind: ServiceAccount
    name: csi-node-sa
    namespace: k8s-csi-tencentcloud
roleRef:
  kind: ClusterRole
  name: csi-node-role
  apiGroup: rbac.authorization.k8s.io
```
csi-controller.yaml:
```yaml
---
kind: Deployment
apiVersion: apps/v1
metadata:
  name: csi-cbsplugin-controller
  namespace: k8s-csi-tencentcloud
spec:
  replicas: 2
  selector:
    matchLabels:
      app: csi-cbsplugin-controller
  template:
    metadata:
      labels:
        app: csi-cbsplugin-controller
    spec:
      hostNetwork: true
      serviceAccountName: csi-controller-sa
      priorityClassName: system-cluster-critical
      containers:
        - name: csi-provisioner
          #image: quay.io/k8scsi/csi-provisioner:v1.6.0
          image: ccr.ccs.tencentyun.com/weimob-k8scsi/csi-provisioner:v1.6.0
          args:
            - "--feature-gates=Topology=true"
            - "--csi-address=$(ADDRESS)"
            - "--v=5"
            - "--timeout=120s"
            - "--enable-leader-election"
            - "--leader-election-type=leases"
          env:
            - name: ADDRESS
              value: /csi/csi.sock
          volumeMounts:
            - mountPath: /csi
              name: socket-dir
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 100Mi
        - name: csi-attacher
          #image: quay.io/k8scsi/csi-attacher:v2.2.0
          image: ccr.ccs.tencentyun.com/weimob-k8scsi/csi-attacher:v2.2.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=5"
            - "--leader-election=true"
          env:
            - name: ADDRESS
              value: /csi/csi.sock
          volumeMounts:
            - mountPath: /csi
              name: socket-dir
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 100Mi
        - name: csi-snapshotter
          #image: quay.io/k8scsi/csi-snapshotter:v1.2.2
          image: ccr.ccs.tencentyun.com/weimob-k8scsi/csi-snapshotter:v1.2.2
          args:
            - "--csi-address=$(ADDRESS)"
            - "--leader-election=true"
            - "--v=5"
          env:
            - name: ADDRESS
              value: /csi/csi.sock
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 100Mi
        - name: csi-resizer
          #image: quay.io/k8scsi/csi-resizer:v0.5.0
          image: ccr.ccs.tencentyun.com/weimob-k8scsi/csi-resizer:v0.5.0
          args:
            - "--csi-address=$(ADDRESS)"
            - "--v=5"
            - "--leader-election=true"
          env:
            - name: ADDRESS
              value: /csi/csi.sock
          volumeMounts:
            - name: socket-dir
              mountPath: /csi
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 100Mi
        - name: csi-cbsplugin
          image: ccr.ccs.tencentyun.com/k8scsi/csi-tencentcloud-cbs:v1.2.0
          command:
          - "/csi-tencentcloud-cbs"
          args:
          - "--v=5"
          - "--logtostderr=true"
          - "--endpoint=$(ADDRESS)"
          env:
            - name: ADDRESS
              value: unix:///csi/csi.sock
            - name: TENCENTCLOUD_CBS_API_SECRET_ID
              valueFrom:
                secretKeyRef:
                  name: csi-tencentcloud
                  key: TENCENTCLOUD_CBS_API_SECRET_ID
            - name: TENCENTCLOUD_CBS_API_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: csi-tencentcloud
                  key: TENCENTCLOUD_CBS_API_SECRET_KEY
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          volumeMounts:
            - mountPath: /csi
              name: socket-dir
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 100m
              memory: 100Mi
      volumes:
        - name: socket-dir
          emptyDir: {}
```

csi-node.yaml:
```
---
kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: csi-cbsplugin-node
  namespace: k8s-csi-tencentcloud
spec:
  selector:
    matchLabels:
      app: csi-cbsplugin-node
  template:
    metadata:
      labels:
        app: csi-cbsplugin-node
    spec:
      serviceAccount: csi-node-sa
      hostNetwork: true
      hostPID: true
      dnsPolicy: ClusterFirst
      containers:
        - name: driver-registrar
          #image: quay.io/k8scsi/csi-node-driver-registrar:v1.1.0
          image: ccr.ccs.tencentyun.com/weimob-k8scsi/csi-node-driver-registrar:v1.1.0
          args:
            - "--v=5"
            - "--csi-address=/csi/csi.sock"
            - "--kubelet-registration-path=/data/kubernetes/kubelet/plugins/com.tencent.cloud.csi.cbs/csi.sock"
          lifecycle:
            preStop:
              exec:
                command: [
                  "/bin/sh", "-c",
                  "rm -rf /registration/com.tencent.cloud.csi.cbs \
                  /registration/com.tencent.cloud.csi.cbs-reg.sock"
                ]
          env:
            - name: KUBE_NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          volumeMounts:
            - name: plugin-dir
              mountPath: /csi
            - name: registration-dir
              mountPath: /registration
        - name: csi-cbsplugin-node
          securityContext:
            privileged: true
            capabilities:
              add: ["SYS_ADMIN"]
            allowPrivilegeEscalation: true
          image: ccr.ccs.tencentyun.com/k8scsi/csi-tencentcloud-cbs:v1.2.0
          command:
          - "/csi-tencentcloud-cbs"
          args:
          - "--v=5"
          - "--logtostderr=true"
          - "--endpoint=unix:///csi/csi.sock"
          env:
            - name: TENCENTCLOUD_CBS_API_SECRET_ID
              valueFrom:
                secretKeyRef:
                  name: csi-tencentcloud
                  key: TENCENTCLOUD_CBS_API_SECRET_ID
            - name: TENCENTCLOUD_CBS_API_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: csi-tencentcloud
                  key: TENCENTCLOUD_CBS_API_SECRET_KEY
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          imagePullPolicy: "Always"
          volumeMounts:
            - name: plugin-dir
              mountPath: /csi
            - name: pods-mount-dir
              mountPath: /data/kubernetes/kubelet/pods
              mountPropagation: "Bidirectional"
            - name: plugin-mount-dir
              mountPath: /data/kubernetes/kubelet/plugins/kubernetes.io/csi/volumeDevices/
              mountPropagation: "Bidirectional"
            - mountPath: /dev
              name: host-dev
            - mountPath: /rootfs
              name: host-rootfs
            - mountPath: /sys
              name: host-sys
            - mountPath: /lib/modules
              name: lib-modules
              readOnly: true
      volumes:
        - name: plugin-dir
          hostPath:
            path: /data/kubernetes/kubelet/plugins/com.tencent.cloud.csi.cbs
            type: DirectoryOrCreate
        - name: plugin-mount-dir
          hostPath:
            path: /data/kubernetes/kubelet/plugins/kubernetes.io/csi/volumeDevices/
            type: DirectoryOrCreate
        - name: registration-dir
          hostPath:
            path: /data/kubernetes/kubelet/plugins_registry/
            type: Directory
        - name: pods-mount-dir
          hostPath:
            path: /data/kubernetes/kubelet/pods
            type: Directory
        - name: host-dev
          hostPath:
            path: /dev
        - name: host-rootfs
          hostPath:
            path: /
        - name: host-sys
          hostPath:
            path: /sys
        - name: lib-modules
          hostPath:
            path: /lib/modules
```
创建以上这些配置，看到命名空间内的pod都有正常启动，说明基本上正常了。
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl apply -f ./

[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pod -n k8s-csi-tencentcloud 
NAME                                        READY   STATUS    RESTARTS   AGE
csi-cbsplugin-controller-57cc977876-86969   5/5     Running   11         47h
csi-cbsplugin-controller-57cc977876-p5sn5   5/5     Running   11         47h
csi-cbsplugin-node-k2sjv                    2/2     Running   0          28h
csi-cbsplugin-node-n66mt                    2/2     Running   0          28h
```
有问题可以查看相应的日志。

最后我们再创建一些存储类：
storageclass.yaml:
```
# kind: StorageClass
# apiVersion: storage.k8s.io/v1
# metadata:
#   name: cbs-basic
# provisioner: com.tencent.cloud.csi.cbs
# parameters:
#   diskType: CLOUD_BASIC
#   diskZone: ap-shanghai-4
#   #allowedTopologies: ap-shanghai-1

---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cbs-premium
provisioner: com.tencent.cloud.csi.cbs
allowVolumeExpansion: true
parameters:
  diskType: CLOUD_PREMIUM
  diskZone: ap-shanghai-4
  #allowedTopologies: ap-shanghai-1

---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cbs-ssd
provisioner: com.tencent.cloud.csi.cbs
allowVolumeExpansion: true
parameters:
  diskType: CLOUD_SSD
  diskZone: ap-shanghai-4
  #allowedTopologies: ap-shanghai-1

# ---
# kind: StorageClass
# apiVersion: storage.k8s.io/v1
# metadata:
#   name: cbs-basic-prepaid
# provisioner: com.tencent.cloud.csi.cbs
# parameters:
#   diskType: CLOUD_BASIC
#   diskChargeType: PREPAID
#   diskChargeTypePrepaidPeriod: "1"
#   diskChargePrepaidRenewFlag: NOTIFY_AND_AUTO_RENEW

---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cbs-premium-prepaid
provisioner: com.tencent.cloud.csi.cbs
allowVolumeExpansion: true
parameters:
  diskType: CLOUD_PREMIUM
  diskChargeType: PREPAID
  diskChargeTypePrepaidPeriod: "1"
  diskChargePrepaidRenewFlag: NOTIFY_AND_AUTO_RENEW

---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cbs-ssd-prepaid
provisioner: com.tencent.cloud.csi.cbs
allowVolumeExpansion: true
parameters:
  diskType: CLOUD_SSD
  diskChargeType: PREPAID
  diskChargeTypePrepaidPeriod: "1"
  diskChargePrepaidRenewFlag: NOTIFY_AND_AUTO_RENEW
```
创建存储类：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl apply -f storageclass-examples.yaml
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get storageclasses.storage.k8s.io 
NAME                  PROVISIONER                 RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
cbs-premium           com.tencent.cloud.csi.cbs   Delete          Immediate           false                  27h
cbs-premium-prepaid   com.tencent.cloud.csi.cbs   Delete          Immediate           false                  27h
cbs-ssd               com.tencent.cloud.csi.cbs   Delete          Immediate           false                  27h
cbs-ssd-prepaid       com.tencent.cloud.csi.cbs   Delete          Immediate           false                  27h
```
这样就配置完成，后面可以进行使用测试。

## 2.2 CBS存储statefulset测试
创建一个测试的statefulset.yaml:
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  generation: 5
  labels:
    app: csi-stateful-app
  name: csi-stateful-app
spec:
  podManagementPolicy: OrderedReady
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: csi-stateful-app
  serviceName: csi-stateful-app
  template:
    metadata:
      labels:
        app: csi-stateful-app
    spec:
      automountServiceAccountToken: true
      containers:
        - name: csi
          image: busybox
          volumeMounts:
          - mountPath: "/data"
            name: data
          command: [ "sleep", "1000000" ]
  volumeClaimTemplates:
  - metadata:
      name: data 
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "cbs-premium"
      resources:
        requests:
          storage: 10Gi
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
```

创建statefulSet:
```
kubectl apply -f ./statefulset.yaml
```
查看POD和PVC：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pod -o wide | grep csi
csi-stateful-app-0                      2/2     Running            0          10m    10.248.1.158   10.19.0.22   <none>           <none>

[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pvc
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-csi-stateful-app-0   Bound    pvc-d284594c-b2bf-41e8-b066-7b075d931273   10Gi       RWO            cbs-premium    7m57s
```
去腾讯云控制台，可以看到自动创建了一块cbs的云盘，命令为pvc开头，并挂到了10.19.0.22这台节点上。

现在将csi-stateful-app扩容到2个pod：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl scale --replicas=2 statefulset csi-stateful-app 
statefulset.apps/csi-stateful-app scaled
```

可以看到又创建了一块云盘并挂上了：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pod -o wide | grep csi
csi-stateful-app-0                      2/2     Running            0          14m    10.248.1.158   10.19.0.22   <none>           <none>
csi-stateful-app-1                      2/2     Running            0          54s    10.248.1.160   10.19.0.22   <none>           <none>
[root@sh-saas-k8stest-master-dev-01 examples]#  kubectl get pvc
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-csi-stateful-app-0   Bound    pvc-d284594c-b2bf-41e8-b066-7b075d931273   10Gi       RWO            cbs-premium    15m
data-csi-stateful-app-1   Bound    pvc-e2f0b093-acdf-45e3-8ceb-bdcdcba62a89   10Gi       RWO            cbs-premium    10m
```

现在将csi-stateful-app缩容到1个pod：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl scale --replicas=1 statefulset csi-stateful-app 
statefulset.apps/csi-stateful-app scaled
```
可以看到POD只有一个了，但是PVC还是有2块，下次再扩容时将会用上，同时可以看到在腾讯云控制台上，data-csi-stateful-app-1对应的这块盘现在没有挂任何节点：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pod -o wide | grep csi
csi-stateful-app-0                      2/2     Running            0          16m    10.248.1.158   10.19.0.22   <none>           <none>
[root@sh-saas-k8stest-master-dev-01 examples]#  kubectl get pvc
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-csi-stateful-app-0   Bound    pvc-d284594c-b2bf-41e8-b066-7b075d931273   10Gi       RWO            cbs-premium    16m
data-csi-stateful-app-1   Bound    pvc-e2f0b093-acdf-45e3-8ceb-bdcdcba62a89   10Gi       RWO            cbs-premium    12m
```
现在将statefulsets.apps csi-stateful-app 删除：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl delete statefulsets.apps csi-stateful-app 
statefulset.apps "csi-stateful-app" deleted
```

可以看到POD都没有了，但是PVC还保留没有删除，需要手动删除PVC：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pod -o wide | grep csi
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pvc
NAME                      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-csi-stateful-app-0   Bound    pvc-d284594c-b2bf-41e8-b066-7b075d931273   10Gi       RWO            cbs-premium    22m
data-csi-stateful-app-1   Bound    pvc-e2f0b093-acdf-45e3-8ceb-bdcdcba62a89   10Gi       RWO            cbs-premium    17m
```
手动删除PVC：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl delete pvc data-csi-stateful-app-0
persistentvolumeclaim "data-csi-stateful-app-0" deleted
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl delete pvc data-csi-stateful-app-1
persistentvolumeclaim "data-csi-stateful-app-1" deleted
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pvc
No resources found in default namespace.
```
再看腾讯云上这2块云盘，可以发现已经不存在了。

## 2.3 CBS存储在线扩容

当我们直接编辑statefulsets csi-stateful-app的volumeClaimTemplates,将之前的10G，改到20G:
```yaml
  volumeClaimTemplates:
  - apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      creationTimestamp: null
      name: data
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 20Gi
      storageClassName: cbs-premium
      volumeMode: Filesystem
```

保存时会发现报错：
```
# statefulsets.apps "csi-stateful-app" was not valid:
# * spec: Forbidden: updates to statefulset spec for fields other than 'replicas', 'template', and 'updateStrategy' are forbidden
#
```

所以不能在这里改，只能修改PVC:
```
kubectl edit  pvc data-csi-stateful-app-1
```
将下面的10G改成20G，并保存退出，发现并没有报错：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  labels:
    app: csi-stateful-app
  name: data-csi-stateful-app-1
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: cbs-premium
  volumeMode: Filesystem
  volumeName: pvc-6de9cb4b-b7fb-48c5-a1a7-9409df5b6028
```

可以看到已经改成20G了：
```
[root@sh-saas-k8stest-master-dev-01 examples]# kubectl get pvc | grep data-csi-stateful-app-1
data-csi-stateful-app-1   Bound    pvc-6de9cb4b-b7fb-48c5-a1a7-9409df5b6028   20Gi       RWO            cbs-premium    11m

[root@sh-saas-k8stest-master-dev-01 examples]# kubectl exec -it csi-stateful-app-1 -- sh
Defaulting container name to csi.
Use 'kubectl describe pod/csi-stateful-app-1 -n default' to see all of the containers in this pod.
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                 196.7G      4.1G    182.7G   2% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                     3.8G         0      3.8G   0% /sys/fs/cgroup
/dev/vdd                 19.6G     44.0M     19.5G   0% /data
/dev/vdb1               196.7G      4.1G    182.7G   2% /dev/termination-log
/dev/vdb1               196.7G      4.1G    182.7G   2% /etc/resolv.conf
/dev/vdb1               196.7G      4.1G    182.7G   2% /etc/hostname
/dev/vdb1               196.7G      4.1G    182.7G   2% /etc/hosts
shm                      64.0M      4.0K     64.0M   0% /dev/shm
tmpfs                     3.8G     12.0K      3.8G   0% /var/run/secrets/kubernetes.io/serviceaccount
tmpfs                     3.8G         0      3.8G   0% /proc/acpi
tmpfs                    64.0M         0     64.0M   0% /proc/kcore
tmpfs                    64.0M         0     64.0M   0% /proc/keys
tmpfs                    64.0M         0     64.0M   0% /proc/timer_list
tmpfs                    64.0M         0     64.0M   0% /proc/sched_debug
tmpfs                     3.8G         0      3.8G   0% /proc/scsi
tmpfs                     3.8G         0      3.8G   0% /sys/firm
```
如果扩容不成功，可以重启POD试一下，因为可能不一定会支持在线热扩容。

另注意：此功能仅可用于扩容卷，不能用于缩小卷。

# 3. CFS存储 CSI

CFS为腾讯云的一个类似NFS的服务，支持共享读写。


# 4. COS存储 CSI

COS存储CSI是以FUSE的方式挂载一个基于COS存储的盘，也支持共享读写。其使用方式有一定局限性，暂时还不支持存储类，只支持手动创建PV的方式使用。

使用COSFS以下需特别注意：
> COSFS 基于 S3FS 构建， 读取和写入操作都经过磁盘中转，仅适合挂载后对文件进行简单的管理，不支持本地文件系统的一些功能用法，性能方面也无法代替云硬盘 CBS 或文件存储 CFS。 需注意以下不适用的场景，例如：
> 
> - 随机或者追加写文件会导致整个文件的下载以及重新上传，您可以使用与 Bucket 在同一个地域的 CVM 加速文件的上传下载。
> - 多个客户端挂载同一个 COS 存储桶时，依赖用户自行协调各个客户端的行为。例如避免多个客户端写同一个文件等。
> - 文件/文件夹的 rename 操作不是原子的。
> - 元数据操作，例如 list directory，性能较差，因为需要远程访问 COS 服务器。
> - 不支持 hard link，不适合高并发读/写的场景。
> - 不可以同时在一个挂载点上挂载、和卸载文件。您可以先使用 cd 命令切换到其他目录，再对挂载点进行挂载、卸载操作。

更多可参考：
[https://cloud.tencent.com/document/product/436/6883](https://cloud.tencent.com/document/product/436/6883)