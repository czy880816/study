# 一、关于消息

mandatory和immediate，这两个参数都有当消息传递过程冲不可达目的地时将消息返回给生产者的功能。RabbitMQ提供的备份交换器可以将未能被交换器路由 的消息存储起来，而不用返回给客户端

## 1、mandatory

当mandatory参数设为true时，如果交换器无法根据自身的类型和路由键找到一个符合条件的队列，那么RabbitMQ会将消息返回给生产者。当mandatory设置为false时，出现上述情况，则将消息直接丢弃

## 2、immediate

当immediate参数设置为true，时，如果交换器在将消息路由到队列时，发现队列上并不存在任何消费者，那么这条消息将不会存入队列中。当与路邮件匹配的所有队列都没有消费者时，该消息会返回给生产者。

**注:该参数在最新版本的RabbitMQ已经去掉，其增加了代码的复杂度和影响了镜像队列的性能，可以使用TTL或者DLX的方法替代**

## 3、备份交换器

备份交换器可以实现在、将违背路由的消息存储在RabbitMQ中，再在需要的时候去处理这些消息

对于备份交换器，有以下几种特殊情况：

- 如果设置的备份交换器不存在，客户端和RabbitMQ服务端都不会有异常出现，此时消息会丢失
- 如果备份交换器没有绑定任何队列，客户端和RabbitMQ服务端都不会有异常信息，此时消息会丢失
- 如果备份交换器没有任何匹配的队列，客户端和RabbitMQ服务端都不会有异常出现，此时消息会丢失
- 如果备份交换器和mandatory参数一起使用，那么mandatory参数无效

## 4、过期时间

TTL，即过期时间，RabbitMQ可以对消息和队列设置TTL

目前有两种方式设置TTL：

- 通过队列属性设置
- 对消息本身进行单独设置，每条消息的TTL可以不同

如果两种方式一起使用，则消息的TTL以两者之间较小的那个数值为准。

## 5、死信队列

DLX，为死信交换器。当消息在一个队列中变成死信后，它能被重新被发送到另一个交换器中，这个交换器就是DLX，绑定DLX的队列就是死信队列

消息变成死信队列一般由于以下几种情况：

- 消息被拒绝，并且设置requeue参数为false
- 消息过期
- 队列达到最大长度

## 6、延迟队列

延迟队列存储的对象是对应的延迟消息，所谓延迟消息是指消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费

使用场景：

1、订单系统下单后的超时失效处理

2、定时执行指定任务

## 7、优先级队列

优先级队列，顾名思义，具有高优先级的队列具有高的优先权，优先级高的消息具备优先被消费的特权，优先级队列可以通过设置`x-max-priority`参数来实现

优先级队列中消息的优先发送是有前提的：如果在消费者的消费速度大于生产者的生产速度且broker中没有消息堆积的情况下，对发送的消息设置优先级也就没有什么实际意义

## 8、持久化

持久化可以提高RabbitMQ的可靠性，以防在异常情况下的数据丢失。RabbitMQ的持久化主要分三部分：交换器的持久化、队列的持久化和消息的持久化

交换器的持久化是通过在声明交换器时将durable参数置为true实现的。如果队列不设置持久化，那么在RabbitMQ服务重启之后，相关队列的元数据会丢失，不过消息不会丢失，只是不能将消息发送到这个交换器中了。对一个长期使用的交换器来说，建议将其置为持久化的

队列的持久化是通过声明队列时将durable参数置为true实现的。如果队列不设置持久化，那么在RabbitMQ服务重启之后，相关队列的元数据会丢失，此时数据也会丢失。队列的持久化能保证其本身的元数据不会因一场情况而丢失。要确保消息不会丢失，需要将其设置为持久化

设置了队列和消息的持久化，当RabbitMQ服务重启之后，消息依旧存在。单单只设置队列持久化，重启之后消息会丢失；单单只设置消息的持久化，重启之后队列消息，继而消息也丢失。

 **注意：**可以将所有的消息都设置为持久化，但是这样会严重影响RabbitMQ的性能。（随机）写入磁盘的速度比写入内存的速度慢的不止一点点。对于可靠性要求不是那么高的消息可以不采用持久化处理以提高整体的吞吐量。

## 9、消息分发

当RabbitMQ队列用友多个消费者时，队列收到的消息将以轮询的分发方式发送给消费者。每条消息只会发送给订阅列表里的一个消费者。但是这样会引起性能问题，因此，针对这种情况，可以使用channel.basicQos方法，来限制信道上的消费者所能保持的最大未确认消息的数量。也就是说，调用这个方法后，RabbitMQ会保存一个消费者列表，每发送一条消息都会为对应的消费者计数，如果达到了所设定的上限，那么RabbitMQ就不会向这个消费者再发送任何消息。直到消费者确认了某条消息之后，RabbitMQ将相应的计数减一，之后消费者可以继续接收消息，直到再次到达计数上限。

**注：**Basic.Qos的使用对于拉模式的消费方式无效

        当prefetchCount设置为0则表示没有上限

## 10、消息顺序性

消息的顺序性是指消费者消费到的消息和发送者发布的消息的顺序是一致的。

RabbitMQ不能保证消息的顺序性，如果要保证消息的顺序性，需要业务方使用RabbitMQ时候做进一步的处理，比如在消息体内添加全局有序标识来实现
