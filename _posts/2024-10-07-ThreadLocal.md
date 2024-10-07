---
layout: post
title: ThreadLocal 演变之路
categories: [java]
description: ThreadLocal 演变之路
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---



# 前言

本文主要是讲ThreadLocal、InheritableThreadLocal和TransmittableThreadLocal。主要内容有：

- **ThreadLocal 使用 和 实现原理**
- **ThreadLocal 副作用**
  - **脏数据**
  - **内存泄漏的分析**
- **InheritableThreadLocal  使用 和 实现原理**
  - **InheritableThreadLocal 副作用**
  - **父子线程传递烦恼**
- **TransmittableThreadLocal 使用和实现原理**

话不多说，火速发车！

# ThreadLocal

## 使用

**ThreadLocal 适用于每个线程需要自己独立的实例且该实例需要在多个方法中被使用，即变量在线程间隔离而在方法或类间共享的场景。** 确切的来说，ThreadLocal 并不是专门为了解决多线程共享变量产生的并发问题而出来的，而是给提供了一个新的思路，曲线救国。

通过实例代码来简单演示下ThreadLocal的使用。

```java
public class ThreadLocalExample {

    private static ThreadLocal<Integer> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {

        ExecutorService service = Executors.newCachedThreadPool();

        service.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " set 1");
            threadLocal.set(1);
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 不会受到线程2的影响，因为ThreadLocal 线程本地存储
            System.out.println(Thread.currentThread().getName() + " get " + threadLocal.get());
            threadLocal.remove();
        });

        service.execute(() -> {
            System.out.println(Thread.currentThread().getName() + " set 2");
            threadLocal.set(2);
            threadLocal.remove();
        });

        ThreadPoolUtil.tryReleasePool(service);
    }
}

/*
 测试结果：
pool-1-thread-1 set 1
pool-1-thread-2 set 2
pool-1-thread-1 get 1
*/
```

可以看到，线程1不会受到线程2的影响，因为ThreadLocal 创建的是线程私有的变量。

## 原理

### 核心类

为什么线程之间的变量能够相互不影响呢？我们需要首先理解以下几个类的关系：

- Thread
- ThreadLocal
- ThreadLocalMap
- ThreadLocalMap.Entry

Thread 类中有一个 threadLocals 成员变量（实际上还有一个inheritableThreadLocals，后面讲），它的类型是ThreadLocal 的内部静态类ThreadLocalMap

```java
public class Thread implements Runnable {
  
          // ...... 省略
          
        /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;

}
```

ThreadLocalMap 是一个定制化的Hashmap，为什么是个HashMap？很好理解，每个线程可以关联多个ThreadLocal变量。

ThreadLocalMap 初始化时会创建一个大小为16的Entry 数组，Entry 对象也是用来保存 key- value 键值对（这个Key固定是ThreadLocal 类型）。值得注意的是，这个Entry 继承了 `WeakReference`

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
       /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

### ThreadLocal的set、get及remove方法的源码

#### void set(T value)

```java
public void set(T value) {
    // ① 获取当前线程
    Thread t = Thread.currentThread();
    // ② 去查找对应线程的ThreadLocalMap变量
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
                    // ③ 第一次调用就创建当前线程的对应的ThreadLocalMap
        // 并且会将值保存进去，key是当前的threadLocal，value就是传进来的值
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

#### T get()

```java
public T get() {
    // ① 获取当前线程
    Thread t = Thread.currentThread();
    // ② 去查找对应线程的ThreadLocalMap变量
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // ③ 不为null，返回当前threadLocal 对应的value值
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // ④ 当前线程的threadLocalMap为空，初始化
    return setInitialValue();
}

