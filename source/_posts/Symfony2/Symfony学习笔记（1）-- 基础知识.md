title: Symfony学习笔记（1）-- 基础知识
tags: symfony
categories: Symfony学习笔记
--------

##什么是Bundle？

Bundle在Symfony中大量使用，是Symfony的核心。Bundle是一系列文件集合（PHP，JS，CSS，images等等），其实现了一个完整的功能或者模块（如博客模块，论坛模块），Bundle最重要的特点就是能被各个开发者所共享。

###注册Bundle

每个应用都是由在__AppKernel__类的__registerBundles()__方法里声明的代码包组成的。每一个代码包都包含了一个描述性的Bundle类：

```php
public function registerBundles()
{
    $bundles = array(
        new Symfony\Bundle\FrameworkBundle\FrameworkBundle(),
        new Symfony\Bundle\SecurityBundle\SecurityBundle(),
        new Symfony\Bundle\TwigBundle\TwigBundle(),
        new Symfony\Bundle\MonologBundle\MonologBundle(),
        new Symfony\Bundle\SwiftmailerBundle\SwiftmailerBundle(),
        new Symfony\Bundle\DoctrineBundle\DoctrineBundle(),
        new Symfony\Bundle\AsseticBundle\AsseticBundle(),
        new Sensio\Bundle\FrameworkExtraBundle\SensioFrameworkExtraBundle(),
        new JMS\SecurityExtraBundle\JMSSecurityExtraBundle(),
    );
 
    if (in_array($this->getEnvironment(), array('dev', 'test'))) {
        $bundles[] = new Acme\DemoBundle\AcmeDemoBundle();
        $bundles[] = new Symfony\Bundle\WebProfilerBundle\WebProfilerBundle();
        $bundles[] = new Sensio\Bundle\DistributionBundle\SensioDistributionBundle();
        $bundles[] = new Sensio\Bundle\GeneratorBundle\SensioGeneratorBundle();
    }
 
    return $bundles;
}
```

尤其注意__FrameworkBundle__，__DoctrineBundle__，__SwiftmailerBundle__，__AsseticBundle__等等Bundle，他们都是Symfony2的常用及核心bundle。

###配置Bundle
Bundle可以通过YAML，XML，或PHP代码格式的配置文件进行配置，下面是框架的默认配置：

```yaml
imports:
    - { resource: parameters.ini }
    - { resource: security.yml }
 
framework:
    secret:          "%secret%"
    charset:         UTF-8
    router:          { resource: "%kernel.root_dir%/config/routing.yml" }
    form:            true
    csrf_protection: true
    validation:      { enable_annotations: true }
    templating:      { engines: ['twig'] } #assets_version: SomeVersionScheme
    session:
        default_locale: "%locale%"
        auto_start:     true
 
# Twig Configuration
twig:
    debug:            "%kernel.debug%"
    strict_variables: "%kernel.debug%"
 
# Assetic Configuration
assetic:
    debug:          "%kernel.debug%"
    use_controller: false
    filters:
        cssrewrite: ~
        # closure:
        #     jar: "%kernel.root_dir%/java/compiler.jar"
        # yui_css:
        #     jar: "%kernel.root_dir%/java/yuicompressor-2.4.2.jar"
 
# Doctrine Configuration
doctrine:
    dbal:
        driver:   "%database_driver%"
        host:     "%database_host%"
        dbname:   "%database_name%"
        user:     "%database_user%"
        password: "%database_password%"
        charset:  UTF8
 
    orm:
        auto_generate_proxy_classes: "%kernel.debug%"
        auto_mapping: true
 
# Swiftmailer Configuration
swiftmailer:
    transport: "%mailer_transport%"
    host:      "%mailer_host%"
    username:  "%mailer_user%"
    password:  "%mailer_password%"
 
jms_security_extra:
    secure_controllers:  true
    secure_all_services: false
```

