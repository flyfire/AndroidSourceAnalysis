先看下ThreadLocal实现原理
+ 每个线程都有 一个 ThreadLocalMap 类型的 threadLocals 属性。
+ ThreadLocalMap 类相当于一个Map，key 是 ThreadLocal 本身，value 就是我们的值。
+ 当我们通过 threadLocal.set(new Integer(123)); ，我们就会在这个线程中的 threadLocals 属性中放入一个键值对，key 是 这个 threadLocal.set(new Integer(123)); 的 threadlocal，value 就是值。
+ 当我们通过 threadlocal.get() 方法的时候，首先会根据这个线程得到这个线程的 threadLocals 属性，然后由于这个属性放的是键值对，我们就可以根据键 threadlocal 拿到值。 注意，这时候这个键 threadlocal 和 我们 set 方法的时候的那个键 threadlocal 是一样的，所以我们能够拿到相同的值。

InheritableThreadLocal 概念
从上面的介绍我们可以知道，我们其实是根据 Thread.currentThread()，拿到该线程的 threadlocals，从而进一步得到我们之前预先 set 好的值。那么如果我们新开一个线程，这个时候，由于 Thread.currentThread() 已经变了，从而导致获得的 threadlocals 不一样，我们之前并没有在这个新的线程的 threadlocals 中放入值，那么我就再通过 threadlocal.get()方法 是不可能拿到值的。例如如下代码：

```
public class Test {

    public static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>();

    public static void main(String args[]){
        threadLocal.set(new Integer(123));

        Thread thread = new MyThread();
        thread.start();

        System.out.println("main = " + threadLocal.get());
    }

    static class MyThread extends Thread{
        @Override
        public void run(){
            System.out.println("MyThread = " + threadLocal.get());
        }
    }
}
```

输出是

```
main = 123
MyThread = null
```
那么这个时候怎么解决？ **InheritableThreadLocal** 就可以解决这个问题。先看一个官方对它的介绍：

```
* This class extends <tt>ThreadLocal</tt> to provide inheritance of values
 * from parent thread to child thread: when a child thread is created, the
 * child receives initial values for all inheritable thread-local variables
 * for which the parent has values.  Normally the child's values will be
 * identical to the parent's; however, the child's value can be made an
 * arbitrary function of the parent's by overriding the <tt>childValue</tt>
 * method in this class.
```

也就是说，我们把上面的``public static ThreadLocal<Integer> threadLocal = new ThreadLocal<Integer>();``改成``public static ThreadLocal<Integer> threadLocal = new InheritableThreadLocal<Integer>();``再运行，就会有结果：

```
main = 123
MyThread = 123
```

也就是子线程或者说新开的线程拿到了该值。 那么，这个究竟是怎么实现的呢，key 都变了，为什么还可以拿到呢？

InheritableThreadLocal 原理
我们可以首先可以浏览下 InheritableThreadLocal 类中有什么东西：
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffmz3u6hdmj31gy11q7dj.jpg)

其实就是重写了3个方法。

首先，当我们调用 get 方法的时候，由于子类没有重写，所以我们调用了父类的 get 方法：

```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
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
这里会有一个``Thread.currentThread()`` ， ``getMap(t)`` 方法，所以就会得到这个线程 ``threadlocals``。 但是，由于子类 ``InheritableThreadLocal`` 重写了 ``getMap()``方法，再看上述代码，我们可以看到：
**其实不是得到 threadlocals，而是得到 inheritableThreadLocals**。 inheritableThreadLocals 之前一直没提及过，其实它也是 Thread 类的一个 ThreadLocalMap 类型的 属性，如下 Thread 类的部分代码：

```
ThreadLocal.ThreadLocalMap threadLocals = null;
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

那么，这里看 InheritableThreadLocal 重写的方法，感觉 inheritableThreadLocals 和 threadLocals 几乎是一模一样的作用，只是换了个名字而且，那么究竟 为什么在新的 线程中 通过 threadlocal.get() 方法还能得到值呢？

这时候要注意 childValue 方法，我们可以看下它的官方说明：

```
 * Computes the child's initial value for this inheritable thread-local
 * variable as a function of the parent's value at the time the child
 * thread is created.  This method is called from within the parent
 * thread before the child is started.
```

这个时候，你明白了，是不是在 创建线程的时候做了手脚，做了一些值的传递，或者这里利用上了 inheritableThreadLocals 之类的。

其实，是的：**关键在于 Thread thread = new MyThread();**

这不是一个简简单单的 new 操作。当我们 new 一个 线程的时候：

```
public Thread() {
    init(null, null, "Thread-" + nextThreadNum(), 0);
}
```
然后
```
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize) {
    init(g, target, name, stackSize, null);
}
```
再然后
```
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc) {
     ......
    if (parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;
    ......
    }
```

这时候有一句 ``'ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);' ``，然后

```
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
```

继续跟踪：

```
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

当我们创建一个新的线程的时候X，X线程就会有 ThreadLocalMap 类型的 inheritableThreadLocals ，因为它是 Thread 类的一个属性。

然后

先得到当前线程存储的这些值，例如 Entry[] parentTable = parentMap.table; 。再通过一个 for 循环，不断的把当前线程的这些值复制到我们新创建的线程X 的inheritableThreadLocals 中。就这样，就ok了。

那么这样会有一个什么结果呢？

结果就是我们创建的新线程X 的inheritableThreadLocals 变量中已经有了值了。那么我在新的线程X中调用 threadlocal.get() 方法，首先会得到新线程X 的 inheritableThreadLocals，然后，再根据threadlocal.get()中的 threadlocal，就能够得到这个值。

这样就避免了 新线程中得到的 threadlocals 没有东西。之前就是因为没有东西，所以才拿不到值。

所以说 整个 InheritableThreadLocal 的实现原理就是这样的。

总结

+ 首先要理解 为什么 在 新线程中得不到值，是因为**我们其实是根据 Thread.currentThread()，拿到该线程的 threadlocals，从而进一步得到我们之前预先 set 好的值。那么如果我们新开一个线程，这个时候，由于 Thread.currentThread() 已经变了，从而导致获得的 threadlocals 不一样，我们之前并没有在这个新的线程的 threadlocals 中放入值，那么我就再通过 threadlocal.get()方法 是不可能拿到值的。**
+ 那么解决办法就是 我们在新线程中，要把父线程的 threadlocals 的值 给复制到 新线程中的 threadlocals 中来。这样，我们在新线程中得到的 threadlocals 才会有东西，再通过 threadlocal.get() 中的 threadlocal，就会得到值。
