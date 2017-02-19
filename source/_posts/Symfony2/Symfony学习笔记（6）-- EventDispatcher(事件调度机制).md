title: Symfony学习笔记（6）-- EventDispatcher(事件调度机制)
tags: symfony
categories: Symfony学习笔记
------------
EventDispatcher通过分配事件并且监听事件来使得应用中的各部件能够相互通信。

##例子：
__假设一个Response对象被创建，那么在它被真正调用之前能够被系统中的其他元素进行一定修改将会是十分实用的。为实现之一过程，Symfony2 kernel将会抛出一个事件---kernel.response。__以下是该例子的工作过程：

* 一个__listener（监听者）__告诉事件调度器__（dispatcher）__他很想监听kernel.response事件。

* Symfony2 __kernel__告诉dispathcher来对__kernel.response__事件进行调度，并向该dispatcher传递一个能够获得Response对象的__Event__对象。

* 该dispatcher通知（可以将该通知过程称之为On方法）所有监听着kernel.response事件的listener，允许他们对Response对象作出修改。

##使用：
###事件：

当一个事件被调度后，它将有一个唯一识别名（例如kernel.response）,该事件能被任何数量的listener监听。通常，事件本身包括一些事件被调度时候的数据。

###命名习惯：

* 小写字母，.，下划线

* 以命名空间.作为名称首部（如kernel.）

* 名称尾部使用一个形象的动词来标明什么动作（action）将被执行（如.request）

>Ex：form.pre_set_data----表单中的“数据设置前”事件。

###事件名称及事件对象：

当dispatcher通知listener的时候，会传递一个__Event__对象给这些listeners。基本的Even类十分简单，仅包含了一个方法用于“停止事件传递”。

通常，一个具体事件的数据将会和Event对象一并传递给listeners，因为listeners需要这些数据。例如被传递给listeners的kernel.response事件，其实际上属于FilterResponseEvent类，该类是基础Event类的一个子类，并包含了getResponse及setResponse等方法，使listener在监听到该事件后能够对Response对象进行修改。

综上，我们能够知道，当创建了一个listener去监听某一事件后，传递给listener对象的Event对象将会是一个Event类的子类，这样，通过事件子类当中的方法，监听器能够取得数据并作出响应。

##调度器（Dispatcher）：
dispatcher是事件调度系统中的核心所在，当一个调度器被创建，其就将维护一个供所有listener进行注册的注册处（registry）。当一个事件通过调度器被调度后，调度器将通知所有listeners注册该事件：

```php
use Symfony\Component\EventDispatcher\EventDispatcher;
$dispatcher = new EventDispatcher();
```

##建立listeners与事件间的联系：

```php
$listener = new AcmeListener();
$dispatcher->addListener('foo.action', array($listener, 'onFooAction'));//将foo.action事件绑定到AcmeListener对象中的onFooAction（很形象的命名，即此时listener正处于foo.action事件的处理）。
/**当AcmeListener通过dispatcher与foo.action事件绑定之后，他就默默等着该事件的发生，当foo.action事件被调度发生的时候，dispatcher将会向该listener的onFooAction方法传递一个Event对象作为参数。**/
```

```php
use Symfony\Component\EventDispatcher\Event;
class AcmeListener
{
 // ...
	public function onFooAction(Event $event)
 	{
 		// ... do something
	}
}
```

但更多情况下，我们需要传递一个更加具体的Event子类对象到listener中的相关方法中，使这些方法能够调用事件中的一些方法来完成操作。


```php
use Symfony\Component\HttpKernel\Event\FilterResponseEvent;
public function onKernelResponse(FilterResponseEvent $event)
{
	$response = $event->getResponse();//我们需要FilterResponseEvent中封装好的方法
 	$request = $event->getRequest();
	// ...
}
```

##事件的创建以及调度：
这里将会讨论如何创建以及调度自己定义的事件。

现在创建一个事件__store.order__，该事件将会在每次订单创建后被调度。

###静态事件类：
我们将会定义一个__StoreEvents__final类来使代码结构更加规范，该类用于定义及说明我们所要定义的事件。


```php
namespace Acme\StoreBundle;
final class StoreEvents
{
	 /**
	 * The store.order event is thrown each time an order is created
	 * in the system.
	 *
	 * The event listener receives an
	 * Acme\StoreBundle\Event\FilterOrderEvent instance.
	 *
	 * @var string
	 */
	 const STORE_ORDER = 'store.order';
}
```

上面代码也显示了我们将会传递一个FilterOrderEvent对象给响应监听器。

###创建一个事件对象：

再调度一个新的事件时，你将创建一个Event实例并且将其传递给dispatcher。该dispatcher再将这个 Event实例传递给绑定到该事件上的所有listeners。但要注意 的是，现实情况下，我们传递的会是一个更加具体，更加富有针对性的Event子类对象给监听器。

在下面这个例子中，listener需要获取一些订单对象，然而基础的Event对象并不能为listeners提供订单信息，因而，我们需要定义一个更加具体的Event子类来满足listener的需求。


```php
namespace Acme\StoreBundle\Event;
use Symfony\Component\EventDispatcher\Event;
use Acme\StoreBundle\Order;
class FilterOrderEvent extends Event
 	{
 		protected $order;
		
		public function __construct(Order $order)
 		{
 			$this->order = $order;
 		}
 		
		public function getOrder()
 		{
 			return $this->order;
 		}
 }
```
如上，每个listener就能够访问订单对象通过getOrder方法。

###进行事件调度：