每一个类似framework的键值对应的都是某一个具体Bundle的配置。比如，__framework__配置的是__FrameworkBundle__，而__swiftmailer__配置的是__SwiftmailerBundle__。 每一个运行环境（environment）的默认配置都可以被覆盖。比如，dev环境所加载的config_dev.yml文件，即是对主配置（__config.yml__）的一个扩展，其启用了一些调试的工具：

```yaml
imports:
    - { resource: config.yml }
 
framework:
    router:   { resource: "%kernel.root_dir%/config/routing_dev.yml" }
    profiler: { only_exceptions: false }
 
web_profiler:
    toolbar: true
    intercept_redirects: false
 
monolog:
    handlers:
        main:
            type:  stream
            path:  "%kernel.logs_dir%/%kernel.environment%.log"
            level: debug
        firephp:
            type:  firephp
            level: info
 
assetic:
    use_controller: true
```

##Symfony的环境。

在Symfony2中，支持两种配置环境，Dev环境，即开发者环境（该环境相当强大，symfony为此提供了专用的toolbar来监测网站的基本信息），Prod环境，即生产环境，将其理解为产品，成品环境更好。

##Symfony目录结构

__app/__: 应用配置,缓存，日志，控制台命令等。目录下的__AppKernel__类是整个应用配置的入口，它存在于app/目录中。该类必须实现两个方法:

* __registerBundles()__:返回应用所需的所用代码包
* 
* __registerContainerConfiguration()__: 加载应用的配置文件
 
__src/__: 项目源码 

__vendor/__: 第三方代码 

__web/__: 网站根目录,也是网站的外部访问入口.包含了所用公共可访问的静态文件（如图片，样式表，JS脚本等），同时也包含了每一个前端控制器：

```php
// web/app.php
 require_once __DIR__.'/../app/bootstrap.php.cache';
 require_once __DIR__.'/../app/AppKernel.php';
 use Symfony\Component\HttpFoundation\Request;
 $kernel = new AppKernel('prod', false);
 $kernel->loadClassCache();
 $kernel->handle(Request::createFromGlobals())->send();
```

可以看到，框架的核心代码首先引用了__bootstrap.php.cache__文件，其任务是对框架的各组件进行预加载，并注册__autoloader__。 与其他的入口文件一样，app.php使用一个Kernel Class（AppKernel）来初始化整个程序。


##路由（URL地址解析）

可以将路由理解为前后台通信或者说切换的转发站，路由绝对是Symfony以及Symfony系框架（如Laravel）的最大特色和吸引人的地方，Symfony2中主要支持两种形式的路由书写规范，一种是yml形式的路由，一种是annotation形式的路由。

先来看__注解（annotation）__形式的路由，当我访问[http://localhost:8000/demo/hello/wxj](http://localhost:8000/demo/hello/wxj)的时候，业务逻辑就会交付Acme（__大方向__）中DemoBundle（__模块__）中DemoController（__控制器，逻辑__）中的hello方法，且该方法捕获到name为“wxj”。

```yaml
# src/Acme/DemoBundle/Resources/config/routing.yml

_demo:
    resource: "@AcmeDemoBundle/Controller/DemoController.php"
    type:     annotation
    prefix:   /demo
```

```php
// src/Acme/DemoBundle/Controller/DemoController.php
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Template;
 
class DemoController extends Controller
{
    /**
     * @Route("/hello/{name}", name="_demo_hello")
     * @Template()
     */
    public function helloAction($name)
    {
        return array('name' => $name);
    }
 
    // ...
```

再来看__yaml__形式的路由，更在直观，但是会造成某一Bundle的路由文件过长。当我访问[http://localhost:8000/test/wxj](http://localhost:8000/test/wxj)的时候，进入Acme中的DemoBundle中的Test控制器中的print方法，并捕获到name为“wxj”。

```yaml
_test:

 path:   /test/{name}

 defaults: {_controller:AcmeDemoBundle:Test:print}
```
