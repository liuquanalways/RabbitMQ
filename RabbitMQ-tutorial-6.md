# 远程调用 PRC(Remote procedure Call)

> using php-amqplib

在第二节教程中，我们知道了怎样使用工作队列将耗时的任务分发给多个消费者。

但是，如果我们需要在远程计算机上运行并等待结果怎么办？那样就是一个新的应用场景。这种模式通常称为远程调用或者叫 PRC。

在本教程中，我们将使用RabbitMQ构建一个RPC系统：一个客户端和一个可扩展的RPC服务器。由于我们没有需要分发的耗时任务，我们将创建一个返回斐波那契数列的虚拟RPC服务。

## 客户端接口

为了说明如何使用RPC服务，我们将创建一个简单的客户端类。它将暴露一个名为call的方法，该方法发送RPC请求并阻塞，知道收到响应：

```php
$fibonacci_rpc = new FibonacciRpcClient();
$response = $fibonacci_rpc->call(30);
echo " [.] Got ", $response, "\n";
```

> **关于RPC的注释** 

> 虽然RPC是一种非常常见的计算模式，但是它经常被人们诟病。当程序员不知道该函数是调用的本地函数还是缓慢的RPC服务时，就会出现问题。这样的混乱导致了系统的不可预知，并且增加了调试程序时不必要的复杂性。滥用RPC服务可能导致不可维护意大利面条式的代码，而不是简化软件。

> 铭记这一点，请参考以下建议：
> + 确保调用本地和调用远程的函数非常易于区分
> + 完善系统的文档，使得系统的组件之间的依赖关系清晰
> + 处理异常情况，当RPC服务器长时间宕机时，客户端该怎样反应

> 当对使用RPC有疑问时应避免使用。如果可以的话，你应该使用异步管道 - 而不是类似RPC的阻塞，结果将被异步返回给下一个计算阶段。

## 回调队列

一般来说，在RabbitMQ上实现RPC很容易。客户端发送请求消息，服务器回复响应消息。为了收到响应，我们需要发送一个回调队列地址与请求。我们可以使用默认队列。来试试吧：

```php
list($queue_name, ,) = $channel->queue_declare("", false, false, true, false);

$msg = new AMQPMessage(
    $payload,
    array('reply_to' => $queue_name));

$channel->basic_publish($msg, '', 'rpc_queue');

# ... then code to read a response message from the callback_queue ...
```

> **消息的属性**

> AMQP 0-9-1 协议为消息预定义了一组默认的14个属性。大多数属性很少使用，除了以下几个：
> + ```delivery_mode```: 将消息标记为持久性(值为2)或者费持久(1). 你可能记得这个属性，我们在 第二节 教程中使用过。
> + ```content_type```: 用于描述MIME类型的编码。例如，对于经常使用的JSON编码，将此属性设置为 "application/json" 是一个很好的做法。
> + ```reply_to```: 主要用于命名一个回调队列
> + ```correlation_id```: 用于将RPC响应与请求相关联

## 关联ID

在上面提出的方法中，我们建议为每个RPC请求创建一个回调队列。这样效率非常低，但幸运的是，这有一个更好的方法 - 让我们为每个客户端创建一个回调队列。

这样有引发了一个新的问题，在该队列中收到响应后，不能确定响应所属的请求。这就是 correlation_id 派上用场的时候。我们将为每个请求设置一个唯一值(correlation_id)。之后，当我们在回调队列中收到一条消息时，我们将查看此属性，并且基于此，我们能够将响应与请求向匹配。如果我们收到一个未知的 correlation_id 的值，我们就能安全的丢弃该消息 - 它不属于我们的请求。

你可能会问，为什么我们应该忽略回调消息中未知的消息，而不是抛出一个异常呢？这是由于在服务端可能发生条件竞争的情况，尽管这不太可能，RPC服务器可能会在发送给我们消息确认后，但在它发送请求消息之前挂掉。如果发生这种情况，RPC服务器重启之后，将再次处理该请求。这就是为什么在客户端上，我们必须合适的处理这些重复的响应，而RPC应该理想地是幂等的。

## 总结

