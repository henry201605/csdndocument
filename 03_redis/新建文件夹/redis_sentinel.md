

Redis之Sentinel应用及源码分析



Jedis 有4 种工作模式：单节点、分片（Sharded）、哨兵(Sentinel)、集群(Cluster)。上一篇我们通过实际应用的例子进行了源码剖析，请点击https://blog.csdn.net/qq_33996921/article/details/105602288，本文将通过实际的例子对Redis的Sentinel进行源码剖析。

## 1、环境：

对于Redis Sentinel的搭建请参考https://blog.csdn.net/qq_33996921/article/details/105440113

1、操作系统：Centos7.7

2、服务器配置如下：（本文采用了在单台机子上部署多个实例来模拟多机部署）

| 主机     |       ip        | 角色和端口 |
| -------- | :-------------: | :--------- |
| master   | 132.232.125.196 | 6380       |
| slave1   | 132.232.125.196 | 6381       |
| slave2   | 132.232.125.196 | 6382       |
| Sentinel | 132.232.125.196 | 16380      |
| Sentinel | 132.232.125.196 | 16381      |
| Sentinel | 132.232.125.196 | 16382      |

## 2、启动redis和Sentinel服务

```shell
[root@m logs]# ps -ef|grep redis
root     13037     1  0 18:07 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6380
root     13042     1  0 18:07 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6381
root     13049     1  0 18:07 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6382
root     13942     1  0 18:07 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16380 [`sentinel`]
root     13947     1  0 18:07 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16381 [`sentinel`]
root     13952     1  0 18:07 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-sentinel *:16382 [`sentinel`]
root     14019   937  0 18:07 ?        00:00:00 [redis-server] <defunct>
root     14021 21486  0 18:07 pts/1    00:00:00 grep --color=auto redis

```

## 3、测试代码

```shell
package nci.henry;

import org.junit.Before;
import org.junit.Test;
import redis.clients.jedis.JedisSentinelPool;

import java.util.HashSet;
import java.util.Set;

/**
 * @Author: henry
 * @Date: 2020/4/18 15:43
 * @Description: 测试Sentinel
 */
public class JedisSentinelTest {


    private JedisSentinelPool pool;

    @Before
    public void initJedis(){
        // master的名字是sentinel.conf配置文件里面的名称
        String masterName = "mymaster";
        Set<String> sentinels = new HashSet<String>();
        sentinels.add("132.232.115.96:16380");
        sentinels.add("132.232.115.96:16381");
        sentinels.add("132.232.115.96:16382");
        pool = new JedisSentinelPool(masterName, sentinels);
    }

    @Test
    public void testGet(){
        try {
            pool.getResource().set("henry", "time:" + new Date());
            System.out.println(pool.getResource().get("henry"));
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



执行 testGet()方法，结果如下：

```shell
time:Sat Apr 18 18:32:27 CST 2020
```

`疑问`：

Jedis 连接Sentinel 的时候，我们配置的是全部哨兵的地址。Sentinel 是如何返回可用的master 地址的呢？下面我们通过分析源码，看一下Sentinel 具体实现方式。

## 4、源码分析

### 4.1 原理

1)、客户端连接到哨兵集群后，通过发送`Protocol.SENTINEL_GET_MASTER_ADDR_BY_NAME`命令;

2)、从哨兵机器中询问`master`节点的信息，拿到`master`节点的`ip`和`端口号`以后，再到客户端发起连接。

3)、建立连接以后，需要在客户端建立监听机制，当`master`重新选举之后，客户端需要重新连接到新的`master`节点。

### 4.2 JedisSentinelPool构造方法

```java
pool = new JedisSentinelPool(masterName, sentinels);
```

先来看下`JedisSentinelPool`的构造方法:

```java
public JedisSentinelPool(String masterName, Set<String> sentinels,
      final GenericObjectPoolConfig poolConfig, final int connectionTimeout, final int soTimeout,
      final String password, final int database, final String clientName,
      final int sentinelConnectionTimeout, final int sentinelSoTimeout, final String sentinelPassword,
      final String sentinelClientName) {

    this.poolConfig = poolConfig;
    this.connectionTimeout = connectionTimeout;
    this.soTimeout = soTimeout;
    this.password = password;
    this.database = database;
    this.clientName = clientName;
    this.sentinelConnectionTimeout = sentinelConnectionTimeout;
    this.sentinelSoTimeout = sentinelSoTimeout;
    this.sentinelPassword = sentinelPassword;
    this.sentinelClientName = sentinelClientName;

    HostAndPort master = initSentinels(sentinels, masterName);
    //首次调用initPool()方法
    initPool(master);
  }
