（如有错误或补充，望指出）

Handler 的作用是将一个任务切换到Handler所在的线程去执行。

#### ThreadLocal：
ThreadLocal并不是线程，它的作用是在每个线程中存储并提供数据，并Handler内部可以通过它来获得当前线程的Looper。
ThreadLocal是一个线程内部的数据存储类，可以在指定线程中存储数据。数据存储后，只有在指定线程中可以获取到存储的数据，其他线程则无法获取。
例子：
```
private ThreadLocal<Boolean> mBooleanThreadLocal = new ThreadLocal<Boolean>();
	mBooleanThreadLocal .set(true);
	Log.d(TAG,"[Thread#Main]mBooleanThreadLocal = "+mBooleanThreadLocal.get());

new Thread("Thread#1"){
	public void run(){
		mBooleanThreadLocal .set(fasle);
		Log.d(TAG,"[Thread#1]mBooleanThreadLocal = "+mBooleanThreadLocal.get());
	}
}

new Thread("Thread#2"){
	public void run(){
		Log.d(TAG,"[Thread#2]mBooleanThreadLocal = "+mBooleanThreadLocal.get());
	}
}
```
结果：
```
[Thread#Main]mBooleanThreadLocal = true
[Thread#1]mBooleanThreadLocal = fasle
[Thread#2]mBooleanThreadLocal = null
```
#### Looper

对于looper，需要关注这几个方法：构造方法，prepare()，loop()。

构造方法：
```
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
在构造方法里面，做了两件事，
一是创建一个消息队列，二是将线程对象指向了创建Looper的线程。

prepare()：
```
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
首先判断当前线程有没有存在Looper，如果存在，则抛出异常，不存在则新建一个Looper并存储在ThreadLocal中。（这就解析了为什么一个线程只有一个Looper）

loop()：

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
        if (traceTag != 0) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
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
loop() 方法的实现步骤：

1、首先通过 final Looper me = myLooper(); 和 final MessageQueue queue = me.mQueue; 获取当前线程绑定的 Looper 和与 Looper 绑定的消息队列。

2、然后开启一个for的无限循环，在循环里面：

   （1）通过 queue.next(); 不断指向消息队列下一个节点（也就是不断遍历消息队列）。
   
   （2）当找到消息后，调用 msg.target.dispatchMessage(msg); 来分发消息。
   
   （3）最后通过 msg.recycleUnchecked(); 来将消息标记为正在使用。而 msg.target 追踪一下会发现其实就是 Handler 。
   
#### Handler

对于Handler，需要关心的方法有 sendMessage，和 handleMessage 方法，Handler还可以通过 post 一个 Runnable，里面其实也是调用了 sendMessage 方法。

sendMessage：

所有的发送消息方法最终都会调用 sendMessageAtTime 方法。
```
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
在 sendMessageAtTime 方法中，可以看到，它最终的操作就是往消息队列里面插入一条消息。（也就是说，发消息的操作其实就是往消息队列里面插入一条消息）

然后值得看下的是 enqueueMessage 方法：
```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

msg.target = this; 也就是说，上面说的 Looper 里面，分发消息的  msg.target.dispatchMessage(msg); 中的 msg.target 就是从这里赋值的。

所以，现在看回 dispatchMessage 方法：
 
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

可以看到，分发消息方法里面，最终会回调 handleCallback(msg); 或者 handleMessage(msg); 方法。

#### MessageQueue

消息队列内部存储了一组消息，以队列的形式对外提供插入和删除的工作。采用的是单链表的数据结构来存储消息。
主要包含两个操作：插入和读取。读取本身会伴随着删除操作。对应的方法是 enqueueMessage 和 next

#### 总结

Handler通过Looper来构建内部的消息循环系统，当send方法被调用时，它会调用MessageQueue的enqueueMessage方法将这个消息放入消息队列中，然后Looper发现新消息来后，处理消息。最终消息中的Runnable或者Handler中的handleMessage方法会被调用。

因为Looper是运行在创建Handler的线程中的，这样Handler的业务逻辑就被切换到创建Handler所在的线程去执行了。
