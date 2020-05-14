spring的IoC的流程可以分为`定位`、`加载`、`注册`三个步骤，在IoC容器中注册的是BeanDefinition，BeanDefinition就是解析出来的存储bean对象的。

     接着，进行DI操作，需要对BeanDefinition进行初始化，变成一个实例化的对象，最开始是调用的`DefaultListableBeanFactory`类的`getBean()`方法，此处不在详细描述，可以参考我另外一篇文章的Spring IoC的流程图，最终调用的是`AbstractAutowireCapableBeanFactory`类中的`doCreateBean()`方法里的`createBeanInstance()`方法，生成一个BeanWrapper对象，这个才是我们真正使用的实例化bean对象。

**spring IoC时序图：**
[https://blog.csdn.net/qq_33996921/article/details/106115472](https://blog.csdn.net/qq_33996921/article/details/106115472)

**spring DI时序图**
[https://blog.csdn.net/qq_33996921/article/details/106115589](https://blog.csdn.net/qq_33996921/article/details/106115589)


源码如下：

```java
//AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {	
    // Instantiate the bean.
		BeanWrapper instanceWrapper = null;
。。。。。。。。。省略。。。。。
if (instanceWrapper == null) {
  //1、初始化bean			
   instanceWrapper = createBeanInstance(beanName, mbd, args);
}
。。。。。。。。。。。。。。。。。
// Initialize the bean instance.
Object exposedObject = bean;
  try {
 	//2、DI依赖注入
  	populateBean(beanName, mbd, instanceWrapper);
 	//3、对exposedObject进行处理，生成代理类
 	exposedObject = initializeBean(beanName, exposedObject, mbd);
 }
}
```

     `doCreateBean()`这个方法其实包含三个非常重要的三个步骤：

> 1、createBeanInstance()，实例化bean,生成一个BeanWrapper对象；
>
> 2、populateBean(）:依赖注入;
>
> 3、initializeBean()：对exposedObject进行处理，生成代理类;

此处并没有直接将生成的`instanceWrapper`对象直接暴露出来，而是将`exposedObject`对象暴露出来，这个对外暴露的对象是经过代理生成后的对象，如果没有代理的对象，那其实跟`instanceWrapper`对象是同一个对象。

简单看一下`initializeBean()`这个方法：

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		......省略代码......
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

代理对象其实就是调用`applyBeanPostProcessorsAfterInitialization()`这个方法，最后调用`AbstractAutoProxyCreator`类`wrapIfNecessary()`方法产生，代码如下：

```java
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
	。。。。。。。。省略。。。。。。。。
		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
            //创建代理对象
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```

从上面可以看到代码中有创建代理对象的`createProxy()`方法了，具体的就不往下跟了。

    言归正传，我们来说一下我们今天的主题，为什么说Spring的AOP是在IoC之后，再DI之前呢？

通过上面的源码分析，

1）IoC阶段先创建了容器，并将`BeanDefinition`注册到容器中；

2）初始化Bean，生成了`BeanWrapper`，再对`BeanWrapper`进行依赖注入；

3）在DI的过程中，如果发现set的属性值是没有初始化`bean`的,就先对其进行初始化操作，然后继续循环调用`getBean()`方法，如果是此对象是代理对象就会将生成的`exposedObject`对象依赖注入进去。

三级缓存的对象如下：

```java
//DefaultSingletonBeanRegistry.java
	//一级缓存：
	/** 保存所有的singletonBean的实例 */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

	//二级缓存：
	/** 保存所有早期创建的Bean对象，这个Bean还没有完成依赖注入 */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

	//三级缓存：
	//存放需要解决循环依赖的bean信息（beanName,和一个回调工厂）
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```



此时如果发生循环依赖，会使用Spring的三级缓存解决循环依赖的问题。本文中不对循环依赖做过多详细描述，后续有时间我会专门再写一篇关于解决循环依赖的文章。

总结一下：

从整体思想上说，在A对象依赖注入（DI）时，需要注入对象B的话，如果容器中没有B的对象，需要再调用B的getBean，B的`populateBean()`，B的`initializeBean()`，通过`initializeBean()`生成了B的代理对象，然后将B的代理对象在注入到A中，这里也说明了在DI之前要先生成要注入属性的代理对象。这也就我们说的Aop要在DI之后，但如果是只针对自己的单个对象来说，还是拿A对象为例子，A在完成依赖注入后，再经过AOP生成自己的代理对象。