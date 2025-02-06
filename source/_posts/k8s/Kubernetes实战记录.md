---
title: Kubernetes实战记录
categories:
  - k8s
tags:
  - k8s
  - 实战
abbrlink: 26a46bdd
date: 2025-02-06 15:20:00
---

## 1.安装说明

> 虽然K8s 1.20版本宣布将在1.23版本之后将不再维护dockershim，意味着K8s将不直接支持Docker，不过大家不必过于担心。一是在1.23版本之前我们仍然可以使用Docker，二是dockershim肯定会有人接盘，我们同样可以使用Docker，三是Docker制作的镜像仍然可以在其他Runtime环境中使用，所以大家不必过于恐慌。
> 
> 本次安装采用的是Kubeadm安装工具，安装版本是K8s 1.20+，采用的系统为CentOS 7.9，其中Master节点3台，Node节点2台，高可用工具采用HAProxy + KeepAlived

## 2.节点规划

|      主机名      |      IP       | 说明               |
| :-----------: | :-----------: | ---------------- |
| k8s-master-lb | 192.168.10.30 | keepalived 虚拟 IP |
| k8s-master-01 | 192.168.10.31 | master节点1        |
| k8s-master-02 | 192.168.10.32 | master节点2        |
| k8s-master-03 | 192.168.10.33 | master节点3        |
|  k8s-node-01  | 192.168.10.34 | worker节点1        |
|  k8s-node-02  | 192.168.10.35 | worker节点2        |

|    信息     | 备注             |
| :-------: | -------------- |
|   系统版本    | CentOS 7.9     |
| Docker版本  | 20.10.x        |
|   K8s版本   | 1.21.x         |
|   Pod网段   | 172.168.0.0/16 |
| Service网段 | 10.96.0.0/12   |

## 3.基本配置

所有节点配置host，修改`/etc/hosts`如下：

``` powershell
vi /etc/hosts  
​  
192.168.10.30 k8s-master-lb  
192.168.10.31 k8s-master-01  
192.168.10.32 k8s-master-02  
192.168.10.33 k8s-master-03  
192.168.10.34 k8s-node-01  
192.168.10.35 k8s-node-02
```

通过ping测试主机名是否生效

``` powershell
[root@k8s-master-01 ~]# ping k8s-master-02  
PING k8s-master-02 (192.168.10.32) 56(84) bytes of data.  
64 bytes from k8s-master-02 (192.168.10.32): icmp_seq=1 ttl=64 time=0.448 ms  
64 bytes from k8s-master-02 (192.168.10.32): icmp_seq=2 ttl=64 time=0.168 ms  
64 bytes from k8s-master-02 (192.168.10.32): icmp_seq=3 ttl=64 time=0.163 ms  
^C  
--- k8s-master-02 ping statistics ---  
3 packets transmitted, 3 received, 0% packet loss, time 2000ms  
rtt min/avg/max/mdev = 0.163/0.259/0.448/0.134 ms  
[root@k8s-master-01 ~]#
```

yum源配置

``` powershell
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo  
yum install -y yum-utils device-mapper-persistent-data lvm2  
yum-config-manager --add-repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

如果repo_gpgcheck设为1，会进行校验，就会报错如下，所以这里设为0

> repomd.xml signature could not be verified for kubernetes

``` powershell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo  
[kubernetes]  
name=Kubernetes  
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/  
enabled=1  
gpgcheck=1  
repo_gpgcheck=0  
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg  
EOF

sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo
```

必备工具安装 `yum install wget jq psmisc vim net-tools telnet yum-utils device-mapper-persistent-data lvm2 git -y`

所有节点关闭防火墙，selinux、dnsmasq、swap，服务器配置如下：

``` powershell
systemctl disable --now firewalld  
systemctl disable --now dnsmasq  
systemctl disable --now NetworkManager #CentOS8 及以上版本无需关闭  
​  
setenforce 0
```

禁止加载 SELinux 策略。

``` powershell
vi /etc/sysconfig/selinux  
SELINUX=disabled
```

关闭swapoff（不关闭会影响Docker性能，建议关闭）

``` powershell
swapoff -a && sysctl -w vm.swappiness=0

