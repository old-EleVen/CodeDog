# 面试内容

## 1. HashMap

### 1.1 构造

HashMap 默认初始化容量为16（必须是2的n次幂，如果不是会调整）

加载因子0.75，超出时会扩容

链表转红黑树阈值为8（数组容量至少达到64），红黑树退化为链表阈值为6

### 1.2 tableSizeFor()函数

```java
static final int tableSizeFor(int cap) {
	int n = cap - 1;
	n |= n >>> 1;
	n |= n >>> 2;
	n |= n >>> 4;
	n |= n >>> 8;
	n |= n >>> 16;
	return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

通过位运算将传入的cap长度转化为2的n次幂

### 1.3 Put插入

首先计算key的hash值：取Key的hashcode，然后高16位异或低16位（异或的原因是让结果分布均匀，避免哈希碰撞），然后&上数组的长度-1得到在数组中的位置（取余）

```java
static final int hash(Object key) {
	int h;
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
i = (n - 1) & hash
```

### 1.4 为什么HashMap中的数组长度要是2的n次幂

这样进行与操作是才与取余操作等价

### 1.5 resize()操作时，如何将链表重新定位

```java
(e.hash & oldCap) == 0
```

相当于判断e的hash值扩容位的值，如果为1则需要放到i+oldlength长度的位置，为0则保持位置不变

### 1.6 为什么HashMap链表会形成死循环

Jdk 1.7链表采用头插法，1.8采用尾插法。使用头插法时resize()中的transfer()方法在多线程的情况下会形成死循环

**插入：**

**O(1)**:hash完之后直接插入到Entry数组头

**O(n)**:hash完之后数组长度小与8，遍历之后插入

**O(logn)**:hash完之后是红黑树，插入树中，复杂度是O(logn)

**查找**

hash时间O(1)，equals时间复杂度O(n)。

## 2. ConcurrentHashMap（JDK1.7）

### 2.1 HashTable缺点

put与get方法锁的都是一个方法，效率低

### 2.2 存储结构

Segment（加ReentrantLock锁）+HashEntry数组+链表 

### 2.3 Put()方法

1. 通过哈希算法计算出当前 key 的 hash 值

2. 通过这个 hash 值找到它所对应的 Segment 数组的下标

   ```java
   int j = (hash >>> segmentShift) & segmentMask;
   ```

   取hash值前（Segment长度）&上Segment长度-1，得到Segment数组中的下标j

3. 再通过 hash 值计算出它在对应 Segment 的 HashEntry数组 的下标

   ```java
   int index = (tab.length - 1) & hash;
   ```

   类似HashMap

4. 找到合适的位置插入元素

hash时计算在Segment和HashEnrty位置分别取hash的前几位和后几位，为的是不让key分布在Segment和HashEntry的集中部位，打散分布。

加锁方法是先使用**ReentrantLock**的**TryLock()**来试图获取锁，如果重试次数超过一定次数，直接使用**Lock()**来获取锁

### 2.4 rehash()方法

每个Segment只管自己的扩容

### 2.5 Size()方法

采用乐观锁方法，在统计每个Segment的size过程中如果发生改变则重试，重试次数超过多少次则把所有的segment都加锁

## 3. ConcurrentHashMap（JDK1.8）

### 3.1 底层结构

不再使用分段锁，而是给**数组每一个头结点**的操作都加锁，锁更加细粒度，增加效率，使用的是Synchronized锁+cas

数组+链表+红黑树

### 3.2 Put()

插入时如果桶中的头结点为空，则使用**CAS**将数据插入，否则使用Synchronized锁住头结点

## 4. 锁

### 4.1 乐观锁 悲观锁 

乐观锁就是认为不会发生冲突，通过**cas**和**版本号**来实现  适用场景：读多写少  （Atomic类）

悲观锁就是认为一定会发生冲突，一定要加锁   **Synchronized Lock**  适用场景：读少写多

#### CAS   （Compare-and-Swap）

CAS有3个操作数：**内存地址V，旧的预期值A，新的更新的目标值B**，CAS指令执行时，当且仅当内存地址V的值与预期值A相等时，将内存地址V的值修改为B，否则就什么都不做

- CAS要循环使用，直至更改完成

- 只能保证一个变量的原子操作  解决办法：1. 使用互斥锁保证原子性；2. 将多个变量封装成对象，使用AtomicReference来保证原子性。

- **ABA**问题 加入版本号来解决（AtomicStamptedReference）

### 4.2 公平锁/非公平锁

公平锁：多个线程按照顺序来申请

非公平锁：不按顺序，有可能抢占

### 4.3 可重入锁

一个线程获取之后可以再次获取，用于递归

## 5 Synchronized

悲观锁、非公平锁、可重入锁、独占锁

Synchronized 属于**JVM**层面，是一个**关键字**

**原子性** **内存可见性** **有序性**

- **修饰实例方法**：作用于当前对象，给**对象**加锁
- **修饰静态方法**：给**类**加锁
- **修饰代码块**：给**对象**或者**类**加锁

```java
class A {
    Integer i = 1;
	public static void test() {
		//修饰代码块的情况也有两种，这里表示对类进行同步
		synchronized (A.class) {
			System.out.println("haha");
		}
	}
	public void test2() {
		//这里表示对当前对象进行同步，两者区别看下面锁有几种
		synchronized (this) {
			System.out.println("haha");
		}
	}
    public void test3() {
		//这里表示对当前对象进行同步，类似synchronized (this)
		synchronized (i) {
			System.out.println("haha");
		}
	}
}
```

- 

## 6 ReentrantLock

非公平/公平（默认非公平）可重入锁 自旋锁

是**API**

需要手动来释放锁，提供机制可以中断等待锁

## 7 Volatile

关键字 修饰**变量**  就是保证**变量的可见性**（从内存拿值）然后还有一个作用是**防止指令重排序**

**内存可见性 有序性**

访问Voliatile关键字不会发生阻塞（**没有原子性**）

## 8 单例模式

- 懒汉式
- 饿汉式
- 