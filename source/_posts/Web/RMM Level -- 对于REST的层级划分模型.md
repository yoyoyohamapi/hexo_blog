title: RMM Level -- 对于REST的层级划分模型
tags: REST
categories: Web开发
-------
##Richardson Maturity Model

原文链接：[http://martinfowler.com/articles/richardsonMaturityModel.html](http://martinfowler.com/articles/richardsonMaturityModel.html)

该模型将REST划作了由低到高四个等级，等级越高，RESTful就越成熟，但是，熟透了东西不一定好，甚至可能烂了，所以，项目中对于RESTful层级的选择要灵活把控。先看模型图：

![RMM Model](http://7pulhb.com1.z0.glb.clouddn.com/web-REST层次.png)

******************************************************************************************
###Level 0:The swarmp of POX(Plain old XML)

该等级的REST有如下特点:

1. HTTP仅作为一个通信隧道（即HTTP只关注通信消息，而不关注客户端及服务器间的行为）

2. 采用远程调用协议（Remote Procedure Call Protocol）：即客户端想要执行某一任务，或者说向服务器请求某一服务，只需发送相关消息（执行某一句柄），而不用关心底层实现。

3. 提供一个调用接口给客户端。

考虑一个例子：某病人（Client）想要预约某位医生为其诊断，那么其必然经历：

1. 病人问询医院（Server）该医生那天是否有空。

2. 医院告知病人该医生当天的空闲时间。

3. 病人执行预约

在Level 0中，该过程如下图所示：

![Level 0](http://7pulhb.com1.z0.glb.clouddn.com/web-level0.png)

注意到绿色的端点（appointmentService），这是医院（Server）为了病人（Client）能够远程咨询（远程调用）而提供的一个调用接口，以下是该过程的伪代码描述,其中请求及相应都用XML描述。

1. 病人想知道2010年1月4号时，jones医生啥时候有空：

```
POST /appointmentService HTTP/1.1
[various other headers]
```

```xml
<openSlotRequest date = "2010-01-04" doctor = "mjones"/>
```

2. 医院告诉病人jones医生当日的闲暇时间：

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<openSlotList>
  <slot start = "1400" end = "1450">
    <doctor id = "mjones"/>
  </slot>
  <slot start = "1600" end = "1650">
    <doctor id = "mjones"/>
  </slot>
</openSlotList>
```

3. 病人选择当中一段时间预约jones医生

```
POST /appointmentService HTTP/1.1
[various other headers]
```

```xml
<appointmentRequest>
  <slot doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
</appointmentRequest>
```

4.查看预约结果

如果预约成功：

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<appointment>
  <slot doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
</appointment>
```

否则：

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<appointmentRequestFailure>
  <slot doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
  <reason>Slot not available</reason>
</appointmentRequestFailure>
```

注意到，这里服务器并没有通过状态码来反映预约成功与否，而通过回调了冗长的错误信息。

**************************************************
###Level 1：Resources

该层的特点有：

1. 通过URI来定位资源，实现资源独立性

2. 采用“面向对象”的通信方式

相比于Level 0，这层更加成熟的地方是客户端需要标明__“我需要什么？”__，考虑上面的例子，其新的模型如下：

![Level 1](http://7pulhb.com1.z0.glb.clouddn.com/web-level1.png)

可以看到，每个资源都有各自调用接口（如医生资源，空闲表资源），该模型的病人预约过程如下：

1. 客户决定预约日期

```
POST /doctors/mjones HTTP/1.1
[various other headers]
```

```xml
<openSlotRequest date = "2010-01-04"/>
```

2. 医院据此返回当日空闲表

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<openSlotList>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650"/>
</openSlotList>
```

3. 病人请求获得id为1234的空闲服务（slot）

```
POST /slots/1234 HTTP/1.1
[various other headers]
```

```xml
<appointmentRequest>
  <patient id = "jsmith"/>
</appointmentRequest>
```

4. 医院返回该slot资源给病人

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<appointment>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
</appointment>
```

***************************************************************************
###Level 2:HTTP Verbs

顾名思义，Level 2追加了HTTP动作来指明我们对于资源要做何种操作，如此，客户端的请求就能完整的表述为__“我需要对XX（资源）做XX（行为）”__,该层级是当前使用最为广泛地REST层级，通常定义如下四个HTTP动作：

1. __GET__----》一般性获得资源，并不改变资源，所以这种操作相对安全

2. __POST__---》通常为创建资源操作

3. __PUT__----》通常为更新资源操作

4. __DELETE__-》删除资源操作

同时，服务端不再通过错误消息（当然，某些系统也会封装错误消息，给予客户友善提示）来告诉客户端执行状态，而是通过返回HTTP状态字来告知客户端请求执行结果。

上例在该层级下的模型如下：

![Level 2](http://7pulhb.com1.z0.glb.clouddn.com/web-level2.png)

任务过程如下：

1. 客户想要获得10年1月4好的医生空闲表

```
GET /doctors/mjones/slots?date=20100104&status=open HTTP/1.1
Host: royalhope.nhs.uk
```

2. 医院返回空闲表

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<openSlotList>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650"/>
</openSlotList>
```

3. 客户执行预约：

```
POST /slots/1234 HTTP/1.1
[various other headers]
```

```xml
<appointmentRequest>
  <patient id = "jsmith"/>
</appointmentRequest>
```

4. 医院告知预约结果：

若成功：

```
HTTP/1.1 201 Created
Location: slots/1234/appointment
[various headers]
```

```xml
<appointment>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
</appointment>
```

若失败：

```
HTTP/1.1 409 Conflict
[various headers]
```

```xml
<openSlotList>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650"/>
</openSlotList>
```

注意其返回状态字的变化，可以和Level 0作对比。

**************************************************************
###Level 3:Hypermedia Controls

首先要知道HATEOAS (Hypertext As The Engine Of Application State)：这种策略解决了我们如何从得到的资源中顺带知晓下一步应当如何进行？

因为要服务端的响应要封装“下一步如何做”，故而上例的模型变为：

![Level 3](http://7pulhb.com1.z0.glb.clouddn.com/web-level3.png)

任务过程如下:

1. 病人请求2010年1月4号的空闲表

```
GET /doctors/mjones/slots?date=20100104&status=open HTTP/1.1
Host: royalhope.nhs.uk
```

2. 医院返回空闲表，__并且附加了下一步操作__，其中，rel用于描述客户端要完成什么行为，uri用于定位该行为需要访问的资源。

```
HTTP/1.1 200 OK
[various headers]
```

```xml
<openSlotList>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450">
     <link rel = "/linkrels/slot/book" 
           uri = "/slots/1234"/>
  </slot>
  <slot id = "5678" doctor = "mjones" start = "1600" end = "1650">
     <link rel = "/linkrels/slot/book" 
           uri = "/slots/5678"/>
  </slot>
</openSlotList>
```

3. 病人执行预约：

```
POST /slots/1234 HTTP/1.1
[various other headers]
```

```xml
<appointmentRequest>
  <patient id = "jsmith"/>
</appointmentRequest>
```

4. 医院回调预约结果，__并告知了客户所有预约完以后可以进行的动作（包括取消，联系，帮助等等）__：

```
HTTP/1.1 201 Created
Location: http://royalhope.nhs.uk/slots/1234/appointment
[various headers]
```

```xml
<appointment>
  <slot id = "1234" doctor = "mjones" start = "1400" end = "1450"/>
  <patient id = "jsmith"/>
  <link rel = "/linkrels/appointment/cancel"
        uri = "/slots/1234/appointment"/>
  <link rel = "/linkrels/appointment/addTest"
        uri = "/slots/1234/appointment/tests"/>
  <link rel = "self"
        uri = "/slots/1234/appointment"/>
  <link rel = "/linkrels/appointment/changeTime"
        uri = "/doctors/mjones/slots?date=20100104@status=open"/>
  <link rel = "/linkrels/appointment/updateContactInfo"
        uri = "/patients/jsmith/contactInfo"/>
  <link rel = "/linkrels/help"
        uri = "/help/appointment"/>
</appointment>
```

显然，Level 3更加考虑周全，但是这就类似与我们拨打10010等查询电话，有时候用户对接下来的动作心里有杆秤，并不需要服务端告知所有能够进行的“下一步”，因而，这种层级在某些时候倒回变成一种累赘。