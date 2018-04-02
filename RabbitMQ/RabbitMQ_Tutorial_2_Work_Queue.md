### Work Queues
在第一部分，我们写了发送消息到一个queue和从一个queue中接收消息的程序。在这一部分，我们将创建一个Work Queue，它将被用于分发耗时的任务到多个workers。

Work Queues背后的主要想法是避免立即执行一个资源密集的任务并且必须等待它完成。相反的，我们让这个任务稍后被调度。我们封装一个任务为一个消息，将它发送到一个队列。一个后台运行的worker进程将会弹出（pop）这个任务，最后执行这个任务。当你运行很多workers时，任务将会在他们之间共享。

这个概念在一些无法在一个Http请求窗口中处理复杂任务的web应用非常有用。

#### PreParation
在教程的前一部分，我们发送了一个包含"hello world"的消息。现在，我们将发送代表复杂任务的字符串。我们没有一个真正世界中的任务，像重新调整图像的大小或者渲染pdf文件，所以我们可以通过假装我们很忙（通过`Thread.sleep()`）来伪装成一个耗时的任务。我们把字符串中`.`的数量作为任务的复杂性；每一个`.`表示一秒的工作。例如，`Hello...`表示一个耗时3秒的伪装的任务。

我们将在之前的例子上轻微的改动`Send.java`，允许从命令行发送任意的消息。这个程序将schedule任务到work queue,所以我们命名为`NewTask.java`。

````
String message = getMessage(argv);

channel.basicPublish("", "hello", null, message.getBytes());
System.out.println(" [x] Sent '" + message + "'");

private static String getMessage(String[] strings){
    if (strings.length < 1)
        return "Hello World!";
    return joinStrings(strings, " ");
}

private static String joinStrings(String[] strings, String delimiter) {
    int length = strings.length;
    if (length == 0) return "";
    StringBuilder words = new StringBuilder(strings[0]);
    for (int i = 1; i < length; i++) {
        words.append(delimiter).append(strings[i]);
    }
    return words.toString();
}
````

我们旧的`Recv.java`程序也需要一些改变：它需要为消息体中的每一个`.`假装一秒的工作。他将处理接收的消息，执行任务，所以我们叫他`Worker.java`。

````
final Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
    String message = new String(body, "UTF-8");

    System.out.println(" [x] Received '" + message + "'");
    try {
      doWork(message);
    } finally {
      System.out.println(" [x] Done");
    }
  }
};
boolean autoAck = true; // acknowledgment is covered below
channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
````

模拟执行时间的fake task：

````
private static void doWork(String task) throws InterruptedException {
    for (char ch: task.toCharArray()) {
        if (ch == '.') Thread.sleep(1000);
    }
}
````

像教程1中一样编译它（jar包在工作目录）：

````
javac -cp rabbitmq-client.jar NewTask.java Worker.java
````

### Round-robin dispatching
使用任务队列的一个优势是轻松并行工作的能力。如果我们创建了积压的任务，我们只需要增加更多的workers。很容易扩容。

首先，我们同时运行两个worker实例。他们从同一个队列中获取消息。

你需要打开三个终端，两个运行worker程序。这些终端将是我们的消费者们--C1,C2.

````
shell1$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C
````

````
shell2$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
 Worker
  [*] Waiting for messages. To exit press CTRL+C
````

在第三个终端，我们发布新任务。只要你启动了Consumer，你可以发布一些信息：

````
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask First message.
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Second message..
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Third message...
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Fourth message....
shell3$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
NewTask Fifth message.....
````

我们看看，传递给我们的workers的是什么：

````
shell1$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'First message.'
 [x] Received 'Third message...'
 [x] Received 'Fifth message.....'
````

````
shell2$ java -cp .:commons-io-1.2.jar:commons-cli-1.1.jar:rabbitmq-client.jar
Worker
 [*] Waiting for messages. To exit press CTRL+C
 [x] Received 'Second message..'
 [x] Received 'Fourth message....'
