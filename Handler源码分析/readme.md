实例化Handler之前需要调用``Looper.prepare()``,在主线程ActivityThread中也是如此
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffn0esbzn4j31ki134alv.jpg) 
可以看到先调用了``Looper.prepareMainLooper``方法，在``prepareMainLooper``方法中调用了``prepare``方法。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffn0hgrlgpj31kq0g642y.jpg)
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffn0igcklrj31i00foq7q.jpg) 
在``prepare``方法中，将ActivityThread的``threadLocals``这个``ThreadLocalMap``实例化并且将``ThreadLocal``作为key,``new Looper()``作为value保存进去。
接着调用``Looper.loop()``方法。

```
    /**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() { 
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
        for(; ; ){
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            msg.recycleUnchecked();
        }
```

loop()进入循环模式，不断重复下面的操作，直到没有消息时退出循环

+ 读取MessageQueue的下一条Message；
+ 把Message分发给相应的target；
+ 再把分发后的Message回收到消息池，以便重复利用。

创建Handler

```
public Handler() {
    this(null, false);
}

public Handler(Callback callback, boolean async) {
    //匿名类、内部类或本地类都必须申明为static，否则会警告可能出现内存泄露
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
    //必须先执行Looper.prepare()，才能获取Looper对象，否则为null.
    mLooper = Looper.myLooper();  //从当前线程的TLS中获取Looper对象【见2.1】
    if (mLooper == null) {
        throw new RuntimeException("");
    }
    mQueue = mLooper.mQueue; //消息队列，来自Looper对象
    mCallback = callback;  //回调方法
    mAsynchronous = async; //设置消息是否为异步处理方式
}
```

对于Handler的无参构造方法，默认采用当前线程TLS中的Looper对象，并且callback回调方法为null，且消息为同步处理方式。只要执行的Looper.prepare()方法，那么便可以获取有效的Looper对象。

