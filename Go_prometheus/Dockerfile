# 使用官方 Go 镜像作为基础镜像
FROM registry.cn-hangzhou.aliyuncs.com/abelcontainer/golang:1.22.4 AS builder

# 设置 Go Modules 代理
RUN go env -w GOPROXY=https://goproxy.io,direct

# 设置工作目录
WORKDIR /app

# 将 go.mod 和 go.sum 复制到工作目录
COPY go.mod go.sum ./

# 下载依赖
RUN go mod download

# 将源代码复制到工作目录
COPY . .

# 编译 Go 应用
RUN go build -o myapp

# 使用相同的基础镜像运行应用
FROM registry.cn-hangzhou.aliyuncs.com/abelcontainer/golang:1.22.4

# 创建一个非 root 用户
RUN useradd -ms /bin/sh myuser

# 创建工作目录
WORKDIR /app

# 从构建阶段复制编译好的二进制文件
COPY --from=builder /app/myapp /app/

# 设置非 root 用户
USER myuser

# 公开应用运行的端口
EXPOSE 8090

# 运行应用
ENTRYPOINT ["/app/myapp"]