private T setInitialValue() {
    // ⑤ 初始化的值为null
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        // 初始化当前线程的threadLocalMap
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

#### void remove()

如果当前线程的threadLocals变量不为空，则删除当前线程中指定ThreadLocal实例对应的本地变量。

```java
public void remove() {
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         m.remove(this);
 }
```

从源码中可以看出来，**自始至终，这些本地变量不是存放在ThreadLocal实例里面，而是存放在调用线程的threadLocals变量，那个线程私有的threadLocalMap 里面**。

ThreadLocal就是一个工具壳和一个key，它通过set方法把value值放入调用线程的threadLocals里面并存放起来，当调用线程调用它的get方法时，再从当前线程的threadLocals变量里面将其拿出来使用。

ThreadLocal实现原理比较简单，但真正的难点和重点是正确认识ThreadLocal的副作用

## ThreadLocal 副作用

ThreadLocal 的主要问题是会产生**脏数据**和**内存****泄露**。

先说一个结论，这两个问题通常是在线程池的线程中使用 ThreadLocal 引发的，因为线程池有**线程复用**和**内存****常驻**两个特点。

### 脏数据

大多数业务异步实现都是通过线程池作为载体实现。由于

线程池会重用 Thread 对象 ，那么与 Thread 绑定的类的静态属性 ThreadLocal 变量也会被重用。如果在实现的线程 run() 方法体中不显式地调用 remove() 清理与线程相关的 ThreadLocal 信息，那么倘若下一个线程不调用 set() 设置初始值，就可能 get() 到复用的线程信息，包括 ThreadLocal 所关联的线程对象的 value 值。

这里提供一个demo供你理解：

```java
public class ThreadLocalDirtyDataDemo {

    private static ThreadLocal<String> threadLocal = new ThreadLocal<>();

    public static void main(String[] args) {

        ExecutorService pool = Executors.newFixedThreadPool(1);

        for (int i = 0; i < 2; i++) {
            MyThread thread = new MyThread();
            pool.execute(thread);
        }
        ThreadPoolUtil.tryReleasePool(pool);
    }

    private static class MyThread extends Thread {
        private static boolean flag = true;

        @Override
        public void run() {
            if (flag) {
                // 第一个线程set之后，并没有进行remove
                // 而第二个线程由于某种原因(这里是flag=false) 没有进行set操作
                String sessionInfo = this.getName();
                threadLocal.set(sessionInfo);
                flag = false;
            }
            System.out.println(this.getName() + " 线程 是 " + threadLocal.get());
            // 线程使用完threadLocal，要及时remove，这里是为了演示错误情况
        }
    }
}

/*
 测试结果：
Thread-0 线程 是 Thread-0
Thread-1 线程 是 Thread-0
*/
```

可以看到，你是Thread-0，我也是Thread-0，那这不是乱套了吗？！解决方式也简单，用完记得remove一下~

### 内存泄漏

回顾一下前文，ThreadLocalMap 内部存储使用Entry 数组保存 key- value 键值对（这个Key固定是ThreadLocal 类型）。值得注意的是，这个Entry 继承了 `WeakReference`。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
      /** The value associated with this ThreadLocal. */
      Object value;

      Entry(ThreadLocal<?> k, Object v) {
          super(k);
          value = v;
      }
  }
