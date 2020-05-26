Spring+MyBatis源码解析之SqlSessionTemplate

# 1、产生的背景

 	使用`MyBatis`就要保证`SqlSession`实例的线程安全，就必须要为每一次的请求单独创建一个`SqlSession`。但如果每一次请求都要用`openSession()`自己去创建，就比较麻烦了。

​	所以在`spring`中，我们使用的`mybatis-spring`包中提供了一个线程安全的`SqlSession`的包装类`sqlSessionTemplate`，用它来替代`SqlSession`.因为他是线程安全的，所以可以在所有的Service层来共享一个实例（默认为单例）。

​	这个跟` Spring `封装其他的组件是一样的，比如` JdbcTemplate，
RedisTemplate` 等等，也是 `Spring` 跟` MyBatis` 整合的最关键的一个类。



# 2、创建SqlSessionTemplate

## 2.1 生成代理对象

​	`SqlSessionTemplate` 里面有` DefaultSqlSession` 的所有的方法：select One()、select List()、insert()、update()、delete()，不过它都是通过一个代理对象实现的。这个代理对象在构造方法里面通过一个代理类创建：

```java
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType, PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;
      //创建一个代理类，被代理的对象实现 SqlSession接口，代理类为SqlSessionInterceptor
    this.sqlSessionProxy = (SqlSession) newProxyInstance(SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class }, new SqlSessionInterceptor());
  }
```

所有的方法都会先走到内部代理类 `SqlSessionInterceptor` 的 `invoke()`方法：

```java
 private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        。。。
        return result;
      } catch (Throwable t) {
          。。。。。。。。。。。。。。。。
    }
  }
```

## 2.2、获取SqlSession

​		首先会使用工厂类、执行器类型、异常解析器创建一个 `sqlSession`，然后再调用
`sqlSession` 的实现类，实际上就是在这里调用了 `DefaultSqlSession` 的方法。此处调用的`getSqlSession()`方法用来获取一个`SqlSession`对象。

```java
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    //获取SqlSessionHolder
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    LOGGER.debug(() -> "Creating a new SqlSession");
    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }

```

​	`TransactionSynchronizationManager`获取当前线程`threadLocal`是否有`SqlSessionHolder`，如果有就从`SqlSessionHolder`取出当前`SqlSession`，如果当前线程`threadLocal`没有`SqlSessionHolder`，就从`sessionFactory`中创建一个`SqlSession`，最后注册会话到当前线程`threadLocal`中。

## 2.3 事务管理器



```java
public abstract class TransactionSynchronizationManager {

	private static final Log logger = LogFactory.getLog(TransactionSynchronizationManager.class);
 	 // 存储当前线程事务资源，比如Connection、session等
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<>("Transactional resources");
	// 存储当前线程事务同步回调器
   // 当有事务，该字段会被初始化，即激活当前线程事务管理器
	private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
			new NamedThreadLocal<>("Transaction synchronizations");
    
    /**
    *从当前线程的threadLocal中获取SqlSessionHolder
    */
    @Nullable
	public static Object getResource(Object key) {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
        //获取
		Object value = doGetResource(actualKey);
		if (value != null && logger.isTraceEnabled()) {
			logger.trace("Retrieved value [" + value + "] for key [" + actualKey + "] bound to thread [" +
					Thread.currentThread().getName() + "]");
		}
		return value;
	}
    
    @Nullable
	private static Object doGetResource(Object actualKey) {
        //private static final ThreadLocal<Map<Object, Object>> resources =
		//	new NamedThreadLocal<>("Transactional resources");
		Map<Object, Object> map = resources.get();
		if (map == null) {
			return null;
		}
		Object value = map.get(actualKey);
		// Transparently remove ResourceHolder that was marked as void...
		if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
			map.remove(actualKey);
			// Remove entire ThreadLocal if empty...
			if (map.isEmpty()) {
				resources.remove();
			}
			value = null;
		}
		return value;
	}
    
}
```

这是`spring`的一个当前线程事务管理器，它允许将当前资源存储到当前线程`ThreadLocal`中，`SqlSessionHolder`是保存在`resources`中。

## 2.4 注册SessionHolder

```java
private static void registerSessionHolder(SqlSessionFactory sessionFactory, ExecutorType executorType,PersistenceExceptionTranslator exceptionTranslator, SqlSession session) {
  SqlSessionHolder holder;
  // 判断当前是否有事务
  if (TransactionSynchronizationManager.isSynchronizationActive()) {
    Environment environment = sessionFactory.getConfiguration().getEnvironment();
    // 判断当前环境配置的事务管理工厂是否是SpringManagedTransactionFactory（默认）
    if (environment.getTransactionFactory() instanceof SpringManagedTransactionFactory) {
      if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("Registering transaction synchronization for SqlSession [" + session + "]");
      }

      holder = new SqlSessionHolder(session, executorType, exceptionTranslator);
      // 绑定当前SqlSessionHolder到线程ThreadLocal中
      TransactionSynchronizationManager.bindResource(sessionFactory, holder);
      // 注册SqlSession同步回调器
      TransactionSynchronizationManager.registerSynchronization(new SqlSessionSynchronization(holder, sessionFactory));
      holder.setSynchronizedWithTransaction(true);
      // 会话使用次数+1
      holder.requested();
    } else {
      if (TransactionSynchronizationManager.getResource(environment.getDataSource()) == null) {
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because DataSource is not transactional");
        }
      } else {
        throw new TransientDataAccessResourceException(
          "SqlSessionFactory must be using a SpringManagedTransactionFactory in order to use Spring transaction synchronization");
      }
    }
  } else {
    if (LOGGER.isDebugEnabled()) {
      LOGGER.debug("SqlSession [" + session + "] was not registered for synchronization because synchronization is not active");
    }
  }
}
```

