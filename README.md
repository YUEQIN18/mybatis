# Mybatis工作流程源码浅读



## 工作流程

Mybatis的工作流程可以简单概括为

1. 配置解析
2. 创建会话
3. 获取Mapper
4. 执行SQL
6. 结果映射



## 配置解析

mybatis有两种配置文件，一种是全局配置文件`mybatis-config.xml`，另一种就是各个具体的Mapper文件`xxxMapper.xml `。在上面的代码中，核心就是基于`mybatis-config.xml`配置文件构建出了`sqlSessionFactory`对象。跟进`SqlSessionFactoryBuilder`的`build()`方法：

```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
  try {
    XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
    return build(parser.parse());
  }
  ...
}
```

上面代码的关键是调用了`parser`的`parse()`方法，它会返回一个`Configuration`类。 

```java
public Configuration parse() {
	...
  parseConfiguration(parser.evalNode("/configuration"));
  return configuration;
}
```

明显可以看出来，解析配置是从configuration根节点开始解析的。具体解析过程如下： 

```java
private void parseConfiguration(XNode root) {
  try {
    // issue #117 read properties first
    propertiesElement(root.evalNode("properties"));
    Properties settings = settingsAsProperties(root.evalNode("settings"));
    loadCustomVfsImpl(settings);
    loadCustomLogImpl(settings);
    typeAliasesElement(root.evalNode("typeAliases"));
    pluginsElement(root.evalNode("plugins"));
    objectFactoryElement(root.evalNode("objectFactory"));
    objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
    reflectorFactoryElement(root.evalNode("reflectorFactory"));
    settingsElement(settings);
    // read it after objectFactory and objectWrapperFactory issue #631
    environmentsElement(root.evalNode("environments"));
    databaseIdProviderElement(root.evalNode("databaseIdProvider"));
    typeHandlersElement(root.evalNode("typeHandlers"));
    mappersElement(root.evalNode("mappers"));
  } catch (Exception e) {
    throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
  }
}
```

解析配置完整方法上所示，主要的解析过程包含属性、类型别名、插件、对象工厂、对象包装工厂、设置、环境、类型处理器、映射器等等。下面挑选最关键的部分具体讲解。˛

### 属性 `propertiesElement()`

第一个是解析`<properties>`标签，读取我们引入的外部配置文件。解析的最终结果就是我们会把所有的配置信息放到名为`defaults`的`Properties`对象里面，最后把`XPathParser`和`Configuration`的`Properties`属性都设置成填充后的`Properties`对象。

```xml
<properties resource="org/mybatis/example/config.properties">
  <properties name="username" value="dev"/>
</properties>
```

```java
private void propertiesElement(XNode context) throws Exception {
  if (context == null) {
    return;
  }
  Properties defaults = context.getChildrenAsProperties();
  String resource = context.getStringAttribute("resource");
  String url = context.getStringAttribute("url");
  if (resource != null && url != null) {
    throw new BuilderException(
        "The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
  }
  if (resource != null) {
    defaults.putAll(Resources.getResourceAsProperties(resource));
  } else if (url != null) {
    defaults.putAll(Resources.getUrlAsProperties(url));
  }
  Properties vars = configuration.getVariables();
  if (vars != null) {
    defaults.putAll(vars);
  }
  parser.setVariables(defaults);
  configuration.setVariables(defaults);
}
```



### 类型别名 `typeAliasesElement()`

类型别名是解析`<typeAliases>`标签，它有两种定义方式，一种是直接定义一个类的别名，另一种是指定一个包，此时这个`package`下面所有的类的名字就会成为这个类全路径的别名。类的别名和类的关系，放在一个`TypeAliasRegistry`对象里面。

```xml
<typeALlases>
  <typeAlias alias="Blog" type="domain.blog.BLog"/> 
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
</typeALiases>

<typeALiases>
  <package name="domain.blog"/>
</typeALiases>
```

