---
title: 简单访问量统计(Spring Boot+Redis+Echarts)
date: 2021-06-13 16:57:16
tags:
  - Java
  - Spring Boot
  - Redis
---

### 环境

* IDE - IDEA
* Spring Boot + Redis - 快速构建部署简单的Web服务
* Thymeleaf - 模板引擎
* VMware Workstation（Red Hat + Redis） - 部署Redis服务
* ECharts - 前端使用柱状图展示访问量

---

### 准备

#### 1. 虚拟机 - Red Hat - Redis服务

[Redis官网](https://redis.io/download) 最新的安装信息：

```shell
$ wget https://download.redis.io/releases/redis-6.2.4.tar.gz
$ tar xzf redis-6.2.4.tar.gz
$ cd redis-6.2.4
$ make
```

下载、解压、编译完成后便可启动Redis服务：

```shell
$ src/redis-server
```

Redis服务启动完毕，此时的服务是前端服务，该Terminal挂着（<kbd>ctrl</kbd>+<kbd>c</kbd>可shutdown服务）。

另外打开新的Terminal启动Redis客户端进行操作：

```shell
$ src/redis-cli
```

另外把`protected-mode`设为no，方便本机（非虚拟机本身）的客户端连接该Redis服务

此处在`redis-cli`客户端命令行进行临时配置（关掉服务就失效）

```shell
127.0.0.1:6379> config set protected-mode no
127.0.0.1:6379> config get protected-mode
1) "protected-mode"
2) "no"
```

也可以对配置文件`redis-6.2.4/redis.config`进行配置，eg:

```shell
# 绑定环回接口地址，只允许本机客户端连接，注释掉则放开允许所有网络接口进行连接
bind 127.0.0.1
# yes-后台守护进程运行，no-前台运行
daemonize no
# yes-如果没有对Redis设置密码或者尝试连接Redis的主机ip没有被bind，会被拒绝连接
protected-mode yes
```

#### 2. Spring Boot项目配置

1. web、redis、thymeleaf的依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

2. `application.yml`配置，这里只配置了虚拟机的ip，如果redis配置了账号密码等可以参照`RedisProperties.java`里头的配置选项

```yaml
spring:
  redis:
    host: 192.168.30.128
```

3. 项目结构

![project-struture](project-struture.png)

---

### 代码

> 1. 创建拦截器对uri进行拦截，使用Redis进行统计，并放行
> 2. 获取Redis统计的数据传到前端，使用ECharts柱状图显示统计数据

#### TrafficIntercepter.java

```java
@Component
public class TrafficIntercepter implements HandlerInterceptor {

    private static final String KEY_URI = "uri";

    @Autowired
    StringRedisTemplate redisTemplate;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        String uri = request.getRequestURI();
        HashOperations<String, Object, Object> operations = redisTemplate.opsForHash();
        operations.increment(KEY_URI, uri, 1);
        return true;
    }
}
```

1. `TrafficIntercepter`实现`HandlerInterceptor`接口
2. 重写`preHandle`方法，注入Redis模板`StringRedisTemplate`，获取Hash操作实例
3. `operations.increment(KEY_URI, uri, 1)`为Hash表 key(KEY_URI="uri") 中的指定field(请求的uri)的整数值加上1
4. 将拦截的所有请求放行

#### TrafficConfiguration.java

```java
@Configuration
public class TrafficConfiguration implements WebMvcConfigurer {

    @Autowired
    TrafficIntercepter trafficIntercepter;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(trafficIntercepter)
                .addPathPatterns("/**")
                .excludePathPatterns("/stat");
    }
}
```

1. `TrafficConfiguration`实现`WebMvcConfigurer`接口
2. 重写`addInterceptors`方法，添加拦截器（注入的`trafficIntercepter`实例）
3. 路径匹配/**，排除统计页面请求路径/stat

#### TrafficController.java

```java
@Controller
public class TrafficController {

    private static final String KEY_URI = "uri";

    @Autowired
    StringRedisTemplate redisTemplate;

    @RequestMapping("/stat")
    public String trafficStat(Model model) {
        HashOperations<String, Object, Object> operations = redisTemplate.opsForHash();
        Map<Object, Object> entries = operations.entries(KEY_URI);
        List<String> uris = new ArrayList<>();
        List<String> counts = new ArrayList<>();
        if (!entries.isEmpty()) {
            entries.forEach((k, v) -> {
                uris.add((String) k);
                counts.add((String) v);
            });
        }
        model.addAttribute("uris", uris);
        model.addAttribute("counts", counts);
        return "traffic-stat";
    }

    @ResponseBody
    @RequestMapping("/path1")
    public String path1() {
        return "path1";
    }

    @ResponseBody
    @RequestMapping("/path2")
    public String path2() {
        return "path2";
    }

    @ResponseBody
    @RequestMapping("/path3")
    public String path3() {
        return "path3";
    }
}
```

1. `trafficStat(Model model)`获取Redis中存储的Hash表中key为”uri“的各个请求访问路径以及其访问量，分别保存到`List`中，并通过model将数据传到前端页面`traffic-stat.html`
2. `path1()`/`path2()`/`path3()`用于访问统计

#### traffic-stat.html

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>访问量统计</title>
    <script src="https://cdn.staticfile.org/jquery/2.2.4/jquery.min.js"></script>
    <script src="https://cdn.staticfile.org/echarts/4.3.0/echarts.min.js"></script>
</head>
<body>
<div id="data" style="width: 600px;height:400px;"></div>
<script th:inline="javascript" type="text/javascript">
        var uris = [[${uris}]];
        var counts = [[${counts}]];

        var myChart = echarts.init(document.getElementById('data'));

        var option = {
            title: {
                text: '访问量统计'
            },
            tooltip: {},
            legend: {
                data:['访问量']
            },
            xAxis: {
                data: uris
            },
            yAxis: {},
            series: [{
                name: '访问量',
                type: 'bar',
                data: counts
            }]
        };

        myChart.setOption(option);
</script>
</body>
</html>
```

1. `<html lang="en" xmlns:th="http://www.thymeleaf.org">`声明使用Thymeleaf标签
2. `<head>`分别引入jquery和ECharts的js资源
3. `<body>`放置一个存放ECharts图表的Dom容器`<div>`
4. `<script th:inline="javascript" type="text/javascript">`，Thymeleaf内联声明`th:inline="javascript"`允许JavaScript代码块里访问model中的数据，更好地集成 JavaScript
5. `[[${uris}]]`、`[[${counts}]]`分别获取后台传来的uri、counts数组
6. `var myChart = echarts.init(document.getElementById('data'))`初始echarts实例，放如准备好的Dom容器中
7. `option`指定图表的配置项和数据:
   * title：配置图表标题
   * tooltip：配置提示信息
   * legend：配置图例，可以通过点击图例控制图例类型显示与否
   * xAxis：配置X轴坐标，将访问路径uris设入X轴data
   * yAxis：配置Y轴坐标
   * series：配置列表内容，将访问量counts设入data
8. `myChart.setOption(option)`指定的配置项和数据显示图表

---

### Apache Bench压测

```shell
C:\Users\ucandoooh>ab -n 10000 -c 10 localhost:8080/path1
C:\Users\ucandoooh>ab -n 8000 -c 10 localhost:8080/path2
C:\Users\ucandoooh>ab -n 3000 -c 10 localhost:8080/path3
```

1. 对`/path1`路径进行并发数为10的10000次请求访问
2. 对`/path2`路径进行并发数为10的8000次请求访问
3. 对`/path3`路径进行并发数为10的3000次请求访问

---

### 页面展示

![traffic-stat](traffic-stat.png)

访问`/stat`查看访问量统计柱状图

访问量对应Apache Bench压测请求数

由此可以看出Redis是高并发的

>1. Redis是单线程程序，保证单个操作的原子性，是线程安全的
>2. Redis是基于内存的，内存的读写速度非常快，并且能将内存中的数据持久化到磁盘
>3. Redis支持master-slave主从数据备份......

---

### 注意事项

> Thymeleaf内联 - JavaScript代码块里访问model中的数据 - escaped和unescaped方式

```javascript
// escaped方式
var uris = [[${uris}]];
// 页面显示结果如下，内容会用引号括起，并且经过转义
// 数据正确，柱状图正常显示
var uris = ["\/path1","\/path2","\/path3"];
```

```javascript
// unescaped方式
var uris = [(${uris})];
// 页面显示结果如下，内容未用引号括起，并未经过转义
// 页面会渲染失败，Console控制台报错
var uris = [/path1, /path2, /path3];
```

```javascript
// 但如果unescaped方式处理的是list的json字符串
var urisjson = [(${urisjson})];
// 页面显示结果如下，json字符串最外层引号被去掉，字符串里面特殊符号未转义
// 数据正确，柱状图正常显示
var urisjson = ["/path1","/path2","/path3"];
```

这两种方式大家斟酌使用，不同场景不同使用方式，甚是有趣
