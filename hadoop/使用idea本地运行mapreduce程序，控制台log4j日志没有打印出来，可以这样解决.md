https://blog.csdn.net/qq_31975963/article/details/83898920

官方给的解答是：我们的项目中没有找到log4j.properties或者log4j.xml等默认的配置文件。

加上一个 <font color="red">log4j.properties 文件</font>，项目使用 maven 构建的，我把 log4j.properties 放到了 <font color="red">resources 目录下面</font>。  
在 官网上 copy 一个例子过来：

```
# Set root logger level to DEBUG and its only appender to A1.
log4j.rootLogger=DEBUG, A1

# A1 is set to be a ConsoleAppender.
log4j.appender.A1=org.apache.log4j.ConsoleAppender

# A1 uses PatternLayout.
log4j.appender.A1.layout=org.apache.log4j.PatternLayout
log4j.appender.A1.layout.ConversionPattern=%-4r [%t] %-5p %c %x - %m%n
```
