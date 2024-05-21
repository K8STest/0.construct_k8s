# 二进制安装方式会带一个 calico 终端工具

calicoctl node status
ip r

# 跨节点不同 pod 通信流程图

```
pod(nginx)                                           pod(busybox)
    eth0                                                   eth0
(172.20.139.94 fa:ed:2c:cc:c9:cd)                     (172.20.247.1 1e:a4:fa:76:7e:95)
         |                                                      |
         |                                                      |
    cali1342368ccea                                        cali12d4a061371
         |                                                      |
         |                                                      |
tunl0(172.20.139.64)                                       tunl0(172.20.247.0)
         |                                                      |
         |                                                      |
node ens32(10.0.1.203 00:16:3e:1e:ea:ea)      node ens32(10.0.1.202 00:16:3e:1e:fa:fb)
         |                       icmp(1)-ipip(4)                |
         |____________________________<<<_______________________|



```

# 查看相邻节点的信息

```
# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 10.0.1.202   | node-to-node mesh | up    | 12:49:54 | Established |
| 10.0.1.203   | node-to-node mesh | up    | 12:49:54 | Established |
| 10.0.1.204   | node-to-node mesh | up    | 12:49:54 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.



```

# 实战抓包观察 calico 的流量

```
                ↑--- endpoints ---↓
ingress ---> service ----------- pods


service --- kube-proxy(ipvs)


# kubectl create deployment nginx --image=nginx:1.21.6
# kubectl expose deployment nginx --port=80 --target-port=80

# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
nginx-6f648b8457-7ld9v   1/1     Running   0          10m     172.20.139.94   10.0.1.203   <none>           <none>


# kubectl get svc
NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.68.0.1     <none>        443/TCP   21d
nginx        ClusterIP   10.68.6.226   <none>        80/TCP    10m

# ipvsadm -ln |grep -A1 10.68.6.226
TCP  10.68.6.226:80 rr
  -> 172.20.139.94:80             Masq    1      0          0



# ipvs会配置一个虚拟网卡kube-ipvs0，实际了类似于nginx upstream的功能，只要有新增svc，就会在k8s集群里面的每台node下的kube-ipvs0配置一段inet
# ip a|grep kube-ipvs0
4: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    inet 10.68.0.1/32 scope global kube-ipvs0
    inet 10.68.0.2/32 scope global kube-ipvs0
    inet 10.68.107.220/32 scope global kube-ipvs0
    inet 10.68.229.156/32 scope global kube-ipvs0
    inet 10.68.6.226/32 scope global kube-ipvs0



# 至此 service 到 pod这一段的旅程完成，然后开始重头戏，实际到pod的流量旅程
# 注：
在 Kubernetes 中，Service 是一个虚拟的抽象层，负责将请求转发到指定的 Pod 上。因此，Service 的流量实际上是通过 Kubernetes 集群内部的网络组件进行转发的，而不是直接流经宿主机。

如果您需要捕获 Kubernetes 中 Service 的流量，可以通过选择一个 Pod，在其中启动抓包工具的方式来实现。在启动抓包工具时，需要注意安全性和特权模式的问题



# calico默认采用ipip协议来在多节点间传输数据，在每台节点上面会生成一个虚拟网卡 tunl0，它用来在一个IP报文中封装新的IP头部，下面我们实际以跨节点的pod通信来解释这一切
# ip a|grep tunl0
12: tunl0@NONE: <NOARP,UP,LOWER_UP> mtu 1480 qdisc noqueue state UNKNOWN group default qlen 1000
    inet 172.20.84.128/32 scope global tunl0



# kubectl exec -it nginx-6f648b8457-7ld9v -- bash
root@nginx-6f648b8457-7ld9v:/# cat /sys/class/net/eth0/iflink
9

# 安装route
# apt install net-tools

# 确保测试pod调度在和nginx不同的node上面
# kubectl run busybox --rm -it --image=registry.cn-shanghai.aliyuncs.com/acs/busybox:v1.29.2 -- sh
If you don't see a command prompt, try pressing enter.
/ # ping 172.20.139.94
PING 172.20.139.94 (172.20.139.94): 56 data bytes
64 bytes from 172.20.139.94: seq=0 ttl=62 time=2.243 ms


# 在运行 nginx的node上抓包
# tcpdump -n -s0 -i ens32 ip host 10.0.1.202 -w 1.pcap

# kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE         NOMINATED NODE   READINESS GATES
busybox                  1/1     Running   0          5m58s   172.20.247.1    10.0.1.202   <none>           <none>
nginx-6f648b8457-7ld9v   1/1     Running   0          10m     172.20.139.94   10.0.1.203   <none>           <none>


# 用 Wireshark 分析 1.pcap, 设置过滤条件 ip.addr == 172.20.139.94
# 然后点开最下面的 Internet Protocol Version 4, Src: 10.0.1.201, Dst: 10.0.1.202
# 可以看到节点间通信采用的协议：  Protocol: IPIP (4)

"Protocol: IPIP (4)" 表示使用的协议是 IPIP（IP封装）协议，并且指定了协议版本为 4。


```

