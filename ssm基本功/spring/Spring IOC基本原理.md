# 前言

> 常见的概念就不一一介绍(例如一些什么注入方式,官方文档里面都是有的,文档最后会给出链接的),这里我们抓主干,上帝视角把这IOC原理简单的摸摸清

# 基本概念

通过官方的一张高层视图,很容易理解:通过配置(注解/xml形式)容器帮我们负责创建对象,我们只需要负责get,然后做就行了;相当于说容器做了一层解耦:剥离了对象实例的创建销毁过程和调用的过程,调用者只需要关心对象本身就行了,容器会帮我们管理好对象的存亡

![容器高层原理图](assets/Untitled/container-magic-20191230160543810.png)

![image-20191231104332673](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20191231104332673.png)



Spring启动时读取应用程序提供的Bean配置信息,并在Spring容器中生成一份相应的Bean配置注册表,然后根据这张注册表实例化 Bean,装配好Bean之间的依赖关系,为上层应用提供准 备就绪的运行环境.其中Bean缓存池为HashMap实现

# 核心组成

## BeanDefinition

> 是Spring内部的一个接口,定义了对Bean的基本规范(beanClassName、scope、lazyInit等),对象在容器中的抽象;

| Property                 | Explained in…                                                |
| :----------------------- | :----------------------------------------------------------- |
| Class                    | [Instantiating Beans](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-class) |
| Name                     | [Naming Beans](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-beanname) |
| Scope                    | [Bean Scopes](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-scopes) |
| Constructor arguments    | [Dependency Injection](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-collaborators) |
| Properties               | [Dependency Injection](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-collaborators) |
| Autowiring mode          | [Autowiring Collaborators](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-autowire) |
| Lazy initialization mode | [Lazy-initialized Beans](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-lazy-init) |
| Initialization method    | [Initialization Callbacks](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-lifecycle-initializingbean) |
| Destruction method       | [Destruction Callbacks](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-lifecycle-disposablebean) |

**官方文档给了如下说明:**

In addition to bean definitions that contain information on how to create a specific bean, the `ApplicationContext` implementations also permit the registration of existing objects that are created outside the container (by users). This is done by accessing the ApplicationContext’s BeanFactory through the `getBeanFactory()` method, which returns the BeanFactory `DefaultListableBeanFactory` implementation. `DefaultListableBeanFactory` supports this registration through the `registerSingleton(..)` and `registerBeanDefinition(..)` methods. However, typical applications work solely with beans defined through regular bean definition metadata.

**机翻:**

除了包含有关如何创建特定bean的信息的bean定义之外，ApplicationContext实现还允许注册在容器外部（由用户）创建的现有对象。这是通过通过方法访问ApplicationContext的BeanFactory来完成的getBeanFactory()，该方法返回BeanFactory DefaultListableBeanFactory实现。DefaultListableBeanFactory 通过registerSingleton(..)和 registerBeanDefinition(..)方法支持此注册。但是，典型的应用程序只能与通过常规bean定义元数据定义的bean一起使用。

**简而言之**

BeanDefinition这个东西怎么玩呢,通过getBeanFactory().registerBeanDefinition()方式进行注册(就是put)

## BeanFactory

> 位于类结构树的顶端,它最主要的方法就是getBean(String beanName),该方法从容器中返回特定名称的Bean,BeanFactory的功能通过其他的接口得到不断扩展

![image-20191231103355286](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20191231103355286.png)

## ApplicationContext

> ApplicationContext由BeanFactory派生而来,提供了更多面向实际应用的功能.ApplicationContext 继承了HierarchicalBeanFactory和ListableBeanFactory接口,在此基础上,还通过多个其他的接口扩展了BeanFactory的功能

![image-20191231104029993](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20191231104029993.png)

1. FileSystemXmlApplicationContext:默认从文件系统中装载配置文件

2. ApplicationEventPublisher:让容器拥有发布应用上下文事件的功能,包括容器启动事

   件、关闭事件等。

3. MessageSource:为应用提供 i18n 国际化消息访问的功能;