```
public Handler(Looper looper) {
    this(looper, null, false);
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

Handler类在构造方法中，可指定Looper，Callback回调方法以及消息的处理方式(同步或异步)，对于无参的handler，默认是当前线程的Looper。

消息分发机制,在Looper.loop()中，当发现有消息时，调用消息的目标handler，执行dispatchMessage()方法来分发消息。

```
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //当Message存在回调方法，回调msg.callback.run()方法；
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //当Handler存在Callback成员变量时，回调方法handleMessage()；
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //Handler自身的回调方法handleMessage()
        handleMessage(msg);
    }
}
```

分发消息流程：

+ 当Message的回调方法不为空时，则回调方法msg.callback.run()，其中callBack数据类型为Runnable,否则进入步骤2；
+ 当Handler的mCallback成员变量不为空时，则回调方法mCallback.handleMessage(msg),否则进入步骤3；
+ 调用Handler自身的回调方法handleMessage()，该方法默认为空，Handler子类通过覆写该方法来完成具体的逻辑。

![](http://ww1.sinaimg.cn/large/006tNc79ly1ffn103ebebj30fe0b5jrk.jpg)

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
``Handler.sendEmptyMessage()``等系列方法最终调用``MessageQueue.enqueueMessage(msg, uptimeMillis)``，将消息添加到消息队列中，其中uptimeMillis为系统当前的运行时间，不包括休眠时间。

``MessageQueue``是消息机制的Java层和C++层的连接纽带，大部分核心方法都交给native层来处理。
创建MessageQueue
```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    //通过native方法初始化消息队列，其中mPtr是供native代码使用
    mPtr = nativeInit();
}
```
next()提取下一条message
```
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) { //当消息循环已经退出，则直接返回
        return null;
    }
    int pendingIdleHandlerCount = -1; // 循环迭代的首次为-1
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }
        //阻塞操作，当等待nextPollTimeoutMillis时长，或者消息队列被唤醒，都会返回
        nativePollOnce(ptr, nextPollTimeoutMillis);
        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                //当消息Handler为空时，查询MessageQueue中的下一条异步消息msg，则退出循环。
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //当异步消息触发时间大于当前时间，则设置下一次轮询的超时时长
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // 获取一条消息，并返回
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    //设置消息的使用状态，即flags |= FLAG_IN_USE
                    msg.markInUse();
                    return msg;   //成功地获取MessageQueue中的下一条即将要执行的消息
                }
            } else {
                //没有消息
                nextPollTimeoutMillis = -1;
            }
            //消息正在退出，返回null
            if (mQuitting) {
                dispose();
                return null;
            }
            //当消息队列为空，或者是消息队列的第一个消息时
            if (pendingIdleHandlerCount < 0 && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                //没有idle handlers 需要运行，则循环并等待。
                mBlocked = true;
                continue;
            }
            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }
        //只有第一次循环时，会运行idle handlers，执行完成后，重置pendingIdleHandlerCount为0.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; //去掉handler的引用
            boolean keep = false;
            try {
                keep = idler.queueIdle();  //idle时执行的方法
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }
            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }
        //重置idle handler个数为0，以保证不会再次重复运行
        pendingIdleHandlerCount = 0;
        //当调用一个空闲handler时，一个新message能够被分发，因此无需等待可以直接查询pending message.
        nextPollTimeoutMillis = 0;
    }
}
```
nativePollOnce是阻塞操作，其中nextPollTimeoutMillis代表下一个消息到来前，还需要等待的时长；当nextPollTimeoutMillis = -1时，表示消息队列中无消息，会一直等待下去。

当处于空闲时，往往会执行IdleHandler中的方法。当nativePollOnce()返回后，next()从mMessages中提取一个消息。

enqueueMessage添加一条消息到消息队列

```
boolean enqueueMessage(Message msg, long when) {
    // 每一个Message必须有一个target
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }
    synchronized (this) {
        if (mQuitting) {  //正在退出时，回收msg，加入到消息池
            msg.recycle();
            return false;
        }
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            //p为null(代表MessageQueue没有消息） 或者msg的触发时间是队列中最早的， 则进入该该分支
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked; //当阻塞时需要唤醒
        } else {
            //将消息按时间顺序插入到MessageQueue。一般地，不需要唤醒事件队列，除非
            //消息队头存在barrier，并且同时Message是队列中最早的异步消息。
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
            msg.next = p;
            prev.next = msg;
        }
        //消息没有退出，我们认为此时mPtr != 0
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

MessageQueue是按照Message触发时间的先后顺序排列的，队头的消息是将要最早触发的消息。当有消息需要加入消息队列时，会从队列头开始遍历，直到找到消息应该插入的合适位置，以保证所有消息的时间顺序。


``MessageQueue.removeMessages``

```
void removeMessages(Handler h, int what, Object object) {
    if (h == null) {
        return;
    }
    synchronized (this) {
        Message p = mMessages;
        //从消息队列的头部开始，移除所有符合条件的消息
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }
        //移除剩余的符合要求的消息
        while (p != null) {
            Message n = p.next;
            if (n != null) {
                if (n.target == h && n.what == what
                    && (object == null || n.obj == object)) {
                    Message nn = n.next;
                    n.recycleUnchecked();
                    p.next = nn;
                    continue;
                }
            }
            p = n;
        }
    }
}
```

这个移除消息的方法，采用了两个while循环，第一个循环是从队头开始，移除符合条件的消息，第二个循环是从头部移除完连续的满足条件的消息之后，再从队列后面继续查询是否有满足条件的消息需要被移除。

在代码中，可能经常看到recycle()方法，咋一看，可能是在做虚拟机的gc()相关的工作，其实不然，这是用于把消息加入到消息池的作用。这样的好处是，当消息池不为空时，可以直接从消息池中获取Message对象，而不是直接创建，提高效率。

静态变量sPool的数据类型为Message，通过next成员变量，维护一个消息池；静态变量MAX_POOL_SIZE代表消息池的可用大小；消息池的默认大小为50。

消息池常用的操作方法是obtain()和recycle()。

