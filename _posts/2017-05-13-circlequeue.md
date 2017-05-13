---
layout: post
title: 循环队列
date:   2017-05-13 10:54:37
categories: Java
---

# Java实现的循环队列
`队列`具有先进先出的特点，本文所说的`循环队列`（以下简称队列）的长度是一定的（可以扩容），如果队列满了，新来的元素会覆盖队列中最先来的元素。

它的使用场景是对内存占用有要求的情况下按序缓存一些数据，比如直播中的弹幕。

* 注意：此实现非线程安全
 

```java
import java.util.ArrayList;
import java.util.List;

/**
 * 容量受限的队列，在队列已满的情况下，新入队的元素会覆盖最老的元素。<br>
 * <font color="red">此类非线程安全</font>
 * 
 * @author rongzhisheng
 *
 * @param <E>
 */
public class CircleQueue<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private int capacity;
    private int head;// 即将出队的位置
    private int tail;// 即将入队的位置
    private List<E> queue;
    private boolean isFull;// 队列是否已满，队列空和满的时候head==tail

    public CircleQueue() {
        this(DEFAULT_CAPACITY);
    }

    public CircleQueue(int capacity) {
        this.capacity = capacity;
        queue = new ArrayList<>(this.capacity);
        head = tail = 0;
    }

    /**
     * 入队
     * 
     * @param e
     * @return 是否丢弃了最旧的元素
     */
    public boolean add(E e) {
        if (isFull) {
            head = addIndex(head);
            addOrSet(e, tail);
            tail = head;
            return true;
        }
        addOrSet(e, tail);
        tail = addIndex(tail);
        if (head == tail) {
            isFull = true;
        }
        return false;
    }

    public E poll() {
        if (!isFull && head == tail) {
            return null;
        }
        E e = queue.get(head);
        queue.set(head, null);
        head = addIndex(head);
        isFull = false;
        return e;
    }

    public int size() {
        if (!isFull && head == tail) {
            return 0;
        }
        if (isFull) {
            return capacity;
        }
        int i = 0;
        int h = head;
        while (h != tail) {
            i++;
            h = addIndex(h);
        }
        return i;
    }

    public int capacity() {
        return capacity;
    }

    public E element(int index) {
        if (index < 0 || index >= size()) {
            throw new IllegalArgumentException();
        }
        index = addIndex(head + index - 1);
        return queue.get(index);
    }

    public boolean isEmpty() {
        return size() == 0;
    }

    public List<E> getList() {
        List<E> list = new ArrayList<>();
        final int size = size();
        for (int i = 0; i < size; i++) {
            list.add(element(i));
        }
        return list;
    }

    public void clean() {
        head = tail = 0;
        isFull = false;
        queue.clear();
    }

    public void grow(int newCapacity) {
        if (newCapacity <= capacity) {
            return;
        }
        List<E> data = getList();
        queue.clear();
        queue.addAll(data);
        head = 0;
        tail = head + queue.size();
        capacity = newCapacity;
        isFull = false;
    }

    @Override
    public String toString() {
        return getList().toString();
    }

    private void addOrSet(E e, int index) {
        if (queue.size() > index) {
            queue.set(index, e);
            return;
        }
        if (queue.size() == index) {
            queue.add(e);
            return;
        }
        throw new IllegalStateException();
    }

    private int addIndex(int i) {
        return (i + 1) % capacity;
    }

    public static void main(String[] args) {
        int a = 1;
        CircleQueue<Integer> queue = new CircleQueue<Integer>(5);

        queue.grow(8);
        queue.grow(10);

        for (int i = a; i < a + 6; i++) {
            queue.add(i);
        }
        a += 6;
        System.out.println(queue);

        for (int i = a; i < a + 6; i++) {
            queue.add(i);
        }
        a += 6;
        System.out.println(queue);

        queue.grow(10);

        for (int i = a; i < a + 6; i++) {
            queue.add(i);
        }
        a += 6;
        System.out.println(queue);

        for (int i = 0; i < 5; i++) {
            System.out.println(queue.poll());
        }
        System.out.println(queue);

        queue.clean();

        for (int i = 0; i < 3; i++) {
            queue.add(i);
        }
        System.out.println(queue);
    }

}
```