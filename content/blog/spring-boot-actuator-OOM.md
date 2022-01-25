---
title: 【review-项目】spring-cloud-gateway，spring-boot-actuator 导致 OOM
date: 2019-02-20 15:23:41
updated: 2019-02-20 15:23:38
categories: 
- review
tags: 
- spring-boot-actuator
- OOM
- JVM
description: 解决因为 spring-cloud-gateway 对 其他应用服务的 uriPattern 不可知，导致 actuator 的 metrics 中 tag 数量爆炸，进而导致 OOM。 
---
> 解决因为 spring-cloud-gateway 对 其他应用服务的 uriPattern 不可知，导致 actuator 的 metrics 中 tag 数量爆炸，进而导致 OOM。

<!--more-->

# 背景
- 2018-12-31 凌晨3点多，最终成功上线了新版 CRM。
- 31 号 10点多，就收到反馈：CRM 上不去了。
检查后，发现进程死掉了。日志中有 OOM 的报错。
马上修改了 supervisor 中的启动配置，`将内存从 256M 增大到 512M`。
- 中午，系统又挂了。再`配置内存监控`。服务器上跑着 `jstat -gc`，本地开着 jconsole 和 visualVM，发现运行一段时间后，内存又满了，而且`越往后，GC 清除出的空间越少`，最后变成了密集的毛刺。
继续查看 JVM 的各个数据报告。
同事建议：把 CMS 改成 G1。
完全没有道理，但是一时半会儿查不出问题。先改了，重新启动。
改成 G1 后，系统没再挂过，此时已经是下午4点多。
- 然后是元旦假期，没有人在钉钉上找过来。元旦一直在家，中午和晚上都连上 visualVM 看一下，内存没有占满。

- 元旦后，第一天上班，10点多，系统又挂了。继续开着监控 排查问题，同时还盯着，如果挂了，及时重启服务。

