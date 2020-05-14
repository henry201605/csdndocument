在maven打包的时候有如下错误：

```shell
init......
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
Logging initialized using 'class org.apache.ibatis.logging.stdout.StdOutImpl' adapter.
。。。。。。。。。。。。。
```

先根据提示到提示的网址`http://www.slf4j.org/codes.html#StaticLoggerBinder`进行查看，发现如下：

```text
Failed to load class org.slf4j.impl.StaticLoggerBinder
This warning message is reported when the org.slf4j.impl.StaticLoggerBinder class could not be loaded into memory. This happens when no appropriate SLF4J binding could be found on the class path. Placing one (and only one) of slf4j-nop.jar slf4j-simple.jar, slf4j-log4j12.jar, slf4j-jdk14.jar or logback-classic.jar on the class path should solve the problem.
```

通过上面给出的解决方案，加上`slf4j-nop.jar`、`slf4j-simple.jar`, `slf4j-log4j12.jar`, `slf4j-jdk14.jar` or `logback-classic.jar`中的一个就可以了，本文加入如下jar包，完美解决。

```java
  <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>1.7.25</version>
            <scope>compile</scope>
  </dependency>
```

