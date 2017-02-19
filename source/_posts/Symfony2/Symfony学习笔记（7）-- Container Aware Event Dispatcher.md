title: Symfony学习笔记（7）-- Container Aware Event Dispatcher
tags: symfony
categories: Symfony学习笔记
------------
Container Aware Event Dispatcher允许service作为listener来监听事件，这使得EventDispatcher的性能更加卓著。

服务是延迟加载的，这就意味着作为listener的服务只会在事件调度时才被创建。

##安装：
安装仅需要把__ContainerInterface__注入到__ContainerAwareeEventDispatcher__中即可：

```php
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\EventDispatcher\ContainerAwareEventDispatcher;
 
$container = new ContainerBuilder();
$dispatcher = new ContainerAwareEventDispatcher($container);
```

##添加监听者：
* __添加服务__:
 

```php
$dispatcher->addListenerService($eventName, array('foo', 'logListener'));//foo是服务编号，logListner是方法名
```

* __添加订阅器服务__：

```php
$dispatcher->addSubscriberService( 'kernel.store_subscriber', 'StoreSubscriber' );//参数1为服务器编号，参数2为服务类名
```

```php
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
// ...
 
class StoreSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return array(
            'kernel.response' => array(
                array('onKernelResponsePre', 10),
                array('onKernelResponsePost', 0),
            ),
            'store.order'     => array('onStoreOrder', 0),
        );
    }
 
    public function onKernelResponsePre(FilterResponseEvent $event)
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
