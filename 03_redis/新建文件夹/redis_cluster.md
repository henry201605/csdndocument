

Redis之Cluster应用及源码分析



Jedis 有4 种工作模式：单节点、分片（Sharded）、哨兵(Sentinel)、集群(Cluster)。前面两篇我们分别对Sharded、Sentinel通过实际应用的例子进行了源码剖析，本文将通过实际的例子对Redis的Sentinel进行源码剖析。

参考：



## 1、环境：

对于Redis Cluster的搭建请参考https://blog.csdn.net/qq_33996921/article/details/105462595

1、操作系统：Centos7.7

2、服务器配置如下：

| 角色   |       ip        | 端口 |
| ------ | :-------------: | :--- |
| master | 132.232.125.196 | 6373 |
| master | 132.232.125.196 | 6374 |
| master | 132.232.125.196 | 6375 |
| slave  | 132.232.125.196 | 6376 |
| slave  | 132.232.125.196 | 6377 |
| slave  | 132.232.125.196 | 6378 |

3主3从的Redis集群已经搭建好了。还可以查看出master的slot分布

| 角色   | 端口 | 位置        | 数量 |
| ------ | :--- | ----------- | ---- |
| master | 6373 | 0-5460      | 5461 |
| master | 6374 | 5461-10922  | 5462 |
| master | 6375 | 10923-16383 | 5461 |

## 2、启动redis服务

```shell
[root@m ~]# ps -ef|grep redis
server *:6378 [cluster]
root     15226     1  0 23:22 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6377 [cluster]
root     15231     1  0 23:22 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6376 [cluster]
root     15236     1  0 23:22 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6375 [cluster]
root     15241     1  0 23:22 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6374 [cluster]
root     15246     1  0 23:22 ?        00:00:00 /usr/local/soft/redis-5.0.8/src/redis-server *:6373 [cluster]
root     15528 15422  0 23:23 pts/0    00:00:00 grep --color=auto redis

```



## 3、测试代码

```shell
package nci.henry;

import org.junit.Before;
import org.junit.Test;
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.JedisSentinelPool;

import java.io.IOException;
import java.util.Date;
import java.util.HashSet;
import java.util.Set;

/**
 * @Author: henry
 * @Date: 2020/4/18 22:43
 * @Description: 测试Cluster
 */
public class JedisClusterTest {


    private JedisCluster cluster;

    @Before
    public void initJedis(){
        // 不管是连主备，还是连几台机器都是一样的效果
        HostAndPort hap1 = new HostAndPort("132.232.125.196",6373);
        HostAndPort hap2 = new HostAndPort("132.232.125.196",6374);
        HostAndPort hap3 = new HostAndPort("132.232.125.196",6375);
        HostAndPort hap4 = new HostAndPort("132.232.125.196",6376);
        HostAndPort hap5 = new HostAndPort("132.232.125.196",6377);
        HostAndPort hap6 = new HostAndPort("132.232.125.196",6378);

        Set nodes = new HashSet<HostAndPort>();
        nodes.add(hap1);
        nodes.add(hap2);
        nodes.add(hap3);
        nodes.add(hap4);
        nodes.add(hap5);
        nodes.add(hap6);

        cluster = new JedisCluster(nodes);
    }

    @Test
    public void testGet(){
        try {
            String key = "cluster:henry";
            cluster.set(key, "henry2016");
            System.out.println("获取集群中" + key + "的值--->"+cluster.get(key));;
            cluster.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

```



执行 testGet()方法，结果如下：

```shell
获取集群中cluster:henry的值--->henry2016
```

我们尝试一下只连一台主机（主从都可以），修改上述代码中的initJedis()方法

```java
  @Before
    public void initJedis(){
        // 不管是连主备，还是连几台机器都是一样的效果
//        HostAndPort hap1 = new HostAndPort("132.232.125.196",6373);
//        HostAndPort hap2 = new HostAndPort("132.232.125.196",6374);
//        HostAndPort hap3 = new HostAndPort("132.232.125.196",6375);
//        HostAndPort hap4 = new HostAndPort("132.232.125.196",6376);
//        HostAndPort hap5 = new HostAndPort("132.232.125.196",6377);
        HostAndPort hap6 = new HostAndPort("132.232.125.196",6378);

        Set nodes = new HashSet<HostAndPort>();
//        nodes.add(hap1);
//        nodes.add(hap2);
//        nodes.add(hap3);
//        nodes.add(hap4);
//        nodes.add(hap5);
        nodes.add(hap6);

        cluster = new JedisCluster(nodes);
    }
```