装了 jprofiler 分析 dump 文件，找到了内存占用的元凶,如图所示：
![](https://user-images.githubusercontent.com/5629307/50624685-8a68a600-0f5d-11e9-926d-0e5e6476ad2c.png)
去项目里查看依赖：
![](https://user-images.githubusercontent.com/5629307/50582177-62187300-0e9b-11e9-8b48-b55e37ad20d8.png)
去 micrometer，spring-boot-actuator，spring-boot-admin，spring-gateway 的项目里搜索，提了 issue。

在 JVM 内存报告，jprofiler，github issue 折腾了很久之后。
- 同事告诉我，MAT 有 Mac 版的，可以试下。这时已经是元旦后上班第三天了。我一直以为 eclipse 没有 mac 版的，都没有尝试去搜过。（eclipse 是 java 开发的，有 linux 环境下的版本应该是很正常的，但是完全没想到。）

使用 MAT 分析了 dump 后，很快看到了熟悉的数据：
`带参数的具体请求路径`保存在 UnmodifiableMap 中。
![](https://user-images.githubusercontent.com/5629307/50626579-f51fde80-0f69-11e9-9f54-b3781c25ef16.png)
## 本地调试
之后，本地启动服务，在 ImmutableTag 处打了断点，然后再在调用链上向上找，最后找到了`org.springframework.boot.actuate.metrics.web.reactive.server.WebFluxTags#uri`。
本地测试观察到：
- 在表单页面，重复点击搜索，没有触发 meterMap 的 断点。
- 项目里大部分的查询都是通过参数的形式，也就是不同的 url，而不是不同的 body。

tag 的结构如图，`(exception,method,status,uri)` 这个元组 作为 tag 的 key。
因为 gateway 中没有其他服务的 pattern，所以 uri 直接保存了完整的uri。
在项目中的表现是：`一个列表页，所有不同参数组合的查询请求都被视为不同的请求，`，被保存了起来。
这也解释了，为什么一晚上内存没爆，上班一个多小时之后，内存爆了。
![](https://user-images.githubusercontent.com/5629307/50630346-7764ce80-0f7b-11e9-871a-519af3a6a8b6.png)
spring-boot-actuator 2.0.0：
![](https://user-images.githubusercontent.com/5629307/50630540-38834880-0f7c-11e9-8aa5-31b4a4fe8a80.png)
升级到 2.0.6 后，可以解决 3xx 和 404 的问题，
spring-boot-actuator 2.0.6：
```java
public static Tag uri(ServerWebExchange exchange){
	PathPattern pathPattern = exchange
			.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
	if (pathPattern != null) {
		return Tag.of("uri", pathPattern.getPatternString());
	}
	HttpStatus status = exchange.getResponse().getStatusCode();
	if (status != null) {
        // 3XX 和 404 的请求，保存为 指定的两个 Tag 对象
		if (status.is3xxRedirection()) {
			return URI_REDIRECTION;
		}
		if (status == HttpStatus.NOT_FOUND) {
			return URI_NOT_FOUND;
		}
	}
	String path = exchange.getRequest().getPath().value();
	if (path.isEmpty()) {
		return URI_ROOT;
	}
	return Tag.of("uri", path);
}
```
2.0.6 并没有解决问题，我去提了 issue [actuator WebFluxTags#uri() cannot access other server's handlerMapping leads to tag explosion](https://github.com/spring-projects/spring-boot/issues/15608)

有人针对我的 issue，提了 PR：https://github.com/spring-projects/spring-boot/pull/15609/files
方案是把所有的无法匹配 pattern 的请求全部归到 root 和 unknown 两个 tag 中。
在我的项目中，结果就是：所有 gateway 自己处理的请求会被记录下来，所有转发的请求都被归为 unknown。
我之前想着如何在 gateway 中获取其他服务的 uri-pattern-list。没想到 直接不保存的 方案 被官方合入了。错过了成为 Spring 贡献者的机会。。。
继续追问了有集中监控的业务需要，有没有可能获取其他服务 pattern的可能性，开发者回复：
> I guess a gateway should be more concerned about producing local metrics, and each application should publish its own to a shared repository.

- actuator（spring-boot） 不知道 gateway（spring-cloud） 的存在。
- gateway 只关心自己的请求统计。
- 各个应用应该`主动提交`自己的 metric 信息到统一存储中。

从问题域的角度，也有道理。这些 url 的 mapping 本就不是 gateway 的问题域，gateway 需要保持对其他服务的最小了解，只需要知道转发的前缀 和 服务定位。
其他服务的 url 不应该在gateway 被监控。

## 内存状态
只留下了 G1 的截图。
特点是：
- 每次 GC后，释放出的内存越来越少。
- 使用 G1 的情况下，老年代越来越大。

明显的内存泄漏问题。
（**todo** `内存泄漏`，总觉得这个翻译有问题。）
![](https://user-images.githubusercontent.com/5629307/50622373-829e0700-0f47-11e9-92df-40b87a5fd9ba.png)

![](https://user-images.githubusercontent.com/5629307/53032875-b8bd2900-34aa-11e9-8042-9bba91e9b1bf.png)


## 原因
`http://localhost:8772/actuator/metrics/http.server.requests` ，spring-boot-actuator 提供的请求 时间记录。

`gateway 服务`没有 uri.pattern 的信息，导致所有进来的 uri 都被视为不同的请求，保存了下来，导致了`tag explosion`，如图：
![](https://user-images.githubusercontent.com/5629307/50676374-00890d80-102f-11e9-8c80-bb946f31485f.png)
这是 `业务应用服务`的统计：
可以看到，其中是有占位符的。uri 被归到了 pattern 上。
![](https://user-images.githubusercontent.com/5629307/50676336-c881ca80-102e-11e9-9950-e476fde5eac4.png)
## 解决
- 3xx 和 404 请求：
升级 spring-boot-actuator 到 2.0.6 以上
- 无法匹配 pattern 的请求：
  - 15609 那个修改已经合入主干分支。 `v2.1.3.RELEASE` `v2.1.2.RELEASE` 可用。
  - 如果使用之前的版本，关闭 http.server.requests 记录
```yaml
management:
  metrics:
    web:
      server:
        auto-time-requests: false
```

## MAT
作为 dump 文件分析工具：
和 jprofiler 相比，MAT 的数据表达很清晰，全面，交互很自然。
jprofiler 唯一的优点就是好看了。
# 总结
1. MAT 真好用。
2. MAT 真好用。
3. MAT 真好用。
4. 熟悉工具很重要。

[解决问题过程中的其他截图，记录，相关 issue](https://github.com/youzipi/notes/issues/3)