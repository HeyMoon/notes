在RabbitMQ中，exchange和queue默认都是非持久化的,所以当服务器重启之后，exchange和queue，以及里面的消息都消失了。原因在于每个queue和exchange的durable属性。该属性默认为false,它决定了RabbitMQ是否需要在crash 或 重启之后重新创建queue或exchange。将它设置为true,这样你就不需要在服务器断电后重新创建queue和exchange了。你也许认为将queue和exchange的durable属性设置为true就足够了。但是这还不够。你需要把消息也设置成持久化的。

所以，要想消息持久化，需要设置以下属性：

1. 把消息的delivery mode设置为2.

2. 发送到持久化的exchange

3. 到达持久化的queue

生产者的代码如下所示：

````
import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * Created by Administrator on 2017/1/5 0005.
 */
public class DurableSender {

    private static final String DURABLE_EXCHANGE ="durable_exchange";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();

        AMQP.BasicProperties prop = new AMQP.BasicProperties().builder().deliveryMode(2).build();
        boolean durable = true;

        channel.exchangeDeclare(DURABLE_EXCHANGE,"direct",durable);

        String msg = "";
        channel.basicPublish(DURABLE_EXCHANGE,//exchange
                "",//routingKey
                prop,
                msg.getBytes());

        channel.close();
        connection.close();
    }

}
````

或者你可以用一个更简单的方式设置Message的持久化:`MessageProperties.PERSISTENT_TEXT_PLAIN`,这将使得发送的消息的deliveryMode 为2，priority 为1已经content-type为"text/plain"。代码如下所示：

````
public static void main(String[] args) throws IOException, TimeoutException {
    ConnectionFactory connectionFactory = new ConnectionFactory();
    connectionFactory.setHost("localhost");
    Connection connection = connectionFactory.newConnection();
    Channel channel = connection.createChannel();

    //AMQP.BasicProperties prop = new AMQP.BasicProperties().builder().deliveryMode(2).build();
    boolean durable = true;

    channel.exchangeDeclare(DURABLE_EXCHANGE,"direct",durable);

    String msg = "";
    channel.basicPublish(DURABLE_EXCHANGE,//exchange
            "",//routingKey
            MessageProperties.PERSISTENT_TEXT_PLAIN,
            msg.getBytes());

    channel.close();
    connection.close();
}
````

Consumer的代码如下所示：

````
import com.rabbitmq.client.*;

import java.io.IOException;
import java.util.concurrent.TimeoutException;

/**
 * Created by Administrator on 2017/1/5 0005.
 */
public class DurableConsumer {
    private static final String DURABLE_EXCHANGE = "durable_exchange";

    private static final String DURABLE_QUEUE = "durable_queue";

    public static void main(String[] args) throws IOException, TimeoutException {
        ConnectionFactory connectionFactory = new ConnectionFactory();
        connectionFactory.setHost("localhost");
        Connection connection = connectionFactory.newConnection();
        Channel channel = connection.createChannel();
        boolean durable = true;

        channel.exchangeDeclare(DURABLE_EXCHANGE,"direct",durable);
        channel.queueDeclare(DURABLE_QUEUE,durable,false,false,null);

        channel.queueBind(DURABLE_QUEUE,
                DURABLE_EXCHANGE,
                ""//bindKey
        );

        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag, Envelope envelope,
                                       AMQP.BasicProperties properties, byte[] body) throws IOException {
                String message = new String(body, "UTF-8");
                System.out.println(" [x] Received '" + message + "'");
            }
        };
        channel.basicConsume(DURABLE_QUEUE, true, consumer);
    }
}
````

参考：

[rabbitmq java  client api guide](http://www.rabbitmq.com/api-guide.html)
