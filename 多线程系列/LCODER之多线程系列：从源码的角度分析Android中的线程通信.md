#### 准备知识
ThreadLocal： 
   首先要搞清楚ThreadLocal的作用是什么，然后再去看它的源码。ThreadLocal的作用是为了实现线程间的数据隔离。（分析源码就要分析为什么ThreadLocal能做到数据隔离，以及，它在Handler中起了什么作用？）

#### 第一个问题： ThreadLocal是如何做到数据隔离的？
  要搞清楚这个问题，首先我们追踪源码，看一下，ThreadLocal是怎么放置和取出数据的。

<!--###### 1.1 ThreadLocal的初始化方法：initialValue()。
源码中对该方法的实现是：
```
     protected T initialValue() {
        return null;
       }
```
默认返回为空的。
-->
###### 1.1 从ThreadLocal中取数据:get()
```
     public T get() {
        //拿到当前线程 
        Thread t = Thread.currentThread();
        //拿到当前线程中的 ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //通过下面两步，追踪可以看到，map此时为空。
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```
点击进入源码发现这个方法首先是获取到当前的线程，然后拿到当前线程中的ThreadLocalMap。我们追踪其中的getMap(t):
```
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
返回的是threadLocals，这是Thread类中的一个全局变量。追踪进去可以看到：
Thread.java
```
 ThreadLocal.ThreadLocalMap threadLocals = null;
  threadLocals = null;
```
在Thread.java中，对这个全局变量的定义均为null。因此在get()中，map为空，会走到setInitialValue()中。我们继续追踪到setInitialValue()中。看看setInitialValue()在源码中如何实现的：
```
    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```
首先调用初始化方法initialValue()，得到value，如果用户重写了initialValue()，那么获得到就是用户定义的返回值。再往下走，得到的Map仍然是null，因此会走到createMap(t, value)中。继续追踪下去，看看createMap()是怎么实现的。
```
  void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
在这个方法中，定义了threadLocals这个变量。创建了ThreadLocalMap。

#### 1.2 往ThreadLocal中添加数据。set()
```
   public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

```
仍然是这样：首先获取当前线程，再通过当前线程获取到 ThreadLocalMap，在之前通过对getMap(t)的分析可以知道，此时的map = null，因此set(t)最终也会走 createMap(t, value)。之前对 createMap(t, value)的源码进行分析过， createMap(t, value)会创建一个ThreadLocalMap，并将value放入ThreadLocalMap中。

看到这里，我们发现，ThreadLocal取数据和拿数据，都是通过一个叫做ThreadLocalMap的类，那么这个类到底是个啥呢？
阅读源码发现，ThreadLocalMap是ThreadLocal中的一个静态内部类：
```
 static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;
      .......
```
这个静态内部类中，还有一个静态内部类，这个类叫做Entry，这个类比较简单，它其中维护了两个变量，分别是ThreadLocal类型的Key和Object类型的Value。在每个ThreadLocalMap中都有一个Entry类型的数组Entry[]，用来存储数据。

