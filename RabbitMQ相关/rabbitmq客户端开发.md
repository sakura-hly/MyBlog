# 客户端开发向导
## 连接 RabbitMQ
下面的代码用来在给定的参数(ip 地址、端口号、用户名、密码等)下连接RabbitMQ:
```
ConnectionFactory factory = new ConnectionFactory();
factory.setHost(IP_ADDRESS);
factory.setPort(PORT);
factory.setVirtualHost(virtualHost);
factory.setUsername("guest");
factory.setPassword("guest");

Connection connection = factory.newConnection(); // 创建连接
Channel channel = connection.createChannel(); // 创建通道
```
也可以选择使用URI的方式来实现
```
ConnectionFactory factory = new ConnectionFactory();
factory.setUri("amqp://userName:password@ipAddress:portNumber/virtualHost");
Connection conn = factory.newConnection();
```
Connection接口被用来创建一个Channel:
```
Channel channel = conn.createChannel();
```
在创建之后，Channel可以用来发送或者接收消息了。

**注意：** Connection可以用来创建多个Channel实例，但是Channel实例不能在线程问共享，应用程序应该为每一个线程开辟一个Channel。
某些情况下Channel的操作可以并发运行，但是在其他情况下会导致在网络上出现错误的通信帧交错，同时也会影响发送方确认
(publisher confirm)机制的运行，所以多线程问共享Channel实例是非线程安全的。

Channel或者Connection中有个isOpen方法可以用来检测其是否己处于开启状态。
但并不推荐在生产环境的代码上使用isOpen方法，这个方法的返回值依赖于shutdownCause的存在，有可能会产生竞争。
```
public boolean isOpen() {
    synchronized(this.monitor) {
        return this.shutdownCause == null;
    }
}
```
通常情况下，在调用 createXXX 或者 newXXX 方法之后，我们可以简单地认为 Connection 或者 Channel 已经成功地处于开启状态，而并不会在代码中使用 isOpen 这个检测方法。
如果在使用 Channel 的时候其己经处于关闭状态，那么程序会抛出一个com.rabbitmq.client.ShutdownSignalException，我们只需捕获这个异常即可。当然同时也要试着捕获 IOException 或者 SocketException，以防 Connection 意外关闭。


## 使用交换器和队列
交换器和队列是 AMQP 中 high-level 层面的构建模块，应用程序需确保在使用它们的时候就已经存在了，在使用之前需要先声明 ( declare ) 它们。
```
channel.exchangeDeclare(exchangeName, "direct", true);
String queueName = channel.queueDeclare().getQueue();
channel.queueBind(queueName, exchangeName, routingKey);
```
上面创建了一个持久化的、非自动删除的、绑定类型为 direct 的交换器，同时也创建了一个非持久化的、排他的、自动删除的队列(此队列的名称由 RabbitMQ 自动生成)。这里的交换器和队列也都没有设置特殊的参数。

上面声明的队列具备如下特性：只对当前应用同一个Connection层面可用，同一个Connection的不同Channel可共用，并且也会在应用连接断开时自动删除。

如果要在应用中共享一个队列，可以做如下声明
```
channel.exchangeDeclare(EXCHANGE_NAME, "direct", true);
channel.queueDeclare(QUEUE_NAME, true, false, false, null);
channel.queueBind(QUEUE_NAME, EXCHANGE_NAME, ROUTING_KEY);
```
这里的队列被声明为持久化的、非排他的、非自动删除的，而且也被分配另一个确定的己知的名称(由客户端分配而非 RabbitMQ 自动生成)。

生产者和消费者都可以声明一个交换器或者队列。如果尝试声明一个已经存在的交换器或者队列，只要声明的参数完全匹配现存的交换器或者队列，RabbitMQ 就可以什么都不做，并成功返回。如果声明的参数不匹配则会抛出异常。


