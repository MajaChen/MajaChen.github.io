# 基于kafka+workerpool+context构建一个流式数据处理框架

## 1.基本的数据处理框架

项目组有一套成熟的数据处理框架，各个模块的业务逻辑均运行于该框架之上。框架的基本设计思路是：把每个模块看成是一个加工数据的handler，对外部输入的每一条数据，开启一个handler进行加工，最后把加工后的数据输出到外部，如果在数据处理的过程中发生错误，则通过日志系统记录这条错误信息连同错误数据。

![image-20210713134137136](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210713134137136.png)

而hadler又可以分解为多个子handler，每个子handler完成一个子任务，各个子hander之间是严格的顺序关系，即只有排在前面的子handler完成任务后，后面的hander才可以开始任务，因为它们之间存在数据依赖关系：

![image-20210713134209706](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210713134209706.png)

我们来看看这个框架具体是怎么实现的？

首先定义一个结构体BaseContext，其属性如下：

```
type BaseContext struct {
	ctx       context.Context
	canceller context.CancelFunc
	baseHandlerList      []ContextHandler
	deferHandlerList     []ContextHandler
	Record      PayloadRecord
	other fields......
}

```

- ctx/canceller:控制业务逻辑的生命周期，她可以设定一个超时时间，在规定的超时时间之前必须完成baseHandlerList中的所有业务逻辑，否则这条数据视为处理失败，终止业务逻辑。也可以基于ctx在业务逻辑执行的过程中跨goroutines传递数据；关于golang context包的使用可以参看[这篇文章]()；
- baseHandleList:封装了各个子业务逻辑，每个ContextHandler实例对应一个子业务逻辑，形成一条完整的handler链；
- deferHandleList：同样封装了各个子业务逻辑，但是这里的子业务逻辑一般指的是这类操作——即无论系统发生什么故障，这些操作一定会执行，比如资源的回收操作，通过golang的defer关键字来实现，因此得名defer handler。defer handler的执行顺序是逆序，即排在最前面的defer handler最后执行；
- Record：数据载体，是一个map[string]interface{}类型变量；
- ContextHandler是一个函数变量，其原型如下

```
type ContextHandler func(ctx c.Context) (needBreak bool, err error)
```

她的本质是一个函数，她接受一个context实例作为入参,该实例一般是ctx或者是基于ctx通过调用context.WithCanecl创建的derived context实例。handler可以根据context的状态来决定何时结束运行，也可基于context实例在goroutines之间传递数据，详细情况可参看[我的相关博客]()。她返回一个bool值——needBreak ，表示这个子逻辑的处理是否发生异常，如果发生异常则需要中断整条子逻辑链，整个处理逻辑结束。

其主要的方法如下：

- AddBaseHandler/AddDeferHandler：把handler加入到basehandlerlist或者deferHandlerList;

- Run:在一个for循环中按baseHandleList规定的顺序运行所有的base handler，并通过defer关键字设置函数在返回的时候执行所有的defer hander。任何handler在执行的过程中返回needbreak则退出函数，处理逻辑结束。如果ctx设定的超时时间到函数也会return，处理逻辑结束。

  ![image-20210713135138167](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210713135138167.png)

各个模块如何调用数据处理框架呢？

## 2.使用框架构建应用

首先各个模块基于自己的业务情况，通过继承BaseContext创建ChildBaseContext，并把各个子业务逻辑封装成ContextHandler实例；然后每从外部收到一条数据，就新创建一个ChildContext的实例，把base handlers通过AddBaseHandler方法注册到ChildContext实例，最后调用ChildContext的Run方法，即可完成一条数据的加工处理。

那么问题是——如何从外部收到数据，如何把数据写入外部，如何并发地处理多条数据，这时就需要结合kafka和workpool等组件构成一个完整的流式数据处理框架。

## 3.结合kafka和workerpool构建一个完整的流式数据处理框架

workpool中的worker与handler绑定，当worker从kafka收到消息之后，worker会回调handler处理消息，我们把这个handler称为ConsumerDispatchHandler，顾名思义，该handler的作用在于封装业务逻辑，并在收到消息之后，把消息分发给workers处理。

ConsumerDispatchHandler也是一个函数变量，原型如下：

```
type WorkerHandler func(interface{})
```

它的本质是一个函数，函数的主要入参是sarama.ProducerMessage实例,即一条待处理消息,在函数内部，初始化ChildBaseContext实例,并基于该实例启动业务逻辑的执行。它没有返回值，因为处理之后的消息会被写回kafka，所以base handlers链条的最后一个handler必须是写回kafka。

基于这样一个流式处理框架的，我们可以轻松构建一个完整的、基于微服务架构的流式应用处理程序。

![image-20210713133800285](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210713133800285.png)

框架的完整代码可以见[github]()

完结！