源码阅读到这里，再结合到上面分析的ThreadLocal源码中set()的实现：
```
 if (map != null)
 map.set(this, value);
```
这里的Map指的是ThreadLocalMap，追踪到ThreadLocalMap中的set()
```
 /**
         * Set the value associated with key.
         *
         * @param key the thread local object
         * @param value the value to be set
         */
        private void set(ThreadLocal<?> key, Object value) {

            // We don't use a fast path as with get() because it is at
            // least as common to use set() to create new entries as
            // it is to replace existing ones, in which case, a fast
            // path would fail more often than not.

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
通过代码可以看到，ThreadLocalMap装数据的过程，就是把数据放入Entry中的过程，而Entry就是把当前的ThreadLocal作为Key值，用户要放入的值作为Value，存入Entry中。

那么，ThreadLocal是如何做到数据隔离的呢？
来看下面这个图：![线程间的数据隔离](//p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9db2a1f6a2264ab8afb3682fdaec0edf~tplv-k3u1fbpfcp-zoom-1.image)

每个线程中有一个自己的ThreadLocalMap，这句话，看源码可以看出来，Thread的源码中有这样一句话：
```
/* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
在Thread中，threadLocals是null的，它是在哪里被赋值的呢？ 答案是在ThreadLocal中被赋值的。我们上面分析ThreadLocal的set()和get()的源码可以知道，当我们在ThreadLocal中set()或者get()时，如果初始值为空，或者要get()的值为空，都会判断ThreadLocalMap是否为空，如果ThreadLocalMap为空，会走到createMap中，而createMap的源码：
```
/**
     * Create the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param t the current thread 在前面的代码中： Thread t = Thread.currentThread(); 可以得到这一结论
     * @param firstValue value for the initial entry of the map
     */

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
一目了然，每个线程中都会new 一个新的ThreadLocalMap，每一个ThreadLocalMap的key值都是ThreadLocal，对于同一个ThreadLocal变量来说，这个key是一样的，但是并不会冲突，因为存储数据的是ThreadLocalMap中的Entry数组，即使Key相同，每个线程中的ThreadLocalMap不同，是两个完全不相干的ThreadLocalMap，其中的Entry[]更不同。这样，ThreadLocal就做到了线程间的数据隔离。

简单来说，为什么ThreadLocal可以做到线程间的数据隔离，因为每个线程中都有一个自己的ThreadLocalMap来存储数据。

#### 第二个问题，ThreadLocal在Android中起到了什么作用呢？
#### 从App的启动开始分析，Android的消息机制。

当App启动时，会创建全局唯一的Looper和MessageQueue对象。
这句话如何验证呢？当然是去看源码。

App的程序入口在哪里呢？在ActivityThread的main()这里。这个main()就是我们Java当中的main(),也就是程序的入口。
在main()中，调用了Looper.prepareMainLooper()。这个方法创建了全局唯一的MainLooper。我们来看一下这个方法是如何实现的：
```
 public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
首先看 prepare(false) 这句，追踪到 prepare(false)中：
```
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            // 此时还未创建过Looper ，因此抛出异常
            throw new RuntimeException("Only one Looper may be created per thread");
        }
       // set Looper  
        sThreadLocal.set(new Looper(quitAllowed));
    }

```
在这个方法中往sThreadLocal中set了一个Looper，再往下走，myLooper()，看看myLooper()是如何实现的：
```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
可以非常清楚的看到，这里调用了ThreaLoacl的get(),通过刚才对ThreadLocal的分析，在set()调用后，get()获得的就是set进ThreaLocal中的数据。
接下来再进入Looper的构造方法中看一看：
```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```
这里创建了MessageQueue。

这里的一切，都是在主线程中进行的。由于ThreadLocal可以将线程间的数据隔离，因此，这个在主线程中创建的Looper和MessageQueue，是主线程中的。

#### 发送消息
Handler登上舞台，发送消息离不开Handler。

如果要从子线程中，发送消息到主线程，那么，是这样的写法：
1. 创建Handler
2. 创建子线程，在子线程中进行网络请求等操作。
3. 子线程网络请求结束，通过handler发送消息给主线程，通知主线程。
MainActivity.java
```
public class MainActivity extends AppCompatActivity {

//1.创建Handler
Handler mHandler = new Handler(){
    @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
};

//2. 创建子线程，在子线程中进行网络请求等操作。
new Thread(new Runnable() {
            @Override
            public void run() {
                //.....网络请求等操作

                //3. 子线程网络请求结束，通过handler发送消息给主线程，通知主线程。
                Message message = Message.obtain();
                message.what = 1;
                mHandler.sendMessage(message);

            }
        }).start();
 
}

```
基本上一个Handler发送消息的过程就是这样。那么到底是怎样通过Handler把消息从子线程发送到主线程的呢？
看源码： 首先看Handler的源码。我们首先分析的是sendMessage()
```
public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }
    
    
    ....
    
    public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
    ....
    
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

```
一步一步追踪下来，我们可以看到，最终调用的是sendMessageAtTime()，首先看这个方法的第一句：
MessageQueue queue = mQueue;mQueue在Handler中赋值的过程是这样的：