```

ThreadLocalMap 的每个 Entry 都是一个对**键**的弱引用 - `WeakReference<ThreadLocal<?>>`，这一点从`super(k)`可看出。另外，每个 Entry都包含了一个对 **值** 的强引用。

回顾一下JVM，当 JVM 进行垃圾回收时，**无论****内存****是否充足，都会回收只被弱引用关联的对象。**

通过这种设计，**即使线程正在执行中， 只要 ThreadLocal 对象引用被置成 null，Entry 的 Key 就会自动在下一次** **YGC** **时被垃圾回收（因为只剩下ThreadLocalMap 对其的弱引用，没有强引用了）**。

> 如果这里Entry 的key 值是对 ThreadLocal 对象的强引用的话，那么**即使ThreadLocal的对象引用被声明成null 时，这些 ThreadLocal 不能被回收，因为还有来自 ThreadLocalMap 的强引用，这样子就会造成****内存****泄漏**。

这类key被回收（ `key == null`）的Entry 在 ThreadLocalMap 源码中被称为 **stale entry** （翻译过来就是 **“过时的条目”**），**会在下一次执行 ThreadLocalMap 的 getEntry 和 set 方法中，将 这些 stale entry 的value 置为 null，使得原来value 指向的变量可以被垃圾回收**。

这样子来看，ThreadLocalMap 是通过这种设计，**解决了 ThreadLocal 对象可能会存在的****内存****泄漏的问题**，**并且对应的value 也会因为上述的 stale entry 机制被垃圾回收**。

读到这里看起来不会出现内存泄漏问题啊，线程销毁之后一切都归于虚无了，怎么还会说使用ThreadLocal 可能存在内存泄露问题呢？

⚠️注意，

上述机制的前提是ThreadLocal 的引用被置为null，才会触发弱引用机制，继而回收Entry 的 Value对象实例。我们来看下ThreadLocal 源码中的注释

> instances are typically private static fields in classes
>
> ThreadLocal 对象通常作为私有静态变量使用
>
> -- 如果说一个 ThreadLocal 是非静态的，属于某个线程实例类，那就失去了线程内共享的本质属性。

作为静态变量使用的话， 那么其生命周期至少不会随着线程结束而结束。也就是说，绝大多数的静态threadLocal对象都不会被置为null。这样子的话，通过 stale entry 这种机制来清除Value 对象实例这条路是走不通的。必须要手动remove() 才能保证。

此外，也是只有在线程复用状态下会出现这个问题。因为这些本地变量都是存储在线程的内部变量中的，当线程销毁时，threadLocalMap的对象引用会被置为null，value实例对象随着线程的销毁，在内存中成为了不可达对象，然后被垃圾回收。

总结一下，上述两种ThreadLocal的副作用都是在线程复用前提下因为没有手动remove()造成的。解决副作用的方法很简单，就是每次用完ThreadLocal，都要及时调用 remove() 方法去清理。

# InheritableThreadLocal

ThreadLocal已经能满足大部分场景的使用了，但在遇到线程间传递私有变量场景下，比如用一个统一的ID来追踪记录调用链路。但是ThreadLocal 是不支持继承性的，同一个ThreadLocal变量在父线程中被设置值后，在子线程中是获取不到对应的对象的。

为了解决这个问题，InheritableThreadLocal 也就应运而生。

## 使用

InheritableThreadLocal的使用方式和ThreadLocal别无二致，提供一个demo供你理解：

```java
public class InheritableThreadLocalDemo {

    private static ThreadLocal<String> threadLocal = new InheritableThreadLocal<>();

    public static void main(String[] args) {
        // 主线程
        threadLocal.set("hello world");
        // 启动子线程
        Thread thread = new Thread(() -> {
            // 子线程输出父线程的threadLocal 变量值
            System.out.println("子线程: " + threadLocal.get());
        });

        thread.start();

        System.out.println("main: " +threadLocal.get());

    }
}

/*
 测试结果：
main: hello world
子线程: hello world
*/
```

## 原理

我们要探究的是如何将value传给子线程的，接下来通过源码给你答案：

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {

    // ①
    protected T childValue(T parentValue) {
        return parentValue;
    }

        // ②
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    // ③
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
public class Thread implements Runnable {

    /*
     * InheritableThreadLocal values pertaining to this thread. This map is
     * maintained by the InheritableThreadLocal class.
     */
    ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

InheritableThreadLocal 继承了ThreadLocal，并且重写了三个方法，看来实现的门道就在这三个方法里面。

先看代码③，InheritableThreadLocal 重写了createMap方法，那么现在当第一次调用set方法时，创建的是当前线程的inheritableThreadLocals 变量的实例而不再是threadLocals。由代码②可知，当调用get方法获取当前线程内部的map变量时，获取的是inheritableThreadLocals而不再是threadLocals。

可以这么说，在InheritableThreadLocal的世界里，变量inheritableThreadLocals替代了threadLocals。

代码②③都讲了，再来看看代码①，以及如何让子线程可以访问父线程的本地变量。

这要从创建Thread的代码说起，打开Thread类的默认构造函数，代码如下。

```java
public Thread(Runnable target) {
    init(null, target, "Thread-" + nextThreadNum(), 0);
}

