---
title: 通过Jenkins Pipeline在Dockerfile内自动构建包含私有模块的golang项目
author: 阿辉
date: 2021-11-22T08:05:58+00:00
categories:
- Golang
- Dockerfile
tags:
- Golang
- Dockerfile
keywords:
- Golang
- Dockerfile
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
以前写的golang项目一直使用的都是公开的module，最近使用私有gitlab仓库的module时，发现go get mod时需要输入用户名密码，开发环境还好解决，手动输一次让他记住就好了。但是项目正式上线，通过公司统一的jenkins Pipeline执行构建时，没有办法去手动输用户名密码。

## Dockerfile配置

要解决这个问题，可以通过.netrc文件来处理。Dockerfile如下：

```bash
FROM mirror.ccs.tencentyun.com/library/golang:1.17 as builder

ARG GIT_USR
ARG GIT_PWD

ENV GOPROXY=https://goproxy.cn,direct
ENV GOPRIVATE=stash.xxxx.com
ENV GOOS=linux
ENV GIT_TERMINAL_PROMPT=1

WORKDIR /build
COPY . ./

RUN go env -w GOPRIVATE=stash.xxxx.com \
    && echo "machine stash.xxxx.com login ${GIT_USR} password ${GIT_PWD}" &gt; ~/.netrc \
    && GO111MODULE=on CGO_ENABLED=0 GOOS=${GOOS} GOPROXY=${GOPROXY} go build -o=knative-audit-log

################################################################################
##                               MAIN STAGE                                   ##
################################################################################
# Copy the manager into the distroless image.
FROM mirror.ccs.tencentyun.com/library/alpine:3.13

RUN echo 'https://mirrors.cloud.tencent.com/alpine/v3.13/main' &gt; /etc/apk/repositories \
    && echo 'https://mirrors.cloud.tencent.com/alpine/v3.13/community' &gt;&gt;/etc/apk/repositories \
    && apk update && apk add tzdata && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" &gt; /etc/timezone

WORKDIR /usr/local/app
COPY --from=builder /build/knative-audit-log /usr/local/app/knative-audit-log
RUN chmod 755 /usr/local/app/knative-audit-log

ENTRYPOINT ["/usr/local/app/knative-audit-log"]
```

重点在3,4行和15行，3，4行通过传入GIT\_USR和GIT\_PWD两个参数到docker构建内，第15行生成.netrc配置。

现在我们就可以通过传参的方式来构建了：

`docker build -t hub.docker.com/xxx/knative-audit-log:v0.1 --build-arg GIT_USR=**** --build-arg 'GIT_PWD=****' -f ./Dockerfile .`

<!--more-->

## Jenkins配置

修改Jenkins Pipeline脚本，通过withCredentials将GIT的用户名和密码传入到docker.build的参数内。使用withCredentials后，jenkins的构建记录内，GIT\_USR和GIT\_PWD将被显示为××××。以提高安全性。

```java   
    stage('Docker build and push image'){
        echo "Start Building..."

        env.git_branch=env.git_branch.replaceAll("/","-")
        image_url = "${image_url}-${git_branch}-${git_commit}"
        currentBuild.displayName = "#${BUILD_ID} $image_url"
        withCredentials([usernamePassword(credentialsId: '1', usernameVariable: 'GIT_USR', passwordVariable: 'GIT_PWD')]) {
            if ( env.root_dir ) {
              def root_dir=env.root_dir
              echo "$root_dir"
              customImage = docker.build(image_url, "--build-arg GIT_USR=${GIT_USR} --build-arg GIT_PWD=${GIT_PWD} --build-arg root_dir=${root_dir} -f ${dockerfile} .")
            } else {
              customImage = docker.build(image_url, "--build-arg GIT_USR=${GIT_USR} --build-arg GIT_PWD=${GIT_PWD} -f ${dockerfile} .")
            }
        }
        docker.withRegistry('https://dockerhub.xxxx.com', 'dockerhub.xxxx.com') {
            customImage.push()
        }
        echo "Build Done, Image Name: ${image_url}"
    }

```

## netrc参考

.netrc是linux系统提供的一个自动保存用户名密码的功能，支持ftp,curl,git等。.netrc文件格式为：

**machine name**

标识远程计算机名称. curl在.netrc文件中搜索与URL中指定的远程机器相匹配的机器令牌. 一旦匹配,随后的.netrc令牌将被处理,当获取的文件末尾或遇到另一台机器时停止.

**login name**

远程机器的用户名字符串.

**password string**

提供密码.如果存在此令牌,则在远程服务器需要密码作为登录过程的一部分时,curl将提供指定的字符串.请注意,如果这个令牌存在.netrc文件中,您确实是 **应该** 确保文件不可被任何人读到,除了用户.

一个例子.netrc对于主机 stash.xxxx.com 和 名为&#8221;username&#8221;的用户,使用密码&#8217;pwdxxxxx&#8217;, 像这样:

```
machine stash.xxxx.com
login username
password pwdxxxxx
```