```
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null; //从sPool中取出一个Message对象，并消息链表断开
            m.flags = 0; // 清除in-use flag
            sPoolSize--; //消息池的可用大小进行减1操作
            return m;
        }
    }
    return new Message(); // 当消息池为空时，直接创建Message对象
}
```
obtain()，从消息池取Message，都是把消息池表头的Message取走，再把表头指向next;

```
public void recycle() {
    if (isInUse()) { //判断消息是否正在使用
        if (gCheckRecycle) { //Android 5.0以后的版本默认为true,之前的版本默认为false.
            throw new IllegalStateException("This message cannot be recycled because it is still in use.");
        }
        return;
    }
    recycleUnchecked();
}

//对于不再使用的消息，加入到消息池
void recycleUnchecked() {
    //将消息标示位置为IN_USE，并清空消息所有的参数。
    flags = FLAG_IN_USE;
    what = 0;
    arg1 = 0;
    arg2 = 0;
    obj = null;
    replyTo = null;
    sendingUid = -1;
    when = 0;
    target = null;
    callback = null;
    data = null;
    synchronized (sPoolSync) {
        if (sPoolSize < MAX_POOL_SIZE) { //当消息池没有满时，将Message对象加入消息池
            next = sPool;
            sPool = this;
            sPoolSize++; //消息池的可用大小进行加1操作
        }
    }
}
```

recycle()，将Message加入到消息池的过程，都是把Message加到链表的表头；

