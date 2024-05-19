# nginx.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: new-nginx
  labels:
    app: new-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: new-nginx
  template:
    metadata:
      labels:
        app: new-nginx
    spec:
      containers:
#--------------------------------------------------
      - name: new-nginx
        image: nginx:1.21.6
#        image: nginx:1.25.1
        env:
          - name: TZ
            value: Asia/Shanghai
        ports:
        - containerPort: 80
        volumeMounts:
          - name: html-files
            mountPath: "/usr/share/nginx/html"
#--------------------------------------------------
      - name: busybox
        image: registry.cn-shanghai.aliyuncs.com/acs/busybox:v1.29.2
#        image: nicolaka/netshoot
        args:
        - /bin/sh
        - -c
        - >
           while :; do
             if [ -f /html/index.html ];then
               echo "[$(date +%F\ %T)] ${MY_POD_NAMESPACE}-${MY_POD_NAME}-${MY_POD_IP}" > /html/index.html
               sleep 1
             else
               touch /html/index.html
             fi
           done
        env:
          - name: TZ
            value: Asia/Shanghai
          - name: MY_POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: MY_POD_NAMESPACE
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.namespace
          - name: MY_POD_IP
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: status.podIP
        volumeMounts:
          - name: html-files
            mountPath: "/html"
          - mountPath: /etc/localtime
            name: tz-config

#--------------------------------------------------
      volumes:
        - name: html-files
          emptyDir:
            medium: Memory
            sizeLimit: 10Mi
        - name: tz-config
          hostPath:
            path: /usr/share/zoneinfo/Asia/Shanghai



```

# 给这个 nginx 的 deployment 生成一个 service（简称 svc）

# 同时也可以用生成 yaml 配置的形式来创建 kubectl expose deployment new-nginx --port=80 --target-port=80 --dry-run=client -o yaml

# 我们可以先把上面的 yaml 配置导出为 svc.yaml 提供后面，这里就直接用命令行创建了

# kubectl expose deployment new-nginx --port=80 --target-port=80

# 获取 endpoint kubectl get ep

kubectl get endpointslice

# 修改 svc 的类型来提供外部访问

# kubectl patch svc nginx -p '{"spec":{"type":"NodePort"}}'

# 当然可以 kubectl edit svc 进行修改

curl 192.168.1.20:32202

# svc 调优

```
# 首先看下kube-proxy的配置
# cat /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/opt/kube/bin/kube-proxy \
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=always
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target


# cat /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: "/etc/kubernetes/kube-proxy.kubeconfig"
# 根据clusterCIDR 判断集群内部和外部流量，配置clusterCIDR选项后，kube-proxy 会对访问 Service IP 的请求做 SNAT
clusterCIDR: "172.20.0.0/16"
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
healthzBindAddress: 0.0.0.0:10256
# hostnameOverride 值必须与 kubelet 的对应一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则
hostnameOverride: "10.0.1.201"
metricsBindAddress: 0.0.0.0:10249
mode: "ipvs"        #<-------  我们在最开始部署kube-proxy的时候就设定它的转发模式为ipvs，因为默认的iptables在存在大量svc的情况下性能很低
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: False
  syncPeriod: 30s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s






# 看下本地网卡，会有一个ipvs的虚拟网卡
# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens32: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:20:b8:39 brd fe:fe:fe:fe:fe:ff
    inet 10.0.1.202/24 brd 10.0.1.255 scope global noprefixroute ens32
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29fe:fe20:b839/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:91:ac:ce:13 brd fe:fe:fe:fe:fe:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: dummy0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 22:50:98:a6:f9:e4 brd fe:fe:fe:fe:fe:ff
5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500 qdisc noop state DOWN group default
    link/ether 96:6b:f0:25:1a:26 brd fe:fe:fe:fe:fe:ff
    inet 10.68.0.2/32 brd 10.68.0.2 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.68.0.1/32 brd 10.68.0.1 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.68.120.201/32 brd 10.68.120.201 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.68.50.42/32 brd 10.68.50.42 scope global kube-ipvs0
       valid_lft forever preferred_lft forever
    inet 10.68.18.121/32 brd 10.68.18.121 scope global kube-ipvs0    # <-------- SVC的IP配置在这里
       valid_lft forever preferred_lft forever
6: caliaeb0378f7a4@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether ee:ee:ee:ee:ee:ee brd fe:fe:fe:fe:fe:ff link-netnsid 0
    inet6 fe80::ecee:eefe:feee:eeee/64 scope link
       valid_lft forever preferred_lft forever
7: tunl0@NONe: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 172.20.247.0/32 brd 172.20.247.0 scope global tunl0
       valid_lft forever preferred_lft forever





