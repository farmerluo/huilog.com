---
title: docker swarm 集群及多主机overlay网络测试
author: 阿辉
date: 2018-08-10T11:43:10+00:00
categories:
- Docker
- Vxlan
tags:
- Docker
- Vxlan
keywords:
- Docker
- Vxlan
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
clearReading: false
---
{{< toc >}}

docker的swarm集群已经支持多主机的overlay网络，而且目前测试下来发现安装及配置非常方便，跟k8s相比，安装及配置要轻松好多。

## 1. 测试环境
使用2台虚拟机来测试，操作系统为ubuntu 14.04.04，系统自带内核为4.2，注意overlay需要3.16以上的内核版本。

| 主机名  |  IP | 备注  |
| ------------ | ------------ | ------------ |
| ubuntu1  | 192.168.11.21  | manger  |
| ubuntu2  | 192.168.11.22  | worker  |

## 2. 安装docker
在所有主机上安装docker,使用官方APT源。
```bash
#删除系统自带的docker
apt-get remove docker docker-engine docker.io

#安装内核模块
apt-get install \
    linux-image-extra-$(uname -r) \
    linux-image-extra-virtual

#下载安装Docker APT库源证书
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
apt-key fingerprint 0EBFCD88

#增加APT库，使用阿里云镜像
add-apt-repository \
   "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu/ \
   $(lsb_release -cs) \
   stable"

#安装docker
apt-get update
apt-get install docker-ce
```

<!--more-->

## 3. 配置防火墙策略
在ubuntu上可以使用iptables-persistent来管理防火墙
```bash
#安装
apt-get install iptables-persistent

#清空策略
/etc/init.d/iptables-persistent flush

#在manager节点增加以下策略:
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 2376 -j ACCEPT
iptables -A INPUT -p tcp --dport 2377 -j ACCEPT
iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 4789 -j ACCEPT

#在worker节点增加以下策略:
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 2376 -j ACCEPT
iptables -A INPUT -p tcp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 7946 -j ACCEPT
iptables -A INPUT -p udp --dport 4789 -j ACCEPT

#保存策略
/etc/init.d/iptables-persistent save

#重启docker
service docker restart
```
各端口的作用如下：
- TCP port 2376 for secure Docker client communication. This port is required for Docker Machine to work. Docker Machine is used to orchestrate Docker hosts.
- TCP port 2377. This port is used for communication between the nodes of a Docker Swarm or cluster. It only needs to be opened on manager nodes.
- TCP and UDP port 7946 for communication among nodes (container network discovery).
- UDP port 4789 for overlay network traffic (container ingress networking).


## 4. 配置swarm集群
在manager节点上初始化集群，在manager上执行以下命令:

```bash
root@ubuntu1:/etc/apt# docker swarm init --advertise-addr 192.168.11.21

Swarm initialized: current node (tg8klhxnuk89tya2lhe35tqx7) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-4m8sl3yl15aop8g7045evqcdh7yxvkrg6be2hhatz2wcyne4d2-ed56hwycegzmq18bvpm3pmodz 10.16.16.56:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

把worker节点加入集群，在worker上执行以下命令:
```bash
docker swarm join --token SWMTKN-1-4m8sl3yl15aop8g7045evqcdh7yxvkrg6be2hhatz2wcyne4d2-ed56hwycegzmq18bvpm3pmodz 10.16.16.56:2377
```
好了，现在我们可以在manager上查看集群及网络：

```bash
root@ubuntu1:/etc/apt# docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
150g6f1pd68qjk7wqggtc4zyf *   ubuntu1             Ready               Active              Leader              18.06.0-ce
mzdlapyot4zb7dj15t0wcy99h     ubuntu2             Ready               Active                                  18.06.0-ce