![](http://ww1.sinaimg.cn/large/006tNc79ly1ffn22l5l94j30p20fatah.jpg)

+ Handler通过sendMessage()发送Message到MessageQueue队列；
+ Looper通过loop()，不断提取出达到触发条件的Message，并将Message交给target来处理；
+ 经过dispatchMessage()后，交回给Handler的handleMessage()来进行相应地处理。
+ 将Message加入MessageQueue时，处往管道写入字符，可以会唤醒loop线程；如果MessageQueue中没有Message，并处于Idle状态，则会执行IdelHandler接口中的方法，往往用于做一些清理性地工作。

消息分发的优先级：
+ Message的回调方法：message.callback.run()，优先级最高；
+ Handler的回调方法：Handler.mCallback.handleMessage(msg)，优先级仅次于1；
+ Handler的默认方法：Handler.handleMessage(msg)，优先级最低。

===================================================================

MessageQueue类里面涉及到多个native方法，除了MessageQueue的native方法，native层本身也有一套完整的消息机制，用于处理native的消息。在整个消息机制中，而MessageQueue是连接Java层和Native层的纽带，换言之，Java层可以向MessageQueue消息队列中添加消息，Native层也可以向MessageQueue消息队列中添加消息。

![](http://ww3.sinaimg.cn/large/006tNc79ly1ffn2oadbmdj30ic0a3gmf.jpg)

在MessageQueue中的native方法如下：

```
private native static long nativeInit();
private native static void nativeDestroy(long ptr);
private native void nativePollOnce(long ptr, int timeoutMillis);
private native static void nativeWake(long ptr);
private native static boolean nativeIsPolling(long ptr);
private native static void nativeSetFileDescriptorEvents(long ptr, int fd, int events);
```

1.nativeInit()初始化过程的调用链如下：

![](http://ww1.sinaimg.cn/large/006tNc79ly1ffn2q3yrmcj30jq0i2glw.jpg)

下面来进一步来看看调用链的过程：

【1】new MessageQueue()
==> MessageQueue.java

```
MessageQueue(boolean quitAllowed) {
    mQuitAllowed = quitAllowed;
    mPtr = nativeInit();  //mPtr记录native消息队列的信息 【2】
}
```

【2】android_os_MessageQueue_nativeInit()
```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue(); //初始化native消息队列 【3】
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }
    nativeMessageQueue->incStrong(env);
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

【3】new NativeMessageQueue()
android_os_MessageQueue.cpp
```
NativeMessageQueue::NativeMessageQueue() : mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread(); //获取TLS中的Looper对象
    if (mLooper == NULL) {
        mLooper = new Looper(false); //创建native层的Looper 【4】
        Looper::setForThread(mLooper); //保存native层的Looper到TLS中
    }
}
```

+ ``Looper::getForThread()``，功能类比于Java层的Looper.myLooper();
+ ``Looper::setForThread(mLooper)``，功能类比于Java层的ThreadLocal.set();
``MessageQueue``是在Java层与Native层有着紧密的联系，但是此次Native层的Looper与Java层的Looper没有任何的关系，可以发现native基本等价于用C++重写了Java的Looper逻辑，故可以发现很多功能类似的地方。

【4】new Looper()
==> Looper.cpp

```
Looper::Looper(bool allowNonCallbacks) :
        mAllowNonCallbacks(allowNonCallbacks), mSendingMessage(false),
        mPolling(false), mEpollFd(-1), mEpollRebuildRequired(false),
        mNextRequestSeq(0), mResponseIndex(0), mNextMessageUptime(LLONG_MAX) {
    mWakeEventFd = eventfd(0, EFD_NONBLOCK); //构造唤醒事件的fd
    AutoMutex _l(mLock);
    rebuildEpollLocked();  //重建Epoll事件【5】
}
```

【5】epoll_create/epoll_ctl
==> Looper.cpp

```
void Looper::rebuildEpollLocked() {
    if (mEpollFd >= 0) {
        close(mEpollFd); //关闭旧的epoll实例
    }
    mEpollFd = epoll_create(EPOLL_SIZE_HINT); //创建新的epoll实例，并注册wake管道
    struct epoll_event eventItem;
    memset(& eventItem, 0, sizeof(epoll_event)); //把未使用的数据区域进行置0操作
    eventItem.events = EPOLLIN; //可读事件
    eventItem.data.fd = mWakeEventFd;
    //将唤醒事件(mWakeEventFd)添加到epoll实例(mEpollFd)
    int result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeEventFd, & eventItem);

    for (size_t i = 0; i < mRequests.size(); i++) {
        const Request& request = mRequests.valueAt(i);
        struct epoll_event eventItem;
        request.initEventItem(&eventItem);
        //将request队列的事件，分别添加到epoll实例
        int epollResult = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, request.fd, & eventItem);
        if (epollResult < 0) {
            ALOGE("Error adding epoll events for fd %d while rebuilding epoll set, errno=%d", request.fd, errno);
        }
    }
}
```
需要注意Request队列，也添加到epoll的监控范围内。

2.nativeDestroy()
清理回收的调用链如下：

![](http://ww4.sinaimg.cn/large/006tNc79ly1ffn2xh6yjsj30jj0fumxg.jpg)

下面来进一步来看看调用链的过程：

【1】MessageQueue.dispose()

```
private void dispose() {
    if (mPtr != 0) {
        nativeDestroy(mPtr); 【2】
        mPtr = 0;
    }
}
```

【2】android_os_MessageQueue_nativeDestroy()

```
static void android_os_MessageQueue_nativeDestroy(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->decStrong(env); 【3】
}
```

nativeMessageQueue继承自RefBase类，所以decStrong最终调用的是RefBase.decStrong().

【3】RefBase::decStrong()

```
void RefBase::decStrong(const void* id) const
{
    weakref_impl* const refs = mRefs;
    refs->removeStrongRef(id); //移除强引用
    const int32_t c = android_atomic_dec(&refs->mStrong);
    if (c == 1) {
        refs->mBase->onLastStrongRef(id);
        if ((refs->mFlags&OBJECT_LIFETIME_MASK) == OBJECT_LIFETIME_STRONG) {
            delete this;
        }
    }
    refs->decWeak(id); // 移除弱引用
}
```

3.nativePollOnce()
nativePollOnce用于提取消息队列中的消息，提取消息的调用链，如下：
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffn30av445j30k50jp3yu.jpg)

下面来进一步来看看调用链的过程：

【1】MessageQueue.next()

```
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    for (;;) {
        ...
        nativePollOnce(ptr, nextPollTimeoutMillis); //阻塞操作 【2】
        ...
    }
