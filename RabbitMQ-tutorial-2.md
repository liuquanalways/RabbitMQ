# Work Queues

> using php-amqplib

<center>![python-two](http://www.rabbitmq.com/img/tutorials/python-two.png)</center>

在第一篇教程中，我们编写了两段程序通过一个指定的队列来收发消息。在本节教程中，我们将创建一个工作队列，用于多个工作人员之间分配耗时的任务。

工作队列(又名：任务队列)其主要的思想是避免立即执行资源密集型任务，并且阻塞进程等待任务完成。相反，我们让这类型任务稍后执行，我们将它封装为消息，并将其添加到任务队列中。在后台启动一个工作进程，读取工作队列中的任务，并且解释执行。当你运行多个工作进程时，他们共享工作队列中的任务。

这个思想在Web应用程序中特别有用，用于处理在短时间的HTTP请求中无法处理的复杂任务。


## 准备

在上一节的教程中，我们发送了一个包含 "hello world" 的消息。下面我们将发送代表复杂任务的字符串。我们当前没有一个真实的应用场景，比如：调整图像的大小或者渲染pdf文件，所以我们通过使用 ```sleep()``` 函数来模拟复杂的任务需要较多的处理时间的场景。我们假设字符串中的 ``` . ``` 的数量为任务的复杂度；每个``` . ```将占用一秒钟的工作时间。例如 ``` Hello ... ``` 代表任务需要执行 3 秒钟。

我们稍微修改上一节例子中的 ```send.php```，以允许从命令行发送任意消息。这段程序将会把任务放到工作队列中，所以我们将其命名为 ``` new_task.php ```

```php

$data = implode(' ', array_slice($argv, 1));
if(empty($data)) $data = "Hello World!";
$msg = new AMQPMessage($data,
                        array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
                      );

$channel->basic_publish($msg, '', 'task_queue');

echo " [x] Sent ", $data, "\n";

```

我们之前的 ```receive.php```也需要一些修改：它需要为消息体中的每个 ``` . ``` 伪造一秒的工作时间。它将从队列中取出消息并解析执行任务，所以我们称之为 ``` worker.php ```

```php

$callback = function($msg){
  echo " [x] Received ", $msg->body, "\n";
  sleep(substr_count($msg->body, '.'));
  echo " [x] Done", "\n";
};

$channel->basic_qos(null, 1, null);
$channel->basic_consume('task_queue', '', false, true, false, false, $callback);

```

请注意，我们模拟的任务会占用执行时间。

运行他们的方式和上一节一样：

```
# shell 1

php worker.php

# shell 2

php new_task.php "A very hard task takes two seconds.."

```

## 循环调度

使用队列的有点之一就是能够轻松地并行工作。如果我们正面临大量积压的工作，我们可以启动更多的工作进程，这样就可以轻松地扩展系统的处理能力。

首先，我们尝试同时运行两个```worker.php``` 脚本。它们都会从队列中获取消息，但是究竟如何，让我们试试看。

你需要打开三个命令行控制台，其中两个运行```worker.php```脚本，它们就是我们启动的两个消费者-C1和C2.


```php
# shell 1

php worker.php
# => [*] Waiting for messages. To exit press CTRL+C
```

```php
# shell 2

php worker.php
# => [*] Waiting for messages. To exit press CTRL+C
```

在第三个控制台我们将发布新的任务。一旦你开始使用消费者，你可以发布一些消息：

```php
# shell 3
php new_task.php First message.
php new_task.php Second message..
php new_task.php Third message...
php new_task.php Fourth message....
php new_task.php Fifth message.....

```

接着我们看看两个消费者控制台输出的内容：

```php
# shell 1
php worker.php
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'First message.'
# => [x] Received 'Third message...'
# => [x] Received 'Fifth message.....'
```

```php
# shell 2
php worker.php
# => [*] Waiting for messages. To exit press CTRL+C
# => [x] Received 'Second message..'
# => [x] Received 'Fourth message....'
```

在默认情况下，RabbitMQ 将按顺序将队列中的消息分发给消费者。平均每个消费者获得相同数量的消息。这种奋发消息的方式叫做循环(round-robin)。你可以尝试一下三个或者更多的消费者同时运行。

## 消息确认机制 (ack)

执行任务可能需要几秒钟。你可能会想，如果当一个消费者在执行一个长时间的任务的过程中异常退出会怎样。在我们目前的代码下，一旦RabbitMQ向消费者发送了消息后，此条消息将立即从内存中删除。在这种情况下，如果你杀掉一个正在处理消息工作进程，那么我们将丢失这条消息。并且我们还会丢失发送给这个工作进程所有还未处理的消息。

但是我们不想丢失任何任务。如果一个工作进程发生异常退出，我们希望将他在处理的任务交给另一个工作进程继续处理。

为了确保消息永远不会丢失，RabbitMQ支持消息确认机制。消费者发送一个确认信息告诉RabbitMQ已经收到消息并且处理完成，此后RabbitMQ就可以自由删除这条消息了。

如果消费者进程异常退出(其通道关闭，连接断开或者TCP连接丢失)，而不发送确认信息，RabbitMQ将会认为此进程当前处理的消息未处理完成，就会将此消息重新放回消息队列。如果此时有其他消费者在线，则会迅速将这条消息重新提供给另一个消费者。这样就可以确保没有消息丢失。即使有消费者进程异常推出的情况。

不会有消息处理超时的情况，只有当消费者进程异常退出时，RabbitMQ才会将其消息重发。即使处理消息需要一段非常长的时间。

在默认情况下，消息确认机制是关闭的。现在是时候开启消息确认机制，将basic_consumer的第四个参数设置为false(true表示不开启消息确认)，并且工作进程处理完消息后发送确认消息。

```php
# 处理消息回调函数
$callback = function($msg){
  echo " [x] Received ", $msg->body, "\n";
  sleep(substr_count($msg->body, '.'));
  echo " [x] Done", "\n";
  $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};
# 开启消息确认
$channel->basic_consume('task_queue', '', false, false, false, false, $callback);

```

使用这段代码，我们可以确认即使在处理消息时，使用 CTRL + C 杀掉一个工作进程，也不会丢失任何消息。工作进程挂掉之后不久，所有未确认的消息将会被重新发送。

## 忘记发送确认消息

忘记发送确认消息是一个常见的错误，并且这是一个很容易犯的错误，但是它的后果是很严重的。当你的客户端退出时，消息将被重新发送，但RabbitMQ将会消耗越来越多的内存，因为它将无法释放任何没有确认信息的消息。

为了调试这种错误，你可以使用 rabbitmqctl 打印 messages_unacknowledged 字段：

```
sudo rabbitmqctl list_queues name messages_ready messages_unacknowledged
```

在windows 上去掉前面的 sudo

```
rabbitmqctl.bat list_queues name messages_ready messages_unacknowledged
```

## 消息持久化

我们已经学习了如何确保即使消费者异常退出后，任务也不会丢失。但是如果RabbitMQ服务器挂了，我们的任务任然会丢失。

当RabbitMQ退出或崩溃时，它会丢失队列和其中的消息，除非你告诉他要持久化这些信息。需要两件事来确保消息不会丢失：我们需要将队列和消息标记为 durable(持久化)。

首先，我们需要确保RabbitMQ不会丢失消息队列。为了做到这一点，我们需要将队列声明为持久化，只要将 queue_declare 的第三个参数设置为 true 就行了：

```php
$channel->queue_declare('hello', false, true, false, false);
```

虽然这个命令本身是正确的，但是在我们目前的设置中是不行的。那是因为我们已经声明了一个非持久化名为```hello```的队列了。RabbitMQ不允许使用不同的参数重新定义一个已经存在的队列，并且会返回一个错误给任何尝试做此事的程序。但有一个快速的解决方法-我们用不同的名称声明一个队列，例如 task_queue:

```php
$channel->queue_declare('task_queue', false, true, false, false);
```

设置为 true 的标志需要同时应用于生产者和消费者中。

到这一步，我们确信即使RabbitMQ重启，task_queue队列也不会丢失。现在我们需要将我们需要的消息标记为持久化 - 通过设置AMQPMessage 的参数 delivery = 2:

```php
$msg = new AMQPMessage($data,
       array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
       );
```

## 注意消息的持久性

将消息标记为持久化并不能完全确保消息一点也不会丢失。虽然它告诉RabbitMQ将消息保存到磁盘，但是当RabbitMQ接收到消息并且还没有保存到磁盘之间，仍然有一个很短的时间段。此外，RabbitMQ也不会每来一条信息就执行 fsync - 它可能只是保存在缓存中，而不是真的写入磁盘。持久化为保证不强，但是对于我们简单的任务队列来说已经足够了。如果你需要更加强大的保证，那么你可以使用 ```publisher confirms```(起初不要过分纠结于此)。

## 公平调度

你可能已经注意到，任务调度并不是完全按照我们所设想一样执行。比如在两个消费者的情况下，当所有奇数序号的任务量都很大，并且偶数序号的任务量小的情况下，一个消费者将不断忙碌，另一个消费者几乎没有任何工作量。在这种情境下，RabbitMQ依然不知道什么信息，还是会将消息平均分配。

这种情况的发生是因为RabbitMQ只管分配进入队列的消息。它不会去看消费者的未确认消息的数量。它只是盲目地向第n个消费者发送第n条消息。

<center>![prefetch-count](http://www.rabbitmq.com/img/tutorials/prefetch-count.png)</center>

为了解决以上问题，我们可以通过设置 basic_qos 第二个参数 prefetch_count = 1。这一项告诉RabbitMQ不要一次给一个消费者发送多个消息。或者换一种说法，在确认前一个消息之前，不要向消费者发送新的消息。相反，新的消息将发送到一个处于空闲的消费者。

```php
$channel->basic_qos(null, 1, null);
``` 

## 关于队列的大小

如果所有的消费者进程都处于忙碌状态，你的队列可能会溢出。你应该留意一下这种情况，也许你可以增加更多的消费者进程，或者采取一些其他的策略。

## 程序汇总

我们最终版本的 new_task.php

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('task_queue', false, true, false, false);

$data = implode(' ', array_slice($argv, 1));
if(empty($data)) $data = "Hello World!";
$msg = new AMQPMessage($data,
                        array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
                      );

$channel->basic_publish($msg, '', 'task_queue');

echo " [x] Sent ", $data, "\n";

$channel->close();
$connection->close();

?>
```

worker.php

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('task_queue', false, true, false, false);

echo ' [*] Waiting for messages. To exit press CTRL+C', "\n";

$callback = function($msg){
  echo " [x] Received ", $msg->body, "\n";
  sleep(substr_count($msg->body, '.'));
  echo " [x] Done", "\n";
  $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
};

$channel->basic_qos(null, 1, null);
$channel->basic_consume('task_queue', '', false, false, false, false, $callback);

while(count($channel->callbacks)) {
    $channel->wait();
}

$channel->close();
$connection->close();

?>
```

使用消息确认和预读取你可以设置一个持久化的工作队列，即使RabbitMQ重新启动，持久性选项也能让任务不丢失。

现在我们可以进入第三节学习向多个消费者发送给相同的消息。