private void init(ThreadGroup g, Runnable target, String name,
                  long stackSize, AccessControlContext acc,
                  boolean inheritThreadLocals) {
            
    // ... 省略无关部分
    // 获取父线程 - 当前线程
    Thread parent = currentThread();
            
    // ... 省略无关部分
    // 如果父线程的inheritThreadLocals不为null 且 inheritThreadLocals=true
    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        // 设置子线程中的inheritableThreadLocals变量
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
            // ... 省略无关部分
}

static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

再来看看里面是如何执行createInheritedMap 的。

```java
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];

    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                // 这里调用了重写的代码① childValue
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

可以看到**在构造函数中**，遍历了父线程的inheritableThreadLocals，然后遍历父线程inheritableThreadLocals的Entry数组，重新封装成Entry，并且计算数组下标放入到子线程的inheritableThreadLocals中。这也就把数据从父线程传递给了子线程。

**也就是说使用InheritableThreadLocal只能在创建线程时同步父级线程中的值，后面父级线中的值修改是不会同步到****子线程****的。**这是重点，后面要考。

## 副作用

### 脏数据

ThreadLocal 实现线程内部变量共享，InheritableThreadLocal 实现了父线程与子线程的变量继承。但是还有一种场景，InheritableThreadLocal 无法得到想要的结果，**也就是在使用线程池等会池化复用线程的执行组件情况下，异步执行执行任务，需要传递上下文的情况**。

因为线程池中的线程是复用的，并没有重新初始化线程，InheritableThreadLocal之所以起作用是因为在Thread类中最终会调用init()方法去把InheritableThreadLocal的map复制到子线程中。由于线程池复用了已有线程，所以没有调用init()方法这个过程，也就不能将父线程中的InheritableThreadLocal值传给子线程。

```java
public class ThreadDemo implements Runnable {
    private static InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
    private static ExecutorService executorService = Executors.newFixedThreadPool(1);

    public static void main(String[] args) throws Exception{
        System.out.println("----主线程启动");
        inheritableThreadLocal.set("主线程第一次赋值");
        System.out.println("----主线程设置后获取值：" + inheritableThreadLocal.get());
        executorService.submit(new ThreadDemo());
        System.out.println("主线程休眠2秒");
        Thread.sleep(2000);
        inheritableThreadLocal.set("主线程第二次赋值");
        executorService.submit(new ThreadDemo());
        executorService.shutdown();
    }

    @Override
    public void run() {
        System.out.println("----子线程获取值：" + inheritableThreadLocal.get());
    }
}

/**
 * 测试结果：
 * ----主线程启动
 * ----主线程设置后获取值：主线程第一次赋值
 * 主线程休眠2秒
 * ----子线程获取值：主线程第一次赋值
 * ----子线程获取值：主线程第一次赋值
 */
```

从上图可以看出，我们在main线程中第二次set并没有被第二次submit的线程get到。也印证了我们的结论。

针对这种情况，其实有一种通用解决思路：把线程上下文转换为任务上下文，这样的话才能避免多个任务共用一个线程上下文，为此我们不得不封装一下每一个传入线程池的任务。

但是这样做确实不是很优雅，所以为何不用`TransmittableThreadLocal`试试呢？

# TransmittableThreadLocal

针对上述情况，阿里开源了一个`TTL库`，即Transmittable ThreadLocal来解决这个问题，接下来通过一个示例演示`TransmittableThreadLocal`是否能够在线程池中实现上下文的传递，并且满足任务间上下文的隔离效果：

```java
public class ThreadDemo implements Runnable {

    public static ExecutorService executorService = Executors.newFixedThreadPool(1);

    private static TransmittableThreadLocal<String> transmittableThreadLocal = new TransmittableThreadLocal<>();

