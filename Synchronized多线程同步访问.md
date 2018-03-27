### Synchronized 多线程同步访问

- synchronized 实例锁
- static synchronized 全局锁，无论实例化了多少个对象，线程都是共享这个锁的。

#### 实例分析

```java
public class AppleModel {

    private static boolean mIsApple;

    public synchronized void setValue(boolean isApple) {
        System.out.println("setValue ---> " + Thread.currentThread().getName());
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        mIsApple = isApple;
        System.out.println("setValue <--- " + isApple + " , " + Thread.currentThread().getName());
    }

    public synchronized boolean getValue() {
        System.out.println("getValue ---> " + mIsApple + " , " + Thread.currentThread().getName());
        return mIsApple;
    }

    public static synchronized void setStaticValue(boolean isApple) {
        System.out.println("setStaticValue ---> " + Thread.currentThread().getName());
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        mIsApple = isApple;
        System.out.println("setStaticValue <--- " + mIsApple + " , " + Thread.currentThread().getName());
    }

    public static synchronized boolean getStaticValue() {
        System.out.println("getStaticValue ---> " + mIsApple + " , " + Thread.currentThread().getName());
        return mIsApple;
    }

    private boolean mObjectIsApple;

    private final Object sLock = new Object();

    public void setObjectValue(boolean isApple) {
        synchronized (sLock) {
            System.out.println("setObjectValue ---> " + Thread.currentThread().getName());
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            mObjectIsApple = isApple;
            System.out.println("setObjectValue <--- " + mObjectIsApple + " , " + Thread.currentThread().getName());
        }
    }

    public boolean getObjectValue() {
        System.out.println("getObjectValue ---> " + Thread.currentThread().getName());
        return mObjectIsApple;
    }
}

private void testSync() {
	final AppleModel a = new AppleModel();
	final AppleModel b = new AppleModel();
}
```

1. 线程 1 调用 a.setValue()，线程 2 调用 b.getValue() ？

   ```java
   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		a.setValue(true);
   	}
   }, "1-thread").start();

   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		b.getValue();
   	}
   }, "2-thread").start();

   //        03-27 10:49:36.573 15714-16458/I/System.out: setValue ---> 1-thread
   //        03-27 10:49:36.574 15714-16459/I/System.out: getValue ---> true , 2-thread
   //        03-27 10:49:38.574 15714-16458/xI/System.out: setValue <--- true , 1-thread
   ```

   结论：可以同时调用，synchronized 是对象锁，这里是两个对象，锁的是不同的对象，互不影响。

2. 线程 1 调用 a.setStaticValue()，线程 2 调用 b.getStaticValue() ？

   ```java
   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		a.setStaticValue(true);
   	}
   }, "1-thread").start();

   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		b.getStaticValue();
   	}
   }, "2-thread").start();

   //        03-27 11:50:56.598 25337-27895/I/System.out: setStaticValue ---> 1-thread
   //        03-27 11:50:58.601 25337-27895/I/System.out: setStaticValue <--- true , 1-thread
   //        03-27 11:50:58.601 25337-27896/I/System.out: getStaticValue ---> true , 2-thread
   ```

   结论：不能同时调用，static synchronized 是 Class 全局锁，锁的是 AppleModel.class。

3. 线程 1 调用 AppleModel.setStaticValue()，线程 2 调用 AppleModel.getStaticValue() ？

   ```java
   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		AppleModel.setStaticValue(true);
   	}
   }, "1-thread").start();

   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		AppleModel.getStaticValue();
   	}
   }, "2-thread").start();

   //        03-27 10:41:10.887 4324-5269/I/System.out: setStaticValue ---> 1-thread
   //        03-27 10:41:12.888 4324-5269/I/System.out: setStaticValue <--- true , 1-thread
   //        03-27 10:41:12.888 4324-5270/I/System.out: getStaticValue ---> true , 2-thread
   ```

   结论：不能同时调用，static synchronized 是 Class 全局锁，锁的是 AppleModel.class。

4. 线程 1 调用 AppleModel.setStaticValue()，线程 2 调用 b.getValue() ？

   ```java
   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		AppleModel.setStaticValue(true);
   	}
   }, "1-thread").start();

   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		b.getValue();
   	}
   }, "2-thread").start();

   //        03-27 10:43:19.907 7408-8015/I/System.out: setStaticValue ---> 1-thread
   //        03-27 10:43:19.907 7408-8017/I/System.out: getValue ---> false , 2-thread
   //        03-27 10:43:21.907 7408-8015/I/System.out: setStaticValue <--- true , 1-thread
   ```

   结论：可以同时调用，`setStaticValue` 锁的是 AppleModel.class，`getValue` 锁的是对象 b。

5. 线程 1 调用 a.setValue()，线程 2 调用 a.setObjectValue() ？

   ```java
   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		a.setValue(true);
   	}
   }, "1-thread").start();

   new Thread(new Runnable() {
   	@Override
   	public void run() {
   		a.setObjectValue(false);
   	}
   }, "2-thread").start();

   //        03-27 10:56:07.819 23965-24526/I/System.out: setValue ---> 1-thread
   //        03-27 10:56:07.820 23965-24527/I/System.out: setObjectValue ---> 2-thread
   //        03-27 10:56:09.819 23965-24526/I/System.out: setValue <--- true , 1-thread
   //        03-27 10:56:09.820 23965-24527/I/System.out: setObjectValue <--- false , 2-thread
   ```

   结论：可以同时调用，`setValue` 锁的是 a 对象，`setObjectValue` 锁的是 sLock 对象。

#### 总结

- 对于`实例同步方法`，锁的是当前的`实例对象`。
- 对于`静态同步方法`，锁的是当前对象的` Class对象`，Class 对象是全局唯一、共享的。
- 对于`同步代码块`，锁的是`括号里`配置的`对象`。