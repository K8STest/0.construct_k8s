# 解决不能下载镜像的问题

https://www.bilibili.com/video/BV1Py411877t/?spm_id_from=333.999.top_right_bar_window_dynamic.content.click&vd_source=b9464c0ecfa02375d34808b56bd7e53f

# 介绍

经过多方面测试，log-pilot 对现有业务 pod 侵入性很小，只需要在原有 pod 的内传入几行 env 环境变量，即可对此 pod 相关的日志进行收集，已经测试了后端接收的工具有 logstash、elasticsearch、kafka、redis、file，均 OK，下面开始部署整个日志收集环境。

我们这里用一个 tomcat 服务来模拟业务服务，用 log-pilot 分别收集它的 stdout 以及容器内的业务数据日志文件到指定后端存储，2023 课程会以真实生产的一个日志传输架构来做测试：

```
模拟业务服务   日志收集       日志存储     消费日志         消费目的服务
tomcat ---> log-pilot ---> kafka ---> filebeat ---> elasticsearch ---> kibana

```

# 操作

# 准备好 docker-compose 命令

ln -svf /etc/kubeasz/bin/docker-compose /usr/bin/docker-compose
docker version

cd /root/kafka/

# 测试用的 kafka 服务

# 这里还有一系列操作 镜像不能下载的问题 ，可以参考截图

https://github.com/AbelLiuKai/docker_image_pusher/actions/workflows/docker.yaml
https://www.bilibili.com/video/BV1Py411877t/?spm_id_from=333.788.top_right_bar_window_dynamic.content.click&vd_source=b9464c0ecfa02375d34808b56bd7e53f
https://cr.console.aliyun.com/cn-hangzhou/instance/repositories

docker-compose.yml

```
version: '2'
services:
  zookeeper:
    image: registry.cn-hangzhou.aliyuncs.com/abelcontainer/zookeeper:latest
    ports:
      - "2181:2181"
    restart: unless-stopped

  kafka:
    image: registry.cn-hangzhou.aliyuncs.com/abelcontainer/kafka:2.8.1
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 192.168.1.20
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /mnt/kafka:/kafka
    restart: unless-stopped
    depends_on:
      - zookeeper


```

docker-compose version
docker-compose ps
ss -tlnp|grep 9092

```
Recreating kafka-docker-compose_kafka_1 ... done
Starting kafka-docker-compose_zookeeper_1 ... done

# 2. result look:

# docker-compose ps


# 3. run test-docker

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -e HOST_IP=192.168.1.20 -e ZK=192.168.1.20:2181 -i -t registry.cn-hangzhou.aliyuncs.com/abelcontainer/kafka:2.8.1 /bin/bash

# 4. list topic

bash-4.4# kafka-topics.sh --zookeeper 192.168.1.20:2181 --list
tomcat-access
tomcat-syslog

# 5. consumer topic data:

bash-4.4# kafka-console-consumer.sh --bootstrap-server 192.168.1.20:9092 --topic tomcat-access --from-beginning

```

# 部署日志收集服务 log-pilot (适配容器运行时 Containerd)

cd /opt/k8s/kafka/
vim log-pilot.yaml

