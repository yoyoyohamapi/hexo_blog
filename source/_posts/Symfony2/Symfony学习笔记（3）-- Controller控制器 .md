title: Symfony学习笔记（3）-- Controller控制器
tags: symfony
categories: Symfony学习笔记
----

##方便的代码书写
一旦你声明好控制器中的方法返回的模板对象，那么一切都交给Symfony吧（甚至你都没有声明返回的路径，Symfony也可以根据你的__Action__名称返回信息到相应模板（HTML，XML，JSON以及更多））！

```php
// src/Acme/DemoBundle/Controller/DemoController.php
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Template;
// ...
/**
* @Route("/hello/{name}", defaults={"_format"="xml"}, name="_demo_hello")
* @Template()
*/
public function helloAction($name)
{
return array('name' => $name);
}
```
如上面代码片段所示，，通过注解,你声明了返回的模板对象是XML格式，那么即便在helloAction中，即便你没有写全返回路径，即return $this->render('AcmeDemoBundle:Demo:hello.xml.twig', array('name' => $name,))，Symfony仍然可以找到你需要的返回路径（其实Symfony已然知道你的Bundle，Controller，Action，format等一系列信息，不过是再重写一下返回值而已）。



下面的xml文件很容易就收到了响应信息

```xml
<!-- src/Acme/DemoBundle/Resources/views/Demo/hello.xml.twig -->
<hello>
    <name>{{ name }}</name>
</hello>
```
 
##跳转与重定向：

如下代码所示，你可以完成一个跳转，并且该跳转所产生的URL将会是相当美观大方的。

```php
return $this->redirect($this->generateUrl('_demo_hello', array('name' => 'Lucas')));
```
并且你也可以到指定的Action，并向该Action传递参数：

```php
return $this->forward('AcmeDemoBundle:Hello:fancy', array('name' => $name, 'color' => 'green'))；
```

##获取请求信息

通过Symfony的Http基础组件__Request__，你能很轻易地获得请求信息。

```php
use Symfony\Component\HttpFoundation\Request;
public function indexAction(Request $request)
{
	$request->isXmlHttpRequest(); // is it an Ajax request?
	$request->getPreferredLanguage(array('en', 'fr'));
	$request->query->get('page');
	// get a $_GET parameter
	$request->request->get('page'); // get a $_POST parameter
}
```

在视图中，你同样能能够通过app.request来访问Request对象

```twig
{{ app.request.query.get('page') }}
{{ app.request.parameter('page') }}
```

##Symfony的Session：

通过__Request__的__getSession()__方法，能够获得站点的session对象。

```php
use Symfony\Component\HttpFoundation\Request;
public function indexAction(Request $request)
{
	$session = $this->request->getSession();
// store an attribute for reuse during a later user request
	$session->set('foo', 'bar');
// get the value of a session attribute
	$foo = $session->get('foo');
// use a default value if the attribute doesn't exist
	$foo = $session->get('foo', 'default_value');
```


值得注意的是，Symfony支持“闪存式”session，该session中储存的变量将在下一次请求之后自动删除，这对于一些不需要长期驻留session的回调信息该功能显得十分有用。

添加“闪存式”session:

```php 
// store a message for the very next request (in a controller)
$session->getFlashBag()->add('notice', 'Congratulations, your action succeeded!');
```

获得“闪存式”session:

```twig
{# display the flash message in the template #}
<div>{{ app.session.flashbag.get('notice') }}</div>
```

##缓存机制：
如下，在Symfony中，通过annotation注解来配置缓存

```php
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Template;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Cache;
/**
* @Route("/hello/{name}", name="_demo_hello")
* @Template()
* @Cache(maxage="86400")
*/
public function helloAction($name)
{
	return array('name' => $name);
}
```

在该代码片段中，helloAction方法返回的HTML页面将提示浏览器可以将其缓存一天（86400秒），当然，也可以通过redis或者memcached来实现缓存，使你的网站获得更高的性能。

##安全机制：

Symfony2标准版包含了能满足常规需要的安全设置：

```yaml
# app/config/security.yml
security:
    encoders:
        Symfony\Component\Security\Core\User\User: plaintext

    role_hierarchy:
        ROLE_ADMIN:       ROLE_USER
        ROLE_SUPER_ADMIN: [ROLE_USER, ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]

    providers:
        in_memory:
            users:
                user:  { password: userpass, roles: [ 'ROLE_USER' ] }
                admin: { password: adminpass, roles: [ 'ROLE_ADMIN' ] }

    firewalls:
        dev:
            pattern:  ^/(_(profiler|wdt)|css|images|js)/
            security: false

        login:
            pattern:  ^/demo/secured/login$
            security: false

        secured_area:
            pattern:    ^/demo/secured/
            form_login:
                check_path: /demo/secured/login_check
                login_path: /demo/secured/login
            logout:
                path:   /demo/secured/logout
                target: /demo/
```

这个配置使得用户需要先登录，才能访问以__/demo/secured/__开头的URL。配置里还定义了两个用户：__user__和__admin__。__admin__用户有一个__ROLE_ADMIN__的身份，这个身份包含了__ROLE_USER__（即角色的分布是层次结构）。

__Tip__

>为了方便阅读，例子里的密码都是明文的，但你在实际代码里应该运用hash或bcrypt算法来增强安全性。

由于有__“防火墙”（firewall）__的保护，访问[http://localhost/Symfony/web/app_dev.php/demo/secured/hello](http://localhost/Symfony/web/app_dev.php/demo/secured/hello)会自跳转到登录页面。

你还可以通过__@Secure__注解为控制器的某个动作增加用户必须具有指定角色的限制：

```php
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
use Sensio\Bundle\FrameworkExtraBundle\Configuration\Template;
use JMS\SecurityExtraBundle\Annotation\Secure;

/**
 * @Route("/hello/admin/{name}", name="_demo_secured_hello_admin")
 * @Secure(roles="ROLE_ADMIN")
 * @Template()
 */
public function helloAdminAction($name)
{
    return array('name' => $name);
}
```

如是，以user（不具备ROLE_ADMIN角色）到达helloAdminAction时，Symfony将返回一个HTTP__403(Forbidden)__状态码，即不允许当前用户访问。