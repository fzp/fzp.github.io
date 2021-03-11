---
title: Sentry 前端监控SDK分析
date: 2021-03-08 12:12:39
tags: 
    - 前端监控
    - Sentry
    - web-vitals
---

Sentry是开源前端监控软件。

1. SDK丰富，可集成多种语言和框架。
2. 功能强大，包含异常检测，性能监控，路径追踪。
3. 后端能处理大规模数据，可用于生产环境部署。

本章从[源码](https://github.com/getsentry/sentry-javascript)出发探讨前端监控SDK的原理，主要回答以下问题：

1. 如何收集错误？
2. 如何收集性能数据？
3. 如何追踪路径？
4. 什么时候发送数据？
5. 怎么发送数据？
6. 如何扩展Sentry SDK？

<!--more-->

## 从入口开始

在分析具体问题之前，我们回到代码的入口，查看具体的代码做了什么事情。

以VUE为例，Sentry的集成代码如下：

```javascript
import Vue from "vue";
import * as Sentry from "@sentry/vue";

Sentry.init({
  Vue: Vue,
  dsn: "https://examplePublicKey@o0.ingest.sentry.io/0",
});
```

在init函数里面。核心是两行代码：

```javascript
browserInit(finalOptions);
...
vueHelper.setup();
```

`browserInit`来自`@sentry/browser`包。这个包主要对代码进行插装(Instrumentation)，记录每一次操作。

`vueHelper.setup()`设置错误处理器(`_attachErrorHandler`)并开始追踪VUE的生命周期(`_startTracing`)：

```javascript
public setup(): void {
    this._attachErrorHandler();

    if ('tracesSampleRate' in this._options || 'tracesSampler' in this._options) {
        this._startTracing();
    }
}
```

`_attachErrorHandler`利用了VUE的错误处理器收集错误。捕获的错误和事件放在hub并发送到Sentry:

```javascript
this._options.Vue.config.errorHandler = (error: Error, vm?: ViewModel, info?: string): void => {
     ...
    // Capture exception in the next event loop, to make sure that all breadcrumbs are recorded in time.
    setTimeout(() => {
        getCurrentHub().withScope(scope => {
            scope.setContext('vue', metadata);
            getCurrentHub().captureException(error);
        });
    });
    ...
}
```

hub和scope的[官方文档](https://docs.sentry.io/platforms/javascript/enriching-events/scopes/)。

从代码看，hub是提供了一个栈用于管理scope, scope保存了一些有用的上下文信息和追踪信息，最后与捕捉到的事件同时上传Sentry:

```javascript
export class Hub implements HubInterface {
  /** Is a {@link Layer}[] containing the client and scope */
  private readonly _stack: Layer[] = [{}];
  ...
}

```

`_startTracing`默认记录VUE组件的`activate`, `mount`和`update`的行为。以`mount`行为为例，它会捕捉`beforeMount`事件之后到`mounted`事件之前经历的时间。具体参考[VUE的生命周期](https://cn.vuejs.org/v2/guide/instance.html)。

上面是关于VUE框架的错误追踪，现在回到`browserInit`。

`browserInit`会创建`client`并绑定到`hub`。`client`包含初始的配置和这些配置对应的行为，包括发送到Sentry的操作等。

```javascript
const hub = getCurrentHub();
const client = new clientClass(options);
hub.bindClient(client);
```

`bindClient`调用了setupIntegrations方法，初始化integratinos.

```javascript
if (client && client.setupIntegrations) {
    client.setupIntegrations();
}
```

## Integrations

[Integrations](https://docs.sentry.io/platforms/javascript/configuration/integrations/)用于扩展SDK的功能。官方文档在[这里](https://docs.sentry.io/platforms/javascript/configuration/integrations/)。

`setupIntegrations`会调用所有`integration`的初始化方法`setupOnce`. 比如下面的代码是breadCrumb的integration给Console API插装callback方法。

```javascript
public setupOnce(): void {
    if (this._options.console) {
      addInstrumentationHandler({
        callback: (...args) => {
          this._consoleBreadcrumb(...args);
        },
        type: 'console',
      });
    }
    ...
}
```

在Integration的初始化方法里，经常会看到来自`@sentry/utils`包的`addInstrumentationHandler`方法。该方法用于为一些native API添加处理器(handler)，也就是插装。

它的原理是将处理器注册在API下（其实就是放在一个特定数组里）。然后在相应的API和浏览器事件处理器中包装一层切面。在切面中，依次触发注册的处理器。

比如在插装console API时`addInstrumentationHandler`会调用了`fill`函数。

`fill`函数接收三个参数：对象，对象里的方法名和高阶函数。高阶函数接收对象里方法名对应的函数并返回一个新函数，最后将原函数替换成新函数。

在下面的代码里，`console[level]`函数被替换。新函数首先触发了所有注册的处理器，再重新调用原来的`console[level]`。

```javascript
['debug', 'info', 'warn', 'error', 'log', 'assert'].forEach(function(level: string): void {
    if (!(level in global.console)) {
      return;
    }

    fill(global.console, level, function(originalConsoleLevel: () => any): Function {
      return function(...args: any[]): void {
        triggerHandlers('console', { args, level });

        // this fails for some browsers. :(
        if (originalConsoleLevel) {
          Function.prototype.apply.call(originalConsoleLevel, global.console, args);
        }
      };
    });
  });
```

除了在API中调用时增加切面，也可在浏览器事件处理器中增加切面。

比如在插装浏览器error事件时，用增加切面后的方法替换原有的方法：

```javascript
let _oldOnErrorHandler: OnErrorEventHandler = null;
/** JSDoc */
function instrumentError(): void {
  _oldOnErrorHandler = global.onerror;

  global.onerror = function(msg: any, url: any, line: any, column: any, error: any): boolean {
    triggerHandlers('error', {
      column,
      error,
      line,
      msg,
      url,
    });

    if (_oldOnErrorHandler) {
      // eslint-disable-next-line prefer-rest-params
      return _oldOnErrorHandler.apply(this, arguments);
    }

    return false;
  };
}
```

默认的Integrations有InboundFilters, FunctionToString, Breadcrumbs, GlobalHandlers, LinkedErrors 和 UserAgent。它们的代码在[这里](https://github.com/getsentry/sentry-javascript/tree/master/packages/browser/src/integrations)。下面介绍下GlobalHandlers和Breadcrumbs。

### GlobalHandlers

`GlobalHandlers`会捕捉未被捕捉的异常或promise未被处理的rejection.

在`GlobalHandlers`中，调用`addInstrumentationHandler`为浏览器的error事件和unhandledrejection事件添加处理器。

error事件处理器中将事件信息通过hub的`captureEvent`发送到Sentry：

```javascript
currentHub.captureEvent(event, {
          originalException: error,
        });
```

hub中调用了client的`captureEvent`方法整理信息后发送出去。根据浏览器支持的方法和配置的不同，通过不同的方式把数据发送到DSN.

```javascript
// https://github.com/getsentry/sentry-javascript/blob/58b2ba1f0a27496942c00cf343b17bef527ccb61/packages/browser/src/backend.ts#L73

if (this._options.transport) {
      return new this._options.transport(transportOptions);
    }
    if (supportsFetch()) {
      return new FetchTransport(transportOptions);
    }
    return new XHRTransport(transportOptions);
```

顺便一提，client在发送前有处理采样率的逻辑：

```javascript
// 1.0 === 100% events are sent
// 0.0 === 0% events are sent
// Sampling for transaction happens somewhere else
if (!isTransaction && typeof sampleRate === 'number' && Math.random() > sampleRate) {
    return SyncPromise.reject(
        new SentryError(
            `Discarding event because it's not included in the random sample (sampling rate = ${sampleRate})`,
        ),
    );
}
```

unhandledrejection事件的处理方式与之类似。

### Breadcrumbs

> Sentry uses breadcrumbs to create a trail of events that happened prior to an issue.

Breadcrumbs是问题发生前的一连串事件。

在Breadcrumbs中主要插装了下列的API:

- Console API : 控制台输出
- DOM API (click/typing) : 用户交互，点击输入
- XMLHttpRequest API : 网络请求
- Fetch API : 网络请求
- History API : 路由跳转

这些行为会记录在scope的_breadcrumbs数组里。当异常发生时随之上传Sentry。

## Performance

Sentry除了能够监控异常外，还能够监控前端的性能。在前端SDK中，性能数据的收集功能是通过integration扩展的。

在集成时需要导入:

```javascript
import { Integrations as TracingIntegrations } from "@sentry/tracing";

Sentry.init({
  dsn: "https://examplePublicKey@o0.ingest.sentry.io/0",
  integrations: [new TracingIntegrations.BrowserTracing()]
});
```

在了解前端性能监控之前，先了解下Sentry性能追踪的一些概念：[Traces, Transactions, and Spans](https://docs.sentry.io/product/performance/distributed-tracing/#traces-transactions-and-spans)

![Traces, Transactions, and Spans](diagram-transaction-trace.png)

简单而言，trace就是所有操作的记录。它可以是分布式的，包括前端，后端，数据库等。trace由transaction组成，transaction是一个树状结构，因此可以记录并发的行为。tarnsation的节点就是span，代表服务执行的单元操作。

直接查看`@sentry/tracing`包的TracingIntegrations.BrowserTracing对哪些代码进行了插装。首先对路由功能进行插装。

```javascript
routingInstrumentation(
    (context: TransactionContext) => this._createRouteTransaction(context),
    startTransactionOnPageLoad,
    startTransactionOnLocationChange,
);
```

跟进routingInstrumentation, 发现调用了两次startTransaction，也就是开启了两个transaction。transaction在perfromance分两种，一种是pageload，指的是页面加载后的操作。 一种是navigation， 指的是路由切换后做的操作。

从代码可以看出，pageload transaction从页面加载的时候开始的，navigation transaction是在调用history API的pushState和replaceState的时候开始的。

```javascript
activeTransaction = startTransaction({ name: global.location.pathname, op: 'pageload' });
...
addInstrumentationHandler({
    callback: (...) => {
    ...
    activeTransaction = startTransaction({ name: global.location.pathname, op: 'navigation' });
    },
    type: 'history',
})
```

跟进`startTransaction`，它是`routingInstrumentation`传入的`_createRouteTransaction`方法。这个方法启动了一个IdleTransaction，然后为IdleTransaction注册了一个函数，这个函数在transaction结束前回调，它的主要的作用是为transaction添加了performance相关数据和调整了transaction的时间。

```javascript
const idleTransaction = startIdleTransaction(
    hub,
    finalContext,
    idleTimeout,
    true,
    { location }, // for use in the tracesSampler
);
idleTransaction.registerBeforeFinishCallback((transaction, endTimestamp) => {
    this._metrics.addPerformanceEntries(transaction);
    adjustTransactionDuration(secToMs(maxTransactionDuration), transaction, endTimestamp);
});
```

IdleTransaction是在原有transaction的功能的基础下添加了自动结束的功能。它的基本原理是心跳检查。即定时检查发生的活动（网络请求）。如果连续三次（默认间隔5秒）检查所有活动的状态都没有改变，则结束transaction并上传数据。另外，如果网页的可见性发生变化，比如切换到别的网页，都会结束transaction。

`addPerformanceEntries`从两方面获取数据，一是直接从浏览器的performance API中获取数据，二是从[web-vitals](https://github.com/GoogleChrome/web-vitals)包提供的方法里获取CLS, LCP, FID, TTFB等数据。

## 回答问题

现在基本摸清了Sentry SDK的代码。回到我们前面提到的问题。

### 如何收集错误？

对于浏览器，是在error事件处理器里插装Sentry的异常机制。

对于前端框架，则是在前端框架的错误处理方法里处理。

### 如何收集性能数据？

利用web-vitals和performance API。

### 如何追踪路径？

在系统API或者浏览器事件里插装代码。

一类是breadcrumbs,记录用户交互和网络请求结果，线性结构。

一类是transaction, 记录网络请求的时间，树状结构。

### 什么时候发送数据？

对于异常，一旦捕捉到就会发送。

对于性能数据，一旦一段时间内没有新的网络请求或网页可见性发生改变就发送数据。

### 怎么发送数据？

首先使用自定义的方法，没有的话使用fetch或者xhr。

### 如何扩展Sentry SDK？

可以开发自己的IIntregration. 如果需要在某些API或事件中埋点，可以导入`@Sentry/utils`.