# 操作步骤

kubectl create deployment nginx --image=nginx:1.21.6
kubectl get po -owide
kubectl expose deployment nginx --port=80 --target-port=80
kubectl describe svc nginx
kubectl get po -n default -l app=nginx
curl 10.68.91.185
kubectl get po -n default -l app=nginx -owide

ipvsadm -ln|grep -A1 10.68.91.185 # svc 的 ip 地址，查看 svc 都往哪些 pod 转发流量

ip a|grep kube-ipvs0

kubectl exec -it nginx-6f648b8457-pfjjw -- bash

root@node-1:~# kubectl exec -it nginx-6f648b8457-pfjjw -- bash
root@nginx-6f648b8457-pfjjw:/# cat /sys/class/net/eth0/iflink
5

ip a # 然后去 22 节点上执行，会发现有一个 5: cali85bd0a302dc@if3

kubectl run busybox --rm -it --image=registry.cn-shanghai.aliyuncs.com/acs/busybox:v1.29.2 -- sh # busybox pod 运行在 23 节点上

# 抓包操作

ip a #会看到 6: tunl0@NONE
kubectl exec -it nginx-6f648b8457-nzrmn -- bash
cat /sys/class/net/eth0/iflink

ssh 192.168.1.22 # 21 机器登录到 22 机器
ip a

可以看到
23: caliaa565b440ad@if4

kubectl get po -owide # 可以看到 pod 是在节点 192.168.1.22 中运行的

# 实际跑一个

logout #退出 ssh 登录

kubectl run busybox --rm -it --image=registry.cn-shanghai.aliyuncs.com/acs/busybox:v1.29.2 -- sh # 20 机器运行
kubectl get po -owide # 再打开一个 20， 容器 po 是运行在 192.168.1.22 机器上的
回到 busybox 中
ping 172.20.139.120 # ping nginx pod 的 IP

ip r # 在 22 机器上查看 192.168.1.0/24 dev ens32 proto kernel scope link src 192.168.1.22

# 在运行 nginx 的 node 上抓包

# tcpdump -n -s0 -i ens32 ip host 192.168.1.22

sudo apt update # 22 中执行安装 tcpdump
sudo apt install tcpdump

tcpdump -n -s0 -i ens32 ip host 192.168.1.22|grep 172.20.139.123 # 用 nginx 的 pod 进行过滤

ping 172.20.139.120 # 回到 busybox 中 ping nginx pod

tcpdump -n -s0 -i ens32 ip host 192.168.1.22 -w 1.pcap

# calico 生产实战排错案例

# 1、谷歌云 google cloud 自建 K8S 集群节点间通信问题

