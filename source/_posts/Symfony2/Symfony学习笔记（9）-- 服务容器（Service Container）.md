title: Symfony学习笔记（9）-- 服务容器（Service Container）
tags: symfony
categories: Symfony学习笔记
------------
####首先，你得知道什么是服务？

服务也是对象，但服务更加针对一些__“全局性”__的任务，例如，完成传递邮件任务的对象就可以是一个服务，任何时候我需要传递邮件时，只要通过该服务便能轻松达到目的，而邮件对象则不能称之为服务，因为它是__具象__的，即一个具体的邮件并不能在任何时候像服务一样完成“全局性”的工作。

####那么，使用服务有什么好处？

显然，我们可以将一些完成“全局性”任务的方法封装到某一服务类中，这样，当该服务类的服务对象一旦建立（或者说我们开启这项服务），那么在整个应用当中，我们可以随时随地享受这些服务来解决问题，而不用每次建立不同的对象，调用不同对象的不同方法，这更加使得我们的代码结构更加清晰，任务职责更加方便确定。

####在Symfony2中，什么是服务容器（Service Container）？

服务容器同样是一个PHP对象，负责服务的初始化。亦即，服务容器是各个服务的孵化器。

考虑下面这样一个例子，你创建了一个简单的PHP类用以传递邮件，假设没有服务容器，那么很遗憾，你就必须扮演那只母鸡（服务容器），去孵化这只小鸡（服务）。


```php
use Acme\HelloBundle\Mailer;//Mailer类即一个服务，其中的send方法
$mailer = new Mailer('sendmail');//没有服务容器，手动创建服务
$mailer->send('ryan@foobar.net', ...);
```

看见没有？你觉得很简单？但是如果在每个你需要传递邮件的地方都这样去手动获取服务对象，是不是很恶心？

来看看服务容器是怎么做的吧！
当你被上面的例子恶心到以后，你还是想偷懒说还是交给别的人帮我“孵化”服务吧，那么你就得求助与服务容器（service container）了。

__首先你得教会服务容器怎么创建服务（通过配置文件）__

```yaml
# app/config/config.yml
services:
    my_mailer:
        class:        Acme\HelloBundle\Mailer
        arguments:    [sendmail]
```        

好了，在任何控制里你想要获得某项服务就通过__get__方法拿取就好。


```php
class HelloController extends Controller //HelloController继承了BaseController，且BaseController提供了访问服务容器的机会
{
    // ...
 
    public function sendEmailAction()
    {
        // ...
        $mailer = $this->get('my_mailer');//你向服务容器请求名字为“my_mailer”的服务，那么容器就会为你创建该服务
        $mailer->send('ryan@foobar.net', ...);
    }
}
```

使用服务容器的一大好处是，你想要服务容器时才会给你创建，若你不用那么相应服务也就不会产生。亦即，丰富的服务并不会消耗多少性能。

要注意的是，上例Mailer服务对象只会创建一次，任何时候你向服务容器请求的Mailer服务对象都只对应一个相同的实例（当然，Symfony也允许你获得不同的服务对象）。

####服务的参数配置：

```yaml
# app/config/config.yml
parameters:
    my_mailer.class:      Acme\HelloBundle\Mailer
    my_mailer.transport:  sendmail
 
services:
    my_mailer:
        class:        "%my_mailer.class%"
        arguments:    ["%my_mailer.transport%"]
```

将服务配置成参数的好处是显而易见的的：我不用反复书写冗长的代码，一切只要指定好参数名即可，同时，参数可用于各个不同的服务。

####数组形式的参数如下书写：

```yaml
# app/config/config.yml
parameters:
    my_mailer.gateways:
        - mail1
        - mail2
        - mail3
    my_multilang.language_fallback:
        en:
            - en
            - fr
        fr:
            - fr
            - en
```

##如何引入其他服务配置文件？
服务容器的建立是通过一个简单的配置文件（默认情况下是app/config/config.yml），本节将讨论如何将在其他地方定义的服务配置文件引入到默认配置中（app/config/config.yml）。

