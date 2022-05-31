# List

## 1. ArrayList

### 1.1 数据结构

ArrayList底层就是一个数组，数组会随着数据的增长而扩容，数组的扩容就是建立一个新的容量更大的数组，然后把旧数组上面的数据复制到新数组上。

因为是数组，所以支持随机访问，且有序。

### 1.2 常用方法

#### 1.2.1 无参构造方法

初始化数组为一个空数组。

```java
/**
* Constructs an empty list with an initial capacity of ten.
*/
public ArrayList() {
  this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

#### 1.2.2 add(E e) 添加元素

1. 将指定的元素追加到此List的末尾

2. 添加元素时，容量判断（会先检查是否超出容量，如果超出，则进行扩容操作）。

   当第一次添加元素时，size默认值是0，会计算出一个最小容量minCapacity，如果是无参构造创建的List，则会取默认的容量10，根据Math.max(DEFAULT_CAPACITY, minCapacity) ， 因为这里传入的minCapacity是0，所以选择了更大的值10返回。如果计算出来的最小容量大于原有容量（minCapacity - elementData.length > 0）则会进行扩容。

```java
/**
* Appends the specified element to the end of this list.
*
* @param e element to be appended to this list
* @return <tt>true</tt> (as specified by {@link Collection#add})
*/
public boolean add(E e) {
	ensureCapacityInternal(size + 1);  // Increments modCount!!
	elementData[size++] = e;
	return true;
}

private void ensureCapacityInternal(int minCapacity) {
  ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private void ensureExplicitCapacity(int minCapacity) {
  modCount++;

  // overflow-conscious code
  if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
  if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
    return Math.max(DEFAULT_CAPACITY, minCapacity);
  }
  return minCapacity;
}
```

3. 添加元素时，扩容 - 计算扩容需要的容量

   1. 扩容的算法是：扩大为原容量的1.5倍，入股扩容后的容量仍小于需要的最小容量minCapacity，则新的容量就取最小容量进行扩容。
   2. 如果扩容后的大小超过最大容量，则会触发hugeCapacity的操作。

   ```java
   /**
   * Increases the capacity to ensure that it can hold at least the
   * number of elements specified by the minimum capacity argument.
   *
   * @param minCapacity the desired minimum capacity
   */
   private void grow(int minCapacity) {
     // overflow-conscious code
     int oldCapacity = elementData.length;
     int newCapacity = oldCapacity + (oldCapacity >> 1);
     if (newCapacity - minCapacity < 0)
       newCapacity = minCapacity;
     if (newCapacity - MAX_ARRAY_SIZE > 0)
       newCapacity = hugeCapacity(minCapacity);
     // minCapacity is usually close to size, so this is a win:
     elementData = Arrays.copyOf(elementData, newCapacity);
   }
   private static int hugeCapacity(int minCapacity) {
     if (minCapacity < 0) // overflow
       throw new OutOfMemoryError();
     return (minCapacity > MAX_ARRAY_SIZE) ?
       Integer.MAX_VALUE :
     MAX_ARRAY_SIZE;
   }
   ```

4. 添加元素时，扩容 - 复制旧数据

   新建一个数组初始化为新容量，然后复制旧元素到新数组。

   elementData = Arrays.copyOf(elementData, newCapacity);

   ```java
   /**
        * Copies the specified array, truncating or padding with nulls (if necessary)
        * so the copy has the specified length.  For all indices that are
        * valid in both the original array and the copy, the two arrays will
        * contain identical values.  For any indices that are valid in the
        * copy but not the original, the copy will contain <tt>null</tt>.
        * Such indices will exist if and only if the specified length
        * is greater than that of the original array.
        * The resulting array is of exactly the same class as the original array.
        *
        * @param <T> the class of the objects in the array
        * @param original the array to be copied
        * @param newLength the length of the copy to be returned
        * @return a copy of the original array, truncated or padded with nulls
        *     to obtain the specified length
        * @throws NegativeArraySizeException if <tt>newLength</tt> is negative
        * @throws NullPointerException if <tt>original</tt> is null
        * @since 1.6
        */
   public static <T> T[] copyOf(T[] original, int newLength) {
     return (T[]) copyOf(original, newLength, original.getClass());
   }
   ```



### 1.3 线程安全问题

#### 1.3.1 线程不安全的原因

1. 扩容过程

   假设数组目前容量是10，当前的元素个数也是10， 这时有A，B两个线程；

   A线程和B线程同时执行 ADD，同时发现size是10， 需要扩容，调用 ensureCapacityInternal(size + 1);方法

2. elementData在对应位置设置对应值的时候



#### 1.3.2 线程不安全的解决办法

1. 使用synchronized关键字

2. 使用Collections.synchronizedList()；

   ```java
   List<String> stringList = Collections.synchronizedList(new ArrayList<String>());
   ```



### 1.4 优劣势和应用场景

#### 1.4.1 优劣势

1. ArrayList的底层是数组，存储诗句的内存也是连续的，所以寻址读取数据效率高，但是插入和删除效率低。

   - 在读取数据的时候，只需要告诉数组从哪个位置（索引）取数据，数组就可以直接把对应索引的数据返回。
   - 由于存储内存是连续的，要插入和删除就要变更整个数组中元素的位置，这个过程就是耗时的过程。
   - ArrayList在添加元素的过程中，可能会触发扩容。

2. 应用场景
   - 随机访问较多，但是插入和删除比较少的场景。
   - ArrayList线程不安全，并发的场景需要处理。

   

   

   

   

## 2 LinkedList

### 2.1 数据结构

LinkedList底层通过双向链表实现，双向链表的每个节点用内部类Node表示。

LinkedList通过first和last引用分别指向链表的第一个和最后一个元素。

item用来存放元素值；next指向下一个node节点，prev指向上一个node节点。

```java

