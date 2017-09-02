### Introduction
When using Spring Security to secure our applications, we must be aware of its inner workings. The foundation is [SecurityContext](https://docs.spring.io/spring-security/site/docs/current/apidocs/org/springframework/security/core/context/SecurityContext.html) which holds data produced by the authentication process and needed for proper authorization. By definition it's thread-bound - [ThreadLocal](https://docs.oracle.com/javase/7/docs/api/java/lang/ThreadLocal.html) is used as a holder, created during the security filter process of a request. [(read more)](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#technical-overview) 
The thread-bound solution is convenient, but there is one drawback - security context is not propagated to child threads by default. Luckly, Spring provides tools to deal with this problem.

### Delegating the context
As stated in the [documentation](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#concurrency), we are given [DelegatingSecurityContextRunnable](http://docs.spring.io/spring-security/site/docs/current/apidocs/org/springframework/security/concurrent/DelegatingSecurityContextRunnable.html)  and [DelegatingSecurityContextExecutor](http://docs.spring.io/spring-security/site/docs/current/apidocs/org/springframework/security/concurrent/DelegatingSecurityContextExecutor.html). 
The first class is a low level wrapper for our [Runnable](https://docs.oracle.com/javase/7/docs/api/java/lang/Runnable.html) instances, implemented using the [delegation pattern](https://en.wikipedia.org/wiki/Delegation_pattern) . It simply takes given context and sets its during the execution of the **run()** method. The usage is as simple as:
```java
SecurityContext context = SecurityContextHolder.getContext();
DelegatingSecurityContextRunnable wrappedRunnable =
	new DelegatingSecurityContextRunnable(originalRunnable, context);
```

**DelegatingSecurityContextExecutor** is a more high-level abstraction. It delegates [Executor](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executor.html)  instances instead of **Runnables**, enabling the management of pools of threads aware of Spring's security context. 

In modern Java we would most likely use it with [stream parallel API](https://docs.oracle.com/javase/tutorial/collections/streams/parallelism.html) or [CompletableFutures](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html). Both of these abstractions by default use Java 8 default [ForkJoinPool.commonPool](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html#commonPool--), which is fine, but commonly we create custom pools dedicated to specific tasks. While **ForkJoinPool** is designed to handle work-stealing divide and conquer algorithms, we can use good old [FixedThreadPool](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Executors.html#newFixedThreadPool(int)) as well. [(read more)](https://zeroturnaround.com/rebellabs/fixedthreadpool-cachedthreadpool-or-forkjoinpool-picking-correct-java-executors-for-background-tasks/)

The following example shows the creation of custom **FixedThreadPool** with **DelegatingSecurityContextExecutor** and creating new **CompletableFuture** task:
```java
SecurityContext securityContext = SecurityContextHolder.getContext();
Executor delegatedExecutor = Executors.newFixedThreadPool(10);
Executor delegatingExecutor =
	new DelegatingSecurityContextExecutor(delegatedExecutor, securityContext);
CompletableFuture.supplyAsync(() -> veryHardTask(),delegatingExecutor);
```

### @Async methods
Above example shows delegating security context with plain Java concurrent methods. When using Spring we often use [@Async](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/scheduling.html) annotation to make our methods run asynchronously. It uses very own [SimpleAsyncTaskExecutor](http://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/SimpleAsyncTaskExecutor.html) with its own thread pool. In order to pass our context we could create another wrapping delegation. However, Spring Security again [gives us a convenient way to deal with the problem](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/#securitycontextholder-securitycontext-and-authentication-objects):
```java
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);
```

This property can be configured with:

```java
@Bean
public MethodInvokingFactoryBean methodInvokingFactoryBean() {
    MethodInvokingFactoryBean methodInvokingFactoryBean = new MethodInvokingFactoryBean();
    methodInvokingFactoryBean.setTargetClass(SecurityContextHolder.class);
    methodInvokingFactoryBean.setTargetMethod("setStrategyName");
    methodInvokingFactoryBean.setArguments(new String[]{SecurityContextHolder.MODE_INHERITABLETHREADLOCAL});
    return methodInvokingFactoryBean;
}
```
It forces Spring to wrap its async executor with Security delegate [DelegatingSecurityContextTaskExecutor](http://docs.spring.io/autorepo/docs/spring-security/4.0.0.M1/apidocs/org/springframework/security/task/DelegatingSecurityContextTaskExecutor.html). Simple as that, we are safe to use @Async methods without worring about security context.

### Wrap-up
Spring Security by definition is thread-bound, but by default is not ready to be used in multithreading environment.  However, with simple steps we are able to deal fix the problem.