```java
private void typeAliasesElement(XNode context) {
  if (context == null) {
    return;
  }
  for (XNode child : context.getChildren()) {
    if ("package".equals(child.getName())) {
      String typeAliasPackage = child.getStringAttribute("name");
      configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
    } else {
      String alias = child.getStringAttribute("alias");
      String type = child.getStringAttribute("type");
      try {
        Class<?> clazz = Resources.classForName(type);
        if (alias == null) {
          typeAliasRegistry.registerAlias(clazz);
        } else {
          typeAliasRegistry.registerAlias(alias, clazz);
        }
      } catch (ClassNotFoundException e) {
        throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
      }
    }
  }
}
```

### 插件 `pluginElement()`

插件是解析`<plugins>`标签。`<plugins>`标签里面只有`<plugin>`标签，`<plugin>`标签里面只有`<property>`标 签。标签解析完以后，会生成一个`Interceptor`对象，并且添加到`Configuration`的 `InterceptorChain`属性里面，它是一个`List`。

```xml
<plugins>
	<plugin interceptor="org.mybatis.example.ExamplePlugin">
  	<property name="a" value="100"/>
  </plugin>
</plugins>
```

```java
private void pluginsElement(XNode context) throws Exception {
  if (context != null) {
    for (XNode child : context.getChildren()) {
      String interceptor = child.getStringAttribute("interceptor");
      Properties properties = child.getChildrenAsProperties();
      Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor()
          .newInstance();
      interceptorInstance.setProperties(properties);
      configuration.addInterceptor(interceptorInstance);
    }
  }
}
```

### 对象工厂 `objectFactoryElement()`、`objectWrapperFactoryElement()`

这两个标签是用来实例化对象用的，分别解析的是`<objectFactory>`和`<objectWrapperFactory>`这两个标签，对应生成`ObjectFactory`、`ObjectWrapperFactory`对象，同样设置到`Configuration`的属性里面。

```java
private void objectFactoryElement(XNode context) throws Exception {
  if (context != null) {
    String type = context.getStringAttribute("type");
    Properties properties = context.getChildrenAsProperties();
    ObjectFactory factory = (ObjectFactory) resolveClass(type).getDeclaredConstructor().newInstance();
    factory.setProperties(properties);
    configuration.setObjectFactory(factory);
  }
}

private void objectWrapperFactoryElement(XNode context) throws Exception {
  if (context != null) {
    String type = context.getStringAttribute("type");
    ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).getDeclaredConstructor().newInstance();
    configuration.setObjectWrapperFactory(factory);
  }
}
```

### 设置 `settingsElement()`

设置是解析`<settings>`标签，首先将所有子标签全部转换成了`Properties`对象，然后再将相应的属性设置到`Configuration`中。

二级标签里面有很多的配置，比如二级缓存，延迟加载，自动生成主键这些。需要注意的是，**所有的默认值都是在这里赋值的**。如果我们不知道这个属性的值是什么，也可以到这一步来确认一下。 所有的值，都会赋值到`Configuration`的属性中。

```java
private void settingsElement(Properties props) {
  configuration
      .setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
  configuration.setAutoMappingUnknownColumnBehavior(
      AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
  configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
  configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
  configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
  configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
  configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
  configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
  configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
  configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
  configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
  configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
  configuration.setDefaultResultSetType(resolveResultSetType(props.getProperty("defaultResultSetType")));
  configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
  configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
  configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
  configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
  configuration.setLazyLoadTriggerMethods(
      stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
  configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
  configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
  configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
  configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
  configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
  configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
  configuration.setLogPrefix(props.getProperty("logPrefix"));
  configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
  configuration.setShrinkWhitespacesInSql(booleanValueOf(props.getProperty("shrinkWhitespacesInSql"), false));
  configuration.setArgNameBasedConstructorAutoMapping(
      booleanValueOf(props.getProperty("argNameBasedConstructorAutoMapping"), false));
  configuration.setDefaultSqlProviderType(resolveClass(props.getProperty("defaultSqlProviderType")));
  configuration.setNullableOnForEach(booleanValueOf(props.getProperty("nullableOnForEach"), false));
}
```

### 环境 `environmentsElement()`

一个`environment`就是对应一个数据源，所以在这里我们会根据配置的`<transactionManager>`创建一个事务工厂，根据`<dataSource>`标签创建一个数据源，最后把这两个对象设置成`Environment`对象的属性，放到 `Configuration`里面。