###通过__impotrs__方式引入
之前，我们将对Mailer服务的配置直接放到了app/config/config.yml下，但是，将其放到其所在Bundle的目录下似乎更符合实际，下面，我们将对Mailer服务的配置转移到

__src/Acme/HelloBundle/Resources/config/services.yml__下。


```yaml
# src/Acme/HelloBundle/Resources/config/services.yml
parameters:
    my_mailer.class:      Acme\HelloBundle\Mailer
    my_mailer.transport:  sendmail
 
services:
    my_mailer:
        class:        "%my_mailer.class%"
        arguments:    ["%my_mailer.transport%"]
```

再将该配置引入到主配置__app/config/config.yml中：

```yaml
# app/config/config.yml
imports:
    - { resource: "@AcmeHelloBundle/Resources/config/services.yml" }
```


###通过容器扩展（Container Extensions）引入服务配置

一个服务容器扩展将由bundle的作者所创建，该扩展用来完成两件事：

1. 引入所有的服务容器，这些服务容器用来配置该Bundle的服务。

2. 提供语法上简介配置，让bundle能够直接被配置，而不需要再与bundle的服务容器配置参数交互。

概括说来，就是对于某个Bundle，我们可以定制化的配置一些他所带有的服务为几用。


以__FrameworkBundle__作为例子，以下代码显示了我们配置了FrameworkBundle的许多服务，如__secret__,__form__等等。

```yaml
# app/config/config.yml
framework://服务，该服务将会被“服务容器扩展”所配置
    secret:          xxxxxxxxxx
    form:            true
    csrf_protection: true
    router:        { resource: "%kernel.root_dir%/config/routing.yml" }
    # ...
```

当该配置文件被解析后，容器会寻找一个能够直接操纵framework配置的扩展。该扩展存在与FrameworkBundle内部，并被用来加载该bundle下的服务。如果你从你的配置文件中完全去掉framework键，Symfony 核心服务将不会被加载（因为Framework因此丢失）。

同时，你可以对该服务进行一些个性化服务，如自定义error_handler, csrf_protection, router的配置，bundle的服务容器扩展将会识别这些配置，并创建相应的parameters和services。另外，大部分服务容器扩展都有校验功能，告诉你哪些配置是存在错误的。

Symfony本身的服务容器能够识别parameters，services 和imports命令，其它的命令则需要你自己创建服务容器扩展来处理。

##引入（注入）服务：
假定你有一个新的服务，叫做是__NewsletterManager__，其负责管理新建。上文中的my_mailer服务已经能够很好的实现邮件传递了，所以你可以在NewsletterManager服务中使用该服务提供的方法。NewsletterManager类如下：

```php
// src/Acme/HelloBundle/Newsletter/NewsletterManager.php
namespace Acme\HelloBundle\Newsletter;
 
use Acme\HelloBundle\Mailer;
 
class NewsletterManager
{
    protected $mailer;
 
    public function __construct(Mailer $mailer)//构造函数中获得my_mailer服务
    {
        $this->mailer = $mailer;
    }
 
    // ...
}
```

以上代码并没有用到服务容器，如下代码所示，在某个控制器中，你可以很容易的创建一个NewsletterManager服务对象：

```php
use Acme\HelloBundle\Newsletter\NewsletterManager;
 
// ...
 
public function sendNewsletterAction()
{
    $mailer = $this->get('my_mailer');
    $newsletter = new NewsletterManager($mailer);
    // ...
}
```

还算不错吧，但是一旦你在之后发现NewsletterManager类的构造函数需要更多的参数怎么办？难道你想要重构你的代码或者重命名类？这两种假设都不得不让你要跳回去所有用到NewsletterManager的地方去修改它。

然而，服务容器为你解决了后顾之忧：

