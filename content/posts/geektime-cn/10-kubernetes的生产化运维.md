---
title: "模块十：kubernetes的生产化运维"
date: 2022-12-04T15:15:25+08:00
tags: ["geektime-cn"]
---


## 1.镜像仓库的原理与搭建

## 2.镜像安全


## 3.基于kubernetes的DevOps


- 计划、开发、测试、生产运维
- git版本管理，镜像版本管理
- 通过代码变更触发生产部署更新，gitOps

## 4.基于Github Action、Jenkins的自动化流水线

docker in docker方案
- 挂载docker.sock到容器内
- 使用谷歌的kaniko方案


## 5.基于声明式API的自动化流水线：Tekton

## 6.连续交付工具：Argo CD

## 7.日志收集与分析

- 日志收集工具Grafana loki


## 8.构建监控系统


## 课后练习

本周没有好好实操部署prometheus，只完成过了http server的改造，代码参考孟老师视频内的内容。完整代码[链接](https://github.com/Giuliao/cn-practices/blob/main/module2/main.go)

1. 为 HTTPServer 添加 0-2 秒的随机延时；

```golang
func DelayedHello(w http.ResponseWriter, r *http.Request) {
	glog.V(4).Info("entering root handler")
	timer := metrics.NewTimer()
	defer timer.ObserveTotal()

	user := r.URL.Query().Get("user")
	delay := randRange(0, 2000)
    // 0～2s的时延
	time.Sleep(time.Duration(delay) * time.Millisecond)
	if user != "" {
		io.WriteString(w, fmt.Sprintf("hello [%s]\n", user))
	} else {
		io.WriteString(w, "hello [stranger]\n")
	}
	io.WriteString(w, "===================Details of the http request header:============\n")
	for k, v := range r.Header {
		io.WriteString(w, fmt.Sprintf("%s=%s\n", k, v))
	}
	glog.V(4).Infof("Respond in %d ms", delay)
}

func randRange(min int, max int) int {
	rand.Seed(time.Now().UnixNano())
	return rand.Intn(max-min+1) + min
}

```

2. 为 HTTPServer 项目添加延时 Metric；

```golang

func newServerHandler() *http.ServeMux {
	interceptor := func(hf http.HandlerFunc) http.HandlerFunc {
		return middleware.SetVersion(middleware.GetIP(middleware.GetResponseStatus(hf)))
	}
	mux := http.NewServeMux()
	mux.HandleFunc("/", interceptor(handler.Index))
	mux.HandleFunc("/healthz", interceptor(handler.Healthz))
	mux.HandleFunc("/hello", handler.DelayedHello)
    // 添加时延metric
	mux.Handle("/metrics", promhttp.Handler())
	return mux
}


```

3. 将 HTTPServer 部署至测试集群，并完成 Prometheus 配置；

此部分主要修改deployment文件，在pod的定义里添加prometheus相关注解，完整文件[链接](https://github.com/Giuliao/cn-practices/blob/main/module2/.build/build-stage.yml)

```yaml
metadata:
    annotation:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"

```

4. 从 Prometheus 界面中查询延时指标数据；（暂未完成）
5. （可选）创建一个 Grafana Dashboard 展现延时分配情况。（暂未完成）
