
## 服务消费方设计
### 1，连接管理

- 连接池
- 阻塞在哪里？详见4（超时处理）
- dubbo一条长连接就可以处理很多请求
   - 一般情况http，一个req对应一个res
   - 如果是一条长连接的话,不同线程请求 通过一个tcp通道 进行交互, 就需要有sessionId为key的存储. 比如sessionid->completeFuture
### 2，负载均衡

- 轮询，随机，取模，带权重，一致性hash（环）
- 权重设计
   - 数据结构：数组 根据权重随机填充，如 数组长度为10，权重为2/10，填2个1，8个0，随机打乱
   - **这个数组存在消费端，消费端启动的时候根据服务提供方的权重初始化好**
   - **调用过程，按顺序访问数组，这样即可达到权重访问的目的（因为数组是按权重分配的，顺序是随机打乱的）**
   - 为什么用顺序访问数组，不用生成随机值index，访问数组呢？
      - 这是一个优化，rpc server是被频繁调用的，防止频繁的调用导致频繁计算随机数。
### 3，请求路由
### 4，超时处理

- 关于工作线程阻塞位置
   - 整体流程：
      - 工作主线程注册windowData(包括sessionid,event) - 主线程send(放入队列) - 主线程执行完毕
      - 另线程run取队列,真正发送（有一个批量发送的优化点） - receive（阻塞）
         - receive阻塞具体：
            - 通过sessionId取windowData 
            -  windowData里event wait (本质是countDownlatch) 
            - decode方法里 当有网络数据返回时，解析出sessionid, 拿出windowData,在通过event解除阻塞，再把返回包数据 set进windowData中。
      - 阻塞本质和另一种 completeFuture一样。

## rpc服务端设计
### 1，队列线程池

- 队列线程池模型
   - 将不同请求放入不同队列，分配不同的线程池，资源隔离
   - 多队列-单线程
      - 无锁化
      - 对于某个请求时间长的情况，处理最差
   - 单队列-多线程
      - 有锁
      - 对于慢请求，比较友好
   - 压测
      - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641142314628-50334113-38d5-453e-83a3-7aab30cea665.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=320&id=u88262410&margin=%5Bobject%20Object%5D&name=image.png&originHeight=640&originWidth=1228&originalType=binary&ratio=1&rotation=0&showTitle=false&size=499830&status=done&style=none&taskId=u1cb420d8-ddf7-4d7f-95fb-fd4fa835d53&title=&width=614)
      - 5w qps以下，时间基本一致。所以我们选用多线程处理一个队列的模型。