发现结果还是：

```shell
获取集群中cluster:henry的值--->henry2016
```

`疑问`：

上述代码在使用Jedis 连接Cluster 的时候，只需要连接到任意一个或者多个redis group 中的实例地址，就可以进行redis的操作，那我们是怎么获取到需要操作的Redis Master 实例的？带着这样的疑问，下面我们开始进行源码的剖析。

## 4、源码分析

### 4.1 类结构图

![image-20200419002325188](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200419002325188.png)

1）从图中可以看出，JedisCluster主要是继承二进制的BinaryJedisCluster类，这个类中的各种操作都是基于字节数组方式进行的。而且BinaryJedisCluster类实现的4个接口中有3个是基于字节数组操作。

2）JedisCluster实现了JedisCommands，MultiKeyJedisClusterCommands，JedisClusterScriptingCommands接口。这三个接口提供的基于字符串类型的操作，即key都是字符串类型。

3）BasicCommands是关于redis服务本身基本操作，比如save,ping,bgsave等操作。

4）MultiKeyBinaryJedisClusterCommands和MultiKeyJedisClusterCommands接口一个字节数组的批量操作，一个是字符串的批量操作。


### 4.2 JedisCluster构造方法

```java
//测试代码中initJedis（）方法
cluster = new JedisCluster(nodes)
```

先来看下`JedisCluster`的构造方法:

```java
public JedisCluster(Set<HostAndPort> nodes) {
    this(nodes, DEFAULT_TIMEOUT);
  }
```

构造方法最终的实现在BinaryJedisCluster中的构造方法。

```java
 public BinaryJedisCluster(Set<HostAndPort> jedisClusterNode, int timeout, int maxAttempts,
      final GenericObjectPoolConfig poolConfig) {
    this.connectionHandler = new JedisSlotBasedConnectionHandler(jedisClusterNode, poolConfig,
        timeout);
    this.maxAttempts = maxAttempts;
  }
```

此构造方法又调用了JedisClusterConnectionHandler中的构造方法。

```java
  public JedisClusterConnectionHandler(Set<HostAndPort> nodes,
      final GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout, String password, String clientName,
      boolean ssl, SSLSocketFactory sslSocketFactory, SSLParameters sslParameters,
      HostnameVerifier hostnameVerifier, JedisClusterHostAndPortMap portMap) {
    this.cache = new JedisClusterInfoCache(poolConfig, connectionTimeout, soTimeout, password, clientName,
        ssl, sslSocketFactory, sslParameters, hostnameVerifier, portMap);
    initializeSlotsCache(nodes, connectionTimeout, soTimeout, password, clientName, ssl, sslSocketFactory, sslParameters, hostnameVerifier);
  }
```

该构造方法中调用了`initializeSlotsCache()`方法，此方法用来初始化集群环境。

### 4.3 initializeSlotsCache():初始化集群环境

```java
private void initializeSlotsCache(Set<HostAndPort> startNodes,
      int connectionTimeout, int soTimeout, String password, String clientName,
      boolean ssl, SSLSocketFactory sslSocketFactory, SSLParameters sslParameters, HostnameVerifier hostnameVerifier) {
    for (HostAndPort hostAndPort : startNodes) {
      Jedis jedis = null;
      try {
        // 获取一个Jedis 实例
        jedis = new Jedis(hostAndPort.getHost(), hostAndPort.getPort(), connectionTimeout, soTimeout, ssl, sslSocketFactory, sslParameters, hostnameVerifier);
        if (password != null) {
          jedis.auth(password);
        }
        if (clientName != null) {
          jedis.clientSetname(clientName);
        }
        // 获取Redis节点和Slot虚拟槽
        cache.discoverClusterNodesAndSlots(jedis);
        // 直接跳出循环
        break;
      } catch (JedisConnectionException e) {
        // try next nodes
      } finally {
        if (jedis != null) {
          jedis.close();
        }
      }
    }
  }
```

从上面的代码中我们可以看出：

1、无论是主从，无论多少个节点，只要拿到第一个，获取redis连接实例，后面直接break，所以我们在应用中只需连接集群中的任意一个节点就可以对集群进行操作，这也解答了第3节中的疑惑；