````

默认，RabbitMQ将顺序的发送每个消息到下一个Consumer。平均的，每个Consumer将会接收到同样数目的消息。这种分发消息的方式称为`Round-robin`。


### Message Acknowledgment
执行一个任务可能耗费几秒。你可能好奇，如果一个Consumer开始一个长时间的任务，并且在只完成一部分时死了会发生什么。从我们目前的代码来看，一旦RabbitMQ递送一个消息到Consumer，它立即从内存中移除这个消息。在这种情况下，如果你kill一个worker，我们将丢失它正在处理的消息。我们也会丢失所有分发到这个特定worker的没有处理的消息。

但是我们不像丢失任何的任务。如果一个任务die了，我们希望任务分发到其他的worker。

为了确保一个消息永远不会丢失，RabbitMQ支持消息确认。Consumer在接收到，处理完消息后，发送一个Ack(nowledgment)回RabbitMQ,告诉RabbitMQ可以删除消息了。

如果一个Consumer dies(它的channel 被关闭了，connection 被关闭了，或者丢失了TCP 连接)，而没有发送Ack，RabbitMQ将会认为一个消息没有完全处理并且将会re-queue（重新添加到队列）。如果同时，有其他的consumer在线，它将快速重新递送到其他的consumer。这种方式可以保证没有消息丢失，即使worker有时die了。

没有任何的消息超时；当Consumer die，RabbitMQ将重新递送消息。这很有用，即使处理一个消息花费很长的时间。

消息确认默认是打开的。在前一个例子中，我们显式的关闭，通过设置`autoAck = true`。是时候设置这个标示为false,然后在worker 完成任务时发送一个合适的确认。

````
channel.basicQos(1); // accept only one unack-ed message at a time (see below)

final Consumer consumer = new DefaultConsumer(channel) {
  @Override
  public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
    String message = new String(body, "UTF-8");

    System.out.println(" [x] Received '" + message + "'");
    try {
      doWork(message);
    } finally {
      System.out.println(" [x] Done");
      channel.basicAck(envelope.getDeliveryTag(), false);
    }
  }
};
boolean autoAck = false;
channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
````

使用这些代码，我们可以确保即使在worker正在处理消息时，你通过`CTRL + C` kill了它,没有消息会丢失。这个worker die后，很快没有确认的消息会重新递送。

> Forgotten Acknowledgment
>
> 忘记`basicAck`是很普遍的错误。这是很容易犯的一个错误，但是后果是很严重的。当你的客户端退出时，消息将会重新递送，但是RabbitMQ将会吃掉越来越多的内存，因为它无法释放任何没有Ack的消息。

### Message durability
我们已经知道如何保证即使consumer die,任务也不会丢失。但是我们的任务仍然可能丢失，如果RabbitMQ服务器停了。

当RabbitMQ 退出或crash，他将会丢失queue和message,除非你告诉它不要。我们要做两件事来保证消息不会丢失：我们需要将queue和message标记为 `durable`（可持久的）。

首先，我们需要保证RabbitMQ不会丢失queue。为了做到，我们需要声明它为`durable`。

````
boolean durable = true;
channel.queueDeclare("hello", durable, false, false, null);
````

尽管这个命令是正确的，但是在我们之前的步骤中不会生效。因为我们已经定义了一个名为`hello`的queue，它不是`durable`的。RabbitMQ不允许以不同的参数重新定义一个已经存在的队列，试图这样做会抛出异常。所以，我们需要定义一个名称不同的队列，例如`task_queue`。

````
boolean durable = true;
channel.queueDeclare("task_queue", durable, false, false, null);
````

此时，我们可以确保`task_queue`不会丢失即使RabbitMQ重启了。现在我们需要标记我们的消息为`durable`--通过设置`MessageProperties`(实现`BasicProperties`)为`PERSISTENT_TEXT_PLAIN`。

````
import com.rabbitmq.client.MessageProperties;