4. ResourcePatternResolver:所有ApplicationContext实现类都实现了类似于

   PathMatchingResourcePatternResolver 的功能,可以通过带前缀的 Ant 风格的资源文

   件路径装载 Spring 的配置文件。

5. LifeCycle:该接口是 Spring 2.0 加入的,该接口提供了 start()和 stop()两个方法，主要

   用于控制异步处理过程。在具体使用时该接口同时被 ApplicationContext 实现及具体 Bean 实现， ApplicationContext 会将 start/stop 的信息传递给容器中所有实现了该接 口的 Bean，以达到管理和控制 JMX、任务调度等目的。

6. ConfigurableApplicationContext 扩展于 ApplicationContext，它新增加了两个主要 的方法: refresh()和 close()，让 ApplicationContext 具有启动、刷新和关闭应用上下文的能力。在应用上下文关闭的情况下调用 refresh()即可启动应用上下文,在已经启动的状态下,调用 refresh()则清除缓存并重新装载配置信息,而调用close()则可关闭应用上下文。

# 原理分析

有了前面的基本概念的介绍,IOC的三板斧

- bean配置信息封装到BeanDefinition,并塞进注册表,怎么塞？通过getBeanFactory().registerBeanDefinition()
- 根据注册表信息解析并实例化bean放进Map缓存里面
- 调用者getBean 获取对应的bean

所以弄清上面三个步骤,大致原理我们就门清了

### demo示例 很简单

java代码

```java
@Setter
@Getter
public class AliasDemo {
    private String content;
}
// test方法
@Test
public void testAlias() {
  String configLocation = "application-alias.xml";
  ApplicationContext applicationContext = new ClassPathXmlApplicationContext(configLocation);
  System.out.println("       alias-hello -> " + applicationContext.getBean("alias-hello"));
}
```

xml配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd"
       default-lazy-init="true">
    <context:component-scan base-package="com.zhangpeng.**"/>
    <bean id="hello" class="com.zhangpeng.study.ioc.alias.AliasDemo">
        <property name="content" value="hello"/>
    </bean>
    <alias name="hello" alias="alias-hello"/>
</beans>

```

## debug

### 1. IOC配置读取

```java
public class ClassPathXmlApplicationContext extends AbstractXmlApplicationContext {
@Nullable
	private Resource[] configResources;
	...省略部分源码...
	// 这是我们上面例子跳转进去的构造,传入一个配置路径
	public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}
	// 看看重载的构造方法里面有个setConfigLocations,
	// 那肯定就是设置配置路径了
	// 还有个refresh方法
	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
}
```

setConfigLocations主要就是配置一些路径

### 2. 核心refresh方法

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
  synchronized (this.startupShutdownMonitor) {
    // Prepare this context for refreshing.
    prepareRefresh();

    // Tell the subclass to refresh the internal bean factory.
    // 1.获取全新的BeanFactory实例
    ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

    // Prepare the bean factory for use in this context.
    prepareBeanFactory(beanFactory);

    try {
      // Allows post-processing of the bean factory in context subclasses.
      postProcessBeanFactory(beanFactory);

      // Invoke factory processors registered as beans in the context.
      invokeBeanFactoryPostProcessors(beanFactory);

      // Register bean processors that intercept bean creation.
      registerBeanPostProcessors(beanFactory);

      // Initialize message source for this context.
      initMessageSource();

      // Initialize event multicaster for this context.
      initApplicationEventMulticaster();

      // Initialize other special beans in specific context subclasses.
      onRefresh();

      // Check for listener beans and register them.
      registerListeners();

      // Instantiate all remaining (non-lazy-init) singletons.
      finishBeanFactoryInitialization(beanFactory);

      // Last step: publish corresponding event.
      finishRefresh();
    }

    catch (BeansException ex) {
      if (logger.isWarnEnabled()) {
        logger.warn("Exception encountered during context initialization - " +
                    "cancelling refresh attempt: " + ex);
      }

      // Destroy already created singletons to avoid dangling resources.
      destroyBeans();

      // Reset 'active' flag.
      cancelRefresh(ex);

      // Propagate exception to caller.
      throw ex;
    }

    finally {
      // Reset common introspection caches in Spring's core, since we
      // might not ever need metadata for singleton beans anymore...
      resetCommonCaches();
    }
  }
}
```