## exchangeDeclare方法详解
exchangeDeclare有多个重载方法，这些重载方法都是由下面这个方法中缺省的某些参数构成的。
```
/**
 * @param exchange 交换器的名称
 * @param type 交换器的类型，常见的如 fanout、 direct 、 topic
 * @param durable 设置是否持久化。 durable设置为 true 表示持久化，反之是非持久化。持久化可以将交换器存盘，在服务器重启的时候不会丢失相关信息。
 * @param autoDelete 设置是否自动删除。 autoDelete 设置为 true 则表示自动删除。自动删除的前提是至少有一个队列或者交换器与这个交换器绑定， 
 之后所有与这个交换器绑定的队列或者交换器都与此解绑。注意不能错误地把这个参数理解为: "当与此交换器连接的客户端都断开时， RabbitMQ 会自动删除本交换器"。
 * @param internal 设置是否是内置的。如果设置为 true ，则表示是内置的交换器，客户端程序无法直接发送消息到这个交换器中，只能通过交换器路由到交换器这种方式。
 * @param arguments 其他一些结构化参数
 */
Exchange.DeclareOk exchangeDeclare(String exchange,
                                              String type,
                                              boolean durable,
                                              boolean autoDelete,
                                              boolean internal,
                                              Map<String, Object> arguments) throws IOException;
```
这个方法的返回值是Exchange.DeclareOk，用来标识成功创建了一个交换器。

除开一些重载方法之外，与 exchangeDeclare 师出同门的还有几个方法， 比如 exchangeDeclareNoWait 方法，
```
void exchangeDeclareNoWait(String exchange,
                               String type,
                               boolean durable,
                               boolean autoDelete,
                               boolean internal,
                               Map<String, Object> arguments) throws IOException;
```
这个 exchangeDeclareNoWait 比 exchangeDeclare多设置了一个NoWait参数，这个指的是AMQP中Exchange.Declare命令的参数，意思是不需要服务器
返回，注意这个方法的返回值是void，而普通的exchangeDeclare方法的返回值是Exchange.DeclareOk，意思是在客户端声明了一个交换器之后，需要等待服务器的返回
(服务器会返回 Exchange.Declare-Ok 这个 AMQP 命令)。

针对 "exchangeDeclareNoWait 不需要服务器任何返回值"这一点，考虑这样一种情况，在声明完一个交换器之后(实际服务器还并未完成交换器的创建)，
那么此时客户端紧接着使用这个交换器，必然会发生异常。如果没有特殊的缘由和应用场景，并不建议使用这个方法。

这里还有师出同门的另一个方法
```
/**
 * Declare an exchange passively; that is, check if the named exchange exists.
 * @param name check the existence of an exchange named this
 * @throws IOException the server will raise a 404 channel exception if the named exchange does not exist.
 */
Exchange.DeclareOk exchangeDeclarePassive(String name) throws IOException;
```
这个方法在实际应用过程中还是非常有用的，它主要用来检测相应的交换器是否存在。如果存在则正常返回:如果不存在则抛出异常: 404 channel exception，同时 Channel 也会被关闭。

有声明创建交换器的方法，当然也有删除交换器的方法。相应的方法如下
```
Exchange.DeleteOk exchangeDelete(String exchange, boolean ifUnused) throws IOException;
void exchangeDeleteNoWait(String exchange, boolean ifUnused) throws IOException;
Exchange.DeleteOk exchangeDelete(String exchange) throws IOException;
```
其中 exchange 表示交换器的名称，而 ifUnused 用来设置是否在交换器没有被使用的情况下删除。 如果 isUnused 设置为 true，则只有在此交换器没有被使用的情况下才会被删除:如果设置 false，则无论如何这个交换器都要被删除。


