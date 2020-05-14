

Redis之Sharded应用及源码分析



Jedis 有4 种工作模式：单节点、分片（Sharded）、哨兵(Sentinel)、集群(Cluster)。单节点比较简单，本章我们先从分片开始讲解。

## 1、环境：

1、操作系统：Centos7.7

2、服务器配置如下：（本文采用了在单台机子上部署多个实例来模拟多机部署）

| 主机  |       ip        | 角色和端口 |
| ----- | :-------------: | :--------- |
| node1 | 132.232.125.196 | 9526       |
| node2 | 132.232.125.196 | 9527       |



## 2、启动redis服务

```shell
[root@m 9526]# ps -ef|grep redis
root       937     1  0 Apr14 ?        00:06:04 /usr/local/bin/redis-server 0.0.0.0:6379
root      9426     1  0 14:20 ?        00:00:03 ./redis-5.0.8/src/redis-server *:9526
root      9574     1  0 14:20 ?        00:00:03 ./redis-5.0.8/src/redis-server *:9527
root     18522   813  0 15:20 pts/2    00:00:00 grep --color=auto redis
```

## 3、测试代码

```shell
import org.junit.After;
import org.junit.Before;
import org.junit.Test;
import redis.clients.jedis.*;

import java.util.Arrays;
import java.util.List;

/**
 * @Author: henry
 * @Date: 2020/4/18 15:43
 * @Description: 测试ShardedJedis
 */
public class ShardedJedisTest {

    private ShardedJedisPool shardedJedisPool;
    private ShardedJedis shardedJedis;

    @Before
    public void initJedis(){
        JedisPoolConfig poolConfig = new JedisPoolConfig();

        // Redis服务器
        JedisShardInfo shardInfo1 = new JedisShardInfo("132.232.115.96", 9527);
        JedisShardInfo shardInfo2 = new JedisShardInfo("132.232.115.96", 9526);

        // 连接池
        List<JedisShardInfo> infoList = Arrays.asList(shardInfo1, shardInfo2);
        shardedJedisPool = new ShardedJedisPool(poolConfig, infoList);
    }

	//测试向redis中set值
    @Test
    public void testSet(){
        try {
            shardedJedis = shardedJedisPool.getResource();
            for (int i = 0; i < 100; i++) {
                shardedJedis.set("k" + i, "" + i);
                //根据key获取主机的信息
                Client client = shardedJedis.getShard("k"+i).getClient();
                System.out.println("set值："+i+ "，到：" + client.getHost() + ":" + client.getPort());
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

	//测试从redis中get值
    @Test
    public void testGet(){
        try{
            shardedJedis = shardedJedisPool.getResource();
            for(int i=0; i<100; i++){
                //根据key获取主机的信息
                Client client = shardedJedis.getShard("k"+i).getClient();
                System.out.println("取到值："+shardedJedis.get("k"+i)+"，"+"当前key位于：" + client.getHost() + ":" + client.getPort());
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @After
    public void close(){
        if(shardedJedis!=null) {
            shardedJedis.close();
        }
    }
}

```



执行 testGet()方法，结果如下：

```shell
set值：0，到：132.232.115.96:9527
set值：1，到：132.232.115.96:9527
set值：2，到：132.232.115.96:9527
set值：3，到：132.232.115.96:9526
set值：4，到：132.232.115.96:9526
set值：5，到：132.232.115.96:9526
set值：6，到：132.232.115.96:9527
set值：7，到：132.232.115.96:9527
set值：8，到：132.232.115.96:9526
set值：9，到：132.232.115.96:9526
set值：10，到：132.232.115.96:9526
。。。。。。。。。
```

执行testGet()方法，结果如下：

```shell
取到值：0，当前key位于：132.232.115.96:9527
取到值：1，当前key位于：132.232.115.96:9527
取到值：2，当前key位于：132.232.115.96:9527
取到值：3，当前key位于：132.232.115.96:9526
取到值：4，当前key位于：132.232.115.96:9526
取到值：5，当前key位于：132.232.115.96:9526
取到值：6，当前key位于：132.232.115.96:9527
取到值：7，当前key位于：132.232.115.96:9527
取到值：8，当前key位于：132.232.115.96:9526
取到值：9，当前key位于：132.232.115.96:9526
取到值：10，当前key位于：132.232.115.96:9526
。。。。。。。。。。。。。
```



从上面结果可以看出，key基本是均匀的分布到不同的Redis实例上，下面我们通过分析源码，看一下shard具体实现方式。



## 4、源码分析

### 4.1 类结构图

![image-20200418154528318](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200418154528318.png)



上图为ShardedJedis的类结构（其内部保存一个对象池，与常规的JedisPool的不同之处在于，内部的`PooledObjectFactory`实现不同）,分片信息保存在基类`Sharded`中，先来看下Sharded的构造方法。

### 4.2 构造方法

```java
  public ShardedJedis(List<JedisShardInfo> shards, Hashing algo, Pattern keyTagPattern) {
    super(shards, algo, keyTagPattern);
  }
```



* 1、shards，是一个JedisShardInfo的列表，一个JedisShardedInfo类代表一个数据分片的主体，它可以指定redis的服务信息对象的list集合（ShardInfo的子类，如JedisShardInfo，存放了redis子节点的ip、端口、weight等信息）。

* 2、hash算法（默认一致性hash），jedis中指定了两种hash实现，一种是一致性hash，一种是基于md5的实现，在redis.clients.util.Hashing中指定的。

* 3、tagPattern，可以指定按照key的某一部分进行hash分片（比如我们可以将以order开头的key分配到redis节点1上，可以将以product开头的key分配到redis节点2上），默认情况下是根据整个key进行hash分片的。这样通过合理命名key，可以将一组相关联的key放入同一个Redis节点，这在避免跨节点访问相关数据时很重要。