vi /etc/fstab
#注释掉swap那行
[root@k8s-master-01 ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Mon Aug  8 12:58:27 2022
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=1ea31dd3-8488-4170-8434-a300bcb2f386 /boot                   xfs     defaults        0 0
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

安装ntpdate

``` powershell
rpm -ivh http://mirrors.wlnmp.com/centos/wlnmp-release-centos.noarch.rpm  
yum install -y ntpdate  
# yum install wntp -y #CentOS8 执行
```

所有节点同步时间，时间同步配置如下：

``` powershell
ln -sf /usr/shar/zoneinfo/Asia/Shanghai /etc/localtime  
echo 'Asia/Shanghai' > /etc/timezone  
ntpdate time2.aliyun.com
```

加入到 crontab

``` powershell
crontab -e  
*/5 * * * * ntpdate time2.aliyun.com
```


所有节点配置limit

``` powershell
ulimit -SHn 65535  
​  
vim /etc/security/limits.conf  
# 末尾添加如下内容  
* soft nofile 655360  
* hard nofile 131072  
* soft nproc 655350  
* hard nproc 655350  
* soft memlock unlimited  
* hard memlock unlimited
```

master01节点免密钥登陆其他节点，安装过程中生成配置文件和证书均在Master01上操作，集群管理也在master01上操作，阿里云或AWS上需要单独一台kubectl服务器。密钥配置如下

``` powershell
#以下语句只在master01节点执行  
ssh-keygen -t rsa  
for i in k8s-master-lb k8s-master-01 k8s-master-02 k8s-master-03 k8s-node-01 k8s-node-02;do ssh-copy-id -i .ssh/id_rsa.pub $i;done
```

下载安装所有的源码文件：

``` powershell
git clone https://github.com/dotbalo/k8s-ha-install.git
```

所有节点升级系统并重启：

``` powershell
yum update -y  && reboot
```

## 4.内核配置

### 4.1 升级内核版本

查询内核版本

``` powershell
[root@k8s-master-01 ~]# cat /proc/version  
Linux version 3.10.0-1160.71.1.el7.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC) ) #1 SMP Tue Jun 28 15:37:28 UTC 2022  
# 或者  
[root@k8s-master-01 ~]# uname -sr  
Linux 3.10.0-1160.71.1.el7.x86_64
```

CentOS7需要升级内核至4.18+

> ELRepo 仓库是基于社区的用于企业级 Linux 仓库，提供对 RedHat Enterprise (RHEL) 和 其他基于 RHEL的 Linux 发行版（CentOS、Scientific、Fedora 等）的支持。 ELRepo 聚焦于和硬件相关的软件包，包括文件系统驱动、显卡驱动、网络驱动、声卡驱动和摄像头驱动等。
> 
> 启用 ELRepo 仓库：(建议参考 [http://elrepo.org/tiki/HomePage](http://elrepo.org/tiki/HomePage) 上的内容)

``` powershell
# 先导入公钥  
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org  
# CentOS 7  
yum install -y https://www.elrepo.org/elrepo-release-7.el7.elrepo.noarch.rpm  
# CentOS 8  
yum install https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
```

安装 DNF 包管理器

``` powershell
yum install epel-release -y  
yum install -y dnf
```

选择安装内核

> 我们的存储库包含 elrepo-kernel 通道，它为 CentOS 和 RHEL Linux 系统提供了长期（kernel-lt） 支持内核和最新的稳定主线内核（kernel-ml）。

我们可以列出主线内核版本

``` powershell
[root@k8s-master-01 ~]# dnf --disablerepo="*" --enablerepo="elrepo-kernel" list available | grep kernel-ml  
kernel-ml.x86_64                        5.19.0-1.el7.elrepo        elrepo-kernel  
kernel-ml-devel.x86_64                  5.19.0-1.el7.elrepo        elrepo-kernel  
kernel-ml-doc.noarch                    5.19.0-1.el7.elrepo        elrepo-kernel  
kernel-ml-headers.x86_64                5.19.0-1.el7.elrepo        elrepo-kernel  
kernel-ml-tools.x86_64                  5.19.0-1.el7.elrepo        elrepo-kernel  
kernel-ml-tools-libs.x86_64             5.19.0-1.el7.elrepo        elrepo-kernel  
kernel-ml-tools-libs-devel.x86_64       5.19.0-1.el7.elrepo        elrepo-kernel
```

长期支持版本，可以用下面命令查看：

``` powershell
[root@k8s-master-01 ~]# dnf --disablerepo="*" --enablerepo="elrepo-kernel" list available | grep kernel-lt  
kernel-lt.x86_64                        5.4.209-1.el7.elrepo       elrepo-kernel  
kernel-lt-devel.x86_64                  5.4.209-1.el7.elrepo       elrepo-kernel  
kernel-lt-doc.noarch                    5.4.209-1.el7.elrepo       elrepo-kernel  
kernel-lt-headers.x86_64                5.4.209-1.el7.elrepo       elrepo-kernel  
kernel-lt-tools.x86_64                  5.4.209-1.el7.elrepo       elrepo-kernel  
kernel-lt-tools-libs.x86_64             5.4.209-1.el7.elrepo       elrepo-kernel  
kernel-lt-tools-libs-devel.x86_64       5.4.209-1.el7.elrepo       elrepo-kernel
```

下面可以选择长期版本或主线最新稳定版本：

``` powershell
# Install mainline kernels  
sudo dnf --enablerepo=elrepo-kernel install kernel-ml  
或者  
# Install long term kernels  
sudo dnf --enablerepo=elrepo-kernel install kernel-lt
```

切换默认内核版本

``` powershell
# 查看所有可用内核版本  
grubby --info=ALL | grep ^kernel  
#查看默认的内核版本  
grubby --default-kernel  
# 设置内核版本  
grubby --set-default "/boot/vmlinuz-5.16.10-1.el8.elrepo.x86_64"  
#重启  
reboot
```

检查当前内核版本

``` powershell
uname -sr
```

### 4.2 内核配置

所有节点安装ipvsadm：

``` powershell
yum install ipvsadm ipset sysstat conntrack libseccomp -y
```

所有节点配置ipvs模块

``` powershell
vim /etc/modules-load.d/ipvs.conf   
# 加入以下内容  
ip_vs  
ip_vs_lc  
ip_vs_wlc  
ip_vs_rr  
ip_vs_wrr  
ip_vs_lblc  
ip_vs_lblcr  
ip_vs_dh  
ip_vs_sh  
ip_vs_fo  
ip_vs_nq  
ip_vs_sed  
ip_vs_ftp  
ip_vs_sh  
nf_conntrack_ipv4  
ip_tables  
ip_set  
xt_set  
ipt_set  
ipt_rpfilter  
ipt_REJECT  
ipip
```

加载内核配置

``` powershell
systemctl enable --now systemd-modules-load.service
```

开启一些k8s集群中必须的内核参数，所有节点配置k8s内核

``` powershell
cat <<EOF > /etc/sysctl.d/k8s.conf  
net.ipv4.ip_forward = 1  
net.bridge.bridge-nf-call-iptables = 1  
net.bridge.bridge-nf-call-ip6tables = 1  
fs.may_detach_mounts = 1  
vm.overcommit_memory=1  
vm.panic_on_oom=0  
fs.inotify.max_user_watches=89100  
fs.file-max=52706963  
fs.nr_open=52706963  
net.netfilter.nf_conntrack_max=2310720  
​  
net.ipv4.tcp_keepalive_time = 600  
net.ipv4.tcp_keepalive_probes = 3  
net.ipv4.tcp_keepalive_intvl =15  
net.ipv4.tcp_max_tw_buckets = 36000  
net.ipv4.tcp_tw_reuse = 1  
net.ipv4.tcp_max_orphans = 327680  
net.ipv4.tcp_orphan_retries = 3  
net.ipv4.tcp_syncookies = 1  
net.ipv4.tcp_max_syn_backlog = 16384  
net.ipv4.ip_conntrack_max = 65536  
net.ipv4.tcp_max_syn_backlog = 16384  
net.ipv4.tcp_timestamps = 0  
net.core.somaxconn = 16384  
EOF  
sysctl --system
```

## 5.基本组件安装

查看k8s对应版本依赖 [https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG) 

查看Docker版本

``` powershell
yum list docker-ce.x86_64 --showduplicates | sort -r
```

所有节点安装Docker-ce最新版

``` powershell
yum install docker-ce -y
```

查看kubeadm版本

``` powershell
yum list kubeadm.x86_64 --showduplicates | sort -r
```

安装kubeadm

``` powershell
# 安装最新版  
yum install kubeadm -y  
# 安装指定版本（当前部署1.21.x）  
yum install -y kubeadm-1.21.* kubectl-1.21.* kubelet-1.21.*
```

所有节点设置开机自启动Docker

``` powershell
systemctl daemon-reload && systemctl enable --now docker
```

默认配置的pause镜像使用gcr.io仓库，国内可能无法访问，所以这里配置Kubelet使用阿里云的pause镜像：

``` powershell
cat >/etc/sysconfig/kubelet<<EOF
KUBELET_EXTRA_ARGS="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2"
EOF
```

设置Kubelet开机自启动

``` powershell
systemctl daemon-reload
systemctl enable --now kubelet
```
## 6.高可用组件安装

**注意：如果不是高可用集群或者在云上安装，haproxy和keepalived无需安装** 所有Master节点通过yum安装HAProxy和KeepAlived：

``` powershell
yum install keepalived haproxy -y
```

所有Master节点配置HAProxy（详细配置参考HAProxy文档，所有Master节点的HAProxy配置相同）：

``` powershell
[root@k8s-master-01 ~]# mkdir /etc/haproxy
[root@k8s-master-01 ~]# vim /etc/haproxy/haproxy.cfg 
global
  maxconn  2000
  ulimit-n  16384
  log  127.0.0.1 local0 err
  stats timeout 30s

defaults
  log global
  mode  http
  option  httplog
  timeout connect 5000
  timeout client  50000
  timeout server  50000
  timeout http-request 15s
  timeout http-keep-alive 15s

frontend monitor-in
  bind *:33305
  mode http
  option httplog
  monitor-uri /monitor

frontend k8s-master
  bind 0.0.0.0:16443
  bind 127.0.0.1:16443
  mode tcp
  option tcplog
  tcp-request inspect-delay 5s
  default_backend k8s-master

backend k8s-master
  mode tcp
  option tcplog
  option tcp-check
  balance roundrobin
  default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
  server k8s-master-01	192.168.10.31:6443  check
  server k8s-master-02	192.168.10.32:6443  check
  server k8s-master-03	192.168.10.33:6443  check
```

所有Master节点配置KeepAlived，配置不一样，注意区分 注意每个节点的IP和网卡（interface参数）

``` powershell
# 查看网卡
[root@k8s-master-01 ~]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens192: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:2b:3f:e4 brd ff:ff:ff:ff:ff:ff
    inet 192.168.10.31/24 brd 192.168.10.255 scope global ens192
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe2b:3fe4/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:3f:8b:98:ec brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

Master01节点的配置：

``` powershell
[root@k8s-master-01 ~]# mkdir /etc/keepalived

[root@k8s-master-01 ~]# vim /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
    interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state MASTER
    interface ens192
    mcast_src_ip 192.168.10.31
    virtual_router_id 51
    priority 101
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.10.30
    }
#    track_script {
#       chk_apiserver
#    }
}
```

Master02节点的配置：

``` powershell
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
   interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    mcast_src_ip 192.168.10.32
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.10.30
    }