#### 2.1 obtainFreshBeanFactory()

```java
@Override
protected final void refreshBeanFactory() throws BeansException {
  if (hasBeanFactory()) {
    destroyBeans();
    closeBeanFactory();
  }
  try {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
      this.beanFactory = beanFactory;
    }
  }
  catch (IOException ex) {
    throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
  }
}
```

方法名起的很好,很好理解

- 进来先销毁beanFactory 并且创建一个DefaultListableBeanFactory 实例
- customizeBeanFactory 自定义一些属性(是否支持BeanDefinition可覆盖、是否支持循环依赖等)

接着进入loadBeanDefinitions

```java
//org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)
@Override
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
  // Create a new XmlBeanDefinitionReader for the given BeanFactory.
  XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

  // Configure the bean definition reader with this context's
  // resource loading environment.
  beanDefinitionReader.setEnvironment(this.getEnvironment());
  beanDefinitionReader.setResourceLoader(this);
  beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

  // Allow a subclass to provide custom initialization of the reader,
  // then proceed with actually loading the bean definitions.
  initBeanDefinitionReader(beanDefinitionReader);
  loadBeanDefinitions(beanDefinitionReader);
}
```

然后走到doLoadBeanDefinitions,方法名起的很顶啊 xx -> doXx

```java
// 接着看下具体解析的地方
/**
	 * Actually load bean definitions from the specified XML file.
	 * @param inputSource the SAX InputSource to read from
	 * @param resource the resource descriptor for the XML file
	 * @return the number of bean definitions found
	 * @throws BeanDefinitionStoreException in case of loading or parsing errors
	 * @see #doLoadDocument
	 * @see #registerBeanDefinitions
	 */
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
  throws BeanDefinitionStoreException {
 // ...省略...
  try {
    Document doc = doLoadDocument(inputSource, resource);
    int count = registerBeanDefinitions(doc, resource);
    if (logger.isDebugEnabled()) {
      logger.debug("Loaded " + count + " bean definitions from " + resource);
    }
    return count;
  }
	// ...省略...
}
```

接着看 registerBeanDefinitions

```java
//org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#registerBeanDefinitions
	/**
	 * This implementation parses bean definitions according to the "spring-beans" XSD
	 * (or DTD, historically).
	 * <p>Opens a DOM Document; then initializes the default settings
	 * specified at the {@code <beans/>} level; then parses the contained bean definitions.
	 */
	@Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
//org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions
		/**
	 * Register each bean definition within the given root {@code <beans/>} element.
	 */
	@SuppressWarnings("deprecation")  // for Environment.acceptsProfiles(String...)
	protected void doRegisterBeanDefinitions(Element root) {
		// Any nested <beans> elements will cause recursion in this method. In
		// order to propagate and preserve <beans> default-* attributes correctly,
		// keep track of the current (parent) delegate, which may be null. Create
		// the new (child) delegate with a reference to the parent for fallback purposes,
		// then ultimately reset this.delegate back to its original (parent) reference.
		// this behavior emulates a stack of delegates without actually necessitating one.
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
	  // ...省略...
    // 标准开头pre前置
		preProcessXml(root);
    // 解析doc
		parseBeanDefinitions(root, this.delegate);
    // 标准结尾post后置
		postProcessXml(root);
		this.delegate = parent;
	}
```

我擦又来一波doXX,厉害了

![spring你个老司机](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/d119e938f9af6144471e271153a8b2b3.jpg)