```

在构造方法中调用了`initSentinels（）`方法，此方法是用来初始化Sentinel集群；



### 4.3 Sentinel集群初始化

```java
 private HostAndPort initSentinels(Set<String> sentinels, final String masterName) {

    HostAndPort master = null;
    boolean sentinelAvailable = false;

    log.info("Trying to find master from available Sentinels...");
    // 有多个sentinels,遍历这些个sentinels
    for (String sentinel : sentinels) {

      // host:port 表示的sentinel 地址转化为一个HostAndPort 对象。
      final HostAndPort hap = HostAndPort.parseString(sentinel);

      log.debug("Connecting to Sentinel {}", hap);

      Jedis jedis = null;
      try {
        // 连接到sentinel
        jedis = new Jedis(hap.getHost(), hap.getPort(), sentinelConnectionTimeout, sentinelSoTimeout);
        if (sentinelPassword != null) {
          jedis.auth(sentinelPassword);
        }
        if (sentinelClientName != null) {
          jedis.clientSetname(sentinelClientName);
        }
        // 根据masterName 得到master 的地址，返回一个list，host= list[0], port =// list[1]
        List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);

        // connected to sentinel...
        sentinelAvailable = true;

        if (masterAddr == null || masterAddr.size() != 2) {
          log.warn("Can not get master addr, master name: {}. Sentinel: {}", masterName, hap);
          continue;
        }
        // 如果在任何一个sentinel 中找到了master，不再遍历sentinels
        master = toHostAndPort(masterAddr);
        log.debug("Found Redis master at {}", master);
        break;
      } catch (JedisException e) {
        // resolves #1036, it should handle JedisException there's another chance
        // of raising JedisDataException
        log.warn(
          "Cannot get master address from sentinel running @ {}. Reason: {}. Trying next one.", hap,
          e.toString());
      } finally {
        if (jedis != null) {
          jedis.close();
        }
      }
    }
    // 到这里，如果master 为null，则说明有两种情况，一种是所有的sentinels节点都down掉了，一种是master
    //节点没有被存活的sentinels 监控到
    if (master == null) {
      if (sentinelAvailable) {
        // can connect to sentinel, but master name seems to not
        // monitored
        throw new JedisException("Can connect to sentinel, but " + masterName
            + " seems to be not monitored...");
      } else {
        throw new JedisConnectionException("All sentinels down, cannot determine where is "
            + masterName + " master is running...");
      }
    }
    // 如果走到这里，说明找到了master 的地址
    log.info("Redis master running at " + master + ", starting Sentinel listeners...");

    // 启动对每个sentinels 的监听为每个sentinel 都启动了一个监听者MasterListener。MasterListener 本身是一个线
//    程，它会去订阅sentinel 上关于master 节点地址改变的消息。
    for (String sentinel : sentinels) {
      final HostAndPort hap = HostAndPort.parseString(sentinel);
      MasterListener masterListener = new MasterListener(masterName, hap.getHost(), hap.getPort());
      // whether MasterListener threads are alive or not, process can be stopped
      masterListener.setDaemon(true);
      masterListeners.add(masterListener);
      masterListener.start();
    }

    return master;
  }
```

根据masterName 得到master 的地址：

```dart
 public List<String> sentinelGetMasterAddrByName(String masterName) {
    client.sentinel(Protocol.SENTINEL_GET_MASTER_ADDR_BY_NAME, masterName);
    final List<Object> reply = client.getObjectMultiBulkReply();
    return BuilderFactory.STRING_LIST.build(reply);
  }
```

通过以上源码可以得出如下初始化流程：

1) 遍历Sentinel节点集合，找到一个可用的Sentinel节点，如果找不到就从Sentinel节点集合中去找下一个；如果都找不到直接抛出异常给客户端：
2）找到一个可用的Sentinel节点， 执行sentinelGetMasterAddrByName（ masterName），通过主机名称找到对应主节点信息：

```java
List<String> masterAddr = jedis.sentinelGetMasterAddrByName(masterName);
```

3）JedisSentinelPool中没有发现对主节点角色验证的代码，这是因为get-master-addr-by-name master-name这个API本身就会自动获取真正的主节点（例如故障转移期间）。

```java
 client.sentinel(Protocol.SENTINEL_GET_MASTER_ADDR_BY_NAME, masterName);

  public static final String SENTINEL_GET_MASTER_ADDR_BY_NAME = "get-master-addr-by-name";
