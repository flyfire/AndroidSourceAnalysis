ThreadLocal主要是实现线程本地存储的，也就是说不同线程访问ThreadLocal的set(T value)方法，会在线程本地保存一份value，对其他线程不可见。那他是如何实现多个线程每个线程一个变量副本的呢？这个value保存在哪里呢？

ThreadLocal提供了set和get访问器用来访问与当前线程相关联的线程局部变量。

先来看看set方法。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffmx8009t8j31b60io0xc.jpg)

可以看到初次调用的时候会调用``createMap(thread, value)``方法。
![](http://ww4.sinaimg.cn/large/006tNc79ly1ffmx9gde8sj31860c8gop.jpg)
![](http://ww3.sinaimg.cn/large/006tNc79ly1ffmxaf773rj31be0403zj.jpg)
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffmxbe8eigj31fa0dqadx.jpg)
可以看到在``createMap``中将线程的``threadLocals``变量实例化为一个类似HashMap的存储容器中，将以``ThreadLocal``为key,``firstvalue``为value的Entry插入到以``ThreadLocal``的``threadLocalHashCode``和Entry数组初始大小``INITIAL_CAPACITY``得到的下标Entry数组中。这样就实现了每个线程访问set方法时，会先去初始化各个线程的``threadLocals``变量，然后将``ThreadLocal``和``firstvalue``存储到``threadLocals``中。从代码中可以看到，一个thread可以有多个``ThreadLocal``和``value``的键值对。

存的过程了解了，再来看看取得过程以及初始值的相关细节。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffmxn7s8rxj318y0k20xh.jpg)
![](http://ww2.sinaimg.cn/large/006tNc79ly1ffmxo90mk7j314e0bo40i.jpg)
可以看到是从``Thread``的``threadLocals``这个``ThreadLocalMap``中取出以``ThreadLocal``为key的value。
![](http://ww1.sinaimg.cn/large/006tNc79ly1ffmyiev6cfj315w0hqadl.jpg)
可以看到当``ThreadLocalMap``也就是``Thread.threadLocals``为空，没有``set``时会调用``setInitialValue``将``initialValue``中的值取出并存储到``ThreadLocalMap``中去。

总结一下：

+ 每个线程都有 一个 ThreadLocalMap 类型的 threadLocals 属性。
+ ThreadLocalMap 类相当于一个Map，key 是 ThreadLocal 本身，value 就是我们的值。
+ 当我们通过 threadLocal.set(new Integer(123)); ，我们就会在这个线程中的 threadLocals 属性中放入一个键值对，key 是 这个 threadLocal.set(new Integer(123)); 的 threadlocal，value 就是值。
+ 当我们通过 threadlocal.get() 方法的时候，首先会根据这个线程得到这个线程的 threadLocals 属性，然后由于这个属性放的是键值对，我们就可以根据键 threadlocal 拿到值。 注意，这时候这个键 threadlocal 和 我们 set 方法的时候的那个键 threadlocal 是一样的，所以我们能够拿到相同的值。



