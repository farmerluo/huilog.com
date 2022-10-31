---
title: 在kubernetes 上部署ceph Rook测试
author: 阿辉
date: 2019-08-16T06:05:25+00:00
categories:
- Kubernetes
- Rook
tags:
- Kubernetes
- Rook
keywords:
- Kubernetes
- Rook
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true

---

{{< toc >}}

# 1. 部署rook
下载：
```
mkdir rook
cd rook/

wget https://raw.githubusercontent.com/rook/rook/release-1.0/cluster/examples/kubernetes/ceph/common.yaml
wget https://raw.githubusercontent.com/rook/rook/release-1.0/cluster/examples/kubernetes/ceph/cluster.yaml
wget https://raw.githubusercontent.com/rook/rook/release-1.0/cluster/examples/kubernetes/ceph/operator.yaml
```
修改配置：
```
vim cluster.yaml
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: ceph/ceph:v13 #ceph版本
    allowUnsupported: false
  dataDirHostPath: /data/ceph/rook  #存储的目录
  mon:
    count: 3
    allowMultiplePerNode: false
  dashboard:
    enabled: true
  network:
    hostNetwork: false
  rbdMirroring:
    workers: 0
  annotations:
  resources:
    useAllNodes: true
    useAllDevices: true
    deviceFilter:
    location:
    config:
    directories:
    - path: /data/ceph/rook
```
部署：
```
kubectl create -f common.yaml
kubectl create -f operator.yaml
kubectl create -f cluster.yaml
```

<!--more-->

