---
title: 延迟加载应用-01-java语言
date: 2021-05-08 08:57:23
tags:
---



## 应用场景

一般有几种延迟初始化的场景：

- 对于会消耗较多资源的对象：加快对象的创建速度，从而从整体上提升性能；并且如果这些对象可能根本用不到，那么就节省资源了。
- 某些数据在启动时无法获取：比如一些上下文信息可能在其他拦截器或处理中才能被设置，导致当前bean在加载的时候可能获取不到对应的变量的值 <!--bean被加载的时候？加载是针对类说的，对于bean，只能是实例化。如果说的是bean实例化阶段，那么就是发生在spring容器启动阶段以后。从另一方面来说，如果作者想表达的是bean被spring容器加载，那么bean被容器进行di时，spring推荐的是懒加载（延迟初始化），即只注入对象句柄，不关心这个对象句柄指向的对象是否有值 --> ，使用 延迟初始化可以在真正调用的时候去获取，通过延迟来保证数据的有效性。

在Java8中引入的lambda对于我们实现延迟操作提供很大的便捷性，如Stream、Supplier等，下面介绍几个例子。

## 应用实例

#### Lambda Supplier

通过调用`get()`方法来实现具体对象的计算和生成并返回，而不是在定义Supplier的时候计算，从而达到了_延迟初始化_的目的。

> 但是在使用 中需要考虑并发的问题，即防止多次被实例化，就像Spring的@Lazy注解一样。

```java
public class Holder {
    // 默认第一次调用heavy.get()时，触发同步方法；不会在此定义的时候触发
    private Supplier<Heavy> heavy = () -> createAndCacheHeavy(); 
    public Holder() {
        System.out.println("Holder created");
    }
    public Heavy getHeavy() {
        // 第一次调用后heavy已经指向了新的instance，所以后续不再执行synchronized。为什么？？heavy.get()应该是每次调用getHeavy()，都会被执行一次；但是由于
        return heavy.get(); 
    }
   
    private synchronized Heavy createAndCacheHeavy() {
        // 方法内定义class，注意和类内的嵌套class在加载时的区别
        class HeavyFactory implements Supplier<Heavy> {
            // 饥渴初始化
            private final Heavy heavyInstance = new Heavy(); 
            public Heavy get() {
                // 每次返回固定的值
                return heavyInstance; 
            } 
        }
        
        //第一次调用方法来会将heavy重定向到新的Supplier实例
        if(!HeavyFactory.class.isInstance(heavy)) {
            heavy = new HeavyFactory();
        }
        return heavy.get();
    }
}
```

当Holder的实例被创建时，其中的Heavy实例还没有被创建。下面我们假设有三个线程会调用getHeavy方法，其中前两个线程会同时调用，而第三个线程会在稍晚的时候调用。

当前两个线程调用该方法的时候，都会调用到createAndCacheHeavy方法，由于这个方法是同步的。因此第一个线程进入方法体，第二个线程开始等待。在createAndCacheHeavy方法体中会首先判断当前的heavy是否是HeavyInstance的一个实例。

如果不是，就会将heavy对象替换成HeavyFactory类型的实例。

- 第一个线程执行判断时，heavy对象还只是一个`Supplier的实例`，所以heavy会被替换成为`HeavyFactory的实例`，此时`heavy实例`会被真正的实例化，然后返回实例heavyInstance
- 第二个线程执行判断时，heavy已经是HeavyFactory的一个实例了，跳过判断，即不实例化，然后直接返回已存在的实例heavyInstance。

当第三个线程执行getHeavy方法时，由于此时的heavy对象已经是内嵌的HeavyFactory的实例了，因此它会直接调用HeavyFactory的get()方法从而返回需要的实例（即heavyInstance），即不再调用Heavy的同步方法createAndCacheHeavy没有任何关系了。<!--因为Heavy的同步方法是createAndCacheHeavy()-->

<!--在我看来，这是通过把Heavy类向下转为另一个类HeavyFactory（从新实现get()方法），从而get()方法的执行对象变了，这在JUC书中阐述的使用工厂类来避免初始化阶段的线程安全性，一致-->

以上代码实际上实现了一个轻量级的虚拟代理模式(Virtual Proxy Pattern)。保证了懒加载在各种环境下的正确性。

##### 基于delegate：容易理解

```java
public class MemoizeSupplier<T> implements Supplier<T> {

 final Supplier<T> delegate;
 ConcurrentMap<Class<?>, T> map = new ConcurrentHashMap<>(1);

 public MemoizeSupplier(Supplier<T> delegate) {
  this.delegate = delegate;
 }

 @Override
 public T get() {
     // 利用computeIfAbsent方法的特性，保证只会在key不存在的时候调用一次实例化方法，进而实现单例
  return this.map.computeIfAbsent(MemoizeSupplier.class,k -> this.delegate.get());
 }

 public static <T> Supplier<T> of(Supplier<T> provider) {
  return new MemoizeSupplier<>(provider);
 }
}
```

##### 基于CloseableSupplier ：复杂但功能多