## queueDeclare方法详解
queueDeclare只有两个重载方法:
```
Queue.DeclareOk queueDeclare() throws IOException;
/**
 * @param queue 队列的名称
 * @param durable 设置是否持久化。为 true 则设置队列为持久化。持久化的队列会存盘，在服务器重启的时候可以保证不丢失相关信息
 * @param exclusive 设置是否排他。为 true 则设置队列为排他的。如果一个队列被声明为排他队列，该队列仅对首次声明它的连接可见，并在连接断开时自动删除。
 这里需要注意 三点:排他队列是基于连接( Connection) 可见的，同一个连接的不同信道 (Channel)是可以同时访问同一连接创建的排他队列;
 "首次"是指如果一个连接己经声明了 一个排他队列，其他连接是不允许建立同名的排他队列的，这个与普通队列不同;
 即使该队列是持久化的，一旦连接关闭或者客户端退出，该排他队列都会被自动删除，这种队列适用于一个客户端同时发送和读取消息的应用场景。
 * @param autoDelete 设置是否自动删除。为 true 则设置队列为自动删除。自动删除的前提是:至少有一个消费者连接到这个队列，之后所有与这个队列连接的消费者都断开时，才会自动删除。
 不能把这个参数错误地理解为: "当连接到此队列的所有客户端断开时，这个队列自动删除"，因为生产者客户端创建这个队列，或者没有消费者客户端与这个队列连接时，都不会自动删除这个队列。
 * @param arguments 设置队列的其他一些参数，如 x-rnessage-ttl 、x-expires 、x -rnax-length 、x-rnax-length-bytes 、x-dead-letter-exchange 、x-deadletter-routing-key、x-rnax-priority 等
 */
Queue.DeclareOk queueDeclare(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                                 Map<String, Object> arguments) throws IOException;
```
不带任何参数的 queueDeclare 方法默认创建一个由 RabbitMQ 命名的(类似这种amq.gen-LhQzlgv3GhDOv8PIDabOXA 名称，这种队列也称之为匿名队列)、排他的、自动删除的、非持久化的队列。

**注意：** 生产者和消费者都能够使用 queueDeclare 来声明一个队列，但是如果消费者在同一个信道上订阅了另一个队列，就无法再声明队列了。必须先取消订阅，然后将信道直为"传输"模式，之后才能声明队列。

这里也有一个 queueDeclareNoWait 方法:
```
void queueDeclareNoWait(String queue, boolean durable, boolean exclusive, boolean autoDelete,
                            Map<String, Object> arguments) throws IOException;
```
与交换器对应，关于队列也有删除的相应方法
```
Queue.DeleteOk queueDelete(String queue) throws IOException;
Queue.DeleteOk queueDelete(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
void queueDeleteNoWait(String queue, boolean ifUnused, boolean ifEmpty) throws IOException;
```
其中 queue 表示队列的名称，ifUnused 可以参考上一小节的交换器。ifEmpty 设置为 true 表示在队列为空(队列里面没有任何消息堆积)的情况下才能够删除。

与队列相关的还有一个有意思的方法——queuePurge，区别于 queueDelete ，这个方法用来清空队列中的内容，而不删除队列本身
```
Queue.PurgeOk queuePurge(String queue) throws IOException;
```

## queueBind方法详解
将队列和交换器绑定的方法如下
```
Queue.BindOk queueBind(String queue, String exchange, String routingKey) throws IOException;
/**
 * @param queue 队列名称
 * @param exchange 交换器的名称:
 * @param routingKey 用来绑定队列和交换器的路由键;
 * @param arguments 定义绑定的一些参数
 */
Queue.BindOk queueBind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
void queueBindNoWait(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```
不仅可以将队列和交换器绑定起来，也可以将已经被绑定的队列和交换器进行解绑。具体方法可以参考如下
```
Queue.UnbindOk queueUnbind(String queue, String exchange, String routingKey) throws IOException;
Queue.UnbindOk queueUnbind(String queue, String exchange, String routingKey, Map<String, Object> arguments) throws IOException;
```

## queueBind方法详解
我们不仅可以将交换器与队列绑定，也可以将交换器与交换器绑定，后者和前者的用法如出一辙，
```
Exchange.BindOk exchangeBind(String destination, String source, String routingKey) throws IOException;
Exchange.BindOk exchangeBind(String destination, String source, String routingKey, Map<String, Object> arguments) throws IOException;
void exchangeBindNoWait(String destination, String source, String routingKey, Map<String, Object> arguments) throws IOException;
```
绑定之后，消息从 source 交换器转发到 destination 交换器，某种程度上来说 destination 交换器可以看作一个队列。


