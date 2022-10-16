---
title: "模块三：Docker核心技术"
date: 2022-10-04T11:15:25+08:00
tags: ["geektime-cn"]
---


## 1.从系统架构谈起

### 微服务改造

除了遵循分离微服务的方法建议，也要从公司的人力成本，维护团队管辖的功能范围进行考虑拆分

### 微服务间通讯

- 点对点通讯
- 使用api网关统一管理访问（包含身份认证、鉴权能力）



## 2.理解Docker

## 3.Docker核心技术

## 4.容器网络

## 5.Dockerfile的最佳实践
 
多段构建

docker常用指令
- from as ...
- run
- add：从源地址复制到目的地址，add会做解压缩操作
- copy：从源地址


## 课后练习 3.1
- Memory 子系统练习
- 在 cgroup memory 子系统目录中创建目录结构
  ```bash
        cd /sys/fs/cgroup/memory
        mkdir memorydemo
        cd memorydemo
   ```
- 运行 malloc（在 linux 机器 make build）
- 查看内存使用情况
  ```bash
    watch 'ps -aux|grep malloc|grep -v grep‘
  ```

- 通过 cgroup 限制 memory
  - 把进程添加到 cgroup 进程配置组
    ```bash
        echo ps -ef|grep malloc |grep -v grep|awk ‘{print $2}’ > cgroup.procs
    ```
  - 设置 memory.limit_in_bytes
    ```bash
        echo 104960000 > memory.limit_in_bytes
    ```
- 等待进程被 oom kill


## 思考题：容器的劣势
1. 加了容器层对性能的影响
2. 容器内的问题排查，集成测试时日志、监控系统的建设

## 课后练习 3.2

### 1. 构建本地镜像
```bash
  docker build -f ./module3-Dockerfile
```

- 报错记录1: 无法拷贝上级目录的文
  ```bash
  Step 3/9 : COPY ../module2/  /go/src
  COPY failed: forbidden path outside the build context: ../module2/ ()
  ```
  解决方案：移动dockerfile到父级目录，然后重新执行build


- 报错记录2: mac os下无法正常build
  ```bash
  failed to solve with frontend dockerfile.v0: failed to read dockerfile: open /var/lib/docker/tmp/buildkit-mount3040494485/module3-Dockerfile: no such file or directory
  ```
  解决方案：
  ```bash
  export DOCKER_BUILDKIT=0
  export COMPOSE_DOCKER_CLI_BUILD=0
  ```


### 2.编写 Dockerfile 将练习 2.2 编写的 httpserver 容器化
Dockerfile如下
```Dockerfile
# stage1: build binary
FROM golang:1.19.2-alpine3.16 AS build
RUN apk add --no-cache git
COPY ./module2/  /go/src/module2
WORKDIR /go/src/module2
RUN go build -o /bin/server

# stage2: build imag
# FROM scratch
# error exec ./server: no such file or directory
FROM alpine:latest
WORKDIR /
COPY  --from=build /bin/server /bin/server
ENV VERSION v1
CMD ["/bin/server"]

```
执行如下命令构建
```bash
docker build -f module3-Dockerfile . --target build -t module3-build-stage
docker build -f module3-Dockerfile . -t giuliao/module3-httpserver:v1.0
docker rmi module3-build-stage
```
- 报错记录：用scratch做为基础镜像编译出来的镜像会报如下错误，暂时没有找到解决方案
  ```bash
  exec /bin/server: no such file or directory
  ```


### 3.将镜像推送至 docker 官方镜像仓库

```bash
docker push giuliao/module3-httpserver:v1.0
```
- dockerhub访问[链接](https://hub.docker.com/repository/docker/giuliao/module3-httpserver)

### 4.通过 docker 命令本地启动 httpserver

启动命令
```bash
 docker run -it -p 3000:3000 giuliao/module3-httpserver:v1.0
```
日志记录
```bash
I1016 03:07:24.903419       1 main.go:129] response status code 200, ip addr 172.17.0.1
I1016 03:07:25.221227       1 main.go:129] response status code 200, ip addr 172.17.0.1
2022/10/16 03:07:34 http: superfluous response.WriteHeader call from main.(*loggingResponseWriter).WriteHeader (main.go:81)
I1016 03:07:34.660287       1 main.go:129] response status code 200, ip addr 172.17.0.1
I1016 03:07:34.928845       1 main.go:129] response status code 200, ip addr 172.17.0.
```

### 5.通过 nsenter 进入容器查看 IP 配置

查看容器在宿主机的pid
```bash
docker inspect -f '{{.State.Pid}}' <containerID>
```
- [参考链接](https://www.cnblogs.com/xy14/p/12002816.html)

```
nsenter -t <pid> -n ip addr
```