ShardedJedis()构造最终调用的是Sharded的构造方法：

```java
public Sharded(List<S> shards, Hashing algo, Pattern tagPattern) {
    this.algo = algo;
    this.tagPattern = tagPattern;
    initialize(shards);
  }
```

在构造方法中一个初始化的方法，是用来初始化分片的。

### 4.3 分片初始化

```java
 private void initialize(List<S> shards) {
    nodes = new TreeMap<Long, S>();

    for (int i = 0; i != shards.size(); ++i) {
      final S shardInfo = shards.get(i);
      if (shardInfo.getName() == null) for (int n = 0; n < 160 * shardInfo.getWeight(); n++) {
        nodes.put(this.algo.hash("SHARD-" + i + "-NODE-" + n), shardInfo);
      }
      else for (int n = 0; n < 160 * shardInfo.getWeight(); n++) {
        nodes.put(this.algo.hash(shardInfo.getName() + "*" + n), shardInfo);
      }
      resources.put(shardInfo, shardInfo.createResource());
    }
  }
```



- 1、首先根据redis节点集合信息创建虚拟节点（一致性hash上0~2^32之间的点），通过上面的源码可以看出，根据每个redis节点的name计算出对应的hash值（如果没有配置节点名称，就是用默认的名字），并创建了160*weight个虚拟节点，weight默认情况下等于1，如果某个节点的配置较高，可以适当的提高虚拟节点的个数，将更多的请求打到这个节点上。

- 2、Sharded中使用TreeMap来实现hash环。

- 3、resources是一个LinkedHashMap，存放着JedisShardinfo和一个Jedis实例的对应关系。

  ```java
   private final Map<ShardInfo<R>, R> resources = new LinkedHashMap<ShardInfo<R>, R>();
  ```

  

### 4.4 分析测试代码

```java
 @Test
    public void testSet(){
        try {
            shardedJedis = shardedJedisPool.getResource();
            for (int i = 0; i < 100; i++) {
                shardedJedis.set("k" + i, "" + i);
                Client client = shardedJedis.getShard("k"+i).getClient();
                System.out.println("set值："+i+ "，到：" + client.getHost() + ":" + client.getPort());
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

#### 4.4.1  获取shardedJedis对象

先来看一下`shardedJedisPool.getResource();`,其执行流程如下：

![image-20200418164830023](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200418164830023.png)



通过上图我们可以看到执行到了Sharded的initialize()方法；初始化完成后，来查看一下shardedJedis的值：

![image-20200418165639572](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200418165639572.png)

> 1、nodes就是虚拟节点的个数：本测试用例中使用了两个redis实例，每个实例在调用initialize()初始化分片的时候，被分配了160个虚拟节点，此处有2个实例，因此有320个nodes；
>
> 2、默认使用的是一致性hash算法（MurmurHash）；
>
> 3、resouces中存放的是JedisShardinfo（key）和Jedis（value）实例的关系，是用一个LinkedHashMap进行存储。



#### 4.4.2 执行set操作

```java
 shardedJedis.set("k" + i, "" + i);
```

调用ShardedJedis中的set()方法

```java
//1、调用set
public String set(String key, String value) {
    Jedis j = getShard(key);
    return j.set(key, value);
  }

```

继续调用Sharded中的方法：

```java 
//1、调用getShard（），通过JedisShardInfor实例对象从resources的Map中获取到对应的Jedis实例对象：
   public R getShard(String key) {
    return resources.get(getShardInfo(key));
  }

//2、调用getShardInfo（），根据key获取到对应的JedisShardInfor对象：
 public S getShardInfo(String key) {
    return getShardInfo(SafeEncoder.encode(getKeyTag(key)));
  }

  /*
  * 3、在介绍Sharded的构造方法时，指定了一个tagPattern，它的作用就是在使用key进行分片操作时，
  * 可以根据key的一部分来计算分片，getKeyTag（）方法用来获取key对应的keytag，
  *默认情况下是根据整个key来分片,
  */
  public String getKeyTag(String key) {
    if (tagPattern != null) {
      Matcher m = tagPattern.matcher(key);
      if (m.find()) return m.group(1);
    }
    return key;
  }
 /**
   * 4、
   *（1）首先通过key或keytag计算出hash值；
   *（2）然后在TreeMap中找到比这个hash值大的第一个虚拟节点；
   * （这个过程就是在一致性hash环上顺时针查找的过程），如果这个hash值大于所有虚拟节点对应的hash，
   * 则使用第一个虚拟节点
   */
public S getShardInfo(byte[] key) {
    SortedMap<Long, S> tail = nodes.tailMap(algo.hash(key));
    if (tail.isEmpty()) {
      return nodes.get(nodes.firstKey());
    }
    return tail.get(tail.firstKey());
  }
```



#### 4.4.3 获取redis实例

```java
Client client = shardedJedis.getShard("k"+i).getClient();
```

调用sharded的getShard（）方法

```java
  public R getShard(String key) {
    return resources.get(getShardInfo(key));
  }
```

此处可以看到调用的方法跟4.4.2中set方法时一样，是用来获取redis实例的。



#### 4.4.4 执行get操作

```java
String value = shardedJedis.get("k" + i);
```

调用ShardedJedis中的set()方法：

```java 
  public String get(String key) {
    Jedis j = getShard(key);
    return j.get(key);
  }
```



上面可以看到还是需要先调用到Sharded中的getShard方法，这个可以看set中的源码分析。在获取到Jedis的实例后再调用jedis的get方法。



## 结语：

通过源码的分析，能使我们对Redis的分区原理更加清楚，后续将继续对Sentinel及Cluster的原理进行源码分析。