```xml
<environments default="development">
	<environment id="development">
  	<treansactionManager type="JDBC">
    	<property name="a" value="100"/>
    </treansactionManager>
    
    <dataSource type="POOLED">
    	<property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
</environments>
```

```java
private void environmentsElement(XNode context) throws Exception {
  if (context == null) {
    return;
  }
  if (environment == null) {
    environment = context.getStringAttribute("default");
  }
  for (XNode child : context.getChildren()) {
    String id = child.getStringAttribute("id");
    if (isSpecifiedEnvironment(id)) {
      TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
      DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
      DataSource dataSource = dsFactory.getDataSource();
      Environment.Builder environmentBuilder = new Environment.Builder(id).transactionFactory(txFactory)
          .dataSource(dataSource);
      configuration.setEnvironment(environmentBuilder.build());
      break;
    }
  }
}
```

### 类型处理器 `typeHandlerElement()`

跟`TypeAlias`一样，`TypeHandler`有两种配置方式，一种是单独配置一个类，一种是指定一个`package`。最后我们得到的是`JavaType`和`JdbcType`，以及用来做相互映射的`TypeHandler`，并将其保存在 `TypeHandlerRegistry`对象里面。

```java
private void typeHandlersElement(XNode context) {
  if (context == null) {
    return;
  }
  for (XNode child : context.getChildren()) {
    if ("package".equals(child.getName())) {
      String typeHandlerPackage = child.getStringAttribute("name");
      typeHandlerRegistry.register(typeHandlerPackage);
    } else {
      String javaTypeName = child.getStringAttribute("javaType");
      String jdbcTypeName = child.getStringAttribute("jdbcType");
      String handlerTypeName = child.getStringAttribute("handler");
      Class<?> javaTypeClass = resolveClass(javaTypeName);
      JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
      Class<?> typeHandlerClass = resolveClass(handlerTypeName);
      if (javaTypeClass != null) {
        if (jdbcType == null) {
          typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
        } else {
          typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
        }
      } else {
        typeHandlerRegistry.register(typeHandlerClass);
      }
    }
  }
}
```

### 映射器 `mapperElement()`

```xml
// 使用xml路径
<mappers>
	<mapper resource="org/mybatis/builder/AutherMapper.xml"/>
  <mapper resource="org/mybatis/builder/BlogMapper.xml"/>
  <mapper resource="org/mybatis/builder/PostMapper.xml"/>
</mappers>
// 或者使用java类名
<mappers>
	<mapper class="org.mybatis.builder.AutherMapper"/>
  <mapper class="org.mybatis.builder.BlogMapper"/>
  <mapper class="org.mybatis.builder.PostMapper"/>
</mappers>
// 或者自动扫描宝 最常用
<mappers>
	<package name="org.mybatis.builder"
</mappers>
```

映射器支持4种配置方式，包括类路径、绝对url路径、Java类名和自动扫描包方式。

```java
private void mappersElement(XNode context) throws Exception {
  if (context == null) {
    return;
  }
  for (XNode child : context.getChildren()) {
    if ("package".equals(child.getName())) {
      String mapperPackage = child.getStringAttribute("name");
      configuration.addMappers(mapperPackage);
    } else {
      String resource = child.getStringAttribute("resource");
      String url = child.getStringAttribute("url");
      String mapperClass = child.getStringAttribute("class");
      if (resource != null && url == null && mapperClass == null) {
        ErrorContext.instance().resource(resource);
        try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource,
              configuration.getSqlFragments());
          mapperParser.parse();
        }
      } else if (resource == null && url != null && mapperClass == null) {
        ErrorContext.instance().resource(url);
        try (InputStream inputStream = Resources.getUrlAsStream(url)) {
          XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url,
              configuration.getSqlFragments());
          mapperParser.parse();
        }
      } else if (resource == null && url == null && mapperClass != null) {
        Class<?> mapperInterface = Resources.classForName(mapperClass);
        configuration.addMapper(mapperInterface);
      } else {
        throw new BuilderException(
            "A mapper element may only specify a url, resource or class, but not more than one.");
      }
    }
  }
}
```

下面以自动扫描包为例，具体说明，查看`configuration.addMappers(mapperPackage)`实现： 

