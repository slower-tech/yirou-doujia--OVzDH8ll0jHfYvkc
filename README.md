
## 前言


网上冲浪时刷到线程池的文章，想想看自己好像还没在实际场景中设置过线程名称，小小研究一下。


## 研究过程


### 默认命名


创建的线程都会有自己的名字，如果不设置，程序会给线程默认的名字，如`Thread-0`



```
Thread t = new Thread(() -> {
    System.out.println(Thread.currentThread().getName());
});
t.start();
// Thread-0

```

设置线程名称，应当有个理由。现在一个项目中有订单相关线程池、有付款相关线程池，给线程命名，可以容易区分线程种类。如`Thread-order-0`、`Thread-fund-0`


### Thread内部实现原理


首先我们打开Thread类，属性name就是线程的名称。



```
public
class Thread implements Runnable {
    /* Make sure registerNatives is the first thing  does. */
    private static native void registerNatives();
    static {
        registerNatives();
    }
    
    // 线程名字
    private volatile String name;
    private int            priority;
    private Thread         threadQ;
    private long           eetop;
    
    // ...省略
    
}

```

测试：


![](https://img2024.cnblogs.com/blog/1704037/202412/1704037-20241214233900321-1915442933.png)


## 抽象实现


首先实现线程工厂构造器，主要构造线程工厂对象



```
@Getter
public class ThreadFactoryBuilder {
    
    private String nameFormat;

    public ThreadFactoryBuilder setNameFormat(String nameFormat) {
        this.nameFormat = nameFormat;
        return this;
    }

    public ThreadFactory build(){
        return buildThreadFactory(this);
    }

    public static ThreadFactory buildThreadFactory(ThreadFactoryBuilder builder) {
        final AtomicInteger counter = new AtomicInteger(0);
        return r -> {
            Thread t = new Thread(r);
            // 这里设置了线程池名称，使用counter区分不同线程
            t.setName(String.format(builder.getNameFormat(), counter.getAndIncrement()));
            return t;
        };
    }
}

```

查看源码ExecutorService源码，发现预留了线程工厂的入参



```
// ExecutorService newFixedThreadPool 预留了ThreadFactory
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue(),
                                  threadFactory);
}

```

测试代码正确性：



```
ThreadFactory threadFactory = new ThreadFactoryBuilder().setNameFormat("测试命名线程-%d").build();
ExecutorService executor = Executors.newFixedThreadPool(5, threadFactory);

for (int i = 0; i < 10; i++) {
    executor.execute(() -> {
        try {
            sleep(1000);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println(Thread.currentThread().getName());
    });
}

// 测试命名线程-3
// 测试命名线程-2
// 测试命名线程-1
// 测试命名线程-0
// 测试命名线程-4
// 测试命名线程-0
// 测试命名线程-2
// 测试命名线程-1
// 测试命名线程-3
// 测试命名线程-4

```

 本博客参考[蓝猫机场](https://fenfang.org)。转载请注明出处！
