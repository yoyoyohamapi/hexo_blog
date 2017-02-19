title: Symfony学习笔记（5）-- Doctrine（2）
tags: symfony
categories: Symfony学习笔记
---------

这节主要描述Doctrine中实体关系（Relation）的设计。

###创建实体
考虑商品Product属于某一分类Category，每个分类下先创建category(注意，id是可以自动创建的):

```
php app/console doctrine:generate:entity --entity="AcmeStoreBundle:Category" --fields="name:string(255)"
```

###关系映射
配置Category:

__Annotations__:

```php
// src/Acme/StoreBundle/Entity/Category.php
// ...
use Doctrine\Common\Collections\ArrayCollection;
class Category
{
 // ...
/**
 * @ORM\OneToMany(targetEntity="Product", mappedBy="category")//映射到Product中的category对象
 */
 protected $products;
public function __construct()//该方法必不可少
 {
 $this->products = new ArrayCollection();
 }
}
```

__YAML__:

```yaml
# src/Acme/StoreBundle/Resources/config/doctrine/Category.orm.yml
Acme\StoreBundle\Entity\Category:
     type: entity
     # ...
     oneToMany:
                products:
                          targetEntity: Product
                         mappedBy: category
      # don't forget to init the collection in the __construct() method of the entity
```

配置Product:

__Annotations__:

```php
// src/Acme/StoreBundle/Entity/Product.php
// ...
class Product
{
 // ...
/**
 * @ORM\ManyToOne(targetEntity="Category", inversedBy="products")
 * @ORM\JoinColumn(name="category_id", referencedColumnName="id")
 */
 protected $category;
}
```

__YAML__:

```yaml
# src/Acme/StoreBundle/Resources/config/doctrine/Product.orm.yml
Acme\StoreBundle\Entity\Product:
           type: entity
           # ...
           manyToOne:
                  category:
                           targetEntity: Category
                          inversedBy: products
                           joinColumn:
                          name: category_id
                          referencedColumnName: id
```

别忘了产生相应的getters和setters:

```
php app/console doctrine:generate:entities Acme
```

也别忘了其实我们的Category表尚未在数据库中建立:

```
php app/console doctrine:schema:update --force
``` 

![Doctrine关系映射](http://7pulhb.com1.z0.glb.clouddn.com/ip-对象关系映射.png)

如图，Product对象中保存有category对象，通过get和set方法获取及设置，而对应的product表中，仅存有category_id.

 

###保存及获取关联对象：

__保存关联对象__：

```php
// ...
use Acme\StoreBundle\Entity\Category;
use Acme\StoreBundle\Entity\Product;
use Symfony\Component\HttpFoundation\Response;
class DefaultController extends Controller
{
	public function createProductAction()
	{
		$category = new Category();
		$category->setName('Main Products');
		$product = new Product();
 		$product->setName('Foo');
 		$product->setPrice(19.99);
 		// relate this product to the category
 		$product->setCategory($category);
		$em = $this->getDoctrine()->getManager();
 		$em->persist($category);
 		$em->persist($product);
 		$em->flush();
		return new Response(
 			'Created product id: '.$product->getId()
 .' and category id: '.$category->getId()
 		);
 	}
}
```

__取得关联对象__：

```php
public function showAction($id)
{
 	$product = $this->getDoctrine()
 				->getRepository('AcmeStoreBundle:Product')
 				->find($id);
	$categoryName = $product->getCategory()->getName();
// ...
}
```

###延迟加载：（lazily load）

![延迟加载](http://7pulhb.com1.z0.glb.clouddn.com/ip-延迟加载.png)

注意，在第二步中，Doctrine并没有返回category对象，而当你真正调用了getCateory()方法后，Doctrine才真正返回了相应数据。

###在对应Repository中自写联合查询

```php
// src/Acme/StoreBundle/Entity/ProductRepository.php
public function findOneByIdJoinedToCategory($id)
{
	$query = $this->getEntityManager()
 				->createQuery('SELECT p, c FROM AcmeStoreBundle:Product p JOIN p.category c WHERE p.id = :id')
 				->setParameter('id', $id);
	try {
 		return $query->getSingleResult();
 	} catch (\Doctrine\ORM\NoResultException $e) {
 		return null;
 	}
}
```