```java
public void addMappers(String packageName) {
  addMappers(packageName, Object.class);
}
```

查找指定包下所有Object的子类的`mapperClass`，遍历。

```java
public void addMappers(String packageName, Class<?> superType) {
  ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
  resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
  Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
  for (Class<?> mapperClass : mapperSet) {
    addMapper(mapperClass);
  }
}
```

继续跟进`addMapper(mapperClass)`实现： 

```java
public <T> void addMapper(Class<T> type) {
  if (type.isInterface()) {
    if (hasMapper(type)) {
      throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
    }
    boolean loadCompleted = false;
    try {
      knownMappers.put(type, new MapperProxyFactory<>(type));
      // It's important that the type is added before the parser is run
      // otherwise the binding may automatically be attempted by the
      // mapper parser. If the type is already known, it won't try.
      MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
      parser.parse();
      loadCompleted = true;
    } finally {
      if (!loadCompleted) {
        knownMappers.remove(type);
      }
    }
  }
}
```

添加映射就是将`Mapper`类型和对应的`MapperProxyFactory`关联，放到一个名为`knownMappers `的 `Map`容器中。并且在这里还会去解析`Mapper`接口方法上的注解。具体来说是创建了一个`MapperAnnotationBuilder`，我们点进去看一下 `parse()`方法。 

```java
public void parse() {
  String resource = type.toString();
  if (!configuration.isResourceLoaded(resource)) {
    loadXmlResource();
    configuration.addLoadedResource(resource);
    assistant.setCurrentNamespace(type.getName());
    parseCache(); // @CacheNamespace
    parseCacheRef(); // @CacheNamespaceRef
    for (Method method : type.getMethods()) {
      if (!canHaveStatement(method)) {
        continue;
      }
      if (getAnnotationWrapper(method, false, Select.class, SelectProvider.class).isPresent()
          && method.getAnnotation(ResultMap.class) == null) {
        parseResultMap(method);
      }
      try {
        parseStatement(method);
      } catch (IncompleteElementException e) {
        configuration.addIncompleteMethod(new MethodResolver(this, method));
      }
    }
  }
  parsePendingMethods();
}
```

`parseCache()`和`parseCacheRef()`方法其实是对`@CacheNamespace`和`@CacheNamespaceRef`这两个注解的处理。`parseStatement()`方法里面的各种`getAnnotation()`，都是对注解的解析，比如`@Options`，`@SelectKey`，`@ResultMap`等等。

**最后同样会解析成`MappedStatement`对象，也就是说在`XML`中配置，和使用注解配置，最后起到一样的效果**。

### 小结

经过上述步骤，我们主要完成了`config`配置文件、`Mapper`文件、`Mapper`接口上的注解的解析。我们得到了一个最重要的对象`Configuration`，这里面存放了全部的配置信息，它在属性里面还有各种各样的容器。最后，返回了一个`DefaultSqlSessionFactory`，里面持有了`Configuration`的实例。



## 创建会话

在构建好`SqlSessionFactory`对象，接下来创建会话了，也就是执行`SqlSession session = sqlSessionFactory.openSession()`，获取会话对象。具体源码如下： 

```java
@Override
public SqlSession openSession() {
  return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}
```

```java
private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level,
    boolean autoCommit) {
  Transaction tx = null;
  try {
    final Environment environment = configuration.getEnvironment();
    final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
    // 通过事务工厂生产一个事务
    tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
    // 生成一个执行器（事务包含在执行器里）
    final Executor executor = configuration.newExecutor(tx, execType);
    return new DefaultSqlSession(configuration, executor, autoCommit);
  }
  ...
}
```

首先从`Configuration`里面拿到`Enviroment`，再通过`Enviroment`获取事务工厂`TransactionFactory`。接下里，通过事务工厂来产生一个事务，再生成一个执行器(事务包含在执行器里)，然后生成`DefaultSqlSession`。

### 创建 Transaction

如果配置的是`JDBC`，则会使用`Connection`对象的`commit()`、`rollback()`、`close()`管理事务。