```
【2】android_os_MessageQueue_nativePollOnce()

```
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj, jlong ptr, jint timeoutMillis) {
    //将Java层传递下来的mPtr转换为nativeMessageQueue
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis); 【3】
}
```

【3】NativeMessageQueue::pollOnce()

```
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis); 【4】
    mPollObj = NULL;
    mPollEnv = NULL;
    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}
```
【4】Looper::pollOnce()
==> Looper.h
```
inline int pollOnce(int timeoutMillis) {
    return pollOnce(timeoutMillis, NULL, NULL, NULL); 【5】
}
```

【5】 Looper::pollOnce()

==> Looper.cpp

```
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        // 先处理没有Callback方法的 Response事件
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) { //ident大于0，则表示没有callback, 因为POLL_CALLBACK = -2,
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;
                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }
        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }
        // 再处理内部轮询
        result = pollInner(timeoutMillis); 【6】
    }
}
```

参数说明：

+ timeoutMillis：超时时长
+ outFd：发生事件的文件描述符
+ outEvents：当前outFd上发生的事件，包含以下4类事件
  + EVENT_INPUT 可读
  + EVENT_OUTPUT 可写
  + EVENT_ERROR 错误
  + EVENT_HANGUP 中断
+ outData：上下文数据

【6】Looper::pollInner()

==> Looper.cpp

```
int Looper::pollInner(int timeoutMillis) {
    ...
    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;
    mPolling = true; //即将处于idle状态
    struct epoll_event eventItems[EPOLL_MAX_EVENTS]; //fd最大个数为16
    //等待事件发生或者超时，在nativeWake()方法，向管道写端写入字符，则该方法会返回；
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    mPolling = false; //不再处于idle状态
    mLock.lock();  //请求锁
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();  // epoll重建，直接跳转Done;
        goto Done;
    }
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = POLL_ERROR; // epoll事件个数小于0，发生错误，直接跳转Done;
        goto Done;
    }
    if (eventCount == 0) {  //epoll事件个数等于0，发生超时，直接跳转Done;
        result = POLL_TIMEOUT;
        goto Done;
    }

    //循环遍历，处理所有的事件
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken(); //已经唤醒了，则读取并清空管道数据【7】
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                //处理request，生成对应的reponse对象，push到响应数组
                pushResponse(events, mRequests.valueAt(requestIndex));
            }
        }
    }
