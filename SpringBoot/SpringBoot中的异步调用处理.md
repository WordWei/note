### SpringBoot中开启异步支持

> 要开启异步支持， 首先得在Spring Boot入口类上加上 `@EnableAsync` 

> 编写异步方法， 只需要在方法上加上`@Async`注解便是异步方法了 

### 自定义异步线程池

添加自定义异步线程池类

```java
@Configuration
public class AsyncPoolConfig {

    @Bean
    public ThreadPoolTaskExecutor asyncThreadPoolTaskExecutor(){
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(200);
        executor.setQueueCapacity(25);
        executor.setKeepAliveSeconds(200);
        executor.setThreadNamePrefix("asyncThread");
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);

        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());

        executor.initialize();
        return executor;
    }
}
```

-  `corePoolSize`：线程池核心线程的数量，默认值为1（这就是默认情况下的异步线程池配置使得线程不能被重用的原因）。  
-  `maxPoolSize`：线程池维护的线程的最大数量，只有当核心线程都被用完并且缓冲队列满后，才会开始申超过请核心线程数的线程，默认值为`Integer.MAX_VALUE`。 
-  `queueCapacity`：缓冲队列。 
-  `keepAliveSeconds`：超出核心线程数外的线程在空闲时候的最大存活时间，默认为60秒。 
-  `threadNamePrefix`：线程名前缀。 
-  `waitForTasksToCompleteOnShutdown`：是否等待所有线程执行完毕才关闭线程池，默认值为false。 
-  `awaitTerminationSeconds`：`waitForTasksToCompleteOnShutdown`的等待的时长，默认值为0，即不等待。 
-  `rejectedExecutionHandler`：当没有线程可以被使用时的处理策略（拒绝任务），默认策略为`abortPolicy` 
-  `callerRunsPolicy`：用于被拒绝任务的处理程序，它直接在 execute 方法的调用线程中运行被拒绝的任务；如果执行程序已关闭，则会丢弃该任务。 
-  `abortPolicy`：直接抛出`java.util.concurrent.RejectedExecutionException`异常。 
-  `discardOldestPolicy`：当线程池中的数量等于最大线程数时、抛弃线程池中最后一个要执行的任务，并执行新传入的任务。 
-  `discardPolicy`：当线程池中的数量等于最大线程数时，不做任何动作。 

要使用该线程池，只需要在`@Async`注解上指定线程池Bean名称即可： 

```java
@Async("asyncThreadPoolTaskExecutor")
public void asyncMethod() {
    ......
}
```

### 异步回调处理

 如果异步方法具有返回值的话，需要使用`Future`来接收回调值。 

```java
@Async("asyncThreadPoolTaskExecutor")
public Future<String> asyncMethod() {
    sleep();
    logger.info("异步方法内部线程名称：{}", Thread.currentThread().getName());
    return new AsyncResult<>("hello async");
}
```

 `AsyncResult`为Spring实现的`Future`实现类.

```java
@GetMapping("async")
public String testAsync() throws Exception {
    long start = System.currentTimeMillis();
    logger.info("异步方法开始");

    Future<String> stringFuture = testService.asyncMethod();
    String result = stringFuture.get();
    logger.info("异步方法返回值：{}", result);
    
    logger.info("异步方法结束");

    long end = System.currentTimeMillis();
    logger.info("总耗时：{} ms", end - start);
    return stringFuture.get();
}
```

 `Future`接口的`get`方法用于获取异步调用的返回值。 

 `Future`的`get`方法为阻塞方法，只有当异步方法返回内容了，程序才会继续往下执行。 

 `get`还有一个`get(long timeout, TimeUnit unit)`重载方法，我们可以通过这个重载方法设置超时时间，即异步方法在设定时间内没有返回值的话，直接抛出`java.util.concurrent.TimeoutException`异常。 

 比如设置超时时间为60秒： 

```java
String result = stringFuture.get(60, TimeUnit.SECONDS);
```





> 本文转自  https://mrbird.cc/Spring-Boot-Async.html