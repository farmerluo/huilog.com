---
title: k8s集群从v1.18升级到v1.23后，再执行helm upgrade报错：no matches for kind Ingress in version
author: 阿辉
date: 2022-06-05T09:53:56+00:00
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
clearReading: false
---
k8s集群从v1.18升级到v1.23后，再执行helm upgrade报错：no matches for kind &#8220;Ingress&#8221; in version &#8220;networking.k8s.io/v1beta1&#8243;，如下所示：

```bash
luohui@luohuideMBP16 ~/git/victoria-metrics-k8s-stack (master*?) $ helm upgrade -i victoria-metrics-k8s-stack  .  -n victoria -f  values-override-dev.yaml --debug
history.go:56: [debug] getting history for release victoria-metrics-k8s-stack
upgrade.go:142: [debug] preparing upgrade for victoria-metrics-k8s-stack
upgrade.go:150: [debug] performing update for victoria-metrics-k8s-stack
Error: UPGRADE FAILED: current release manifest contains removed kubernetes api(s) for this kubernetes version and it is therefore unable to build the kubernetes objects for performing the diff. error from kubernetes: unable to recognize "": no matches for kind "Ingress" in version "networking.k8s.io/v1beta1"
helm.go:84: [debug] unable to recognize "": no matches for kind "Ingress" in version "networking.k8s.io/v1beta1"
current release manifest contains removed kubernetes api(s) for this kubernetes version and it is therefore unable to build the kubernetes objects for performing the diff. error from kubernetes
helm.sh/helm/v3/pkg/action.(*Upgrade).performUpgrade
    helm.sh/helm/v3/pkg/action/upgrade.go:269
helm.sh/helm/v3/pkg/action.(*Upgrade).RunWithContext
    helm.sh/helm/v3/pkg/action/upgrade.go:151
main.newUpgradeCmd.func2
    helm.sh/helm/v3/cmd/helm/upgrade.go:197
github.com/spf13/cobra.(*Command).execute
    github.com/spf13/cobra@v1.3.0/command.go:856
github.com/spf13/cobra.(*Command).ExecuteC
    github.com/spf13/cobra@v1.3.0/command.go:974
github.com/spf13/cobra.(*Command).Execute
    github.com/spf13/cobra@v1.3.0/command.go:902
main.main
    helm.sh/helm/v3/cmd/helm/helm.go:83
runtime.main
    runtime/proc.go:255
runtime.goexit
    runtime/asm_arm64.s:1133
UPGRADE FAILED
main.newUpgradeCmd.func2
    helm.sh/helm/v3/cmd/helm/upgrade.go:199
github.com/spf13/cobra.(*Command).execute
    github.com/spf13/cobra@v1.3.0/command.go:856
github.com/spf13/cobra.(*Command).ExecuteC
    github.com/spf13/cobra@v1.3.0/command.go:974
github.com/spf13/cobra.(*Command).Execute
    github.com/spf13/cobra@v1.3.0/command.go:902
main.main
    helm.sh/helm/v3/cmd/helm/helm.go:83
runtime.main
    runtime/proc.go:255
runtime.goexit
    runtime/asm_arm64.s:1133
```

此类问题已有先例，helm官方文档也有说明：https://helm.sh/zh/docs/topics/kubernetes_apis/

<!--more-->

有一个叫mapkubeapis的插件可以专门做这事。安装测试下：

```bash
luohui@luohuideMBP16 ~/git/victoria-metrics-k8s-stack (master*?) $ helm plugin install https://github.com/helm/helm-mapkubeapis
Downloading and installing helm-mapkubeapis v0.3.0 ...
https://github.com/helm/helm-mapkubeapis/releases/download/v0.3.0/helm-mapkubeapis_0.3.0_darwin_amd64.tar.gz
Installed plugin: mapkubeapis
```

看看是不是有用?

```bash
luohui@luohuideMBP16 ~/git/victoria-metrics-k8s-stack (master*?) $ helm list -n victoria
NAME                        NAMESPACE   REVISION    UPDATED                                 STATUS      CHART                               APP VERSION
victoria-metrics-k8s-stack  victoria    114         2022-05-19 14:40:17.696855 +0800 CST    deployed    victoria-metrics-k8s-stack-0.8.1    1.76.1
luohui@luohuideMBP16 ~/git/victoria-metrics-k8s-stack (master*?) $ helm mapkubeapis victoria-metrics-k8s-stack -n victoria
2022/05/31 11:19:33 Release 'victoria-metrics-k8s-stack' will be checked for deprecated or removed Kubernetes APIs and will be updated if necessary to supported API versions.
2022/05/31 11:19:33 Get release 'victoria-metrics-k8s-stack' latest version.
2022/05/31 11:19:36 Check release 'victoria-metrics-k8s-stack' for deprecated or removed APIs...
2022/05/31 11:19:36 Found 4 instances of deprecated or removed Kubernetes API:
"apiVersion: networking.k8s.io/v1beta1
kind: Ingress
"
Supported API equivalent:
"apiVersion: networking.k8s.io/v1
kind: Ingress
"
2022/05/31 11:19:36 Finished checking release 'victoria-metrics-k8s-stack' for deprecated or removed APIs.
2022/05/31 11:19:36 Deprecated or removed APIs exist, updating release: victoria-metrics-k8s-stack.
2022/05/31 11:19:36 Set status of release version 'victoria-metrics-k8s-stack.v114' to 'superseded'.
2022/05/31 11:19:38 Release version 'victoria-metrics-k8s-stack.v114' updated successfully.
2022/05/31 11:19:38 Add release version 'victoria-metrics-k8s-stack.v115' with updated supported APIs.
2022/05/31 11:19:39 Release version 'victoria-metrics-k8s-stack.v115' added successfully.
2022/05/31 11:19:39 Release 'victoria-metrics-k8s-stack' with deprecated or removed APIs updated successfully to new version.
2022/05/31 11:19:39 Map of release 'victoria-metrics-k8s-stack' deprecated or removed APIs to supported versions, completed successfully.
```

再helm upgrade，发现可以正常运行升级了。

```bash
luohui@luohuideMBP16 ~/git/victoria-metrics-k8s-stack (master*?) $ helm upgrade -i victoria-metrics-k8s-stack  .  -n victoria -f  values-override-dev.yaml
W0531 11:21:58.181695   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.243156   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.286186   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.314193   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.371877   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.442703   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.470533   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.532463   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
W0531 11:21:58.595351   97939 warnings.go:70] policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
Release "victoria-metrics-k8s-stack" has been upgraded. Happy Helming!
NAME: victoria-metrics-k8s-stack
LAST DEPLOYED: Tue May 31 11:21:50 2022
NAMESPACE: victoria
STATUS: deployed
REVISION: 116
```