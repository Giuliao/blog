---
title: "模块一：Go 语言特性"
date: 2022-09-17T22:58:56+08:00
tags: ["geektime-cn"]
---

## 作业

### 课后练习 1.1

编写一个小程序：
给定一个字符串数组
[“I”,“am”,“stupid”,“and”,“weak”]
用 for 循环遍历该数组并修改为
[“I”,“am”,“smart”,“and”,“strong”]

```go
func main() {
	initArr := []string{"I", "am", "stupid", "and", "weak"}

	for i, v := range initArr {
		if v == "stupid" {
			initArr[i] = "smart"
		}

		if v == "weak" {
			initArr[i] = "strong"
		}
	}

	fmt.Println(strings.Join(initArr, " "))
}

```

### 课后练习 1.2

基于 Channel 编写一个简单的单线程生产者消费者模型：

- 队列：队列长度 10，队列元素类型为 int
- 生产者：每 1 秒往队列中放入一个类型为 int 的元素，队列满时生产者可以阻塞
- 消费者：每一秒从队列中获取一个元素并打印，队列为空时消费者阻塞

```go
func main() {
	que := make(chan int, 10)
	ticker := time.NewTicker(time.Second)
	ctx, cancel := context.WithCancel(context.Background())

	// producer
	go func() {
		for {
			select {
			case <-ticker.C:
				que <- 1
			case <-ctx.Done():
				fmt.Println("producer exit")
				return
			}
		}

	}()

	// consumer
	go func() {
		for {
			select {
			case v := <-que:
				fmt.Printf("consume data %d\n", v)
			case <-ctx.Done():
				fmt.Println("consumer exit")
				return
			}
		}
	}()

	time.Sleep(time.Second * 10)
	cancel()
	time.Sleep(time.Second * 2)

}
```
