# Dubbo SPI

## 简介

SPI 全称为 Service Provider Interface，是一种服务发现机制。SPI 的本质是将接口实现类的全限定名配置在文件中，并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。

## JDK SPI原理

### 先看示例

- 接口

  ```java
  public interface JavaSPI {
      void test();
  }
  ```

- 实现类

  ```java
  public class JavaAdapter1 implements JavaSPI {
      @Override
      public void test() {
          System.out.println("JavaAdapter1.test()");
      }
  }
  public class JavaAdapter2 implements JavaSPI {
      @Override
      public void test() {
          System.out.println("JavaAdapter2.test()");
      }
  }
  ```

- META-INF/services配置

  ![image-20191225143149648](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225143149648.png)

  

  ```java
  com.study.javax.spi.JavaAdapter1
  com.study.javax.spi.JavaAdapter2
  ```

  

- 测试类

  ```java
  public class JavaSPITest {
      @Test
      void test() {
          ServiceLoader<JavaSPI> serviceLoader = ServiceLoader.load(JavaSPI.class);
          System.out.println("Java SPI");
          serviceLoader.forEach(JavaSPI::test);
      }
  }
  ```

- 结果输出

  ```java
  Java SPI
  JavaAdapter1.test()
  JavaAdapter2.test()
  ```

因为我们配置了2个实现类,对应也会输出两个实现类的调用方法日志;

其实原理很明显了

![image-20191225143734615](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225143734615.png)

### 再看技术总结

- 调用仔去调用JavaSPI接口的test方法
- 查看一下META-INF/services下配置的实现类,配置哪个就调用哪个,没有配置的,就不会拿出来给你调用

关于java SPI原理解析,看下另一篇分析文章

好了 我们继续走我们的主题 dubbo spi

## dubbo SPI原理

Dubbo 改进了 JDK 标准的 SPI 的以下问题：

- JDK 标准的 SPI 会一次性实例化扩展点所有实现，如果有扩展实现初始化很耗时，但如果没用上也加载，会很浪费资源。
- 如果扩展点加载失败，连扩展点的名称都拿不到了。
- 增加了对扩展点 IoC 和 AOP 的支持，一个扩展点可以直接 setter 注入其它扩展点。

有了前面jdk SPI的用法,我们大概摸到了一些套路,接下来先看下dubbo spi是怎么玩的

### 先看示例

- 接口

  ```java
  @SPI
  public interface DubboSPI {
      void test();
  }
  ```

- 实现类

  ```java
  public class DubboAdapter1 implements DubboSPI {
      private String str1;
  
      // set/get 省略
      @Override
      public void test() {
          System.out.println("DubboAdapter1.test()");
      }
  }
  public class DubboAdapter2 implements DubboSPI {
      private String str2;
  
      // set/get 省略
      @Override
      public void test() {
          System.out.println("DubboAdapter2.test()");
      }
  }
  ```

- META-INF/dubbo配置

  ![image-20191225145344333](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225145344333.png)

   ```java
  dubboAdapter1 = com.study.javax.dubbospi.DubboAdapter1
  dubboAdapter2 = com.study.javax.dubbospi.DubboAdapter2
   ```

- 测试类

  ```java
  public class DubboSPITest {
      @Test
      void test() {
          ExtensionLoader<DubboSPI> extensionLoader = ExtensionLoader.getExtensionLoader(DubboSPI.class);
          System.out.println("Dubbo SPI");
          DubboSPI dubboAdapter1 = extensionLoader.getExtension("dubboAdapter1");
          dubboAdapter1.test();
          DubboSPI dubboAdapter2 = extensionLoader.getExtension("dubboAdapter2");
          dubboAdapter2.test();
      }
  }
  ```

- 结果输出

  ```java
  Dubbo SPI
  DubboAdapter1.test()
  DubboAdapter1.test()
  ```

  我们可以根据name 加载对应的实例做到按需加载

### 开始源码分析

#### 1.主方法入口 extensionLoader.getExtension

```java
		/**
     * 返回指定名字的扩展。如果指定名字的扩展不存在，则抛异常 {@link IllegalStateException}.
     *
     * @param name
     * @return
     */
	@SuppressWarnings("unchecked")
	public T getExtension(String name) {
		if (name == null || name.length() == 0)
		    throw new IllegalArgumentException("Extension name == null");
		if ("true".equals(name)) {
        // 默认扩展实现类
		    return getDefaultExtension();
		}
    // 接着就是经典的缓存模式 get => null判断 => 空再赋值
		Holder<Object> holder = cachedInstances.get(name);
		if (holder == null) {
		    cachedInstances.putIfAbsent(name, new Holder<Object>());
		    holder = cachedInstances.get(name);
		}
		Object instance = holder.get();
    // DCL 双检锁经典的单例模式 保证并发安全 知道的朋友就不用啰嗦了吧
		if (instance == null) {
		    synchronized (holder) {
	            instance = holder.get();
	            if (instance == null) {
                  // 创建实例并赋值到holer里面
	                instance = createExtension(name);
	                holder.set(instance);
	            }
	        }
		}
		return (T) instance;
	}
```

