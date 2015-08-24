title: Spring的两种任务调度Scheduled和Async
date: 2015-08-24 07:53:40
tags: [schedule,Async]
categories: Spring
---

 Spring提供了两种后台任务的方法,分别是:

1. 调度任务，@Schedule
2. 异步任务，@Async


当然，使用这两个是有条件的，需要在spring应用的上下文中声明 
 `<task:annotation-driven/>`当然，如果我们是基于java配置的，需要在配置哪里加多`EnableScheduling`和`@EnableAsync` 就像下面这样

 
```
@EnableScheduling
@EnableAsync 
public class WebAppConfig {

            ....
｝
```

除此之外，还是有第三方库可以调用的，例如Quartz.


----
<!--more-->

## @Schedule

先看下`@Schedule`怎么调用再说


```
 
 public final static long ONE_DAY = 24 * 60 * 60 * 1000;
 public final static long ONE_HOUR =  60 * 60 * 1000;

 	@Scheduled(fixedRate = ONE_DAY)
    public void scheduledTask() {
        System.out.println(" 我是一个每隔一天就会执行一次的调度任务");
    }

    @Scheduled(fixedDelay = ONE_HOURS)
    public void scheduleTask2() {
        System.out.println(" 我是一个执行完后，隔一小时就会执行的任务");
    }
    
    @Scheduled(initialDelay=1000, fixedRate=5000)
 	public void doSomething() {
      // something that should execute periodically
    }

    @Scheduled(cron = "0 0/1 * * * ? ")
    public void ScheduledTask3() {
        System.out.println(" 我是一个每隔一分钟就就会执行的任务");
    }
    
```


**需要注意的**

1. 关于最后一个，在指定时间执行的任务，里面使用的是`Cron表达式`，同时我们看到了两个不一样的面孔`fixedDelay`& `fixedRate`，前者`fixedDelay`表示在指定间隔运行程序，例如这个程序在今晚九点运行程序，跑完这个方法后的一个小时，就会再执行一次，而后者`fixedDelay`者是指，这个函数每隔一段时间就会被调用（我们这里设置的是一天），不管再次调度的时候，这个方法是在运行还是结束了。而前者就要求是函数运行结束后开始计时的，这就是两者区别。

2. 这个还有一个initialDelay的参数，是第一次调用前需要等待的时间，这里表示被调用后的，推迟一秒再执行，这适合一些特殊的情况。 

3. 我们在`serviceImpl`类写这些调度任务时候，也需要在这些我们定义的`serviceInterface`的借口中写多这个接口，要不然会爆 ` but not found in any interface(s) for bean JDK proxy.Either pull the method up to an interface or`  



## @Async
有时候我们会调用一些特殊的任务，任务会比较耗时，重要的是，我们不管他返回的后果。这时候我们就需要用这类的异步任务啦，调用后就让他去跑，不堵塞主线程，我们继续干别的。代码像下面这样:


```
public void AsyncTask(){

    @Async
    public void doSomeHeavyBackgroundTask(int sleepTime) {
        try {
            Thread.sleep(sleepTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    
    
    @Async
    public Future<String> doSomeHeavyBackgroundTask() {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return null;
    }
    
    public void printLog() {
        System.out.println("  i print a log  ,time=" + System.currentTimeMillis());
    }

 }
     
```

我们写个简单的测试类来测试下


```
@RunWith(SpringJUnit4ClassRunner.class)
@WebAppConfiguration
@ContextConfiguration(classes = AsycnTaskConfig.class) //要声明@EnableASync
public class AsyncTaskTest {


    @Autowired
    AsyncTask asyncTask;

    @Test
    public void AsyncTaskTest() throws InterruptedException {
        if (asyncTask != null) {
            asyncTask.doSomeHeavyBackgroundTask(4000);
            asyncTask.printLog();
            Thread.sleep(5000);            
        }
    }
}

```

