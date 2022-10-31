---
title: k8s创建集群只读service account
author: 阿辉
date: 2019-06-04T07:39:11+00:00
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

有时需要在k8s 集群上给比如开发人员创建一个只读的service account，在这里记录一下创建方法：

先创建oms-viewonly.yaml:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: oms-viewonly
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - nodes
  - persistentvolumeclaims
  - persistentvolumes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - clusterroles
  - roles
  - rolebindings
  verbs:
  - get
  - list
  - watch

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: oms-read 
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: oms-read
  labels: 
    k8s-app: oms-read
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: oms-viewonly
subjects:
- kind: ServiceAccount
  name: oms-read
  namespace: kube-system
```

<!--more-->

然后创建：
`kubectl apply -f oms-viewonly.yaml `

最后就可以使用以下命令查找刚刚创建SA的token:
`kubectl -n kube-system describe secret $(kubectl -n kube-system  get secret | grep oms-read | awk '{print $1}')`



