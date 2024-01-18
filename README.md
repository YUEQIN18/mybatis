## 寻找入口

要想理解完整的工作流程，肯定要从mybatis原始`API`入手，下面是使用原始`API`使用mybatis的示例代码(`MyBatisTest.java`)：

```java
java
复制代码@Test
public void testSelect() throws IOException {
    String resource = "config/mybatis-config.xml";
    InputStream inputStream = Resources.getResourceAsStream(resource);
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

    SqlSession session = sqlSessionFactory.openSession(); // ExecutorType.BATCH
    try {
        // 通过sqlSession先获取到Mapper接口对象
        BlogMapper mapper = session.getMapper(BlogMapper.class);
        Blog blog = mapper.selectBlogById(1688);
        System.out.println(blog);
    } finally {
        session.close();
    }
}
```

可以明显看出来，入口代码就是`SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);`。



## 配置解析

mybatis有两种配置文件，一种是全局配置文件`mybatis-config.xml`，另一种就是各个具体的Mapper文件`xxxMapper.xml `。在上面的代码中，核心就是基于`mybatis-config.xml`配置文件构建出了`sqlSessionFactory`对象。跟进`SqlSessionFactoryBuilder`的`build()`方法： ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3088dfd17fad4893bd51fcd975d8048b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

上面代码的关键是调用了`parser`的`parse()`方法，它会返回一个`Configuration`类。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/650f1f33520940439290f40c6b25c5a7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

明显可以看出来，解析配置是从configuration根节点开始解析的。具体解析过程如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/920a41a99da542cb9a721d5dc85ce25b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

解析配置完整方法上所示，主要的解析过程包含属性、类型别名、插件、对象工厂、对象包装工厂、设置、环境、类型处理器、映射器等等。下面挑选最关键的部分具体讲解。

### 属性 `propertiesElement()`

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/264c2ff9745148daaf967b0f37a9a7ff~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

第一个是解析`<properties>`标签，读取我们引入的外部配置文件。解析的最终结果就是我们会把所有的配置信息放到名为`defaults`的`Properties`对象里面，最后把`XPathParser`和`Configuration`的`Properties`属性都设置成填充后的`Properties`对象。

### 类型别名 `typeAliasesElement()`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d208301e0f7f477a8db1368e1842c675~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

类型别名是解析`<typeAliases>`标签，它有两种定义方式，一种是直接定义一个类的别名，另一种是指定一个包，此时这个`package`下面所有的类的名字就会成为这个类全路径的别名。类的别名和类的关系，放在一个`TypeAliasRegistry`对象里面。

### 插件 `pluginElement()`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cdc7bea5e73143e0968f49a96dac9a7a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

插件是解析`<plugins>`标签。`<plugins>`标签里面只有`<plugin>`标签，`<plugin>`标签里面只有`<property>`标 签。标签解析完以后，会生成一个`Interceptor`对象，并且添加到`Configuration`的 `InterceptorChain`属性里面，它是一个`List`。

### 对象工厂 `objectFactoryElement()`、`objectWrapperFactoryElement()`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9fab0ae547d5451eb7bb33b5aac8bc67~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

这两个标签是用来实例化对象用的，分别解析的是`<objectFactory>`和`<objectWrapperFactory>`这两个标签，对应生成`ObjectFactory`、`ObjectWrapperFactory`对象，同样设置到`Configuration`的属性里面。

### 设置 `settingsElement()`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7edc855d2a1a4352aaf87ba5705ada4e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

设置是解析`<settings>`标签，首先将所有子标签全部转换成了`Properties`对象，然后再将相应的属性设置到`Configuration`中。

二级标签里面有很多的配置，比如二级缓存，延迟加载，自动生成主键这些。需要注意的是，**所有的默认值都是在这里赋值的**。如果我们不知道这个属性的值是什么，也可以到这一步来确认一下。 所有的值，都会赋值到`Configuration`的属性中。

### 环境 `environmentsElement()`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf880c62bb6e498bb0594b4452ba8211~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

