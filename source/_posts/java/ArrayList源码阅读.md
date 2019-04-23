---
title: ArrayList源码阅读
date: 2019/04/23 16:00:00  # 文章发表时间
toc: true
tags:
- Java
categories:
- Java
- 源码阅读
thumbnail: http://pic.jingl.wang/2019-04-23-timg%20-1-.jpeg
---
ArrayList是List接口的一个可伸缩数组实现，实现了所有List接口的方法，用来存储一组数据，允许所有类型的对象，包括NULL，它是非线程安全的。ArrayList底层是一个可以伸缩的数组，List中的每个元素依次存放在这个数组中，并使用成员变量size来标记List中元素的个数，当插入新元素时，新元素会被放入到数组中size的位置，并将size+1，当删除一个元素时，例如删除下标为n的元素，则将n+1之后的元素向前移动一位，并将size-1。数组是可伸缩的，每次插入时,会检查当前数组是否可以容纳新元素，如果不能则会对其扩容，将数组长度变为原来的1.5倍。除了被动的改变数组长度，ArrayList也提供了2个主动改变数组的方法：
<!--more-->

```java
public void trimToSize()
public void ensureCapacity(int minCapacity)
```

`trimToSize`用于将数组大小缩短为size，减少数组所占用的空间，`ensureCapacity`用来增加数组长度，以确保数组长度大于minCapacity。

`size`、`isEmpty`、`get`、`set`、`iterator`因为是对数组直接操作，所以时间复杂度是O（1）的。

ArrayList平凡的扩容可能会影响插入的性能，所以为了减少ArrayList扩容的次数，可以在初始化阶段，提前指定一个初始化容量。

ArrayList的迭代也是遵循fail-fast机制的，当在迭代过程中，发生结构修改时，会抛出`ConcurrentModificationException`异常，这个检查是通过`modCount`实现的。

## 构造方法

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}	

public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

ArrayList提供了3个构造方法，默认的无参构造函数只是将数组大小赋值为一个空数组，在之后的插入过程中在进行扩容操作，`ArrayList(int initialCapacity)`方法指定了一个初始化容量大小，构造方法通过指定这个初始化容量新建一个指定大小的数组，当ArrayList元素较多时，可以使用这个构造方法，并传入一个较大的initialCapacity。

## add

```java
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

ArrayList的插入操作一共有2步操作，第一步就是判断当前数组大小是否足够容纳新元素的插入，如果大小不够，则进行扩容，这一步在`ensureCapacityInternal`中进行，此外在这个方法中，还会对`modCount`进行自增操作，第二步就可以将新元素放到数组对应的位置了。

### 扩容

当ArrayList中数组长度不够或者调用`ensureCapacity`方法时，会触发扩容，ArrayList的扩容逻辑非常的简单：

```java
	private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        //新数组长度为原来的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;	//如果新的长度小于指定值，则使用minCapacity
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        //复制
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

默认将会把数组长度扩大为原来的1.5倍。然后从老数组中将所有数据复制到新数组中，替换老数组。

## remove

ArrayList提供了2个remove方法：

```java
 	public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);	//指定下标的元素

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);	//将下标大于index的元素向前移动
        
        //将数组中最后的元素复制为null 防止内存泄漏
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }	

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
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    
   
```

ArrayList提供了2个remove方法，第一个用来移除指定下标的元素，第二个用来移除指定对象。

## get

没啥好讲，直接从数组拿数据。。。