注册`SqlSession`到当前线程事务管理器的条件首先是当前环境中有事务，否则不注册，判断是否有事务的条件是`synchronizations`的`ThreadLocal`是否为空：

```java
public static boolean isSynchronizationActive() {
  return (synchronizations.get() != null);
}
```

每当我们开启一个事务，会调用`initSynchronization()`方法进行初始化`synchronizations`，以激活当前线程事务管理器。

```java
public static void initSynchronization() throws IllegalStateException {
  if (isSynchronizationActive()) {
    throw new IllegalStateException("Cannot activate transaction synchronization - already active");
  }
  logger.trace("Initializing transaction synchronization");
  synchronizations.set(new LinkedHashSet<TransactionSynchronization>());
}
```

所以当前有事务时，会注册`SqlSession`到当前线程`ThreadLocal`中。

## 2.5 小结

在`mybatis`中`SqlSession`是非线程安全的，如下：

```java
/**
 * The default implementation for {@link SqlSession}.
 * Note that this class is not Thread-Safe.
 *
 * @author Clinton Begin
 */
public class DefaultSqlSession implements SqlSession {
    
}
```



在spring中，通过`ThreadLocal`为每一个请求创建一个SqlSession，这样保证了线程的安全，从前面的背景中了解到在spring中是使用了`SqlSession`的包装类`SqlSessionTemplate`，用它来替代`SqlSession`。接下里就重点介绍`SqlSessionTemplate`。

# 3、获取SqlSessionTemplate

​		接着我们来讲解怎么获取这个`SqlSessionTemplate`,在 `Service` 层可以使用`@Autowired `自动注入的 Mapper 接口，需要保存在
`BeanFactory`（比如 XmlWebApplicationContext）中。也就是说接口肯定是在 Spring
启动的时候被扫描了，注册过的。我们先来看一下spring的配置文件`applicationContext.xml`

```xml
<bean id="mapperScanner" class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.org.nci.henry.dao"/>
</bean>
```

## 3.1 MapperScannerConfigurer 

​	`MapperScannerConfigurer` 实现了 `BeanDefinitionRegistryPostProcessor` 接口，
`BeanDefinitionRegistryPostProcessor `是`BeanFactoryPostProcessor` 的子类，可以通过编码的方式修改、新增或者删除某些 Bean 的定义。类图如下：

![image-20200523123506426](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200523123506426.png)

spring初始化ioc容器的时候，会调用`postProcessBeanDefinitionRegistry()`方法；

```java
//MapperScannerConfigurer.java
    MapperScannerConfigurer
    implements BeanDefinitionRegistryPostProcessor, InitializingBean, ApplicationContextAware, BeanNameAware
    
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
    
    scanner.scan(
        StringUtils.tokenizeToStringArray(this.basePackage, ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS));
}    
    

```

`scanner.scan()` 方 法 是 `ClassPathBeanDefinitionScanner` 中 的 ， 而 它 的 子 类
`ClassPathMapperScanner` 覆 盖 了 `doScan() `方 法 ， 在 `doScan() `中 调 用 了
`processBeanDefinitions()`。

```java
public int scan(String... basePackages) {
		int beanCountAtScanStart = this.registry.getBeanDefinitionCount();

		doScan(basePackages);

		// Register annotation config processors, if necessary.
		if (this.includeAnnotationConfig) {
			AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
		}

		return (this.registry.getBeanDefinitionCount() - beanCountAtScanStart);
	}
```





调用子类`doScan()`方法，把接口全部注添加到`beanDefinitions`中。

```java
   public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner { 
.............
public Set<BeanDefinitionHolder> doScan(String... basePackages) {
    Set<BeanDefinitionHolder> beanDefinitions = super.doScan(basePackages);

    if (beanDefinitions.isEmpty()) {
      LOGGER.warn(() -> "No MyBatis mapper was found in '" + Arrays.toString(basePackages)
          + "' package. Please check your configuration.");
    } else {
     //对生成的beanDefinition做处理
      processBeanDefinitions(beanDefinitions);
    }

    return beanDefinitions;
  }
   }
```

## 3.2 MapperFactoryBean

调用`processBeanDefinitions()`，在注册前将`beanClass`设置为`MapperFactoryBean`,也就是说所有的Mapper接口在容器里都被注册成了一个`MapperFactoryBean`.

