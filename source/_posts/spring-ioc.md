---
title: Spring源码解析之IoC(一)
date: 2019-07-11 16:41:50
tags:
 - ioc
 - spring	
categories: Java
top: True

---

IOC、AOP是Spring的两大核心，通过IOC(依赖注入)，Spring容器为我们管理程序中的java bean，省去了开发人员很多繁琐工作。

## BeanFactory和ApplicationContext
在Spring IOC容器设计中，分为了两个主要的容器系列，`BeanFactory`的简单容器系列以及`ApplicationContext`应用上下文的高级容器系列。而`BeanFactory`作为所有容器的基类，定义了`getBean()`等最基本的方法，抽象出容器的概念。
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

Spring中容器的继承关系非常复杂，每个容器的定义与分工非常明确，这里我们先认识一下几个重要的容器

![](http://ww1.sinaimg.cn/large/e3ea9f2dgy1g4w0cfgx9aj21qw0ykaci.jpg)

### BeanFacotry的一些扩展接口
其实这些容器一般可以从它们的名字就能猜出来各自的作用

- ListableBeanFactory：可列表的，意思是扩展了容器对bean批量的操作
- HierarchicalBeanFactory：层级的，定义了`getParentBeanFactory()`方法，引入了父子容器的概念
- ConfigurableBeanFactory：继承`HierarchicalBeanFactory`，官方解释是这个扩展接口只是为了允许框架-内部的即插即用，以及对bean工厂配置方法的特殊访问。意思就是主要方便框架内部使用的，稍作了解即可。
-  ConfigurableListableBeanFactory：继承上面三个直接子接口，主要也是为了内部框架的使用

### 重要的实现类
1. AbstractAutowireCapableBeanFactory：该抽象类实现了bean的创建以及属性注入的功能，以及`aware`接口、`InitializingBean`等Spring接口功能逻辑的实现
2. DefaultListableBeanFactory：扩展了上面`ConfigurableListableBeanFactory`和`AbstractAutowireCapableBeanFactory`同时提供了最简单的容器实现，从而它也成为了一个最基本的IOC容器。其他IOC容器，比如说`XmlBeanFacotry`都是在这个简单容器的基础上进行相应的扩展。

以上的容器继承关系的分析有助于我们下面分析容器的启动流程，磨刀不误砍柴工。

## IOC容器的初始化
正常来说，一个IOC容器的完全启动分两步，容器的初始化以及Bean的初始化。但这并不是绝对，我们可以通过设置bean的lazy-init属性来控制bean的初始化时间，使得bean在容器初始化过程中就进行初始化。这里我们先忽略lazy-init属性，假设两个的初始化是分开的。首先来看IOC容器的初始化过程。

IOC容器的初始化通过调用`refresh()`，这是IOC容器的核心方法，通过`refresh()`触发了一系列容器的初始化操作。这里面其实主要干了三件事情：bean资源的定位、beanDefinition的解析和注册。

>BeanDefinition是Spring中对bean的一种抽象，其实就是bean在IOC容器中的表现形式。是Spring中定义的一种核心数据结构，依赖注入等功能都是围绕对BeanDefnition的处理来完成的。

### BeanDefinition资源定位
也就是Resource的定位，它由ResourceLoader通过统一的Resource接口来完成。Spring中已经为我们封装了很多resource，比如ClassPathResource。这里我们不做详解。
```java
ClassPathResource resource = new ClassPathResource("beans.xml");
```
创建一个类路径下定位xml文件的resource，Spring就会去类路径下寻找BeanDefinition定义的信息，注意这里的resource并不能给`DefaultListableBeanFactory`直接使用，而是通过`BeanDefinitionReader`来完成读取。

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

`FileSystemXmlApplicationContext`内部的代码很简单，通过构造方法传入的locations设置资源路径，然后调用refresh()方法，启动容器初始化过程。`getResourceByPath`是一个模板方法，与各个类如何加载xml文件的bean有关，每个类定义自己的资源获取方式，由ResourceLoader在getResource时调用。

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

我们来看一下`refreshBeanFactory()`这个方法，主要做了两件事：创建了一个`DefaultListableBeanFactory`类型的容器，然后以该容器对象为入参调用`loadBeanDefinitions(beanFactory)`。这个方法也是一个模板方法，由具体的子类实现。因为上面我们是以xml的资源为例，所以我们直接看`AbstractXmlApplicationContext`中的实现

```java
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

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
}
```

方法很简单，创建一个`XmlBeanDefinitionReader`的对象，初始化相应参数，然后具体的资源获取就委托给该reader对象。

我们进入`XmlBeanDefinitionReader`的源码中，根据调用链一步步找到我们具体的加载方法，这里是`doLoadBeanDefinitions()`

```java
Document doc = doLoadDocument(inputSource, resource);
return registerBeanDefinitions(doc, resource);
```

这里也是两步，第一步：获取Document的Dom结构对象；第二步，解析Dom对象并注册BeanDefinition。第一步相对比较简单，我们具体来看第二步。

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

下面其实就是我们的解析步骤了

### BeanDefinition的解析

首先会创建一个DocumentReader对象，然后具体的register逻辑就委托给该对象。`BeanDefinitionDocumentReader`是一个接口，这里使用的是Spring的默认实现类`DefaultBeanDefinitionDocumentReader`

```java
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    logger.debug("Loading bean definitions");
    Element root = doc.getDocumentElement();
    doRegisterBeanDefinitions(root);
}

protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);

		this.delegate = parent;
}

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
```

具体的逻辑都在`DefaultBeanDefinitionDocumentReader`对象中。这里讲解一下，主要是递归遍历节点，对不同的标签进行相应的解析，最后是解析`<Bean>`标签，每个`Bean`标签对应一个BeanDefinition对象。这里的标签解析同样会委托给一个delegate对象：

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
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

调用`parseBeanDefinitionElement`进行解析，得到一个BeanDefinition的包装类对象`BeanDefinitionHolder`。解析完成后就是保存我们的beanDefintion对象，也就是注册。

### BeanDefinition的注册

注册是`BeanDefinitionReaderUtils.registerBeanDefinition`这个调用过程，这里的Registry对象其实就是我们的容器对象，该方法的内部其实还是通过调用Registry的`registerBeanDefinition`的方法来进行注册的。具体的实现在`DefaultListableBeanFacotry`中，逻辑很简单，就是讲BeanDefinition保存到Map中，这里就不做详述。

### 容器初始化成功

到这里容器大致就算初始化完成了，严格来说`refresh()`方法中还进行了很多参数及内部bean的初始化，比如'BeanFactoryPostProcessors'注册及调用, `BeanPostProcessors`注册等，这里我们先不考虑，只以最简单的容器的bean管理功能来看，这里容器算是初始化完成了。

## Bean的初始化

不考虑lazy-init=false的情况，bean的初始化是在容器首次调用getBean触发的

### getBean()

下面我们从DefaultListableBeanFactory的基类AbstractBeanFactory着手去看getBean的实现

```java
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						getBean(dep);
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
						@Override
						public Object getObject() throws BeansException {
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
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

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

				else {
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
							@Override
							public Object getObject() throws BeansException {
								beforePrototypeCreation(beanName);
								try {
									return createBean(beanName, mbd, args);
								}
								finally {
									afterPrototypeCreation(beanName);
								}
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

		// Check if required type matches the type of the actual bean instance.
		if (requiredType != null && bean != null && !requiredType.isInstance(bean)) {
			try {
				return getTypeConverter().convertIfNecessary(bean, requiredType);
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
}
```

我们看到主要逻辑在doGetBean方法中。可以看到该方法比较复杂，这里会进行单例、原型不同类型bean的处理，循环依赖的解决，依赖注入等。因为篇幅问题，这里我们简单看一下流程，而这些具体的过程我会在下一篇文章继续分析。

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
		Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);
		mbd.resolvedTargetType = beanType;

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

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			addSingletonFactory(beanName, new ObjectFactory<Object>() {
				@Override
				public Object getObject() throws BeansException {
					return getEarlyBeanReference(beanName, mbd, bean);
				}
			});
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			populateBean(beanName, mbd, instanceWrapper);
			if (exposedObject != null) {
				exposedObject = initializeBean(beanName, exposedObject, mbd);
			}
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
}
```

首先如果是单例的话，会先从缓存中获取，如果没有的话，最后会调用createBean创建，

createBean再调用*doCreateBean*，这里要注意，该方法是重点，包含了核心逻辑，下一篇我们会具体分析该方法。

创建完成后，首先会检查需不需要注册到容器中，同时return给调用端。Bean的初始化也就完成了。

未完待续…...