```java
// JdbcTransaction
@Override
public void commit() throws SQLException {
  if (connection != null && !connection.getAutoCommit()) {
    if (log.isDebugEnabled()) {
      log.debug("Committing JDBC Connection [" + connection + "]");
    }
    connection.commit();
  }
}
```

如果是`Spring + MyBatis`，则没有必要配置，因为会直接使用`applicationContext.xml`里面配置数据源和事务管理器，覆盖MyBatis的配置。 

### 创建 Executor

```java
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
  executorType = executorType == null ? defaultExecutorType : executorType;
  Executor executor;
  if (ExecutorType.BATCH == executorType) {
    executor = new BatchExecutor(this, transaction);
  } else if (ExecutorType.REUSE == executorType) {
    executor = new ReuseExecutor(this, transaction);
  } else {
    executor = new SimpleExecutor(this, transaction);
  }
  if (cacheEnabled) {
    executor = new CachingExecutor(executor);
  }
  return (Executor) interceptorChain.pluginAll(executor);
}
```

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

```java
// Configuration类的方法
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  return mapperRegistry.getMapper(type, sqlSession);
}
```

```java
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
  final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
  if (mapperProxyFactory == null) {
    throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
  }
  try {
    return mapperProxyFactory.newInstance(sqlSession);
  } catch (Exception e) {
    throw new BindingException("Error getting mapper instance. Cause: " + e, e);
  }
}
```

最后调用`MapperRegistry`的`getMapper()`方法。在前面**配置解析**阶段，我们讲过**添加映射就是将`Mapper`类型和对应的`MapperProxyFactory`关联，放到一个`Map`容器中**。而这里就是根据`Mapper`类型，得到对应的`MapperProxyFactory`，接下来通过代理工厂就可以创建一个`Mapper`代理对象了。 

```java
public T newInstance(SqlSession sqlSession) {
  final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
  return newInstance(mapperProxy);
}

protected T newInstance(MapperProxy<T> mapperProxy) {
  return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
}
```



## 执行SQL

由于所有的`Mapper`都是`MapperProxy`代理对象，所以任意的方法都是执行`MapperProxy`的`invoke()`方法。 `invoke()`方法的执行步骤主要有2步，第一步是**查找MapperMethodInvoker**，第二步是**执行方法**。

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  try {
    if (Object.class.equals(method.getDeclaringClass())) {
      return method.invoke(this, args);
    }
    return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
  } catch (Throwable t) {
    throw ExceptionUtil.unwrapThrowable(t);
  }
}
```



### 查找MapperMethodInvoker

优先从缓存中获取`MapperMethodInvoker`，缓存中没有则创建一个。

```java
private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
  try {
    // MapUtil的这个方法的穿参 methodCache正是缓存
    return MapUtil.computeIfAbsent(methodCache, method, m -> {
      // 这里判断了是不是默认方法，默认方法就是接口声明的默认实现方法，不需要实现类去实现其方法
      if (!m.isDefault()) {
        // 不是默认方法就new一个PlainMethodInvoker
        return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
      }
      try {
        if (privateLookupInMethod == null) {
          return new DefaultMethodInvoker(getMethodHandleJava8(method));
        }
        return new DefaultMethodInvoker(getMethodHandleJava9(method));
      } catch (IllegalAccessException | InstantiationException | InvocationTargetException
          | NoSuchMethodException e) {
        throw new RuntimeException(e);
      }
    });
  } catch (RuntimeException re) {
    Throwable cause = re.getCause();
    throw cause == null ? re : cause;
  }
}
```



### 执行方法

如果不是默认方法，执行方法就是调用`PlainMethodInvoker`的`invoke()`，也就是`mapperMethod `的`execute()`方法。 

```java
private static class PlainMethodInvoker implements MapperMethodInvoker {
  private final MapperMethod mapperMethod;

  public PlainMethodInvoker(MapperMethod mapperMethod) {
    this.mapperMethod = mapperMethod;
  }