```
谷歌云创建虚拟机自建K8S（如果CNI组件用的calico,协议是IPIP的话），需要在防火墙中放开IPIP协议

# 问题现象
service以及跨节点的POD  IP无法通信，tcpdump抓包对应的IP地址什么包都没收到


# 解决步骤

在 Google Cloud 上，要开放 IPIP（IP in IP）协议以允许在虚拟机之间进行通信，你需要创建一个自定义防火墙规则。请遵循以下步骤创建此规则：

登录到 Google Cloud Console ↗。

在左侧导航菜单中，选择 "VPC 网络" > "防火墙"。

单击 "创建防火墙规则" 按钮。

为防火墙规则提供一个描述性名称，例如 "allow-ipip"。

在 "网络" 下拉菜单中，选择你要应用此规则的 VPC 网络。

在 "优先级" 字段中，输入一个数字（例如 1000），以表示此规则的优先级。较低的数字表示较高的优先级。

在 "方向" 下拉菜单中，选择 "入站"。

在 "动作" 下拉菜单中，选择 "允许"。

在 "来源过滤器" 下拉菜单中，选择 "IP 地址范围"。在 "来源 IP 地址范围" 字段中，输入允许 IPIP 通信的 IP 地址范围，例如 "10.128.0.0/9"（用于允许来自 Google Cloud 内部所有 VM 的 IPIP 通信）。

在 "协议和端口" 部分，选择 "指定协议和端口"。在协议字段中输入 "4"（IPIP 协议的 IP 协议号），并将端口字段留空。

单击 "创建" 按钮，完成防火墙规则的创建。

现在，你已经创建了一个防火墙规则，允许 Google Cloud 虚拟机之间的 IPIP 协议通信。


```

# 2、calico-node 无法正常运行

kubectl -n kube-system get pod

calico-node 无法正常运行，报错如下：
查看日志信息：
calico/node is not ready: BIRD is not ready: BGP not established with x.x.x.x,y.y.y.y...

问题原因是 calico node 名称冲突

扩容工具扩容调用 API 扩容机器，会给扩容的机器以 AWS-na-k8s-temp[1-9]来命名，集群上存在之前扩容的一台机器名称为 aws-na-k8s-temp1，昨晚扩容了三台机器，名称为 aws-na-k8s-temp1、aws-na-k8s-temp2、aws-na-k8s-temp3

然后就是这两台同样主机名（aws-na-k8s-temp1）的机器，在 calico 里面注册节点名称时就会产生冲突 ，导致冲突节点只能有一个注册进来

## 解决问题时相关命令记录

calicoctl node status

# 删除冲突的已注册 calico 节点

calicoctl delete node aws-na-k8s-temp1

# 给冲突机器重命名下 hostname

hostnamectl set-hostname ip-172-31-7-227

# 将新名称写入 calico 的节点名称配置里面（注意不能有\n 换行符）

echo -n ip-172-31-7-227 > /var/lib/calico/nodename

# 然后删除问题的 calico-node 使其重启

kubectl -n kube-system delete pod calico-node-xf99f

# 观察是否正常

# calicoctl get node

NAME  
aws-na-k8s-temp2  
aws-na-k8s-temp3  
ip-172-31-1-50  
ip-172-31-2-247  
ip-172-31-7-227

# calicoctl node status

Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS | PEER TYPE | STATE | SINCE | INFO |
+--------------+-------------------+-------+----------+-------------+
| 172.31.8.108 | node-to-node mesh | up | 13:00:30 | Established |
| 172.31.3.33 | node-to-node mesh | up | 13:08:09 | Established |
| 172.31.2.247 | node-to-node mesh | up | 02:40:43 | Established |
| 172.31.7.227 | node-to-node mesh | up | 02:40:49 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

# 后续扩容改进操作

扩容好的机器要记得把 hostname 改成唯一的名称（相关扩容脚本已更新），记得操作前先把节点驱逐打污点：
kubectl drain 172.31.3.33 --delete-local-data --ignore-daemonsets
