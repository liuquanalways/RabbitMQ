# RabbitMQ


## 介绍

RabbitMQ 是一个消息代理：它接收并转发消息。你可以把它当成一个邮局，当你把你想发送的邮件投进邮箱时，你可以确定邮递员最终会把邮件送到你的收件人。在上面的比喻中，我们把 RabbitMQ 比作 邮箱、邮局、邮递员。

RabbitMQ 和 邮局主要的不同是RabbitMQ不处理真实的信件，它接收、存储和转发二进制的数据--消息(message)。

RabbitMQ和消息传递(messaging)中一些常使用术语：

+ **Producing** 仅仅指发送消息(message)。发送消息的程序就称为生产者 **producer**

<center>![producer](http://www.rabbitmq.com/img/tutorials/producer.png)</center>

+ **queue** 就像是RabbitMQ内部的邮箱。虽然消息在RabbitMQ和你的应用程序之间流动，但它们只能存储在队列中。队列的大小只受主机的内存和磁盘的限制，他本质上是一个打的消息缓冲区。多个生产者可以向一个队列发送消息，多个消费者也可以尝试从一个队列接收数据，这就是队列。

<center>![queue](http://www.rabbitmq.com/img/tutorials/queue.png)</center>

+ 消费和接收邮件相似，**consumer** 是一个主要等待接收消息的程序。

<center>![consumer](http://www.rabbitmq.com/img/tutorials/consumer.png)</center>


> 请注意, producer, consumer, broker 并不需要在同一个主机上，尽管它们在大多数应用程序中是存在一个主机上的。

## hello world

在这一章节我们将使用PHP来写两段程序： 一个生产者负责发送一条消息，还有一个消费者负责接收消息并打印出来。在这里简单说一下 php-amqlib API, 主要集中精力来完成上面的例子。

在下面的图中， P 代表生产者，C 代表消费者。中间的盒子代表一个队列 - RabbitMQ提供给消费者的一个消息缓冲区。

<center>![python-one](http://www.rabbitmq.com/img/tutorials/python-one.png)</center>


 在你的项目目录下添加 composer.json (通过composer安装)

```
{
    "require": {
        "php-amqplib/php-amqplib": ">=2.6.1"
    }
}

```

安装

```
composer install
``` 

现在我们已经安装了 php-amqplib 类库， 接下来我们仅需要写少量的代码。

### 生产者

<center>![sending](http://www.rabbitmq.com/img/tutorials/sending.png)</center>

我们将调用我们的消息发布者 **sender.php** 和我们的消息接受者 **receive.php**。消息发布者将会连接 RabbitMQ、发送一条消息、然后退出。

在 **send.php** 中。我们需要引入 php-amqplib 类库并且 使用必要的类：

```php
require_once __DIR__ . '/vendor/autoload.php';

use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;
```

然后我们可以创建一个 RabbitMQ 的连接：

```php
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();
```

上面创建的连接是socket连接的抽象，并且完成了协议版本的协商和认证等等。在这里我们连接到一个本地机器的代理 - 因此使用 localhost 。如果我们想要连接不同机器上的代理，我们可以使用它的域名或者IP地址来替换localhost。

接下来我们新建一个 channel, 这是大部分用于完成任务的API所在的地方。

要发送消息，我们必须先声明一个队列给我们发送消息使用；然后我们就可以将消息发送到队列中：

```php
$channel->queue_declare('hello', false, false, false, false);

$msg = new AMQPMessage('Hello World!');
$channel->basic_publish($msg, '', 'hello');

echo " [x] Sent 'Hello World!'\n";

```

声明一个队列是幂等的 - 只有当它不存在时才会被创建。消息的内容是一个字节数组，所以你可以编码你喜欢的任何东西。

最后，我们关闭channel 和 connection:

```php
$channel->close();
$connection->close();
```

完整的 send.php:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();
$channel->queue_declare('hello', false, false, false, false);
$msg = new AMQPMessage('Hello World!');
$channel->basic_publish($msg, '', 'hello');
echo " [x] Sent 'Hello World!'\n";

$channel->close();
$connection->close();

?>
```

> 不能正常发送消息
> 可能是RabbitMQ 代理没有足够的的硬盘空间(默认至少需要 200MB)

### 消费者

我们的消费者是从 RabbitMQ 中获取消息，因此与发布单个消息的生产者不同，消费者一直运行以监听消息并将它们打印出来。

<center>![receiving](http://www.rabbitmq.com/img/tutorials/receiving.png)</center>

receive.php 代码几乎和 sender.php 有相同的依赖：

```
require_once __DIR__ . '/vendor/autoload.php';

use PhpAmqpLib\Connection\AMQPStreamConnection;
```

设置和生产者相同；我们首先打开一个连接和创建channel，并声明我们要消费的消息队列。注意： 和生产者发送的队列名一致。

```
$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('hello', false, false, false, false);

echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";
```

>请注意，我们在这里也声明了队列。这是因为我们可能在生产者之前启动消费者，所以我们要确保队列一定存在，然后再尝试从中消费信息。

在我们将要告诉服务器将队列中的消息传递给我们之前，我们需要定义一个可以接收服务器发送的消息的回调函数。请注意，消息是从服务器异步发送到客户端。、

```
$callback = function($msg) {
  echo " [x] Received ", $msg->body, "\n";
};

$channel->basic_consume('hello', '', false, true, false, false, $callback);

while(count($channel->callbacks)) {
    $channel->wait();
}

```

当回调时我们的代码将会阻塞接收消息的channel。每当我们接收到一条消息，我们的回调函数将会被传递接收到消息。

receive.php 完整代码：

```
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();
$channel->queue_declare('hello', false, false, false, false);
echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";
$callback = function($msg) {
  echo " [x] Received ", $msg->body, "\n";
};
$channel->basic_consume('hello', '', false, true, false, false, $callback);
while(count($channel->callbacks)) {
    $channel->wait();
}

$channel->close();
$connection->close();

?>
```

### 整合代码

现在我们可以运行一下两个脚本。在命令行下，运行消费者(receive.php)

```
php receive.php
```

然后，运行生产者(send.php)

```
php send.php
```

消费者将会打印 生产者通过RabbitMQ 发送的消息。receive.php 将会一直运行，等待接收消息(使用 Ctrl+C 停止)，因此尝试从另一个终端运行 send.php。


> 列出RabbitMQ中队列
> 你可能希望看到RabbitMQ中有什么队列和其中有多少消息。你可以使用您可以使用rabbitmqctl 工具
> ``` sudo rabbitmqctl list_queues ```
> 在windows上省略 sudo
> ``` rabbitmqctl list_queues ```


下一节： [创建一个简单的工作队列](#)