# Spring源码阅读

## ApplicationContext构造函数 ioc 入口

使用策略模式，分为以下实现类：

* ClassPathXmlApplicationContext 基于 xml配置 ioc 容器 

* XmlWebApplicationContext 基于 web 注解配置的 ioc 容器

* AnnotationConfigApplicationContext 基于注解配置的 ioc 容器

下面已	ClassPathXmlApplicationContext 主体分析 IoC容器

```java
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent){
		//初始化资源加载器
		super(parent);
		//设置资源文件路径
		setConfigLocations(configLocations);
		if (refresh) {
			//ioc主入口 
			refresh();
		}
	}
```

首先是初始化配置文件资源加载器，并设置父容器

```java
public AbstractApplicationContext(@Nullable ApplicationContext parent) {
		this();// ==> this.resourcePatternResolver = getResourcePatternResolver();
		setParent(parent);
	}
```

```java
	//获取一个Spring Source的加载器用于读入Spring Bean定义资源文件
	protected ResourcePatternResolver getResourcePatternResolver() {
		//AbstractApplicationContext继承DefaultResourceLoader，因此也是一个资源加载器
		//Spring资源加载器，其getResource(String location)方法用于载入资源
		return new PathMatchingResourcePatternResolver(this);
	}
```

初始完类加载器以后便是设置 配置文件路径,配置文件名称储存到 configLocations 中

```java
	this.configLocations = new String[locations.length];
	for (int i = 0; i < locations.length; i++) {
		// resolvePath为同一个类中将字符串解析为路径的方法
		this.configLocations[i] = resolvePath(locations[i]).trim();
	}
```

	当以上配置都配好了以后我们就可以去查看 ioc 真正做事情的方法了 refresh(),这个方法调用了 父类的 AbstractApplicationContext refresh()方法

主要代码实现如下：

```java
synchronized (this.startupShutdownMonitor) {
	ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
}
```
这个方法返回一个 beanFactory工厂
其他方法暂时不去深究，主要关注主线方法。让我们查看 obtainFreshBeanFactory() 看看它到底做了什么!然后我们进入到了 AbstractApplicationContext 类中，

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		//这里使用了委派设计模式，父类定义了抽象的refreshBeanFactory()方法，具体实现调用子类容器的refreshBeanFactory()方法
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}
```
	
		这个方法调用了 父类 refreshBeanFactory() 和 getBeanFactory() 方法，我们点击去发现是两个抽象方法，所以我们去查看他的子类实现，
	由于 ClassPathXmlApplicationContext	继承了 AbstractRefreshableApplicationContext 所以我们进入到这个类里面查看这两个方法。

**这里应用了委派模式，父类 AbstractApplicationContext 调用了子类的 refreshBeanFactory()和getBeanFactory()这两个方法**

**委派模式定义: 只看结果，不看过程。这里父类调用了子类的方法，只关注返回结果，不关注过程**

```java
	//创建IOC容器
	DefaultListableBeanFactory beanFactory = createBeanFactory();
	//对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
	customizeBeanFactory(beanFactory);
	//
	loadBeanDefinitions(beanFactory);
```

		我们继续往下看会看到子类创建了一个 beanFactory 工厂，这个bean 工厂 DefaultListableBeanFactory 才是之后的真正创建 bean 对象类。
	然后我们去查看 loadBeanDefinitions() 这个方法，结果发现又是一个抽象方法，然后我们去看他的子类，由于我们通过 ClassPathXmlApplicationContext
	进入的，他的父类为 AbstractXmlApplicationContext，所以我们去查看 AbstractXmlApplicationContext 的实现。

**调用loadBeanDefinitions()也应用到了委派模式**

```java
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
	loadBeanDefinitions(beanDefinitionReader);
```

		这里首先将 beanFactory 工厂设置到了 beanDefinitionReader 对象了，然后经过一系列的配置以后最终调用了 loadBeanDefinitions(方法)。
	beanDefinitionReader对象是将 配置文件中的 javaBean 解析成 BeanDefinition。

**现在的 beanFactory 对象已经存储到了 beanDefinitionReader 对象中，可以通过 beanDefinitionReader 的 getBeanFactory()和getRegistry()拿到**

		我们深入查看一下 loadBeanDefinitions(beanDefinitionReader)到底做了什么，代码如下：

```java
	reader.loadBeanDefinitions(configResources);
