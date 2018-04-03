RabbitMQ将顺序的发送每一消息到下一个Consumer。平均的，每一个Consumer将会接收到同样数目的消息。这种分发消息的方式称为 Round-robin。

持久化：

1. 将queue设置为durable
`channel.queueDeclare(QUEUE_NAME,durable,false,false,null);`

2. 将message设置为durable
通过设置MessageProperties为`PERSISTENT_TEXT_PLAIN`

`channel.basicPublish("",QUEUE_NAME,MessageProperties.PERSISTENT_TEXT_PLAIN,message.getBytes());`


RabbitMQ与RocketMQ最大的区别是

RabbitMQ 是一个通用消息传递解决方案，通常用于帮助 Web 服务器快速响应请求,RabbitMQ的消息主要存储于内存中(当配置了persist选项后，也会存在于磁盘)，并且在消息被确认消费后会被删除，RabbitMQ的队列是有限的。

RabbitMQ 支持消息的持久化，即将消息数据持久化到磁盘上，如果消息服务器中途断开，下次开启会将持久化的消息重新发送，消息队列持久化需要保证 exchange（指定 durable=1）、queue（指定 durable=1）和消息（delivery_mode=2）3 个部分都是持久化。出于数据安全考虑，一般消息都会进行持久化。

而RocketMQ是一个持久性消息存储库，消息存储于磁盘上，消息堆积能力很强，RocketMQ是无线队列。

通过一个 Rabbit 队列每秒推送 20K 消息并不难，实际上就算推送更多消息也不难做到，在保证方面没有很高的要求。队列由一个 Erlang 轻量级线程提供支持，该线程与本地操作系统线程池协同调度，这使它成为一个天然瓶颈，因为一个队列绝不可能完成超出它可以获得的 CPU 周期容许范围的工作。


https://mp.weixin.qq.com/s/c-AbgNKKI2WEeMFyk4ZNYQ
https://www.rabbitmq.com/persistence-conf.html