  @Override
  public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
    return mapperMethod.execute(sqlSession, args);
  }
}
```

可以看到执行时就是5种情况，`insert|update|delete|select|flush`，分别调用`SqlSession`的5大类方法。调用 `convertArgsToSqlCommandParam()`将参数转换为SQL的参数。

```java
public Object execute(SqlSession sqlSession, Object[] args) {
  Object result;
  switch (command.getType()) {
    case INSERT: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
      break;
    }
    case UPDATE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
      break;
    }
    case DELETE: {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
      break;
    }
    case SELECT:
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
        if (method.returnsOptional() && (result == null || !method.getReturnType().equals(result.getClass()))) {
          result = Optional.ofNullable(result);
        }
      }
      break;
    case FLUSH:
      result = sqlSession.flushStatements();
      break;
    default:
      throw new BindingException("Unknown execution method for: " + command.getName());
  }
	...
  return result;
}
```



接下来，我们以查询单行记录为例，最终会执行`DefaultSqlSession.selectOne()`方法。 

```java
@Override
public <T> T selectOne(String statement, Object parameter) {
  // Popular vote was to return null on 0 results and throw exception on too many.
  List<T> list = this.selectList(statement, parameter);
  if (list.size() == 1) {
    return list.get(0);
  }
  if (list.size() > 1) {
    throw new TooManyResultsException(
        "Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
  } else {
    return null;
  }
}
```

可以看到，`selectOne()`最终也是调用了`selectList()`。 在`SelectList()`中，我们先根据`Statement ID`从`Configuration`中拿到 `MappedStatement`，这个`ms`上面有我们在 xml中配置的所有属性，包括`id`、`statementType`、`sqlSource`、`useCache`、入参、出参等等。

```java
private <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler) {
  try {
    //根据 statement id查找缓存的MappedStatement
    MappedStatement ms = configuration.getMappedStatement(statement);
    dirty |= ms.isDirtySelect();
    //用执行器查询结果，这里的ResultHandler是null
    return executor.query(ms, wrapCollection(parameter), rowBounds, handler);
  } catch (Exception e) {
    throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
  } finally {
    ErrorContext.instance().reset();
  }
}
```



### 查询

在获取到`MappedStatement`之后，接下就是调用执行器的`query()`方法了。查询是最复杂的sql处理，接下来详细分析查询的执行流程。

前面我们说到了`Executor`有三种基本类型，`SIMPLE/REUSE/BATCH`。另外还有一种包装类型`CachingExecutor`。 

如果启用了二级缓存，就会先调用`CachingExecutor`的`query()`方法，里面有缓存相关的操作，主要包含获取绑定sql、创建CacheKey和执行查询。

```java
@Override
public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler)
    throws SQLException {
  BoundSql boundSql = ms.getBoundSql(parameterObject);
  CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
  return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
}
```



#### 获取绑定sql

最主要是调用了sqlSource.getBoundSql()，这个方法跟据sql语句是xml配置的还是注解来生成BoundSql

```java
public BoundSql getBoundSql(Object parameterObject) {
  BoundSql boundSql = sqlSource.getBoundSql(parameterObject);
	...
  return boundSql;
}
```



#### 创建`CacheKey`

MyBatis 对于其 Key 的生成采取规则为：`[mappedStementId + offset + limit + SQL + queryParams + environment]`生成一个哈希码

```java
public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
  if (closed) {
    throw new ExecutorException("Executor was closed.");
  }
  CacheKey cacheKey = new CacheKey();
  cacheKey.update(ms.getId());
  cacheKey.update(rowBounds.getOffset());
  cacheKey.update(rowBounds.getLimit());
  cacheKey.update(boundSql.getSql());
  List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
  TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
  // mimic DefaultParameterHandler logic
  MetaObject metaObject = null;
  for (ParameterMapping parameterMapping : parameterMappings) {
    if (parameterMapping.getMode() != ParameterMode.OUT) {
      Object value;
      String propertyName = parameterMapping.getProperty();
      if (boundSql.hasAdditionalParameter(propertyName)) {
        value = boundSql.getAdditionalParameter(propertyName);
      } else if (parameterObject == null) {
        value = null;
      } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
        value = parameterObject;
      } else {
        if (metaObject == null) {
          metaObject = configuration.newMetaObject(parameterObject);
        }
        value = metaObject.getValue(propertyName);
      }
      cacheKey.update(value);
    }
  }
  if (configuration.getEnvironment() != null) {
    // issue #176
    cacheKey.update(configuration.getEnvironment().getId());
  }
  return cacheKey;
}
```

#### 执行查询

最后调用基本类型的执行器，比如默认的`SimpleExecutor`。最终会调用`BaseExecutor.query()`方法： 

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler,
    CacheKey key, BoundSql boundSql) throws SQLException {
  ...
  List<E> list;
  try {
    queryStack++;
    // 先根据上面得到的cacheKey查询缓存
    list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
    if (list != null) {
      handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
    } else {
      // 查不到就查询数据库
      list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
  } finally {
    queryStack--;
  }
	...
  return list;
}
```

