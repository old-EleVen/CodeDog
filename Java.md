# 面试内容

## 1. HashMap

### 1.1 构造

HashMap 默认初始化容量为16（必须是2的n次幂，如果不是会调整）

加载因子0.75，超出时会扩容

链表转红黑树阈值为8（数组容量至少达到64），红黑树退化为链表阈值为6

**无序、允许为null（只有一个键为null）、不同步**

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

**不支持value为空**

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

**key value都不为null**

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

## 5 Synchronized 关键字

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

## 6 ReentrantLock 锁

非公平/公平（默认非公平）可重入锁 自旋锁

是**API**

需要手动来释放锁，提供机制可以中断等待锁

## 7 Volatile 关键字

关键字 修饰**变量**  就是保证**变量的可见性**（从内存拿值）然后还有一个作用是**防止指令重排序**

**内存可见性 有序性**

访问Voliatile关键字不会发生阻塞（**没有原子性**）

## 8 单例模式

- 饿汉式

  ```java
  public class Singleton{
  	private static Singleton instance = new Singleton();
  	
  	private Singleton() {
  	}
  	
  	public static Singleton getInstance(){
  		return instance;
  	}
  }
  ```

  

- 懒汉式

  ```java
  public class Singleton{
  	private static Singleton instance;
  	
  	private Singleton(){
  	}
  	
  	public static Singleton getInstance(){
  		if(instance == null)
  			instance = new Singleton();
  		return instance;
  	}
  }
  ```

  

- 双重check懒汉式

  ```java
  public class Singleton{
  	private static volatile Singleton instance;
  	
  	private Singleton{
  	}
  	
  	public static Singleton getInstance(){
  		if(instance == null){
  			synchronized(Singleton.class){
  				if(instance == null)
  					instance = new Singleton();
  			}
  		}
  		return instance;
  	}
  }
  ```

  

- 静态内部类

  ```java
  public class Singleton{
  	private Singleton(){
  	}
  	
  	private static class SingletonHolder{
  		private static Singleton instance = new Singleton();
  	}
  	
  	public static Singleton getInstance(){
  		return SingletonHolder.instance;
  	}
  }
  ```

  利用的原理就是类的加载初始化顺序：

  1. 当类不被调用的时候，类的静态内部类是不会进行初始化的，这就避免了内存浪费问题；

  2. 当有方法调用 getInstance()方法时，会先初始化静态内部类，而静态内部类中的成员变量是**final**的，所以即便是多线程，其成员变量是不会被修改的，所以就解决了添加 synchronized 所带来的性能问题。

  3. 虚拟机会保证一个类的<clinit>()方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么**只会有一个线程**去执行这个类的<clinit>()方法，其他线程都需要阻塞等待，直到活动线程执行<clinit>()方法完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，就可能造成多个进程阻塞(需要注意的是，其他线程虽然会被阻塞，但如果执行<clinit>()方法后，其他线程唤醒之后不会再次进入<clinit>()方法。同一个加载器下，一个类型只会初始化一次。)，在实际应用中，这种阻塞往往是很隐蔽的。

  4. 故而，可以看出INSTANCE在创建过程中是线程安全的，所以说静态内部类形式的单例可保证线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。

- 枚举单例

  ```
  public enum EnumSingleton {
      INSTANCE;
  
      private Object instance;
  
      EnumSingleton() {
          instance = new EnumResource();
      }
  
  }
  EnumSingleton instance = EnumSingleton.INSTANCE;
  ```

  

## 9 java基本类型和占的字节

|  类型   | 所占字节 |
| :-----: | :------: |
|   int   |    4     |
|  short  |    2     |
|  long   |    8     |
|  float  |    4     |
| double  |    8     |
| boolean |    1     |
|  byte   |    1     |
|  char   |    2     |

## 10 final 关键字

