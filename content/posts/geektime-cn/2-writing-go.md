---
title: "模块二：编写Go程序"
date: 2022-09-24T22:58:56+08:00
tags: ["geektime-cn"]
---

## 笔记

### 1. 线程加锁

- sync.Mutex：Lock()加锁，Unlock()解锁
- sync.RWMutex：不限制读，只限制并发写和并发读写
- sync.WaitGroup：等待一组 goroutine 返回
- sync.Once：保证某段代码只执行一次
- sync.Cond：让一组 goroutine 在满足特定条件时被唤醒

### 2. 线程调度

- 进程和线程间共享 Memory Manage、fs、files、signal
- 调用线程不需要切换上下文，但仍然涉及系统调用
- 用户态线程解决了系统调用问题，Go 语言基于 GMP 实现用户态线程

### 3. Go 语言内存管理

### 4. 包引用与依赖管理，Makefile 项目编译

- go mod replace 用法，使用原始 import 路径，替换源为私有地址
- makefile

  ```makefile
    root:
    export ROOT=github.com/cncamp/golang;
    .PHONY: root

    release:
    echo "building httpserver binary"
    mkdir -p bin/amd64
    CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/amd64 .
    .PHONY: release
  ```

### 5. 动手编写一个 HTTP Server

- Go httpserver 底层实现（[参考](https://cncamp.notion.site/http-server-socket-detail-e1f350d63c7c4d9f86ce140949bd90c2)）

### 6. Go 语言调试

- go 专属调试工具 dlv

- glog 日志工具

- 性能分析工具
  - pprof 中 top 各个字段的含义（[参考](https://zhuanlan.zhihu.com/p/371713134)）：
    - flat：函数在 CPU 上运行的时间
    - flat%：函数在CPU上运行时间的百分比
    - sum%：是从上到当前行所有函数累加使用 CPU 的比例
    - cum：这个函数以及子函数运行所占用的时间，应该大于等于flat
    - cum%：这个函数以及子函数运行所占用的比例，应该大于等于flat%
- linux top 指标中的含义（[参考](https://www.jianshu.com/p/af584c5a79f2))）

### 7. kubernetes 中如何使用 Go 语言

## 课后练习 2.1

- 将练习 1.2 中的生产者消费者模型修改为多个生产者和多个消费者模式
  ```go
    	que := make(chan int, 10)
      ticker := time.NewTicker(time.Second)
      ctx, cancel := context.WithCancel(context.Background())
      wg := sync.WaitGroup{}
      producerNum := 10
      customerNum := 10
      wg.Add(producerNum + customerNum)
      // producer
      for i := 0; i < producerNum; i++ {
        go func(n int) {
          for {
            select {
            case <-ticker.C:
              que <- n
            case <-ctx.Done():
              wg.Done()
              fmt.Printf("producer %d exit\n", n)
              return
            }
          }

        }(i)
      }

      // consumer
      for i := 0; i < customerNum; i++ {
        go func(n int) {
          for {
            select {
            case v := <-que:
              fmt.Printf("consume data %d\n", v)
            case <-ctx.Done():
              wg.Done()
              fmt.Printf("consumer %d exit\n", n)
              return
            }
          }
        }(i)

	}

	time.Sleep(20 * time.Second)
	cancel()
	wg.Wait()
  ```

## 课后练习 2.2

1. 接收客户端 request，并将 request 中带的 header 写入 response header
2. 读取当前系统的环境变量中的 VERSION 配置，并写入 response header
3. Server 端记录访问日志包括客户端 IP，HTTP 返回码，输出到 server 端的标准输出
4. 当访问 localhost/healthz 时，应返回 200

```go
// 参考：https://gist.github.com/Boerworz/b683e46ae0761056a636
type loggingResponseWriter struct {
	http.ResponseWriter
	statusCode int
}

func NewLoggingResponseWriter(w http.ResponseWriter) *loggingResponseWriter {
	// WriteHeader(int) is not called if our response implicitly returns 200 OK, so
	// we default to that status code.
	return &loggingResponseWriter{w, http.StatusOK}
}

func (lrw *loggingResponseWriter) WriteHeader(code int) {
	lrw.statusCode = code
	lrw.ResponseWriter.WriteHeader(code)
}

// getIP returns the ip address from the http request
// 参考：https://gist.github.com/miguelmota/7b765edff00dc676215d6174f3f30216
func getIP(r *http.Request) (string, error) {
	ips := r.Header.Get("X-Forwarded-For")
	splitIps := strings.Split(ips, ",")

	if len(splitIps) > 0 {
		// get last IP in list since ELB prepends other user defined IPs, meaning the last one is the actual client IP.
		netIP := net.ParseIP(splitIps[len(splitIps)-1])
		if netIP != nil {
			return netIP.String(), nil
		}
	}

	ip, _, err := net.SplitHostPort(r.RemoteAddr)
	if err != nil {
		return "", err
	}

	netIP := net.ParseIP(ip)
	if netIP != nil {
		ip := netIP.String()
		if ip == "::1" {
			return "127.0.0.1", nil
		}
		return ip, nil
	}

	return "", errors.New("IP not found")
}

// 每个请求都要记录日志需要使用拦截器
func interceptor(handler http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// 2. 读取当前系统的环境变量中的 VERSION 配置，并写入 response header
		w.Header().Add("version", os.Getenv("VERSION"))

		lrw := NewLoggingResponseWriter(w)

		handler(lrw, r)

		// 3.Server 端记录访问日志包括客户端 IP，HTTP 返回码，输出到 server 端的标准输出
		addr, err := getIP(r)
		if err != nil {
			glog.Errorf("get addr error %v\n", err)
		}
		glog.Infof("response status code %d, ip addr %s\n", lrw.statusCode, addr)

	}
}

func main() {
	flag.Parse()
	http.HandleFunc("/", interceptor(func(w http.ResponseWriter, r *http.Request) {
		for k, v := range r.Header {
			// 1. 接收客户端 request，并将 request 中带的 header 写入 response header
			w.Header().Add(k, strings.Join(v, ", "))
		}

	}))

	http.HandleFunc("/healthz", interceptor(func(w http.ResponseWriter, r *http.Request) {
		// 4. 当访问 localhost/healthz 时，应返回 200
		w.WriteHeader(http.StatusBadRequest)
	}))
	http.ListenAndServe(":3000", nil)
}
```
