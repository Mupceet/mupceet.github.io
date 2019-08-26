---
title: 遍历数组删除某元素的方法
date: 2017-05-13 23:05:33
categories:
- 技术向
tags:
- Java
---

从数组中删除元素是经常需要用到的情况，可能根据经验你知道要从后往前删除，但是你知道具体的原因吗？本文通过简单的解析让你知其所以然。

假设一个需求，从数组
`["a", "bb", "bb", "ccc", "ccc", "ccc", "ccc"]` 中删除"bb"元素，即一个数组需要遍历其中的元素，当该元素符合某个条件的时候从数组中将该元素中删除。

## 错误写法
新手可能会直接写出使用迭代器的以下代码：

**写法一:**
``` java
public static void remove(ArrayList<String> list) {
    for (String s : list) {
        if (s.equals("bb")) {
            list.remove(s);
        }
    }
}
```
<!--more-->

实际上，这段代码运行时会抛出 `ConcurrentModificationException` 异常：
``` terminal
java.util.ConcurrentModificationException
        at java.util.ArrayList$Itr.checkForComodification(Unknown Source)
        at java.util.ArrayList$Itr.next(Unknown Source)
        at ArrayListRemove.remove(ArrayListRemove.java:22)
        at ArrayListRemove.main(ArrayListRemove.java:14)
```

我们暂时先不管它，换成普通的遍历的写法：

**写法二：**
```java
public static void remove(ArrayList<String> list) {
    for (int i = 0; i < list.size(); i++) {
        String s = list.get(i);
        if (s.equals("bb")) {
            list.remove(s);
        }
    }
}
```

这样子写运行时不报错了，但是执行完之后数组打印结果如下：
``` terminal
element : a
element : bb
element : ccc
element : ccc
element : ccc
```

可以发现并没有把所有的 “bb” 删除掉。

## 源码解析
我们看看这两种写法是怎样出错的。

首先看看方法二为什么运行结果出错，通过查看 ArrayList 的 remove 方法一探究竟。

```java
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                // 删除第一个匹配的元素
                fastRemove(index);
                return true;
            }
    }

    return false;
}
```

可以看到删除元素时只删除了第一个匹配到的元素。再查看具体的 `fastRemove()` 方法：
```java
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // Let gc do its work
}
```

可以看到，删除时实际上是元素的移动。写法二中，从前往后遍历，index 遍历到第一个 “bb”，删除时即把从第二个 “bb” 及之后的元素拷贝到当前指向的位置，也就是第二个 “bb” 移动到了第一个 “bb” 的位置上，从而“删除”了第一个 “bb”。接着，index 就跳过了当前位置，所以，第二个 “bb” 就被跳过了，也就不会被删除了。

针对写法二这种会引起错误结果的写法，可以通过倒序遍历的方式解决。

再回头来看写法一，发生了 `ConcurrentModificationException`，这是因为迭代器内部维护了索引位置相关的数据，它要求在迭代过程中，容器不能发生结构性变化，所谓结构性变化就是 **添加**、**插入** 和 **删除** 元素，而修改元素内容不算结构性变化。要避免该异常，就需要使用迭代器的 remove 方法。

迭代器怎么知道发生了结构性变化，并抛出异常呢？它自己的 remove 方法为何又可以使用呢？我们需要看下迭代器的工作原理。
```java
public Iterator<E> iterator() {
        return new Itr();
    }
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
        //省略……
    }
```

我们来看下 ArrayList 中 iterator 方法的实现，代码为：
```java
public Iterator<E> iterator() {
    return new Itr();
}
```

新建了一个 Itr 对象，而 Itr 是一个成员内部类，实现了 Iterator 接口，它有三个实例成员变量，为：
```java
int cursor;       // index of next element to return
int lastRet = -1; // index of last element returned; -1 if no such
int expectedModCount = modCount;
```
cursor 表示下一个要返回的元素位置，lastRet 表示最后一个返回的索引位置，expectedModCount 表示期望的修改次数，初始化为外部类当前的修改次数 modCount。

每次发生结构性变化的时候 modCount 都会增加，而每次迭代器操作的时候都会检查 expectedModCount 是否与 modCount 相同，这样就能检测出结构性变化。

```Java
if (modCount != expectedModCount)
    throw new ConcurrentModificationException();
```

而正确使用 `iterator.remove()` 方法却不会引发异常，查看源码得知：
```Java
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
```

可以看到 remove 调用的虽然也是 ArrayList 的 remove 方法，但它同时更新了 cursor, lastRet 和 expectedModCount 的值，所以它可以正确删除而不引发异常。

从代码中注意到，调用 remove 之前需要 lastRet，所以调用 `remove()` 方法前必须先调用 `next()` 来更新 lastRet。

通过以上查看源码分析，写法一、二这两种错误写法做出相应的修正，可以得到正确写法。

## 正确写法

**写法三：倒序遍历**
```java
public static void remove(ArrayList<String> list) {
    // 这里要注意数组越界的问题，要用 >= 0 来界定
    for (int i = list.size() - 1; i >= 0; i--) {
        String s = list.get(i);
        if (s.equals("bb")) {
            list.remove(s);
        }
    }
}
```

**写法四：**
```java
public static void remove(ArrayList<String> list) {
    Iterator<String> it = list.iterator();
    while (it.hasNext()) {
        String s = it.next();
        if (s.equals("bb")) {
            it.remove();
        }
    }
}
```

## 引申及简化
在这里，做一个引申，对于数组来说，可以使用一个更加简单的写法。也就是如果知道要删除的元素是什么就可以使用 ArrayList 对象的方法：`remove` 和 `removeAll`。
```java
public boolean remove(Object o);
public boolean removeAll(Collection<?> c);
```

可以看到，remove 方法可以传一个对象进去，但它和正序遍历一样只会删除第一个匹配到的元素，而 removeAll 方法可以删除所有匹配的元素，但是传入的需要一个容器类对象。

所以说要删除所有的 “bb” 元素，那么就应该这样子写。

**写法五：**
```java
public static void remove(ArrayList<String> list) {
    // 构造一个 Collection
    ArrayList<String> listTmp = new ArrayList<String>();
    listTmp.add("bb");
    list.removeAll(listTmp);
}
```
可以看到这样子需要单独构造一个 Collection 的写法是很不优雅的，还好，Collections 类给我们提供了一个静态方法 `singleton()`：
```java
public static <E> Set<E> singleton(E o);
```

它可以将一个普通对象转换成一个容器对象，所以可以改写成如下代码：

**写法六：**
```java
public static void remove(ArrayList<String> list) {
    list.removeAll(Collections.singleton("bb"));
}
```
容器类的内容实在是太多了，可以多多查看源码以及《Thinking in Java》容器相关内容。