一个`environment`就是对应一个数据源，所以在这里我们会根据配置的`<transactionManager>`创建一个事务工厂，根据`<dataSource>`标签创建一个数据源，最后把这两个对象设置成`Environment`对象的属性，放到 `Configuration`里面。

### 类型处理器 `typeHandlerElement()`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa4206bd3e5348818e4bfb82908a28e8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

跟`TypeAlias`一样，`TypeHandler`有两种配置方式，一种是单独配置一个类，一种是指定一个`package`。最后我们得到的是`JavaType`和`JdbcType`，以及用来做相互映射的`TypeHandler`，并将其保存在 `TypeHandlerRegistry`对象里面。

### 映射器 `mapperElement()`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f767f0165fb4c62bc75a05e0d167cb8~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5d590da4353449029f7e08d72a780289~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

映射器支持4种配置方式，包括类路径、绝对url路径、Java类名和自动扫描包方式。下面以自动扫描包为例，具体说明。查看`configuration.addMappers(mapperPackage)`实现： ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/55e2bf1289844b698db986aed91f78f0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/58d4577bad414f61a1c9f641dead8ba1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

查找指定包下所有的`mapperClass`，并注册。继续跟进`addMapper(mapperClass)`实现： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/57ddf19f40c64b1a839e6272e3db18f1~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

添加映射就是将`Mapper`类型和对应的`MapperProxyFactory`关联，放到一个`Map`容器中。并且在这里还会去解析`Mapper`接口方法上的注解。具体来说是创建了一个`MapperAnnotationBuilder`，我们点进去看一下 `parse()`方法。 ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0b1161efe0c84b9cb65223fac64e876a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`parseCache()`和`parseCacheRef()`方法其实是对`@CacheNamespace`和`@CacheNamespaceRef`这两个注解的处理。`parseStatement()`方法里面的各种`getAnnotation()`，都是对注解的解析，比如`@Options`，`@SelectKey`，`@ResultMap`等等。

**最后同样会解析成`MappedStatement`对象，也就是说在`XML`中配置，和使用注解配置，最后起到一样的效果**。

### 小结

经过上述步骤，我们主要完成了`config`配置文件、`Mapper`文件、`Mapper`接口上的注解的解析。我们得到了一个最重要的对象`Configuration`，这里面存放了全部的配置信息，它在属性里面还有各种各样的容器。最后，返回了一个`DefaultSqlSessionFactory`，里面持有了`Configuration`的实例。



## 创建会话

在构建好`SqlSessionFactory`对象，接下来创建会话了，也就是执行`SqlSession session = sqlSessionFactory.openSession()`，获取会话对象。具体源码如下： ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/547e727854484b3ca797968334dd3146~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

首先从`Configuration`里面拿到`Enviroment`，再通过`Enviroment`获取事务工厂`TransactionFactory`。接下里，通过事务工厂来产生一个事务，再生成一个执行器(事务包含在执行器里)，然后生成`DefaultSqlSession`。

### 创建 Transaction

如果配置的是`JDBC`，则会使用`Connection`对象的`commit()`、`rollback()`、`close()`管理事务。

如果是`Spring + MyBatis`，则没有必要配置，因为会直接使用`applicationContext.xml`里面配置数据源和事务管理器，覆盖MyBatis的配置。

### 创建 Executor

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/538c980da4634b53a3e3246d11c26db9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`Executor`的基本类型有三种:`SIMPLE`、`BATCH`、`REUSE`，默认是`SIMPLE` (`settingsElement()`读取默认值)，他们都继承了抽象类`BaseExecutor`。

`Executor`三种类型的区别是什么？

1. `SimpleExecutor`:每执行一次`update`或`select`，就开启一个`Statement`对象，用完立刻关闭 `Statement`对象。
2. `ReuseExecutor`:执行`update`或`select`，以 sql 作为`key`查找`Statement`对象，存在就使用，不存在就创建。用完后，不关闭`Statement`对象，而是放置于`Map`内， 供下一次使用。**简言之，就是重复使用 `Statement`对象**。
3. `BatchExecutor`:执行`update`(没有`select`，JDBC 批处理不支持`select`)，将所有sql都添加到批处理中(`addBatch()`)，等待统一执行(`executeBatch()`)，它缓存了多个`Statement`对象，每个 `Statement`对象都是`addBatch()`完毕后，等待逐一执行`executeBatch()`批处理。与 JDBC 批处理相同。