### 2，超时丢弃
#### 服务端队列模型图
![](https://cdn.nlark.com/yuque/0/2021/jpeg/22881482/1640657474380-571751d7-3fcd-4e9f-95d5-3c59ce2bb3c6.jpeg)
如果服务端没有设置超时丢弃的功能，那么对于几个慢请求(比如某个http接口)可能会导致队列后续的任务都超时。
_**有超时丢弃功能之后，会保证服务还能正常处理一些任务，但是归根到底是服务处理能力不足或者某个阻塞请求的原因，这个是最终需要解决的。**_
### 3，优雅关闭
#### 优雅关闭的两种方案：
##### 服务端

1. 返回数据中带关闭信息
   - java中`Signal`类，初始化监听kill12 
   - 修改服务端状态为reboot 
   - 通知客户端
      - 一般的请求链路为 request filters - do业务 - response filters
      - 放在request filters里面做，返回信息，让客户端重新发到其他节点
      - 放在response filters里面做 ，返回这次请求结果，再加上rebot信息
      - 两种思路，是一个需要多端配合的事情
2. ~~专门的协议来处理这个 ，不细写~~
##### 客户端优雅关闭实现思路
![](https://cdn.nlark.com/yuque/0/2021/jpeg/22881482/1640573729936-9c9bf19c-6788-4213-8da2-af47500fb345.jpeg)

1. 根据返回包信息，设置该server状态为reboot，向上层抛出异常
1. 服务端重启完成，客户端需要知道，所以需要有一个**节点探活**
### 4,过载保护

- 对超过limitSize的task，实行拒绝策略

### 5.转转rpc思考

- `[https://mp.weixin.qq.com/s?__biz=Mzg3MzYxMjkyMg==&mid=2247483923&idx=1&sn=3e58e437212a9675df169e509b5b3f93&chksm=cedc1106f9ab98104a459ac52719d095fd33f1a8378dbe86212870f5824be82396b38b17baab&mpshare=1&scene=1&srcid=0310biJWAx3jnbfphbnHkD5f&sharer_sharetime=1646877117390&sharer_shareid=d7117e118228e78a9754af9505d4ed50&version=4.0.0.70098&platform=mac#rd](https://mp.weixin.qq.com/s?__biz=Mzg3MzYxMjkyMg==&mid=2247483923&idx=1&sn=3e58e437212a9675df169e509b5b3f93&chksm=cedc1106f9ab98104a459ac52719d095fd33f1a8378dbe86212870f5824be82396b38b17baab&mpshare=1&scene=1&srcid=0310biJWAx3jnbfphbnHkD5f&sharer_sharetime=1646877117390&sharer_shareid=d7117e118228e78a9754af9505d4ed50&version=4.0.0.70098&platform=mac#rd)`
- 连接的建立时机
   - 流程是创建代理类-注册中心发现节点-建立连接，那么连接是启动服务的时候就创建好，还是延迟等首次请求到来的时候创建？
   - 启动立即创建可以减少初次请求耗时
   - 延迟创建目的在于解决循环依赖的问题，A服务需要发现B, B服务也需要发现A,那么如果没有配置延迟创建连接的话，A,B启动都因为没有对方而报错。
   - 延迟创建连接有一个实用性是：在测试单个API调用的时候，在这个api请求到来，只创建这两个点的连接，（不像启动创建，需要创建所有依赖的服务连接）这样会提高启动测试效率
- 需要创建多少连接
   - 参见上面的连接管理内容
- 连接保活 
   - tcp层面的连接保活暂不考虑，时间长(小时级别)
   - 应用层面保活设计
      - 以使用netty通信为例, 通信层图如下
      - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1646894428487-793dcad2-0e19-4bb8-87fd-80c2bbe638a9.png#clientId=u3608fd4b-df31-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=182&id=u97f46f04&margin=%5Bobject%20Object%5D&name=image.png&originHeight=936&originWidth=1662&originalType=binary&ratio=1&rotation=0&showTitle=false&size=109919&status=done&style=none&taskId=ud799689a-a042-48c9-81dd-de3be843d9a&title=&width=323)
      - 保护设计流程如下
         - IdieStateHandler只负责一个事情：判断是否为idie状态，是就发布idieStateEvent
         - BizHandler 在 channel read的时候，设置lastRead属性为当前时间。在收到idieStateEvent的时候判断是否idieTimeout，处理连接是否需要关闭的逻辑。连接不健康的情况下，不代表完全收不到连接，可能只是连接速度慢，也可能丢了几次连接，所以还是可以收到idieStateEvent的，这时候超时关闭即可。
- 优雅关闭
   - 跟上面的流程基本一致
   - netty模型下，首先关闭boss获取连接的功能，服务端把关闭消息放到response，客户端收到response之后，不会再向该服务发送连接。 同时客户端开启轮训，检测剩余没有返回的请求数量，如果是0，还不能关闭，因为客户端收到关闭消息的时候，可能有部分线程刚刚负载均衡完毕，发送请求过去了，需要给这些线程留一定的时间。等到未返回请求=0&&留取的时间到达，则关闭连接。 服务端去检查连接，如果都关闭了，服务端则开始关机。

## 
## 高级功能
### 熔断

- 如返回商品 需要调用商品，浏览计数，用户三个服务，计数服务不可用，这一次请求就为失败
- 单一服务不可用，线程打满，可能会引发雪崩
- 单一服务不可用，触发熔断，不调用该服务
### 降级

- 熔断之后的处理方案
### 动态权重

- 关于权重 上面设计的负载均衡权重方面是❌不可用 不可落地的。因为我们设计的方案 权重数组存储在消费端。如果权重想改变，那么需要通知到每一个消费端团队。显然是不现实的，更不用说动态权重了。
- 需要借助第三方，例如公司的管理平台，注册中心什么的。
- **落地场景：管理平台可视化配置-下发到注册中心-服务发现的时候 也把权重拉取。**
### 限流

- 上面设计的过载保护，limitSize存在单个server，比如商品服务给消费者暴露，商品服务可能有10台机器组成集群，10个机器是一个整体，限流也是限制10个机器输出的总量，做配置的时候不能让用户去计算10台机器怎么分配的，也需要借助管理平台，不能单靠rpc。
- 管理平台，不关心D集群每台机器的流量，关心整体，D集群对a限制了多少流量，对b限制了多少流量。

## dubbo扩展性
#### dubbo spi

- 可扩展点 @spi
- META-INF/dubbo/org.apache.dubbo.rpc.cluster.LoadBalance
- ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641146388678-442059e0-d770-4b13-80d1-a885d687f4d1.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=85&id=u3034be91&margin=%5Bobject%20Object%5D&name=image.png&originHeight=170&originWidth=1232&originalType=binary&ratio=1&rotation=0&showTitle=false&size=282728&status=done&style=none&taskId=ubf28f31e-fff6-4b21-933f-03522e0ffd7&title=&width=616)
#### dubbo扩展点特性

- 扩展点自动包装
   - CWapper(BWapper(AWapper(instance)))
- 扩展点自动装配
   - 扩展点实现类的属性如果是其他扩展类型，自动注入依赖
- 扩展点自适应
   - @Adaptive注解在方法上
   - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641146785882-1fe9490c-caac-4c56-a4e8-87b83f5c7df9.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=233&id=u2789b2ee&margin=%5Bobject%20Object%5D&name=image.png&originHeight=466&originWidth=866&originalType=binary&ratio=1&rotation=0&showTitle=false&size=235746&status=done&style=none&taskId=u3c4a96c5-24f3-4e8f-9602-ea481524b4e&title=&width=433)
   - select在调用的时候，根据url的loadbalance参数决定使用哪个扩展类