    public static void main(String[] args) {
        transmittableThreadLocal.set("旭春秋");
        Runnable task = new ThreadDemo();
        //首次提交任务
        executorService.submit(TtlRunnable.get(task));
        //主线程修改值后需要再次提交任务
        transmittableThreadLocal.set("麦克阿旭");
        executorService.submit(TtlRunnable.get(task));
    }
    @Override
    public void run() {
        System.out.println("子线程transmittableThreadLocal：" + transmittableThreadLocal.get());
    }
}

/**
 * 测试结果：
 * 子线程transmittableThreadLocal：旭春秋
 * 子线程transmittableThreadLocal：麦克阿旭
 */
```

> 1.我们准备了只有一个线程任务，主要测试线程复用的情况；
>
> 2.准备了两个任务，第一个任务检查是否能够拿到正确的上下文数据；第二个任务测试是否因为第一个任务修改上下文受到影响；

可以看出子线程中成功获取到了主线程中修改后的值。

## 使用

- 包装任务

通过上述示例，可以观察到`TransmittableThreadLocal`和`TtlRunnable`是配套使用的，即通过`TtlRunnable`包装`Runnable`接口的所有实例

无独有偶，针对`Callable`下的实例，也可以使用`TtlCallable.get()`来包装

- 包装线程池

但每次包装任务这种方法显得有些繁琐，业务场景多使用线程池，能不能让线程池代替实现包装功能呢？

你想到的这些问题，TTL已经替你实现了，你只需要传入一个线程池，就能享受丝滑的切换：

```java
TtlExecutors.getTtlExecutor(Executors.newFixedThreadPool(1));
```

思考一下，既然任务是通过包装的方式实现，那线程池是否也是这一类型呢？深入源码解析一下：

```java
@Nullable
public static Executor getTtlExecutor(@Nullable Executor executor) {
    if (TtlAgent.isTtlAgentLoaded() || null == executor || executor instanceof TtlEnhanced) {
        return executor;
    }
    return new ExecutorTtlWrapper(executor, true);
}
                    ||
                    \/

@Override
public void execute(@NonNull Runnable command) {
    executor.execute(TtlRunnable.get(command, false, idempotent));
}
```

从包装好的线程池中我们可以发现，返回的实例其实是`ExecutorTtlWrapper`对象，里面的`execute()`方法上把传进去`Runnable`参数使用`TtlRunnable.get()`做了一层包装；

## 原理

接下来要探究的是TTL如何具备这种跨线程复用的功能

从定义上看，`TransimittableThreadLocal`继承于`InheritableThreadLocal`，并实现`TtlCopier`接口，它里面只有一个`copy`方法。所以主要是对`InheritableThreadLocal`的扩展。

```java
public class TransmittableThreadLocal<T> extends InheritableThreadLocal<T> implements TtlCopier<T> 
```

### 缓存容器holder

在`TransimittableThreadLocal`中添加`holder`属性。这个属性的作用就是被标记为具备线程传递资格的对象都会被添加到这个对象中。**要标记一个类，比较容易想到的方式，就是给这个类新增一个****`Type`****字段，还有一个方法就是将具备这种类型的的对象都添加到一个静态全局集合中。之后使用时，这个集合里的所有值都具备这个标记。**

1. `holder`本身是一个`InheritableThreadLocal`对象
2. 这个`holder`对象的`value`是`WeakHashMap<TransmittableThreadLocal<Object>, ?>`

重写了`childValue`方法，实现上直接将父线程的属性作为子线程的本地变量对象。

```java
private static final InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>> holder =
        new InheritableThreadLocal<WeakHashMap<TransmittableThreadLocal<Object>, ?>>() {
            @Override
            protected WeakHashMap<TransmittableThreadLocal<Object>, ?> initialValue() {
                return new WeakHashMap<TransmittableThreadLocal<Object>, Object>();
            }

            @Override
            protected WeakHashMap<TransmittableThreadLocal<Object>, ?> childValue(WeakHashMap<TransmittableThreadLocal<Object>, ?> parentValue) {
                return new WeakHashMap<TransmittableThreadLocal<Object>, Object>(parentValue);
            }
        };
