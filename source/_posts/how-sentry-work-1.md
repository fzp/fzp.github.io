---
layout: blog
title: Sentry 后端监控SDK分析
date: 2021-03-16 16:49:32
tags: 
    - 后端监控
    - Sentry
---

本章从[源码](https://github.com/getsentry/sentry-java)出发探讨后端监控SDK的原理，主要集中在Java SDK和Spring Boot SDK.

并回答以下问题：

1. 如何收集错误？
2. 如何收集性能数据？
3. 如何追踪路径？
4. 什么时候发送数据？
5. 怎么发送数据？
6. 如何扩展Sentry SDK？

<!--more-->

## 代码架构

```
|- packages
    |- sentry-spring
    |- sentry-spring-boot-starter
    |- sentry-logback
    |- sentry
```

Sentry SDK的代码分为多个模块，放在Github仓库的根目录下。

- sentry-spring-boot-starter在sentry-spring的基础上提供了跟mvc框架相关的插装项。
- sentry-spring提供对spring框架的插装（Instumentation）。
- sentry-logback提供了对日志系统logback的插装。
- sentry提供接口定义，基本的类和方法。

## Java SDK

有了之前分析前端SDK的经验，很多术语已经比较熟悉了。比如Hub, Scope, Event, Breadcrumbs, Transaction, Span等。

- Event：收集的异常。
- Hub: 将异常发送到Sentry的地方。
- Scope: Event相关的上下文和Breadcrumbs。
- Breadcrumbs: 异常发生前的事情。
- Tansaction: 用于性能追踪的操作记录。
- Span: transaction树的节点。

Java SDK实现了上述的数据结构。在任何Java项目里，都可以导入[Sentry Java SDK](https://docs.sentry.io/platforms/java/)。手动上传Event和Transaction。

## Spring Boot SDK

有了框架，就可以利用框架提供的接口自动上传异常和性能数据。

在初始化阶段，只需在`application.properties` 或 `application.yml`里添加配置信息即可。这些配置信息会自动注入到相应的对象中。

它的实现原理是利用了[Spring Boot starter自动装配](https://docs.spring.io/spring-boot/docs/2.1.18.RELEASE/reference/html/boot-features-developing-auto-configuration.html)的特性。

> SpringBoot 定义了一套接口规范，这套规范规定：SpringBoot 在启动时会扫描外部引用 jar 包中的META-INF/spring.factories文件，将文件中配置的类型信息加载到 Spring 容器（此处涉及到 JVM 类加载机制与 Spring 的容器知识），并执行类中定义的各种操作。对于外部 jar 来说，只需要按照 SpringBoot 定义的标准，就能将自己的功能装置进 SpringBoot。

查看`META-INF/spring.factories`.两个自动装配入口。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
io.sentry.spring.boot.SentryAutoConfiguration,\
io.sentry.spring.boot.SentryLogbackAppenderAutoConfiguration
```

### SentryLogbackAppenderAutoConfiguration

查看`SentryLogbackAppenderAutoConfiguration`，创建了SentryLogbackInitializer对象。

```Java
@Bean
public @NotNull SentryLogbackInitializer sentryLogbackInitializer(
    final @NotNull SentryProperties sentryProperties) {
return new SentryLogbackInitializer(sentryProperties);
}
```

该对象实现了`GenericApplicationListener`接口。实际上是监听Spring的`ContextRefreshedEvent`事件。

```Java
@Override
public boolean supportsEventType(final @NotNull ResolvableType eventType) {
return eventType.getRawClass() != null
    && ContextRefreshedEvent.class.isAssignableFrom(eventType.getRawClass());
}
```

该事件在Spring容器里所有对象都实例化后触发。捕捉到事件后，`sentryAppender.start()`对Sentry进行了初始化，然后将sentryAppender加入到logger。Appender用于将日志事件发送到目的地。sentryAppender将日志发送到Sentry。

```Java
@Override
public void onApplicationEvent(final @NotNull ApplicationEvent event) {
    final Logger rootLogger = (Logger) LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME);
    ...
    sentryAppender.start();
    rootLogger.addAppender(sentryAppender);
}
```

当logger接收到事件时，会触发appender里的append方法。根据日志的级别，会发送日志或者将日志放入到BreadCrumbs。

```Java
@Override
  protected void append(@NotNull ILoggingEvent eventObject) {
    if (eventObject.getLevel().isGreaterOrEqual(minimumEventLevel)) {
      Sentry.captureEvent(createEvent(eventObject));
    }
    if (eventObject.getLevel().isGreaterOrEqual(minimumBreadcrumbLevel)) {
      Sentry.addBreadcrumb(createBreadcrumb(eventObject));
    }
  }
```

### SentryAutoConfiguration

回到另一个配置入口SentryAutoConfiguration。创建了Hub对象，Spring MVC相关对象， performance相关对象，传输对象工厂等。Hub对象由于捕捉和发送异常的，之前Hub在前端SDK中已经分析过，基本逻辑是一致的。Spring MVC和performance相关对象是收集数据用的。传输对象工厂用于创建发送到Sentry的客户端。

#### Spring MVC相关对象

Spring MVC相关对象都在主要有SentryRequestResolver, SentrySpringRequestListener, SentryExceptionResolver, SentryTracingFilter.

SentryRequestResolver主要是获取Http request相关信息并记录下来。

```Java
public @NotNull Request resolveSentryRequest(final @NotNull HttpServletRequest httpRequest) {
    final Request sentryRequest = new Request();
    sentryRequest.setMethod(httpRequest.getMethod());
    sentryRequest.setQueryString(httpRequest.getQueryString());
    sentryRequest.setUrl(httpRequest.getRequestURL().toString());
    sentryRequest.setHeaders(resolveHeadersMap(httpRequest));

    if (hub.getOptions().isSendDefaultPii()) {
        sentryRequest.setCookies(toString(httpRequest.getHeaders("Cookie")));
    }
    return sentryRequest;
}
```

SentrySpringRequestListener实现了ServletRequestListener用于监听http请求。它主要用于初始化scope,添加breadscrumb并将前面的SentryRequestResolver加到scope的事件处理器中。事件处理器会在捕捉到错误的执行。

```Java
public void requestInitialized(ServletRequestEvent sre) {
    hub.pushScope();

    final ServletRequest servletRequest = sre.getServletRequest();
    if (servletRequest instanceof HttpServletRequest) {
        final HttpServletRequest request = (HttpServletRequest) sre.getServletRequest();
        hub.addBreadcrumb(Breadcrumb.http(request.getRequestURI(), request.getMethod()));

        hub.configureScope(
            scope -> {
            scope.addEventProcessor(
                new SentryRequestHttpServletRequestProcessor(request, requestResolver));
            });
    }
}
```

SentryExceptionResolver实现了HandlerExceptionResolver从而捕捉Spring全局异常,最后通过`hub.captureEvent(event)`发送给Sentry.

```Java
@Override
public @Nullable ModelAndView resolveException(
    final @NotNull HttpServletRequest request,
    final @NotNull HttpServletResponse response,
    final @Nullable Object handler,
    final @NotNull Exception ex) {

    final Mechanism mechanism = new Mechanism();
    mechanism.setHandled(false);
    final Throwable throwable =
        new ExceptionMechanismException(mechanism, ex, Thread.currentThread());
    final SentryEvent event = new SentryEvent(throwable);
    event.setLevel(SentryLevel.FATAL);
    event.setTransaction(transactionNameProvider.provideTransactionName(request));
    hub.captureEvent(event);

    // null = run other HandlerExceptionResolvers to actually handle the exception
    return null;
}
```

SentryTracingFilter通过FilterRegistrationBean注册了一个Servelt的过滤器。并且拥有最高优先级。

```Java
@Bean
@ConditionalOnProperty(name = "sentry.enable-tracing", havingValue = "true")
@ConditionalOnMissingBean(name = "sentryTracingFilter")
public FilterRegistrationBean<SentryTracingFilter> sentryTracingFilter(
    final @NotNull IHub hub, final @NotNull SentryRequestResolver sentryRequestResolver) {
    FilterRegistrationBean<SentryTracingFilter> filter =
        new FilterRegistrationBean<>(new SentryTracingFilter(hub, sentryRequestResolver));
    filter.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return filter;
}
```

从@ConditionalOnProperty注解可知，SentryTracingFilter只有在`sentry.enable-tracing=true`的时候才创建。它的作用是开启一个transaction，并在执行完所有filter后（包括业务代码）关闭transaction。`finish()`里面调用了`hub.captureTransaction(transaction);` 将transation发送给Sentry.

另外，注意到代码里从头文件中取出了sentryTraceHeader。这是前端传过来的trace Id, 如果前端也配有Sentry,那么前后端的性能性能监控就能联合起来。

```Java
@Override
protected void doFilterInternal(){
    ...
    final String sentryTraceHeader = httpRequest.getHeader(SentryTraceHeader.SENTRY_TRACE_HEADER);
    ...
    final ITransaction transaction = startTransaction(httpRequest, sentryTraceHeader);
    try {
        filterChain.doFilter(httpRequest, httpResponse);
    } finally {
        // after all filters run, templated path pattern is available in request attribute
        final String transactionName = transactionNameProvider.provideTransactionName(httpRequest);
        // if transaction name is not resolved, the request has not been processed by a controller
        // and
        // we should not report it to Sentry
        if (transactionName != null) {
            transaction.setName(transactionName);
            transaction.setOperation(TRANSACTION_OP);
            transaction.setStatus(SpanStatus.fromHttpStatusCode(httpResponse.getStatus()));
            transaction.finish();
        }
    }
    ...
}
```

### performance相关对象

此处引入了Spring 配置类`SentryTransactionPointcutConfiguration`和`SentrySpanPointcutConfiguration`.它们分别定义了以@SentryTransaction和@SentrySpan注解为标记的切点。同时还导入了`SentryAdviceConfiguration`配置类，定义了处理切点函数行为的类sentryTransactionAdvice和SentrySpanAdvice。

以`SentrySpanPointcutConfiguration`为例。它定义了两类切点，一类是Class上带有@SentrySpan的方法。另一类是直接带有@SentrySpan的方法。

```Java
@Bean
public @NotNull Pointcut sentrySpanPointcut() {
    return new ComposablePointcut(new AnnotationClassFilter(SentrySpan.class, true))
        .union(new AnnotationMatchingPointcut(null, SentrySpan.class));
}
```

处理@SentrySpan的行为定义在SentrySpanAdvice。主要将该函数调用行为以span方式添加到transaction里。

```Java
final String operation = resolveSpanOperation(targetClass, mostSpecificMethod, sentrySpan);
final ISpan span = activeSpan.startChild(operation);
```

### 传输对象工厂

传输对象工厂会创建ApacheHttpClientTransport，该对象利用CloseableHttpAsyncClient发送请求。异步请求不会阻塞原来的业务逻辑。

```Java
@Override
public ITransport create(
    final @NotNull SentryOptions options, final @NotNull RequestDetails requestDetails) {
    Objects.requireNonNull(options, "options is required");
    Objects.requireNonNull(requestDetails, "requestDetails is required");

    final PoolingAsyncClientConnectionManager connectionManager =
        PoolingAsyncClientConnectionManagerBuilder.create()
            .setPoolConcurrencyPolicy(PoolConcurrencyPolicy.LAX)
            .setConnectionTimeToLive(connectionTimeToLive)
            .setMaxConnTotal(options.getMaxQueueSize())
            .setMaxConnPerRoute(options.getMaxQueueSize())
            .build();

    final CloseableHttpAsyncClient httpclient =
        HttpAsyncClients.custom()
            .setKeepAliveStrategy(DefaultConnectionKeepAliveStrategy.INSTANCE)
            .setConnectionManager(connectionManager)
            .setConnectionReuseStrategy(DefaultConnectionReuseStrategy.INSTANCE)
            .build();

    final RateLimiter rateLimiter = new RateLimiter(options.getLogger());

    return new ApacheHttpClientTransport(options, requestDetails, httpclient, rateLimiter);
}
```

## 回答问题

现在基本摸清了Sentry 后端SDK的代码。回到我们前面提到的问题。

### 如何收集错误？

对于Java, 手动收集。

对于Spring Boot, 一是利用它提供的HandlerExceptionResolver捕捉全局异常。 二是利用日志系统的收集。

### 如何收集性能数据？

对于Java, 手动收集。

对于Spring Boot, 一是利用了Servlet的filter, 开启和完成transaction。

### 如何追踪路径？

在日志系统里添加breadcrumbs.

利用Spring的AOP机制，跟据@SentryTransaction和@SentrySpan获取更多调用细节。

### 什么时候发送数据？

对于异常，一旦捕捉到就会发送。

对于性能数据，请求结束就会发送。

### 怎么发送数据？

Spring boot框架下默认使用CloseableHttpAsyncClient。

### 如何扩展Sentry SDK？

在 Java SDK中也有Intregration的概念，但大多数用在了安卓端。在后端SDK中相对较少。如果在Spring boot中， 利用Spring Boot Starter扩展就够了。