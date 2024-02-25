```
强制删除pod
kubectl delete pod nginx-po --grace-period=0 --force
```

```

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
    lifecycle:
      postStart:
        exec:
          command:
          - sh
          - -c
          - "echo '<h1>pre stop</h1>'> /usr/share/nginx/html/prestop.html"
      preStop:
        exec:
          command:
          - sh
          - -c
          - "sleep 50; echo 'sleep over' >> /usr/share/nginx/html/prestop.html"
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
        cpu: 100m # 限制CPU最少使用0.1个
        memory: 128Mi #内存最多使用128M
      limits: #最多的限制
        cpu: 200m
        memory: 256Mi
  restartPolicy: OnFailure #重启策略，只有失败才会重启
```

```
查看html里面的内容
curl 10.244.36.96/prestop.html

持续监听状态，然后输出到控制台
kubectl get po -w

监听命令执行的时间
time kubectl delete po nginx-po

```
