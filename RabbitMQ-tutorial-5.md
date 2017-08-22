# 主题 (topics)

> using php-amqplib

在上一节教程中，我们改进了我们的日志记录系统。我们使用可以选择性接收信息的 ```direct``` 类型交换机，而不是使用只能进行虚拟广播的 ```fanout``` 类型交换机。

虽然使用 ```direct``` 类型交换改进了我们的系统，但它仍有限制 - 它不能基于对多个标准进行路由选择。

在我们的日志系统中，我们可能不仅要根据日志的严重性订阅日志，还可以基于发出日志的源进行订阅。你也许在 unix syslog 中了解了这个概念(不理解也没关系，接着往下看，自然就明白了)。

这将给我们的系统带来很大的灵活性 - 我们可能要监听来自 ```cron```的严重错误，也要监听 ```kern``` 的所有日志。

为了在我们的系统中实现这样的功能，我们需要学习一个更复杂的 ```topic exchange``` 。

## Topic exchange

发送到 topic exchange 的消息不能任意命名一个 ```routing key``` - 它必须是由一个```.```划分单词列表。这些单词可以是任意的，但它们通常指定与消息相关联的一些功能。这里有几个有效的 ```routing key```: ```"stock.usd.nyse", "nyse.vmw", "quick.orange.rabbit"``` , ```routing key``` 中可以包含多个单词，最多可以达到255个字节。

````binding key````也必须是相同的形式。```topic exchange```背后的逻辑类似于 ```direct exchange``` - 使用特定```routing key1``` 发送的消息将被传递到与```binding key```绑定的所有队列。但是，```binding key``` 有两个重要的特殊情况：

1. *(星号) 可以代替一个单词
2. \#(哈希) 可以匹配0个或多个单词

在下面的例子中简单解释一下(类似于正则)：

<center> ![python-five](http://www.rabbitmq.com/img/tutorials/python-five.png) </center>

在本例中，我们将发送所有描述动物的消息。消息将使用由三个单词（两个点）组成的```routing key```发送。其中第一个单词描述 速度，第二个描述颜色，第三个描述种类："<speed>.<colour>.<species>"。

我们接着创建三个绑定：Q1 binding key "\*.orange.\*", Q2 binding key "\*.\*.rabbit" "lazy.#"。

这三个绑定可以解释为：

Q1 对所有橙色的动物感兴趣。
Q2 想要获取有关兔子的一切消息，以及所有惰性动物的一切。

一条 ```routing key``` 为 "quick.orange.rabbit" 的消息将传递上面的到两个对列。```routing key``` 为 "lazy.orange.elephant" 的消息也将传递上面的到两个对列。另外 "lazy.pink.rabbit" 消息将只会被传递到Q2一次， 即使它匹配了两个 ```binding key```。"quick.brown.fox" 不匹配任何 ```binding key```, 所以它将被丢弃。

如果我们不遵守以上的规则发送 ```routing key``` 为一个或者四个单词的消息会发生什么？ 比如，"orange" 或者 "quick.orange.male.rabbit"。那么我们将丢失这些消息，因为它们不匹配任何 ```binding key```。

另一方面，"lazy.orange.male.rabbit" 即使它有四个单词，但它能匹配最后一个绑定，并且将被传递到Q2中。

> 注意事项
> topic exchange 很强大，并且表现得和其他类型交换一样
> 当队列与 "#" 绑定时，它将接收所有消息而不管```routing key```的值，如同 ```fanout exchange```
> 当不使用 "*" 和 "#" 特殊字符时，```topic exchange``` 如同 ```direct exchange```

## 整合代码

我们将在日志记录系统中使用主题交换。我们将假设 ```routing key```中有两个单词开始工作，比如："<facility>.<severity>"。

代码几乎和上一节一样。

emit_log_topic.php:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('topic_logs', 'topic', false, false, false);

$routing_key = isset($argv[1]) && !empty($argv[1]) ? $argv[1] : 'anonymous.info';
$data = implode(' ', array_slice($argv, 2));
if(empty($data)) $data = "Hello World!";

$msg = new AMQPMessage($data);

$channel->basic_publish($msg, 'topic_logs', $routing_key);

echo " [x] Sent ",$routing_key,':',$data," \n";

$channel->close();
$connection->close();

?>
```

receive_logs_topic.php:

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->exchange_declare('topic_logs', 'topic', false, false, false);

list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$binding_keys = array_slice($argv, 1);
if( empty($binding_keys )) {
    file_put_contents('php://stderr', "Usage: $argv[0] [binding_key]\n");
    exit(1);
}

foreach($binding_keys as $binding_key) {
    $channel->queue_bind($queue_name, 'topic_logs', $binding_key);
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

收听所有日志：

```php
php receive_logs_topic.php "#"
```

收听所有来自 ```kern``` 的日志：

```php
php receive_logs_topic.php "kern.*"
```

只收听 "critical" 类型的日志：

```php
php receive_logs_topic.php "*.critical"
```

你也可以创建多个绑定：

```php
php receive_logs_topic.php "kern.*" "*.critical"
```

发出 ```routing key```为 "kern.critical"类型的日志

```php
php emit_log_topic.php "kern.critical" "A critical kernel error"
```

至此你可以尽情鼓弄这些程序。需要注意的是，这些代码没有对 ```binding key``` 和 ```routing key``` 赋默认值，除此之外你可能需要路由不止两个单词。

接下来，我们需要弄明白怎样做一个往返信息的远程程序：[远程调用]()