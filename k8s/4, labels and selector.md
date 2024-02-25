```
kubectl get po --show-labels
kubectl get po --show-labels -n kube-system

根据标签去找 pod
kubectl get po -l app=hello

kubectl get po -A -l app=hello --show-labels

多值匹配
kubectl get po -l 'test in (1.0.0, 1.1.0, 1.2.0)'

kubectl get po -l test!=1.0.0,type=app

```

```nginx-po.yaml

apiVersion: v1 #api文档版本
kind: Pod  #资源对象类型Deployment StatefulSet 一类的对象
metadata:
  name: nginx-po
  labels:
    type: app  #自定义label标签，名字为type，值为 app
    test: 1.0.0  #自定义label标签描述， pod的版本号
  namespace: 'default'
spec:  #期望pod 按照这里的信息进行创建
  containers:  #对于容器中pod的描述
  - name: nginx   #容器的名称
    image: nginx:1.7.9   #容器的镜像
    imagePullPolicy: IfNotPresent   #如果本地有就用本地的额，如果没有，就远程的拉取
    startupProbe: #容器应用启动探针
      #httpGet: #探测方式 基于http请求
      #  path: /index.html  #http 探测路径
      #tcpSocket:
      #  port: 80  # 请求端口
      exec:
        command:
        - sh
        - -c
        - "sleep 5; echo 'success' > /init"
      failureThreshold: 3 # 失败多少次才算失败
      periodSeconds: 10 #间隔时间
      successThreshold: 1  #多少次检测成功，算成功
      timeoutSeconds: 10
    command:  #指定容器启动时候执行的命令
    - nginx
    - -g
    - 'daemon off;' #nginx -g 'daemon off;'
    workingDir: /usr/share/nginx/html   #定义容器启动后工作目录
    ports:
    - name: http #端口名称
      containerPort: 80 #描述容器内要暴露什么端口
      protocol: TCP #描述改端口是基于哪种协议通信的
    env: #环境变量
    - name: JVM_OPTS  #环境变量名称
      value: '-Xms128m -Xmx128m'

    resources:
      requests: # 最少需要多少资源
        cpu: 100m # 限制CPU最少使用0.1个核心  1000m==一个核心
        memory: 128Mi #内存最多使用128M
      limits: #最多的限制
        cpu: 200m
        memory: 256Mi
  restartPolicy: OnFailure #重启策略，只有失败才会重启

```
