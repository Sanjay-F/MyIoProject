title:  源码探索系列15---那个AsyncTask里面的FutureTask
date: 2015-12-29 01:46
tags: [android,源码,AsyncTask,Future,futureTask,Callable]
categories: android
------------------------------------------


很久前在写[源码探索系类2--AsyncTask](http://blog.csdn.net/sanjay_f/article/details/50237729)时候有提及到这个类，现在在这里把`FutureTask`，`Future`和`Callback`，这三个火枪手的关系温习下 

# 起航

就让我们进入主题，开始说说这个**FutureTask**吧。

在安装开发过程中，系统限制我们对于耗时的任务是不能执行在主线程的，必须单独开一个线程去做。
所以我们在开发过程的一种写法是下面这样

## 1. **用Runnable**

		new Thread(new Runnable() {
	            @Override
	            public void run() {
	            
	             //do something
	             ...
	             
	             myHandler.sendMessage(msg);   
	            }
		}).run();
但有时候我们需要这个线程的运算结果，可我们没办法直接获取，因此安卓配套了一个Handler给我们用，利用他发送消息会我们的主线程，执行一些更新任务等。

<!--more-->

## 2.用Callable
除了使用Runnable，我们还可以使用Callable，示例如下

	ExecutorService executor = Executors.newCachedThreadPool();
	Future future= executor.submit(new MyCallableTask());
	System.out.println(" result=" + future3.get());
	
	class MyCallableTask implements Callable<String> {
        @Override
        public String call() throws Exception {                          
            return "call-result";
        }
    }
我们的Callable和Runnable的一点区别是可以有返回值了，而且能抛出异常。
不过他只能用ExecutorService来执行，不能用在新线程中`new Thread(Runnable r)`
但获得他的返回值，好像不是很方便。而且这个get操作是堵塞线程的，例如改成下面这样

	System.out.println("before" + System.currentTimeMillis());
    System.out.println(" result=" + future3.get());
    System.out.println("after" + System.currentTimeMillis());

打印的结果是：

	before1451383552900
	result=call-result
	after1451383555900
时间刚好差了3秒钟。

很显然，有时候我们需要异步的，希望等运行结束了通知下我，我去获取结果，然后做点什么，改怎办呢？如果不用`AsyncTask`,`Thread+Handler`的方式？看下这个`FutureTask`能不能帮我们点什么

## FutureTask
 我们先看下示例代码：
	
    ExecutorService executor = Executors.newCachedThreadPool();
    MyFutureTask futureTask = new MyFutureTask(new MyCallableTask());
    executor.submit(futureTask);
	
	class MyCallableTask implements Callable<String> {
         @Override
         public String call() throws Exception {
             Thread.sleep(3000);
             return "call-result";
         }
     }


    class MyFutureTask extends FutureTask<String> {

        public MyFutureTask(Callable<String> callable) {
            super(callable);
        }

        @Override
        protected void done() {
            try {
                System.out.println("执行完毕，结果是:"+get());
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
 
我们的`done()`函数会被调用，当这个任务结束返回结果的话。
小小问题来了，为何这个FutureTask可以被Executor执行？我们看下他的构造

	 public class FutureTask<V> implements RunnableFuture<V>
	 public interface RunnableFuture<V> extends Runnable, Future<V>
他实现了`Runnable`和`Future`，是个混血儿！混血儿！混血儿！Amazing
就像是AsyncTask封装好了，帮我们解脱这些繁琐的事情一样，有用!	

既然这样，我们去看下他的内部的`run`方法吧
 
	 public void run() {
      
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
			...
        }
    }
他会去调用我们的callable.call()函数，然后把结果扔给`Set（）`函数，如果一切正常的话。
我们看下set里面的

	protected void set(V v) {
        if (U.compareAndSwapInt(this, STATE, NEW, COMPLETING)) {
            outcome = v;
            U.putOrderedInt(this, STATE, NORMAL); // final state
            finishCompletion();
        }
    }
最后调用的是`finishComPletion（）`

	private void finishCompletion() {
        // assert state > COMPLETING;    
        ....
        
        done();
        callable = null;        // to reduce footprint
    }
    
我们看到他调用了`done`函数了，而且最后把**callbale**给 **清清清清清清清**了，因为我们拿到结果了。
这个`done()` 函数里面什么也没有，主要是通知我们计算完毕，我们可以在这个时候去调用`get（）`函数去获取结果了。


---

说到这，想提下，知道为何`AsyncTask`不能够执行两次吗？和这个FutureTask有关系吗？


# 后记

这次没有后记的内容。