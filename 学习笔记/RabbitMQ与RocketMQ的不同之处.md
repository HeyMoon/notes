1. RabbitMQ消息的发布是发布到Exchanger,消费者从queue中获取消息。
RocketMQ消息的发布是发布到队列中

2. RabbitMQ消息是在内存中，除非你配置了持久化选项(message,exchanger,queue)才会将消息持久化到磁盘。而RocketMQ的消息和队列都是持久化到磁盘的。