#    track_script {
#       chk_apiserver
#    }
}
```

Master03节点的配置：

``` powershell
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
script_user root
    enable_script_security
}
vrrp_script chk_apiserver {
    script "/etc/keepalived/check_apiserver.sh"
 interval 5
    weight -5
    fall 2  
rise 1
}
vrrp_instance VI_1 {
    state BACKUP
    interface ens192
    mcast_src_ip 192.168.10.33
    virtual_router_id 51
    priority 100
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass K8SHA_KA_AUTH
    }
    virtual_ipaddress {
        192.168.10.30
    }
#    track_script {
#       chk_apiserver
#    }
}
```

注意上述的健康检查是关闭的，集群建立完成后再开启：

``` powershell
#    track_script {  
#       chk_apiserver  
#    }
```

配置KeepAlived健康检查文件：

``` powershell
[root@k8s-master-01 ~]# vim /etc/keepalived/check_apiserver.sh 
#!/bin/bash

err=0
for k in $(seq 1 3)
do
    check_code=$(pgrep haproxy)
    if [[ $check_code == "" ]]; then
        err=$(expr $err + 1)
        sleep 1
        continue
    else
        err=0
        break
    fi
done

if [[ $err != "0" ]]; then
    echo "systemctl stop keepalived"
    /usr/bin/systemctl stop keepalived
    exit 1
