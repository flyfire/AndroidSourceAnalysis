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
改
![](https://ws1.sinaimg.cn/large/006tNbRwly1ffxze7v9vwj317o0iywjc.jpg)
查
![](https://ws4.sinaimg.cn/large/006tNbRwly1ffxzf4uys3j316o0eewhw.jpg)

清空
![](https://ws1.sinaimg.cn/large/006tNbRwly1ffxzgxi5mej313y0eo40m.jpg)
克隆
![](https://ws2.sinaimg.cn/large/006tNbRwly1ffxzhlezb2j319i0ie0x1.jpg)

+ ArrayList 实际上是通过一个数组去保存数据的。当我们使用无参构造函数构造 ArrayList 时，则 ArrayList 的默认容量大小是10。

+ 当ArrayList容量不足以容纳全部元素时，ArrayList会重新设置容量：新的容量=“(原始容量x3)/2 + 1”;如果设置后的新容量还不够，则直接把新容量设置为传入的参数。

+ ArrayList 查找效率高，插入删除元素的效率低。

+ ArrayList 的克隆函数，即是将全部元素克隆到一个数组中。

+ ArrayList 实现 java.io.Serializable 的方式。当写入到输出流时，先写入“容量”，再依次写入“每一个元素”；当读出输入流时，先读取“容量”，再依次读取“每一个元素”。

``toArray``

![](https://ws1.sinaimg.cn/large/006tNbRwly1ffxzjp203hj31ae0j0dk4.jpg)
![](https://ws4.sinaimg.cn/large/006tNbRwly1ffxzk4pgoyj31cm108499.jpg)

toArray() 会抛出“java.lang.ClassCastException”异常原因: toArray()返回的是 Object[] 数组，Java不支持向下转型。(例如，将Object[]转换为的Integer[])
解决方案

```
public static Integer[] vectorToArray2(ArrayList<Integer> v) {
	Integer[] newText = (Integer[])v.toArray(new Integer[0]);
	return newText;
}
```

迭代器实现

```
	/**
     * An optimized version of AbstractList.Itr
     */
    private class Itr implements Iterator<E> {
        // The "limit" of this iterator. This is the size of the list at the time the
        // iterator was created. Adding & removing elements will invalidate the iteration
        // anyway (and cause next() to throw) so saving this value will guarantee that the
        // value of hasNext() remains stable and won't flap between true and false when elements
        // are added and removed from the list.
        protected int limit = ArrayList.this.size;

        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        public boolean hasNext() {
            return cursor < limit;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            int i = cursor;
            if (i >= limit)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
                limit--;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;

            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```

```
	/**
     * An optimized version of AbstractList.ListItr
     */
    private class ListItr extends Itr implements ListIterator<E> {
        ListItr(int index) {
            super();
            cursor = index;
        }

        public boolean hasPrevious() {
            return cursor != 0;
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor - 1;
        }

        @SuppressWarnings("unchecked")
        public E previous() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            int i = cursor - 1;
            if (i < 0)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i;
            return (E) elementData[lastRet = i];
        }

        public void set(E e) {
            if (lastRet < 0)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                ArrayList.this.set(lastRet, e);
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        public void add(E e) {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();

            try {
                int i = cursor;
                ArrayList.this.add(i, e);
                cursor = i + 1;
                lastRet = -1;
                expectedModCount = modCount;
                limit++;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    }
```

