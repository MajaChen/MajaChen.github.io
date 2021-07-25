---
layout: post
title: golang context的使用
subtitle: golang cmd package module
cover-img: /assets/img/romaniya.jpg
thumbnail-img: /assets/img/animals/TawnyFrogmouth.jpg
share-img: /assets/img/romaniya.jpg
tags: [env,basic,golang]

---

# golang context的使用

## 1.背景

&emsp;&emsp;context是golang提供的一个基础库，对于构建服务器程序非常有用。服务器程序需要并发地处理用户的请求，对每个request，服务器程序会生成一个goroutine来处理它，并返回处理的结果。但是对于比较复杂的业务逻辑，每个请求只用一个goroutine去处理可能就不太合适，比如，如果处理逻辑涉及到对数据库的访问，比较耗时，会阻塞goroutine(我们姑且称其为父goroutine),那么可以创建一个子goroutine用于处理数据库的访问，父goroutine继续去实现其他的业务逻辑，等到子goroutine执行完毕，将访问数据库的结果同步给父goroutine即可。也即，对每个request，需要一打goroutine来处理它，此时context就派上用场了。示意图如下：

![image-20210712214037603](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210712214037603.png)

&emsp;&emsp;context可以在不同的goroutine之间传递数据，而这些goroutines都带有同一个“标签”（即处理同一个request），属于同一个集合，而且context可以向承载业务逻辑的goroutine发送“信号”，控制goroutine的生命周期，但是这里并不涉及到显示的信号发送和接受，具体实现见后文。比如，如果用户为它的请求设置超时时间，那么当超时时间已到，context可以向goroutine发送“信号”，通知她时间已到，应该立即结束业务逻辑，退出运行。<br>
&emsp;&emsp;所以context的两大功能我总结如下：<br>

- 1.跨goroutines传递数据；
- 2.控制goroutines的生命周期；

## 2.context的数据结构及其与业务逻辑的交互

&emsp;&emsp;context的数据结构如下：

```
type Context interface {
    Done() <-chan struct{}
    Err() error
    Deadline() (deadline time.Time, ok bool)
    Value(key interface{}) interface{}
}
```

- Done:返回一个channel，后文这个channel被称为done channel,如果该context“被销毁”，则该channel关闭，对该channel的侦听将立即返回，否则对该channel的侦听将一直被阻塞。

- Err:返回导致context被销毁的错误信息；

- Deadline:返回context最长还有多久被销毁；

- Value:context的数据体，**key必须支持相等操作，value必须保证线程安全；**


&emsp;&emsp;一般把context作为函数的传参嵌套进业务逻辑，在函数执行的过程中，不断侦听done channel，如果侦听到done channel已经被关闭了,那么这就是一个**信号**，这个函数应该立即终止业务逻辑，结束运行。再者，通过Deadline返回的时间，函数可以得知：本函数还有多长的运行时间，函数应该在deadline之前结束运行，函数可以选择继续运行，也可以选择在deadline规定的时间范围内做好收尾工作，比如，及时关闭、归还资源等。

## 3.基于derived context构建多层级业务处理逻辑

&emsp;&emsp;可以从一个父contex创建多个子context，子context称为derived context，父context可以控制子context的done channel的开闭，从而决定子context何时销毁，其api如下：

```
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

type CancelFunc func()

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```

&emsp;&emsp;WithCancel:从已有的context中创建一个新的子context，返回cancel方法，当cancel方法被调用的时候，子context的done channel关闭，子context被销毁。父context被销毁，也会直接导致子context被销毁。<br>
WithTimeout:多了一个超时时间做入参，在WithCancel的基础上，对子context增加了如下限制——当设置超时时间一到，子context的done channel会关闭，子context应立即被销毁。

**WithCancel和WithTimeout方法都会把父context的数据拷贝到子context。**基于derived context,我们可以可以构建如下一棵context tree:

![image-20210712214637535](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210712214637535.png)

&emsp;&emsp;其中最顶层的Background,是整个系统中所有其他context的“母亲”，她永远不会被cancel,也永远不会超时。第二层的context是由Background直接创建的，控制的是最顶层的reqeust处理逻辑,当request处理完毕，或者处理request发生超时，这一层级的context会被销毁，连带她创建的子context；如果把一个对request的处理任务进行分解，就得到第三层的context，它们控制的是处理子任务的业务逻辑，这一层级的context被销毁有很多原因，比如：父context被销毁，父context出于业务需要显示地调用了cancel方法等。