else
    exit 0
fi


chmod +x /etc/keepalived/check_apiserver.sh
```

启动haproxy和keepalived

``` powershell
[root@k8s-master-01 ~]# systemctl daemon-reload
[root@k8s-master-01 ~]# systemctl enable --now haproxy
[root@k8s-master-01 ~]# systemctl enable --now keepalived
```

测试VIP

``` powershell
[root@k8s-master-01 ~]# ping 192.168.10.30 -c 4
PING 192.168.10.30 (192.168.10.30) 56(84) bytes of data.
64 bytes from 192.168.10.30: icmp_seq=1 ttl=64 time=0.035 ms
64 bytes from 192.168.10.30: icmp_seq=2 ttl=64 time=0.033 ms
64 bytes from 192.168.10.30: icmp_seq=3 ttl=64 time=0.035 ms
64 bytes from 192.168.10.30: icmp_seq=4 ttl=64 time=0.029 ms
```

## 7.集群初始化

Master01节点创建kubeadm-config.yaml配置文件如下：

``` powershell
vim kubeadm-config.yaml

apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: 7t2weq.bjbawausm0jaxury
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.10.31
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-master-01
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  certSANs:
  - 192.168.10.30
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: 192.168.10.30:16443
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.21.14
networking:
  dnsDomain: cluster.local
  podSubnet: 172.168.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

