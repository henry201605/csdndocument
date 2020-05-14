解决 because there was insufficient free space available after evicting expired cache entries - consider increasing the maximum size of the cache的问题


```shell
19-Mar-2020 16:48:06.947 WARNING [localhost-startStop-1] org.apache.catalina.webresources.Cache.getResource Unable to add the resource at [/WEB-INF/classes/templates/error/500.ftl] to the cache for web application [/henry] because there was insufficient free space available after evicting expired cache entries - consider increasing the maximum size of the cache
19-Mar-2020 16:48:06.947 WARNING [localhost-startStop-1] org.apache.catalina.webresources.Cache.getResource Unable to add the resource at [/WEB-INF/classes/templates/quebank/add-que-new.ftl] to the cache for web application [/henry] because there was insufficient free space available after evicting expired cache entries - consider increasing the maximum size of the cache
```

以上出现了Tomcat静态空间不足的情况，可以到Tomcat中conf文件夹下的context.xml文件做如下修改：

```xml
<Context>  
    <WatchedResource>WEB-INF/web.xml</WatchedResource>
    <WatchedResource>${catalina.base}/conf/web.xml</WatchedResource>
    <Resources
        cachingAllowed="true"  
        cacheMaxSize="100000"
    />
</Context>

```

参数：

1、cachingAllowed：设置是否允许静态资源缓存，默认为true；

2、cacheMaxSize：设置静态资源缓存的最大值，默认值为10240 ，单位是KB



除了上面的解决方案，还可以通过以下方案来解决：直接关闭缓存，设置 cachingAllowed为false。