## 4.举例——并列争球

&emsp;&emsp;context是一个非常抽象的组件，我们结合并列争球这个例子，来讲解一下如何使用context来控制goroutine的并发执行和利用context在goroutine之间传递数据。并列争球说的是多个球员同时抢球，只要有一名队员最先抢到球则游戏结束，而这里的抢球在实际实现的时候为了简单起见设为“抽奖”——多个goroutine不断生成随机数，看谁最先生成一个幸运数字，之后所有goroutine立即停止运行。代码如下

```
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

type stringType string
const num stringType  = "luckynum"

func main() {

	rand.Seed(time.Now().Unix())

	ctx, canFunc := context.WithCancel(context.Background())
	defer canFunc()
	luckyNum, isHit := 666, make(chan int)
	defer close(isHit)

	ctx = context.WithValue(ctx, num, luckyNum)
	subCanFuncs := make([]context.CancelFunc, 0, 10)
	for i := 0; i < 10; i++ {
		subCtx, subCanFun := context.WithCancel(ctx)
		subCanFuncs = append(subCanFuncs, subCanFun)
		go func(subCtx context.Context,isHit chan int) {
			// 从context中获取幸运数
			luckyNum := subCtx.Value(num).(int)
			for {
				select {
				case <-subCtx.Done():
					fmt.Printf("the ball has been hitten\n")
					return
				default:
					// 生成随机数,如果命中幸运数，向“妈妈”报告
					if ranndom := rand.Intn(1000); ranndom == luckyNum {
						isHit <- 1
					}
				}
			}
		}(subCtx,isHit)
	}


	<-isHit
	fmt.Print("one of my childern has hit the bool\n")
	for _, subCanFunc := range subCanFuncs { 
		subCanFunc()
	}

	// 假设母goroutine后面还有耗时工作
	time.Sleep(time.Second*10)

	fmt.Println("Done!")
}

```

&emsp;&emsp;最后再次体会一下context是如何与我们的业务逻辑结合从而解决问题的

![image-20210712215233641](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210712215233641.png)



&emsp;&emsp;context是一个简洁而功能强大的组件，这种使用模型只是我总结的其中一种，事实上，context可以非常灵活地与我们的业务逻辑相结合，帮助我们解决复杂业务问题。

## 拓展



**拓展1：利用context跨goroutines传递数据**

&emsp;&emsp;这个地方我不是非常清楚，[golang大神的博客]()中只是非常含混地说“A `Context` is safe for simultaneous use by multiple goroutines. ”，我的理解是同一个context被传递到不同的goroutine中，可以被线程安全的使用，包括写入k-v,不需要业务代码提供额外的并发控制机制，但是我发现实际情况不是这样的

```
var lock sync.Mutex

func main() {

	ctx, canFunc := context.WithCancel(context.Background())
	defer canFunc()

	for i := 0; i < 10; i++ {
		go func(i int) {
			lock.Lock()
			ctx = context.WithValue(ctx, i, i)
			fmt.Printf("the addr is %p\n", &ctx)
			fmt.Printf("the key is %v,the val is %v\n", i, ctx.Value(i).(int))
			lock.Unlock()
		}(i)
	}

	time.Sleep(5 * time.Second)
	fmt.Printf("the addr is %p\n", &ctx)
	for i := 0; i < 10; i++ {
		fmt.Printf("the key is %v,the val is %v\n", i, ctx.Value(i).(int))
	}

	fmt.Println("Done!")
}

```

&emsp;&emsp;如果去掉第10行的锁，则程序会出现难以描述之错误，这里我唯一能够得出的结论是：**使用context在goroutine之间传递参数，一定要考虑到并发导致的数据不一致问题、且稳妥的办法还是加锁访问。**<br>
&emsp;&emsp;**拓展2：**这两种引用变量i的方式的区别在于——方式1中新的goroutine对i的修改会同步到外部的i，而方式2是把外部的变量i拷贝了一份，新的goroutine对i的修改不会同步到外部的i。

```
func main() {
	var i int = 0
	go func() {
		//do something with i...
	}()

	go func(i int) {
		// do something with i...
	}(i)
}
```



