---
title: Spring源码解析之IoC
date: 2019-07-11 16:41:50
tags: ioc
categories: Java
top: True

---

IOC、AOP是Spring的两大核心，通过IOC(依赖注入)，Spring容器为我们管理程序中的java bean，省去了开发人员很多繁琐工作。

## BeanFactory和ApplicationContext

在Spring IOC容器设计中，分为了两个主要的容器系列，`BeanFactory`的简单容器系列以及`ApplicationContext`应用上下文的高级容器系列。这里`BeanFactory`是所有容器的基类，定义了`getBean()`等最基本的方法，抽象出容器的概念。
<!-- more -->
`BeanFactory`源码：

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String var1) throws BeansException;

    <T> T getBean(String var1, Class<T> var2) throws BeansException;

    Object getBean(String var1, Object... var2) throws BeansException;

    <T> T getBean(Class<T> var1) throws BeansException;

    <T> T getBean(Class<T> var1, Object... var2) throws BeansException;

    boolean containsBean(String var1);

    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, ResolvableType var2) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String var1, Class<?> var2) throws NoSuchBeanDefinitionException;

    Class<?> getType(String var1) throws NoSuchBeanDefinitionException;

    String[] getAliases(String var1);
}
```

## 容器关系分析

Spring中容器的继承关系非常复杂，每个容器的定义与分工非常明确，这里我们讲解几个重要的容器类

![](http://ww1.sinaimg.cn/large/e3ea9f2dgy1g4w0cfgx9aj21qw0ykaci.jpg)

### BeanFacotry的一些扩展接口

其实这可以从它的名字猜出来它的作用

- ListableBeanFactory：可列表的，意思是扩展了容器对bean批量的操作
- HierarchicalBeanFactory：层级的，定义了`getParentBeanFactory()`方法，引入了父子容器的概念

- ConfigurableBeanFactory：继承`HierarchicalBeanFactory`，官方解释是这个扩展接口只是为了允许框架-内部的即插即用，以及对bean工厂配置方法的特殊访问。意思就是主要方便框架内部使用的，稍作了解即可。

-  ConfigurableListableBeanFactory：继承上面三个直接子接口，主要也是为了内部框架的使用

### 重要的实现类

1. AbstractAutowireCapableBeanFactory：该抽象类实现了bean的创建以及属性注入的功能，以及`aware`接口、`InitializingBean`等Spring接口功能逻辑的实现
2. DefaultListableBeanFactory：扩展了上面`ConfigurableListableBeanFactory`和`AbstractAutowireCapableBeanFactory`同时提供了最简单的容器实现，从而它也成为了一个最基本的IOC容器。其他IOC容器，比如说`XmlBeanFacotry`都是在这个简单容器的基础上进行相应的扩展。

以上的容器继承关系的分析有助于我们下面分析容器的启动流程，磨刀不误砍柴工。

## IOC容器的初始化

正常来说，一个IOC容器的完全启动分两步，容器的初始化以及Bean的初始化。但并不是绝对，可以通过设置bean的lazy-init属性来控制bean的初始化时间，使得bean在容器初始化过程中就进行初始化。这里我们先忽略lazy-init，首先来看IOC容器的初始化过程。

IOC容器的初始化通过调用`refresh()`，这是IOC容器的核心方法，通过`refresh()`触发了一系列容器的初始化操作。这里面其实主要干了三件事情：bean资源的定位、beanDefinition的解析和注册。

>BeanDefinition是Spring中对bean的一种抽象，其实就是bean在IOC容器中的表现形式。是Spring中定义的一种核心数据结构，依赖注入等功能都是围绕对BeanDefnition的处理来完成的。

### BeanDefinition资源定位

也就是Resource的定位，它由ResourceLoader通过统一的Resource接口来完成。Spring中已经为我们封装了很多resource，比如ClassPathResource。这里我们不做详解。

```java
ClassPathResource resource = new ClassPathResource("beans.xml");
```

创建一个类路径下定位xml文件的resource，Spring就会去类路径下寻找beanDefinition定义的信息，注意这里的resource并不能给`DefaultListableBeanFactory`直接使用，而是通过`BeanDefinitionReader`来完成读取。

接着我们以一个常用的实现`FileSystemXmlApplicationContext`来分析Resource的定位过程。
```java
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
        throws BeansException {
        super(parent);
        setConfigLocations(configLocations);
        if (refresh) {
            refresh();
        }
    }

    @Override
    protected Resource getResourceByPath(String path) {
        if (path != null && path.startsWith("/")) {
            path = path.substring(1);
        }
        return new FileSystemResource(path);
    }
```

`FileSystemXmlApplicationContext`内部的代码很简单，通过构造方法传入的locations设置资源路径，然后调用refresh()方法，启动容器初始化过程。`getResourceByPath`是一个模板方法，与各个类如何加载xml文件的bean有关，由ResourceLoader在getResource时调用。

`refresh()`方法的实现在`AbstractApplicationContext`中

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

我们跳到`refresh()`方法中，这里我们先看第二步`obtainFreshBeanFactory()`方法，这里是创建容器对象的过程。再跳到`obtainFreshBeanFactory()`中，具体的过程在`refreshBeanFactory()`中。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		refreshBeanFactory();
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}


```

`refreshBeanFactory()`是一个抽象方法，我们看`AbstractRefreshableApplicationContext`中的实现

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