root@ubuntu1:/etc/apt# 
root@ubuntu1:/etc/apt# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
9b76e6c3cada        bridge              bridge              local
97a02511ca57        docker_gwbridge     bridge              local
bb0a7a05d2c5        host                host                local
ojjiuarwgrpm        ingress             overlay             swarm
68dac67e9965        none                null                local
```

## 5. 创建容器overlay网络

```bash
#创建网络
root@ubuntu1:/etc/apt# docker network create -d overlay --subnet=192.168.0.0/24 --gateway=192.168.0.254 --attachable testnetwork
0crsggk0wycauo9kjwj8z00f1

#查看网络
root@ubuntu1:/etc/apt# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
9b76e6c3cada        bridge              bridge              local
97a02511ca57        docker_gwbridge     bridge              local
bb0a7a05d2c5        host                host                local
ojjiuarwgrpm        ingress             overlay             swarm
68dac67e9965        none                null                local
0crsggk0wyca        testnetwork         overlay             swarm

#查看刚刚创建的网络
root@ubuntu1:/etc/apt# docker network inspect testnetwork
[
    {
        "Name": "testnetwork",
        "Id": "0crsggk0wycauo9kjwj8z00f1",
        "Created": "2018-08-09T17:05:06.757781593Z",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.254"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": null,
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4097"
        },
        "Labels": null
    }
]
```

## 6. 容器网络测试
在manager节点上创建一个容器busybox1:

```bash
root@ubuntu1:/etc/apt# docker run -itd --name=busybox1 --network=testnetwork busybox /bin/sh
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
8c5a7da1afbc: Pull complete 
Digest: sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd
Status: Downloaded newer image for busybox:latest
1a0c723dd2990813a07d1e9d95b8924edea0bf4e507471ebb619a3ad68ee3a70

#再次查看网络，可以发现有了容器的IP及LB IP
root@ubuntu1:/etc/apt# docker network inspect testnetwork                                   
[
    {
        "Name": "testnetwork",
        "Id": "0crsggk0wycauo9kjwj8z00f1",
        "Created": "2018-08-10T01:06:34.830676798+08:00",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.254"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "1a0c723dd2990813a07d1e9d95b8924edea0bf4e507471ebb619a3ad68ee3a70": {
                "Name": "busybox1",
                "EndpointID": "95bc3d1c0ddd3aacb15070aafb8aebb9dc31029ca0684288af72081ab34ad085",
                "MacAddress": "02:42:c0:a8:00:03",
                "IPv4Address": "192.168.0.3/24",
                "IPv6Address": ""
            },
            "lb-testnetwork": {
                "Name": "testnetwork-endpoint",
                "EndpointID": "0f28c078488afb7a19a2d8ec37bb6df5991f763db2f714c0f8c1f23728fb5b46",
                "MacAddress": "02:42:c0:a8:00:01",
                "IPv4Address": "192.168.0.1/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4097"
        },
        "Labels": {},
        "Peers": [
            {
                "Name": "67603512578a",
                "IP": "192.168.11.21"
            },
            {
                "Name": "f77f0897a85b",
                "IP": "192.168.11.22"
            }
        ]
    }
]
```

在worker节点上创建一个容器busybox2:
```bash
root@ubuntu2:~# docker run -itd --name=busybox2 --network=testnetwork busybox /bin/sh
Unable to find image 'busybox:latest' locally
latest: Pulling from library/busybox
8c5a7da1afbc: Pull complete 
Digest: sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd
Status: Downloaded newer image for busybox:latest
bb544eaf149086f93e6c35d9098a937282ed442be582e4516c24ac5fce9100da