```
 public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }

        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
    
```
可以通过 mQueue = mLooper.mQueue;mLooper = Looper.myLooper();这两句代码得到：mLooper是主线程中的Looper，而mQueue是主线程中的MessageQueue。
怎么得到这一结论的呢？ 通过mLooper = Looper.myLooper();这一关键代码得到的。前面已经分析过myLooper()。
```
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
myLooper()，就是获取ThreadLocal中的数据。而ThreadLocal中的数据就是主线程中的Looper。既然Looper是主线程中的Looper，那么mQueue当然就是Looper构造方法中创建的那一个MessageQueue了。分析到这里，我们就发现了，此时子线程和主线程已经联系起来了。


再接着分析sendMessageAtTime()，此时，我们已经知道，sendMessageAtTime()中的MessageQueue被主线程的mQueue赋值，也就是说sendMessageAtTime()中的MessageQueue就是主线程的MessageQueue。此时mQueue!=null 程序往下走到enqueueMessage(queue, msg, uptimeMillis);这里来。
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```
分析enqueueMessage：


msg.target = this;这里的this就是此时这个Handler。首先点进Message的源码中，target就是一个Handler。此时将这个Handler赋值给msg，在处理消息的过程中，会使用这个Handler进行消息的分发。


queue.enqueueMessage(msg, uptimeMillis);
在这里就是往主线程的MessageQueue中插入Message了。
```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                //把msg赋值给全局变量mMessages
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }
```
这里的关键代码,是一个死循环,MessageQueue就是一个链表。当有消息到来时，就会往链表中插入一条消息。也就是说，这时，主线程的MessageQueuq中已经有一条消息了。

### 处理消息：
在ActivityThread的main()中，下面有一句
Looper.loop();我们追踪进去看一下：
```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        // Allow overriding a threshold with a system prop. e.g.
        // adb shell 'setprop log.looper.1000.main.slow 1 && stop && start'
        final int thresholdOverride =
                SystemProperties.getInt("log.looper."
                        + Process.myUid() + "."
                        + Thread.currentThread().getName()
                        + ".slow", 0);

        boolean slowDeliveryDetected = false;

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;
            long slowDeliveryThresholdMs = me.mSlowDeliveryThresholdMs;
            if (thresholdOverride > 0) {
                slowDispatchThresholdMs = thresholdOverride;
                slowDeliveryThresholdMs = thresholdOverride;
            }
            final boolean logSlowDelivery = (slowDeliveryThresholdMs > 0) && (msg.when > 0);
            final boolean logSlowDispatch = (slowDispatchThresholdMs > 0);

            final boolean needStartTime = logSlowDelivery || logSlowDispatch;
            final boolean needEndTime = logSlowDispatch;

            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }

            final long dispatchStart = needStartTime ? SystemClock.uptimeMillis() : 0;
            final long dispatchEnd;
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (logSlowDelivery) {
                if (slowDeliveryDetected) {
                    if ((dispatchStart - msg.when) <= 10) {
                        Slog.w(TAG, "Drained");
                        slowDeliveryDetected = false;
                    }
                } else {
                    if (showSlowLog(slowDeliveryThresholdMs, msg.when, dispatchStart, "delivery",
                            msg)) {
                        // Once we write a slow delivery log, suppress until the queue drains.
                        slowDeliveryDetected = true;
                    }
                }
            }
            if (logSlowDispatch) {
                showSlowLog(slowDispatchThresholdMs, dispatchStart, dispatchEnd, "dispatch", msg);
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }
```
代码很长 ，我们首先抽出最关键代码进行分析：
```
 final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        
        
        
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
           
            try {
                msg.target.dispatchMessage(msg);
                dispatchEnd = needEndTime ? SystemClock.uptimeMillis() : 0;
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            msg.recycleUnchecked();
        }
```
首先，还是通过myLooper()，取得ThreadLocal中的Looper，再取得Looper中的MessageQueue，此时获取的Looper和MessageQueue都是主线程中的。
接下来是一个死循环：for(;;) 在其中通过queue.next()，不断的从MessageQueue中取消息，接下来  if (msg == null) return;也就是说，当消息为空时，不执行下面的操作，只有消息不为null走到下面一句msg.target.dispatchMessage(msg);使用msg中的Handler，进行消息的分发。
继续跟踪到dispatchMessage(msg)中：
```
public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```
可以看到，这里出现了handleMessage(msg); 
```
public void handleMessage(Message msg) {
    }
```
这个handleMessage(Message msg)就是我们重写的Handler中的handleMessage(Message msg)，而这个消息就被传递过来，供我们使用。