优先从缓存中查询，如果没有，则从数据库查询:`queryFromDatabase()`。



#### 数据库查询

```java
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds,
    ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
  List<E> list;
  localCache.putObject(key, EXECUTION_PLACEHOLDER);
  try {
    list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
  } finally {
    localCache.removeObject(key);
  }
  localCache.putObject(key, list);
  if (ms.getStatementType() == StatementType.CALLABLE) {
    localOutputParameterCache.putObject(key, parameter);
  }
  return list;
}
```

先向缓存用占位符占位。执行查询后，移除占位符，放入数据。执行查询调用的是`doQuery()`方法。



### 执行查询

```java
@Override
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler,
    BoundSql boundSql) throws SQLException {
  Statement stmt = null;
  try {
    Configuration configuration = ms.getConfiguration();
    // 创建StatementHandler
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
    // 创建PreparedStatement
    stmt = prepareStatement(handler, ms.getStatementLog());
    // 查询
    return handler.query(stmt, resultHandler);
  } finally {
    closeStatement(stmt);
  }
}
```

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9a14722bb2464d8f9091cb7abe5ec674~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

主要包含创建`StatementHandler`、创建PreparedStatement和查询三步。

创建`StatementHandler`

在`configuration.newStatementHandler()`中，创建了一个 `StatementHandler`，先得到 `RoutingStatementHandler`。`RoutingStatementHandler`里面没有任何的实现，它是用来创建基本的 `StatementHandler`的。

```java
public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement,
    Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  // 先new一个包装类型的StatementHandler
  StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject,
      rowBounds, resultHandler, boundSql);
  // 插件在这时插入
  return (StatementHandler) interceptorChain.pluginAll(statementHandler);
}
```

 这里会根据`MappedStatement`里面的`statementType`决定`StatementHandler`的类型 。 默认是`PREPARED`。 

```java
public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  switch (ms.getStatementType()) {
    case STATEMENT:
      delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case PREPARED:
      delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    case CALLABLE:
      delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
      break;
    default:
      throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
  }
}
```

接下来以预处理语句处理器(`PREPARED`)为例进行分析。

这里直接调用了父类`BaseStatementHandler`的构造方法。 

```java
public class PreparedStatementHandler extends BaseStatementHandler {

  public PreparedStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameter,
      RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    super(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
  }
  ...
}
```

在构造方法里面，重点是生成了处理参数的`ParameterHandler`和处理结果集的`ResultSetHandler`。它们都可以被插件拦截。所以在创建之后都要用拦截器包装。 

```java
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject,
    RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
  this.configuration = mappedStatement.getConfiguration();
  this.executor = executor;
  this.mappedStatement = mappedStatement;
  this.rowBounds = rowBounds;

  this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
  this.objectFactory = configuration.getObjectFactory();

  if (boundSql == null) { // issue #435, get the key before calculating the statement
    generateKeys(parameterObject);
    boundSql = mappedStatement.getBoundSql(parameterObject);
  }

  this.boundSql = boundSql;

  this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
  this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
}
```

```java
public ParameterHandler newParameterHandler(MappedStatement mappedStatement, Object parameterObject,
    BoundSql boundSql) {
  ParameterHandler parameterHandler = mappedStatement.getLang().createParameterHandler(mappedStatement,
      parameterObject, boundSql);
  return (ParameterHandler) interceptorChain.pluginAll(parameterHandler);
}

public ResultSetHandler newResultSetHandler(Executor executor, MappedStatement mappedStatement, RowBounds rowBounds, ParameterHandler parameterHandler, ResultHandler resultHandler, BoundSql boundSql) {
  ResultSetHandler resultSetHandler = new DefaultResultSetHandler(executor, mappedStatement, parameterHandler, resultHandler, boundSql, rowBounds);
  return (ResultSetHandler) interceptorChain.pluginAll(resultSetHandler);
}
```



