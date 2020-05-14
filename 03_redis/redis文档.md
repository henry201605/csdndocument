embstr内部编码

# Redis小于等于44个字节的字符串是embstr编码、大于44个字节是raw编码

## 1、字符串编码类型

字符串类型的内部编码有三种：
1、int，存储 8 个字节的长整型（long，2^63-1）。
2、embstr, 代表 embstr 格式的 SDS（Simple Dynamic String 简单动态字符串），
存储小于 44 个字节的字符串，只分配一次内存空间（因为 Redis Object 和 SDS 是连续的）。
3、raw，存储大于 44 个字节的字符串（3.2 版本之前是 39 个字节），需要分配两次内存空间（分别为 Redis Object 和 SDS 分配空间）。

## 2、embstr结构

今天主要是来研究一下，embstr编码的问题，先来看一下embstr的内存结构：

![image-20200331161704547](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200331161704547.png)

embstr分配的是连续的一块内存，包含redisObject和sds,redisObject占用了16个字节，先来看一下redisObject的存储结构：

```java
typedef struct redisObject {
    /*对象的类型*/
    unsigned type:4;
    /*具体的数据结构，embstr、sds、*/
    unsigned encoding:4;
    /* 24位，对象最后一次被命令程序访问的时间，与内存回收有关*/
    /* LRU time (relative to global lru_clock) or
     * LFU data (least significant 8 bits frequency
     * and most significant 16 bits access time). */
    unsigned lru:LRU_BITS;
    /*引用计数。当refcount为0的时候，表示该对象已经不被任何对象引用，则可以进行垃圾回收了*/
    int refcount;
    /*指向对象实际的数据结构*/
    void *ptr;
} robj;
```

下面再来看一下sds的结构:

### 2.1 Reids3.2之前版本

因为在之前很多书上都会说存储小于39个字节，编码为embstr，超过39个字节，编码为raw。这是在3.2版本之前。下面我们就先来看一下之前的版本：

```java
/*
 * 保存字符串对象的结构
 */
struct sdshdr {
    // buf 中已占用空间的长度
    unsigned int len;
    // buf 中剩余可用空间的长度
    unsigned int free;
    // 数据空间
    char buf[];
};

```

先来剖析一下为什么之前embstr在存储小于39个字符都是embstr的，大于39个之后就是raw的了。

```shell
 sdshdr = unsigned int * 2 = 4 * 2 = 8
```

embstr ： redisObject（16）+sds{char(39个) + 8 + 1} =64

此时可以看出如果是39个字符就是64个字节，那么也就是说只要是小于39个字符的，分配的空间都是64个字节。超过64个字节就为raw。



### 2.2 Reids3.2之后版本

本文选用的源代码是5.0.8版本，在 3.2 以后的版本中，SDS 又有多种结构（sds.h文件中）：sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64，用于存储不同的长度的字符串，分别代表 2^5=32byte，2^8=256byte，2^16=65536byte=64KB，2^32byte=4GB。先来查看一下sds的结构：

```shell
/* object.c */
#define OBJ_ENCODING_EMBSTR_SIZE_LIMIT 44
```

```java
struct __attribute__ ((__packed__)) sdshdr8 {
    //当前字符数组的长度
    uint8_t len; /* used */
//    当前字符数组总共分配的内存大小
    uint8_t alloc; /* excluding the header and null terminator */
//    当前字符数组的属性、用来标识到底是 sdshdr8 还是 sdshdr16 等
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
//    字符串真正的值
    char buf[];
};
```



我们来看一下5.0的中的sds的结构，经过分析可以看到sds使用了uint8_t代替int,占用的内存空间更小了，下面计算一下sds除字符串内容，占用的空间大小：sdsdr8 = uint8_t （1个字节）* 2 + char（1个字节） = 3个字节

由于对sds的结构做了调整，所以在原来39的基础上又可以多存储了5个字节，为44个字节。

从2.4版本开始，redis开始使用jemalloc内存分配器。这个比glibc的malloc要好不少，还省内存。在这里可以简单理解，jemalloc会分配8，16，32，64等字节的内存。

所以embstr分配的最新内存为：

redisObject（16）+sds（uint8_t （1个字节）* 2 + char（1个字节）+8(最小分配内存) +1（\0结束符）)=28个字节；

## 3、疑惑

写到这里突然发现一个问题，因为redis中提供了sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64，五种用于存储不同的长度的sds字符串。现在最小的大小为28个字节，会不会使用sdshdr5来进行存储呢，那如果是这样的话，我们之前最小字节数又得重新计算，因为sdshdr5()的结构跟其他几个有所不同,如下

```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
```

不过我们仔细看下上面的注释：`sdshdr5 is never used, we just access the flags byte directly`.这个方法居然从未没使用，感觉有点奇怪，只能继续扒代码，然后有一个重大发现：

```c
//在sds.c文件中
sds sdsnewlen(const void *init, size_t initlen) {
    void *sh;
    sds s;
    char type = sdsReqType(initlen);
    /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
    
    。。。。。。。。。。。。。。。。。。。。。。。。。。。
    switch(type) {
        case SDS_TYPE_5: {
            *fp = type | (initlen << SDS_TYPE_BITS);
            break;
        }
        case SDS_TYPE_8: {
            SDS_HDR_VAR(8,s);
            sh->len = initlen;
            sh->alloc = initlen;
            *fp = type;
            break;
        }
      。。。。。。。。。。。。。。。。。。。。。。。
    }
    if (initlen && init)
        memcpy(s, init, initlen);
    s[initlen] = '\0';
    return s;
}
```

上面可以看到：

```c
  /* Empty strings are usually created in order to append. Use type 8
     * since type 5 is not good at this. */
    if (type == SDS_TYPE_5 && initlen == 0) type = SDS_TYPE_8;
```

当initlen=0的时候，直接使用的是SDS_TYPE_8，所以头字节信息还是占用的3个字节，验证我们上面的推断是没有问题的。

## 4、结论

`embstr`存储形式是这样一种存储形式，它将 **RedisObject 对象头和 SDS 对象连续存在一起**，使用 malloc 方法一次分配。`embstr`的最小占用空间为19（16+3），而64-19-1（结尾的\0）=44，所以empstr只能容纳44字节。

当字节数小于44时，分配的大小一直都是64个字节。一旦超过44个字节，整体的大小超过64个字节，在Redis中将认为是一个大的字符串，不再使用 emdstr 形式存储，存储结构将变为raw。