## Mybatis 源码

## 简介

​		`mybatis` 源码阅读主要通过`mybatis `原生`java  api ` 进行调用，不断进行深入，从而解析出`mybatis` 整体执行情况，之后在与`spring ` 进行整合，最后针对于`spring boot` 进行整合。

​        本文主要关注于 配置解析、各类插件加载与执行、二级缓存、参数解析、结果映射 这几块内容。

​		本文主要目标为 熟悉`mybatis` 整体流程，进行源码修改。熟悉插件编写，能够较为全面的考虑其可用性、扩展行。

## 源码解析之配置读取

​		一切入口的起点，在于`Mybatis ` 中的 `SqlSessionFactroyBuilder` 进行创建 `SqlSessionFactory` 开始，所有框架的第一步基本都是解析配置文件，`mybatis` 也不例外，在创建 `SqlSessionFatory` 之前，`Mybatis` 将 `config 文件以及 mapper文件`解析到了 `Configuration` 对象中。其中该对象比较常用的字段如下：

  1. `ExecutorType defaultExecutorType` ( `Mybatis` 核心，执行器的默认类型，默认为 `SIMPLE` ) 

  2. `ObjectFactory objectFactory` 对象创建工厂

  3. `MapperRegistry mapperRegistry` 一个`mapper` 文件与一个接口的映射集合。

  4. `TypeHandlerRegistry typeHandlerRegistry`  `TypeHandler` 映射集合。

  5. `TypeAliasRegistry typeAliasRegistry` 别名映射。

  6. `Map<String, MappedStatement> mappedStatements` 所有`sql` 存放地点。

  7. `Map<String, Cache> caches` 缓存映射。

  8. `Map<String, ResultMap> resultMaps` 返回结果集 映射。

  9. `Map<String, ParameterMap> parameterMaps` 参数集合映射。

  10. `Map<String, KeyGenerator> keyGenerators` 主键自动生成映射。

      那么以上的参数是如何进行初始化的？ 具体存储的内容是什么？

​	    一切都要从 `XMLConfigBuilder` 说起，这个对象主要用于读取 `mybatis` 的 `config` 文件，`mybatis` 解析 `xml` 文件 ，主要引用的是  原生`Java 中的com.sun.org.apache.xerces.internal.jaxp.DocumentBuilderFactoryImpl`对象， 通过这个工厂创建出 `javax.xml.parsers.DocumentBuilder` ， 将 `xml ` 文件解析成一个 `Document` 。进行配置文件读取时，会从最外层的 `configuration` 开始。一层层的遍历子节点。

​       其中涉及到的节点为 `properties` 、`settings`、`typeAliases`、`plugins` 、`objectFactory` 、`objectWrapperFactory` 、`reflectorFactory` 、`environments` 、`databaseIdProvider` 、`typeHandlers` 、`mappers`。其中 常用的我们只分析常用的几个标签。

​		首先将`setting` 中的标签存储到 `Properties` 对象中 ，并根据 `setting` 中的 `logImpl` 初始化 `configruation` 中的日志。

​		然后解析 `typeAliases`  标签，遍历其子标签，如果为 `package`  则进行扫描所有子目录，将目录下所有的类都注册到 `registerAliases` 中，默认为类名，也可以进行 `org.apache.ibatis.type.Alias`，进行指定对应类的别名。最后以别名为 `key` 对应的 `Class<?>` 为元素注册到 `typeAliases` 中。

  	 别名解析完成以后 `mybatis` 有解析了 `plugins` ，遍历了每一个子标签 `interceptor` 并根据对应的配置类进行初始化，将初始化后的对象加入到 `configuration 的 interceptorChain` 属性中。

​		处理 `setting` 中的属性，将配置的值进行设定 `configuration` 中，如果没有的话 赋予默认值。

​		解析 `environments` 环境变量标签，主要根据配置 设置了 事务工厂 `TransactionFactory` 以及数据源 `DataSourceFactory` 	，最后将其 赋值给 `configruation 中的 environment` 对象。

​		`typeHandlers` 标签解析，循环所有子标签，子标签分为 `package` 与  `typeHandler` 两种。如果是 ·`package`  则需要找到该路径下所有实现 `typeHandler` 的类，注册 `typeHandler` 需要三个参数 `java类型，jdbc类型,typehandler`。遍历找到的所有类，在 `typehandler 实现类的注解MappedTypes` 中获取 `java类型` ，然后在 `typehandler 实现类的注解MappedJdbcTypes`  中获取 `jdbc类型` 将它们赋值给 `cofiguration` 中的 `typeHandlerMap` 属性。`key` 为 `JavaType` `value` 为以 `jdbcType` 为 `key` `typehandler` 为 `value` 的 `map`。

​		以上主要解析了 `setting、typeAliases、plugins、typeHandler` 标签，我明白了，每一个标签具体的存储形式，而 `myabtis` 对于 `sql` 的解析主要是在 解析`mappers` 标签中进行设定的。

对于 `mappers`  的标签解析，其子标签分为 `package` 与 `mapper` 标签两类，不论是哪种都需要进行两部操作，

1. 将接口的 `Class` 对象注册到 `configuration 的 mapperRegistry 属性的 knownMappers` 中，以 `接口的Class 对象`为 `key` ,`new MapperProxyFactory<>(type)`  为 `value` , `type`为 接口的` Class对象`。

               2. 解析 方法上的`sql` 语句注解，将注解上配置的属性进行初始化 并赋值给 `configruation 对象的 MappedStatement`。

`package` 与 `mapper` 的主要不同点主要在于 `package` 扫描包下所有的接口进行解析，而 `mapper` 是扫描路径下所有的 `xml` 文件 将每一个命名空间 进行注册。 `mapper` 在注册完了以后会解析 `xml 中的 sql` 标签、 `cache` 标签、`ResultMap` 标签进行解析与存储。具体代码在 `org.apache.ibatis.builder.xml.XMLMapperBuilder#parse 中`。

​		到此 `mybatis` 的配置都已经解析完了。我们得到了一个丰富的 `configuration` 对象，并利用这个对象初始化了一个 `DefaultSqlSessionFactory` 对象。

## 源码分析之数据库会话SqlSession

​		一个数据库会话的开始，源于 `SqlSessionFactory` 的 `openSession` ,这里面会做那些事情呢？插件是如何执行的，二级缓存又是如何做到的？

​		`openSession` 首先会在配置 `configuration` 中获取到 数据库配置`Environment`，再根据环境配置 `Environment` 获取到对应的事务工厂，如果没有则进行创建默认的 `ManagedTransactionFactory` 也就是说，你在配置文件中的 `Environments`单个实例中配置了 `transactionManager` 事务工厂就用你提供的，如果没有配置 就用`ManagedTransactionFactory`。通过 该事务工厂拿到 一个 `Transaction` 事务 ，通过该事务拿到创建一个 `Executor`(用于 `sql` 执行与调度)。

​		针对于 `Mybatis` 核心 `Executor` 的创建，有必要深入进行了解。`Executor` 的创建主要分为创建 对象、二级缓存封装、插件封装 三个步骤。

​		第一步创建对象逻辑比较简单，主要根据 配置文件 `setting 中的 defaultExecutorType` 指定的 `Executor Type` 创建不同的对象实例。`Executor` 主要分为三种类型，第一种 `Simpla`