```

父线程在每次进行`get,set,remove`操作时都会对`holder`进行操作

```java
public final T get() {
    T value = super.get();
    if (disableIgnoreNullValueSemantics || null != value) addThisToHolder();
    return value;
}

public final void set(T value) {
    if (!disableIgnoreNullValueSemantics && null == value) {
        // may set null to remove value
        remove();
    } else {
        super.set(value);
        addThisToHolder();
    }
}

public final void remove() {
    removeThisFromHolder();
    super.remove();
}
```

添加和删除逻辑也比较简单，但需要注意的是holder存储的key是父线程TransmittableThreadLocal的引用，value则为空值

```java
private void addThisToHolder() {
    if (!holder.get().containsKey(this)) {
        holder.get().put((TransmittableThreadLocal<Object>) this, null); // WeakHashMap supports null value.
    }
}

public final void remove() {
    removeThisFromHolder();
    super.remove();
}
/** 删除holder中的引用 **/
private void removeThisFromHolder() {
    holder.get().remove(this);
}
```

### 如何拷贝

> TTL有一个静态内部类 Transmitter ，专门用于操作TTL本地线程缓存的重放、恢复备份、清除等操作。下面以TtlRunnable作为一个入口进行分析

无论是通过包装任务还是包装线程池的方式，底层都会通过`TtlRunnable.get(runnable)`进行增强调用，会调用到TtlRunnable的构造方法，然后调用到`capture()`拷贝方法

```java
public final class TtlRunnable implements Runnable, TtlWrapper<Runnable>, TtlEnhanced, TtlAttachments {
    private TtlRunnable(@NonNull Runnable runnable, boolean releaseTtlValueReferenceAfterRun) {
        // capturedRef：拷贝副本的引用
        this.capturedRef = new AtomicReference<Object>(capture());
        // runnable待执行逻辑对象
        this.runnable = runnable;
        this.releaseTtlValueReferenceAfterRun = releaseTtlValueReferenceAfterRun;
    }
}

/**************************************************
 * capture()：拷贝副本
 * - 分为TTL拷贝、ThreadLocal拷贝
 **************************************************/
public static Object capture() {
    // 抓取快照
    return new Snapshot(captureTtlValues(), captureThreadLocalValues());
}
/** 抓取 TransmittableThreadLocal 的快照 **/
private static WeakHashMap<TransmittableThreadLocalCode<Object>, Object> captureTtlValues() {
    WeakHashMap<TransmittableThreadLocalCode<Object>, Object> ttl2Value = new WeakHashMap<TransmittableThreadLocalCode<Object>, Object>();
    // 主线程和子线程其实都是共用一个holder的，所以主线程new一个TTL并做一个set操作之后，会搞一份数据put到holder中。
    // 这时候就可以进行一个副本的拷贝，遍历holder子线程的值，然后拷贝一份出来
    // eg：主线程用这个ttl.set("我是主线程");，这时候holder就会对应多了要给ttl，并且值是"我是主线程"
    for (TransmittableThreadLocalCode<Object> threadLocal : holder.get().keySet()) {
        // threadLocal.copyValue()默认还是拷贝引用
        ttl2Value.put(threadLocal, threadLocal.copyValue());
    }
    return ttl2Value;
}
/** 抓取 ThreadLocal 的快照 **/
private static WeakHashMap<ThreadLocal<Object>, Object> captureThreadLocalValues() {
    final WeakHashMap<ThreadLocal<Object>, Object> threadLocal2Value = new WeakHashMap<ThreadLocal<Object>, Object>();
    // 从 threadLocalHolder 中，遍历注册的 ThreadLocal，将 ThreadLocal 和 TtlCopier 取出，将值复制到 Map 中
    for (Map.Entry<ThreadLocal<Object>, TtlCopier<Object>> entry : threadLocalHolder.entrySet()) {
        final ThreadLocal<Object> threadLocal = entry.getKey();
        final TtlCopier<Object> copier = entry.getValue();
        // 默认拷贝的是引用
        threadLocal2Value.put(threadLocal, copier.copy(threadLocal.get()));
    }
    return threadLocal2Value;
}
```

### 如何维护

```java
@Override
public void run() {
    /**
     * capturedRef是主线程传递下来的ThreadLocal的值。
     */
    Object captured = capturedRef.get();
    if (captured == null || releaseTtlValueReferenceAfterRun && !capturedRef.compareAndSet(captured, null)) {
        throw new IllegalStateException("TTL value reference is released after run!");
    }
    /**
     * 1.  backup（备份）是线程已经存在的ThreadLocal变量；
     * 2. 将captured的ThreadLocal值在子线程中set进去；
     */
    Object backup = replay(captured);
    try {
        /**
         * 待执行的线程方法；
         */
        runnable.run();
    } finally {
        /**
         *  在子线程任务中，ThreadLocal可能发生变化，该步骤的目的是
         *  回滚{@code runnable.run()}进入前的ThreadLocal的线程
         */
        restore(backup);
    }
}
 /**
 * 将快照重做到执行线程
 * @param captured 快照
 */