# 来看下lvs的虚拟服务器列表
# ipvsadm -ln
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  172.17.0.1:20651 rr
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0
TCP  172.20.247.0:20651 rr
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0
TCP  10.0.1.202:20651 rr
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0
TCP  10.68.0.1:443 rr
  -> 10.0.1.201:6443              Masq    1      0          0
  -> 10.0.1.202:6443              Masq    1      3          0
TCP  10.68.0.2:53 rr
  -> 172.20.247.2:53              Masq    1      0          0
TCP  10.68.0.2:9153 rr
  -> 172.20.247.2:9153            Masq    1      0          0
TCP  10.68.18.121:80 rr                                        #<-----------   SVC转发Pod的明细在这里
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0
TCP  10.68.50.42:443 rr
  -> 172.20.217.71:4443           Masq    1      0          0
TCP  10.68.120.201:80 rr
  -> 10.0.1.201:80                Masq    1      0          0
  -> 10.0.1.202:80                Masq    1      0          0
TCP  10.68.120.201:443 rr
  -> 10.0.1.201:443               Masq    1      0          0
  -> 10.0.1.202:443               Masq    1      0          0
TCP  10.68.120.201:10254 rr
  -> 10.0.1.201:10254             Masq    1      0          0
  -> 10.0.1.202:10254             Masq    1      0          0
TCP  127.0.0.1:20651 rr
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0
UDP  10.68.0.2:53 rr
  -> 172.20.247.2:53              Masq    1      0          0

# 查看svc流量能否正常转发
ipvsadm -ln | grep -C5 10.68.97.176


```

# 除了直接用 cluster ip，以及上面说到的 NodePort 模式来访问 Service，我们还可以用 K8s 的 DNS 来访问

```
# 我们前面装好的CoreDNS，来提供K8s集群的内部DNS访问
# kubectl -n kube-system get deployment,pod|grep dns
deployment.apps/coredns                   1/1     1            1           5d2h
pod/coredns-d9b6857b5-tt7j2                    1/1     Running   1          27h

# coredns是一个DNS服务器，每当有新的Service被创建的时候，coredns就会添加该Service的DNS记录，然后我们通过serviceName.namespaceName就可以来访问到对应的pod了，下面来演示下：

# kubectl create ns test
# kubectl -n test run -it --rm busybox --image=registry.cn-shanghai.aliyuncs.com/acs/busybox:v1.29.2 -- sh    # --rm代表等我退出这个pod后，它会被自动删除，当作一个临时pod在用
If you don't see a command prompt, try pressing enter.
/ # ping nginx.default
PING nginx.default (10.68.18.121): 56 data bytes
64 bytes from 10.68.18.121: seq=0 ttl=64 time=0.096 ms
64 bytes from 10.68.18.121: seq=1 ttl=64 time=0.067 ms
64 bytes from 10.68.18.121: seq=2 ttl=64 time=0.065 ms
^C
--- nginx.default ping statistics ---
3 packets transmitted, 3 packets received, 0% packet loss
round-trip min/avg/max = 0.065/0.076/0.096 ms
/ # wget nginx.default
Connecting to nginx.default (10.68.18.121:80)
saving to 'index.html'
index.html           100% |********************************************************************|    22  0:00:00 ETA
'index.html' saved
/ # cat index.html
nginx-f89759699-bzwd2


```

# service 生产小技巧 通过 svc 来访问非 K8s 上的服务

# 这里我们挑选 node-2 节点，用 python 运行一个简易 web 服务器

```
# 在 node-2节点中启动一个python服务
[root@node-2 mnt]# python2 -m SimpleHTTPServer 9999
Serving HTTP on 0.0.0.0 port 9999 ...

回到master节点执行

curl 192.168.1.21:9999

```

# 创建 mysvc.yaml

```
# 注意我这里把两个资源的yaml写在一个文件内，在实际生产中，我们经常会这么做，方便对一个服务的所有资源进行统一管理，不同资源之间用"---"来分隔

apiVersion: v1
kind: Service
metadata:
  name: mysvc
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP

---

apiVersion: v1
kind: Endpoints
metadata:
  name: mysvc
  namespace: default
subsets:
- addresses:
  - ip: 192.168.1.21
    nodeName: 192.168.1.21
  ports:
  - port: 9999
    protocol: TCP


