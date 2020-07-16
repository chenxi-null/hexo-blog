title: Seata 分布式事务框架 源码解析——优雅停机
author: chenxi
tags:
  - seata
  - distributed
categories: []
date: 2020-06-05 17:00:00
---

### 前言

Seata 是什么？
> Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。

Github 地址: https://github.com/seata/seata

文章概述：

- 基于源码分析了 Seata 优雅停机的实现方式

- 同时分析了 Spring 的优雅停机, 介绍了 Spring Context 的生命周期管理，顺带提及了它对于提高软件可测试性的意义

- 对比了 Dubbo 的优雅停机策略

Seata 的优雅停机模块其实并没有涉及太多 Seata 本身的领域概念，本文更多的还是以 Seata 为引子，介绍了优雅停机在框架设计中的优秀实践，不熟悉 Seata 的同学也可以放心食用～


---

下面是正文部分：

Seata 优雅停机的逻辑主要是放在 `io.seata.core.rpc.netty.ShutdownHook` 这个类。

我们先看 `io.seata.server.Server` 里是如何使用 `ShutdownHook` 的:

在 `Server` 的 main 方法里调用 `ShutdownHook` 的 addDisposable 方法，注册 disposable 对象
```java
// register ShutdownHook
ShutdownHook.getInstance().addDisposable(coordinator);
ShutdownHook.getInstance().addDisposable(rpcServer);
```

### JVM 层面的优雅停机
`ShutdownHook` 在类加载时会注册了一个 JVM shutdown hook
```java
static {
    Runtime.getRuntime().addShutdownHook(SHUTDOWN_HOOK);
}
```
`ShutdownHook` 继承自 `Thread`，JVM 在正常关闭时执行 Thread#run 方法里面的逻辑，依次执行之前添加的所有 disposable 的对象的 destory 方法：
```java
@Override
public void run() {
    destroyAll();
}

public void destroyAll() {

    if (LOGGER.isDebugEnabled()) {
        LOGGER.debug("destroyAll starting");
    }
    if (!destroyed.compareAndSet(false, true) && CollectionUtils.isEmpty(disposables)) {
        return;
    }
    for (Disposable disposable : disposables) {
        disposable.destroy();
    }
}
```

### Spring 层面的优雅停机
以上是 Seata `Server` 使用 `ShutdownHook` 的方式，`ShutdownHook` 还会被 Seata Client 端使用，具体是被 `DefaultSagaTransactionalTemplate` `GlobalTransactionScanner` 这两个类使用，我们看下在 `GlobalTransactionScanner` 里的具体使用：
```java
// GlobalTransactionScanner 实现了 Spring 的 DisposableBean 接口
// Spring Context 在关闭时会回调这些 DisposableBean 的 destory 方法
@Override
public void destroy() {
    ShutdownHook.getInstance().destroyAll();
}

private void initClient() {
    //...
    //init TM
    //...
    //init RM
    //...
    registerSpringShutdownHook();
}

// 在初始化时，向 Seata ShutdownHook 注册 Disposable 对象，
// 这些的 Disposable 是 Seata 自定义的，注意和 Spring 的 DisposableBean 区分
private void registerSpringShutdownHook() {
    if (applicationContext instanceof ConfigurableApplicationContext) {
        ((ConfigurableApplicationContext) applicationContext).registerShutdownHook(); // [1]
        ShutdownHook.removeRuntimeShutdownHook(); // [2]
    }
    ShutdownHook.getInstance().addDisposable(TmRpcClient.getInstance(applicationId, txServiceGroup));
    ShutdownHook.getInstance().addDisposable(RmRpcClient.getInstance(applicationId, txServiceGroup));
}
```

注意这里有两个有意思的地方，分别是代码里标注[1]和[2]，下面我们来逐一分析：

### `ConfigurableApplicationContext#registerShutdownHook`