```yaml
# src/Acme/HelloBundle/Resources/config/services.yml
parameters:
    # ...
    newsletter_manager.class: Acme\HelloBundle\Newsletter\NewsletterManager
 
services:
    my_mailer:
        # ...
    newsletter_manager:
        class:     "%newsletter_manager.class%"
        arguments: ["@my_mailer"]
```

通过如上配置，服务器容器将会在NewsletterManager对象每次实例化的时候自动为其__注入mailer服务__。

使用表达式：
例如，你有一个第三方服务，叫做mailer_configuration,其拥有一个方法getMailerMethod()方法,该方法将会返回一个字符串，该字符串可作为my_mailer服务创建时的“sendmail”参数,如下所示：

```yaml
# app/config/config.yml
services:
    my_mailer:
        class:        Acme\HelloBundle\Mailer
        arguments:    [sendmail]
```

现在，我们就可以通过mailer_configuration提供的getMailerMethod方法来为my_mailer服务的创建提供参数：

```yaml
# app/config/config.yml
services:
    my_mailer:
        class:        Acme\HelloBundle\Mailer
        arguments:    ["@=service('mailer_configuration').getMailerMethod()"]
```

##通过setter方法注入对某个服务的依赖（假设该依赖不是必须的，那么这会是一个好方法）：
 
```php
namespace Acme\HelloBundle\Newsletter;
 
use Acme\HelloBundle\Mailer;
 
class NewsletterManager
{
    protected $mailer;
 
    public function setMailer(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }
 
    // ...
}
```

记得更新配置文件：

```yaml
# src/Acme/HelloBundle/Resources/config/services.yml
services:
    my_mailer:
        # ...

    newsletter_manager:
        class:     Acme\HelloBundle\Newsletter\NewsletterManager
        calls:
            - [setMailer, ["@my_mailer"]]
```

```php
namespace Acme\HelloBundle\Newsletter;
 
use Acme\HelloBundle\Mailer;
 
class NewsletterManager
{
    protected $mailer;
 
    public function setMailer(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }
 
    // ...
}
```

加入注入是可选的，可以这样配置：

```yaml
# src/Acme/HelloBundle/Resources/config/services.yml
parameters:
    # ...
 
services:
    newsletter_manager:
        class:     "%newsletter_manager.class%"
        arguments: ["@?my_mailer"]
```

记得更新构造函数：

```php
public function __construct(Mailer $mailer = null)
{
    // ...
}
```

##Symfony核心以及第三方Bundle的服务：
例如，你可以获取session服务：

```php
public function indexAction($bar)
{
    $session = $this->get('session');
    $session->set('foo', $bar);
 
    // ...
}
```

在Symfony2中，你将经常使用Symfoy或第三方bundles提供的服务来执行任务，比如渲染模板的templating， 发送邮件的mailer访问请求信息的request等。如下例所示：


```php
namespace Acme\HelloBundle\Newsletter;
 
use Symfony\Component\Templating\EngineInterface;
 
class NewsletterManager
{
    protected $mailer;
 
    protected $templating;
 
    public function __construct(
        \Swift_Mailer $mailer,
        EngineInterface $templating
    ) {
        $this->mailer = $mailer;
        $this->templating = $templating;
    }
 
    // ...
}
```

配置文件如下：

```yaml
services:
    newsletter_manager:
        class:     "%newsletter_manager.class%"
        arguments: ["@mailer", "@templating"]
```
如上，newsletter_manage服务也能够使用mailer服务，templating服务了。

##标签（Tags）：
通过标签来指明某项服务的目的：

```yaml
services:
    foo.twig.extension:
        class: Acme\HelloBundle\Extension\FooExtension
        tags:
            -  { name: twig.extension }
```

这里的__twig.extension__标签就是一个专用标签，是TwigBundle在配置时使用的。通过给服务标注这个twig.extension标签，bundle就会知道 foo.twig.extension 服务应该被注册为一个Twig的扩展。换句话说，Twig会查找所有标记为twig.extension的服务并自动把它们注册为扩展。