![image-20191225155450354](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225155450354.png)

上面一顿操作构造了一个空的holder 整个流程看起来还是蛮简单的 我们继续看下createExtension逻辑

#### 2.createExtension

```java
    @SuppressWarnings("unchecked")
    private T createExtension(String name) {
        // 1.从META-INF/dubbo加载所有扩展类
        Class<?> clazz = getExtensionClasses().get(name);
        if (clazz == null) {
            throw findException(name);
        }
        try {
          // 2.经典缓存模式 通过反射构造实例
            T instance = (T) EXTENSION_INSTANCES.get(clazz);
            if (instance == null) {
                EXTENSION_INSTANCES.putIfAbsent(clazz, (T) clazz.newInstance());
                instance = (T) EXTENSION_INSTANCES.get(clazz);
            }
          // 3.注入依赖
            injectExtension(instance);
          // 4.初次进来cachedWrapperClasses是空的 下面我打算 调用两遍看下这里干啥子用的
            Set<Class<?>> wrapperClasses = cachedWrapperClasses;
            if (wrapperClasses != null && wrapperClasses.size() > 0) {
              // 遍历创建Wrapper实例
                for (Class<?> wrapperClass : wrapperClasses) {
                    instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
                }
            }
            return instance;
        } catch (Throwable t) {
            throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                    type + ")  could not be instantiated: " + t.getMessage(), t);
        }
    }
```

##### 2.1.获取扩展类 得到一个clazz对象

```java
// 还是双检锁,dubbo真是双检锁狂魔	
private Map<String, Class<?>> getExtensionClasses() {
        Map<String, Class<?>> classes = cachedClasses.get();
        if (classes == null) {
            synchronized (cachedClasses) {
                classes = cachedClasses.get();
                if (classes == null) {
                  // 加载扩展类(1.解析SPI注解 2.加载配置文件)
                    classes = loadExtensionClasses();
                    cachedClasses.set(classes);
                }
            }
        }
        return classes;
	}
```

![image-20191225160634179](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225160634179.png)

##### 2.2 反射搞了个实例

![image-20191225164240789](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225164240789.png)

这会实例属性还没填充 我们继续看

##### 2.3 属性注入 or 灵魂注入

```java
    private T injectExtension(T instance) {
        try {
            if (objectFactory != null) {
              // 1.遍历目标类的方法
                for (Method method : instance.getClass().getMethods()) {
                  // 2.判断是否以set开头,并且入参个数为1,public修饰符 简单粗暴
                    if (method.getName().startsWith("set")
                            && method.getParameterTypes().length == 1
                            && Modifier.isPublic(method.getModifiers())) {
                       // 获取参数类型
                        Class<?> pt = method.getParameterTypes()[0];
                        try {
                          // 获取属性名,也是很直接直接截取字符串+小写转换
                            String property = method.getName().length() > 3 ? method.getName().substring(3, 4).toLowerCase() + method.getName().substring(4) : "";
                          // 3.获取对象
                            Object object = objectFactory.getExtension(pt, property);
                            if (object != null) {
                              // 反射调用setter方法设置依赖
                                method.invoke(instance, object);
                            }
                        } catch (Exception e) {
                            logger.error("fail to inject via method " + method.getName()
                                    + " of interface " + type.getName() + ": " + e.getMessage(), e);
                        }
                    }
                }
            }
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        }
        return instance;
    }
```

###### 2.3.1 获取目标方法

![image-20191225165329789](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225165329789.png)

12个方法 9个java原生的 3个我们自己的

###### 2.3.2 set注入判断&解析参数类型&获取属性名

![image-20191225165621491](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225165621491.png)

###### 2.3.3 依赖对象获取

![image-20191225170409058](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225170409058.png)

会跳到AdaptiveExtensionFactory这个方法里面去 这个类实现了ExtensionFactory接口 后面单独讲解下扩展点机制

## ExtensionLoader 扩展点加载机制

dubbo spi整个流程我觉得是在jdk spi基础上加了扩展点机制 接下来看下这个是怎么设计的

![image-20191225172219888](assets/dubbo%20SPI%E6%9C%BA%E5%88%B6&%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/image-20191225172219888.png)

```shell
com.alibaba.dubbo.common.extension
 |
 |--factory
 |     |--AdaptiveExtensionFactory   
 |     |--SpiExtensionFactory        
 |
 |--support
 |     |--ActivateComparator
 |
 |--Activate  #自动激活加载扩展的注解
 |--Adaptive  #自适应扩展点的注解
 |--ExtensionFactory  #扩展点对象生成工厂接口
 |--ExtensionLoader   #扩展点加载器，扩展点的查找，校验，加载等核心逻辑的实现类
 |--SPI   #扩展点注解
```

# 最后

## 画图工具

[draw.io](https://www.draw.io/)

## 项目链接

[示例代码git地址(后续文章用到的都会更新在这里)](https://github.com/ZhangDaFoYe/javaX-code)

## 参考

[dubbo源码导读spi机制](http://dubbo.apache.org/zh-cn/docs/source_code_guide/dubbo-spi.html)