这是 Spring `AbstractApplicationContext` 里的实现逻辑：
```java
/**
 * Register a shutdown hook with the JVM runtime, closing this context
 * on JVM shutdown unless it has already been closed at that time.
 * <p>Delegates to {@code doClose()} for the actual closing procedure.
 * @see Runtime#addShutdownHook
 * @see #close()
 * @see #doClose()
 */
@Override
public void registerShutdownHook() {
	if (this.shutdownHook == null) {
		// No shutdown hook registered yet.
		this.shutdownHook = new Thread() {
			@Override
			public void run() {
				synchronized (startupShutdownMonitor) {
					doClose();
				}
			}
		};
		Runtime.getRuntime().addShutdownHook(this.shutdownHook);
	}
}
```
这里的 `doClose` 方法就是 Spring Context 关闭时做的各种清理工作，包括刚才提到的回调各个 DisposableBean 的 destroy 方法。

通过这行`Runtime.getRuntime().addShutdownHook(this.shutdownHook);` 我们可以知道 ApplicationContext 的 registerShutdownHook 方法最终还是把清理工作的时机交给 JVM 的 shutdown hook 

所以调用了 ApplicationContext#registerShutdownHook 之后就把原先 Seata `ShutdownHook` 注册的 JVM shutdown hook 移除掉，避免重复调用 `ShutdownHook#destoryAll`。

`ShutdownHook#removeRuntimeShutdownHook`:
```java
/**
 * for spring context
 */
public static void removeRuntimeShutdownHook() {
    Runtime.getRuntime().removeShutdownHook(SHUTDOWN_HOOK);
}
```

### `ConfigurableApplicationContext#close`

说到 `AbstractApplicationContext` 的 `registerShutdownHook` 方法，不得不提一下它的另一个 `close` 方法，
close 方法和 registerShutdownHook 一样，也是对 `doClose` 的委托，只是他们的调用 `doClose` 的时机不同:  
- `registerShutdownHook` 是把调用时机交给 JVM 的 shutdown hook，在 JVM 关闭时执行
- `close` 是直接执行，同时把 ApplicationContext 之前注册的 JVM shutdown hook 移除掉

```java
/**
 * Close this application context, destroying all beans in its bean factory.
 * <p>Delegates to {@code doClose()} for the actual closing procedure.
 * Also removes a JVM shutdown hook, if registered, as it's not needed anymore.
 * @see #doClose()
 * @see #registerShutdownHook()
 */
@Override
public void close() {
	synchronized (this.startupShutdownMonitor) {
		doClose();
		// If we registered a JVM shutdown hook, we don't need it anymore now:
		// We've already explicitly closed the context.
		if (this.shutdownHook != null) {
			try {
				Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
			}
			catch (IllegalStateException ex) {
				// ignore - VM is already shutting down
			}
		}
	}
}
```

这里在加上 Spring 的清理逻辑之后似乎有点绕，但是不要慌，我们来整理一下:

Seata 的 `ShutdownHook` 通过 `addDisposable` 注册 disposable 对象，这里 server 端和 client 端的用法都是一样的。

`ShutdownHook` 在类加载时会注册一个 JVM shutdown hook，在 JVM 正常关闭时，最终会调用 `ShutdownHook#distroyAll` 对象销毁所有先前注册过的 disposable 对象。

而 Client 端因为是放在 Spring 容器当中的，所以把 `ShutdownHook#distroyAll` 的执行时机完全交给 Spring Context 来控制。

Spring Context 的销毁时机又分为两种:

- 调用 `registerShutdownHook`: 注册一个 JVM shutdown hook，在 JVM 关闭时执行

- 调用 `close`，直接执行清理逻辑


### 可测试性架构

`ConfigurableApplicationContext#close` 方法可以让用户灵活地控制 Spring Context 的生命周期，比如一个 JVM 应用内存在多个 ApplicationContext 或者我们需要销毁某个 ApplicationContext 的时候，可以直接调用 close 方法，而不需要等到 JVM 关闭时才能执行 context 的清理工作。