## 何时创建
RabbitMQ 的消息存储在队列中，交换器的使用并不真正耗费服务器的性能，而队列会。如果要衡量 RabbitMQ 当前的 QPS 只需看队列的即可。
在实际业务应用中，需要对所创建的队列的流量、内存占用及网卡占用有一个清晰的认知，预估其平均值和峰值，以便在固定硬件资源的情况下能够进行合理有效的分配。

按照 RabbitMQ 官方建议，生产者和消费者都应该尝试创建(这里指声明操作)队列。这是一个很好的建议，但不适用于所有的情况。
如果业务本身在架构设计之初己经充分地预估了队列的使用情况，完全可以在业务程序上线之前在服务器上创建好(比如通过页面管理、 RabbitMQ命令或者更好的是从配置中心下发)，这样业务程序也可以免去声明的过程，直接使用即可。

预先创建好资源还有一个好处是，可以确保交换器和队列之间正确地绑定匹配。很多时候，由于人为因素、代码缺陷等，发送消息的交换器并没有绑定任何队列，那么消息将会丢失:或者
交换器绑定了某个队列，但是发送消息时的路由键无法与现存的队列匹配，那么消息也会丢失。当然可以配合 mandatory 参数或者备份交换器来提高程序的健壮性。


## 发送消息
如果要发送一个消息，可以使用 Channel 类的 basicPublish 方法，比如发送一条内容为 "Hello World!"的消息，参考如下:
```
String message = "hello world!";
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN,
        message.getBytes());
```
为了更好地控制发送，可以使用 mandatory 这个参数。

上面这行代码发送了一条消息，这条消息的投递模式 (delivery mode) 设置为 2 ，即消息会被持久化(即存入磁盘)在服务器中。同时这条消息的优先级 (priority)设置为 1 ，content-type
为"text/plain"。 可以自己设定消息的属性:
```
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY,
        new AMQP.BasicProperties.Builder()
                .contentType("text/p1ain")
                .deliveryMode(2)
                .priority(1)
                .userId("hidden")
                .build(), message.getBytes());
```
还可以发送一条带有过期时间 (expiration) 的消息:
```
channel.basicPublish(EXCHANGE_NAME, ROUTING_KEY,
        new AMQP.BasicProperties.Builder()
                .expiration("60000")
                .build(), message.getBytes());
```
对于basicPublish，有几个重载方法
```
void basicPublish(String exchange, String routingKey, BasicProperties props, byte[] body) throws IOException;
void basicPublish(String exchange, String routingKey, boolean mandatory, BasicProperties props, byte[] body)
            throws IOException;
/**
 * @param exchange 交换器的名称，指明消息需要发送到哪个交换器中。 如果设置为空字符串，则消息会被发送到 RabbitMQ 默认的交换器中。
 * @param routingKey 路由键，交换器根据路由键将消息存储到相应的队列之中。
 * @param mandatory true if the 'mandatory' flag is to be set
 * @param immediate true if the 'immediate' flag is to be set. Note that the RabbitMQ server does not support this flag.
 * @param props 消息的基本属性集，其包含 14 个属性成员，分别有 contentType、contentEncoding、 headers(Map<String，Object>)、deliveryMode、priority、
correlationld、replyTo、expiration、messageld、timestamp、type、userld、appld、clusterId。
 * @param body 消息体 (pay1oad)，真正需要发送的消息。
 */
void basicPublish(String exchange, String routingKey, boolean mandatory, boolean immediate, BasicProperties props, byte[] body)
            throws IOException;
```