2、调用JedisClusterInfoCache的discoverClusterNodesAndSlots()方法来获取所有Redis节点和Slot虚拟槽；

### 4.4 discoverClusterNodesAndSlots()方法

此方法主要是用来获取集群中所有Redis节点和Slot虚拟槽，源码如下：

```java
public void discoverClusterNodesAndSlots(Jedis jedis) {
    w.lock();

    try {
      reset();
      //根据当前redis实例，获取集群中master,slave节点信息。包括每个master节点 上分配的数据嘈
      List<Object> slots = jedis.clusterSlots();

      // 遍历master 节点
      for (Object slotInfoObj : slots) {
        // slotInfo 槽开始，槽结束，主，从
        List<Object> slotInfo = (List<Object>) slotInfoObj;

        // 如果<=2，代表没有分配slot
        if (slotInfo.size() <= MASTER_NODE_INDEX) {
          continue;
        }

        // 获取分配到当前master 节点的数据槽，例如6373 节点的{0,1,2,3……5460}
        List<Integer> slotNums = getAssignedSlotArray(slotInfo);

        // hostInfos
        // size 是4，依次为槽最小（0）、最大（1），主（2），从（3）
        int size = slotInfo.size();
        // 第3 位和第4 位是主从端口的信息
        for (int i = MASTER_NODE_INDEX; i < size; i++) {
          List<Object> hostInfos = (List<Object>) slotInfo.get(i);
          if (hostInfos.size() <= 0) {
            continue;
          }
          // 根据IP 端口生成HostAndPort 实例
          HostAndPort targetNode = generateHostAndPort(hostInfos);
          // 据HostAndPort 解析出ip:port 的key 值，再根据key 从缓存中查询对应的jedisPool 实例。如果没有jedisPool
//          实例，就创建JedisPool 实例，最后放入缓存中。nodeKey 和nodePool 的关系
          setupNodeIfNotExist(targetNode);
          if (i == MASTER_NODE_INDEX) {
            // 把slot和jedisPool缓存起来（16384 个），key 是slot 下标，value 是连接池
            assignSlotsToNode(slotNums, targetNode);
          }
        }
      }
    } finally {
      w.unlock();
    }
  }
```

此方法比较长，下面来描述一下该方法的流程：

#### 4.4.1 clusterSlots()方法

1、用获取的redis 连接实例执行`clusterSlots ()`方法，实际执行redis 服务端clusterslots 命令，获取虚拟槽信息；

```java
List<Object> slots = jedis.clusterSlots();
```

该集合的大小为4，基本信息为[long, long, List, List], 第一，二个元素是该节点负责槽点的起始位置，第三个元素是主节点信息，第四个元素为主节点对应的从节点信息。该list 的基本信息为[string,int,string],第一个为host 信息，第二个为port 信息，第三个为唯一id。

2、代码debug，获取值

![image-20200419005940883](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200419005940883.png)



3、通过redis的客户端查看一下集群的信息：

```shell
132.232.115.96:6378> cluster slots
	`槽的起点位置`
1) 1) (integer) 5795  
	`槽的终点位置`
   2) (integer) 10922
   	 `master节点的ip`
   3) 1) "132.232.115.96"
  	   `master节点的端口号`
      2) (integer) 6374
      `master节点的id`
      3) "a22f06edc5ae87f16703f4bcaf6e9de30d944c63"
      `slave节点的ip`
   4) 1) "132.232.115.96"
      `slave节点的端口号`
      2) (integer) 6377
       `slave节点的id`
      3) "062719135b227803a1e91f18606d58bd455e5ae9"
2) 1) (integer) 0
   2) (integer) 332
   3) 1) "132.232.115.96"
      2) (integer) 6372
      3) "ede53d3ca156bfe99493209aaf99e51ea9b2a8e9"
..............................
```



通过代码以及redis客户端查看，可以清晰的看出slots中的具体含义。

#### 4.4.2 获取slot槽信息

获取有关节点的槽点信息后，调用`getAssignedSlotArray(slotinfo)`来获取所有的槽点值。