```
# 先创建一个namespace
kubectl create ns ns-elastic

# 部署yaml


apiVersion: v1
kind: ConfigMap
metadata:
  name: log-pilot5-configuration
  namespace: ns-elastic
data:
  logging_output: "kafka"
  kafka_brokers: "192.168.1.20:9092"
  kafka_version: "0.10.0"
  kafka_topics: "tomcat-syslog,tomcat-access"
  #kafka_username: "user"
  #kafka_password: "bogeit"

---
apiVersion: v1
data:
  filebeat.tpl: |+
    {{range .configList}}
    - type: log
      enabled: true
      paths:
          - {{ .HostDir }}/{{ .File }}
      scan_frequency: 5s
      fields_under_root: true
      {{if eq .Format "json"}}
      json.keys_under_root: true
      {{end}}
      fields:
          {{range $key, $value := .Tags}}
          {{ $key }}: {{ $value }}
          {{end}}
          {{range $key, $value := $.container}}
          {{ $key }}: {{ $value }}
          {{end}}
      tail_files: false
      close_inactive: 2h
      close_eof: false
      close_removed: true
      clean_removed: true
      close_renamed: false

    {{end}}

kind: ConfigMap
metadata:
  name: filebeat5-tpl
  namespace: ns-elastic

---
apiVersion: v1
data:
  config.filebeat: |+
    #!/bin/sh

    set -e

    FILEBEAT_CONFIG=/etc/filebeat/filebeat.yml
    if [ -f "$FILEBEAT_CONFIG" ]; then
        echo "$FILEBEAT_CONFIG has been existed"
        exit
    fi

    mkdir -p /etc/filebeat/prospectors.d

    assert_not_empty() {
        arg=$1
        shift
        if [ -z "$arg" ]; then
            echo "$@"
            exit 1
        fi
    }

    cd $(dirname $0)

    base() {
    cat >> $FILEBEAT_CONFIG << EOF
    path.config: /etc/filebeat
    path.logs: /var/log/filebeat
    path.data: /var/lib/filebeat/data
    filebeat.registry_file: /var/lib/filebeat/registry
    filebeat.shutdown_timeout: ${FILEBEAT_SHUTDOWN_TIMEOUT:-0}
    logging.level: ${FILEBEAT_LOG_LEVEL:-info}
    logging.metrics.enabled: ${FILEBEAT_METRICS_ENABLED:-false}
    logging.files.rotateeverybytes: ${FILEBEAT_LOG_MAX_SIZE:-104857600}
    logging.files.keepfiles: ${FILEBEAT_LOG_MAX_FILE:-10}
    logging.files.permissions: ${FILEBEAT_LOG_PERMISSION:-0600}
    ${FILEBEAT_MAX_PROCS:+max_procs: ${FILEBEAT_MAX_PROCS}}
    setup.template.name: "${FILEBEAT_INDEX:-filebeat}"
    setup.template.pattern: "${FILEBEAT_INDEX:-filebeat}-*"
    filebeat.config:
        prospectors:
            enabled: true
            path: \${path.config}/prospectors.d/*.yml
            reload.enabled: true
            reload.period: 10s
    EOF
    }

    es() {
    if [ -f "/run/secrets/es_credential" ]; then
        ELASTICSEARCH_USER=$(cat /run/secrets/es_credential | awk -F":" '{ print $1 }')
        ELASTICSEARCH_PASSWORD=$(cat /run/secrets/es_credential | awk -F":" '{ print $2 }')
    fi

    if [ -n "$ELASTICSEARCH_HOSTS" ]; then
        ELASTICSEARCH_HOSTS=$(echo $ELASTICSEARCH_HOSTS|awk -F, '{for(i=1;i<=NF;i++){printf "\"%s\",", $i}}')
        ELASTICSEARCH_HOSTS=${ELASTICSEARCH_HOSTS%,}
    else
        assert_not_empty "$ELASTICSEARCH_HOST" "ELASTICSEARCH_HOST required"
        assert_not_empty "$ELASTICSEARCH_PORT" "ELASTICSEARCH_PORT required"
        ELASTICSEARCH_HOSTS="\"$ELASTICSEARCH_HOST:$ELASTICSEARCH_PORT\""
    fi

    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.elasticsearch:
        hosts: [$ELASTICSEARCH_HOSTS]
        index: ${ELASTICSEARCH_INDEX:-filebeat}-%{+yyyy.MM.dd}
        ${ELASTICSEARCH_SCHEME:+protocol: ${ELASTICSEARCH_SCHEME}}
        ${ELASTICSEARCH_USER:+username: ${ELASTICSEARCH_USER}}
        ${ELASTICSEARCH_PASSWORD:+password: ${ELASTICSEARCH_PASSWORD}}
        ${ELASTICSEARCH_WORKER:+worker: ${ELASTICSEARCH_WORKER}}
        ${ELASTICSEARCH_PATH:+path: ${ELASTICSEARCH_PATH}}
        ${ELASTICSEARCH_BULK_MAX_SIZE:+bulk_max_size: ${ELASTICSEARCH_BULK_MAX_SIZE}}
    EOF
    }

    default() {
    echo "use default output"
    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.console:
        pretty: ${CONSOLE_PRETTY:-false}
    EOF
    }

    file() {
    assert_not_empty "$FILE_PATH" "FILE_PATH required"

    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.file:
        path: $FILE_PATH
        ${FILE_NAME:+filename: ${FILE_NAME}}
        ${FILE_ROTATE_SIZE:+rotate_every_kb: ${FILE_ROTATE_SIZE}}
        ${FILE_NUMBER_OF_FILES:+number_of_files: ${FILE_NUMBER_OF_FILES}}
        ${FILE_PERMISSIONS:+permissions: ${FILE_PERMISSIONS}}
    EOF
    }

    logstash() {
    assert_not_empty "$LOGSTASH_HOST" "LOGSTASH_HOST required"
    assert_not_empty "$LOGSTASH_PORT" "LOGSTASH_PORT required"

    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.logstash:
        hosts: ["$LOGSTASH_HOST:$LOGSTASH_PORT"]
        index: ${FILEBEAT_INDEX:-filebeat}-%{+yyyy.MM.dd}
        ${LOGSTASH_WORKER:+worker: ${LOGSTASH_WORKER}}
        ${LOGSTASH_LOADBALANCE:+loadbalance: ${LOGSTASH_LOADBALANCE}}
        ${LOGSTASH_BULK_MAX_SIZE:+bulk_max_size: ${LOGSTASH_BULK_MAX_SIZE}}
        ${LOGSTASH_SLOW_START:+slow_start: ${LOGSTASH_SLOW_START}}
    EOF
    }

    redis() {
    assert_not_empty "$REDIS_HOST" "REDIS_HOST required"
    assert_not_empty "$REDIS_PORT" "REDIS_PORT required"

    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.redis:
        hosts: ["$REDIS_HOST:$REDIS_PORT"]
        key: "%{[fields.topic]:filebeat}"
        ${REDIS_WORKER:+worker: ${REDIS_WORKER}}
        ${REDIS_PASSWORD:+password: ${REDIS_PASSWORD}}
        ${REDIS_DATATYPE:+datatype: ${REDIS_DATATYPE}}
        ${REDIS_LOADBALANCE:+loadbalance: ${REDIS_LOADBALANCE}}
        ${REDIS_TIMEOUT:+timeout: ${REDIS_TIMEOUT}}
        ${REDIS_BULK_MAX_SIZE:+bulk_max_size: ${REDIS_BULK_MAX_SIZE}}
    EOF
    }

    kafka() {
    assert_not_empty "$KAFKA_BROKERS" "KAFKA_BROKERS required"
    KAFKA_BROKERS=$(echo $KAFKA_BROKERS|awk -F, '{for(i=1;i<=NF;i++){printf "\"%s\",", $i}}')
    KAFKA_BROKERS=${KAFKA_BROKERS%,}

    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.kafka:
        hosts: [$KAFKA_BROKERS]
        topic: '%{[topic]}'
        codec.format:
            string: '%{[message]}'
        ${KAFKA_VERSION:+version: ${KAFKA_VERSION}}
        ${KAFKA_USERNAME:+username: ${KAFKA_USERNAME}}
        ${KAFKA_PASSWORD:+password: ${KAFKA_PASSWORD}}
        ${KAFKA_WORKER:+worker: ${KAFKA_WORKER}}
        ${KAFKA_PARTITION_KEY:+key: ${KAFKA_PARTITION_KEY}}
        ${KAFKA_PARTITION:+partition: ${KAFKA_PARTITION}}
        ${KAFKA_CLIENT_ID:+client_id: ${KAFKA_CLIENT_ID}}
        ${KAFKA_METADATA:+metadata: ${KAFKA_METADATA}}
        ${KAFKA_BULK_MAX_SIZE:+bulk_max_size: ${KAFKA_BULK_MAX_SIZE}}
        ${KAFKA_BROKER_TIMEOUT:+broker_timeout: ${KAFKA_BROKER_TIMEOUT}}
        ${KAFKA_CHANNEL_BUFFER_SIZE:+channel_buffer_size: ${KAFKA_CHANNEL_BUFFER_SIZE}}
        ${KAFKA_KEEP_ALIVE:+keep_alive ${KAFKA_KEEP_ALIVE}}
        ${KAFKA_MAX_MESSAGE_BYTES:+max_message_bytes: ${KAFKA_MAX_MESSAGE_BYTES}}
        ${KAFKA_REQUIRE_ACKS:+required_acks: ${KAFKA_REQUIRE_ACKS}}
    EOF
    }

    count(){
    cat >> $FILEBEAT_CONFIG << EOF
    $(base)
    output.count:
    EOF
    }

    if [ -n "$FILEBEAT_OUTPUT" ]; then
        LOGGING_OUTPUT=$FILEBEAT_OUTPUT
    fi

    case "$LOGGING_OUTPUT" in
        elasticsearch)
            es;;
        logstash)
            logstash;;
        file)
            file;;
        redis)
            redis;;
        kafka)
            kafka;;
        count)
            count;;
        *)
            default
    esac



kind: ConfigMap
metadata:
  name: config5.filebeat
  namespace: ns-elastic

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-pilot5
  namespace: ns-elastic
  labels:
    k8s-app: log-pilot5
spec:
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      k8s-app: log-pilot5
  template:
    metadata:
      labels:
        k8s-app: log-pilot5
    spec:
      imagePullSecrets:
      - name: aliyun-registry-secret
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      - effect: NoSchedule
        key: ToBeDeletedByClusterAutoscaler
        operator: Exists
      containers:
      - name: log-pilot5
        image: registry.cn-hangzhou.aliyuncs.com/abelcontainer/log-pilot-filebeat:0.9.1
        imagePullPolicy: IfNotPresent
        env:
          - name: "LOGGING_OUTPUT"
            valueFrom:
              configMapKeyRef:
                name: log-pilot5-configuration
                key: logging_output
          - name: "KAFKA_BROKERS"
            valueFrom:
              configMapKeyRef:
                name: log-pilot5-configuration
                key: kafka_brokers
          - name: "KAFKA_VERSION"
            valueFrom:
              configMapKeyRef:
                name: log-pilot5-configuration
                key: kafka_version
          #- name: "KAFKA_USERNAME"
          #  valueFrom:
          #    configMapKeyRef:
          #      name: log-pilot5-configuration
          #      key: kafka_username
          #- name: "KAFKA_PASSWORD"
          #  valueFrom:
          #    configMapKeyRef:
          #      name: log-pilot5-configuration
          #      key: kafka_password
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        volumeMounts:
        - name: sock
          mountPath: /var/run/containerd/containerd.sock
        - name: logs
          mountPath: /var/log/filebeat
        - name: state
          mountPath: /var/lib/filebeat
        - name: root
          mountPath: /host
          readOnly: true
        - name: localtime
          mountPath: /etc/localtime
        - name: config-volume
          mountPath: /etc/filebeat/config
        - name: filebeat-tpl
          mountPath: /pilot/filebeat.tpl
          subPath: filebeat.tpl
        - name: config-filebeat
          mountPath: /pilot/config.filebeat
          subPath: config.filebeat
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/containerd/containerd.sock
          type: Socket
      - name: logs
        hostPath:
          path: /var/log/filebeat
          type: DirectoryOrCreate
      - name: state
        hostPath:
          path: /var/lib/filebeat
          type: DirectoryOrCreate
      - name: root
        hostPath:
          path: /
          type: Directory
      - name: localtime
        hostPath:
          path: /etc/localtime
          type: File
      - name: config-volume
        configMap:
          name: log-pilot5-configuration
          items:
          - key: kafka_topics
            path: kafka_topics
      - name: filebeat-tpl
        configMap:
          name: filebeat5-tpl
      - name: config-filebeat
        configMap:
          name: config5.filebeat
          defaultMode: 0777

```