```java
 public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner { 
     
    private Class<? extends MapperFactoryBean> mapperFactoryBeanClass = MapperFactoryBean.class;

    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
        GenericBeanDefinition definition;
        for (BeanDefinitionHolder holder : beanDefinitions) {
          definition = (GenericBeanDefinition) holder.getBeanDefinition();
          String beanClassName = definition.getBeanClassName();
    ................
            /**核心在此处，为definition设置beanClass,也就是对bd做后置处理，改变bd  		  *的属性,设置成mapperFactoryBeanClass，此为MapperFactoryBean对象
            */
          definition.setBeanClass(this.mapperFactoryBeanClass);

          definition.getPropertyValues().add("addToConfig", this.addToConfig);

  }
}
```

来看下`MapperFactoryBean`这个类

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    
}
```

这个类继承了`SqlSessionDaoSupport`,我们知道这个类中包含有spring操作MyBatis的`sqlSessionTemplate`的对象。

```java
public abstract class SqlSessionDaoSupport extends DaoSupport {

  private SqlSessionTemplate sqlSessionTemplate;
}
```

​		因为`MapperFactoryBean`实现了`FactoryBean`接口，所以在实例化的时候会调用`getObject()`方法，了解过MyBatis源码的同学应该就能知道，此处调用`getMapper()`是为了生成一个代理对象。这个我会在后续的文章中讲解MyBatis源码的时候将调用`getMapper()`方法生成代理对象的流程做详细介绍，此处不再赘述。

```java
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }

```

## 3.3 SqlSessionTemplate

先通过调用父类`SqlSessionDaoSupport`的`getSqlSession()`拿到`sqlSessionTemplate`对象，

```java
public SqlSession getSqlSession() {
    return this.sqlSessionTemplate;
  }
```

继续调用`SqlSessionTemplate`的`getMapper()`

```java
 public class SqlSessionTemplate implements SqlSession, DisposableBean {

    public <T> T getMapper(Class<T> type) {
        return getConfiguration().getMapper(type, this);
      }
     
      @Override
      public Configuration getConfiguration() {
          //获取Configuration对象
        return this.sqlSessionFactory.getConfiguration();
      } 
          
 }
```



## 3.4 Configuration

调用`Configuration`类中`getMapper`

```java
public class Configuration {
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
}
```

```java
public class MapperRegistry {
    public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
      final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
      if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
      }
      try {
          //生成动态代理
        return mapperProxyFactory.newInstance(sqlSession);
      } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
      }
    }
}
```

​		后面就是`MyBatis`中生成mapper代理对象的流程了，后续会在MyBatis源码篇中详细讲解，此处不再详细赘述。

通 过 工 厂 类`MapperProxyFactory `获得一个` MapperProxy` 代理对象。也就是说，我们注入到` Service `层的接口，实际上还是一个 `MapperProxy `代理对象。所以最后调用 `Mapper` 接口的方法，也是执行 `MapperProxy` 的 `invoke()`方法.

## 3.5 小结

简单总结一下上面的流程：

1、`MapperScannerConfigurer`实现了`BeanDefinitionRegistryPostProcessor`类，spring初始化IoC容器bean名字为`mapperScannerConfigurer`的时候，会调用`postProcessBeanDefinitionRegistry()`方法；

2、通过`MapperScannerConfigurer`调用`postProcessBeanDefinitionRegistry`,来进行指定mapper包的扫描，从而获取到`beanDefinition`;

3、对`beanDefinition`进行处理，通过`ClassPathMapperScanner`类中的`processBeanDefinitions()`的对bean进行重新处理：

>```java
>//这个definition就变成了mapperFactoryBeanClass对象（MapperFactoryBean）
>definition.setBeanClass(this.mapperFactoryBeanClass);
>```

​	`MapperFactoryBean`是一个继承了`SqlSessionDaoSupport`的类，它可以对`sqlSessionTemplate`进行一系列的操作，`SqlSessionTemplate`实现了`SqlSession`的接口，为了方便理解，我们可以简单的把`SqlSessionTemplate`当成是`MyBatis`中的`sqlSession`来使用，而spring为了更加便捷的使用MyBatis，所以将sqlSession，封装成了`SqlSessionTemplate`来使用。

4、在初始化扫描到的mapper对应的bean时，获取到的是`MapperFactoryBean`的bean，此时需要调用该类的`getObject()`方法，才能获取真正的bean.

5、先调用`getSqlSession()`方法获取到`sqlSessionTemplate`,调用`getMapper()`方法生成mapper相关的代理对象，这样我们就可以直接在业务代码中加入`@Autowired`进行依赖注入，然后轻松使用就可以了。

```java
    @Autowired
    private UserMapper userMapper;
```

# 4、应用：

经过上面的分析我们基本了解获取`sqlSessionTemplate`的基本原理，所以我们使用 `Mapper` 的时候，只需要在加了 `Service` 注解的类里面使用`@Autowired`注入 `Mapper` 接口就好了。

```java
@Service
public class User UserService {
	@Autowired
    private UserMapper userMapper;
    
    public List<User> getAll() {
        return userMapper.selectByAll();
	}
}
```

通过每一个注入`Mapper`的地方，都可以拿到`sqlSessionTemplate`。