```java
 private List<Integer> getAssignedSlotArray(List<Object> slotInfo) {
    List<Integer> slotNums = new ArrayList<Integer>();
    for (int slot = ((Long) slotInfo.get(0)).intValue(); slot <= ((Long) slotInfo.get(1))
        .intValue(); slot++) {
      slotNums.add(slot);
    }
    return slotNums;
  }
```

#### 4.4.3  获取主节点信息

根据ip和端口号，获取主节点的地址信息，调用generateHostAndPort(hostInfo)方法，生成一个hostAndPort 对象。

```java
 private HostAndPort generateHostAndPort(List<Object> hostInfos) {
    String host = SafeEncoder.encode((byte[]) hostInfos.get(0));
    int port = ((Long) hostInfos.get(1)).intValue();
    if (ssl && hostAndPortMap != null) {
      HostAndPort hostAndPort = hostAndPortMap.getSSLHostAndPort(host, port);
      if (hostAndPort != null) {
        return hostAndPort;
      }
    }
    return new HostAndPort(host, port);
  }
```

#### 4.4.4 设置jedisPool

再根据节点地址信息来设置节点对应的JedisPool ， 即设置Map<String,JedisPool> nodes 的值。

```java
 public JedisPool setupNodeIfNotExist(HostAndPort node) {
    w.lock();
    try {
      String nodeKey = getNodeKey(node);
      JedisPool existingPool = nodes.get(nodeKey);
      if (existingPool != null) return existingPool;

      JedisPool nodePool = new JedisPool(poolConfig, node.getHost(), node.getPort(),
          connectionTimeout, soTimeout, password, 0, clientName, 
          ssl, sslSocketFactory, sslParameters, hostnameVerifier);
      nodes.put(nodeKey, nodePool);
      return nodePool;
    } finally {
      w.unlock();
    }
  }
```

#### 4.4.5 设置slot和jedisPool关系

接下来判断，若此时节点信息为master主节点信息时，则调用assignSlotsToNodes 方法，把slot和jedisPool缓存起来，设置每个槽点值对应的连接池，即设置Map<Integer, JedisPool> slots 的值

```java
 public void assignSlotsToNode(List<Integer> targetSlots, HostAndPort targetNode) {
    w.lock();
    try {
      JedisPool targetPool = setupNodeIfNotExist(targetNode);
      for (Integer slot : targetSlots) {
        slots.put(slot, targetPool);
      }
    } finally {
      w.unlock();
    }
  }
```



以上为初始化集群环境的操作，下面开始进行执行set的操作。



### 4.5 set()方法操作集群

```java
cluster.set(key, "henry2016");
```

调用JedisCluster.java的`set()`方法

```java
@Override
  public String set(final String key, final String value) {
    return new JedisClusterCommand<String>(connectionHandler, maxAttempts) {
      @Override
      public String execute(Jedis connection) {
        return connection.set(key, value);
      }
    }.run(key);
  }
```

此方法中采用JedisClusterCommand匿名内部类的方式来实现父类中的`execute()`抽象方法，供JedisClusterCommand中的`runWithRetries()`的方法来调用。（后面会说到）

#### 4.5.1 JedisClusterCommand类

先来介绍一下JedisClusterCommand类：

* 1）在JedisCluster客户端中，JedisClusterCommand是一个非常重要的类，采用模板方法设计，高度封装了操作Redis集群的操作。对于集群中各种存储操作，提供了一个抽象execute方法。
* 2）JedisCluster各种具体操作Redis集群方法，只需要通过匿名内部类的方式，灵活扩展Execute方法。
* 3）内部通过JedisClusterConnectionHandler封装了Jedis的实例。


下面开始接着上面的调用进行分析：

JedisCluster的`set()`方法调用了JedisClusterCommand类的`run()`方法，

```java
  public T run(String key) {
    return runWithRetries(JedisClusterCRC16.getSlot(key), this.maxAttempts, false, null);
  }
```

#### 4.5.2 runWithRetries()方法

简单介绍一下改方法的功能：

>  1） 该方法采用递归方式，保证在往集群中存取数据时，发生MOVED，ASKing,数据迁移过程中遇到问题，
>    也是一种实现高可用的方式。

> 2）该方法中调用execute方法，该方法由子类具体实现。