- 修饰变量：成员是**基本类型**时，不可变。修饰的是**引用**时，引用不可变，但是引用的值可以变
  - 成员变量：声明时初始化或者在构造器中初始化
  - 本地变量：声明时赋值
- 修饰类：类不能被**继承**
- 修饰方法：方法不能被**重写**

匿名内部类里面的变量使用必须为final，保持内外数据不变

接口中声明的变量都是final(static)

## 11 finally关键字

try-catch 中finally关键字用于异常处理

try-catch-finally 中return返回流程

- 不管return关键字在哪，finally一定会执行完毕
- finally中的块定义的返回值会覆盖try、catch中的值，如果为传**址**（数组、对象）类型则影响，传**值**（基本数据类型及其包装类）类型，不影响。
- 在try中执行System.exit(0)，finally语句不会被执行到

## 12 finalize()方法

Object类中的方法，在垃圾回收前被JVM调用的方法，当某个对象被系统收集为无用信息的时候,finalize()将被自动调用,但是jvm不保证finalize()一定被调用,也就是说,finalize()的调用是不确定的

一般用来回收JNI对象

## 13 HashTable

线程安全，对put、get方法进行加锁（效率低）

不允许key、value为null

## 14 AQS (AbstractQueuedSynchronizer）抽象队列同步器

[AQS](https://zhuanlan.zhihu.com/p/86072774)

AQS就是一个并发包的基础组件，用来实现各种锁，各种同步组件的。

>**AQS核心思想是，如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制AQS是用CLH队列锁实现的，即将暂时获取不到锁的线程加入到队列中。**

包含了**state变量、加锁线程、等待队列**等核心组件。

- state变量：表示加锁的状态，每有一个线程获取到锁之后，state会CAS从0变1。可重入锁就是当前线程持续获取锁是，state一直++
- 加锁线程：记录当前加锁的是哪个线程  **exclusiveOwnerThread**

- 等待队列：存放未获取到锁的线程

- 使用了模板方法模式

## 类加载

### 类加载过程

[类加载]([https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/%E7%B1%BB%E5%8A%A0%E8%BD%BD%E8%BF%87%E7%A8%8B](https://snailclimb.gitee.io/javaguide/#/docs/java/jvm/类加载过程))

![类加载过程](C:\Users\Admin\Pictures\类加载过程-完善.png)

- 加载： **将.class文件加载到内存**
  - 通过全类名获取定义此类的二进制字节流
  - 将字节流所代表的静态存储结构转换为方法区的运行时数据结构
  - 在**内存中生成一个代表该类的 Class 对象**,作为方法区这些数据的访问入口

- 验证：验证加载的类文件的正确性
- 准备：准备阶段是正式为**类变量分配内存并设置类变量初始值**的阶段（static变量）

- 解析：**虚拟机将常量池中的符号引用转化为直接引用**，得到类或者字段、方法在内存中的指针或者偏移量
- 初始化：执行类构造器方法的过程（线程安全，也是静态内部类单例的原因）

### 类加载器

- 启动类加载器：**加载系统类**
- 扩展类加载器：**加载库类**
- 应用程序类加载器：**加载自己的class类**
- 用户自定义类加载器

**双亲委派加载**：加载一个类时首先请求父类加载器来处理，避免加载重复类，同时保证核心API不会被修改。

## 对象创建

![对象创建过程](C:\Users\Admin\Pictures\Java创建对象的过程.png)

### 对象的创建过程

- 类加载检查：检查能否在常量池中定位到这个类的符号引用，如果不能，则先加载类

- 分配内存：

  - 指针碰撞

  - 空闲列表

    **并发问题**：使用CAS+失败重试；TLAB

- 初始化零值：将分配到的内存空间都初始化零值

- 设置对象头：对象类实例的一些信息

- 执行init方法：初始化对象

### 对象访问定位

- 句柄：reference中存放了对象的句柄地址，句柄地址中包含了对象实例数据
- 指针：reference中存放了对象的地址