**注意：如果不是高可用集群，192.168.10.30:16443改为master01的地址，16443改为apiserver的端口，默认是6443，注意更改v1.20.0为自己服务器kubeadm的版本：kubeadm version** 将kubeadm-config.yaml文件复制到其他master节点，之后所有Master节点提前下载镜像，可以节省初始化时间：

``` powershell
kubeadm config images pull --config kubeadm-config.yaml
```

所有节点设置开机自启动kubelet

``` powershell
systemctl enable --now kubelet #（如果启动失败无需管理，初始化成功以后即可启动）
```

Master01节点初始化，初始化以后会在/etc/kubernetes目录下生成对应的证书和配置文件，之后其他Master节点加入Master01即可：

``` powershell
kubeadm init --config kubeadm-config.yaml  --upload-certs
```

如果初始化失败，重置后再次初始化，命令如下：

``` powershell
kubeadm reset
```

初始化成功以后，会产生Token值，用于其他节点加入时使用，因此要记录下初始化成功生成的token值（令牌值）：

``` powershell
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

kubeadm join 192.168.10.30:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:6ae64507d60f30c41785c09fa258bc8c7446fa99e0cae0d2df12ac616554a52e \
        --control-plane --certificate-key ec2ae81a9975920815b9ead426e4531c6cb3381c0f340dee8b36d13eb689982a

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.10.30:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:6ae64507d60f30c41785c09fa258bc8c7446fa99e0cae0d2df12ac616554a52e
```

Master01节点配置环境变量，用于访问Kubernetes集群：

``` powershell
cat <<EOF >> /root/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
source /root/.bashrc
```

查看节点状态：

``` powershell
kubectl get nodes

NAME            STATUS     ROLES                  AGE     VERSION
k8s-master-01   NotReady   control-plane,master   2m39s   v1.21.14
```

采用初始化安装方式，所有的系统组件均以容器的方式运行并且在kube-system命名空间内，此时可以查看Pod状态：