- 扩展点自激活
   - @Activate的类
   - 对于集合类扩展点，例如Filter，每个Filter实现不同的功能，需要多个同时激活，可以用自动激活来简化配置 
   - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641227817424-97eb42ca-585f-4988-b7d9-b8c830518a5d.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=219&id=uf203cc97&margin=%5Bobject%20Object%5D&name=image.png&originHeight=438&originWidth=1296&originalType=binary&ratio=1&rotation=0&showTitle=false&size=399505&status=done&style=none&taskId=u4b44fb9d-d79e-4699-95ed-342058cb4e0&title=&width=648)
#### 源码

- 开始调用
   - _ExtensionLoader_.getExtensionLoader(_LoadBalance_.class).getExtension(_Constants_.**_DEFAULT_LOADBALANCE_**);
   - _ExtensionLoader_.getExtensionLoader(_Protocol_.class).getExtension(_DubboProtocol_.**_NAME_**);



- _ExtensionLoader两个主要缓存_
   - _class -> ExtensionLoader 一个可扩展点对应一个loader_
   - _class -> Object 类型to实例_
- 调用过程
1. getExtension , getDefaultExtension
      - _ConcurrentMap_<_String_, _Holder_<_Object_>> cachedInstances 缓存通过name(spi配置文件的name)找实例
      - getExtensionClasses - 把扩展类class加载进内存才可以实例化
         - cacheClasses缓存，name（配置name） -> class （_Map_<_String_, _Class_<?>> extensionClasses）
         - loadExtensionClasses
            - 获取默认值 --- @SPI(_RandomLoadBalance_.**_NAME_**) ---赋值给xxx属性
            - 多个目录的loadFile，给_Map_<_String_, _Class_<?>> extensionClasses填充，返回
               - loadFile里面干了什么
                  - 读取配置文件-一行一行解析
                  - class.forName(_String name(com.miaozhem.cc.cc类全限定名)_, boolean _initialial, ClassLoader load_)加载类，并返回该class
                  - Adaptive：判断class类上是否@Adaptive注解， cachedAdaptiveClass, 全局只有一个
                  - Wrapper:  判断是否有参数为 type的构造函数，有说明是wrapper类，缓存cachedWrapperClasses(set集合)
                  - Activate:  cachedActivates（name->activate注解），填充extensionClasses
            - 至此，有了如下缓存内容：
               - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641150972284-9e5081de-808e-4ad4-828c-40edc8eeda23.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=155&id=udd2606b2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=310&originWidth=1754&originalType=binary&ratio=1&rotation=0&showTitle=false&size=121930&status=done&style=none&taskId=u27519fba-980a-442a-bf6a-0a4d9069723&title=&width=877)
      - _ConcurrentMap_<_Class_<?>, _Object_> **_EXTENSION_INSTANCES通过上面拿到的的 class找缓存有没有实例_**
      - injectExtension 扩展点自动装配，后面细说
      - 循环上面组装的 wrapperClasses set集合，包装`CWapper(BWapper(AWapper(instance)))`