## 消费消息
RabbitMQ 的消费模式分两种: 推 (Push)模式和拉 (Pull)模式。推模式采用 Basic.Consume 进行消费，而拉模式则是调用 Basic.Get 进行消费。
1. 推模式
   
   在推模式中，可以通过持续订阅的方式来消费消息，使用到的相关类有:com.rabbitmq.client.Consumer 和 com.rabbitmq.client.DefaultConsumer。
   
   接收消息一般通过实现 Consumer 接口或者继承 DefaultConsumer 类来实现。当调用与 Consumer 相关的 API 方法时 ， 不同的订阅采用不同的消费者标签 (consumerTag) 来区分彼此，
   在同一个 Channel 中的消费者也需要通过唯一的消费者标签以作区分
   ```
   channel.basicQos(64); // 设置客户端最多接收未被 ack 的消息的个数
   boolean autoAck = false;
   Consumer consumer = new DefaultConsumer(channel) {
       @Override
       public void handleDelivery(String consumerTag,
                                  Envelope envelope,
                                  AMQP.BasicProperties properties,
                                  byte[] body) throws IOException {
           System.out.println(" recv message: " + new String(body));
           try {
               TimeUnit.SECONDS.sleep(1);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }

           channel.basicAck(envelope.getDeliveryTag(), false);
       }
   };

   channel.basicConsume(QUEUE_NAME, autoAck, "myConsumerTag", consumer);
   ```
   注意，上面代码中显式地设置 autoAck 为 false，然后在接收到消息之后进行显式 ack 操作 (channel.basicAck )，对于消费者来说这个设置是非常必要的，可以防止消息不必要地丢失。
   
   Channel 类中 basicConsume 方法有如下几种形式:
   ```
   String basicConsume(String queue, boolean autoAck, String consumerTag, Consumer callback) throws IOException;
   String basicConsume(String queue, boolean autoAck, String consumerTag, DeliverCallback deliverCallback, CancelCallback cancelCallback) throws IOException;
   String basicConsume(String queue, boolean autoAck, String consumerTag, DeliverCallback deliverCallback, ConsumerShutdownSignalCallback shutdownSignalCallback) throws IOException;
   String basicConsume(String queue, boolean autoAck, String consumerTag, DeliverCallback deliverCallback, CancelCallback cancelCallback, ConsumerShutdownSignalCallback shutdownSignalCallback) throws IOException;
   String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive, Map<String, Object> arguments, Consumer callback) throws IOException;
   String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive, Map<String, Object> arguments, DeliverCallback deliverCallback, CancelCallback cancelCallback) throws IOException;
   String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive, Map<String, Object> arguments, DeliverCallback deliverCallback, ConsumerShutdownSignalCallback shutdownSignalCallback) throws IOException;
    /**
     * @param queue 队列的名称
     * @param autoAck 设置是否自动确认。建议设成 false ，即不自动确认
     * @param consumerTag 消费者标签，用来区分多个消费者
     * @param noLocal 置为 true 则表示不能将同一个 Connection 中生产者发送的消息传送给这个 Connection 中的消费者
     * @param exclusive 设置是否排他
     * @param arguments 设置消费者的其他参数:
     * @param deliverCallback 设置消费者的回调函数。用来处理 RabbitMQ 推送过来的消息，比如DefaultConsumer， 使用时需要客户端重写 (override) 其中的方法。
     * @param cancelCallback callback when the consumer is cancelled
     * @param shutdownSignalCallback callback when the channel/connection is shut down
     */
   String basicConsume(String queue, boolean autoAck, String consumerTag, boolean noLocal, boolean exclusive, Map<String, Object> arguments, DeliverCallback deliverCallback, CancelCallback cancelCallback, ConsumerShutdownSignalCallback shutdownSignalCallback) throws IOException;
   ```
   对于消费者客户端来说重写 handleDelivery 方法是十分方便的。更复杂的消费者客户端会重写更多的方法，具体如下
   ```
   // 会在其他方法之前调用，返回消费者标签。
   void handleConsumeOk(String consumerTag);
   void handleCancelOk(String consumerTag);
   void handleCancel(String consumerTag) throws IOException;
   // 当 Channel 或者 Connection 关闭的时候会调用
   void handleShutdownSignal(String consumerTag, ShutdownSignalException sig);
   void handleRecoverOk(String consumerTag);
   ``` 
   也可以通过 channel.basicCancel 方法来显式地取消一个消费者的订阅:
   ```
   channel.basicCancel(consumerTag);
   ```
   注意上面这行代码会首先触发 handleConsumerOk 方法，之后触发 handleDelivery 方法，最后才触发 handleCancelOk 方法。
   
   和生产者一样，消费者客户端同样需要考虑线程安全的问题。最常用的做法是一个 Channel 对应一个消费者，也就是意味着消费者彼此之间没有任何关联。
   当然也可以在一个 Channel 中维持多个消费者，但是要注意一个问题，如果 Channel 中的一个消费者一直在运行，那么其他消费者的 callback 会被"耽搁"。
   