查看POD状态：
```
[root@sh-saas-k8s1-master-qa-01 rook]# kubectl get -n rook-ceph all  -o wide                               
NAME                                          READY     STATUS      RESTARTS   AGE       IP             NODE
pod/rook-ceph-agent-7vm6m                     1/1       Running     0          4m        10.11.96.25    10.11.96.25
pod/rook-ceph-agent-7wppf                     1/1       Running     0          4m        10.11.96.27    10.11.96.27
pod/rook-ceph-agent-mwd4r                     1/1       Running     0          4m        10.11.96.24    10.11.96.24
pod/rook-ceph-agent-qpjkk                     1/1       Running     0          4m        10.11.96.26    10.11.96.26
pod/rook-ceph-agent-rvp84                     1/1       Running     0          4m        10.11.96.28    10.11.96.28
pod/rook-ceph-mgr-a-759db679fd-2vh8g          1/1       Running     0          3m        10.252.5.83    10.11.96.27
pod/rook-ceph-mon-a-788bd6f7b8-72jmn          1/1       Running     0          4m        10.252.3.79    10.11.96.24
pod/rook-ceph-mon-b-79dd77678f-d7x6x          1/1       Running     0          4m        10.252.3.223   10.11.96.25
pod/rook-ceph-mon-c-554db8d78b-npxkb          1/1       Running     0          4m        10.252.4.120   10.11.96.26
pod/rook-ceph-operator-576599b75f-mvjj7       1/1       Running     0          4m        10.252.5.81    10.11.96.27
pod/rook-ceph-osd-0-5d65b5666d-6xgfs          1/1       Running     0          3m        10.252.3.225   10.11.96.25
pod/rook-ceph-osd-1-5df5f749b7-t4zhk          1/1       Running     0          3m        10.252.4.123   10.11.96.26
pod/rook-ceph-osd-2-7449d5775-2xpgx           1/1       Running     0          3m        10.252.3.81    10.11.96.24
pod/rook-ceph-osd-3-5c696bff5b-nm26s          1/1       Running     0          3m        10.252.5.86    10.11.96.27
pod/rook-ceph-osd-4-66c9cb685f-bqvnk          1/1       Running     0          3m        10.252.4.140   10.11.96.28
pod/rook-ceph-osd-prepare-10.11.96.24-bbwfl   0/2       Completed   0          3m        10.252.3.80    10.11.96.24
pod/rook-ceph-osd-prepare-10.11.96.25-czw9p   0/2       Completed   0          3m        10.252.3.224   10.11.96.25
pod/rook-ceph-osd-prepare-10.11.96.26-m28pc   0/2       Completed   0          3m        10.252.4.121   10.11.96.26
pod/rook-ceph-osd-prepare-10.11.96.27-d86dq   0/2       Completed   0          3m        10.252.5.84    10.11.96.27
pod/rook-ceph-osd-prepare-10.11.96.28-89hp7   0/2       Completed   0          3m        10.252.4.136   10.11.96.28
pod/rook-discover-9pz22                       1/1       Running     0          4m        10.252.3.75    10.11.96.24
pod/rook-discover-cv8p6                       1/1       Running     0          4m        10.252.4.135   10.11.96.28
pod/rook-discover-k87wm                       1/1       Running     0          4m        10.252.3.222   10.11.96.25
pod/rook-discover-tfr6r                       1/1       Running     0          4m        10.252.5.82    10.11.96.27
pod/rook-discover-zqlmp                       1/1       Running     0          4m        10.252.4.119   10.11.96.26

NAME                              TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE       SELECTOR
service/rook-ceph-mgr             ClusterIP   10.252.255.223   <none>        9283/TCP   3m        app=rook-ceph-mgr,rook_cluster=rook-ceph
service/rook-ceph-mgr-dashboard   ClusterIP   10.252.252.85    <none>        8443/TCP   3m        app=rook-ceph-mgr,rook_cluster=rook-ceph
service/rook-ceph-mon-a           ClusterIP   10.252.254.127   <none>        6789/TCP   4m        app=rook-ceph-mon,ceph_daemon_id=a,mon=a,mon_cluster=rook-ceph,rook_cluster=rook-ceph
service/rook-ceph-mon-b           ClusterIP   10.252.254.19    <none>        6789/TCP   4m        app=rook-ceph-mon,ceph_daemon_id=b,mon=b,mon_cluster=rook-ceph,rook_cluster=rook-ceph
service/rook-ceph-mon-c           ClusterIP   10.252.254.248   <none>        6789/TCP   4m        app=rook-ceph-mon,ceph_daemon_id=c,mon=c,mon_cluster=rook-ceph,rook_cluster=rook-ceph

NAME                             DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE       CONTAINERS        IMAGES             SELECTOR
daemonset.apps/rook-ceph-agent   5         5         5         5            5           <none>          4m        rook-ceph-agent   rook/ceph:v1.0.6   app=rook-ceph-agent
daemonset.apps/rook-discover     5         5         5         5            5           <none>          4m        rook-discover     rook/ceph:v1.0.6   app=rook-discover

NAME                                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE       CONTAINERS           IMAGES             SELECTOR
deployment.apps/rook-ceph-mgr-a      1         1         1            1           3m        mgr                  ceph/ceph:v13      app=rook-ceph-mgr,ceph_daemon_id=a,instance=a,mgr=a,rook_cluster=rook-ceph
deployment.apps/rook-ceph-mon-a      1         1         1            1           4m        mon                  ceph/ceph:v13      app=rook-ceph-mon,ceph_daemon_id=a,mon=a,mon_cluster=rook-ceph,rook_cluster=rook-ceph
deployment.apps/rook-ceph-mon-b      1         1         1            1           4m        mon                  ceph/ceph:v13      app=rook-ceph-mon,ceph_daemon_id=b,mon=b,mon_cluster=rook-ceph,rook_cluster=rook-ceph
deployment.apps/rook-ceph-mon-c      1         1         1            1           4m        mon                  ceph/ceph:v13      app=rook-ceph-mon,ceph_daemon_id=c,mon=c,mon_cluster=rook-ceph,rook_cluster=rook-ceph
deployment.apps/rook-ceph-operator   1         1         1            1           4m        rook-ceph-operator   rook/ceph:v1.0.6   app=rook-ceph-operator
deployment.apps/rook-ceph-osd-0      1         1         1            1           3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=0,rook_cluster=rook-ceph
deployment.apps/rook-ceph-osd-1      1         1         1            1           3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=1,rook_cluster=rook-ceph
deployment.apps/rook-ceph-osd-2      1         1         1            1           3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=2,rook_cluster=rook-ceph
deployment.apps/rook-ceph-osd-3      1         1         1            1           3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=3,rook_cluster=rook-ceph
deployment.apps/rook-ceph-osd-4      1         1         1            1           3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=4,rook_cluster=rook-ceph

NAME                                            DESIRED   CURRENT   READY     AGE       CONTAINERS           IMAGES             SELECTOR
replicaset.apps/rook-ceph-mgr-a-759db679fd      1         1         1         3m        mgr                  ceph/ceph:v13      app=rook-ceph-mgr,ceph_daemon_id=a,instance=a,mgr=a,pod-template-hash=3158623598,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-mon-a-788bd6f7b8      1         1         1         4m        mon                  ceph/ceph:v13      app=rook-ceph-mon,ceph_daemon_id=a,mon=a,mon_cluster=rook-ceph,pod-template-hash=3446829364,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-mon-b-79dd77678f      1         1         1         4m        mon                  ceph/ceph:v13      app=rook-ceph-mon,ceph_daemon_id=b,mon=b,mon_cluster=rook-ceph,pod-template-hash=3588332349,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-mon-c-554db8d78b      1         1         1         4m        mon                  ceph/ceph:v13      app=rook-ceph-mon,ceph_daemon_id=c,mon=c,mon_cluster=rook-ceph,pod-template-hash=1108648346,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-operator-576599b75f   1         1         1         4m        rook-ceph-operator   rook/ceph:v1.0.6   app=rook-ceph-operator,pod-template-hash=1321556319
replicaset.apps/rook-ceph-osd-0-5d65b5666d      1         1         1         3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=0,pod-template-hash=1821612228,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-osd-1-5df5f749b7      1         1         1         3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=1,pod-template-hash=1891930563,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-osd-2-7449d5775       1         1         1         3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=2,pod-template-hash=300581331,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-osd-3-5c696bff5b      1         1         1         3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=3,pod-template-hash=1725269916,rook_cluster=rook-ceph
replicaset.apps/rook-ceph-osd-4-66c9cb685f      1         1         1         3m        osd                  ceph/ceph:v13      app=rook-ceph-osd,ceph-osd-id=4,pod-template-hash=2275762419,rook_cluster=rook-ceph

NAME                                          DESIRED   SUCCESSFUL   AGE       CONTAINERS            IMAGES                           SELECTOR
job.batch/rook-ceph-osd-prepare-10.11.96.24   1         1            3m        copy-bins,provision   rook/ceph:v1.0.6,ceph/ceph:v13   controller-uid=33fb8994-cfbe-11e9-a77c-5254001526dc
job.batch/rook-ceph-osd-prepare-10.11.96.25   1         1            3m        copy-bins,provision   rook/ceph:v1.0.6,ceph/ceph:v13   controller-uid=340165fd-cfbe-11e9-a77c-5254001526dc
job.batch/rook-ceph-osd-prepare-10.11.96.26   1         1            3m        copy-bins,provision   rook/ceph:v1.0.6,ceph/ceph:v13   controller-uid=34080e79-cfbe-11e9-a77c-5254001526dc
job.batch/rook-ceph-osd-prepare-10.11.96.27   1         1            3m        copy-bins,provision   rook/ceph:v1.0.6,ceph/ceph:v13   controller-uid=34306cfc-cfbe-11e9-a77c-5254001526dc
job.batch/rook-ceph-osd-prepare-10.11.96.28   1         1            3m        copy-bins,provision   rook/ceph:v1.0.6,ceph/ceph:v13   controller-uid=346d7c8c-cfbe-11e9-a77c-5254001526dc
```