```

kubectl apply -f mysvc.yaml
kubectl get svc
kubectl describe svc mysvc

# 测试，请求 svc 地址

curl 10.68.59.154

# 生产中，通过 ingress 请求到 svc，然后到非 k8s 集群内部的服务上面去

kubectl get svc new-nginx -o yaml

# 外部网络如何访问到 Service 呢？

```
# 我们看下先创建的nginx service的yaml配置
# kubectl get svc nginx -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-11-25T03:55:05Z"
  labels:
    app: nginx
  managedFields:       # 在新版的K8s运行的资源配置里面，会输出这么多的配置信息，这里我们可以不用管它，实际我们在创建时，这些都是忽略的
  - apiVersion: v1
    fieldsType: FieldsV1
    fieldsV1:
      e:metadata:
        e:labels:
          .: {}
          e:app: {}
      e:spec:
        e:externalTrafficPolicy: {}
        e:ports:
          .: {}
          k:{"port":80,"protocol":"TCP"}:
            .: {}
            e:port: {}
            e:protocol: {}
            e:targetPort: {}
        e:selector:
          .: {}
          e:app: {}
        e:sessionAffinity: {}
        e:type: {}
    manager: kubectl
    operation: Update
    time: "2020-11-25T04:00:28Z"
  name: nginx
  namespace: default
  resourceVersion: "591029"
  selfLink: /api/v1/namespaces/default/services/nginx
  uid: 84fea557-e19d-486d-b879-13743c603091
spec:
  clusterIP: 10.68.18.121
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 20651     # 我们看下这里，它定义的一个nodePort配置，并分配了20651端口，因为我们先前创建时并没有指定这个配置，所以它是随机生成的
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}


# 我们看下apiserver的配置
# cat /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
  --allow-privileged=true \
  --anonymous-auth=false \
  --api-audiences=api,istio-ca \
  --authorization-mode=Node,RBAC \
  --bind-address=10.0.1.201 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --endpoint-reconciler-type=lease \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=https://10.0.1.201:2379,https://10.0.1.202:2379,https://10.0.1.203:2379 \
  --kubelet-certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --kubelet-client-certificate=/etc/kubernetes/ssl/kubernetes.pem \
  --kubelet-client-key=/etc/kubernetes/ssl/kubernetes-key.pem \
  --secure-port=6443 \
  --service-account-issuer=https://kubernetes.default.svc \
  --service-account-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-key-file=/etc/kubernetes/ssl/ca.pem \
  --service-cluster-ip-range=10.68.0.0/16 \
  --service-node-port-range=30000-32767 \      # 这就是NodePor随机生成端口的范围，这个在我们部署时就指定了
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --requestheader-client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --requestheader-allowed-names= \
  --requestheader-extra-headers-prefix=X-Remote-Extra- \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-username-headers=X-Remote-User \
  --proxy-client-cert-file=/etc/kubernetes/ssl/aggregator-proxy.pem \
  --proxy-client-key-file=/etc/kubernetes/ssl/aggregator-proxy-key.pem \
  --enable-aggregator-routing=true \
  --v=2
Restart=always
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target




# NodePort端口会在所在K8s的node节点上都生成一个同样的端口，这就使我们无论所以哪个node的ip接端口都能方便的访问到Service了，但在实际生产中，这个NodePort不建议经常使用，因为它会造成node上端口管理混乱，等用到了ingress后，你就不会想使用NodePort模式了，这个接下来会讲到

[root@node-1 ~]# ipvsadm -ln|grep -C6 20651
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
......
TCP  10.0.1.201:20651 rr      # 这里
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0


[root@node-2 mnt]# ipvsadm -ln|grep -C6 20651
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
......
TCP  10.0.1.202:20651 rr      # 这里
  -> 172.20.139.72:80             Masq    1      0          0
  -> 172.20.217.72:80             Masq    1      0          0


```

# 生产中 Service 的调优

```
# 先把nginx的pod数量调整为1，方便呆会观察
# kubectl scale deployment nginx --replicas=1
deployment.apps/nginx scaled

# 看下这个nginx的pod运行情况，-o wide显示更详细的信息，这里可以看到这个pod运行在node 203上面
# kubectl  get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE     IP              NODE         NOMINATED NODE   READINESS GATES
nginx-f89759699-qlc8q   1/1     Running   0          3h27m   172.20.139.72   10.0.1.203   <none>           <none>

# 我们先直接通过pod运行的node的IP来访问测试
[root@node-1 ~]# curl 10.0.1.203:20651
nginx-f89759699-qlc8q

# 可以看到日志显示这条请求的来源IP是203，而不是node-1的IP 10.0.1.201
# 注： kubectl logs --tail=1 代表查看这个pod的日志，并只显示倒数第一条
[root@node-1 ~]# kubectl logs --tail=1 nginx-f89759699-qlc8q
10.0.1.203 - - [25/Nov/2020:07:22:54 +0000] "GET / HTTP/1.1" 200 22 "-" "curl/7.29.0" "-"

# 再来通过201来访问
[root@node-1 ~]# curl 10.0.1.201:20651
nginx-f89759699-qlc8q

