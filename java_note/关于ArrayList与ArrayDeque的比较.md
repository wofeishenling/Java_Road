### 前言

今天在写题目剑指Offer34_二叉树中和为某一值的路径的时候，其中会使用到一个专门的路径变量path，来进行路径加入和删除（回溯）

但是一直以来就是 ArrayList 和 ArrayDeque 进行混用，结果发现就是：

如果使用 ArrayList，时间就是1s
如果使用 ArayDeque，时间就是2s
所以比较困惑，这两种集合的效率查别这么大吗？

简单性能测试
于是我做了下面一个测试：测试为分别往 ArrayList 和 ArrayDeque 中存入一千万个数和删除一千万个数，比较时间：

// 下面是测试代码：
```java
public class ListTest {
    public static void main(String[] args) {
        ArrayList<Integer> list = new ArrayList<>();
        ArrayDeque<Integer> deque = new ArrayDeque<>();

        System.out.println("ArrayList存储时间： " + testArrayListStore(list));
        System.out.println("ArrayList删除时间： " + testArrayListDelete(list));
        System.out.println("ArrayDeque存储时间： " + testArrayDequeStore(deque));
        System.out.println("ArrayDeque删除时间： " + testArrayDequeDelete(deque));
    }

    /**
     * 测试ArrayList大量添加元素效率
     * @param list
     * @return
     */
    private static long testArrayListStore(ArrayList<Integer> list) {
        long start1 = System.currentTimeMillis();
        for (int i = 0; i < 10000000; i ++) {
            list.add(i);
        }
        long end1 = System.currentTimeMillis();
        return end1 - start1;
    }

    /**
     * 测试ArrayList大量删除最后元素效率
     * @param list
     * @return
     */
    private static long testArrayListDelete(ArrayList<Integer> list) {
        for (int i = 0; i < 10000000; i ++) {
            list.add(i);
        }
        long start = System.currentTimeMillis();
        while (!list.isEmpty()) {
            list.remove(list.size() - 1);
        }
        long end = System.currentTimeMillis();
        return end - start;
    }

    /**
     * 测试ArrayDeque大量添加元素效率
     * @param deque
     * @return
     */
    private static long testArrayDequeStore(ArrayDeque<Integer> deque) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10000000; i ++) {
            deque.add(i);
        }
        long end = System.currentTimeMillis();
        return end - start;
    }

    /**
     * 测试ArrayDeque大量删除最后元素效率
     * @param deque
     * @return
     */
    private static long testArrayDequeDelete(ArrayDeque<Integer> deque) {
        for (int i = 0; i < 10000000; i ++) {
            deque.add(i);
        }
        long start = System.currentTimeMillis();
        while (!deque.isEmpty()) {
            deque.removeLast();
        }
        long end = System.currentTimeMillis();
        return end - start;

    }
}
```
效率结果：
ArrayList  存储时间： 4595
ArrayList  删除时间： 20
ArrayDeque 存储时间： 531
ArrayDeque 删除时间： 12
结果发现：两者的效率差别非常大，ArrayDeque 整体上会更加高效！

源码分析
1. ArrayList 的 add() 方法
```java
public boolean add(E e) {
    	// 是否越界检查
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

// 确保数组不越界
private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
	}

// 计算最小需要容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
// 检查是否需要扩容
private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            // ArrayList是扩容为1.5倍
            grow(minCapacity);
    }
```
一看，挺简单是不？当然不是，虽然赋值只需要一句，但是重头戏在 ensureCapacityInternal(size + 1) 方法上！
这里不具体对这个方法进行展开，因为涉及到扩容相关逻辑，简单来说就是：

当向ArrayList中加入一个值的时候，它首先会进行容量判断，如果当前数组长度不够了，就要先扩容，然后才能添加，这个过程比较耗时间！

2. ArrayDeque 的 add() 方法
```java
public boolean add(E e) {
        addLast(e);
        return true;
    }

public void addLast(E e) {
        if (e == null)
            throw new NullPointerException();
        elements[tail] = e;
    	// 判断是否需要扩容，这里是最妙的地方！
        if ( (tail = (tail + 1) & (elements.length - 1)) == head)
            // 扩容双倍
            doubleCapacity();
    }

  // 扩容方法
  private void doubleCapacity() {
        assert head == tail;
        int p = head;
        int n = elements.length;
        int r = n - p; // number of elements to the right of p
        int newCapacity = n << 1;
        if (newCapacity < 0)
            throw new IllegalStateException("Sorry, deque too big");
        Object[] a = new Object[newCapacity];
        System.arraycopy(elements, p, a, 0, r);
        System.arraycopy(elements, 0, a, r, p);
        elements = a;
        head = 0;
        tail = n;
    }
```
在 ArrayDeque 中维护了一个 head 和 tail 指针，分别指向元素头(head)和元素尾的下一个位置(tail)

![avatar](https://img-1300762533.cos.ap-guangzhou.myqcloud.com/my-blog/image-20210428112326180.png)
如果当前tai已经到达数组最后，一旦插入后，tail就会指向下标0的位置，这样就形成了一个环状数组结构，之所以这么设计是因为 ArrayDeque 可以同时当栈和队列来使用。

然后讲讲上面方法中的精髓：
```java
 if ( (tail = (tail + 1) & (elements.length - 1)) == head)
            doubleCapacity();
/**
上面条件判断可以分成两部分来看：
	1）tail = (tail + 1) & (elements.length - 1)
		- 这里是一个赋值操作，tail的值完全取决于后面的 & 运算
		- 首先要交代一个前提就是：ArrayDeque在初始化容量的时候，通过一系列的移位运算保证了数组的长度为2^n（这部分可参考源码）
		- 所以这里 (elements.length-1) 就是：01111...11 这样的数
		(1) 如果(tail + 1) <= (elements.length - 1): 运算结果就是 (tail + 1)
		(2) 如果(tail + 1) == elements.length（比如100...000）： 运算结果就是0，即tail指向数组第一个位置（环状数组）
	 所以这里就可以保证 tail 赋值正确
	2）tail == head
	 这个条件就是判断当前数组是否已满！如果tail指向了head的位置，就代表数组满了，此时就需要进行扩容 doubleCapacity()
*/
```
所以通过简单的比较，这个进行大量的元素插入的时候，ArrayDeque会相对效率更高。

3.ArrayList 的 remove() 方法

ArrayList 的删除方法
```java
 public E remove(int index) {
     	// 合法检查
        rangeCheck(index);
		
        modCount++;
     	// 待删除值
        E oldValue = elementData(index);
		// 确定需要移动的步数
        int numMoved = size - index - 1;
        // 如果不是删除最后一个元素，就需要整体复制数组（最后一个删除不需要移动，这也很好理解）
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        // 原位置直接复制null,让垃圾收集器工作
        elementData[--size] = null; // clear to let GC do its work
		// 返回删除值
        return oldValue;
    }
```
4.ArrayDeque 的 remove() 方法

这个方法和ArrayList相比基本相同，只不过ArrayList可以支持删除任意下标元素，而ArrayDeque只能删除头尾元素
```java
public E remove() {
        return removeFirst();
    }

// 这里就看 removeLast()和 removeFirst()基本一样
public E pollLast() {
    	// 获取待删除位置下标, tail指向下一个待插入的位置
        int t = (tail - 1) & (elements.length - 1);
        @SuppressWarnings("unchecked")
    	// 获取待删除元素
        E result = (E) elements[t];
        if (result == null)
            return null;
        // 待删除位置赋值null
        elements[t] = null;
        // tail指针移到待删除位置
        tail = t;
        return result;
    }
```