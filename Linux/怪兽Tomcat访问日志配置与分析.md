# 1.配置日志文件位置与日志格式



在yml文件里修改

```
`server:   tomcat:     accesslog:       enabled: true       directory: /app/logs/tomcat/{app三字码}/       pattern: '%h "%{yyyy-MM-dd HH:mm:ss.SSS}t" "%r" %s %b %D'`
```

##  1.1 文件目录

参考： <https://docs.spring.io/spring-boot/docs/1.5.6.RELEASE/reference/htmlsingle/#howto-configure-accesslogs>

 SpringBoot内置Tomcat的日志位置默认在Tomcat Base文件夹下（默认未配置，为临时目录；可更改server.tomcat.basedir修改）。

建议通过server.tomcat.accesslog.directory直接修改accesslog文件位置



## 1.2 日志格式

参考：<https://tomcat.apache.org/tomcat-8.0-doc/config/valve.html#Access_Logging>



默认格式为: **'%h %l %u %t "%r" %s %b'**

日志类似于:10.208.203.46 - - [13/Oct/2017:15:31:05 +0800] "GET /health HTTP/1.1" 200 731



推荐使用格式：**'%h "%{yyyy-MM-dd HH:mm:ss.SSS}t" "%r" %s %b %D'**

格式含义：**访问ip 日期时间 Http请求第一行(请求方法+接口地址) 状态码 响应字节 响应毫秒数**



日志类似于：10.27.211.231 "2017-11-23 00:00:00.568" "POST /order/get HTTP/1.1" 200 381 2



有登陆鉴权操作的可以日志格式：**'%h "%{yyyy-MM-dd HH:mm:ss.SSS}t" %{userName}s "%r" %s %b %D'**

日志类似于:10.80.64.140 "2017-11-23 09:43:09.392" wanglin "OPTIONS /api/applications/b5f18f57/heapdump HTTP/1.0" 200 - 4

相比上面的格式增加了**%{userName}s**，表示从Session里取userName，当然前提是Session里有userName字段

# 2.日志分析

## 1.下载日志文件到qa机器

accesslog日志文件如果很大，直接在线上机器上分析会影响线上系统，因此下载至qa机器

在qa机器上执行

```
`scp root@{线上ip}:/app/logs/tomcat/{app三字码}/access_log.2017-11-22.log /app/logs/tomcat/prd-{app三字码}/access_log.2017-11-22.log`
```

## 2.常用分析脚本

awk命令参考：<http://blog.csdn.net/exceptional_derek/article/details/48470717>

按空格分开，日志格式为

```
`1	10.28.228.17 	ip 2	"2017-10-31 	日期 3	13:34:48.419" 	时间 4	"POST 			请求方法 5	/order/latest 	接口地址 6	HTTP/1.1" 		协议版本 7	200 			响应码 8	27				响应字节 9	2				耗时毫秒数`
```

### 2.1 每个 IP 访问的页面数倒序，前100条

```
`awk '{++S[$1]} END {for (a in S) print S[a],a}' access_log.2017-10-30.log | sort -rn|head -100`
```

2.2 响应时间超过 3 秒的请求

```
`awk '($9 > 3000){print $0}' access_log.2017-10-30.log`
```

2.3 某个请求的访问记录

```
`awk '($5 ~ /order/latest){print $0}' access_log.2017-10-30.log`
```

2.4 某个接口每秒并发数前100条

```
`cat access_log.2017-11-22.log| grep "/order/add" | awk '{print substr($3,0,8)}' |sort|uniq -c |sort -rn|head -100`
```

### 2.4 每秒并发数最高的接口前100

```
`cat access_log.2017-11-22.log | awk ' {print $5 substr($3,0,8)}' |sort|uniq -c |sort -rn|head -100`
```