/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;

/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;


private static class Node<E> {
  E item;
  Node<E> next;
  Node<E> prev;

  Node(Node<E> prev, E element, Node<E> next) {
    this.item = element;
    this.next = next;
    this.prev = prev;
  }
}
```



### 2.2 常用方法

#### 2.2.1 LinkedList 新增数据

因为LinkedList是链表，所以LinkedList的新增数据也就是链表的数据新增，这时候要根据要插入的位置区分操作

1. 尾部插入

   默认的add方式就是尾部新增，尾部新增的逻辑很简单，只需要创建一个新的节点，新节点的prev设置现有的末尾节点，现有的末尾Node指向新节点Node，新节点的next设置为null即可。

   ```java
   /**
   * Appends the specified element to the end of this list.
   *
   * <p>This method is equivalent to {@link #addLast}.
   *
   * @param e element to be appended to this list
   * @return {@code true} (as specified by {@link Collection#add})
   */
   public boolean add(E e) {
     linkLast(e);
     return true;
   }
   ```

2. 中间新增

   指定位置插入元素主要分为两部分，

   第一部分是查到node节点，这部分

   第二部分是在查找到的node对象后插入元素。

   ```java
   /**
   * Inserts the specified element at the specified position in this list.
   * Shifts the element currently at that position (if any) and any
   * subsequent elements to the right (adds one to their indices).
   *
   * @param index index at which the specified element is to be inserted
   * @param element element to be inserted
   * @throws IndexOutOfBoundsException {@inheritDoc}
   */
   public void add(int index, E element) {
     checkPositionIndex(index);
   
     if (index == size)
       linkLast(element);
     else
       linkBefore(element, node(index));
   }
   
   void linkBefore(E e, Node<E> succ) {
     // assert succ != null;
     final Node<E> pred = succ.prev;
     final Node<E> newNode = new Node<>(pred, e, succ);
     succ.prev = newNode;
     if (pred == null)
       first = newNode;
     else
       pred.next = newNode;
     size++;
     modCount++;
   }
   ```

   

#### 2.2.2 LinkedList获取数据





#### 2.2.3 LinkedList删除数据





#### 2.2.4 LinkedList 作为队列的应用



   