```

	这个方法调用了 beanDefinitionReader 对象的 loadBeanDefinitions()方法，将操作交给 beanDefinitionReader对象完成

	然后我又去查看了 beanDefinitionReader.loadBeanDefinitions()方法。于是我看到了以下代码：

```java
	for (Resource resource : resources) {
			counter += loadBeanDefinitions(resource);
		}
```

	这段代码就是将用户传入的 spring 配置文件，通过调用loadBeanDefinitions() 进行后续操作。

		然后我们继续往下看  loadBeanDefinitions(resource) 发现这又是一个抽象方法，所以我继续查看其子类实现，然后我们进入到了XmlBeanDefinitionReader中
	查看。

```java
	InputStream inputStream = encodedResource.getResource().getInputStream();
	try {
		//从InputStream中得到XML的解析源
		InputSource inputSource = new InputSource(inputStream);
		if (encodedResource.getEncoding() != null) {
			inputSource.setEncoding(encodedResource.getEncoding());
		}
		//这里是具体的读取过程
		return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
	}
	finally {
		//关闭从Resource中得到的IO流
		inputStream.close();
	}
```

这里是将文件路径转成 io 流 然后在进行操作，那么我们继续往下看：

```java
	//将XML文件转换为DOM对象，解析过程由documentLoader实现
	Document doc = doLoadDocument(inputSource, resource);
	//这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
	return registerBeanDefinitions(doc, resource);
```
这个代码将 io 流封装成 Document对象，通过这个对象读取 xml 配置信息

我们需要关注以下 registerBeanDefinitions() 方法

```java
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	//这里的 createReaderContext() 方法执行
	// return new XmlReaderContext(resource, this.problemReporter, this.eventListener,this.sourceExtractor, this, getNamespaceHandlerResolver());
	//将当前的 xmlDefinitionReader 对象传入 documentReader对象中 然后保存起来
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
```

这里是将当前 xmlDefinitionReader 对象 保存到 documentReader 对象中，然后调用 DefaultBeanDefinitionDocumentReader的 doRegisterBeanDefinitions 方法

```java
	//在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
		preProcessXml(root);
		//从Document的根元素开始进行Bean定义的Document对象
		parseBeanDefinitions(root, this.delegate);
		//在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
		postProcessXml(root);
```

doRegisterBeanDefinitions 对解析后的 xml 对象进行一系列的解析然后调用 parseBeanDefinitions(root, this.delegate) 方法

下面就是循环 xml 的对象一层层的获取节点 然后调用 parseDefaultElement(ele, delegate) 方法， parseDefaultElement() 又调用了processBeanDefinition()

主要代码如下

```java
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
```

		这段代码首先是 先将 bean 封装成 beanDefinition 再将 beanDefinition 封装到 BeanDefinitionHolder 中，
	最后将 bdHolder对象 注册到 beanFactory 中 这里通过 getReaderContext().getRegistry() 获取 beanFactory。
	这里的 readerContext 是在之前调用的 documentReader.registerBeanDefinitions(doc, createReaderContext(resource))
	的时候将 XmlBeanDefinitionReader 对象封装到当前 DefaultBeanDefinitionDocumentReader中的，而 beanFactory 是在
	XmlBeanDefinitionReader 初始化的时候封装到的 XmlBeanDefinitionReader中的（之前提到过）。 然后调用 
	BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry())方法将 beanfinition 
	注册到 beanfactory 中。注册就是一个 将beanDefinition 存储到 map 中 

```java
	this.beanDefinitionMap.put(beanName, beanDefinition);
``` 

# 总结
		spring ioc 主要通过初始化 ApplicationContext 的子类为入口，一步步进行加载，applicationContext 子类通过策略模式，
	进行了不同 spring 入口的实现，然后通过 ClassPathApplicationContext 进行深入查看，构造方法调用了父类 AbstractApplicationContext
	通过父类的定义好的逻辑 定义 beanFactory 工厂，这个用到了 工厂模式以及单例模式，这个工厂用于注册 beanDefiniton 也就是
	bean 的定义,创建好 beanfactory  之后就是如何去解析文件如何封装成 deafinition 。
		加载 beanDefinition 需要用到 beanDefinitionReader 这里也用到了的策略模式，以及所生成的 beanDefinition 也用到了
	策略模式，同过用户自己选择的 context 来决定了 这两个对象。