```java
  /**
   *
   * @param slot
   *            要操作的槽
   * @param attempts
   *            重试次数，每重试一次减1
   * @param tryRandomNode
   *            标识是否随机获取活跃节点连接，true为是，false为否
   * @param redirect
   * @return
   */
  private T runWithRetries(final int slot, int attempts, boolean tryRandomNode, JedisRedirectionException redirect) {
    if (attempts <= 0) {
      throw new JedisClusterMaxAttemptsException("No more cluster attempts left.");
    }

    Jedis connection = null;
    try {
      /**
       *第一执行该方法，asking为false。只有发生JedisAskDataException
       *异常时，才asking才设置为true
       **/
      if (redirect != null) {
        connection = this.connectionHandler.getConnectionFromNode(redirect.getTargetNode());
        if (redirect instanceof JedisAskDataException) {
          // TODO: Pipeline asking with the original command to make it faster....
          connection.asking();
        }
      } else {
        // 第一次执行时，tryRandomNode为false。
        if (tryRandomNode) {
          connection = connectionHandler.getConnection();
        } else {
//          根据key获取分配的槽数，然后根据数据槽从JedisClusterInfoCache 中获取Jedis的实例
          connection = connectionHandler.getConnectionFromSlot(slot);
        }
      }
      /**调用子类方法的具体实现（********）*/
      return execute(connection);

    } catch (JedisNoReachableClusterNodeException jnrcne) {
      throw jnrcne;
    } catch (JedisConnectionException jce) {
      // release current connection before recursion
      //释放已有的连接
      releaseConnection(connection);
      connection = null;
      /***
       ***只是重建键值对slot-jedis缓存即可。已经没有剩余的redirection了。
       ***已经达到最大的MaxRedirection次数，抛出异常即可。
       ***/
      if (attempts <= 1) {
        //We need this because if node is not reachable anymore - we need to finally initiate slots
        //renewing, or we can stuck with cluster state without one node in opposite case.
        //But now if maxAttempts = [1 or 2] we will do it too often.
        //TODO make tracking of successful/unsuccessful operations for node - do renewing only
        //if there were no successful responses from this node last few seconds
        this.connectionHandler.renewSlotCache();
      }
      // 递归调用该方法
      return runWithRetries(slot, attempts - 1, tryRandomNode, redirect);
    } catch (JedisRedirectionException jre) {
      // if MOVED redirection occurred,
      // 发生MovedException,需要重建键值对slot-Jedis的缓存。
      if (jre instanceof JedisMovedDataException) {
        // it rebuilds cluster's slot cache recommended by Redis cluster specification
        this.connectionHandler.renewSlotCache(connection);
      }

      // release current connection before recursion
      releaseConnection(connection);
      connection = null;
      // 递归调用。
      return runWithRetries(slot, attempts - 1, false, jre);
    } finally {
      releaseConnection(connection);
    }
  }   
```

上述代码有详细的注释，此处只重点提一下，该方法中的两个方法：

#### 4.5.3   getConnectionFromSlot()

1、方法中调用`JedisClusterConnectionHandler`的`getConnectionFromSlot(slot)`方法，通过槽的位置来获取redis的实例；

```java
@Override
  public Jedis getConnectionFromSlot(int slot) {
    JedisPool connectionPool = cache.getSlotPool(slot);
    if (connectionPool != null) {
      // It can't guaranteed to get valid connection because of node
      // assignment
      return connectionPool.getResource();
    } else {
      renewSlotCache(); //It's abnormal situation for cluster mode, that we have just nothing for slot, try to rediscover state
      connectionPool = cache.getSlotPool(slot);
      if (connectionPool != null) {
        return connectionPool.getResource();
      } else {
        //no choice, fallback to new connection to random node
        return getConnection();
      }
    }
  }
```

上文中可以看到获取指定的key 存放的位置，是从一个暂存池中获取到指定的连接的。

```java
 JedisPool connectionPool = cache.getSlotPool(slot);
```



2、调用了`execute(connection);`方法，这个方法调用的就是前面在4.5节中所说的，调用子类实现的execute方法。



#### 4.5.2 set流程

1）把key 作为参数，执行CRC16 算法，获取key 对应的slot 值；

2）通过该slot 值，去slots 的map 集合中获取jedisPool 实例；

3）通过jedisPool 实例获取jedis 实例，最终完成redis 数据存取工作；



## 结语：

通过源码的分析，能使我们对Redis的Cluster原理更加清楚，到目前为止我们已经将Redis的Sharded、Sentinel的源码进行了分析。参考如下：