cd /opt/k8s/kafka/log-pilot-filebeat-master/

docker build -t williamguozi/log-pilot-filebeat:containerd.ori .

docker build -t williamguozi/log-pilot-filebeat:containerd .

# 关键操作

这里的 镜像文件不能现在下来，
https://github.com/WilliamGuozi/log-pilot-filebeat/tree/master

cd /opt/k8s/kafka/log-pilot-filebeat-master/
需要根据作者的原数据进行打包镜像 ，
先 Dockerfile.docker 用这个打出来镜像

然后用 Dockerfile.containerd 这个打镜像

# 需要把镜像导出来

docker save log-pilot-filebeat:0.9.1 > log-pilot-filebeat-0.9.1.tar

# 需要把镜像文件导入到 containerd

ctr -n k8s.io images import log-pilot-filebeat-0.9.1.tar

# 把镜像导入到阿里云上

docker tag log-pilot-filebeat:0.9.1 registry.cn-hangzhou.aliyuncs.com/abelcontainer/log-pilot-filebeat:0.9.1
docker push registry.cn-hangzhou.aliyuncs.com/abelcontainer/log-pilot-filebeat:0.9.1

kubectl create secret docker-registry aliyun-registry-secret --namespace=ns-elastic --docker-server=registry.cn-hangzhou.aliyuncs.com --docker-username=abelliu001 --docker-password='ACSdev312!@' --docker-email='13478440986@163.com'