还有一个典型应用是我们在写基于 Spring Context 的集成测试的时候，在每个测试用例执行完毕后需要调用 `ConfigurableApplicationContext#close` 方法销毁 spring context：
```java
public class DemoSpringJUnitTest {

    ConfigurableApplicationContext ctx;

    @Test
    public void test() {
        // 开始测试时创建 context
        ctx = new AnnotationConfigApplicationContext(TestConfig.class);
        // do some testing...
    }

    @After
    public void tearDown() {
        // 测试结束后销毁 context
        if (ctx != null) {
            ctx.close();
        }
    }
}
```

这里暂时先不考虑使用 Spring-Test `@ContextConfiguration` 或者 `@SpringBootTest` 的情况，使用它们的话 Spring 会默认复用 context，详细用法见 [Spring-Test 官方文档](https://docs.spring.io/spring/docs/current/spring-framework-reference/testing.html#testcontext-framework)

如果没有正确关闭 spring context 的话，在批量地执行多个测试用例时就有可能出现在一个 JVM 当中同时存在多个相同的 spring context，可能存在多个类型相同的后台任务，如异步任务调度、MQ Consumer，多个线程竞争同一资源，导致测试用例出现概率性失败。

所以，把框架里的清理工作交给 Spring 来管理是非常明智的，这样用户可以在测试代码中灵活地控制其生命周期，代码的可测试性更强。

**可测试性**也是架构设计中一个非常重要的指标，易于测试的架构才是优雅的、有生命力的架构。 例如 Netty 中的 `EmbeddedChannel`，可以让使用者在不进行实际网络调用的情况下测试 Channel 的功能，这部分就不展开了，感兴趣的同学可以去看下 Netty 的单元测试。

### Dubbo 的优雅停机
Seata 基于 Spring 的优雅停机策略和 Dubbo 的优雅停机策略基本是一样的（注意下面代码中标注[1]和[2]的地方），个人猜测 Seata 大概率是借鉴了 Dubbo，复用了社区或者阿里内部的最佳实践。
```java
public class SpringExtensionFactory implements ExtensionFactory {
    private static final Logger logger = LoggerFactory.getLogger(SpringExtensionFactory.class);

    private static final Set<ApplicationContext> CONTEXTS = new ConcurrentHashSet<ApplicationContext>();
    private static final ApplicationListener SHUTDOWN_HOOK_LISTENER = new ShutdownHookListener();

    public static void addApplicationContext(ApplicationContext context) {
        CONTEXTS.add(context);
        if (context instanceof ConfigurableApplicationContext) {
            // [1] 注册 Spring Context 的 ShutdownHook
            ((ConfigurableApplicationContext) context).registerShutdownHook();
            // [2] 取消 Dubbo AbstractConfig 注册的 ShutdownHook 事件
            DubboShutdownHook.getDubboShutdownHook().unregister();
        }
        BeanFactoryUtils.addApplicationListener(context, SHUTDOWN_HOOK_LISTENER);
    }

    private static class ShutdownHookListener implements ApplicationListener {
        @Override
        public void onApplicationEvent(ApplicationEvent event) {
            if (event instanceof ContextClosedEvent) {
                DubboShutdownHook shutdownHook = DubboShutdownHook.getDubboShutdownHook();
                shutdownHook.doDestroy();
            }
        }
    }
}
```
> 这段代码解决了两个问题：
> 1. Dubbo ShutdownHook 和 Spring Context ShutdownHook 重复执行的问题；
> 
> 2. 在使用 SpringBoot 内嵌 Tomcat 容器时，Spring Context 的 `registerShutdownHook` 方法是会被自动调用的，但使用纯粹的 Spring 框架或者外部 Tomcat 容器时则不会。  
> 这里 Dubbo 显示地调用 `registerShutdownHook`，解决了 Spring ShutdownHook 可能未被注册的问题。

想要了解具体细节的同学，可以阅读靖峰大佬的这篇文章 [一文聊透 Dubbo 优雅停机](https://www.cnkirito.moe/dubbo-gracefully-shutdown/)，非常详细地介绍了 Dubbo 优雅停机的技术实现和演进历程。