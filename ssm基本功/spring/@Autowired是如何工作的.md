# 从哪里入手

> 当然是从注释开始

![image-20191226144224431](assets/Untitled%201/image-20191226144224431.png)

根据源码注释我们易得 AutowiredAnnotationBeanPostProcessor是Autowired的实现类,xxxBeanPostProcessor这不就是后置处理器吗我记得在哪里见过

![0b6252b5c3804c0468b1e5e8c38c7e00](assets/Untitled%201/0b6252b5c3804c0468b1e5e8c38c7e00.jpg)

我记得有个refresh()的方法 里面有注册BeanPostProcessor的操作,走进去瞧瞧

```java
	@Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
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

![image-20191226152438448](assets/Untitled%201/image-20191226152438448.png)



## 战略断点1 registerBeanPostProcessors(beanFactory) 注册处理器

- 进入到org.springframework.context.support.PostProcessorRegistrationDelegate#registerBeanPostProcessors方法

  经过一顿优先级排序操作,不绕弯直接看注册了哪些BeanPostProocessors

  ![image-20191226153433012](assets/Untitled%201/image-20191226153433012.png)

  默认有一个CommonAnnotationBeanPostProcessor处理器也被注册进来

## 战略断点2 finishBeanFactoryInitialization(beanFactory)

![image-20191226153844913](assets/Untitled%201/image-20191226153844913.png)

继续往后走我们直接走到关键的doCreateBean方法

org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean

### 预解析过程

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {
    // 省略...
		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}
		// 省略...
	}
```

峰回路转 快走到调用的地方了

```java
	protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
      // 校验是否MergedBeanDefinitionPostProcessor类型
			if (bp instanceof MergedBeanDefinitionPostProcessor) {
				MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
				bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
			}
		}
	}
```

看下AutowiredAnnotationBeanPostProcessor继承关系 

![image-20191226154840165](assets/Untitled%201/image-20191226154840165.png)好了没问题继续往后走

跳转到org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition

```java
	@Override
	public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
		InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
		metadata.checkConfigMembers(beanDefinition);
	}
```

这里完成对注入元素注解的预解析

### 属性注入过程

> 按照写代码的步骤,创建对象=>依赖对象创建=>赋值=>打工

我们直接调到org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#populateBean

```java
	protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
		PropertyDescriptor[] filteredPds = null;
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
        // 这里会走到AutowiredAnnotationBeanPostProcessor#postProcessProperties
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
	}
```

为什么会走到AutowiredAnnotationBeanPostProcessor

![image-20191227104908107](assets/@Autowired%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84/image-20191227104908107.png)根据继承关系易得 AutowiredAnnotationBeanPostProcessor是InstantiationAwareBeanPostProcessor间接实现类

继续往下看 具体属性的注入过程

```java
@Override
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
  InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
  try {
    metadata.inject(bean, beanName, pvs);
  }
  // ... 省略
  return pvs;
}
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) throws Throwable {
  Collection<InjectedElement> checkedElements = this.checkedElements;
  Collection<InjectedElement> elementsToIterate =
    (checkedElements != null ? checkedElements : this.injectedElements);
  if (!elementsToIterate.isEmpty()) {
    for (InjectedElement element : elementsToIterate) {
      if (logger.isTraceEnabled()) {
        logger.trace("Processing injected element of bean '" + beanName + "': " + element);
      }
      element.inject(target, beanName, pvs);
    }
  }
}
```

![image-20191227110018451](assets/@Autowired%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84/image-20191227110018451.png)

InjectedElement这个包含了多个注入实现对象,给我们设计自己业务代码也有很大的参考,比如签署合同,不同合同要素的注入,可以参考这种设计风格

# 技术总结

> 首先看到类名就要想到这个东西在哪产生,在哪调用,然后就能一步步找到答案;正所谓授人以鱼不如授人以渔

- 1.从注解注释的实现类,联想到refresh方法入口
- 2.找到注册的地方registerBeanPostProcessors
- 3.找到调用的地方finishBeanFactoryInitialization
- 4.找到最后属性复制的地点populateBean

给出一张图小结一下

![image-20191227113226084](assets/@Autowired%E6%98%AF%E5%A6%82%E4%BD%95%E5%B7%A5%E4%BD%9C%E7%9A%84/image-20191227113226084.png)

# END

> 喜欢的可以一键三连,有问题欢迎在留言区评论

[原文git](https://github.com/ZhangDaFoYe/javaX)