可查看安装日志：
```
[root@sh-saas-k8s1-master-qa-01 rook]# kubectl logs -f -n rook-ceph pod/rook-ceph-operator-576599b75f-mvjj7
2019-09-05 09:17:42.894477 I | rookcmd: starting Rook v1.0.6 with arguments '/usr/local/bin/rook ceph operator'
2019-09-05 09:17:42.894597 I | rookcmd: flag values: --alsologtostderr=false, --csi-attacher-image=quay.io/k8scsi/csi-attacher:v1.0.1, --csi-cephfs-image=quay.io/cephcsi/cephfsplugin:v1.0.0, --csi-cephfs-plugin-template-path=/etc/ceph-csi/cephfs/csi-cephfsplugin.yaml, --csi-cephfs-provisioner-template-path=/etc/ceph-csi/cephfs/csi-cephfsplugin-provisioner.yaml, --csi-enable-cephfs=false, --csi-enable-rbd=false, --csi-provisioner-image=quay.io/k8scsi/csi-provisioner:v1.0.1, --csi-rbd-image=quay.io/cephcsi/rbdplugin:v1.0.0, --csi-rbd-plugin-template-path=/etc/ceph-csi/rbd/csi-rbdplugin.yaml, --csi-rbd-provisioner-template-path=/etc/ceph-csi/rbd/csi-rbdplugin-provisioner.yaml, --csi-registrar-image=quay.io/k8scsi/csi-node-driver-registrar:v1.0.2, --csi-snapshotter-image=quay.io/k8scsi/csi-snapshotter:v1.0.1, --help=false, --log-flush-frequency=5s, --log-level=INFO, --log_backtrace_at=:0, --log_dir=, --log_file=, --logtostderr=true, --mon-healthcheck-interval=45s, --mon-out-timeout=10m0s, --skip_headers=false, --stderrthreshold=2, --v=0, --vmodule=
2019-09-05 09:17:42.898920 I | cephcmd: starting operator
2019-09-05 09:17:42.928296 I | op-agent: getting flexvolume dir path from FLEXVOLUME_DIR_PATH env var
2019-09-05 09:17:42.928336 I | op-agent: flexvolume dir path env var FLEXVOLUME_DIR_PATH is not provided. Defaulting to: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/
2019-09-05 09:17:42.928343 I | op-agent: discovered flexvolume dir path from source default. value: /usr/libexec/kubernetes/kubelet-plugins/volume/exec/
2019-09-05 09:17:42.928349 I | op-agent: no agent mount security mode given, defaulting to 'Any' mode
2019-09-05 09:17:42.947121 I | op-agent: rook-ceph-agent daemonset started
2019-09-05 09:17:42.953798 I | op-discover: rook-discover daemonset started
2019-09-05 09:17:42.955972 I | operator: rook-provisioner ceph.rook.io/block started using ceph.rook.io flex vendor dir
I0905 09:17:42.956178       9 leaderelection.go:217] attempting to acquire leader lease  rook-ceph/ceph.rook.io-block...
2019-09-05 09:17:42.956438 I | operator: rook-provisioner rook.io/block started using rook.io flex vendor dir
2019-09-05 09:17:42.956453 I | operator: Watching the current namespace for a cluster CRD
2019-09-05 09:17:42.956458 I | op-cluster: start watching clusters in all namespaces
I0905 09:17:42.956459       9 leaderelection.go:217] attempting to acquire leader lease  rook-ceph/rook.io-block...
2019-09-05 09:17:42.956475 I | op-cluster: Enabling hotplug orchestration: ROOK_DISABLE_DEVICE_HOTPLUG=false
2019-09-05 09:17:42.957539 I | op-cluster: skipping watching for legacy rook cluster events (legacy cluster CRD probably doesn't exist): the server could not find the requested resource (get clusters.ceph.rook.io)
I0905 09:17:42.981618       9 leaderelection.go:227] successfully acquired lease rook-ceph/rook.io-block
I0905 09:17:42.981676       9 leaderelection.go:227] successfully acquired lease rook-ceph/ceph.rook.io-block
I0905 09:17:42.981721       9 event.go:209] Event(v1.ObjectReference{Kind:"Endpoints", Namespace:"rook-ceph", Name:"rook.io-block", UID:"0404b2a6-cfbe-11e9-a77c-5254001526dc", APIVersion:"v1", ResourceVersion:"56346426", FieldPath:""}): type: 'Normal' reason: 'LeaderElection' rook-ceph-operator-576599b75f-mvjj7_04031f02-cfbe-11e9-b3ed-0a580afc0551 became leader
I0905 09:17:42.981748       9 event.go:209] Event(v1.ObjectReference{Kind:"Endpoints", Namespace:"rook-ceph", Name:"ceph.rook.io-block", UID:"04046316-cfbe-11e9-a77c-5254001526dc", APIVersion:"v1", ResourceVersion:"56346425", FieldPath:""}): type: 'Normal' reason: 'LeaderElection' rook-ceph-operator-576599b75f-mvjj7_0402fdff-cfbe-11e9-b3ed-0a580afc0551 became leader
I0905 09:17:42.981781       9 controller.go:769] Starting provisioner controller rook.io/block_rook-ceph-operator-576599b75f-mvjj7_04031f02-cfbe-11e9-b3ed-0a580afc0551!
I0905 09:17:42.981746       9 controller.go:769] Starting provisioner controller ceph.rook.io/block_rook-ceph-operator-576599b75f-mvjj7_0402fdff-cfbe-11e9-b3ed-0a580afc0551!
I0905 09:17:43.982015       9 controller.go:818] Started provisioner controller ceph.rook.io/block_rook-ceph-operator-576599b75f-mvjj7_0402fdff-cfbe-11e9-b3ed-0a580afc0551!
I0905 09:17:44.182443       9 controller.go:818] Started provisioner controller rook.io/block_rook-ceph-operator-576599b75f-mvjj7_04031f02-cfbe-11e9-b3ed-0a580afc0551!
2019-09-05 09:17:48.955066 I | op-cluster: starting cluster in namespace rook-ceph
2019-09-05 09:17:54.977171 I | op-k8sutil: waiting for job rook-ceph-detect-version to complete...
2019-09-05 09:18:00.004866 I | op-cluster: Detected ceph image version: 13.2.6 mimic
2019-09-05 09:18:00.004941 I | op-cluster: CephCluster rook-ceph status: Creating
2019-09-05 09:18:00.019958 I | op-mon: start running mons
2019-09-05 09:18:00.025436 I | exec: Running command: ceph-authtool --create-keyring /var/lib/rook/rook-ceph/mon.keyring --gen-key -n mon. --cap mon 'allow *'
2019-09-05 09:18:00.092210 I | exec: Running command: ceph-authtool --create-keyring /var/lib/rook/rook-ceph/client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mgr 'allow *' --cap mds 'allow'
2019-09-05 09:18:00.164482 I | op-mon: creating mon secrets for a new cluster
2019-09-05 09:18:00.177696 I | op-mon: saved mon endpoints to config map map[data: maxMonId:-1 mapping:{"node":{},"port":{}}]
2019-09-05 09:18:00.387479 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:00.387565 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:00.387610 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:02.188993 I | op-mon: targeting the mon count 3
2019-09-05 09:18:02.585076 I | op-mon: Found 5 running nodes without mons
2019-09-05 09:18:02.585107 I | op-mon: creating mon a
2019-09-05 09:18:02.791085 I | op-mon: mon a endpoint is 10.252.254.127:6789
2019-09-05 09:18:03.186038 I | op-mon: saved mon endpoints to config map map[data:a=10.252.254.127:6789 maxMonId:2 mapping:{"node":{"a":{"Name":"10.11.96.24","Hostname":"10.11.96.24","Address":"10.11.96.24"},"b":{"Name":"10.11.96.25","Hostname":"10.11.96.25","Address":"10.11.96.25"},"c":{"Name":"10.11.96.26","Hostname":"10.11.96.26","Address":"10.11.96.26"}},"port":{}}]
2019-09-05 09:18:04.987468 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:04.987559 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:04.987615 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:04.988856 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:04.988963 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:04.989058 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:05.008944 I | op-mon: mons created: 1
2019-09-05 09:18:05.008971 I | op-mon: waiting for mon quorum with [a]
2019-09-05 09:18:05.185762 I | op-mon: mon a is not yet running
2019-09-05 09:18:05.185791 I | op-mon: mons running: []
2019-09-05 09:18:10.190068 I | op-mon: mons running: [a]
2019-09-05 09:18:10.190376 I | exec: Running command: ceph mon_status --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/641496985
2019-09-05 09:18:10.633281 I | op-mon: Monitors in quorum: [a]
2019-09-05 09:18:10.633304 I | op-mon: creating mon b
2019-09-05 09:18:10.663936 I | op-mon: mon a endpoint is 10.252.254.127:6789
2019-09-05 09:18:10.674175 I | op-mon: mon b endpoint is 10.252.254.19:6789
2019-09-05 09:18:10.687986 I | op-mon: saved mon endpoints to config map map[data:a=10.252.254.127:6789,b=10.252.254.19:6789 maxMonId:2 mapping:{"node":{"a":{"Name":"10.11.96.24","Hostname":"10.11.96.24","Address":"10.11.96.24"},"b":{"Name":"10.11.96.25","Hostname":"10.11.96.25","Address":"10.11.96.25"},"c":{"Name":"10.11.96.26","Hostname":"10.11.96.26","Address":"10.11.96.26"}},"port":{}}]
2019-09-05 09:18:11.216766 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:11.216987 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:11.217045 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:11.218090 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:11.218187 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:11.218247 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:11.225444 I | op-mon: deployment for mon rook-ceph-mon-a already exists. updating if needed
2019-09-05 09:18:11.228198 I | op-k8sutil: updating deployment rook-ceph-mon-a
2019-09-05 09:18:13.240709 I | op-k8sutil: finished waiting for updated deployment rook-ceph-mon-a
2019-09-05 09:18:13.248286 I | op-mon: mons created: 2
2019-09-05 09:18:13.248337 I | op-mon: waiting for mon quorum with [a b]
2019-09-05 09:18:13.255681 I | op-mon: mon b is not yet running
2019-09-05 09:18:13.255721 I | op-mon: mons running: [a]
2019-09-05 09:18:18.265832 I | op-mon: mons running: [a b]
2019-09-05 09:18:18.266963 I | exec: Running command: ceph mon_status --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/992068132
2019-09-05 09:18:22.879217 I | op-mon: Monitors in quorum: [b a]
2019-09-05 09:18:22.879242 I | op-mon: creating mon c
2019-09-05 09:18:22.902640 I | op-mon: mon a endpoint is 10.252.254.127:6789
2019-09-05 09:18:22.924981 I | op-mon: mon b endpoint is 10.252.254.19:6789
2019-09-05 09:18:22.934311 I | op-mon: mon c endpoint is 10.252.254.248:6789
2019-09-05 09:18:22.948040 I | op-mon: saved mon endpoints to config map map[data:a=10.252.254.127:6789,b=10.252.254.19:6789,c=10.252.254.248:6789 maxMonId:2 mapping:{"node":{"a":{"Name":"10.11.96.24","Hostname":"10.11.96.24","Address":"10.11.96.24"},"b":{"Name":"10.11.96.25","Hostname":"10.11.96.25","Address":"10.11.96.25"},"c":{"Name":"10.11.96.26","Hostname":"10.11.96.26","Address":"10.11.96.26"}},"port":{}}]
2019-09-05 09:18:23.870951 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:23.871052 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:23.871109 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:23.873080 I | cephconfig: writing config file /var/lib/rook/rook-ceph/rook-ceph.config
2019-09-05 09:18:23.873240 I | cephconfig: copying config to /etc/ceph/ceph.conf
2019-09-05 09:18:23.873350 I | cephconfig: generated admin config in /var/lib/rook/rook-ceph
2019-09-05 09:18:23.881493 I | op-mon: deployment for mon rook-ceph-mon-a already exists. updating if needed
2019-09-05 09:18:23.883887 I | op-k8sutil: updating deployment rook-ceph-mon-a
2019-09-05 09:18:25.895569 I | op-k8sutil: finished waiting for updated deployment rook-ceph-mon-a
2019-09-05 09:18:25.903379 I | op-mon: deployment for mon rook-ceph-mon-b already exists. updating if needed
2019-09-05 09:18:25.905559 I | op-k8sutil: updating deployment rook-ceph-mon-b
2019-09-05 09:18:27.917708 I | op-k8sutil: finished waiting for updated deployment rook-ceph-mon-b
2019-09-05 09:18:27.924749 I | op-mon: mons created: 3
2019-09-05 09:18:27.924771 I | op-mon: waiting for mon quorum with [a b c]
2019-09-05 09:18:27.941272 I | op-mon: mon c is not yet running
2019-09-05 09:18:27.941288 I | op-mon: mons running: [a b]
2019-09-05 09:18:32.956665 I | op-mon: mons running: [a b c]
2019-09-05 09:18:32.956800 I | exec: Running command: ceph mon_status --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/785290035
2019-09-05 09:18:35.605197 W | op-mon: monitor c is not in quorum list
2019-09-05 09:18:40.620022 I | op-mon: mons running: [a b c]
2019-09-05 09:18:40.620132 I | exec: Running command: ceph mon_status --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/641614582
2019-09-05 09:18:41.084656 I | op-mon: Monitors in quorum: [b a c]
2019-09-05 09:18:41.084692 I | op-mgr: start running mgr
2019-09-05 09:18:41.084897 I | exec: Running command: ceph auth get-or-create-key mgr.a mon allow * mds allow * osd allow * --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/253151709
2019-09-05 09:18:41.528888 I | exec: Running command: ceph config-key get mgr/dashboard/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/789918104
2019-09-05 09:18:41.952176 I | exec: Error ENOENT: error obtaining 'mgr/dashboard/server_addr': (2) No such file or directory
2019-09-05 09:18:41.952300 I | exec: Running command: ceph config-key del mgr/dashboard/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/301805079
2019-09-05 09:18:42.331132 I | exec: no such key 'mgr/dashboard/server_addr'
2019-09-05 09:18:42.331227 I | op-mgr: clearing http bind fix mod=dashboard ver=12.0.0 luminous changed=false err=<nil>
2019-09-05 09:18:42.331301 I | exec: Running command: ceph config-key get mgr/dashboard/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/471784586
2019-09-05 09:18:42.687752 I | exec: Error ENOENT: error obtaining 'mgr/dashboard/a/server_addr': (2) No such file or directory
2019-09-05 09:18:42.688001 I | exec: Running command: ceph config-key del mgr/dashboard/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/345082465
2019-09-05 09:18:43.065333 I | exec: no such key 'mgr/dashboard/a/server_addr'
2019-09-05 09:18:43.065435 I | op-mgr: clearing http bind fix mod=dashboard ver=12.0.0 luminous changed=false err=<nil>
2019-09-05 09:18:43.065492 I | exec: Running command: ceph config get mgr.a mgr/dashboard/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/276260428
2019-09-05 09:18:43.466022 I | exec: Error ENOENT: 
2019-09-05 09:18:43.466175 I | exec: Running command: ceph config rm mgr.a mgr/dashboard/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/729274683
2019-09-05 09:18:43.832629 I | op-mgr: clearing http bind fix mod=dashboard ver=13.0.0 mimic changed=false err=<nil>
2019-09-05 09:18:43.832725 I | exec: Running command: ceph config get mgr.a mgr/dashboard/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/132855646
2019-09-05 09:18:44.229987 I | exec: Error ENOENT: 
2019-09-05 09:18:44.230184 I | exec: Running command: ceph config rm mgr.a mgr/dashboard/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/821459237
2019-09-05 09:18:44.636805 I | op-mgr: clearing http bind fix mod=dashboard ver=13.0.0 mimic changed=false err=<nil>
2019-09-05 09:18:44.636898 I | exec: Running command: ceph config-key get mgr/prometheus/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/251951680
2019-09-05 09:18:44.991648 I | exec: Error ENOENT: error obtaining 'mgr/prometheus/server_addr': (2) No such file or directory
2019-09-05 09:18:44.991844 I | exec: Running command: ceph config-key del mgr/prometheus/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/010972831
2019-09-05 09:18:45.418772 I | exec: no such key 'mgr/prometheus/server_addr'
2019-09-05 09:18:45.418994 I | op-mgr: clearing http bind fix mod=prometheus ver=12.0.0 luminous changed=false err=<nil>
2019-09-05 09:18:45.419061 I | exec: Running command: ceph config-key get mgr/prometheus/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/364481906
2019-09-05 09:18:45.880914 I | exec: Error ENOENT: error obtaining 'mgr/prometheus/a/server_addr': (2) No such file or directory
2019-09-05 09:18:45.881034 I | exec: Running command: ceph config-key del mgr/prometheus/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/566659625
2019-09-05 09:18:46.319600 I | exec: no such key 'mgr/prometheus/a/server_addr'
2019-09-05 09:18:46.319697 I | op-mgr: clearing http bind fix mod=prometheus ver=12.0.0 luminous changed=false err=<nil>
2019-09-05 09:18:46.319759 I | exec: Running command: ceph config get mgr.a mgr/prometheus/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/192833396
2019-09-05 09:18:46.723264 I | exec: Error ENOENT: 
2019-09-05 09:18:46.723479 I | exec: Running command: ceph config rm mgr.a mgr/prometheus/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/274484291
2019-09-05 09:18:47.136602 I | op-mgr: clearing http bind fix mod=prometheus ver=13.0.0 mimic changed=false err=<nil>
2019-09-05 09:18:47.136696 I | exec: Running command: ceph config get mgr.a mgr/prometheus/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/285369542
2019-09-05 09:18:47.530895 I | exec: Error ENOENT: 
2019-09-05 09:18:47.531033 I | exec: Running command: ceph config rm mgr.a mgr/prometheus/a/server_addr --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/590768493
2019-09-05 09:18:47.917627 I | op-mgr: clearing http bind fix mod=prometheus ver=13.0.0 mimic changed=false err=<nil>
2019-09-05 09:18:47.925676 I | op-mgr: skipping enabling orchestrator modules on releases older than nautilus
2019-09-05 09:18:47.925794 I | exec: Running command: ceph mgr module enable prometheus --force --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/666427880
2019-09-05 09:18:48.352425 I | exec: Running command: ceph mgr module enable dashboard --force --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/116054055
2019-09-05 09:18:54.539200 I | exec: Running command: ceph dashboard create-self-signed-cert --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/980231514
2019-09-05 09:18:55.091493 I | op-mgr: Running command: ceph dashboard set-login-credentials admin *******
2019-09-05 09:18:55.927608 I | op-mgr: restarting the mgr module
2019-09-05 09:18:55.927694 I | exec: Running command: ceph mgr module disable dashboard --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/819153308
2019-09-05 09:18:57.193779 I | exec: Running command: ceph mgr module enable dashboard --force --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/050279499
2019-09-05 09:18:58.218069 I | exec: Running command: ceph config get mgr.a mgr/dashboard/url_prefix --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/807282478
2019-09-05 09:18:58.667000 I | exec: Error ENOENT: 
2019-09-05 09:18:58.667119 I | exec: Running command: ceph config rm mgr.a mgr/dashboard/url_prefix --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/441035957
2019-09-05 09:18:59.110168 I | exec: Running command: ceph config get mgr.a mgr/dashboard/server_port --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/313695376
2019-09-05 09:18:59.538209 I | exec: Error ENOENT: 
2019-09-05 09:18:59.538339 I | exec: Running command: ceph config set mgr.a mgr/dashboard/server_port 8443 --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/750564015
2019-09-05 09:18:59.995090 I | exec: Running command: ceph config get mgr.a mgr/dashboard/ssl --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/278010946
2019-09-05 09:19:00.442816 I | exec: Error ENOENT: 
2019-09-05 09:19:00.442972 I | exec: Running command: ceph config rm mgr.a mgr/dashboard/ssl --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/270489785
2019-09-05 09:19:00.888150 I | op-mgr: dashboard config has changed
2019-09-05 09:19:00.888174 I | op-mgr: restarting the mgr module
2019-09-05 09:19:00.888265 I | exec: Running command: ceph mgr module disable dashboard --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/105143492
2019-09-05 09:19:02.327502 I | exec: Running command: ceph mgr module enable dashboard --force --connect-timeout=15 --cluster=rook-ceph --conf=/var/lib/rook/rook-ceph/rook-ceph.config --keyring=/var/lib/rook/rook-ceph/client.admin.keyring --format json --out-file /tmp/031582035
2019-09-05 09:19:03.385202 I | op-mgr: dashboard service started
2019-09-05 09:19:03.400235 I | op-mgr: mgr metrics service started
2019-09-05 09:19:03.400257 I | op-osd: start running osds in namespace rook-ceph
2019-09-05 09:19:03.420804 I | op-osd: 5 of the 8 storage nodes are valid
2019-09-05 09:19:03.420824 I | op-osd: start provisioning the osds on nodes, if needed
2019-09-05 09:19:03.446212 I | op-osd: osd provision job started for node 10.11.96.24
2019-09-05 09:19:03.481946 I | op-osd: osd provision job started for node 10.11.96.25
2019-09-05 09:19:03.524858 I | op-osd: osd provision job started for node 10.11.96.26
2019-09-05 09:19:03.790461 I | op-osd: osd provision job started for node 10.11.96.27
2019-09-05 09:19:04.190516 I | op-osd: osd provision job started for node 10.11.96.28
2019-09-05 09:19:04.190534 I | op-osd: start osds after provisioning is completed, if needed
2019-09-05 09:19:04.379196 I | op-osd: osd orchestration status for node 10.11.96.24 is starting
2019-09-05 09:19:04.379217 I | op-osd: osd orchestration status for node 10.11.96.25 is starting
2019-09-05 09:19:04.379224 I | op-osd: osd orchestration status for node 10.11.96.26 is starting
2019-09-05 09:19:04.379230 I | op-osd: osd orchestration status for node 10.11.96.27 is starting
2019-09-05 09:19:04.379236 I | op-osd: osd orchestration status for node 10.11.96.28 is starting
2019-09-05 09:19:04.379240 I | op-osd: 0/5 node(s) completed osd provisioning, resource version 56347158
2019-09-05 09:19:04.618317 I | op-osd: osd orchestration status for node 10.11.96.25 is computingDiff
2019-09-05 09:19:04.661048 I | op-osd: osd orchestration status for node 10.11.96.25 is orchestrating
2019-09-05 09:19:04.673951 I | op-osd: osd orchestration status for node 10.11.96.26 is computingDiff
2019-09-05 09:19:04.716624 I | op-osd: osd orchestration status for node 10.11.96.26 is orchestrating
2019-09-05 09:19:04.737823 I | op-osd: osd orchestration status for node 10.11.96.24 is computingDiff
2019-09-05 09:19:04.779820 I | op-osd: osd orchestration status for node 10.11.96.24 is orchestrating
2019-09-05 09:19:05.132786 I | op-osd: osd orchestration status for node 10.11.96.27 is computingDiff
2019-09-05 09:19:05.189846 I | op-osd: osd orchestration status for node 10.11.96.27 is orchestrating
2019-09-05 09:19:06.422524 I | op-osd: osd orchestration status for node 10.11.96.28 is computingDiff
2019-09-05 09:19:06.490060 I | op-osd: osd orchestration status for node 10.11.96.28 is orchestrating
2019-09-05 09:19:07.797665 I | op-osd: osd orchestration status for node 10.11.96.25 is completed
2019-09-05 09:19:07.797693 I | op-osd: starting 1 osd daemons on node 10.11.96.25
2019-09-05 09:19:07.804485 I | op-osd: started deployment for osd 0 (dir=true, type=)
2019-09-05 09:19:08.865862 I | op-osd: osd orchestration status for node 10.11.96.26 is completed
2019-09-05 09:19:08.865956 I | op-osd: starting 1 osd daemons on node 10.11.96.26
2019-09-05 09:19:08.873254 I | op-osd: started deployment for osd 1 (dir=true, type=)
2019-09-05 09:19:08.891710 I | op-osd: osd orchestration status for node 10.11.96.27 is completed
2019-09-05 09:19:08.891727 I | op-osd: starting 1 osd daemons on node 10.11.96.27
2019-09-05 09:19:08.900456 I | op-osd: started deployment for osd 3 (dir=true, type=)
2019-09-05 09:19:08.920655 I | op-osd: osd orchestration status for node 10.11.96.24 is completed
2019-09-05 09:19:08.920672 I | op-osd: starting 1 osd daemons on node 10.11.96.24
2019-09-05 09:19:08.929159 I | op-osd: started deployment for osd 2 (dir=true, type=)
2019-09-05 09:19:09.904557 I | op-osd: osd orchestration status for node 10.11.96.28 is completed
2019-09-05 09:19:09.904575 I | op-osd: starting 1 osd daemons on node 10.11.96.28
2019-09-05 09:19:09.912043 I | op-osd: started deployment for osd 4 (dir=true, type=)
2019-09-05 09:19:09.933864 I | op-osd: 5/5 node(s) completed osd provisioning
2019-09-05 09:19:09.933942 I | op-osd: checking if any nodes were removed
2019-09-05 09:19:09.960181 I | op-osd: processing 0 removed nodes
2019-09-05 09:19:09.960197 I | op-osd: done processing removed nodes
2019-09-05 09:19:09.960201 I | op-osd: completed running osds in namespace rook-ceph
2019-09-05 09:19:09.960207 I | rbd-mirror: configure rbd-mirroring with 0 workers
2019-09-05 09:19:09.963517 I | rbd-mirror: no extra daemons to remove
2019-09-05 09:19:09.963533 I | op-cluster: Done creating rook instance in namespace rook-ceph
2019-09-05 09:19:09.963541 I | op-cluster: CephCluster rook-ceph status: Created
2019-09-05 09:19:09.970820 I | op-pool: start watching pool resources in namespace rook-ceph
2019-09-05 09:19:09.971773 I | op-pool: skipping watching for legacy rook pool events (legacy pool CRD probably doesn't exist): the server could not find the requested resource (get pools.ceph.rook.io)
2019-09-05 09:19:09.971791 I | op-object: start watching object store resources in namespace rook-ceph
2019-09-05 09:19:09.973011 I | op-object: skipping watching for legacy rook objectstore events (legacy objectstore CRD probably doesn't exist): the server could not find the requested resource (get objectstores.ceph.rook.io)
2019-09-05 09:19:09.973025 I | op-object: start watching object store user resources in namespace rook-ceph
2019-09-05 09:19:09.973033 I | op-file: start watching filesystem resource in namespace rook-ceph
2019-09-05 09:19:09.973737 I | op-file: skipping watching for legacy rook filesystem events (legacy filesystem CRD probably doesn't exist): the server could not find the requested resource (get filesystems.ceph.rook.io)
2019-09-05 09:19:09.973779 I | op-nfs: start watching ceph nfs resource in namespace rook-ceph
2019-09-05 09:19:09.973795 I | op-cluster: ceph status check interval is 60s
2019-09-05 09:19:09.985289 I | op-cluster: added finalizer to cluster rook-ceph

```
# 2. Dashboard访问
待安装完成后，可以通过pod/rook-ceph-mgr-a的IP来访问Dashboard:

- Luminous: Port 7000 on http
- Mimic and newer: Port 8443 on https

本例访问的URL为：
https://10.252.5.83:8443/

下面的命令可以拿到创建好的admin用户的密码。
`kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}" | base64 --decode && echo`

![](/wp-content/uploads/2019/09/663e1558b9230435b3bc7eada66b1a6b.png)

# 3. toolbox部署
部署toolbox,toolbox是一个工具集，常用都集成到了这个容器内，操作一般在这个容器内执行:
`vim toolbox.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rook-ceph-tools
  namespace: rook-ceph
  labels:
    app: rook-ceph-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rook-ceph-tools
  template:
    metadata:
      labels:
        app: rook-ceph-tools
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      containers:
      - name: rook-ceph-tools
        image: rook/ceph:v1.0.6
        command: ["/tini"]
        args: ["-g", "--", "/usr/local/bin/toolbox.sh"]
        imagePullPolicy: IfNotPresent
        env:
          - name: ROOK_ADMIN_SECRET
            valueFrom:
              secretKeyRef:
                name: rook-ceph-mon
                key: admin-secret
        securityContext:
          privileged: true
        volumeMounts:
          - mountPath: /dev
            name: dev
          - mountPath: /sys/bus
            name: sysbus
          - mountPath: /lib/modules
            name: libmodules
          - name: mon-endpoint-volume
            mountPath: /etc/rook
      # if hostNetwork: false, the "rbd map" command hangs, see https://github.com/rook/rook/issues/2021
      hostNetwork: true
      volumes:
        - name: dev
          hostPath:
            path: /dev
        - name: sysbus
          hostPath:
            path: /sys/bus
        - name: libmodules
          hostPath:
            path: /lib/modules
        - name: mon-endpoint-volume
          configMap:
            name: rook-ceph-mon-endpoints
            items:
            - key: data
              path: mon-endpoints
```
部署：
`kubectl create -f toolbox.yaml`
查看POD：
`kubectl -n rook-ceph get pod -l "app=rook-ceph-tools"`