channel.basicPublish("", "task_queue",
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
````

> Note on Message persistence
>
>Marking messages as persistent doesn't fully guarantee that a message won't be lost. Although it tells RabbitMQ to save the message to disk, there is still a short time window when RabbitMQ has accepted a message and hasn't saved it yet. Also, RabbitMQ doesn't do fsync(2) for every message -- it may be just saved to cache and not really written to the disk. The persistence guarantees aren't strong, but it's more than enough for our simple task queue. If you need a stronger guarantee then you can use publisher confirms.   

### Fair dispatch
你可能注意到分发仍然没有按照我们想要的方式工作。例如，在有两个workers的情况下，当所有奇数的消息是`heavy`（我的理解是，费时的）并且所有的偶数的消息是`light`（相对不费时的）。一个worker 可能一直很忙碌，但是另一个可能几乎没有工作。RabbitMQ并不知道这种情况，仍然平均的分发任务。

这种情况发生因为RabbitMQ只是在消息进入queue时分发一个消息。他没有考虑一个consumer未确认的消息的数量。它只是盲目的分发每第n个消息到第n个consumer。

为了避免这种情况，我们可以使用`basicQos`方法并且设置`prefetchCount = 1`.这告诉RabbitMQ不要同时分发大于一个消息给consumer。或者，用其他的话来说，不要分发一个新的消息给worker直到他已经处理完并且ack了。它将分发消息给下一个不忙的worker。

````
int prefetchCount = 1;
channel.basicQos(prefetchCount);
````

> Note about queue size
>
>If all the workers are busy, your queue can fill up. You will want to keep an eye on that, and maybe add more workers, or have some other strategy.



### Putting it all together
`NewTask.java`

````
import java.io.IOException;
import com.rabbitmq.client.ConnectionFactory;
import com.rabbitmq.client.Connection;
import com.rabbitmq.client.Channel;
import com.rabbitmq.client.MessageProperties;

public class NewTask {

  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv)
                      throws java.io.IOException {

    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    Connection connection = factory.newConnection();
    Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);

    String message = getMessage(argv);

    channel.basicPublish( "", TASK_QUEUE_NAME,
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            message.getBytes());
    System.out.println(" [x] Sent '" + message + "'");

    channel.close();
    connection.close();
  }      
  //...
}
````

接下来是`Worker.java`

````
import com.rabbitmq.client.*;

import java.io.IOException;

public class Worker {
  private static final String TASK_QUEUE_NAME = "task_queue";

  public static void main(String[] argv) throws Exception {
    ConnectionFactory factory = new ConnectionFactory();
    factory.setHost("localhost");
    final Connection connection = factory.newConnection();
    final Channel channel = connection.createChannel();

    channel.queueDeclare(TASK_QUEUE_NAME, true, false, false, null);
    System.out.println(" [*] Waiting for messages. To exit press CTRL+C");

    channel.basicQos(1);

    final Consumer consumer = new DefaultConsumer(channel) {
      @Override
      public void handleDelivery(String consumerTag, Envelope envelope, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body, "UTF-8");

        System.out.println(" [x] Received '" + message + "'");
        try {
          doWork(message);
        } finally {
          System.out.println(" [x] Done");
          channel.basicAck(envelope.getDeliveryTag(), false);
        }
      }
    };
    boolean autoAck = false;
    channel.basicConsume(TASK_QUEUE_NAME, autoAck, consumer);
  }

  private static void doWork(String task) {
    for (char ch : task.toCharArray()) {
      if (ch == '.') {
        try {
          Thread.sleep(1000);
        } catch (InterruptedException _ignored) {
          Thread.currentThread().interrupt();
        }
      }
    }
  }
}
````

使用消息确认和prefetchCount你可以设置一个work queue。这个持久化选项使得当RabbitMQ 重启时survive。

关于`Channel`和`MessageProperties`的更多方法，你可以参考[javadocs online](http://www.rabbitmq.com/releases/rabbitmq-java-client/current-javadoc/)
