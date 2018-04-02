本文来自于：(http://www.rabbitmq.com/blog/2015/04/16/scheduling-messages-with-rabbitmq/)翻译如下：

### Scheduling Messages with RabbitMQ
之前，人们一直在寻找用RabbitMQ实现延迟消息(delayed messaging)。目前为止，接受的解决方案是使用[message TTL](http://www.rabbitmq.com/ttl.html#per-message-ttl)和[Dead Letter Exchangers](http://www.rabbitmq.com/dlx.html)的混合，正如 James Carr在[这里](http://blog.james-carr.org/2012/03/30/rabbitmq-sending-a-message-to-be-consumed-later/)建议的。一段时间以来，我们一直想提哦概念一个out-of-box的解决方案，这几个月，我们有时间将它实现为一个插件。Enter [RabbitMQ Delayed Message Plugin](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/)。

#### Installing the Plugin
为了安装这个插件，去我们的[社区插件页](http://www.rabbitmq.com/community-plugins.html),并为你的RabbitMQ 安装包下载对应的`.ez`文件。将插件复制到RabbitMQ的插件文件夹，然后在命令行启用：

````
rabbitmq-plugins enable rabbitmq_delayed_message_exchange
````

一旦插件启用了，我们就可以开始使用他了。

### Using the Exchange
为了使用Delayed Message Exchange,你只需声明一个exchange,提供`x-delayed-message`exchange类型，如下：

````
// ... elided code ...
Map<String, Object> args = new HashMap<String, Object>();
args.put("x-delayed-type", "direct");
channel.exchangeDeclare("my-exchange", "x-delayed-message", true, false, args);
// ... more code ...
````

稍后，我们将会解释这个特殊参数`x-delayed-type`的含义。

### Delaying messages
为了延迟一个消息，用户必须在发布消息的头部增加`x-delay`和一个整数值，这个整数值代表应该被RabbitMQ延迟的毫秒数。这里要注意延迟的含义：延迟消息路由到queue或到其他的exchange。

exchange对于consumer没有任何概念。所以一旦延迟过期，插件将会尝试路由消息到符合exchanged的routing rule的queue。注意，如果消息不能路由到任何queue，那么他将会被丢弃，正如AMQP对于不能路由的消息所指定的。

下面是一些示例代码，增加`x-delay`头部，发布到exchange：

````
// ... elided code ...
byte[] messageBodyBytes = "delayed payload".getBytes();
AMQP.BasicProperties.Builder props = new AMQP.BasicProperties.Builder();
headers = new HashMap<String, Object>();
headers.put("x-delay", 5000);
props.headers(headers);
channel.basicPublish("my-exchange", "", props.build(), messageBodyBytes);
````

在前面的例子中，message在被插件路由前将会延迟5秒。

### Flexible Routing
当我们声明了上面的exchange，我们提供一个`x-delayed-type`参数，设置为`direct`。它所做的是告诉exchange在routing messages,creating bindings时我们想要哪种行为。在这个例子中，我们的exchange将会像direct exchange一样，但是我们可以传递`topic`,`fanout`,或者一个其他插件提供的自定义的exchange 类型。通过这样，我们没有限制用户delayed message Plugin提供的routing 行为。

### Checking if a Message was Delayed
一旦我们在consumer端接收到一个消息，我们如何判断这个消息是否是延迟的。插件将会保存`x-delay`头部，但是使值变为负。所以，如果你发布了一个消息，5000毫秒的延迟，consumer接收到时会发现`x-delay`头变为`-5000`。

### We need feedback

We have released the plugin as experimental to gather feedback from the community. Please use it and report back to us on the plugin's [issue page](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/issues) or on our [official mailing list](https://groups.google.com/forum/#!forum/rabbitmq-users).
