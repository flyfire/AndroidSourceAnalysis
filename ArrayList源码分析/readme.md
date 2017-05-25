![](https://ws2.sinaimg.cn/large/006tKfTcly1ffxxson5hnj31ay02at9i.jpg)
+ ArrayList基于数组实现，能自动增长。
+ 非线程安全，多线程可以选择Vector或者CopyOnWriteArrayList
+ 实现了RandomAccess接口，遍历时for loop比Iterator方式快
+ 实现了Clone接口，可以被克隆
+ 实现了Serialization接口，可以被序列化。

构造函数初始化

![](https://ws3.sinaimg.cn/large/006tNc79ly1ffxyf2vbqwj31jc138wnr.jpg)

![](https://ws4.sinaimg.cn/large/006tNc79ly1ffxyfzrv7lj31cy0h2aeu.jpg)

**elementData:**保存了添加到 ArrayList 中的元素。实际上，elementData 是个动态数组，我们能通过构造函数 ArrayList(int initialCapacity) 来执行它的初始容量为 initialCapacity；

**size:** 动态数组的实际大小。

![](https://ws3.sinaimg.cn/large/006tNc79ly1ffxylhxmbtj31720zq480.jpg)
![](https://ws1.sinaimg.cn/large/006tNc79ly1ffxymx5lqxj31du0kmn29.jpg)

使用无参构造函数时，容量会自动增长到``DEFAULT_CAPACITY``，即10。后续如果容量不足，会增长为当前容量的1.5倍。``int newCapacity = oldCapacity + (oldCapacity >> 1);``

[--Arrays.copyOf]
![](https://ws2.sinaimg.cn/large/006tNc79ly1ffxyuytcg5j31kw0vbk3p.jpg)
![](https://ws4.sinaimg.cn/large/006tNc79ly1ffxywb4eh5j31cw0ou10m.jpg)

容量增加后，会将原来的元素拷贝到新的数组中，并将``elementData``指向新的数组。

增
![](https://ws3.sinaimg.cn/large/006tNc79ly1ffxyyotoq4j319m0xsqbn.jpg)
增加容量，将元素插入数组尾部。如果插入数组中间，则将数组中间之后的元素拷贝到位移一位的位置。
删
![](https://ws1.sinaimg.cn/large/006tNc79ly1ffxz1hmvw8j31bu0q244z.jpg)
![](https://ws4.sinaimg.cn/large/006tNc79ly1ffxz47is7oj31ek0w0ahr.jpg)
![](https://ws1.sinaimg.cn/large/006tNc79ly1ffxz4ftj9yj317q0e40vw.jpg)
删除元素是将元素后面的拷贝到往前位移一位的位置，并将末尾元素置为null等待回收。