```java
public static class CloseableSupplier<T> implements Supplier<T>, Serializable {

        private static final long serialVersionUID = 0L;
        private final Supplier<T> delegate;
        private final boolean resetAfterClose;
        private volatile transient boolean initialized;
        private transient T value;

        private CloseableSupplier(Supplier<T> delegate, boolean resetAfterClose) {
            this.delegate = delegate;
            this.resetAfterClose = resetAfterClose;
        }

        public T get() {
            // 经典Singleton实现
            if (!(this.initialized)) { // 注意是volatile修饰的，保证happens-before，t一定实例化完全
                synchronized (this) {
                    if (!(this.initialized)) { // Double Lock Check
                        T t = this.delegate.get();
                        this.value = t;
                        this.initialized = true;
                        return t;
                    }
                }
            }
            // 初始化后就直接读取值，不再同步抢锁
            return this.value;
        }

        public boolean isInitialized() {
            return initialized;
        }

        public <X extends Throwable> void ifPresent(ThrowableConsumer<T, X> consumer) throws X {
            synchronized (this) {
                if (initialized && this.value != null) {
                    consumer.accept(this.value);
                }
            }
        }

        public <U> Optional<U> map(Function<? super T, ? extends U> mapper) {
            checkNotNull(mapper);
            synchronized (this) {
                if (initialized && this.value != null) {
                    return ofNullable(mapper.apply(value));
                } else {
                    return empty();
                }
            }
        }

        public void tryClose() {
            tryClose(i -> { });
        }

        public <X extends Throwable> void tryClose(ThrowableConsumer<T, X> close){
            synchronized (this) {
                if (initialized) {
                    close.accept(value);
                    if (resetAfterClose) {
                        this.value = null;
                        initialized = false;
                    }
                }
            }
        }

        public String toString() {
            if (initialized) {
                return "MoreSuppliers.lazy(" + get() + ")";
            } else {
                return "MoreSuppliers.lazy(" + this.delegate + ")";
            }
        }
    }
```

#### Lambda Stream

Stream中的各种方法分为两类：

- 中间方法(limit()/iterate()/filter()/map())
- 结束方法(collect()/findFirst()/findAny()/count())

前者的调用并不会立即执行，只有结束方法被调用后才会依次从前往后触发整个调用链条。但是需要注意，对于集合来说，是每一个元素依次按照处理链条执行到尾，而不是每一个中间方法都将所有能处理的元素全部处理一遍才触发 下一个中间方法。比如：

```java
List<String> names = Arrays.asList("Brad", "Kate", "Kim", "Jack", "Joe", "Mike");

final String firstNameWith3Letters = names.stream()
    .filter(name -> length(name) == 3)
    .map(name -> toUpper(name))
    .findFirst()
    .get();

System.out.println(firstNameWith3Letters);
```

当触发findFirst()这一结束方法的时候才会触发整个Stream链条，每个元素依次经过filter()->map()->findFirst()后返回。所以filter()先处理，names中第一个和第二个元素不符合条件，不会触发后续的map方法；继续处理第三个元素符合条件，再触发map()方法，最后将转换的结果返回给findFirst()。所以filter()触发了_3_次，map()触发了_1_次。

Stream类型的一个特点是：它们可以是无限的。Stream之所以可以是无限的也是源于Stream「懒」的这一特点。Stream只会返回你需要的元素，而不会一次性地将整个无限集合返回给你。

Stream接口中有一个静态方法iterate()，这个方法能够为你创建一个无限的Stream对象。它需要接受两个参数：

> public static Stream iterate(final T seed, final UnaryOperator f)

其中，seed表示的是这个无限序列的起点，而UnaryOperator则表示的是如何根据前一个元素来得到下一个元素，比如序列中的第二个元素可以这样决定：f.apply(seed)。

下面是一个计算从某个数字开始并依次返回后面count个素数的例子：

```java
public class Primes {
    
    public static boolean isPrime(final int number) {
        return number > 1 &&
            // 依次从2到number的平方根判断number是否可以整除该值，即divisor
            IntStream.rangeClosed(2, (int) Math.sqrt(number))
                .noneMatch(divisor -> number % divisor == 0);
    }
    
    private static int primeAfter(final int number) {
        if(isPrime(number + 1)) // 如果当前的数的下一个数是素数，则直接返回该值
            return number + 1;
        else // 否则继续从下一个数据的后面继续找到第一个素数返回，递归
            return primeAfter(number + 1);
    }
    public static List<Integer> primes(final int fromNumber, final int count) {
        return Stream.iterate(primeAfter(fromNumber - 1), Primes::primeAfter)
            .limit(count)
            .collect(Collectors.<Integer>toList());
    }
    //...
}
```

对于iterate和limit，它们只是中间操作，得到的对象仍然是Stream类型。对于collect方法，它是一个结束操作，会触发中间操作来得到需要的结果。

如果用非Stream的方式需要面临两个问题：

- 一是无法提前知晓fromNumber后count个素数的数值边界是什么
- 二是无法使用有限的集合来表示计算范围，无法计算超大的数值

所以，因为不知道第一个素数的位置在哪儿，需要提前计算出来第一个素数，然后用while来处理count次查找后续的素数。