# 部署测试用的 tomcat 服务

vim tomcat-test.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: tomcat
  name: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  template:
    metadata:
      labels:
        app: tomcat
    spec:
      tolerations:
      - key: "node-role.kubernetes.io/master"
        effect: "NoSchedule"
      containers:
      - name: tomcat
        image: "tomcat:7.0"
        env:      # 注意点一，添加相应的环境变量（下面收集了两块日志1、stdout 2、/usr/local/tomcat/logs/catalina.*.log）
        - name: aliyun_logs_tomcat-syslog   # 如日志发送到es，那index名称为 tomcat-syslog  aliyun_logs这段是固定的
          value: "stdout"
        - name: aliyun_logs_tomcat-access   # 如日志发送到es，那index名称为 tomcat-access
          value: "/usr/local/tomcat/logs/catalina.*.log"
        volumeMounts:   # 注意点二，对pod内要收集的业务日志目录需要进行共享，可以收集多个目录下的日志文件
          - name: tomcat-log
            mountPath: /usr/local/tomcat/logs
      volumes:  # 如此节点上会有 对应目录创建， log pilot就会去收集
        - name: tomcat-log
          emptyDir: {}

```

# 4、部署 es 和 kibana

这里的 elasticsearch 和 kibana 服务我们就用上节课部署的服务接着使用即可

第 25 关 利用 operator 部署生产级别的 Elasticserach 集群和 kibana

# 5、部署 filebeat 服务

```
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: filebeat-config-test
data:
  filebeat.yml: |
    filebeat.inputs:
    - type: kafka
      hosts: ["192.168.1.20:9092"]
      topics: ["tomcat-syslog"]
      group_id: "filebeat"
      #username: "$ConnectionString"
      #password: "<your connection string>"
      ssl.enabled: false
    filebeat.shutdown_timeout: 30s
    output:
      elasticsearch:
        enabled: true
        hosts: ['quickstart-es-http.es:9200']  #  kubectl -n es get svc   service 名字 + ns名字
        username: elastic    # 之前已经有了，获取 es账号密码的方式
        password: KJSZTW82853RFC6s022f1WrT
        indices:
          - index: "test-%{+yyyy.MM.dd}"
    logging.level: info