# 可以看到显示的来源IP非node节点的
[root@node-1 ~]# kubectl logs --tail=1 nginx-f89759699-qlc8q
172.20.84.128 - - [25/Nov/2020:07:23:18 +0000] "GET / HTTP/1.1" 200 22 "-" "curl/7.29.0" "-"

# 这就是一个虚拟网卡转发的
[root@node-1 ~]# ip a|grep -wC2 172.20.84.128
9: tunl0@NONe: <NOARP,UP,LOWER_UP> mtu 1440 qdisc noqueue state UNKNOWN group default qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
    inet 172.20.84.128/32 brd 172.20.84.128 scope global tunl0
       valid_lft forever preferred_lft forever

# 可以看下lvs的虚拟服务器列表，正好是转到我们要访问的pod上的
[root@node-1 ~]# ipvsadm -ln|grep -A1 172.20.84.128
TCP  172.20.84.128:20651 rr
  -> 172.20.139.72:80             Masq    1      0          0


详细处理流程如下：

* 客户端发送数据包 10.0.1.201:20651
* 10.0.1.201 用自己的IP地址替换数据包中的源IP地址（SNAT）
* 10.0.1.201 使用 pod IP 替换数据包上的目标 IP
* 数据包路由到 10.0.1.203 ，然后路由到 endpoint
* pod的回复被路由回 10.0.1.201
* pod的回复被发送回客户端

          client
             \ ^
              \ \
               v \
10.0.1.203 <--- 10.0.1.201
    | ^   SNAT
    | |   --->
    v |
 endpoint   

为避免这种情况， Kubernetes 具有保留客户端IP 的功能。设置 service.spec.externalTrafficPolicy 为 Local 会将请求代理到本地端点，不将流量转发到其他节点，从而保留原始IP地址。如果没有本地端点，则丢弃发送到节点的数据包，因此您可以在任何数据包处理规则中依赖正确的客户端IP。

# 设置 service.spec.externalTrafficPolicy 字段如下：
# kubectl patch svc nginx -p '{"spec":{"externalTrafficPolicy":"Local"}}'
service/nginx patched

# 现在通过非pod所在node节点的IP来访问是不通了
[root@node-1 ~]# curl 10.0.1.201:20651
curl: (7) Failed connect to 10.0.1.201:20651; Connection refused

# 通过所在node的IP发起请求正常
[root@node-1 ~]# curl 10.0.1.203:20651
nginx-f89759699-qlc8q

# 可以看到日志显示的来源IP就是201，这才是我们想要的结果
[root@node-1 ~]# kubectl logs --tail=1 nginx-f89759699-qlc8q
10.0.1.201 - - [25/Nov/2020:07:33:42 +0000] "GET / HTTP/1.1" 200 22 "-" "curl/7.29.0" "-"

# 去掉这个优化配置也很简单
# kubectl patch svc nginx -p '{"spec":{"externalTrafficPolicy":""}}'



```

root@node-1:/opt/k8s/service# curl 192.168.1.20:32202
[2024-05-19 09:23:09] default-new-nginx-854cf59647-zv6lr-172.20.139.116

kubectl logs new-nginx-854cf59647-zv6lr --tail=1

kubectl get svc new-nginx -o yaml
kubectl patch svc new-nginx -p '{"spec":{"externalTrafficPolicy":"Local"}}'

curl 192.168.1.22:32202
curl 192.168.1.23:32202
这样就会请求对应节点上的服务 pod，不会走 nat 转换
curl 192.168.1.21:32202 # 这个请求不通，因为服务 pod 并没有在 21 这个机器上

# 注意：这样会带来个什么问题呢，如是一旦 pod 发生重启飘移到了另一台 node 节点上，而你用的 IP 还是 203 的话就会访问不到服务了，这个利弊需要自己权衡，解决方法可以用 nodeSelector 来将服务的 pod 固定在哪几台 node 上运行，这样 ip 还是在我们控制的范围了，我们现在就来试试 nodeSelector 的效果吧

```
# 修改好yaml配置
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
      nodeSelector:           # <--- 这里
        apps/nginx: "true"    # <--- 基于这个label来选择
status: {}


```

# 然后我们这里在需要指定运行的 node 上打上 label

```
# kubectl label node 10.0.1.201 apps/nginx=true
node/10.0.1.201 labeled

# kubectl get node 10.0.1.201 --show-labels |grep nginx
10.0.1.201   Ready    master   5d3h   v1.18.12   apps/nginx=true,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,cb/ingress-controller-ready=true,kubernetes.io/arch=amd64,kubernetes.io/hostname=10.0.1.201,kubernetes.io/os=linux,kubernetes.io/role=master,kubevirt.io/schedulable=true


```