root@ubuntu2:~# docker network inspect testnetwork
[
    {
        "Name": "testnetwork",
        "Id": "0crsggk0wycauo9kjwj8z00f1",
        "Created": "2018-08-10T01:07:23.121549142+08:00",
        "Scope": "swarm",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "192.168.0.0/24",
                    "Gateway": "192.168.0.254"
                }
            ]
        },
        "Internal": false,
        "Attachable": true,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "bb544eaf149086f93e6c35d9098a937282ed442be582e4516c24ac5fce9100da": {
                "Name": "busybox2",
                "EndpointID": "bd61b48e4b066d9e9ca81a267ee0c554c047f6bd68e16795fff81180f4b3fcdd",
                "MacAddress": "02:42:c0:a8:00:04",
                "IPv4Address": "192.168.0.4/24",
                "IPv6Address": ""
            },
            "lb-testnetwork": {
                "Name": "testnetwork-endpoint",
                "EndpointID": "f912103a56c705c31c8ee9476dfca1c53d0e2e321781a38338b3a180c3d08f36",
                "MacAddress": "02:42:c0:a8:00:02",
                "IPv4Address": "192.168.0.2/24",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.driver.overlay.vxlanid_list": "4097"
        },
        "Labels": {},
        "Peers": [
            {
                "Name": "67603512578a",
                "IP": "192.168.11.21"
            },
            {
                "Name": "f77f0897a85b",
                "IP": "192.168.11.22"
            }
        ]
    }
]
```
下面在刚刚创建的2个容器内互相PING对方做测试：
```bash
#manager节点
root@ubuntu1:/etc/apt# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
1a0c723dd299        busybox             "/bin/sh"           About a minute ago   Up About a minute                       busybox1

root@ubuntu1:/etc/apt# docker exec -it 1a0c723dd299 ping 192.168.0.4
PING 192.168.0.4 (192.168.0.4): 56 data bytes
64 bytes from 192.168.0.4: seq=0 ttl=64 time=1.040 ms
64 bytes from 192.168.0.4: seq=1 ttl=64 time=0.763 ms
64 bytes from 192.168.0.4: seq=2 ttl=64 time=0.854 ms
64 bytes from 192.168.0.4: seq=3 ttl=64 time=0.745 ms
64 bytes from 192.168.0.4: seq=4 ttl=64 time=0.846 ms
64 bytes from 192.168.0.4: seq=5 ttl=64 time=0.716 ms
64 bytes from 192.168.0.4: seq=6 ttl=64 time=0.889 ms
^C
--- 192.168.0.4 ping statistics ---
7 packets transmitted, 7 packets received, 0% packet loss
round-trip min/avg/max = 0.716/0.836/1.040 ms

#worker节点
root@ubuntu2:~# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
bb544eaf1490        busybox             "/bin/sh"           2 minutes ago       Up 2 minutes                            busybox2
root@ubuntu2:~# docker exec -it bb544eaf1490 ping 192.168.0.3
PING 192.168.0.3 (192.168.0.3): 56 data bytes
64 bytes from 192.168.0.3: seq=0 ttl=64 time=0.754 ms
64 bytes from 192.168.0.3: seq=1 ttl=64 time=0.677 ms
64 bytes from 192.168.0.3: seq=2 ttl=64 time=0.873 ms
^C
--- 192.168.0.3 ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.677/0.768/0.873 ms
root@ubuntu2:~# 
```

可以发现在2台主机上的容器已经可以互连ping通对方了。

## 7. 云环境限制
由于docker原生的overlay网络使用的是标准的vxlan协议，使用的端口也是标准的vxlan端口（UDP 4789）。而各个云环境，如阿里云，腾讯云，也都是使用vxlan使用的。所以会有冲突，UDP 4789网络是不通的。目前也没有找到变通的办法。docker目前为止还不支持自定义vxlan端口。（在腾讯云的黑石环境验证过，确定不行）

## 8. 参考
[https://docs.docker.com/network/network-tutorial-overlay/](https://docs.docker.com/network/network-tutorial-overlay/ "https://docs.docker.com/network/network-tutorial-overlay/")

[https://www.digitalocean.com/community/tutorials/how-to-configure-the-linux-firewall-for-docker-swarm-on-ubuntu-16-04](https://www.digitalocean.com/community/tutorials/how-to-configure-the-linux-firewall-for-docker-swarm-on-ubuntu-16-04 "https://www.digitalocean.com/community/tutorials/how-to-configure-the-linux-firewall-for-docker-swarm-on-ubuntu-16-04")