大致的流程就是这样。

### MessageQueue的队列处理机制。




### Message的创建，享元设计模式
### 同步屏障

#### 同步屏障：顾名思义，就是阻碍同步消息，让异步消息通过。
  同步消息是指那些普通的，老老实实排队的消息。如果我们的程序需要有紧急消息处理，比如更新UI的操作，我们知道，对于60hz刷新率的手机来说，Android是16ms更新一次UI。更新UI的操作是不能等的，否则就会产生卡顿。Android为了解决这种紧急消息的处理，引入了同步屏障的概念。
  
  同步屏障有什么特征呢？ 既然同步屏障是要阻碍同步消息，让异步消息通过，那么，肯定是在取消息的时候做文章，所以我们来看一下MessageQueue的next()方法，不看不知道，一看吓一跳：
```
 Message next() {
       
        ...// 省略前面的代码
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                // 这里是关键！！ 在这里如果msg.target == null 时，进入循环，去找到异步消息，进行处理
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        // Got a message.
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

               ...// 省略后面的代码
    }
```
  
查看代码发现，当msg.target == null时，会去MessageQueue中轮询消息，直到找出异步消息为止。

所以，成为同步屏障的关键就是把msg的target置为空。

发送同步屏障：MessageQueue中有一个方法，是专门来发送同步屏障的：postSyncBarrier 。 来看一下源码：
```
public int postSyncBarrier() {
        return postSyncBarrier(SystemClock.uptimeMillis());
    }

    private int postSyncBarrier(long when) {
        // Enqueue a new sync barrier token.
        // We don't need to wake the queue because the purpose of a barrier is to stall it.
        synchronized (this) {
            final int token = mNextBarrierToken++;
            // 从消息池里面取出Message
            final Message msg = Message.obtain();
            msg.markInUse();
            msg.when = when;
            msg.arg1 = token;
            
            // 这里在给Message的属性赋值时，并没有给msg的target赋值，这里的Message就是一个同步屏障

            Message prev = null;
            Message p = mMessages;
            if (when != 0) {
                while (p != null && p.when <= when) {
                //如果开启同步屏障的时间（假设记为T）T不为0，且当前的同步消息里有时间小于T，则prev也不为null

                    prev = p;
                    p = p.next;
                }
            }
            if (prev != null) { // invariant: p == prev.next
            //根据prev是不是为null，将 msg 按照时间顺序插入到 消息队列（链表）的合适位置

                msg.next = p;
                prev.next = msg;
            } else {
                msg.next = p;
                mMessages = msg;
            }
            return token;
        }
    }
```



### HandlerThread
什么是HandlerThread？ 看源码：
![HandlerThread](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ba48f7b63eb4579aa4ba00bfb0c43c3~tplv-k3u1fbpfcp-zoom-1.image)
HandlerThread，是Thread的一个子类，严格意义上来说是一个线程，那么它和别的线程相比有什么特殊之处呢？我们知道，线程中的Looper是需要我们在使用的时候手动去创建的，HandlerThread在它的run()方法中，帮我们做了Looper的创建工作：
```
// HandlerThread的run()方法
 @Override
    public void run() {
        mTid = Process.myTid();
        // 在这里创建Looper
        Looper.prepare();
        synchronized (this) {
            // 在这里获取线程对应的唯一Looper
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
```
为我们使用线程和Looper提供了便利，不用再手动的创建Looper。

HandlerThread不光提前创建了Looper，它还有个作用，保证线程安全。


### IntentService
