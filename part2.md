## 工作队列

在第一章我们编写了一段发送和接收消息的代码。在这一章我们会创建一个工作队列来分发一堆工作中的耗时任务。

工作队列的思想是避免立刻执行资源密集型的任务而不得不等到它结束。所以我们制定一个任务延后处理。我们封装好一个任务作为消息，发送到队列中。一个后台的工作进程会pop任务出来然后执行。进程间会共享任务资源。

这个概念在web程序中相当有用，因为在一个http短连接中无法处理这么复杂的任务。

### 准备工作

在上一章我们发送一个消息"Hello World!"。现在我们要发送一些字符串来代表复杂的任务。我们实际上没有真正的任务，比如调整图片的大小，提交pdf文件之类的。所以我们不如假装忙得不可开交——在代码中用sleep()函数来表达。我们用一些句号来表达复杂度，1个句号工作时长1秒。比如，一个假任务用Hello...来表示要花费3秒。

我们稍微改变一下send.php的代码，随便在命令行中写一些消息来发送。这段程序会为工作队列发送一些任务，所以命名为new_task.php：

	$data = implode(' ', array_slice($argv, 1));
	if(empty($data)) $data = "Hello World!";
	$msg = new AMQPMessage($data);
	
	$channel->basic_publish($msg, '', 'hello');
	
	echo " [x] Sent ", $data, "\n";


receive.php中也有所变更：它要假设消息体中有一些复杂的工作。它会从队列中pop消息然后执行任务，我们把程序命名为worker.php:

	$callback = function($msg){
	  echo " [x] Received ", $msg->body, "\n";
	  sleep(substr_count($msg->body, '.'));
	  echo " [x] Done", "\n";
	};
	
	$channel->basic_consume('hello', '', false, true, false, false, $callback);

注意任务模拟的执行时间，然后执行代码。

### 平均调度

使用任务队列的好处之一是可以轻松的并行工作。如果我们创建了一堆的工作，我们可以增加worker，从而轻松达到规模化。

首先，我们同时执行两个worker.php脚本，他们都会从队列中获取消息，但是是如何做到的？让我们看看。

你需要打开三个控制台。两个执行worker.php，说明有有两个消费者——C1和C2。

	# shell 1
	php worker.php
	# => [*] Waiting for messages. To exit press CTRL+C
	# => [x] Received 'First message.'
	# => [x] Received 'Third message...'
	# => [x] Received 'Fifth message.....'

-------------------------
	
	# shell 2
	php worker.php
	# => [*] Waiting for messages. To exit press CTRL+C
	# => [x] Received 'Second message..'
	# => [x] Received 'Fourth message....'

在第三个控制台我们会发布一些任务。一旦你启动了消费者脚本，你就可以推送一些消息了：

	# shell 3
	php new_task.php First message.
	php new_task.php Second message..
	php new_task.php Third message...
	php new_task.php Fourth message....
	php new_task.php Fifth message.....

RabbitMQ默认会轮流发送消息给消费者，每位消费者都能平均获得同样数量的消息。这种消息分发方式成为轮询。你可以试试更多的worker来看看。

### 消息确认

执行一个任务要花一定的时间。有这么一个场景：如果一个消费者启动一个耗时的任务，然后只做了一部分就停止执行，同时RabbitMQ又发送消息给消费者并且把消息删除。在上述场景中，如果你杀掉一个工作进程将会失去消息，同时也会失去还未执行其它消息。

可我们不想失去任何一个任务。如果一个工作进程中止了，我们会想让任务被分配到其它的工作进程中。

为了确保一个消息永不丢失，RabbitMQ支持消息确认。一个消息确认会被消费者发送到RabbitMQ，告诉RabbitMQ一个特定的消息已被接收并执行完毕，RabbitMQ可以删除这个消息了。

如果一个消费者中止了，没有发送消息确认，RabbitMQ就会知道有一个消息没有被执行完毕并把消息重新入队。这时假如有其它的消费者在线，RabbitMQ会重新发送消息给其它消费者。上述可以保证消息不丢失，即使工作偶尔中止。

消息是不会超时的：当一个消费者中止时，RabbitMQ会重新发送消息出去。就算一个消息运行非常长时间也无所谓。

消息确认是默认关闭的。通过设置basic_consume 的第四个参数为false就可以打开。当任务完成时，工作进程会发送一个消息确认给RabbitMQ。

	$callback = function($msg){
	  echo " [x] Received ", $msg->body, "\n";
	  sleep(substr_count($msg->body, '.'));
	  echo " [x] Done", "\n";
	  $msg->delivery_info['channel']->basic_ack($msg->delivery_info['delivery_tag']);
	};
	
	$channel->basic_consume('task_queue', '', false, false, false, false, $callback);

### 消息持久化

我们已经掌握了即使消费者中止，任务也不会丢失的方法。但是RabbitMQ服务器宕机的话，任务其实还是会丢失的。

当RabbitMQ终止或者崩溃的时候，它会丢失队列和消息，除非你让它记住。有两种方法可以让消失不丢失：1、队列持久化；2、消息持久化。

首先让RabbitMQ不丢失队列。为了达到这种效果，我们需要声明*队列持久化*。因此我们要把`queue_declare`的第三个参数设置为true：

	$channel->queue_declare('hello', false, true, false, false);

虽然这条命令本身是对的。但是在当前场景是无法运行的。因为我们之前就已经定义了一个hello队列，而这个队列是没有持久化的。RabbitMQ不允许你用不同的参数去重新定义一个已存在的队列，这样是会报致命错误。重新定义一个队列task_queue就行:

	$channel->queue_declare('task_queue', false, true, false, false);

生产者和消费者都要设置为true。

这时即使RabbitMQ 重启了,我们可以保证`task_queue`这个队列不会丢失，。现在我们需要让*消息持久化*——在`AMQPMessage`中设置第2个参数为数组 `delivery_mode = 2`。

	$msg = new AMQPMessage($data,
       array('delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT)
     );

----------------------

    注意：
    这里设置消息持久化也无法完全保证消息不会丢失。虽然它会指示RabbitMQ 把消息存储在硬盘，但是总是会有一小段时间RabbitMQ接受了消息，却没来得及存储在硬
	盘中。而且RabbitMQ也不会为每段消息做fsync(2)处理：它也许只是把消息保存在内存而没有存储在硬盘。所以这个持久化是没有保障的，但是对于简单的场景来说
	足够了。如果你要更为保障的方法，你就要用[发布确认](https://www.rabbitmq.com/confirms.html)了。
    
### 公平调度

你会注意到分配原则还是跟自己的期望不符。假设有这么一个场景，有两个工作进程，当全部的奇数消息十分庞大，而偶数消息却十分简单，那么一个进程就会非常忙碌，另一个进程会非常轻松。呵呵，RabbitMQ 才不知道这些鬼，它就只会平均分配消息。

这是因为RabbitMQ 只是负责发送进入队列的消息，它不会看消费者有多少没有确认的消息。它只是盲目的平均分配消息。

为了解决上述问题我们要设置`basic_qos`方法中的`prefetch_count = 1`，这个方法告诉RabbitMQ不要给工作进程超过1条消息。换句话说，就是等到工作进程完成任务并确认消息后，RabbitMQ才分配消息给工作进程。

	$channel->basic_qos(null, 1, null);

-----------------------

    注意：
    假如所有工作进程都十分繁忙，你的队列会塞满的，你就要注意一下了。你可以加多一点工作进程，或者用其它策略。

源码：

[new_task.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/new_task.php)

[worker.php](https://github.com/rabbitmq/rabbitmq-tutorials/blob/master/php/worker.php)

