https://blog.csdn.net/weixin_43354152/article/details/139400133

go mod init prometheus

# 用国内加速代理下载包

export GOPROXY=https://goproxy.cn

go get github.com/prometheus/client_golang/prometheus
go get github.com/prometheus/client_golang/prometheus/promhttp

go get 命令来下载这些包，go get 会自动更新 go.mod 文件，添加依赖项及其版本信息。

go run main.go

查看暴露的指标

http://127.0.0.1:8090/metrics

增加一些请求数

http://127.0.0.1:8090/ping

go install github.com/cweill/gotests/gotests@v1.6.0

# 这里我们可以修改 prometheus 的配置，来监控我们自定义的这个服务，或者也可以参照博哥之前的课程，用 serviceMonitor 来暴露指标

这个没有实际去操作

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: simple_server
    static_configs:
      - targets: ["localhost:8090"]


```

# 需要跟代码中 listen 启动的监听端口号一致

docker build -t my-go-app .
docker run -p 9096:9096 my-go-app

docker run -d -p 9096:9096 my-go-app

curl 192.168.1.20:9096/ping
curl 192.168.1.20:9096/metrics

# 生成 harbor 私有镜像仓库的 secret

kubectl -n goapp create secret docker-registry goapp --docker-server=harbor.boge.com --docker-username=boge --docker-password=Boge@666 --docker-email=ops@boge.com

docker tag my-go-app:latest harbor.boge.com/goapp/my-go-app:latest

这里我遇到了一个问题

```
Prometheus的targets发现不到我的 服务呢，我是跟着老师的课程学到这里，打镜像 部署 deployment， service，servicemonitor也部署上了，查服务也能curl到


yaml如下：deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-go-app
  namespace: goapp
  labels:
    app: my-go-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-go-app
  template:
    metadata:
      labels:
        app: my-go-app
    spec:
      containers:
      - name: my-go-app
        image: harbor.boge.com/goapp/my-go-app:latest
        ports:
        - containerPort: 9096
      imagePullSecrets:
      - name: goapp

service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-go-app-service
  namespace: goapp
  labels:
    app: my-go-app
spec:
  type: ClusterIP
  ports:
  - name: my-go-app
    port: 9096
    targetPort: 9096
  selector:
    app: my-go-app

servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-go-app-monitor
  labels:
    app: my-go-app
  namespace: goapp
spec:
  jobLabel: app
  selector:
    matchLabels:
      app: my-go-app
  endpoints:
  - port: my-go-app
    path: /metrics
    interval: 30s
  namespaceSelector:
    matchNames:
    - goapp




```

我都换到 monitoring 这个命名空间下去部署 我的服务就 work 了， 我自己的命名空间 namespace: goapp 不行，
这里我还差一些配置，暂时还不知

# 做些尝试

配置了一下 role 和 rolebinding 后生效了，参考 folder '监控配置'
