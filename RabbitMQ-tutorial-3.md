# Publish/Subscribe

> using php-amqplib

在上一节中我们创建了一个工作队列。在工作队列背后的假设是每个任务都交付给一个工作进程处理。在这一节中，我们会做一些完全不同的事情--我们会向多个消费者传递相同的信息。这种模式被称为“发布/订阅”。

为了说明这种模式，我们将搭建一个简单的日志记录系统。它将包括两段程序--第一个将产生日志消息，第二个将接收并且打印它们。

在我们的日志记录系统中，接收者程序的每个运行副本都会收到消息。这样我们就可以运行一个接收器并将日志指向磁盘，同时我们可以运行另一个接收器并将日志打印在屏幕上。

基本上，已发布的日志消息将被广播所有接收者。

## Exchanges

在上一节中，我们通过消息队列发送和接收消息。现在是时候在Rabbit中引入完整的消息传递模式了。

然我们快速回顾一下前面教程中介绍的内容：

+ 生产者是发送消息的用户应用程序
+ 队列是存储消息的缓冲区
+ 消费者是接收消息的用户应用程序

RabbitMQ中消息传递模型的核心思想是，生产者从不将任何消息直接发送到队列。实际上，生产者甚至通常不知道是否将消息传递到了队列中。

相反，生产者自能将信息发送到信息交换中间件(exchange)。信息交换中间件只做一件非常简单的事：一方面，它接收来自生产者的消息; 另一方面将接收到的消息推送到队列。信息交换中间件准确知道接收到的消息如何处理。应该添加到特定的队列？ 应该添加到许多的队列中？或者应该丢弃。其规则由交换类型定义。

<center>![exchanges](http://www.rabbitmq.com/img/tutorials/exchanges.png)</center> 

有几种交换类型可用：direct, topic, headers, fanout. 我们将重点关注最后一个 - fanout. 让我们创建一个种类型的交换，并且将其称为 ```logs```：

```php
$channel->exchange_declare('logs', 'fanout', false, false, false);
```

fanout 交换非常简单。正如你可以从名字猜测的一样，它只是将所有的消息广播到所有已知的队列。这正是我们需要的日志记录器。

## 列出所有 exchanges

你可以使用非常有用的 ```rabbitmqctl```来获取服务器上的 exchanges:

 ```php
 sudo rabbitmqctl list_exchanges
 ```
 
 这个交换列表中会有一些 ```amq.*```的交换和默认(未命名)交换。这些事默认创建的，但是不太可能需要使用它们。
 
## 默认交换

在本教程的前面部分，我们对交换没有任何了解，但是任然能够将消息发送到队列。这是可能的，因为我们使用的是默认交换，我们通过空字符串""标识。

回顾一下我们之前发布的消息：

```php
$channel->basic_publish($msg, '', 'hello');
```

在这里，我们使用默认或者无名的交换：消息路由到具有 route_key 指定的名称的队列(如果存在)。路由秘钥是 basic_public 的第三个参数。

现在，我们可以将消息发布到我们命名的交换机：

```php
$channel->exchange_declare('logs', 'fanout', false, false, false);
$channel->basic_publish($msg, 'logs');
```

## 临时队列

你还记得我们之前使用的是具有指定名称的队列(```hello```, ```task_queue```)。能够给队列命名对我们而言是非常重要的，因为我们需要将消费者指向同一个队列。当你想要在生产者和消费者之间共享队列时，给队列一个名字是很重要的。

但是对我们现在的日志记录器而言却不是这样。我们希望收到所有的日志消息，而不仅仅是它的一部分。我们也只对当前的日志消息感兴趣，而对之前的老的日志消息不感兴趣。要解决这些问题，我们需要哦解决两件事情。

首先，每当我们连接到RabbitMQ，我们都需要一个新的，空的队列。为止，我们可以创建一个具有随机名称的队列，或者更好--让我们的服务器生成一个随机名称队列。

其次，一旦我们断开消费者，队列应该被自动删除。

在 ```php-amqplib``` 客户端。当我们提供的队列名称为空字符串时，我们会创建一个随机名称的非持久化的队列：

```php
list($queue_name, ,) = $channel->queue_declare("");
```

当该方法返回时，$queue_name 变量包含由RabbitMQ生成的随机队列名称。例如，它可能看起来像这样 ``` amq.gen-JzTY20BRgKO-HjmUJj0wLg ```

当声明它的连接关闭时，队列将自动删除，因为它被声明为独占类型。

## 绑定

<center>![bindings](http://www.rabbitmq.com/img/tutorials/bindings.png)</center>

我们已经创建了一个fanout交换和一个队列。现在我们需要告诉交换机发送消息到我们的队列。交换机和队列之间的关系称为绑定。

```php
$channel->queue_bind($queue_name, 'logs');
```

从现在开始，日志交换器将发送消息到我们的队列。

## 列出绑定项

你可以列出你想要的现有的绑定。

```php
rabbitmqctl list_bindings
```

## 总结

<center>![python-three-overall](http://www.rabbitmq.com/img/tutorials/python-three-overall.png)</center>

发出日志消息的生产者程序与上一节教程中的生产者并没有太大的区别。最重要的变化是，我们现在想把消息发布到我们的```logs```交换机, 而不是不指定交换机的名称。这里是emir_log.php的代码：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('logs', 'fanout', false, false, false);

$data = implode(' ', array_slice($argv, 1));
if(empty($data)) $data = "info: Hello World!";
$msg = new AMQPMessage($data);

$channel->basic_publish($msg, 'logs');

echo " [x] Sent ", $data, "\n";

$channel->close();
$connection->close();

?>
```

如你所见，建立连接后，我们声明了交换机。此步骤是必须的，因为RabbitMQ禁止发布消息到不存在的交换机。

如果没有任何队列绑定到交换机，消息将丢失，但是这对我们来说是没关系的；如果没有消费者正在监听，我们可以放心的放弃消息。

receive.logs.php:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('logs', 'fanout', false, false, false);

list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$channel->queue_bind($queue_name, 'logs');

echo ' [*] Waiting for logs. To exit press CTRL+C', "\n";

$callback = function($msg){
  echo ' [x] ', $msg->body, "\n";
};

$channel->basic_consume($queue_name, '', false, true, false, false, $callback);

while(count($channel->callbacks)) {
    $channel->wait();
}

$channel->close();
$connection->close();

?>
```

如果想要将日志保存到文件，只需要打开控制台并且键入：

```php
php receive_logs.php > logs_from_rabbit.log
```

如果你希望在屏幕上看到日志，则再打开一个控制台并运行：

```php
php receive_logs.php
```

当然，需要生产日志：

```php
php emit_log.php
```

使用 rabbitmqctl list_bindings 可以验证实际上根据需要创建的绑定和队列。现在运行了两个receive_log.php 程序，你应该会看到类似下面的的内容：

```php
sudo rabbitmqctl list_bindings
# => Listing bindings ...
# => logs    exchange        amq.gen-JzTY20BRgKO-HjmUJj0wLg  queue           []
# => logs    exchange        amq.gen-vso0PVvyiRIL2WoV3i48Yg  queue           []
# => ...done.
```

对结果的解释很简单：来自交换机日志数据转到具有服务器分配名称的两个队列。这正是我们想要的。

要了解怎样订阅一个子集，请移步下一节：[路由](#)