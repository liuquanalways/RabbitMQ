# 路由(Routing)

> using php-amqplib

在上一节中，我们构建了一个简单的日志系统。它能够向多个接收器广播消息。

在本教程中，我们将在上一节基础上添加一个功能 - 我们将实现能够只订阅一部分消息。例如，我们想能够仅将关键的错误消息保存到日志文件(以节省磁盘空间)，同时仍然能够在控制台打印所有日志消息。

## 绑定

在前面的例子中，我们已经创建了绑定。你能回忆之前的代码：

```php
$channel->queue_bind($queue_name, 'logs');
```

绑定是交换机和队列之间的关系。可以简单地理解为：队列对来自绑定交换机的信息感兴趣。

绑定可以占用一个额外的 ```routing_key``` 参数。为了避免与 ```$channel::basic_publish``` 参数混淆，我们将其称为 ```binding key ```。下面就是我们如何创建一个绑定：

```php
$binding_key = 'black';
$channel->queue_bind($queue_name, $exchange_name, $binding_key);
```

绑定键的含义取决于交换机的类型。上一节中我们使用的 fanout 会直接忽略它的值。

## 直接交换 (Direct exchange)

我们上一节教程中的日志系统是向所有消费者广播所有日志消息。我们希望扩展上一节中的日志系统，使其允许基于消息的严重性过滤消息。例如，我们可能希望将日志消息写入磁盘的脚本仅接收严重错误消息，而不会在警告和提示信息上面浪费磁盘空间。

我们之前使用的 ```fanout``` 类型交换机，它没有什么灵活性 - 只能广播消息。

这里我们将使用 ```direct``` 交换机，直接交换背后的算法很简单 - 消息将传递到 ```binding key ``` 与消息的 ```routing key``` 完全匹配的队列。

为了说明上面步骤，参考下面的设置：

<center>![direct-exchange](http://www.rabbitmq.com/img/tutorials/direct-exchange.png)</center>

在这设置中，我们可以看到 ```X exchange``` 上绑定了两个队列(Q1, Q2). 第一个队列绑定了 binding key 为 ```orange```, 第二个队列绑定了两个键，一个是 ```black```另一个是 ```green```。

在这样的设置中，发布到交换机上息具有 ```orange``` 路由秘钥的消息将被路由到队列 Q1。消息具有black 或者 green 路由秘钥的消息将会路由到Q2。其它所有的消息将会被丢弃。

## 多重绑定

<center>![direct-multiple](http://www.rabbitmq.com/img/tutorials/direct-exchange-multiple.png)</center>

使用相同的键绑定到多个队列上是完全合法的。在上面的示例中，我们可以在X和Q1之间添加绑定 ```black```。那样的话，```direct``` 交换机将能如 ```fanout``` 交换机一样，在多个匹配的队列之间广播相同的消息。具有 ```black``` 路由键的消息将会被发送到Q1和Q2中。

## 发出日志

我们将使用此模型构建日志记录系统。我们将把消息发送到 ```direct``` 类型交换机上, 而不是使用 ```fanout```类型。我们将提供日志严重性作为 ```routing key```。这样接收消息的脚本就能够选择要接受消息的严重性进行接收。我们首先关注发送日志部分：

一如以往，我们首先要建立一个交换机：

```php
$channel->exchange_declare('direct_logs', 'direct', false, false, false);
```

然后，我们准备发送信息：

```php
$channel->exchange_declare('direct_logs', 'direct', false, false, false);
$channel->basic_publish($msg, 'direct_logs', $severity);
```

## 订阅

接收消息大致和上一节教程一样，除了一个例外 - 我们将为每个我们感兴趣的消息严重性创建一个新的绑定。

```php
foreach($severities as $severity) {
    $channel->queue_bind($queue_name, 'direct_logs', $severity);
}
```

## 代码整合

<center>![python-four](http://www.rabbitmq.com/img/tutorials/python-four.png)</center>

emit_log_direct.php:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('direct_logs', 'direct', false, false, false);

$severity = isset($argv[1]) && !empty($argv[1]) ? $argv[1] : 'info';

$data = implode(' ', array_slice($argv, 2));
if(empty($data)) $data = "Hello World!";

$msg = new AMQPMessage($data);

$channel->basic_publish($msg, 'direct_logs', $severity);

echo " [x] Sent ",$severity,':',$data," \n";

$channel->close();
$connection->close();

?>
```

receive_logs_direct.php:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('direct_logs', 'direct', false, false, false);

list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$severities = array_slice($argv, 1);
if(empty($severities )) {
    file_put_contents('php://stderr', "Usage: $argv[0] [info] [warning] [error]\n");
    exit(1);
}

foreach($severities as $severity) {
    $channel->queue_bind($queue_name, 'direct_logs', $severity);
}

echo ' [*] Waiting for logs. To exit press CTRL+C', "\n";

$callback = function($msg){
  echo ' [x] ',$msg->delivery_info['routing_key'], ':', $msg->body, "\n";
};

$channel->basic_consume($queue_name, '', false, true, false, false, $callback);

while(count($channel->callbacks)) {
    $channel->wait();
}

$channel->close();
$connection->close();

?>
```

如果你只想将 ```warning``` 和 ```error``` (而非 ```info```) 日志信息保存到文件中，只用打开一个控制台键入:

```php
php receive_logs_direct.php warning error > logs_from_rabbit.log
```

如果你想在屏幕上看到所有的日志信息，打开一个控制台键入：

```php
php receive_logs_direct.php info warning error
```

而且，例如，要发送 ```error``` 类型日志，只需要键入：

```php
php emit_log_direct.php error "Run. Run. Or it will explode."
```

接下来请阅读：[怎样根据模式订阅消息](#)