``` powershell
[root@k8s-master-01 ~]# kubectl get pods -n kube-system -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
coredns-6f6b8cc4f6-6lk9d                0/1     Pending   0          3m9s    <none>          <none>          <none>           <none>
coredns-6f6b8cc4f6-9q46x                0/1     Pending   0          3m9s    <none>          <none>          <none>           <none>
etcd-k8s-master-01                      1/1     Running   0          3m5s    192.168.10.31   k8s-master-01   <none>           <none>
kube-apiserver-k8s-master-01            1/1     Running   0          3m4s    192.168.10.31   k8s-master-01   <none>           <none>
kube-controller-manager-k8s-master-01   1/1     Running   0          3m11s   192.168.10.31   k8s-master-01   <none>           <none>
kube-proxy-qrjrx                        1/1     Running   0          3m9s    192.168.10.31   k8s-master-01   <none>           <none>
kube-scheduler-k8s-master-01            1/1     Running   0          3m4s    192.168.10.31   k8s-master-01   <none>           <none>
```

## 8.高可用Master

如果Token过期或者遗忘，可通过以下脚本重新生成：

``` powershell
kubeadm token create --print-join-command

# Master节点需要--certificate-key
kubeadm init phase upload-certs --upload-certs
```

初始化其他master加入集群

``` powershell
kubeadm join 192.168.10.30:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:6ae64507d60f30c41785c09fa258bc8c7446fa99e0cae0d2df12ac616554a52e \
        --control-plane --certificate-key ec2ae81a9975920815b9ead426e4531c6cb3381c0f340dee8b36d13eb689982a
```

## 9.添加其他Node节点

``` powershell
kubeadm join 192.168.10.30:16443 --token 7t2weq.bjbawausm0jaxury \
        --discovery-token-ca-cert-hash sha256:6ae64507d60f30c41785c09fa258bc8c7446fa99e0cae0d2df12ac616554a52e
```

查看集群状态：

``` powershell
[root@k8s-master-01 ~]# kubectl  get node
NAME            STATUS     ROLES                  AGE    VERSION
k8s-master-01   NotReady   control-plane,master   7m1s   v1.21.14
k8s-master-02   NotReady   control-plane,master   103s   v1.21.14
k8s-master-03   NotReady   control-plane,master   103s   v1.21.14
k8s-node-01     NotReady   <none>                 26s    v1.21.14
k8s-node-02     NotReady   <none>                 25s    v1.21.14
```

## 10.Calico安装

官网：[https://www.tigera.io/project-calico/](https://www.tigera.io/project-calico/) 
安装文档：[https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico](https://projectcalico.docs.tigera.io/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico) 
以下步骤只在master01执行

``` powershell
curl https://projectcalico.docs.tigera.io/manifests/calico.yaml -O
```

创建calico

``` powershell
kubectl apply -f calico.yaml
```

## 11.Metrics Server部署

官网：[https://github.com/kubernetes-sigs/metrics-server/](https://github.com/kubernetes-sigs/metrics-server/) 
在新版的Kubernetes中系统资源的采集均使用Metrics-server，可以通过Metrics采集节点和Pod的内存、磁盘、CPU和网络的使用率。
安装metrics server

``` powershell
# 下载配置
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 修改配置
vim components.yaml

  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        imagePullPolicy: IfNotPresent

# 安装
kubectl apply -f components.yaml
```

等待kube-system命令空间下的Pod全部启动后，查看状态

``` powershell
[root@k8s-master01 metrics-server-0.4.x-kubeadm]# kubectl  top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-master01   109m         2%     1296Mi          33%       
k8s-master02   99m          2%     1124Mi          29%       
k8s-master03   104m         2%     1082Mi          28%       
k8s-node01     55m          1%     761Mi           19%       
k8s-node02     53m          1%     663Mi           17%
```

## 12.Dashboard部署

官网：[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)

``` powershell
[root@k8s-master-01 ~]# kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```

在谷歌浏览器（Chrome）启动文件中加入启动参数，用于解决无法访问Dashboard的问题

``` powershell
--test-type --ignore-certificate-errors
```