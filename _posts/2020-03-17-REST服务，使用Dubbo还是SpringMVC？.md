---
layout:     post
title:      REST服务，使用Dubbo还是SpringMVC？
date:       2020-03-17
author:     ChayCao
header-img: img/post-bg-2015.jpg 
catalog: true
tags:  随笔
---





SpringMVC、Dubbo 都支持 REST 服务，那当我们要开发一个 REST 服务接口时，该如何选择？
本文将包括以下两方面内容：

1. REST服务的写法
2. REST服务的应用场景

## 1. REST服务的写法

首先我们看下 SpringMVC 怎么实现一个 REST 服务：

```java
@RestController
@RequestMapping("/greetings")
public class SpringRestController {

    @RequestMapping(method = RequestMethod.GET,
                    value = "/{name}", 
                    produces = MediaType.TEXT_PLAIN_VALUE)
    public ResponseEntity<?> greeting(@PathVariable String name) {
        String greeting = "Hello " + name;
        return new ResponseEntity<>(greeting, HttpStatus.OK);
    }
}
```

使用过 SpringMVC 的同学都很熟悉这些注解吧，`@RestController`、`@RequestMapping`、`@PathVariable`。

在了解 Dubbo 是如何实现 REST 服务之前，先简单聊下 Dubbo 中关于 REST 的那部分历史。Dubbo 于 2011 年开源，而 2014 年 开始发展停滞。早些时候的 Dubbo 是不支持 REST 的，而如果要实现一个 REST 服务，也是有办法的，可以结合 SpringMVC，在 Controller 中调 Dubbo 的服务。但是 REST 这么火，想直接在 Dubbo 上支持 REST 怎么办？2014年，当当 Fork 了一个 Dubbo 版本开始维护，命名为 DubboX，并增加了 REST 风格的远程调用。后来随着 Dubbo 和 DubboX 的合并，Dubbo 将 DubboX 中对 REST 的支持合并了进来。

聊完这段框架史，下面我们来一起看 Dubbo 是如何实现 REST 服务的：

```java
@Path("/greetings")
public class DubboService {

    @GET
    @Path("/{name}")
    @Produces(MediaType.TEXT_PLAIN)
    public Response greeting(@PathParam("name") String name) {
        String greeting = "Hello " + name;
        return Response.ok(greeting).build();
    }
}
```

从中可以看出 Dubbo 使用的注解不同于 SpringMVC，`@Path`、`@GET`和`@PathParam`。这套注解是 JAX-RS 规范所定义的。关于 JAX-RS，这是标准的 Java REST API，具体的开源实现有 Oracle 的 Jersey、RedHat 的 RestEasy、Apache 的 CXF 和 Wink 以及 Restlet 等等。而 Dubbo 则是使用了 RestEasy 来支持 REST 服务。

既然 Java REST 都已经有了 JAX-RS 标准了，为啥 SpringMVC 不使用这套标准？我猜想主要原因应该是 SpringMVC 本身已有一套自己的注解了，如 `@RequestMapping`在没有 REST 之前就在使用了，所以在支持 REST 时，仍考虑使用原有的注解风格。

## 2. REST服务的应用场景

第 1 节已对 SpringMVC、Dubbo 写法的不同之处进行了介绍。那如何根据应用场景进行选择。我们首先看下 Dubbo 的一些 REST 应用场景：

1. **企业内部的异构系统之间的（跨语言）调用**。Dubbo 的系统做服务提供端，其他语言的系统（也包括某些不基于 Dubbo 的 Java 系统）做服务消费端，两者通过HTTP和文本消息进行通信。即使相比 Thrift、ProtoBuf 等二进制跨语言调用方案，REST也有自己独特的优势。
2. **对外 Open API（开放平台）的开发**。既可以用 Dubbo 来开发专门的 Open API 应用，也可以将原内部使用的 Dubbo Service 直接“透明”发布为对外的Open REST API。
3. **简化手机（平板）APP 或者 PC 桌面客户端开发**。类似于第 2 点，既可以用Dubbo 来开发专门针对无线或者桌面的服务器端，也可以将原内部使用的Dubbo Service 直接”透明“的暴露给手机APP或桌面程序。
4. **简化浏览器 AJAX 应用的开发**。类似于第 2 点，既可以用 Dubbo 来开发专门的 AJAX 服务器端，也可以将原内部使用的 Dubbo Service 直接”透明“的暴露给浏览器中 JavaScript。当然，很多 AJAX 应用更适合与 Web 框架协同工作，所以直接访问 Dubbo Service 在很多 Web 项目中未必是一种非常优雅的架构。
5. **为企业内部的 Dubbo系统之间提供一种基于文本的、易读的远程调用方式**
，即服务提供端和消费端都是基于 Dubbo 的系统。
6. **一定程度简化 Dubbo 系统对其它异构系统的调用**。可以用类似 Dubbo 的简便方式“透明”的调用非 Dubbo 系统提供的 REST 服务（不管服务提供端是在企业内部还是外部）。就是第 1 点的升级版。

![Dubbo RESTful Remoting应用蓝图](https://user-gold-cdn.xitu.io/2020/3/17/170e8769a42a610f?w=720&h=540&f=jpeg&s=206610)

其中的 1、2、3 点被认为是 Dubbo 的 REST 服务最有价值的三种应用场景，提供 REST 服务来提供给非 Dubbo 的（异构）消费端。

而 **SpringMVC 则更适合于面向 Web 应用的 REST 服务**，如第 3 点中的 AJAX 调用。这也正符合 MVC 的概念，REST 服务为 View 层的一种实现。

使用 JAX-RS 的 **Dubbo 则更适合纯粹的服务化应用**，将 Service 这类 Bean 发布成 REST 服务。

笔者认为如果是一个面向 Web 的单体应用，那应该使用 SpringMVC，完全不用考虑 Dubbo。而如果是一个微服务应用，使用了 Dubbo 作为 RPC 框架，而这时候又需要面向 Web，那应该直接使用 Dubbo 将服务以 REST 方式进行发布，没必要为了 REST 再引入 SpringMVC ，过于复杂，而且实现的效果都是一样的。



## 3.参考

1. [在 Dubbo 中开发 REST 风格的远程调用（RESTful Remoting）](https://dubbo.apache.org/zh-cn/docs/user/rest.html)
2. [Difference between JAX-RS and Spring Rest](https://stackoverflow.com/questions/42944777/difference-between-jax-rs-and-spring-rest)



![公众号二维码](https://i2.tiimg.com/717558/a410997819862ca9.png)