Done: ;
    //再处理Native的Message，调用相应回调方法
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            {
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();  //释放锁
                handler->handleMessage(message);  // 处理消息事件
            }
            mLock.lock();  //请求锁
            mSendingMessage = false;
            result = POLL_CALLBACK; // 发生回调
        } else {
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }
    mLock.unlock(); //释放锁

    //处理带有Callback()方法的Response事件，执行Reponse相应的回调方法
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;
            // 处理请求的回调方法
            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq); //移除fd
            }
            response.request.callback.clear(); //清除reponse引用的回调方法
            result = POLL_CALLBACK;  // 发生回调
        }
    }
    return result;
}
```
pollOnce返回值说明：
+ POLL_WAKE： 表示由wake()触发，即pipe写端的write事件触发；
+ POLL_CALLBACK： 表示某个被监听fd被触发。
+ POLL_TIMEOUT： 表示等待超时；
+ POLL_ERROR：表示等待期间发生错误；

【7】Looper::awoken()

```
void Looper::awoken() {
    uint64_t counter;
    //不断读取管道数据，目的就是为了清空管道内容
    TEMP_FAILURE_RETRY(read(mWakeEventFd, &counter, sizeof(uint64_t)));
}
```

poll小结

pollInner()方法的处理流程：

+ 先调用epoll_wait()，这是阻塞方法，用于等待事件发生或者超时；
+ 对于epoll_wait()返回，当且仅当以下3种情况出现：
  + POLL_ERROR，发生错误，直接跳转到Done；
  + POLL_TIMEOUT，发生超时，直接跳转到Done；
  + 检测到管道有事件发生，则再根据情况做相应处理：
  + 如果是管道读端产生事件，则直接读取管道的数据；
如果是其他事件，则处理request，生成对应的reponse对象，push到reponse数组；
+ 进入Done标记位的代码段：
  + 先处理Native的Message，调用Native 的Handler来处理该Message;
  + 再处理Response数组，POLL_CALLBACK类型的事件；
从上面的流程，可以发现对于Request先收集，一并放入reponse数组，而不是马上执行。真正在Done开始执行的时候，是先处理native Message，再处理Request，说明native Message的优先级高于Request请求的优先级。

另外pollOnce()方法中，先处理Response数组中不带Callback的事件，再调用了pollInner()方法。

4.nativeWake()
nativeWake用于唤醒功能，在添加消息到消息队列enqueueMessage(), 或者把消息从消息队列中全部移除quit()，再有需要时都会调用 nativeWake方法。包含唤醒过程的添加消息的调用链，如下：

![](http://ww1.sinaimg.cn/large/006tNc79ly1ffn37ti7qhj30k80jxdg5.jpg)

下面来进一步来看看调用链的过程：

【1】MessageQueue.enqueueMessage()

==> MessageQueue.java

```
boolean enqueueMessage(Message msg, long when) {
    ... //将Message按时间顺序插入MessageQueue
    if (needWake) {
            nativeWake(mPtr); 【2】
        }
}
```
往消息队列添加Message时，需要根据mBlocked情况来决定是否需要调用nativeWake。


【2】android_os_MessageQueue_nativeWake()

==> android_os_MessageQueue.cpp

```
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake(); 【3】
}
```

【3】NativeMessageQueue::wake()

```
void NativeMessageQueue::wake() {
    mLooper->wake();  【4】
}
```

【4】Looper::wake()

```
void Looper::wake() {
    uint64_t inc = 1;
    // 向管道mWakeEventFd写入字符1
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            ALOGW("Could not write wake signal, errno=%d", errno);
        }
    }
}
```

其中TEMP_FAILURE_RETRY 是一个宏定义， 当执行write失败后，会不断重复执行，直到执行成功为止。

5.sendMessage
看下Native层如何向MessageQueue发送消息。

【1】sendMessage

```
void Looper::sendMessage(const sp<MessageHandler>& handler, const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now, handler, message);
}
```

【2】sendMessageDelayed

```
void Looper::sendMessageDelayed(nsecs_t uptimeDelay, const sp<MessageHandler>& handler,
        const Message& message) {
    nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    sendMessageAtTime(now + uptimeDelay, handler, message);
}
```

sendMessage(),sendMessageDelayed() 都是调用sendMessageAtTime()来完成消息插入。

【3】sendMessageAtTime

```
void Looper::sendMessageAtTime(nsecs_t uptime, const sp<MessageHandler>& handler,
        const Message& message) {
    size_t i = 0;
    { //请求锁
        AutoMutex _l(mLock);
        size_t messageCount = mMessageEnvelopes.size();
        //找到message应该插入的位置i
        while (i < messageCount && uptime >= mMessageEnvelopes.itemAt(i).uptime) {
            i += 1;
        }
        MessageEnvelope messageEnvelope(uptime, handler, message);
        mMessageEnvelopes.insertAt(messageEnvelope, i, 1);
        //如果当前正在发送消息，那么不再调用wake()，直接返回。
        if (mSendingMessage) {
            return;
        }
    } //释放锁
    //当把消息加入到消息队列的头部时，需要唤醒poll循环。
    if (i == 0) {
        wake();
    }
}
```

nativeInit()方法，最终实现由epoll机制中的epoll_create()/epoll_ctl()完成；
nativeDestroy()方法，最终实现由RefBase::decStrong()完成；
nativePollOnce()方法，最终实现由Looper::pollOnce()完成；
nativeWake()方法，最终实现由Looper::wake()调用write方法，向管道写入字符；

Native结构体和类
Looper.h/ Looper.cpp文件中，定义了Message结构体，消息处理类，回调类，Looper类。

Message结构体

```
struct Message {
    Message() : what(0) { }
    Message(int what) : what(what) { }
    int what; // 消息类型
};
```

消息处理类
MessageHandler类

```
class MessageHandler : public virtual RefBase {
protected:
    virtual ~MessageHandler() { }
public:
    virtual void handleMessage(const Message& message) = 0;
};
```

WeakMessageHandler类，继承于MessageHandler类

```
class WeakMessageHandler : public MessageHandler {
protected:
    virtual ~WeakMessageHandler();
public:
    WeakMessageHandler(const wp<MessageHandler>& handler);
    virtual void handleMessage(const Message& message);
private:
    wp<MessageHandler> mHandler;
};