如果配置了`cacheEnabled=ture`，会用装饰器模式对`executor`进行包装:`new CachingExecutor(executor)`。

最后调用`Executor`的插件，执行对应的插件逻辑。(插件原理后续再讲)

### 小结

**创建会话的过程，主要是获得了一个`DefaultSqlSession`，里面包含了一个`Executor`，它是SQL的执行者**。

## 获取`Mapper`对象

在创建好`SqlSession`对象之后，接下来就是获取`Mapper`对象了。`Mapper.xml`文件与`Mapper`类型通过`namespace`进行关联，`Statement ID`与方法名进行了关联。因此，调用`Mapper`的方法就能执行相应的SQL了。

> `namespace`等于类的全路径名，`Statement ID`等于方法名。

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/44e43ac80a8942b89941f4d5bd2805a0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c53edc3b2d6c408fa99536d2082f9dac~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5cc94c5b09934390bb50915c478dee3e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

最后调用`MapperRegistry`的`getMapper()`方法。在前面配置解析阶段，我们讲过**添加映射就是将`Mapper`类型和对应的`MapperProxyFactory`关联，放到一个`Map`容器中**。而这里就是根据`Mapper`类型，得到对应的`MapperProxyFactory`，接下来通过代理工厂就可以创建一个`Mapper`代理对象了。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fb291df9be094dea8c9aec58919b6455~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

## 执行SQL

由于所有的`Mapper`都是`MapperProxy`代理对象，所以任意的方法都是执行`MapperProxy`的`invoke()`方法。 ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d597cde2dd6440b91b17cf65c687960~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

`invoke()`方法的执行步骤主要有2步，第一步是**查找MapperMethod**，第二步是**执行方法**。

### 查找MapperMethod

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ed12bc0c4ac9450b9a12b76572c97a75~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

优先从缓存中获取`MapperMethod`，缓存中没有则创建一个。

### 执行方法

执行方法就是调用`MapperMethod`的`execute()`方法。 ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c49c0321e174fdaa3f82c314c740f17~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

可以看到执行时就是4种情况，`insert|update|delete|select`，分别调用`SqlSession`的4大类方法。调用 `convertArgsToSqlCommandParam()`将参数转换为SQL的参数。

接下来，我们以查询单行记录为例，最终会执行`DefaultSqlSession.selectOne()`方法。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/423a6b6814c849b19918fb241f1c1f69~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

可以看到，`selectOne()`最终也是调用了`selectList()`。 ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1056b35bc7a4ff9a07f07c9e9529fcd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

在`SelectList()`中，我们先根据`Statement ID`从`Configuration`中拿到 `MappedStatement`，这个`ms`上面有我们在 xml中配置的所有属性，包括`id`、`statementType`、`sqlSource`、`useCache`、入参、出参等等。

### 查询

在获取到`MappedStatement`之后，接下就是调用执行器的`query()`方法了。查询是最复杂的sql处理，接下来详细分析查询的执行流程。

前面我们说到了`Executor`有三种基本类型，`SIMPLE/REUSE/BATCH`。另外还有一种包装类型`CachingExecutor`。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bc215e1aca52409da14fa11707347acc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

如果启用了二级缓存，就会先调用`CachingExecutor`的`query()`方法，里面有缓存相关的操作，然后才是再调用基本类型的执行器，比如默认的`SimpleExecutor`。最终会调用`BaseExecutor.query()`方法： ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/53cd1c27db744518b757165e23318791~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

主要包含获取绑定sql、创建CacheKey和执行查询。

#### 获取绑定sql

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9e1439c5c7c4b42b2e72f44fa6dc6f4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

根据输入参数，获取绑定sql

