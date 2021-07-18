# kafka应用于实际项目

公司项目采用微服务架构，微服务之间采用kafka作为消息交换的中间件，正好以此为契机探索一下kafka如何应用于实际项目。项目采用golang作为开发语言，相应地应该选择kafka的golang客户端，这里的客户端(client)主要指kafka的producer和consumer,brokers则部署于一个单独的集群。我们选择[sarama kafka](https://pkg.go.dev/github.com/Shopify/sarama)这个开源项目作为kafka的golang客户端。

## sarama kafka api

该项目完整的api可以访问[这里](https://pkg.go.dev/github.com/Shopify/sarama#section-documentation),此处仅介绍几个基本的api。

ProducerMessage:对producer侧业务数据的封装，其基本上属性如下

- topic：消息对应的topic
- key:消息的key，用于分区；
- val:封装业务数据；
- partition/offset:消息写入的partition和offset，只有消息成功写入到broker才会被定义；

AsyncProducer:向brokers发布数据。其重要的配置项有：1.向brokers发布数据是选择同步还是异步方式，同步方式下，只有收到brokers的响应才认为数据已经正确发布，异步方式下，数据发送之后收到TCP ACK响应之后即认为数据已正确发布。2.分区方式，默认根据消息的key取hash将消息分布到不同的partitions。一个固化的思维是：创建AsyncProducer实例的时候要提供topic，一个AsyncProducer实例仅为对应的topic提供发送服务，但实际上，AsyncProducer并不与topic强相关，topic包含于ProducerMessage,AsyncProducer实例根据消息的topic字段将其发送到对应的topic。

她提供一个input channel——用于接受消息的输入，和两个output channel——用于返回发送失败时的错误信息和发送成功的消息，示意图如下：

![image-20210710145531341](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710145531341.png)

ConsumerMessage:consumer侧的消息格式，包含的基本字段如下：

- topic/partition/offset:消息的基本信息
- key/val:业务数据

Consumer/PartitionConsumer:从brokers订阅数据。Consumer实例创建并管理多个PartitionConsumer实例，每个PartitionConsumer从指定的offset处消费指定topic下的指定partition。对应关系如下：

![image-20210710145618991](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710145618991.png)

类似于producer,PartitionConsumer提供两个outpu channel——返回从broker处成功消费到的message和消费过程中发生的错误信息

![image-20210710145710155](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710145710155.png)

拓展1：producer基于什么认为数据正确发布或者认为数据没有正确发布，她对此的反应是什么？——我们假定采用同步发送方式，即producer发送数据之后，在规定时间(时间长短可配置)内收到broker的确认之后才认为数据发送成功，通过success channel返回发送成功消息，否则她会重发，重发次数也可配置。如果超过重发次数还没有发送成功，则认定发送失败，通过err channel返回错误信息。

拓展2：client(producer/consumer)如何知道broker的topic、partition以及partition的offset等信息？——这些信息被称为metadata，所有client存储了一份metadata，且定期地与broker同步metadata信息。

## 数据发布模块

在init函数中初始化一个全局的producer对象，各个业务线程完成业务逻辑之后，将处理好的数据封装成sarama.ProducerMessage,由producer发送给kafka

![image-20210710145743166](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710145743166.png)

写入kafka的topic由业务逻辑负责生成，发送方式采用同步方式，即消息发送出去须收到brokers的确认方才认为此消息发送成功，考虑到数据量较大的情况下，success channel会返回大量发送成功的消息，增大系统不必要的负担，所以关闭success channel，而错误信息就地写入日志系统。

## 数据订阅模块

订阅数据相比于发布数据，逻辑会复杂很多，因为从broker消费到消息之后，由具体的业务逻辑处理消息，如果业务逻辑中有耗时的操作，那么接收数据的线程被阻塞，影响其他消息的接受和处理，极大地降低了系统的吞吐量，所以必须实现消息接收与业务逻辑的解耦，很直观的思路是实现一个workerpool，将收到的消息投入workpool处理然后立即返回，由workerpool异步地处理消息：

![image-20210710145940899](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710145940899.png)



在consumer 与 workerpool之间设有缓冲区，如果消息生成速率过快导致workpool来不及处理，消息会进入缓冲区暂存，缓冲区大小默认16

![image-20210710150008062](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710150008062.png)

系统会定期检查consumer的数据消费情况，如果发现consumer超过一定时间间隔(记为x，默认为1分钟)没有消费新的数据，那么workpool处于闲置状态，池内的workers会占用系统资源，所以处理策略是关闭consumer和workerpool， 让系统静默一段时间，时间长度记为y，y是x的倍数，默认为5分钟，即如果consumer1分钟内没有收到任何消息，consumer和workpool被闲置了一分钟，则后面5分钟consumer和workerpool均处于非工作状态，相当于是一种"惩戒"。5分钟之后重启consumer和workerpool，服务恢复正常。x和y均可配置，推荐y取x的倍数。

## workerpool

workerpool的本质是一组线程(对应go的协程)，每个worker对应一个协程。workerpool的大小固定，即workers的数量固定，数量默认为cpu的核数，可通过环境变量进行配置。workerpool在启动的时候，会初始化所有的worker。worker与workerpool的handler相绑定，handler是一个回调函数，封装了具体的业务逻辑。



workers会持续监听名为Job的channel，如果该channel有消息发送过来，则其中某一个worker会收到该消息，并调用handler进行处理。如果该channel没有消息发送过来，则所有 worker会一直阻塞。

![image-20210710150031641](https://gitee.com/xinyuanchen/image_collection/raw/master/image-20210710150031641.png)

源代码可以参看[github]()。