public static Object replay(Object captured) {
    // 获取父线程ThreadLocal快照
    final Snapshot capturedSnapshot = (Snapshot) captured;
    return new Snapshot(replayTtlValues(capturedSnapshot.ttl2Value), replayThreadLocalValues(capturedSnapshot.threadLocal2Value));
}

/*****************************************************
 * 重放TransmittableThreadLocal，并保存执行线程的原值
 ****************************************************/
private static WeakHashMap<TransmittableThreadLocalCode<Object>, Object> replayTtlValues(WeakHashMap<TransmittableThreadLocalCode<Object>, Object> captured) {
    WeakHashMap<TransmittableThreadLocalCode<Object>, Object> backup = new WeakHashMap<TransmittableThreadLocalCode<Object>, Object>();

    for (final Iterator<TransmittableThreadLocalCode<Object>> iterator = holder.get().keySet().iterator(); iterator.hasNext(); ) {
        TransmittableThreadLocalCode<Object> threadLocal = iterator.next();

        // backup
        // 遍历 holder，从 父线程继承过来的,或者之前注册进来的
        backup.put(threadLocal, threadLocal.get());

        // clear the TTL values that is not in captured
        // avoid the extra TTL values after replay when run task
        // 清除本次没有传递过来的 ThreadLocal，和对应值
        //  -- 第一点：可能会有因为 InheritableThreadLocal 而传递并保留的值
        //  -- 第二点：保证主线程set过的ThreadLocal不被传递过来。明确其传递是由业务代码控制，就是明确 set 过值的
        if (!captured.containsKey(threadLocal)) {
            iterator.remove();
            threadLocal.superRemove();
        }
    }

    // set TTL values to captured
    // 将 map 中的值，设置到快照
    // 内部调用了 beforeExecute 和 afterExecute 方法。默认不做任何处理
    setTtlValuesTo(captured);

    // call beforeExecute callback
    // TransmittableThreadLocal 的回调方法，在任务执行前执行
    doExecuteCallback(true);

    return backup;
}

private static WeakHashMap<ThreadLocal<Object>, Object> replayThreadLocalValues( WeakHashMap<ThreadLocal<Object>, Object> captured) {
    final WeakHashMap<ThreadLocal<Object>, Object> backup = new WeakHashMap<ThreadLocal<Object>, Object>();

    for (Map.Entry<ThreadLocal<Object>, Object> entry : captured.entrySet()) {
        final ThreadLocal<Object> threadLocal = entry.getKey();
        backup.put(threadLocal, threadLocal.get());

        final Object value = entry.getValue();
        // 如果值是标记已删除，则清除
        if (value == threadLocalClearMark) threadLocal.remove();
        else threadLocal.set(value);
    }

    return backup;
}
/*********************************************
 * 恢复备份的原快照
 *********************************************/
