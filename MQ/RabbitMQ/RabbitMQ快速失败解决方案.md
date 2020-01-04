# 1. 问题背景

1. RabbitmMQ提供了流控机制，防止消息大量积压
2. 当消息积压，内存或磁盘达到一定的水位线时，会触发流控
3. 当触发流控时，消息发送方会无限等待，很快消耗完jetty工作线程，导致服务大量超时，进一步会传递给其他应用，造成影响面扩大，导致严重事故。

# 2. 目标

1. 当出现流控时，应用发送方可以快速感知到，使发送MQ的请求可以快速失败，其他应用请求不受影响。
2. 当流控恢复时，可以立即回复MQ服务

# 3. 解决方案

## 3.1. 实现原理

1. 新版本的RabbitMQ实现了连接流控阻塞通知机制（https://www.rabbitmq.com/connection-blocked.html）
2. 当创建连接时，注册监听器监控连接阻塞事件
3. 当出现流控时，连接接收到阻塞事件时，打印错误日志，并将连接置为阻塞状态
4. 当发送消息时，首先检查连接的阻塞状态，如果是阻塞状态,立即返回异常，如果是正常状态，则正常发送消息
5. 当恢复流控时，连接接收到非阻塞事件，打印日志，并将连接置为非阻塞状态，可以恢复MQ发送服务

## 3.2. 实现方式

1. 注册监听器代码，默认spring定义的连接没有提供BlockedListener监听器的方法，为了拿到rabbitConnection，需要为每个连接建立一个空的Channel

   ```java
   @Override
       public void onCreate(Connection connection) {
           Channel channel = connection.createChannel(true);
           com.rabbitmq.client.Connection rabbitConnection = channel.getConnection();
           BlockedListener blockedListener = new MtRabbitBlockedListener(mtRabbitConnectionFactory);
           rabbitConnection.addBlockedListener(blockedListener);
           LOG.info("Create RabbitMQ Connection " + connection.hashCode());
       }
   ```

   

2. 当出现或恢复流控时，设置或恢复连接的阻塞状态

   ```java
   public class MtRabbitBlockedListener implements BlockedListener {
       private static final Logger LOG = LoggerFactory.getLogger(MtRabbitConnectionListener.class);
       private MtRabbitConnectionFactory mtRabbitConnectionFactory;
       public MtRabbitBlockedListener(MtRabbitConnectionFactory mtRabbitConnectionFactory) {
           this.mtRabbitConnectionFactory = mtRabbitConnectionFactory;
       }
       @Override
       public void handleBlocked(String reason) throws IOException {
           LOG.warn("RabbitMQ connection is blocked for reason : " + reason);
           mtRabbitConnectionFactory.setBlocked(true);
           mtRabbitConnectionFactory.setBlockedReason(reason);
       }
       @Override
       public void handleUnblocked() throws IOException {
           LOG.warn(" rabbitMQ connection is unblocked" );
           mtRabbitConnectionFactory.setBlocked(false);
           mtRabbitConnectionFactory.setBlockedReason("");
       }
   }
   ```

   

3. 当发送消息时，首先检测当前连接的状态，非阻塞时才正常发送

   ```java
   @Transactional
       public void send(String exchange, String routingKey, Object message) {
           if (!StringUtils.hasText(exchange)) {
               throw new IllegalArgumentException("exchange should not be empty");
           }
           if (!StringUtils.hasText(routingKey)) {
               throw new IllegalArgumentException("routingKey should not be empty");
           }
           ConnectionFactory connectionFactory= mtRabbitTemplate.getConnectionFactory();
           if (connectionFactory instanceof MtRabbitConnectionFactory) {
               MtRabbitConnectionFactory mtRabbitConnectionFactory = (MtRabbitConnectionFactory) connectionFactory;
               if (mtRabbitConnectionFactory.getBlocked()) {
                   String errMessage = "RabbitMQ Connection is blocked for reason : "
                           + mtRabbitConnectionFactory.getBlockedReason();
                   throw new AmqpException(errMessage);
               }
           }
           mtRabbitTemplate.convertAndSend(exchange, routingKey, message);
       }
   ```

   