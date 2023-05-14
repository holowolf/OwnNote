## Laravel事件用法总结

Laravel 的事件提供了一个简单的观察者实现，能够订阅和监听应用中发生的各种事件。事件类保存在 app/Events 目录中，而这些事件的的监听器则被保存在 app/Listeners 目录下。这些目录只有当你使用 Artisan 命令来生成事件和监听器时才会被自动创建。

事件机制是一种很好的应用解耦方式，因为一个事件可以拥有多个互不依赖的监听器。例如，如果你希望每次订单发货时向用户发送一个 Slack 通知。你可以简单地发起一个 OrderShipped 事件，让监听器接收之后转化成一个 Slack 通知，这样你就可以不用把订单的业务代码跟 Slack 通知的代码耦合在一起了。

#### 生成一个事件类

比如通过 artisan 命令生成一个 UserLogin 事件：

```
php artisan make:event UserLogin
```

在 app/Events 中就会自动生成一个 UserLogin.php 文件，内容不多，如下：

```
<?php

namespace App\Events;

use Illuminate\Broadcasting\Channel;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;

class UserLogin
{
    use InteractsWithSockets, SerializesModels;

    /**
     * Create a new event instance.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Get the channels the event should broadcast on.
     *
     * @return Channel|array
     */
    public function broadcastOn()
    {
        return new PrivateChannel('channel-name');
    }
}
```

#### 定义监听器

一个事件可以被一个或多个监听器监听，也就是观察者模式，我们可以定义多个监听器，当这个事件发生，执行一系列逻辑。

在 EventServiceProvider 的 $listen 中可以定义事件和监听器，如下：

```
protected $listen = [
    'App\Events\UserLogin' => [
        'App\Lisenter\DoSomething1',
        'App\Lisenter\Dosomething2',
    ],
];
```

然后执行 artisan 命令，就可以自动在 app/Lisenter 目录生成监听器。

```
php artisan event::generate
```

可以看到 app/Lisenter 目录多了 DoSomething1.php 和 DoSomething2.php 两个文件，我们看看其中一个内容：

```
<?php

namespace App\Lisenter;

use App\Events\UserLogin;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Contracts\Queue\ShouldQueue;

class DoSomething1
{
    /**
     * Create the event listener.
     *
     * @return void
     */
    public function __construct()
    {
        //
    }

    /**
     * Handle the event.
     *
     * @param  UserLogin  $event
     * @return void
     */
    public function handle(UserLogin $event)
    {
        info('do something1');
    }
}
```

在两个监听器的 handle 方法中我们打印一个日志来测试一下，如代码 handle 方法所示。

#### 分发和触发事件

我们在某个控制器的方法中来分发事件，也就是触发事件，看监听器是否正常工作。

就是一句话：

```
event(new UserLogin());
```

然后我们请求这个控制器，观察日志，发现打印了日志：

[2018-06-17 10:04:29] local.INFO: do something1

[2018-06-17 10:04:29] local.INFO: do something2

那么这个事件 - 监听机制就正常工作了。

#### 队列异步处理

如果某个监听器需要执行的操作比较慢，可以放到消息队列进行异步处理。

比如把上面的 DoSomething1 改成需要放入队列的，只需要 implements ShoulQueue 接口。

```
class DoSomething1 implements ShouldQueue
```

也可以指定队列驱动，如下代码。

```
/**
 * 任务应该发送到的队列的连接的名称
 *
 * @var string|null
 */
public $connection = 'redis';

/**
 * 任务应该发送到的队列的名称
 *
 * @var string|null
 */
public $queue = 'listeners';
```

我们再次执行控制器方法。

日志里没有打印 do something1，只有 do something2，但是在 redis 队列里发现了一个名为 queues:default 的列表。

```
{"job":"Illuminate\\Events\\CallQueuedHandler@call","data":{"class":"App\\Listener\\DoSomething1","method":"handle","data":"a:1:{i:0;O:20:\"App\\Events\\UserLogin\":1:{s:6:\"socket\";N;}}"},"id":"3D7VDUwueYGtUvsazicWsifwWQxnnLID","attempts":1}
```

这个时候需要使用 php artisan queue:work 执行队列任务，才是真正执行 DoSomething1 这个监听器的 handle 方法.