最后看下parseBeanDefinitions 不展开具体解析细节了 我们的重点是主原理

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
  if (delegate.isDefaultNamespace(root)) {
    NodeList nl = root.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
      Node node = nl.item(i);
      if (node instanceof Element) {
        Element ele = (Element) node;
        if (delegate.isDefaultNamespace(ele)) {
          parseDefaultElement(ele, delegate);
        }
        else {
          delegate.parseCustomElement(ele);
        }
      }
    }
  }
  else {
    delegate.parseCustomElement(root);
  }
}
 // 解析默认元素
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
// 最终实际解析BeanDefinition的地方
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
  // 这一步采用BeanDefinitionHolder进一步封装BeanDefinition数据
  BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  if (bdHolder != null) {
    bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
    try {
      // Register the final decorated instance.
      // 注册最终的装饰实例
      BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
    }
    catch (BeanDefinitionStoreException ex) {
      getReaderContext().error("Failed to register bean definition with name '" +
                               bdHolder.getBeanName() + "'", ele, ex);
    }
    // Send registration event.
    getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
  }
}
```

到这一步BeanDefinition诞生了并且被封装在了BeanDefinitionHolder,可以回答IOC三板斧的第一步的前半个问题,BeanDefinition由配置解析的过程

既然BeanDefinition有了就该注册,接着上面的代码继续走

```java
	/**
	 * Register the given bean definition with the given bean factory.
	 * @param definitionHolder the bean definition including name and aliases
	 * @param registry the bean factory to register with
	 * @throws BeanDefinitionStoreException if registration failed
	 */
