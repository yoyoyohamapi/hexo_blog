title: Symfony学习笔记（8）-- GenericEvent
tags: symfony
categories: Symfony学习笔记
------------
在调用dispatcher的dispatch方法时如果不给其传入一个自定义的Event对象，那么Dispatcher会自动创建一个默认的Event对象。而GenericEvent的引入是为了在整个程序中都只使用一个事件对象的情况。GenericEvent封装了一个__事件主题“subject”__。并且其还拥有如下简洁的API:

* ____construct()__ 构造器可以接收事件主题和任何参数

* __getSubject()__ 获取主题 setArgument() 通过键设置一个参数 

* __setArguments()__ 设置一个参数数组 

* __getArgument()__ 通过键获取一个参数值 

* __getArguments()__ 获取所有参数值 hasArgument() 如果某个键值存在，则返回true。


GenericEvent同时还在参数集上实现了__ArrayAccess__，所以可以非常方便的通过传入额外的参数。


```php
use Symfony\Component\EventDispatcher\GenericEvent;
$event = new GenericEvent($subject);//构造函数需要知道当前事件主题
$dispatcher->dispatch('foo', $event);
class FooListener
{
 	public function handler(GenericEvent $event)
 	{
 		if ($event->getSubject() instanceof Foo) {
 		// ...
 		}
 	}
}
```

通过ArrayAccess的API传入和处理事件参数：


```php
use Symfony\Component\EventDispatcher\GenericEvent;
$event = new GenericEvent(
 $subject,
 array('type' => 'foo', 'counter' => 0)
 );
$dispatcher->dispatch('foo', $event);
echo $event['counter'];
class FooListener
{
 	public function handler(GenericEvent $event)
 	{
 		if (isset($event['type']) && $event['type'] === 'foo') {
 			// ... do something
 		}
		$event['counter']++;
 	}
}
```

过滤数据：

```php
use Symfony\Component\EventDispatcher\GenericEvent;
 
$event = new GenericEvent($subject, array('data' => 'foo'));
$dispatcher->dispatch('foo', $event);
 
echo $event['data'];
 
class FooListener
{
	public function filter(GenericEvent $event)
	{
		strtolower($event['data']);
	}
}
```