进入POD：
```bash
[root@sh-saas-k8s1-master-qa-01 rook]# kubectl -n rook-ceph exec -it $(kubectl -n rook-ceph get pod -l "app=rook-ceph-tools" -o jsonpath='{.items[0].metadata.name}') bash
bash: warning: setlocale: LC_CTYPE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_MESSAGES: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_NUMERIC: cannot change locale (en_US.UTF-8): No such file or directory
bash: warning: setlocale: LC_TIME: cannot change locale (en_US.UTF-8): No such file or directory
[root@sh-saas-k8s1-node-qa-05 /]# ceph status
  cluster:
    id:     764d5dec-13bb-478d-97c7-fc8369cc964e
    health: HEALTH_OK
 
  services:
    mon: 3 daemons, quorum b,a,c
    mgr: a(active)
    osd: 5 osds: 5 up, 5 in
 
  data:
    pools:   0 pools, 0 pgs
    objects: 0  objects, 0 B
    usage:   324 GiB used, 955 GiB / 1.2 TiB avail
    pgs:     
 
[root@sh-saas-k8s1-node-qa-05 /]# ceph osd status
+----+-------------+-------+-------+--------+---------+--------+---------+-----------+
| id |     host    |  used | avail | wr ops | wr data | rd ops | rd data |   state   |
+----+-------------+-------+-------+--------+---------+--------+---------+-----------+
| 0  | 10.11.96.25 | 61.2G |  135G |    0   |     0   |    0   |     0   | exists,up |
| 1  | 10.11.96.26 | 64.0G |  132G |    0   |     0   |    0   |     0   | exists,up |
| 2  | 10.11.96.24 | 79.6G |  412G |    0   |     0   |    0   |     0   | exists,up |
| 3  | 10.11.96.27 | 55.9G |  140G |    0   |     0   |    0   |     0   | exists,up |
| 4  | 10.11.96.28 | 63.1G |  133G |    0   |     0   |    0   |     0   | exists,up |
+----+-------------+-------+-------+--------+---------+--------+---------+-----------+
[root@sh-saas-k8s1-node-qa-05 /]# ceph df
GLOBAL:
    SIZE        AVAIL       RAW USED     %RAW USED 
    1.2 TiB     955 GiB      324 GiB         25.34 
POOLS:
    NAME     ID     USED     %USED     MAX AVAIL     OBJECTS 
[root@sh-saas-k8s1-node-qa-05 /]# rados df
POOL_NAME USED OBJECTS CLONES COPIES MISSING_ON_PRIMARY UNFOUND DEGRADED RD_OPS RD WR_OPS WR USED COMPR UNDER COMPR 

total_objects    0
total_used       324 GiB
total_avail      955 GiB
total_space      1.2 TiB
[root@sh-saas-k8s1-node-qa-05 /]#
[root@sh-saas-k8s1-node-qa-05 /]# ceph osd pool create ceph-test 512 
For better initial performance on pools expected to store a large number of objects, consider supplying the expected_num_objects parameter when creating the pool.
[root@sh-saas-k8s1-node-qa-05 /]# ceph osd lspools
1 ceph-test
[root@sh-saas-k8s1-node-qa-05 /]# ceph osd pool application enable ceph-test rbd    
enabled application 'rbd' on pool 'ceph-test'
[root@sh-saas-k8s1-node-qa-05 /]# ceph auth ls  
installed auth entries:
这个命令可以看到所有的key
```

# 4. 创建 Ceph Block Pool
实际上创建存储池的方法不是像上面那样，是像下面这样：
`vim rbdpool.yaml`
```yaml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
```
```bash
[root@sh-saas-k8s1-master-qa-01 rook]# kubectl create -f rdbpool.yaml 
cephblockpool.ceph.rook.io "replicapool" created
```
这样就创建好了。

# 5. 创建Ceph FileSystem
如果需要使用Ceph FileSystem:
`vim filesystem.yaml`
```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs-k8s-qa
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - replicated:
        size: 3
  metadataServer:
    activeCount: 1
    activeStandby: true
```
```bash
[root@sh-saas-k8s1-master-qa-01 rook]# kubectl create -f filesystem.yaml 
cephfilesystem.ceph.rook.io "cephfs-k8s-qa" created
```
这样就OK了，rook会自动去启MDS等相关服务。真的非常的方便。
![](/wp-content/uploads/2019/09/276edf704101e9fbd8378ce3d6c50e85.png)