这感觉比我们手动开多一个线程方便多了，不想异步的话直接把`@Async`去掉就可以了，另外如果你想要返回个结果的，这需要加多个Future<>,关于这个Future，完全可以写多几篇文章介绍，顺便把FutureTask介绍了。
**需要注意的：**

 1. 相对于`@scheduled`，这个可以有参数和返回个结果，因为这个是我们调用的，而调度的任务是spring调用的。
 2. 异步方法不能内部调用，只能像上面那样，外部调用，否则就会变成阻塞主线程的同步任务啦！这个坑我居然跳下去了！例如下面这样的。

 
 ```

	public void AsyncTask(){
 
 
       public void fakeAsyncTaskTest(){
          doSomeHeavyBackgroundTask(4000);
          printLog();
          //你会发现，当你像这样内部调用的时候，居然是同步执行的，不是异步的！！       
       }
       

	    @Async
	    public void doSomeHeavyBackgroundTask(int sleepTime) {
	        try {
	            Thread.sleep(sleepTime);
	        } catch (InterruptedException e) {
	            e.printStackTrace();
	        }
	    } 
	    
	    public void printLog() {
	        System.out.println("  i print a log ");
	    }

 }
 
 ```
 
 3. 另外一点就是不要重复的扫描，这也会导致异步无效，具体的可以看这个stackoveflow的[spring-async-not-working](http://stackoverflow.com/questions/6610563/spring-async-not-working) Issue。
 4. 关于异常处理，难免在这个异步执行过程中有异常发生，对于这个问题，spring提供的解决方案如下,实现     AsyncUncaughtExceptionHandler接口。

 
```
 
public class MyAsyncUncaughtExceptionHandler implements 
     AsyncUncaughtExceptionHandler {

	    @Override
	    public void handleUncaughtException(Throwable ex,
	    		 Method method, Object... params) {
	        
	        // handle exception
	    }
	}

```


写好我们的异常处理后，我们需要配置一下，告诉spring，这个异常处理就是我们在运行异步任务时候，抛出错误时的异常终结者


```

	@Configuration
	@EnableAsync
	public class AsyncConfig implements AsyncConfigurer {

	    @Bean
	    public AsyncTask asyncBean() {
	        return new AsyncTask();
	    }

	    @Override
	    public Executor getAsyncExecutor() {
	        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
	        executor.setCorePoolSize(7);
	        executor.setMaxPoolSize(42);
	        executor.setQueueCapacity(11);
	        executor.setThreadNamePrefix("MyExecutor-");
	        executor.initialize();
	        return executor;
	    }

	    @Override
	    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
	        return new MyAsyncUncaughtExceptionHandler();
	    }
	}

```


---


## Quartz登场
处理这两个外，还有一个和spring整合的第三方库叫[Quartz](http://www.quartz-scheduler.org%20Quartz)
看了下官网的使用简介，也是挺逗的，现在都习惯用maven，gradle之类来关系这些依赖了，他还叫人下载，也是不知为何，详情点击－>
http://quartz-scheduler.org/documentation/quartz-2.2.x/quick-start
估计有可能是因为没再维护了的原因吧，看了下，最新版2.2居然是Sep, 2013更新的...
居然是停更的，不过Quartz作为一个企业级应用的任务调度框架，还是一个可以的候选项目的。
这里不铺开讲，有兴趣就去官网看下吧。整体用起来感觉是没有spring自己的后台任务方便，不过也可以接受，只需要简单的配置就可以使用了。
 
---
tips:

1. “Notice that the methods to be scheduled must have void returns and must not expect any arguments. If the method needs to interact with other objects from the Application Context, then those would typically have been provided through dependency injection.”
2. “Make sure that you are not initializing multiple instances of the same @Scheduled annotation class at runtime, unless you do want to schedule callbacks to each such instance. Related to this, make sure that you do not use @Configurable on bean classes which are annotated with @Scheduled and registered as regular Spring beans with the container: You would get double initialization otherwise, once through the container and once through the @Configurable aspect, with the consequence of each @Scheduled method being invoked twice.”

Excerpt From: Rod Johnson. “Spring Framework Reference Documentation.” iBooks. 