2. 拉模式

   通过channel.basicGet方法可以单条地获取消息，其返回值是GetResponse。
   ```
   /**
    * @param queue the name of the queue
    * @param autoAck true if the server should consider messages
    * @return a {@link GetResponse} containing the retrieved message data
    * @throws java.io.IOException if an error is encountered
    */
   GetResponse basicGet(String queue, boolean autoAck) throws IOException;
   ```
   拉模式的关键代码如下
   ```
    GetResponse response = channel.basicGet(QUEUE_NAME, false);
    System.out.println(new String(response.getBody()));
    channel.basicAck(response.getEnvelope().getDeliveryTag(), false);
   ```
   拉模式的消费方式如下图（只展示消费的部分）
   ![拉模式](./doc.img/RabbitMQ.consume.pull.png)
   **注意：** Basic.Consume将信道(Channel)置为接收模式，直到取消队列的订阅为止。在接收模式期间，RabbitMQ会不断推送消息给消费者，当然推送消息的个数还是会受到
   Basic.Qos的限制。如果只想从队列获取单条消息而不是持续订阅，建议还是使用Basic.Get。但是不能将Basic.Get放在一个循环里面代替Basic.Consume，这样做会严重影响
   RabbitMQ的性能。如果要实现高吞吐量，消费者理应使用Basic.Consume。
   

## 消费端的确认与拒绝
为了保证消息从队列可靠地达到消费者， RabbitMQ 提供了消息确认机制。消费者在订阅队列时，可以指定autoAck参数，当autoAck等于false时，RabbitMQ会等待消费者显示地回复确认信号才从内存或磁盘中移去消息
(实质上是先打上删除标记，之后再删除)。当autoAck等于true时，RabbitMQ会自动把发送出去的消息置为确认，然后从内存或磁盘中删除，而不管消费者是否真正地消费到了这些消息。

采用消息确认机制后，只要设置 autoAck 参数为 false，消费者就有足够的时间处理消息(任务)，不用担心处理消息过程中消费者进程挂掉后消息丢失的问题，因为 RabbitMQ 会一直
等待持有消息直到消费者显式调用 Basic.Ack 命令为止。

当 autoAck 参数置为 false，对于 RabbitMQ 服务端而言，队列中的消息分成了两个部分: 一部分是等待投递给消费者的消息；一部分是已经投递给消费者，但是还没有收到消费者确认信号的消息。
如果 RabbitMQ 一直没有收到消费者的确认信号，并且消费此消息的消费者已经断开连接，则 RabbitMQ 会安排此消息重新进入队列，等待投递给下一个消费者，当然也有可能是原来的消费者。

RabbitMQ不会为未确认的消息设置过期时间，它判断此消息是否需要重新投递给消费者的唯一依据是消费该消息的消费者连接是否已经断开，这么涉及的原因是 RabbitMQ 允许消费者消费一条消息可以很久。