// 通过指定的bean factory注册,默认就是DefaultListableBeanFactory 
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
// org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition
@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		if (existingDefinition != null) {
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionOverrideException(beanName, beanDefinition, existingDefinition);
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (logger.isInfoEnabled()) {
					logger.info("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				synchronized (this.beanDefinitionMap) {
					this.beanDefinitionMap.put(beanName, beanDefinition);
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;
					removeManualSingletonName(beanName);
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				removeManualSingletonName(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```

上面这段就是DefaultListableBeanFactory将BeanDefinition塞入map的过程

#### 战略性总结

> obtainFreshBeanFactory方法的获取过程中,加载解析BeanDefinition,将BeanDefinition封装到BeanDefinitionHolder,最后再由DefaultListableBeanFactory注册BeanDefinition到map

![1577785049585](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/1577785049585.jpg)



有的老铁

![image-20191231173442143](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20191231173442143.png)

### 3.getBean

又是xx & doXx方式

```java
public <T> T getBean(String name, @Nullable Class<T> requiredType, @Nullable Object... args)
throws BeansException {
	return doGetBean(name, requiredType, args, false);
}

protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
		// 1.移除&开头字符获取FactoryBean本身 2.如果是别名将其转换为具体实例名
		final String beanName = transformedBeanName(name);
		Object bean;

		// 获取早期缓存,是否实例化过等价于<=>map.get(beanName) map结构<beanName,bean>
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
      // 不为空就直接返回
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
      // BeanFactory不缓存Prototype类型的bean,处理不了该类型bean的循环依赖问题
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
      // 从父容器查找bean实例
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
        // 根据有无参数进入对应的getBean重载方法
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
        // 合并父子BeanDefinition
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);
				
				// 检查是否有依赖的bean
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
            // 循环依赖校验
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
            // 注册依赖的bean信息
						registerDependentBean(dep, beanName);
						try {
              // 加载依赖的bean
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 创建bean实例
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
          // 如果bean是FactoryBean类型,则调用工厂方法获取真正的bean实例,否则直接返回bean实例
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
				// 创建prototype类型的bean
				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}
				// 创建其他类型的bean
				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 检查是否需要进行类型转换,需要则转换并且返回
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

将上面的主逻辑收敛一下

![image-20200102165520545](assets/Spring%20IOC%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20200102165520545.png)



- 1.beanName的转换
- 2.从缓存中获取bean
- 3.如果实例不为空&空参构造就调用 getObjectForBeanInstance立即返回
- 4.不满足3的条件,通过寻找父容器的BeanFactory 查找bean

  不存在通过直接返回父容器找到对应的bean
- 5.存在该bean,合并父子BeanDefinition
- 6.处理depends-on依赖的bean,注册&创建
- 7.处理FactoryBean (同4的流程)
- 8.如果需要类型转换则转换 最终返回bean

#### 3.1 beanName的转换

```java
	protected String transformedBeanName(String name) {
		return canonicalName(BeanFactoryUtils.transformedBeanName(name));
	}
  // 剔除&打头 方便获取factoryBean自身而不是其具体实现的对象
	public static String transformedBeanName(String name) {
		Assert.notNull(name, "'name' must not be null");
		if (!name.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
			return name;
		}
		return transformedBeanNameCache.computeIfAbsent(name, beanName -> {
			do {
				beanName = beanName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
			}
			while (beanName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX));
			return beanName;
		});
	}
	// 别名转换 因为map不支持<aliasName,Bean> 存储方式
	public String canonicalName(String name) {
		String canonicalName = name;
		// Handle aliasing...
		String resolvedName;
		do {
			resolvedName = this.aliasMap.get(canonicalName);
			if (resolvedName != null) {
				canonicalName = resolvedName;
			}
		}
		while (resolvedName != null);
		return canonicalName;
	}
```

关于&的使用 官方文档里面提到了 有兴趣的同学可以去看看

```java
When you need to ask a container for an actual FactoryBean instance itself instead of the bean it produces, preface the bean’s id with the ampersand symbol (&) when calling the getBean() method of the ApplicationContext. So, for a given FactoryBean with an id of myBean, invoking getBean("myBean") on the container returns the product of the FactoryBean, whereas invoking getBean("&myBean") returns the FactoryBean instance itself.
```

#### 3.2缓存获取实例

```java
	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		Object singletonObject = this.singletonObjects.get(beanName);
		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			synchronized (this.singletonObjects) {
				// 主要是为了防止重复依赖
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					// 当某些方法需要提前初始化的时候会调用addSingletionFactory 方法将对应的singletonFactory存储在singletonFactories
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						// 记录在缓存中,earlySingletonObjects 和 singletonFactories互斥
						singletonObject = singletonFactory.getObject();
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
	/** Cache of singleton objects: bean name to bean instance. */
	// 直接可以使用的对象
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
	/** Cache of singleton factories: bean name to ObjectFactory. */
	// 用于存放bean工厂,bean工厂所产生的bean是还未完成初始化的bean,bean工厂所生成的对象最终会被缓存到 earlySingletonObjects中
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
	// 早期的对象引用(还在初始化中的bean)
	/** Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

这段代码涉及到循环依赖的处理,后面会单独写一篇来阐述

#### 3.3 getObjectForBeanInstance返回

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {
		// 是否以&开头，是的话就获取FactoryBean实例
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
      // 不是FactoryBean类型,报错
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}
		// name不是以&开头并且beanInstance不是FactoryBean，说明当前bean是一个普通bean，返回
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
      // 从缓存获取
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
        // 合并beanDefinition
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
      // 从FactoryBean里获取返回
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}

// =========================

protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
  if (factory.isSingleton() && containsSingleton(beanName)) {
    synchronized (getSingletonMutex()) {
      Object object = this.factoryBeanObjectCache.get(beanName);
      if (object == null) {
        // 真正调用了factory.getObject()的地方
        object = doGetObjectFromFactoryBean(factory, beanName);
        Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
        if (alreadyThere != null) {
          object = alreadyThere;
        }
        else {
          // 是否应用后置处理器
          if (shouldPostProcess) {
            if (isSingletonCurrentlyInCreation(beanName)) {
              // Temporarily return non-post-processed object, not storing it yet..
              return object;
            }
            beforeSingletonCreation(beanName);
            try {
              // 执行后置处理器
              object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
              throw new BeanCreationException(beanName,
                                              "Post-processing of FactoryBean's singleton object failed", ex);
            }
            finally {
              afterSingletonCreation(beanName);
            }
          }
          if (containsSingleton(beanName)) {
            // FactoryBean所创建的实例会被缓存在factoryBeanObjectCache中,供后续调用使用
            this.factoryBeanObjectCache.put(beanName, object);
          }
        }
      }
      return object;
    }
  }
  // 获取非单例类型的实例
  else {
    // 工厂类获取
    Object object = doGetObjectFromFactoryBean(factory, beanName);
    // 同上面的 是否需要进行后置处理
    if (shouldPostProcess) {
      try {
        object = postProcessObjectFromFactoryBean(object, beanName);
      }
      catch (Throwable ex) {
        throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
      }
    }
    return object;
  }
}
// =========================
private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
  throws BeanCreationException {

  Object object;
  try {
    // 安全校验
    if (System.getSecurityManager() != null) {
      // ...省略...
    }
    else {
      // 工厂方法生成实例
      object = factory.getObject();
    }
  }
  // ...省略...
  if (object == null) {
    if (isSingletonCurrentlyInCreation(beanName)) {
      throw new BeanCurrentlyInCreationException(
        beanName, "FactoryBean which is currently in creation returned null from getObject");
    }
    object = new NullBean();
  }
  return object;
}
```

捋一下上面的步骤,代码很长,逻辑很清晰

- 校验是否&打头的FactoryBean类型,是的话获取,不是的话报错
- 校验参数beanInstance不是FactoryBean,说明当前bean是一个普通bean,直接返回
- 通过缓存获取FactoryBean
- 针对单例类型的FactoryBean,未命中,则从FactoryBean.getObject()生成实例并缓存
- 针对非单例类型的FactoryBean,直接创建新实例,无需缓存
- 根据shouldPostProcess判断是否要执行对应的后置处理器

#### 3.4 通过寻找父容器的BeanFactory 查找bean

```java
// 获取父BeanFactory
BeanFactory parentBeanFactory = getParentBeanFactory();
// 校验是否不存在该bean
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
  				// Not found -> check parent.
				// 当前xml没找到beanName 对应的信息去parentBeanFactory找
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
}
```

#### 3.5 存在该bean,合并父子BeanDefinition

```java
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
protected RootBeanDefinition getMergedLocalBeanDefinition(String beanName) throws BeansException {
  // Quick check on the concurrent map first, with minimal locking.
  RootBeanDefinition mbd = this.mergedBeanDefinitions.get(beanName);
  if (mbd != null && !mbd.stale) {
  return mbd;
  }
  return getMergedBeanDefinition(beanName, getBeanDefinition(beanName));
}
protected RootBeanDefinition getMergedBeanDefinition(
			String beanName, BeanDefinition bd, @Nullable BeanDefinition containingBd)
			throws BeanDefinitionStoreException {

		synchronized (this.mergedBeanDefinitions) {
			RootBeanDefinition mbd = null;
			RootBeanDefinition previous = null;

			// Check with full lock now in order to enforce the same merged instance.
			if (containingBd == null) {
				mbd = this.mergedBeanDefinitions.get(beanName);
			}

			if (mbd == null || mbd.stale) {
				previous = mbd;
				mbd = null;
        // BeanDefinition升级RootBeanDefinition
				if (bd.getParentName() == null) {
					// Use copy of given root bean definition.
					if (bd instanceof RootBeanDefinition) {
						mbd = ((RootBeanDefinition) bd).cloneBeanDefinition();
					}
					else {
						mbd = new RootBeanDefinition(bd);
					}
				}
				else {
					// Child bean definition: needs to be merged with parent.
					BeanDefinition pbd;
					try {
						String parentBeanName = transformedBeanName(bd.getParentName());
            // 校验父beanName是否与子类beanName相同,相同parentBeanName肯定在map里
						if (!beanName.equals(parentBeanName)) {
              // 使用parentBeanName进行与parent的parent合并,以此类推
							pbd = getMergedBeanDefinition(parentBeanName);
						}
						else {
              // 校验是不是ConfigurableBeanFactory类型 不是就报错
							BeanFactory parent = getParentBeanFactory();
							if (parent instanceof ConfigurableBeanFactory) {
								pbd = ((ConfigurableBeanFactory) parent).getMergedBeanDefinition(parentBeanName);
							}
							else {
								throw new NoSuchBeanDefinitionException(parentBeanName,
										"Parent name '" + parentBeanName + "' is equal to bean name '" + beanName +
										"': cannot be resolved without an AbstractBeanFactory parent");
							}
						}
					}
					catch (NoSuchBeanDefinitionException ex) {
						throw new BeanDefinitionStoreException(bd.getResourceDescription(), beanName,
								"Could not resolve parent bean definition '" + bd.getParentName() + "'", ex);
					}
          // 深拷贝的形式 通过父BeanDefinition配置进行合并的BeanDefinition创建
					mbd = new RootBeanDefinition(pbd);
          // 用子BeanDefinition覆盖父BeanDefinition属性
					mbd.overrideFrom(bd);
				}

        // 配置默认为单例如果未指定
				if (!StringUtils.hasLength(mbd.getScope())) {
					mbd.setScope(RootBeanDefinition.SCOPE_SINGLETON);
				}

				if (containingBd != null && !containingBd.isSingleton() && mbd.isSingleton()) {
					mbd.setScope(containingBd.getScope());
				}

				// Cache the merged bean definition for the time being
				// (it might still get re-merged later on in order to pick up metadata changes)
				if (containingBd == null && isCacheBeanMetadata()) {
          // 缓存合并后的definition
					this.mergedBeanDefinitions.put(beanName, mbd);
				}
			}
			if (previous != null) {
				copyRelevantMergedBeanDefinitionCaches(previous, mbd);
			}
			return mbd;
		}
	}
```

- BeanDefinition升级RootBeanDefinition
- 校验父beanName是否与子类beanName相同,不同再次进行合并
- 校验是不是ConfigurableBeanFactory类型 不是就报错
- 做了那么多肯定要缓存起来 方便快速使用

#### 3.6 处理depends-on依赖的bean,注册&创建

```java
// Guarantee initialization of beans that the current bean depends on.
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
  // 递归实例化依赖
  for (String dep : dependsOn) {
    // 校验是否循环依赖,是就抛出异常
    if (isDependent(beanName, dep)) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                      "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
    }
    // 注册依赖
    registerDependentBean(dep, beanName);
    try {
      getBean(dep);
    }
    catch (NoSuchBeanDefinitionException ex) {
      throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                      "'" + beanName + "' depends on missing bean '" + dep + "'", ex);
    }
  }
}

/**
 * Register a dependent bean for the given bean,
 * to be destroyed before the given bean is destroyed.
 * @param beanName the name of the bean
 * @param dependentBeanName the name of the dependent bean
 */
public void registerDependentBean(String beanName, String dependentBeanName) {
  // 别名转换 因为map不支持<aliasName,Bean> 存储方式
  String canonicalName = canonicalName(beanName);
 
  synchronized (this.dependentBeanMap) {
    Set<String> dependentBeans =
      this.dependentBeanMap.computeIfAbsent(canonicalName, k -> new LinkedHashSet<>(8));
    if (!dependentBeans.add(dependentBeanName)) {
      return;
    }
  }
 // set结构进行去重
  synchronized (this.dependenciesForBeanMap) {
    Set<String> dependenciesForBean =
      this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName, k -> new LinkedHashSet<>(8));
    dependenciesForBean.add(canonicalName);
  }
}
```

#### 3.7 处理FactoryBean (同4的流程) 略

#### 3.8 如果需要类型转换则转换最终返回bean

代码过程很好理解 这里不做深入探究

#### 小结

getBean这段比较长,这个过程主要就回答了实例化bean并塞进缓存 最后返回使用

# 技术总结

- 从IOC原理源码分析中,xx&doXx的方法命名风格值得在工作中实践
- 逻辑抽取的套路值得参考

# END

- 看到最后算是给面子了
- 提前祝大家新年快乐

## 扩展阅读

[spring官方reference](https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/core.html#spring-core)