创建PreparedStatement

 调用`prepareStatement()`方法对语句进行预处理。主要是根据连接准备语句和参数化处理。

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
  Statement stmt;
  Connection connection = getConnection(statementLog);
  stmt = handler.prepare(connection, transaction.getTimeout());
  handler.parameterize(stmt);
  return stmt;
}
```

查询

`RoutingStatementHandler`的`query()`方法最终委派给`PreparedStatementHandler`执行。 

```java
@Override
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
  PreparedStatement ps = (PreparedStatement) statement;
  ps.execute();
  return resultSetHandler.handleResultSets(ps);
}
```

在这里，调用了`PreparedStatement`的`execute()`方法，它的底层就是是调用了JDBC的`PreparedStatement.execute()`进行执行

因为`execute()`方法不会返回结果，所以还要再去手动获取结果集。

首先我们会先拿到第一个结果集，如果没有配置一个查询返回多个结果集的情况，一般只有一个结果集。也就是下面的这个`while`循环只会执行一次，然后会调用`handleResultSet()`方法。 

```java
@Override
public List<Object> handleResultSets(Statement stmt) throws SQLException {
  ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

  final List<Object> multipleResults = new ArrayList<>();

  int resultSetCount = 0;
  ResultSetWrapper rsw = getFirstResultSet(stmt);

  List<ResultMap> resultMaps = mappedStatement.getResultMaps();
  int resultMapCount = resultMaps.size();
  validateResultMapsCount(rsw, resultMapCount);
  while (rsw != null && resultMapCount > resultSetCount) {
    ResultMap resultMap = resultMaps.get(resultSetCount);
    // 处理结果集
    handleResultSet(rsw, resultMap, multipleResults, null);
    rsw = getNextResultSet(stmt);
    cleanUpAfterHandlingResultSet();
    resultSetCount++;
  }
	...
  return collapseSingleResultList(multipleResults);
}
```

```java
private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults,
    ResultMapping parentMapping) throws SQLException {
  try {
    if (parentMapping != null) {
      handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
    } else if (resultHandler == null) {
      // 如果没有配置结果处理器，则会使用默认的DefaultResultHandler，否则使用配置的结果处理器进行处理。
      DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
      handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
      multipleResults.add(defaultResultHandler.getResultList());
    } else {
      handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
    }
  } finally {
    // issue #228 (close resultsets)
    closeResultSet(rsw.getResultSet());
  }
}
```



## 结果映射



```java
public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler,
    RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
  if (resultMap.hasNestedResultMaps()) {
    ensureNoRowBounds();
    checkResultHandler();
    handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  } else {
    handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
  }
}
```







### 小结

总之，调用`Mapper`对象的方法就是执行sql。首先，它会根据方法和参数得到绑定的sql，然后创建语句处理器、准备好语句等等，之后就会通过JDBC执行sql，最后处理结果集，将结果转化为`Mapper`方法的返回类型返回。



## 总结

总的来说，mybatis核心工作流程是非常清晰的。

1. 第一步是配置解析，这一步的重点就是根据配置文件生成`Configuration`，基于它然后构建`DefaultSqlSessionFactory`对象。
2. 第二步是创建会话，主要是获得了一个`DefaultSqlSession`对象，里面包含了一个`Executor`，它是SQL的执行者。
3. 第三步是获取`Mapper`对象，这一步主要是根据第一步注册号的`Mapper`，创建好`MapperProxyFactory`代理对象。后续所有的方法调用都会调用`MapperProxyFactory`的`invoke()`方法。
4. 第四步是执行方法，在这里会创建语句处理器、准备好语句等等，之后就会通过JDBC执行sql，最后处理结果集，将结果转化为`Mapper`方法的返回类型返回。
5. 第五步是结果处理





源码解读参考：

https://github.com/chentianming11/mybatis