<center>![python-six](http://www.rabbitmq.com/img/tutorials/python-six.png)</center>

### 我们RPC工作流程

当客户端启动时，它创建一个匿名独占回调队列。

对于一个RPC请求，客户端发送一个具有两个属性的消息： reply_to 它被设置为回调队列， correlation_id 它被设置为每个请求的唯一值。

请求被发送到rpc_queue队列。

RPC worker(又名： Server)一直在等待队列上的请求。当请求出现时，它将执行请求的任务，并使用reply_to字段中的队列将结果返回给客户端。

客户端等待回调队列中的数据。当信息出现时，它检查correlation_id属性。如果它与请求中的值相匹配，则返回对应的应用程序的响应。

### 合并代码

求斐波纳契任务：

```php
function fib($n) {
    if ($n == 0)
        return 0;
    if ($n == 1)
        return 1;
    return fib($n-1) + fib($n-2);
}
```

上面我们声明了 fib 函数。它只有假定有效的正整数输入。(不要指望这个函数能够计算比较大的数据，这可能是最慢的递归实现)。

我们 rpc_server.php 代码如下：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

$connection = new AMQPStreamConnection('localhost', 5672, 'guest', 'guest');
$channel = $connection->channel();

$channel->queue_declare('rpc_queue', false, false, false, false);

function fib($n) {
    if ($n == 0)
        return 0;
    if ($n == 1)
        return 1;
    return fib($n-1) + fib($n-2);
}

echo " [x] Awaiting RPC requests\n";
$callback = function($req) {
    $n = intval($req->body);
    echo " [.] fib(", $n, ")\n";

    $msg = new AMQPMessage(
        (string) fib($n),
        array('correlation_id' => $req->get('correlation_id'))
        );

    $req->delivery_info['channel']->basic_publish(
        $msg, '', $req->get('reply_to'));
    $req->delivery_info['channel']->basic_ack(
        $req->delivery_info['delivery_tag']);
};

$channel->basic_qos(null, 1, null);
$channel->basic_consume('rpc_queue', '', false, false, false, false, $callback);

while(count($channel->callbacks)) {
    $channel->wait();
}

$channel->close();
$connection->close();

?>
```

服务器的代码非常简单：

+ 如往常一样，一开始我们建立连接，通道和声明队列。
+ 我们可能想要运行多个服务器进程。为了在多个服务器上平均分配负载，我们需要在 $channel.basic_qos中设置prefetch_count。
+ 我们使用 basic_consume 访问队列。然后我们进入等待请求消息的while循环，执行任务并返回响应。

rpc_client.php 代码如下：

```php
<?php

require_once __DIR__ . '/vendor/autoload.php';
use PhpAmqpLib\Connection\AMQPStreamConnection;
use PhpAmqpLib\Message\AMQPMessage;

class FibonacciRpcClient {
    private $connection;
    private $channel;
    private $callback_queue;
    private $response;
    private $corr_id;

    public function __construct() {
        $this->connection = new AMQPStreamConnection(
            'localhost', 5672, 'guest', 'guest');
        $this->channel = $this->connection->channel();
        list($this->callback_queue, ,) = $this->channel->queue_declare(
            "", false, false, true, false);
        $this->channel->basic_consume(
            $this->callback_queue, '', false, false, false, false,
            array($this, 'on_response'));
    }
    public function on_response($rep) {
        if($rep->get('correlation_id') == $this->corr_id) {
            $this->response = $rep->body;
        }
    }

    public function call($n) {
        $this->response = null;
        $this->corr_id = uniqid();

        $msg = new AMQPMessage(
            (string) $n,
            array('correlation_id' => $this->corr_id,
                  'reply_to' => $this->callback_queue)
            );
        $this->channel->basic_publish($msg, '', 'rpc_queue');
        while(!$this->response) {
            $this->channel->wait();
        }
        return intval($this->response);
    }
};

$fibonacci_rpc = new FibonacciRpcClient();
$response = $fibonacci_rpc->call(30);
echo " [.] Got ", $response, "\n";

?>
```

现在是时候看看我们完整的例子，rpc_client.php 和 rpc_server.php

我们的RPC服务器现在已经准备就绪。我们可以启动服务器：

```php
php rpc_server.php
# => [x] Awaiting RPC requests
```

要求运行客户端的fib数字：

```php
php rpc_client.php
# => [x] Requesting fib(30)
```

这里提出的设计不是RPC服务的唯一可能的实现，但它具有一些重要的优点：

+ 如果RPC服务器处理太慢，你可以运行另一个服务来扩展。 尝试在新的控制台运行一个rpc_server.php.
+ 在客户端，RPC需要发送和接收一条消息。不需要像queue_declare这样的同步调用。因此，RPC客户端只需要一个网络往返单个RPC请求。

我们的代码仍然非常简单，不回去尝试解决更复杂(但重要)的问题，如：

+ 如果服务器没有运行，客户端该如何反应？
+ 客户端是否需要RPC服务调用的过期时间？
+ 如果服务器发生故障并且引发异常, 是否应该将其发送到客户端？
+ 在处理之前防止无效的传入消息(例如检查边界，类型)

> 如果要进行实验，可能会发现管理界面对查看队列很有用。