2. getAdaptiveExtension, 链路未知，单看从该方法起步的链路
      - getAdaptiveExtension - createAdaptiveExtension - getAdaptiveExtensionClass - createAdaptiveExtensionClass
      - 结果是生成如下的代理类
         - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641226825996-99538ddc-2803-4d95-8865-a968e507c83a.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=415&id=u6a220ab2&margin=%5Bobject%20Object%5D&name=image.png&originHeight=830&originWidth=1254&originalType=binary&ratio=1&rotation=0&showTitle=false&size=753311&status=done&style=none&taskId=u87b7304f-ea51-4f9b-a51f-3e06b7e83c5&title=&width=627)
         - 代理类表明拿 url的参数，决定使用哪个具体的扩展实现
         - 接口为：
            - ![image.png](https://cdn.nlark.com/yuque/0/2022/png/22881482/1641226855590-5e563bf9-7091-49eb-bd0a-f43db3bc2031.png#clientId=ue9bfe9d5-fb59-4&crop=0&crop=0&crop=1&crop=1&from=paste&height=210&id=uea951e61&margin=%5Bobject%20Object%5D&name=image.png&originHeight=420&originWidth=734&originalType=binary&ratio=1&rotation=0&showTitle=false&size=181389&status=done&style=none&taskId=uef20e188-738e-41b6-b034-8848be4a963&title=&width=367)
3. getActivateExtension
   - 还是上面的加载扩展类 ： getExtensionClasses
   - cachedActivates（name->activate注解）遍历
   - 根据activate注解的 group（比如限定提供者激活 或者 消费者激活） 和 value （url中包含该key）属性，如果符合条件，放到集合里面，集合进行排序，根据注解的 before，after，order属性
4.  ioc，扩展类属性是其他扩展类型的 注入。
   1. _AdaptiveExtensionFactory 全局唯一类上@Aadaptive_
   1. _AdaptiveExtensionFactory 的 _factories 一个spi创建的对象，一个spring的
   1. _AdaptiveExtensionFactory的_getExtension方法参数通过 type和name获取扩展对象
   1. injectExtension方法通过_AdaptiveExtensionFactory注入 ： 判断set方法，然后反射调用_
5. _关于ExtensionLoader_
   1. _构造函数的objectFactory生成的应该是AdaptiveExtensionFactory,但是上面分析2._getAdaptiveExtension的时候，生成的procal$Adaptivel类，这点问题遗留






