```php
use Acme\StoreBundle\StoreEvents;
use Acme\StoreBundle\Order;
use Acme\StoreBundle\Event\FilterOrderEvent;
// the order is somehow created or retrieved
$order = new Order();
// ...
// create the FilterOrderEvent and dispatch it
$event = new FilterOrderEvent($order);
$dispatcher->dispatch(StoreEvents::STORE_ORDER, $event);//dispatch（事件名，事件对象）
```

dispatch__执行__后，亦即事件被__调度__，那么与该事件所绑定的listeners将做出反应。


```php
// 某个与"store.order"事件绑定的listener
use Acme\StoreBundle\Event\FilterOrderEvent;
public function onStoreOrder(FilterOrderEvent $event)
	{
 		$order = $event->getOrder();
 		// do something to or with the order
 	}
```

##使用事件订阅者进行事件监听：

上文提到了最普遍的事件监听方式就是通过dispatcher来注册listener进行事件监听。这样的方式使得listener能够监听一个活多个事件，并且在事件被调度时被告知。

另一个事件监听的方式是通过__事件订阅者（event subscriber）__，该类能够正确告诉dispatcher哪些事件应当被订阅。其实现了__EventSubscriberInterface__接口，该接口仅包括了__getSubscribedEvents__方法。下面的爱买创建了一个实现了EventSubscriberInterface接口的订阅器，该订阅器订阅了若干事件：

```php
namespace Acme\StoreBundle\Event;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\FilterResponseEvent;
class StoreSubscriber implements EventSubscriberInterface
{
 	public static function getSubscribedEvents()
 	{
 		return array(
 			'kernel.response' => array(
 				array('onKernelResponsePre', 10),//订阅器的onKernelResponsePre方法订阅了kernel.response事件，其优先级为10
 				array('onKernelResponseMid', 5),
 				array('onKernelResponsePost', 0),
 			),
 			'store.order' => array('onStoreOrder', 0),
 		);
 }
	public function onKernelResponsePre(FilterResponseEvent $event)
 	{
 		// ...
 	}
	public function onKernelResponseMid(FilterResponseEvent $event)
 	{
 		// ...
 	}
	public function onKernelResponsePost(FilterResponseEvent $event)
 	{
 		// ...
 	}
	public function onStoreOrder(FilterOrderEvent $event)
 	{
 		// ...
 	}
}
```

与listen对象不同的是，订阅器（subscriber）自己能够告诉dispatcher哪些事件他应当监听（listener需要通过dispatcher对象的addListener进行listenener与事件的绑定）。

###注册subscriber：

```php
use Acme\StoreBundle\Event\StoreSubscriber;
$subscriber = new StoreSubscriber();
$dispatcher->addSubscriber($subscriber);
```

##阻止事件传递：
如下，通过基础Event类中提供的stopPropagation对事件传递进行阻止：

```php
//某监听器希望在onStoreOrder结束时停止事件传递
use Acme\StoreBundle\Event\FilterOrderEvent;
public function onStoreOrder(FilterOrderEvent $event)
{
 	// ...
	$event->stopPropagation();
}
```

###如何判断事件传递是否被停止？

```php
//某监听器希望在onStoreOrder结束时停止事件传递
use Acme\StoreBundle\Event\FilterOrderEvent;
public function onStoreOrder(FilterOrderEvent $event)
{
	// ...
	$event->stopPropagation();
}
```

##事件调度器的高级应用：

通过在事件对象中获取EventDispatcher来实现更高级的应用。

###延迟加载：

```php
use Symfony\Component\EventDispatcher\Event;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;
use Acme\StoreBundle\Event\StoreSubscriber;
class Foo
{
	private $started = false;
	
	public function myLazyListener(Event $event, $eventName, EventDispatcherInterface $dispatcher)
 	{
 		if (false === $this->started) {//若事件未发生，则订阅事件
 		$subscriber = new StoreSubscriber();
 		$dispatcher->addSubscriber($subscriber);
 	}
	$this->started = true;
	// ... more code
}
```

###在某listener内部调度另一事件：

```php
use Symfony\Component\EventDispatcher\Event;
use Symfony\Component\EventDispatcher\EventDispatcherInterface;
class Foo
{
 	public function myFooListener(Event $event, $eventName, EventDispatcherInterface $dispatcher)
 	{
 		$dispatcher->dispatch('log', $event);
		// ... more code
 	}
}
```

如果要多以用到EventDispatcher对象，那么将该对象的初始化放在__构造函数__和__set__方法中是不错的选择：

* __Constructor__:

```php
use Symfony\Component\EventDispatcher\EventDispatcherInterface;
 
class Foo
{
    protected $dispatcher = null;
 
    public function __construct(EventDispatcherInterface $dispatcher)
    {
        $this->dispatcher = $dispatcher;
    }
}
```

* __Or setter injection__:

```php
use Symfony\Component\EventDispatcher\EventDispatcherInterface;
 
 
class Foo
{
    protected $dispatcher = null;
 
    public function setEventDispatcher(EventDispatcherInterface     $dispatcher)
    {
        $this->dispatcher = $dispatcher;
    }
}
```

##dispatcher简写：

$dispatcher->dispatch('foo.event');//该调度自动传递一个默认的Event对象

##获取事件名称：

事件在调度的时候，事件名称被注入到传递的Event对象中，故listener可以通过通过__getName__方法获得事件名称。

```php
use Symfony\Component\EventDispatcher\Event;
 
class Foo
{
	public function myEventListener(Event $event)
	{
		echo $event->getName();
	}
}
```