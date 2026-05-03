---
title: Docker运行OpenWRT
categories: docker
date: 2024-07-25 18:26:51
updated: 2024-07-25 18:26:51
---

在日常网络优化中，OpenWRT 作为强大灵活的软路由系统，非常适合作为旁路由运行；而使用 Docker 快速部署 OpenWRT，不仅便于管理，还能避免单独刷固件、物理接入的繁琐；本篇笔记记录了通过 Docker 安装并配置 OpenWRT 为旁路由的全过程；

> 参考资料：
> https://www.ilovn.com/2023/02/23/deploy-openwrt-with-docker/
> https://openwrt.ai/docker%E7%89%88openwrt%E6%97%81%E8%B7%AF%E7%94%B1%E5%AE%89%E8%A3%85%E8%AE%BE%E7%BD%AE%E6%95%99%E7%A8%8B/

## 准备工作-开启网卡混杂模式

旁路由需要 Docker 网络能桥接到主机的物理网卡，因此必须开启混杂模式（Promiscuous Mode）；

使用 ifconfig 或 ip addr 查找你的有线网卡名，比如 enp1s0，然后执行：
```bash
sudo ip link set enp1s0 promisc on
```

确认是否已开启：
```bash
ip link | grep PROMISC
```

## 创建 Macvlan 网络

创建一个 Macvlan 类型的网络，供 OpenWRT 容器使用；你可以根据自己的局域网网段修改 subnet 和 gateway 参数：

```bash
docker network create -d macvlan \
  --subnet=192.168.10.0/24 \
  --gateway=192.168.10.1 \
  -o parent=enp1s0 openwrt
```

- --subnet: 指定 OpenWRT 所在的子网；
- --gateway: 原有主路由地址；
- -o parent: 指定物理网卡；
- openwrt: 为网络命名；

## 获取并导入 OpenWRT 镜像

从 https://openwrt.ai 下载适合自己设备架构的镜像，例如 x86_64 架构的 Generic 镜像；

下载后导入镜像，此命令会将镜像命名为 openwrt：

```bash
docker import 文件名.tar.gz openwrt
```

## 运行 OpenWRT 容器

创建容器并连接至刚才创建的网络：

```bash
docker run -d \
  --restart always \
  --name openwrt \
  --network openwrt --privileged=true \
  openwrt /sbin/init
```

## 配置 OpenWRT 网络信息

进入容器：

```bash
docker exec -it openwrt bash
```

编辑网络配置文件：

```bash
nano /etc/config/network
```

参考配置如下：

```bash
config interface 'lan'
        option device 'br-lan'
        option proto 'static'
        option ipaddr '192.168.10.2'     # OpenWRT 地址
        option netmask '255.255.255.0'
        option gateway '192.168.10.1'    # 主路由地址
        option peerdns '0'
        list dns '223.5.5.5'
```

配置完成后重启网络服务：

```bash
/etc/init.d/network restart
```

## 进入 Web 管理页面

此时你可以在浏览器访问 http://192.168.10.2 进入 OpenWRT 的 LuCI 管理界面，初始用户名密码均为 root；

进入系统后，可以通过“网络向导”配置旁路由模式，建议：
- IP 地址：192.168.10.2（与前文一致）
- 网关与 DNS：均填写主路由地址（如 192.168.10.1）

## 开启 DHCP 功能（可选）

若希望 OpenWRT 自动分配 IP，则需要：
- 关闭主路由的 DHCP 服务；
- 在 OpenWRT 中开启 DHCP 功能，并配置其 IP 分配范围；

## 总结

部署完成后，你的 OpenWRT 容器就具备完整的旁路由能力，可以实现流量分流、广告屏蔽等高级功能；同时也得益于 Docker 的管理便利，容器化的 OpenWRT 更便于升级、迁移和备份，推荐给动手能力强的朋友们试试；