---

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app-name: filebeat
  name: filebeat-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app-name: filebeat
  template:
    metadata:
      labels:
        app-name: filebeat
    spec:
      containers:
      - command:
        - filebeat
        - -e
        - -c
        - /etc/filebeat.yml
        env:
        - name: TZ
          value: Asia/Shanghai
        image: docker.elastic.co/beats/filebeat:8.11.3   # docker.io/elastic/filebeat:8.11.3 版本要跟 kibana 版本一致  https://www.docker.elastic.co/r/beats/filebeat?limit=50&offset=50&show_snapshots=false
        imagePullPolicy: IfNotPresent
        name: filebeat
        resources:
          limits:
            cpu: 100m
            memory: 200M
          requests:
            cpu: 100m
            memory: 200M
        volumeMounts:
        - mountPath: /etc/filebeat.yml
          name: filebeat-config
          subPath: filebeat.yml
      volumes:
      - configMap:
          defaultMode: 420
          name: filebeat-config-test
        name: filebeat-config

```

docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -e HOST_IP=192.168.1.20 -e ZK=192.168.1.20:2181 -i -t registry.cn-hangzhou.aliyuncs.com/abelcontainer/kafka:2.8.1 /bin/bash

kafka-topics.sh --zookeeper 192.168.1.20:2181 --list

kafka-console-consumer.sh --bootstrap-server 192.168.1.20:9092 --topic tomcat-access --from-beginning