#### 创建`CacheKey`

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4fafbc7787dd404584b846e5d5fc2fd6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

MyBatis 对于其 Key 的生成采取规则为：`[mappedStementId + offset + limit + SQL + queryParams + environment]`生成一个哈希码

#### 执行查询

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0171ec922ee24ffc9748c1d066fd5b26~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

优先从缓存中查询，如果没有，则从数据库查询:`queryFromDatabase()`。

#### 数据库查询

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4aded1bb5d234865a52795d4303fd2e5~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

先向缓存用占位符占位。执行查询后，移除占位符，放入数据。执行查询调用的是`doQuery()`方法。

### 执行查询 `doQuery()`

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a14722bb2464d8f9091cb7abe5ec674~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

主要包含创建`StatementHandler`、准备语句和查询三步。

#### 创建`StatementHandler`

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/adc013a55dd340838862605b25efb97b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) ![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3e99f0e715e54c03ba5f0f67ba535dd7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

在`configuration.newStatementHandler()`中，创建了一个 `StatementHandler`，先得到 `RoutingStatementHandler`。`RoutingStatementHandler`里面没有任何的实现，它是用来创建基本的 `StatementHandler`的。这里会根据`MappedStatement`里面的`statementType`决定`StatementHandler`的类型 。 默认是`PREPARED`。 接下来以预处理语句处理器(`PREPARED`)为例进行分析。 ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f9e21ae9d0c4d0abc954a05a6bf9b88~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

这里直接调用了父类`BaseStatementHandler`的构造方法。 ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2417e886bc7429dbe70550a51444328~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

在构造方法里面，重点是生成了处理参数的`ParameterHandler`和处理结果集的`ResultSetHandler`。它们都可以被插件拦截。所以在创建之后都要用拦截器包装。 ![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c4486a70afa4d8f85512f30f4775c82~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

#### 准备语句

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2bd9459e2f004387bdfb1375cb4571ab~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp) 调用`prepareStatement()`方法对语句进行预处理。主要是根据连接准备语句和参数化处理。

#### 查询

`RoutingStatementHandler`的`query()`方法最终委派给`PreparedStatementHandler`执行。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/69e9a01b95644eb28fd47c9e5ca8be8d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

在这里，调用了`PreparedStatement`的`execute()`方法，它的底层就是是调用了JDBC的`PreparedStatement`进行执行，这里就不展开讲了。我们的重点是结果集的处理。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d6bba84ac1974c98820964b57eb5e814~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

首先我们会先拿到第一个结果集，如果没有配置一个查询返回多个结果集的情况，一般只有一个结果集。也就是下面的这个`while`循环只会执行一次，然后会调用`handleResultSet()`方法。 ![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a0f03b047fb045cfac9544cfbbe6df1c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

如果没有配置结果处理器，则会使用默认的结果处理器`DefaultResultHandler`，否则使用配置的结果处理器进行处理。

### 小结

总之，调用`Mapper`对象的方法就是执行sql。首先，它会根据方法和参数得到绑定的sql，然后创建语句处理器、准备好语句等等，之后就会通过JDBC执行sql，最后处理结果集，将结果转化为`Mapper`方法的返回类型返回。



## 总结

总的来说，mybatis核心工作流程是非常清晰的。

1. 第一步是配置解析，这一步的重点就是根据配置文件生成`Configuration`，基于它然后构建`DefaultSqlSessionFactory`对象。
2. 第二步是创建会话，主要是获得了一个`DefaultSqlSession`对象，里面包含了一个`Executor`，它是SQL的执行者。
3. 第三步是获取`Mapper`对象，这一步主要是根据第一步注册号的`Mapper`，创建好`MapperProxyFactory`代理对象。后续所有的方法调用都会调用`MapperProxyFactory`的`invoke()`方法。
4. 第四步是执行方法，在这里会创建语句处理器、准备好语句等等，之后就会通过JDBC执行sql，最后处理结果集，将结果转化为`Mapper`方法的返回类型返回。





源码解读：

https://github.com/chentianming11/mybatis