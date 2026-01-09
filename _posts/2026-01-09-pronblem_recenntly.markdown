---
layout: post
title:	"Problem_recently-1.10"
date:	2026-01-09 19:29:09 +0800
categories: jekyll update
---
一、 podman 和 docker 的区别

```
podman : 完全没有守护进程 runc
docker : 具有一个守护进程 dockerd
命令行兼容性
# Docker 命令
docker run -d --name nginx nginx:alpine
docker pull redis:latest
docker ps -a
docker images
docker build -t myapp:v1 .
docker logs nginx

# Podman 等价命令（直接替换即可）
podman run -d --name nginx nginx:alpine
podman pull redis:latest
podman ps -a
podman images
podman build -t myapp:v1 .
podman logs nginx

# 创建软链接，零成本迁移
# 给所有用户生效（推荐）
sudo ln -s /usr/bin/podman /usr/bin/docker
# 执行完后，原来的 docker 命令完全可用，底层是 podman 在运行
docker run -d nginx

# podman 的 pod 和 kubernetes 的 pod 完全兼容
# 创建一个名为 my-pod 的 Pod
podman pod create --name my-pod -p 8080:80
# 向 Pod 中添加 nginx 容器（共享 Pod 网络）
podman run -d --pod my-pod --name nginx nginx:alpine
# 向同一个 Pod 中添加 redis 容器（和 nginx 共享网络，互相能直接访问）
podman run -d --pod my-pod --name redis redis:latest
```

```
# 总结来说
一、Podman 是 docker 的安全和升级版
二、二者都是容器管理工具，完全遵循OCI标准，底层都调用runc、crun
三、docker run → docker-client → dockerd → containerd → runc
四、podman run → podman 二进制程序 → 直接调用 crun/runc
```

二、容器直接进行网络通讯的差异（ container  和 pod ）

```
# 本次问题是Podman Rootless 模式网络限制
# sysctl 放行权限 / 网桥互通
# Podman Rootless 放行网络权限（核心救命）
# 提前判断是否处于无根模式，Linux 内核 3.10.0-1062
# 完整的内核优化命令
# 内核层面放开对 NAT 转发的校验
# 内核 IP 转发总开关
# 所有网卡转发开关
echo -e "net.ipv4.ip_unprivileged_port_start=0\nnet.ipv4.ip_forward=1\nnet.ipv4.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
# 校验是否生效
sudo sysctl net.ipv4.ip_unprivileged_port_start net.ipv4.ip_forward net.ipv4.conf.all.forwarding
podman restart grafana prometheus
podman exec -it grafana curl 192.168.100.133:9090
# 可能因为内核版本较低无法实现，此时可以用网桥互通和主机网络
# 网桥互通
podman run -d --name prometheus --network monitor-net -p 9090:9090 -v /prometheus:/etc/prometheus prom/prometheus
podman run -d --name grafana --network monitor-net -p 3000:3000 grafana/grafana-oss
http://proemtheus:9090
# 主机网络
podman run -d --name prometheus --network=host registry.cn-hangzhou.aliyuncs.com/google_containers/prometheus:v2.53.0
podman run -d --name grafana --network=host grafana/grafana-oss
# 直接访问容器内 IP
podman inspect proemtheus | grep IPAddress
```

```
# 总结来说
一、podman info | grep -i "rootless" 判断是否处于无根模式（网络隔离、权限隔离、文件系统隔离）
二、net.ipv4.ip_unprivileged_port_start=0 这个内核参数是 Linux 内核 3.10.0-1062 及以上版本 才新增的
三、CentOS7 内核默认没有配置 IP 转发 + 地址伪装规则
```

```
# 宿主机和容器的通讯
podman run -d --name prometheus -p 9090:9090 镜像
# 独立容器之间的通讯 （依赖 iptale、nat、podman0）
查询目标容器的 IP 确定接入的端口
# pod 内部的多个容器的通讯
podman pod create --name monitor-pod -p 9090:9090 -p 3000:3000
podman run -d --name prometheus --pod monitor-pod 镜像
podman run -d --name grafana --pod monitor-pod 镜像
# pod 之间的通讯
查询 Pod 的 IP 确定接入的端口
===========================
解决容器间和 Pod 间转发依靠 Linux 内核是 K8s 的重要作用
所有 Pod 在同一网段下，进行端口转发、网络扁平化
$ 智源大模型关于日期推导可能是计算太复杂，推理超时的缘故
```

```

```
