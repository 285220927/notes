# 可见性

同步还有另一个重要的方面：内存可见性（Memory Visibility）。我们不仅希望防止某个线程正在使用对象状态而另一个线程在同时修改该状态，而且希望确保当一个线程修改了对象状态后，其他线程能够看到发生的状态变化。

重排序（Reordering）问题。

```tex
在没有同步的情况下，编译器、处理器以及运行时等都可能对操作的执行顺序进行一些意想不到的调整。在缺乏足够同步的多线程程序中，要相对内存操作的执行顺序进行判断，几乎无法得出正确的结论。（这里可以用 volatile 对共享变量进行修饰）
```

## 失效数据

就是 count++ 的时候，当前线程可能会获得另一个线程加之前的数据，另一个线程加完的数据就是“有效”的数据，而加之前的数据就是失效数据。

## 非原子的 64 位操作

Long 和 double 都是 64 bit 的，所以在 32 位机器上会把这两种变量拆成两个变量来操作，所以会出现非原子性操作。除非用 volatile 修饰。

## 加锁与可见性

加锁的含义不仅仅局限于互斥行为，还包括内存可见性。为了确保所有线程都能看到共享变量的最新值，所有执行读操作或者写操作的线程都必须在同一个锁上同步。

## Volatile 变量

轻量级同步机制 volatile 变量。用来确保将变量的更新操作通知其他线程。

1. 禁止重排序；
2. volatile 变量不会被缓存在寄存器或者对其他处理器不可见的地方（禁用 CPU 缓存）；

**volatile** 是一种比 **synchronized** 关键字更轻量级的同步机制。（这种类比并不正确，SynchronizedInteger 在内存可见性上的作用比 volatile 变量更强）

```markdown
仅当 volatile 变量能够筒化代码的实现以及对同步策略的验证时，才应该使用它们。如果在验证正确性时需要对可见性进行复杂的判断，那么就不要使用 volatile 变量。volatile 变量的正确使用方式包括：确保它们自身状态的可见性，确保它们所引用对象的状态的可见性，以及标识一些重要的程序生命周期事件的发生（例如初始化或关闭）。
```

count++ 就不是原子操作，volatile 修饰还是不能保证它的正确性。

**加锁机制既可以确保可见性有可以确保原子性，而 volatile 变量只能确保可见性。**

# 发布与逸出

发布（Publish）一个对象的意思是指，使对象能够在当前作用域之外的代码中使用，例如，将一个指向该对象的引用保存到其他代码可以访问的地方，或者在某一个非私有的方法中返回该引用，或者将引用传递到其他类的方法中。

当某个不应该发布的对象被发布时，这种情况就被称为逸出（Escape）。

其实就是锁的粒度问题，如下

```java
public static Set<Secret> knownSecrets;

public void init(){
    knownSecrets = new HashSet<Secret>();
}
```

这里发布了 knownSecrets 的 HashSet 对象，如果里面有个新的 Secret 对象，那么间接地也发布了该 Secret 对象。可以从一个非私有方法中返回一个引用，那么同样会发布返回的对象。

当发布一个对象时，在该对象的非私有域中引用的所有对象同样会被发布。一般来说，如果一个已经发布的对象能够通过非私有的变量引用和方法调用到达其他的对象，那么这些对象也都会被发布。

**安全的对象构造过程**

不要在构造过程中使 this 引用逸出。

使用静态工厂方法来防止 this 引用在构造过程中逸出。

```java
public class SafeListener{
    private final EventListener listener;
    
    private SafeListener(){
        listener = new EventListener(){
            public void onEvent(Event e){
                doSomething(e);
            }
        };
    }
    
    public static SafeListener newInstance(EventSource source){
        SafeListener safe = new SafeListener();
        source.registerListener(safe.listener);
        return safe;
    }
}
```

# 线程封闭（Thread Confinement）

造成线程不安全的因素就是共享数据，没有共享，就没有伤害。如果仅在线程内访问数据，那就不需要同步。

线程封闭技术的一种常见应用是 JDBC（Java Database Connectivity） 的 Connection 对象。Connect 对象并不要求是线程安全的，但是线程池必须是线程安全的。

## Ad-hoc 线程封闭

维护线程封闭性的指责完全由程序实现来承担。

在 Volatile 变量上存在一种特殊的线程封闭。只要你能确保只有单个线程对共享的 volatile 变量执行写入操作，那么就可以安全地在这些共享 volatile 变量上执行 “读取-修改-写入” 的操作。在这种情况下，相当于将修改操作封闭在单个线程中以防止发生竞态条件，并且 volatile 变量的可见性保证还确保了其他线程能看到最新的值。

由于 Ad-hoc 线程封闭技术的脆弱性，因此在程序中尽量少用它，在可能的情况下应该使用更强的线程封闭技术（例如 栈封闭或 ThreadLocal 类）。

## 栈封闭