```



4）得到 master 信息后，再次遍历哨兵集合，为每一个Sentinel节点单独启动一个线程，也可以称之为监听者MasterListener，监听哨兵的发布订阅消息，消息主题是 `+switch-master`. 当主节点发生变化时，将通过 pub/sub 通知该线程，该线程将更新 Redis 连接池。

### 4.4 MasterListener

来看一下MasterListener线程中的run方法：

```java
@Override
    public void run() {

      running.set(true);

      // 死循环
      while (running.get()) {
        //创建一个 Jedis对象
        j = new Jedis(host, port);

        try {
          // 继续检查
          // double check that it is not being shutdown
          if (!running.get()) {
            break;
          }
          // code for active refresh
          List<String> masterAddr = j.sentinelGetMasterAddrByName(masterName);
          if (masterAddr == null || masterAddr.size() != 2) {
            log.warn("Can not get master addr, master name: {}. Sentinel: {}:{}.", masterName, host, port);
          } else {
            initPool(toHostAndPort(masterAddr));
          }

          // jedis 对象，通过 Redis pub/sub 订阅 switch-master 主题
          //订阅sentinel上关于master地址改变的消息
          j.subscribe(new JedisPubSub() {
            @Override
            public void onMessage(String channel, String message) {
              log.debug("Sentinel {}:{} published: {}.", host, port, message);
              // 分割字符串
              String[] switchMasterMsg = message.split(" ");
              // 如果长度大于3
              if (switchMasterMsg.length > 3) {

                // 且第一个字符串的名称和当前 masterName 发生了 switch
                if (masterName.equals(switchMasterMsg[0])) {
                  // 重新初始化连接池（第 4 个和 第 5 个）
                  initPool(toHostAndPort(Arrays.asList(switchMasterMsg[3], switchMasterMsg[4])));
                } else {
                  log.debug(
                    "Ignoring message on +switch-master for master name {}, our master name is {}",
                    switchMasterMsg[0], masterName);
                }

              } else {
                log.error(
                  "Invalid message received on Sentinel {}:{} on channel +switch-master: {}", host,
                  port, message);
              }
            }
          }, "+switch-master");

        } catch (JedisException e) {
          // 如果连接异常
          if (running.get()) {
            log.error("Lost connection to Sentinel at {}:{}. Sleeping 5000ms and retrying.", host,
              port, e);
            try {
              // 默认休息 5 秒
              Thread.sleep(subscribeRetryWaitTimeMillis);
            } catch (InterruptedException e1) {
              log.error("Sleep interrupted: ", e1);
            }
          } else {
            log.debug("Unsubscribing from Sentinel at {}:{}", host, port);
          }
        } finally {
          j.close();
        }
      }
    }
```

1、对每一个哨兵节点通过一个 MasterListener 进行监听（Redis的发布订阅功能），订阅哨兵节点`+switch-master`频道；

2、当发生故障转移时，即master地址变换时，就会再调用一次initPool()方法客户端能收到哨兵的通知，通过重新初始化连接池，完成主节点的切换。

### 4.4 initPool()方法

```java
 private void initPool(HostAndPort master) {
    synchronized(initPoolLock){
        //master与currentHostMaster比较，master没有改变则不需要initPool
        //  private volatile HostAndPort currentHostMaster;
      if (!master.equals(currentHostMaster)) {//
        currentHostMaster = master;
          //首次调用，实例化Jedis工厂
        if (factory == null) {
          factory = new JedisFactory(master.getHost(), master.getPort(), connectionTimeout,
              soTimeout, password, database, clientName);
          initPool(poolConfig, factory);
        } else {
           //非首次调用，修改工厂设置
          factory.setHostAndPort(currentHostMaster);
          // although we clear the pool, we still have to check the
          // returned object
          // in getResource, this call only clears idle instances, not
          // borrowed instances
          internalPool.clear();
        }

        log.info("Created JedisPool to master at " + master);
      }
    }
  }

```



* 1)、master与实例变量currentHostMaster作比较，只有当master值改变后，才进入方法调用`initPool()`方法;

* 2)、如果是第一次调用`initPool`方法(构造函数中调用)，那么会初始化Jedis实例创建工厂，如果不是第一次调用(`MasterListener`中调用)，那么只对已经初始化的工厂进行重新设置。

* 3)、从以上也可以看出为什么`currentHostMaster`和`factory`这两个变量为什么要声明为`volatile`，它们会在多线程环境下被访问和修改，因此必须保证`可见性`。


首次调用initPool()方法，是在 JedisSentinelPool构造方法中，进入判断逻辑里开始调用`initPool(poolConfig, factory);`，此处调用的是Pool.java里的方法，用来初始化内部对象池。

```java
//Pool.java
public void initPool(final GenericObjectPoolConfig poolConfig, PooledObjectFactory<T> factory) {

    if (this.internalPool != null) {
      try {
        closeInternalPool();
      } catch (Exception e) {
      }
    }

    this.internalPool = new GenericObjectPool<T>(factory, poolCofig);
  }
```



## 结语：

通过源码的分析，能使我们对Redis的Sentinel原理更加清楚，后续将继续对Cluster的原理进行源码分析。