public static void restore( Object backup) {
    // 将之前保存的TTL和threadLocal原来的数据覆盖回去
    final Snapshot backupSnapshot = (Snapshot) backup;
    restoreTtlValues(backupSnapshot.ttl2Value);
    restoreThreadLocalValues(backupSnapshot.threadLocal2Value);
}

private static void restoreTtlValues( WeakHashMap<TransmittableThreadLocalCode<Object>, Object> backup) {
    // call afterExecute callback
    // 调用执行完后回调接口
    doExecuteCallback(false);

    // 移除子线程新增的TTL
    for (final Iterator<TransmittableThreadLocalCode<Object>> iterator = holder.get().keySet().iterator(); iterator.hasNext(); ) {
        TransmittableThreadLocalCode<Object> threadLocal = iterator.next();
        // 恢复快照时，清除本次传递注册进来，但是原先不存在的 TransmittableThreadLocal
        // 移除掉所有不在备份里面的TTL数据，应该是为了避免内存泄漏吧
        // clear the TTL values that is not in backup
        // avoid the extra TTL values after restore
        if (!backup.containsKey(threadLocal)) {
            iterator.remove();
            threadLocal.superRemove();
        }
    }

    // 重置为原来的数据(就是恢复回备份前的值)
    // restore TTL values
    setTtlValuesTo(backup);
}

private static void setTtlValuesTo( WeakHashMap<TransmittableThreadLocalCode<Object>, Object> ttlValues) {
    for (Map.Entry<TransmittableThreadLocalCode<Object>, Object> entry : ttlValues.entrySet()) {
        TransmittableThreadLocalCode<Object> threadLocal = entry.getKey();
        // set 的同时，也就将 TransmittableThreadLocal 注册到当前线程的注册表了
        threadLocal.set(entry.getValue());
    }
}
```

可以看到，TtlRunnable利用`holder`维护了一个**线程级别的的****缓存**，每次调用run()前后进行set和还原数据。

## 其他注意点

1. TTL为什么不直接继承ThreadLocal？

- 因为有些业务需要用到ITL特性，如果直接继承ThreadLocal，就会丢失ITL的父拷贝到子线程数据的特性(子线程创建时拷贝)

1. 为什么需要在run执行完之后调用restore()？

- restore里面会主动调用remove()回收，避免内存泄露（会删除子线程新增的TTL）
- 不调用restore()的话，就会覆盖之前backup备份部分子线程的数据，这样可能在业务上有隐患

1. TTL存在线程安全问题？

- 存在的，因为默认都是引用类型拷贝，如果子线程修改了数据，主线程是可以感知到的

1. TTL是否存在内存泄露问题？

- TTL维护的holder本身是一个static来的，使用的时候会调用restore(),然后里面显式调用remove()清楚子线程新增TTL，所以正确使用下是没有内存泄露问题的

### 安全or性能？

可见`TransmittableThreadLocal`不仅保证变量在线程间传递，又不会存在脏数据的问题。

而安全 和 性能是相反的一组质量属性，如果要求非常安全，那么性能这个质量属性势必会降低。

同理TransmittableThreadLocal解决了这父子线程的相关问题，那么势必也会导致一些其它问题，例如：

- **复杂性增加**
- **性能影响**

特别是在频繁创建子线程或进行大量的线程上下文传递的情况下，这种性能开销可能会变得比较明显。

# 总结

- 原生`ThreadLocal`的局限性：创建子线程，threadlocal值不会自动传过去。
- `InheritableThreadLocal`的局限性：创建的子线程，threadlocal会自动传过去，但是threadlocal的生命周期和父线程不一致，并且通常子线程是在线程池中的线程是一直存在的，所以会造成内存泄漏
- `TransmittableThreadLocal`：创建子线程会将threadlocal传过去，并且子线程执行完毕后会还原数据。

# 文章参考

https://juejin.cn/post/6844904102015713293

https://juejin.cn/post/7064370304595787783

https://juejin.cn/post/7171019628750209031