title: Symfony学习笔记（4）-- Doctrine（1）
tags: symfony
categories: Symfony学习笔记
---------

[Doctrine](http://www.doctrine-project.org/)是一个第三方Bundle，其唯一目的就是在Symfony2中操纵数据库。

##一个小例子熟悉Doctrine(假设我们的数据库是MySql)

###在命令行创建一个小例子的Bundle

```
php app/console generate:bundle --namespace=Acme/StoreBundle
```

###配置数据库：
通过__parameters.yml__来确定一些数据库基本参数

###创建数据库

```
$ php app/console doctrine:database:create
```

>很重要！别忘了设置Mysql的编码格式

>在__my.cnf__中：

>```
[mysqld]
collation-server = utf8_general_ci
character-set-server = utf8
```

###创建数据库表实体

__法一：手动书写代码__

```php
// src/Acme/StoreBundle/Entity/Product.php
namespace Acme\StoreBundle\Entity;
class Product
{
	protected $name;
	protected $price;
	protected $description;
}
```

__法二：控制台交互式创建__

```
php app/console doctrine:generate:entity
```

###添加映射信息，如下图所示，Doctrine完成了PHP中实体对象到数据库表的映射

![Doctrine映射](http://7pulhb.com1.z0.glb.clouddn.com/ip-Doctrine映射.png)


我们可以通过Annotations完成PHP class到数据表的映射，当然也可以使用YAML或者XML，但是三者决不能混用。

__Annotations__：

```php
// src/Acme/StoreBundle/Entity/Product.php
namespace Acme\StoreBundle\Entity;
use Doctrine\ORM\Mapping as ORM;
/**
 * @ORM\Entity
 * @ORM\Table(name="product")//表名可以省略，可以通过类名进行自动判断
 */
class Product
{
 /**
 * @ORM\Column(type="integer")
 * @ORM\Id
 * @ORM\GeneratedValue(strategy="AUTO")
 */
 protected $id;
/**
 * @ORM\Column(type="string", length=100)
 */
 protected $name;
/**
 * @ORM\Column(type="decimal", scale=2)
 */
 protected $price;
/**
 * @ORM\Column(type="text")
 */
 protected $description;
}
```

__YAML__：

```yaml
# src/Acme/StoreBundle/Resources/config/doctrine/Product.orm.yml
Acme\StoreBundle\Entity\Product:
    type: entity
    table: product
    id:
        id:
            type: integer
            generator: { strategy: AUTO }
    fields:
        name:
            type: string
            length: 100
        price:
            type: decimal
            scale: 2
        description:
            type: text
```
>注意：类名尽量不要用sql中的保留字，如 Group等。

###产生__Getters__和__Setters__方法：

```
php app/console doctrine:generate:entities Acme/StoreBundle/Entity/Product
```

###创建product对象所映射的数据库表

```
php app/console doctrine:schema:update --force
```

根据实体类的变化Doctrine自动产生相应SQL语句完成数据视图的更新。


###将对象持久化到数据库

```php
// src/Acme/StoreBundle/Controller/DefaultController.php
// ...
use Acme\StoreBundle\Entity\Product;
use Symfony\Component\HttpFoundation\Response;
public function createAction()
{
	$product = new Product();
 	$product->setName('A Foo Bar');
 	$product->setPrice('19.99');
 	$product->setDescription('Lorem ipsum dolor');
	$em = $this->getDoctrine()->getManager();
 	$em->persist($product);
 	$em->flush();//flush以后才真正刷新数据库
	return new Response('Created product id '.$product->getId());
}
```

这里有必要提到一下__flush()__这个方法，他会自动衡量持久化后数据的变更情况，并产生最适当的查询语句。例如，如果你持久化总计100个product对象，并在之后调用flush()方法，Doctrine将自动创建1个简单的预处理语句并在之后每条插入操作中重用该语句，这种足够快速高效的模式称为__“工作单元（Unit of Work）”__。

###从数据库中取得数据。

```php
public function showAction($id)
{
	$product = $this->getDoctrine()
 ->getRepository('AcmeStoreBundle:Product')
 ->find($id);
	if (!$product) {
 		throw $this->createNotFoundException(
 		'No product found for id '.$id
 );
 }
// ... do something, like pass the $product object into a template
}
```

###什么是repository？
可以理解为一个特殊的PHP类，其唯一的工作就是取得其对应实体相关数据。如下可以看到repository封装有很多方法来以不同的形式取得数据，实现了模型层的一些功能。

```php
// query by the primary key (usually "id")
$product = $repository->find($id);
// dynamic method names to find based on a column value
$product = $repository->findOneById($id);
$product = $repository->findOneByName('foo');
// find *all* products
$products = $repository->findAll();
// find a group of products based on an arbitrary column value
$products = $repository->findByPrice(19.99);
```

__利用findBy及findOneBy来控制含有多个限制的对象获取__：

```php
// query for one product matching by name and price
$product = $repository->findOneBy(
	array('name' => 'foo', 'price' => 19.99)
);
// query for all products matching the name, ordered by price
$products = $repository->findBy(
	array('name' => 'foo'),
	array('price' => 'ASC')
);
```

>注意：可以Symfony中toolbar查看了有多少查询被执行。

###更新与删除，都是通过flush刷新

__更新__：

```php
public function updateAction($id)
{
	$em = $this->getDoctrine()->getManager();
	$product = $em->getRepository('AcmeStoreBundle:Product')->find($id);
if (!$product) {
		throw $this->createNotFoundException(
		'No product found for id '.$id
 	);
}
	$product->setName('New product name!');
	$em->flush();
	return $this->redirect($this->generateUrl('homepage'));
}
```

__删除__：

```php
	$em->remove($product);
	$em->flush();
```

##通过QueryBuilder来建立查询语句

```php
$repository = $this->getDoctrine()
 ->getRepository('AcmeStoreBundle:Product');
$query = $repository->createQueryBuilder('p')
 ->where('p.price > :price')//设置:price为占位符(placeholder)，防止SQL注入
 ->setParameter('price', '19.99')//设置占位符的值
 ->orderBy('p.price', 'ASC')
 ->getQuery();
$products = $query->getResult();
```

###通过DQL建立查询，这样书写更像是在写SQL语句，对于复杂的查询非常有用。

```php
$em = $this->getDoctrine()->getManager();
$query = $em->createQuery(
 'SELECT p
 FROM AcmeStoreBundle:Product p //注意，表名，字段名的书写，以及SQL防注入
 WHERE p.price > :price
 ORDER BY p.price ASC'
)->setParameter('price', '19.99');
$products = $query->getResult();
```

##封装

为了更好地代码分离度和代码重用性，更建议将一些查询封装到实体对应的Repository中，仍是可以通过Annotations及YAML完成绑定实体及Repository。

__Annotations__:

```php
// src/Acme/StoreBundle/Entity/Product.php
namespace Acme\StoreBundle\Entity;
use Doctrine\ORM\Mapping as ORM;
/**
 * @ORM\Entity(repositoryClass="Acme\StoreBundle\Entity\ProductRepository")
 */
class Product
{
 //...
}
```

__YAML__:

```yaml
# src/Acme/StoreBundle/Resources/config/doctrine/Product.orm.yml
Acme\StoreBundle\Entity\Product:
    type: entity
    repositoryClass: Acme\StoreBundle\Entity\ProductRepository
 # ...
```

__通过控制台自动产生repository类__：

```
php app/console doctrine:generate:entities Acme
```

__考虑在repository类中添加自己的方法__：

```php
// src/Acme/StoreBundle/Entity/ProductRepository.php
namespace Acme\StoreBundle\Entity;
use Doctrine\ORM\EntityRepository;
class ProductRepository extends EntityRepository
{
	public function findAllOrderedByName()
	{
 		return $this->getEntityManager()
 				->createQuery('SELECT p FROM AcmeStoreBundle:Product p ORDER BY p.name ASC')
 				->getResult();
 	}
}
```