void WeakMessageHandler::handleMessage(const Message& message) {
    sp<MessageHandler> handler = mHandler.promote();
    if (handler != NULL) {
        handler->handleMessage(message); //调用MessageHandler类的处理方法()
    }
}
```

回调类
LooperCallback类

```
class LooperCallback : public virtual RefBase {
protected:
    virtual ~LooperCallback() { }
public:
    //用于处理指定的文件描述符的poll事件
    virtual int handleEvent(int fd, int events, void* data) = 0;
};
```

SimpleLooperCallback类， 继承于LooperCallback类

```
class SimpleLooperCallback : public LooperCallback {
protected:
    virtual ~SimpleLooperCallback();
public:
    SimpleLooperCallback(Looper_callbackFunc callback);
    virtual int handleEvent(int fd, int events, void* data);
private:
    Looper_callbackFunc mCallback;
};

int SimpleLooperCallback::handleEvent(int fd, int events, void* data) {
    return mCallback(fd, events, data); //调用回调方法
}
```

Looper类

```
static const int EPOLL_SIZE_HINT = 8; //每个epoll实例默认的文件描述符个数
static const int EPOLL_MAX_EVENTS = 16; //轮询事件的文件描述符的个数上限
```

其中Looper类的内部定义了Request，Response，MessageEnvelope这3个结构体，关系图如下：

![](http://ww4.sinaimg.cn/large/006tNc79ly1ffn3gq1iovj30it09pzkw.jpg)

```
struct Request { //请求结构体
    int fd;
    int ident;
    int events;
    int seq;
    sp<LooperCallback> callback;
    void* data;
    void initEventItem(struct epoll_event* eventItem) const;
};

struct Response { //响应结构体
    int events;
    Request request;
};

struct MessageEnvelope { //信封结构体
    MessageEnvelope() : uptime(0) { }
    MessageEnvelope(nsecs_t uptime, const sp<MessageHandler> handler,
            const Message& message) : uptime(uptime), handler(handler), message(message) {
    }
    nsecs_t uptime;
    sp<MessageHandler> handler;
    Message message;
};
```

MessageEnvelope正如其名字，信封。MessageEnvelope里面记录着收信人(handler)，发信时间(uptime)，信件内容(message).

MessageQueue通过mPtr变量保存NativeMessageQueue对象，从而使得MessageQueue成为Java层和Native层的枢纽，既能处理上层消息，也能处理native层消息；下面列举Java层与Native层的对应图.

![](http://ww3.sinaimg.cn/large/006tNc79ly1ffn3if4mmij30qb0bq0t1.jpg)

+ 红色虚线关系：Java层和Native层的MessageQueue通过JNI建立关联，彼此之间能相互调用，搞明白这个互调关系，也就搞明白了Java如何调用C++代码，C++代码又是如何调用Java代码。
+ 蓝色虚线关系：Handler/Looper/Message这三大类Java层与Native层并没有任何的真正关联，只是分别在Java层和Native层的handler消息模型中具有相似的功能。都是彼此独立的，各自实现相应的逻辑。
+ WeakMessageHandler继承于MessageHandler类，NativeMessageQueue继承于MessageQueue类
+ 另外，消息处理流程是先处理Native Message，再处理Native Request，最后处理Java Message。理解了该流程，也就明白有时上层消息很少，但响应时间却较长的真正原因。