在消费者接收到消息后，如果想明确拒绝当前的消息而不是确认，那么应该怎么做呢? RabbitMQ 在 2.0.0 版本开始引入了 Basic.Reject 这个命令，消费者客户端可以调用与其对于的 channel.basicReject 方法告诉 RabbitMQ 拒绝这个消息。
```
/**
 * Reject a message. Supply the deliveryTag from the {@link com.rabbitmq.client.AMQP.Basic.GetOk}
 * or {@link com.rabbitmq.client.AMQP.Basic.Deliver} method
 * containing the received message being rejected.
 * @see com.rabbitmq.client.AMQP.Basic.Reject
 * @param deliveryTag the tag from the received {@link com.rabbitmq.client.AMQP.Basic.GetOk} or {@link com.rabbitmq.client.AMQP.Basic.Deliver}
 * @param requeue true if the rejected message should be requeued rather than discarded/dead-lettered
 * @throws java.io.IOException if an error is encountered
 */
void basicReject(long deliveryTag, boolean requeue) throws IOException;
```
Basic.Reject 一次只能拒绝一条消息，如果想要批量拒绝消息，则可以使用 Basic.Nack 这个命令。消费者客户端可以调用 channel.basicNack
```
/**
 * Reject one or several received messages.
 *
 * Supply the <code>deliveryTag</code> from the {@link com.rabbitmq.client.AMQP.Basic.GetOk}
 * or {@link com.rabbitmq.client.AMQP.Basic.GetOk} method containing the message to be rejected.
 * @see com.rabbitmq.client.AMQP.Basic.Nack
 * @param deliveryTag the tag from the received {@link com.rabbitmq.client.AMQP.Basic.GetOk} or {@link com.rabbitmq.client.AMQP.Basic.Deliver}
 * @param multiple true to reject all messages up to and including
 * the supplied delivery tag; false to reject just the supplied
 * delivery tag.
 * @param requeue true if the rejected message(s) should be requeued rather
 * than discarded/dead-lettered
 * @throws java.io.IOException if an error is encountered
 */
void basicNack(long deliveryTag, boolean multiple, boolean requeue)
        throws IOException;
```
multiple 参数设置为 true 则表示拒绝 deliveryTag 编号之前所有未被当前消费者确认的消息。

**注意：** 将 channel.basicReject 或者 channel.basicNack 中的 requeue 设置为 false，可以启用"死信队列"的功能。死信队列可以通过检测被拒绝或者未送达的消息来追踪问题。

对于 requeue，AMQP还有一个命令 Basic.Recover 具备可重入队列的特性。其对应的客户端方法为
1. Basic.RecoverOk basicRecover() throws IOException;
2. Basic.RecoverOk basicRecover(boolean requeue) throws IOException;

这个 channel.basicRecover方法用来请求 RabbitMQ 重新发送还未被确认的消息。如果requeue设置为true，则未被确认的消息会被重新加入到队列中，这样对于同一条消息来说，
可能会被分配给与之前不同的消费者。如果requeue设置为false，那么同一条消息会被分配给与之前相同的消费者。默认情况下，如果不设置 requeue 这个参数，相当于channel.basicRecover(true)，即 requeue 默认为 true。

## 关闭连接
在应用程序使用完之后，需要关闭连接、释放资源
```
channel.close();
connection.close();
```
显式地关闭 Channel 是个好习惯，但这不是必须的，在 Connection 关闭的时候，Channel 也会自动关闭。

AMQP 协议中的 Connection 和 Channel 采用 同样的方式来管理网络失败、内部错误和显式地关闭连接。 Connection 和 Channel 所具备的生命周期如下所述。
* Open: 开启状态，代表当前对象可用
* Closing: 正在关闭状态。当前对象被显示地通知调用关闭方法(shutdown)，这样就产生了一个关闭请求让其内部对象进行相应的操作，并等待这些关闭操作的完成。
* Closed: 已经关闭状态。当前对象已经接收到所有的内部对象已完成关闭动作的通知，并且也关闭了自身。

Connection 和 Channel 最终都是会成为 Closed 的状态，不论是程序正常调用的关闭方法，或者是客户端的异常，再或者是发生了网络异常。

在 Connection 和 Channel 中，与关闭相关的方法有 addShutdownListener(ShutdownListener listener) 和 removeShutdownListener(ShutdownListener listener)。
当 Connection 或者 Channel 的状态转变为 Closed 的时候会调用ShutdownListener。而且如果将一个 ShutdownListener 注册到一个己经处于 Closed 状态的对象(这里特指 Connection 和 Channel 对象)时，会立刻调用 ShutdownListener。

getCloseReason方法可以让你知道对象关闭的原因；isOpen方法检测对象当前是否处于开启状态；close(int closeCode, String closeMessage)方法显式地通知当前对象执行关闭操作。

当触发 ShutdownListener 的时候，就可以获取到 ShutdownSignalException，这个 ShutdownSignalException 包含了关闭的原因，这里原因也可以通过调用前面所提及的 getCloseReason 方法获取。

ShutdownSignalException 提供了多个方法来分析关闭的原因。 isHardError 方法可以知道是 Connection的还是 Channel 的错误; getReason 方法可以获取 cause 相关的信息。