栈封闭是线程封闭的一种特例，在栈封闭中，只能通过局部变量才能访问对象。**使用局部变量。**

## ThreadLocal 类

ThreadLocal 对象通常用于对可变的单实例或全局变量进行共享。例如在单线程应用程序中可能会维持一个全局的数据库连接，并在程序启动时初始化这个连接对象，从而避免在调用每个方法时都要传递一个 Connection 对象。

Spring-tx 就是用这个来实现嵌套事务的，来根据大量 ThreadLocal 变量判断，savepoint 回滚、当前事务隔离级别、事务进行状态、事务同步操作。

**TransactionSynchronizationManager**具体就是这个类。在 **DataSourceTransactionManager** 中调用，这是 AbstractPlatformTransactionManager 的子类，表现的是 DataSourceTransaction ，还有 Jpa、JTA、Hibernate 等 Transaction。

```java
/**
 * 对，没错，就是根据这个来绑定和解绑 resource 的
 */
private static final ThreadLocal<Map<Object, Object>> resources =
new NamedThreadLocal<>("Transactional resources");

private static final ThreadLocal<Set<TransactionSynchronization>> synchronizations =
new NamedThreadLocal<>("Transaction synchronizations");

private static final ThreadLocal<String> currentTransactionName =
new NamedThreadLocal<>("Current transaction name");

private static final ThreadLocal<Boolean> currentTransactionReadOnly =
new NamedThreadLocal<>("Current transaction read-only status");

private static final ThreadLocal<Integer> currentTransactionIsolationLevel =
new NamedThreadLocal<>("Current transaction isolation level");

private static final ThreadLocal<Boolean> actualTransactionActive =
new NamedThreadLocal<>("Actual transaction active");
```

当然还有各种各样的 ResourceCacheManager（例如 HBase 里面的，HBaseUtils 里边有个 getTable 方法） 使用 ThreadLocal 来实现的，本质上就是减少分配操作。 还有事务的断点（savePoint）恢复等操作。

## 不变性

满足同步需求的另一种方法是使对象不可变，例如 Google Guava 的 ImmutableMap，创建之后就不能 put 值进去，只能使用 get 读取。

**不可变对象一定是线程安全的。**

只有满足以下条件时，对象才是不可变的。

1. 对象创建以后其状态就能修改。
2. 对象的所有域都是 final 类型。
3. 对象时正确创建的（在对象的创建期间，this 引用没有逸出）。

```java
ImmutableMap<String,String> myMap = ImmutableMap.builder()
    .put("key1", "value1") 
    .put("key2", "value2") 
    .put("key3", "value3") 
    .put("key4", "value4") 
    .put("key5", "value5") 
    .put("key6", "value6") 
    .put("key7", "value7") 
    .put("key8", "value8") 
    .put("key9", "value9")
    .build();
```

### Final 域

关键字 final 可以视为 C++ 中 const 机制的一种受限版本，用于构造不可变性对象。final 类型的域时不能修改的（但如果是 final 域所引用的对象是可变的，那么这些被引用的对象时可以修改的）。然而在 JMM 中，final 域还有着特殊的语义。final 域能确保初始化过程的安全性，从而可以不受限制地访问不可变对象，并在共享这些对象时无须同步。

```java
/**
 * @author <a href="mailto:young1lin0108@gmail.com">young1lin</a>
 * @version 1.0
 * @since 2020/11/18 9:20 下午
 */
public class OneValueCache {

    private final BigInteger lastNumber;

    private final BigInteger[] lastFactors;

    public OneValueCache(BigInteger i,BigInteger[] factors){
        this.lastNumber = i;
        this.lastFactors = Arrays.copyOf(factors,factors.length);
    }

    public BigInteger[] getLastFactors(BigInteger i){
        if(lastNumber == null || !lastNumber.equals(i)){
            return null;
        }
        return Arrays.copyOf(lastFactors,lastFactors.length);
    }

}
```

## 安全发布

1. 不正确的发布：正确的对象被破坏。
2. 不可变对象与初始化安全性。
3. 安全发布的常用模式。

要安全地发布一个对象，对象的引用以及对象的状态必须同时对其他线程可见。一个正确的构造的对象可以通过以下方式来安全发布。

+ 在静态初始化函数中初始化一个对象引用。
+ 将对象的引用保存到 volatile 类型的域或者 AtomicReferance 对象中。
+ 将对象的引用保存到某个正确构造对象的 final 类型域中。
+ 将对象的引用保存到一个由锁保护的域中。

4. 事实不可变的对象。

就是 ImmutableMap，生成后就不可变。

### 可变对象

对象的发布需求取决于它的可变性：

+ 不可变对象可以通过任意机制来发布。
+ 事实不可变对象必须通过安全方式来发布。
+ 可变对象必须通过安全方式来发布，并且必须是线程安全的或者由某个锁保护起来。

### 安全的地共享对象

线程封闭

只读共享

线程安全共享

保护对象

