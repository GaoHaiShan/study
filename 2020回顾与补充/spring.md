# spring大致功能

## 架构设计

## IoC

### 配置加载-StandardEnvironment

![image-20201228131516809](pic\image-20201228131516809.png)

![image-20210105133431295](E:\haishanStudy\study\2020回顾与补充\pic\image-20210105133431295.png)

### ClassPathXmlApplicationContext 

​	提供程序入口。

​	若使用 以下构造方法创建，则将当前配置资源保存到`ClassPathXmlApplicationContext`类中，并刷新容器

```java
public ClassPathXmlApplicationContext(String[] paths, Class<?> clazz, @Nullable ApplicationContext parent)
			throws BeansException {

    super(parent);
    Assert.notNull(paths, "Path array must not be null");
    Assert.notNull(clazz, "Class argument must not be null");
    this.configResources = new Resource[paths.length];
    for (int i = 0; i < paths.length; i++) {
        this.configResources[i] = new ClassPathResource(paths[i], clazz);
    }
    refresh();
}
```

### AbstractXmlApplicationContext

通过与 XmlBeanDefinitionReader 协作，提供 xml 解析与bean 注册功能实现

`AbstractXmlApplicationContext`应用`XmlBeanDefinedReader` 解析xml 配置，以及通过`XmlBeanDefinedReader` 将bean 以 XmlBeanDefinition形式 注册进入Beanfactory

### AbstractRefreshableConfigApplicationContext

提供配置资源存储功能

```java
@Nullable
private String[] configLocations;
```

### AbstractRefreshableApplicationContext

提供BeanFactory的创建、存储、操作、配置功能，

调用子类`AbstractXmlApplicationContext#loadBeanDefinitions`方法进行配置解析，以及bean 注册

### AbstractApplicationContext

环境变量、事件发布、监听事件处理、容器刷新合并

### Xml配置IoC启动流程

程序写法

```java
public class Application {

    public static void main(String[] args) {
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");
        HelloWord helloWord = (HelloWord) applicationContext.getBean("helloWord");
        helloWord.say();
    }
}
```

配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean class="com.edi.oldspring.Person" id="person">
        <constructor-arg name="name" value="海山"></constructor-arg>
    </bean>
    <bean class="com.edi.oldspring.HelloWord" id="helloWord">
        <constructor-arg name="person" ref="person"></constructor-arg>
    </bean>
</beans>
```

​	

### 大致流程

1. `ClassPathXmlApplicationContext 将配置文件保存到父类 AbstractRefreshableConfigApplicationContext 中，用于之后的配置解析。`

2. `ClassPathXmlApplicationContext 调用父类 AbstractApplicationContext#refresh() 方法刷新IoC容器。`

3. `AbstractApplicationContext#refresh() 调用 AbstractApplicationContext#obtainFreshBeanFactory() 重新获取beanFactory IoC容器。`

4. `AbstractApplicationContext#obtainFreshBeanFactory() 调用子类 AbstractRefreshableApplicationContext#refreshBeanFactory()` 
   `进行容器刷新`

5. `AbstractRefreshableApplicationContext#refreshBeanFactory() 方法，首先判断是否存在 beanFactory 容器，存在则进行销毁`
   `然后调用自身的 createBeanFactory() 方法进行新的IoC容器创建，并设置新容器 allowBeanDefinitionOverriding(如果出现相同的 bean 名称，`
   `是否允许被覆盖)`
   `以及 allowCircularReferences(是否允许循环依赖) 属性。最后调用子类 AbstractXmlApplicationContext#loadBeanDefinitions()` 
   `方法进行配置文件解析，以及bean装载。`

6. `AbstractXmlApplicationContext#loadBeanDefinitions() 首先初始化 XmlBeanDefinitionReader 对象，并为其设置IoC容器`
   `(第五部新创建的 beanFactory)`
   `当前对象引用(当前 ClassPathXmlApplicationContext 对象)、资源路径解析器`
   `最后调用 AbstractXmlApplicationContext#loadBeanDefinitions(XmlBeanDefinitionReader)重载方法，进行配置`

7. `AbstractXmlApplicationContext#loadBeanDefinitions(XmlBeanDefinitionReader) 方法调用   
     AbstractBeanDefinitionReader#loadBeanDefinitions()`
   `将文件解析与注册的任务委派给 XmlBeanDefinitionReader 的父类 AbstractBeanDefinitionReader 执行。`
8. `AbstractBeanDefinitionReader#loadBeanDefinitions()，循环配置文件 调用子类 XmlBeanDefinitionReader#loadBeanDefinitions()进行解析。`

9. `XmlBeanDefinitionReader#loadBeanDefinitions() 将配置文件转成 InputStream 流，然后调用自身的 doLoadBeanDefinitions() 进行配置文件读取`

10. `XmlBeanDefinitionReader#doLoadBeanDefinitions() 读取配置文件 spring.xml所有内容，并将内容转化成 Document 对象。 并调用自身    
    registerBeanDefinitions() 进行注册`

11. `XmlBeanDefinitionReader#registerBeanDefinitions() 创建 BeanDefinitionDocumentReader 对象，并调用`
    `BeanDefinitionDocumentReader#registerBeanDefinitions()进行bean节点解析与注册`

12. `BeanDefinitionDocumentReader#registerBeanDefinitions() 首先从 Document 中读取 <beans> 整个大标签，将其转化成 Element 对象`

13. `BeanDefinitionDocumentReader#registerBeanDefinitions() 调用自身doRegisterBeanDefinitions() 进行bean 标签的解析`

14. `doRegisterBeanDefinitions() 调用自身的 parseBeanDefinitions() 将 bean或其他标签进行解析。`

15. `parseBeanDefinitions() 循环 Element 所有节点，如果节点属于 http://www.springframework.org/schema/beans 里面则调用自身默认的` 
    `parseDefaultElement() 方法解析`
    `否则调用自定义的解析器进行解析 delegate.parseCustomElement(ele);`

16. `BeanDefinitionDocumentReader#parseDefaultElement() 首先判断节点为 <import> 还是 <alias> 或者 <bean> 或者 <beans>`
    `（其他暂时不做了解 主要看 bean 的逻辑）`
    `当标签为bean 时，会调用自身的BeanDefinitionDocumentReader#processBeanDefinition() 进行 Bean 注册`

17. `BeanDefinitionDocumentReader#processBeanDefinition() 首先将 <bean> 标签转换成 BeanDefinitionHolder 对象，`
    `然后调用 BeanDefinitionReaderUtils#registerBeanDefinition() 静态方法，将 BeanDefinitionHolder 对象注册到 第五步初始化的beanFactory IoC`
    `容器中`

18. `最后由ioc容器BeanFactory 将 BeanDefinitionHolder 对象保存在 beanDefinitionMap 属性里面 并将beanName 保存到 beanDefinitionNames 中`

## DI

## AOP

## MVC

## JDBC

## 事件监听

​	