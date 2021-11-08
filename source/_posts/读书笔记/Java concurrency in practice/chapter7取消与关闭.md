---
title: chapter7取消与关闭
date: 2020-12-27 19:53:13
<<<<<<< HEAD
tags: [并发, 读书笔记]
=======
tags: [并发, 读书笔记, java concurrency in practice]
>>>>>>> fdb26db843d2d64440296f4c32d43af5a613f48f
---

https://github.com/funnycoding/blog/issues/35

任务和线程的「启动」很容易，在大多数时候，我们会让它们正常运行直到**结束**或者**「自行停止」**。

然而，有时候我们希望**提前结束**任务或线程，背后的原因可能是用户进行了**「取消」**操作，也可能是「**应用程序」**需要被快速关闭。

要使任务和线程能**「安全」**、**「快速」**、**「可靠」**地停止下来，并不是一件易事。 `Java` 没有提供任何机制来**安全地终止线程**。（`Thread.stop` 和 `suspend` 提供了这样的机制，但是因为**严重的缺陷**，应避免使用这两个方法）。 但是 `Java` 提供了 **「中断」**（Interruption） ，这是一种 **「协作机制」**，**能够使一个线程终止另一个线程的当前工作。**<!--注意了，这是一个协作机制-->

这种协作式的方法是必要的，我们很少希望某个任务、线程或服务**「立即停止」**，因为这种**立即停止**会导致「共享」的「数据结构」 处于**不一致**的状态。<!--是滴，以前工作的时候确实有这种事故-->

相反，在编写任务和服务时可以使用一种**协作**的方式：当需要停止时，它们**首先**会**「清除当前正在执行的工作」**，**然后再结束**。<!--所以说，不要着急，要一步一步的来，不要立马就停止-->

这提供更好的**灵活性**，因为「**任务本身**」的代码比发出取消请求的代码更清楚如何执行清除工作。

**生命周期结束（End-of-Lifecycle)** 的问题会使**「任务」**、**「服务**」以及程序的设计和实现等过程变得**复杂**，而这个在程序设计中非常重要的要素经常被忽略。

一个在**「行为良好」**的软件与**「勉强运行」** 的软件中**最主要的区别**就是：行为良好的软件能很完善地处理**「失败」**、**「关闭」**和**取消**等过程。 

本章将给出各种实现**「取消」** 和 **「中断」** 的机制，以及如何编写任务和服务，使它们对取消请求做出相应。 

### 任务取消

如果外部代码能在某个操作正常完成之前将其设置为「完成」状态，那么这个操作就可以被称为是 **「可取消的」（Cancellable）**。 取消某个操作的**原因**很多：

- **用户主动请求取消。**用户点击图形界面中的"取消"按钮，或者通过管理接口发来取消请求，例如 `JMX（Java Management Extensions）`
- **有时间限制的操作。**某个应用程序需要在限定时间内搜索问题空间，并在这个时间内选择最佳的解决方案。当计时器超过时，需要取消所有正在搜索的任务。
- **一个应用程序事件。**例如当应用程序对某个问题空间进行分解并搜索时，不同的任务可以搜索问题空间中的不同区域，当其中一个任务找到了解决方案时，其他扔在搜索的任务都将被取消。
- **当发生错误时。**比如爬虫程序搜索相关页面，当任务发生错误时（例如磁盘空间不足）那么其他所有搜索任务都会取消，此时可能会「记录它们当前的状态」，以便稍后重新启动。
- **应用程序关闭。** 当一个程序或服务关闭时，必须对正在处理和等待处理的工作执行某种操作。在**「平缓的关闭」**过程中，当前正在执行的任务将继续执行直到完成，而在**「立即关闭」**的过程中，当前的任务可能被取消。

在 `Java` 中没有一种**安全的**「**抢占式**」方法来停止线程，因此也就没有**「安全的抢占式方法」**来停止任务。只有一些 「协作式」 的机制，使**请求取消任务**和**代码**都遵循一种**协商**好的**「协议」**。<!--有些人批评，有些人支持-->

「其中一种」 协作机制能设置某个**「已请求取消（Cancellation Requested）」** 标志，而任务将定期查看该标志。当该标志位被设置为 `TRUE` 时，那么任务将**「提前结束」。**

**程序清单7-1** 就使用了这个方法，其中的 `PrimeGenerator` 持续地枚举素数，直到它被取消。 `cancel` 方法将设置 `cancelled` 标志，并且主循环在每次做搜索下一个素数这个操作之前都会检查一下这个标志（为了使这个过程能**「可靠地」**工作，标志 `cancelled` 必须是 `volatile` 类型）<!--与前面4章的内容联系起来，线程可见性，cancelled属于一个对象的field并且是对外共享的数据，而且必须保证修改后的状态立即对其他线程可见，这里再提一嘴，一般作为标志的状态变量，都属于线程可见的共享变量-->

> **程序清单 7-1** 使用 `volatile` 类型的域来保存取消状态：

```java
// 通过一个被 volatile 修饰的标志位来判断任务是否被取消
@ThreadSafe
public class PrimeGenerator implements Runnable {
    private static ExecutorService exec = Executors.newCachedThreadPool();

    @GuardedBy("this")
    private final List<BigInteger> primes = new ArrayList<>();

    private volatile boolean cancelled;

    @Override
    public void run() {
        BigInteger p = BigInteger.ONE;
      	// 每次进行业务操作之前先检查该任务是否被取消
        while (!cancelled) {
            p = p.nextProbablePrime();
            synchronized (this) {
                primes.add(p);
            }
        }
    }

		// 改变该任务是否被停止的标记位状态
    public void cancel() {
        cancelled = true;
    }

    public synchronized List<BigInteger> get() {
        return new ArrayList<>(primes);
    }
}
```

**程序清单 7-2** 给出了这个类的使用示例，让素数生成器运行 1 秒钟后取消。 素数生成器通常并不会刚好在运行 1秒钟后立即停止，因为在**请求取消**的时刻和 `run` 方法中循环执行**下次检查**之间可能存在**「延迟」**。

`cancel` 方法由 `finally`块调用，从而确保即使在调用 `sleep` 时被中断也能取消素数生成器的执行。

如果发生异常而 `cancel`又没有被正常调用，那么这个程序将一直运行下去，不断消耗 CPU 的时钟周期，并使得 `JVM` 无法正常退出。

> **程序清单 7-2** ：一个仅运行一秒钟的素数生成器

```java
  static List<BigInteger> aSecondOfPrimes() throws InterruptedException {
        final PrimeGenerator generator = new PrimeGenerator();
        exec.execute(generator);
        try {
            SECONDS.sleep(1);
        } finally {
          	// 保证任务被取消，否则 JVM 也无法被关闭
            generator.cancel();
        }
        return generator.get();
    }
```

一个**「可取消」**的任务必须拥有「**取消策略（Cancellation Policy）**」，在这个策略中详细定义取消操作的：

- **How** —— 其他代码**如何请求**取消该任务
- **When** —— 任务**什么时候**检查是否已经请求了取消
- **What** —— 在响应请求取消时应该执行**哪些**操作

考虑现实世界中停止支付（Stop-Payment）支票的示例。 银行通常会规定如何提交一个 「停止支付」的请求，在处理这些请求时需要做出哪些「响应性」的保证，以及当支付中断后需要遵守哪些流程（例如通知该事务中涉及的其他银行，以及对付款人的账户进行费用评估）。

这些**「流程」** 和 **「保证」** 结合在一起就构成了 支票支付的取消**「策略」**。

`PrimeGenerator` 使用了一种简单的取消策略：客户代码 <!--主线程--> 通过调用 `cancel` 来请求取消，`PrimeGenerator` 在每次搜索素数前 <!--另外一个线程--> 首先检查是否存在取消请求，如果存在则退出。<!--这就是协作式-->

#### 中断

`PrimeGenerator` 中的取消机制最终会使得搜索素数的任务退出，但在退出过程中需要花费一定的时间。 然而如果使用这种方法的任务调用了一个**「阻塞方法」**，例如 `BlockingQueue.put` 那么可能产生一个 「更严重」的问题 —— 该任务可能永远不会检查「取消标志」，导致永远不会**「结束」**。<!--上述方案出现了严重问题-->

在**程序清单 7-3** 中的 `BrokenPrimeProducer` 就说明了这个问题。 生产者线程生成素数，并将它们放入一个阻塞队列。 如果「生产者」 的速度 超过了 消费者处理的速度，队列将被**「填满」**，`put` 方法将会**阻塞**。<!--严重问题的一个示例-->

当生产者在 `put` 方法中**阻塞**时，如果**消费者希望能够「取消」** 生产者任务，那么就算调用了 `cancel` 方法 将 `cancelled` 关闭标志设置为 `TRUE`，由于 `put` 操作一直被**阻塞**，导致**生产者不能检查这个标志**（因为消费者此时已经停止从队列中取出素数，所以 `put` 方法将一直保持阻塞。）

> **程序清单 7-3 不可靠的取消操作将把生产者置于阻塞的操作中**（不要这么做）：

```java
// Unreliable cancellation that can leave producers stuck in a blocking operation
public class BrokenPrimeProducer extends Thread {
    // 阻塞队列
    private final BlockingQueue<BigInteger> queue;
    private volatile boolean cancelled = false;

    public BrokenPrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!cancelled) {
                queue.put(p = p.nextProbablePrime());
            }
        } catch (InterruptedException e) {

        }
    }

    public void cancel() {
        cancelled = true;
    }
    // 这段是伪代码
    void consumePrimers() {
        BlockingQueue<BigInteger> primes = ...;
        // 生产者
        final BrokenPrimeProducer producer = new BrokenPrimeProducer(primes);
        // 开始生产素数
        producer.start();

        try {
            // 如果需要更多的素数
            while (needMorePrimes()) {
                // 从生产队列中取出元素
                consume(primes.take())
            }
        }finally {
            // 关闭 生产任务
            producer.cancel();
        }
    }
}
```

**第5章** 曾提到，一些**特殊的阻塞库**的方法支持中断。 `线程中断`是一种协作机制 <!--java语言提供了对协作机制的具体支持-->，线程可以通过这种机制来通知另一个线程，告诉它在**合适**或者**可能**的情况下**停止当前工作**，并转而执行其他工作。

> 在 `Java API` 或 「语言规范」中，并没有将任何 「中断」 与任何 「取消」 的语义关联起来，但实际上，如果**在取消之外的其他操作中使用中断**，那么**都是不合适的**，并且很难支撑起更大的应用。

<!--有没有觉得很绕，以订婚为例，中断和取消虽然没有夫妻关系，但是取消不能再找除了中断以外的其他的男人-->

每个线程都有一个 `boolean` 类型的中断状态。当中断线程时，这个线程的中断状态被设置为 `true`。 在`Thread` 中包含了中断线程以及查询线程中断状态的方法，如 **程序清单 7-4**所示：

- `interrupt` 方法可以**「中断目标线程」**
- `isInterrupted` 方法能返回目标线程的中断状态
- 静态的 `interrupted` 方法将**清除当前线程的中断状态**，并**返回它之前的值**，**这也是清除中断状态的唯一方法**

> **程序清单7-4** Thread 中的中断方法：

```java
public class Thread implements Runnable {
  
	 public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
    
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }
    
     public boolean isInterrupted() {
        return isInterrupted(false);
    }
}
```

「阻塞库」方法，例如 `Thread.sleep` 和 `Object.wait` 等，都会检查「线程何时中断」，并且在发现**中断时「提前返回」**。 它们在「**响应中断**」时执行的操作包括：

- **清除中断状态**<!--源码中没有这个操作，有争议-->
- 抛出 `InterruptedException` 表示阻塞操作由于中断而提前结束。

`JVM` 并不能「保证」 阻塞方法检测到中断的速度，但在实际情况中响应速度还是非常快的。

当线程在**「非阻塞状态下中断」**时，它的**中断状态将被设置**，然后根据将被取消的操作来检查中断状态以判断是否发生了中断。<!--阻塞状态下收到中断，和非阻塞下收到中断，有什么不同呢？-->

通过这样的方法，中断操作将变得 "有黏性" ---如果不触发 `InterruptedException` ，那么中断状态将一直保持，直到明确地清除中断状态。

> 调用 `interrupt` 并不意味着「立即停止」目标线程正在进行的工作，而只是传递了「请求中断」的消息。

**对「中断操作」 的正确理解是：**

它并不会**「真正地中断」**一个正在运行的线程，而只是发出**中断「请求」**，然后由线程在下一个**合适的时刻**「中断自己」。（这些时刻也被称为 **「取消点」**）。有些方法，例如 `wait`、`sleep`、和 `join`等，将严格地处理这种请求，当它们收到中断请求或者在开始执行时发现某个已被设置好的中断状态时，将抛出一个异常。

- **「设计良好的方法」** 可以完全忽略这种请求，只要它们能使调用代码对中断请求进行某种处理。
- **「设计糟糕的方法」** 可能会屏蔽中断请求，从而导致调用栈中的其他代码无法对中断请求作出响应。

在使用`静态的 interrupted` 时需要**「小心」**，因为它会 **「清除当前线程的中断状态」**。 如果在用`interrupted` 时返回了 `true`，那么除非你想屏蔽这个中断，否则必须对它进行处理 —— 可以抛出 `InterruptedException` ，或者通过再次调用 `interrupt` 来恢复中断状态。

[对Java中interrupt、interrupted和isInterrupted的理解](https://my.oschina.net/itblog/blog/787024)

`BrokenPrimerProducer` 中的问题很容易**「解决和简化」**：使用「中断」 而不是使用 `boolean`标志来请求取消任务，如**程序清单 7-5** 所示 ：在每次迭代循环中，有两个位置可以检测出中断：

1. 阻塞的 `put`方法调用中
2. 循环开始处查询中断状态时

由于调用了 **「阻塞的」** `put` 方法，因此这里并不一定需要进行显式的检测，但执行检测却会使 `PrimeProducer` 对中断具有更高的**「响应性」**，因为它是在启动寻找素数任务之前检查中断的，而不是在任务完成之后。 如果可中断的阻塞方法的**「调用频率」**并不高，不足以获得**「足够的响应性」**，那么显式地检测中断状态可以起到一定的帮助作用。<!--不太理解-->

> **程序清单 7-5** 通过中断来取消

```java
// 使用中断机制来取消任务
public class PrimeProducer extends Thread {
    private final BlockingQueue<BigInteger> queue;

    public PrimeProducer(BlockingQueue<BigInteger> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        try {
            BigInteger p = BigInteger.ONE;
            while (!Thread.currentThread().isInterrupted()) {
                queue.put(p = p.nextProbablePrime());
            }
        } catch (InterruptedException e) {
            // 允许 Thread 退出
        }
    }
  
    // 这里中断任务不再是设置 cancellable 的值，而是直接调用 Thread 的 interrupt() 方法
    public void cancel() {
        interrupt();
    }
}
```

#### 中断策略

正如**任务中应该包含取消策略一样，线程同样应该包含中断策略**。中断策略规定**线程如何解释某个中断请求** —— 也就是当**发现中断请求时，应该做哪些工作（如果需要的话）**，哪些工作单元对于中断来说是「原子操作」，以及以多快的速度来响应中断。

**最合理的中断策略是** —— 某种形式的 **「线程级」 （Thread-Level）** 取消操作或 **「服务级」（Service-Level)** 取消操作： **尽快退出**，并在必要时进行清理，通知某个所有者该线程已经退出。<!--简单概述就是：老大（所有者），我（线程）准备溜了，这事你看着解决-->

此外还可以建立**其他的中断策略**：

- **暂停服务。**
- **重新开始服务。**

但是对于那些包含**非标准中断策略**的线程或线程池，只能用于知道这些策略的任务中。

区分任务和线程对中断的反应是很重要的。<!--这块我觉得写的不好，对于任务来说，只有取消--> 一个中断请求可以有**一个或多个接收者** —— 中断线程池中的某个「工作者」线程，同时意味着：**「取消当前任务」** 和 **「关闭工作者线程」**

**任务不会在其自己拥有的线程中执行，而是在某个服务（例如线程池） 拥有的线程中执行。** 对于非线程所有者的代码来说（例如，对于线程池而言，任何在线程池实现以外的代码），都应该小心地**「保存中断状态」**，这样**拥有线程的代码才能对中断做出响应**，即使**"非所有者" 代码也可以做出响应**（当你为一户人家打扫房屋时，即使主人不在，也不应该把在这段时间内收到的邮件扔掉，而是应该把邮件收集起来，等主人回来再交给他们处理，尽管你可以阅读他们的杂志）。

> 这就是为什么**大多数可阻塞的库函数**都只是抛出 `InterruptedException` 作为对中断的响应。<!--举例说明了上一段的表达的意思-->因为它们永远不会在某个由自己拥有的线程中运行，<!--只throw，不catch-->因此它们为任务或库代码实现了 **「最合理的」取消策略**：**尽快退出执行流程，并把中断信息传递给调用者，从而使调用栈中的「上层代码」 可以采取进一步的操作。**
<<<<<<< HEAD
>
=======
>>>>>>> fdb26db843d2d64440296f4c32d43af5a613f48f

当**检查到中断**请求时，**任务**并不需要放弃所有的操作 —— 它**可以推迟处理中断请**求，并直到某**个更合适的时刻**。 因此需要记住中断请求，并在完成当前任务后 抛出 `InterruptedException` 或 表示已收到中断请求。**这项技术能够确保在更新过程中发生中断时，数据结构不会被破坏。**<!--非常合理，符合业务场景-->

任务不应该对执行该任务的线程的中断策略做出任何**假设**，除非该任务被**专门设计**为在**「服务中运行」**，并且这些**服务中**包含**特定**的中断策略。无论任务把中断视为取消还是其他某个中断响应操作，都应该小心地保存**「执行线程」**的**中断状态**。<!--不知道你有没有感觉，作者一直在强调，要保存执行线程的中断状态--> 如果除了将 `InterruptedException` **传递给调用者**外还需要执行其他操作，那么应该在捕获 `InterruptedException` 之后恢复中断状态：

```java
Thread.currentThread().interrupt()
```

**正如任务代码不该对其执行线程的中断策略做出假设一样，执行取消操作的代码也不应该对线程中的中断策略做出假设。**线程应该只能由其**「所有者」** 中断，所有者可以**将线程的中断策略信息**封装到某个合适的 **「取消机制」**中，例如关闭(`shutdown`)方法。

> 由于**每个线程拥有各自的中断策略**，因此除非你知道中断对该线程的含义，否则就不应该中断这个线程。

#### 7.1.3 响应中断

在 5.4 节中 , 当调用**可中断的阻塞函数**时，例如 `Thread.sleep` 或 `BlockingQueue.put` 等，有两种实用策略可用于处理 `InterruptedException`：

- **传递异常**（可能在执行某个特定于任务的清除操作之后），从而使你的方法也成为可中断的阻塞方法。
- **恢复中断状态**，从而使调用栈中的上层代码能够对其进行处理。

传递 `InterruptedException` 与 将 `InterruptedException` 添加到子句中一样容易，如程序清单 7-6 中的 `getNextTask` 所示：

> 程序清单 7-6 将 InterruptedException 传递给调用者：

```
BlockingQueue<Task> queu;
...
public Task getNextTask() throws InterruptException {
	return queue.take();
}
```

<!--也就是把可能发生的 `InterruptedException` 在方法体中 `throws` 出去就可以了-->

如果不想，或者无法传递 `InterruptedException` （或许是通过 `Runnable` 来定义的任务 <!--非常好的举例--> ），那么需要寻找另一种方式来保存中断请求。一种**标准方法**就是通过 「再次」 调用 `interrupt` 来**恢复中断状态**。 你**不能屏蔽** `InterruptedException`，例如在 `catch` 块中捕获到了异常却不做任何处理，**除非在你的代码中实现了线程的中断策略**。 虽然 `PrimeProducer` 屏蔽了中断信息，由于大多数代码并不知道它们将在哪个线程中运行，因此应该保存中断状态。

> **只有实现了线程中断策略的代码才可以屏蔽中断请求。**在常规的任务和库代码中都不应该屏蔽中断请求。

对于一些**不支持取消**，但仍可以调用**中断阻塞**方法的操作，它们必须在循环中调用这些方法，并在发现中断后重新尝试。

在这种情况下，它们应该在本地保存中断状态，并在返回前恢复状态而不是在捕获 `InterruptedException` 时 恢复状态。

如**程序清单 7-7** 所示。 如果过早地设置中断状态，就可能引起无限循环，因为大多数可中断的阻塞方法都会在入口处检查中断状态，并且当发现该状态已被设置时会立即抛出 `InterruptedException` （**通常，可中断的方法会在阻塞或进行重要的工作前首先检查中断，从而尽快地响应中断**）。

> **程序清单 7-7** 不可取消的任务在退出前恢复中断：

```java
// 不可取消的任务在退出前恢复中断
public class NoncancelableTask {
    public Task getNextTask(BlockingQueue<Task> queue) {
        boolean interrupted = false;
        try {
            while (true) {
                return queue.take();
            }
        } catch (InterruptedException e) {
            // 如果被中断，则将中断标志设置为 True
            interrupted = true;
            // fall through and retry 重试
        } finally {
            // 【重点在这里】 ---> 最终会根据 interrupted 标志位 去恢复线程的中断状态
            if (interrupted) {
                Thread.currentThread().interrupt();
            }
        }
        return null;
    }

    interface Task {
    }
}
```

如果代码不会调用**「可中断的阻塞方法」**，那么仍然可以通过在任务代码中**「轮询」** 当前线程的中断状态来**「响应中断」**。

要选择合适的轮询频率，就需要在**效率**和**响应性**之间进行权衡。 如果**响应性的要求较高**，那么**不应该**调用那些执**行时间较长并且不响应中断的方**法，从而对**可调用的库代码进行一些限制**。

在取消过程中可能涉及除了中断状态之外的其他状态。 中断可以用来获得线程的注意，并且由中断线程保存的信息，可以为中断的线程提供进一步的指示。（当**访问这些信息时，要确保使用同步**）。

例如：**当一个由 `ThreadPoolExecutor` 拥有的 工作线程 检测到中断时，它会 检查 线程池是否正在关闭。 如果是，它会在结束之前执行一些线程池清理工作，否则它可能创建一个新线程将线程池恢复到合理的规模。**

#### 7.1.4 示例：计时运行

许多问题永远无法解决（例如**枚举所有素数**），而某些问题可能很快可以得到答案，也可能永远得不到答案。

在这些情况下，给任务的运行增加一个时限是非常有用的。

**程序清单 7-2 ** 中的 `aSencondOfPrimes` 方法将启动一个 `PrimeGenerator` 并在 1 秒钟后中断。 尽管 `PrimeGenerator` 可能需要超过 1 秒的时间才能停止，但它最终会发现中断，然后停止，并使线程结束。

**在任务执行的另一个方面是，你希望知道在任务执行过程中是否会抛出异常。**

如果 `PrimeGenerator` 在指定时限内抛出了一个未检查的异常，那么这个异常可能会被忽略，因为素数生成器在另一个独立的线程中运行，而这个线程并不会显式地处理异常。

在**程序清单 7-8** 中给出了在指定时间内运行一个任意的 `Runnable` 的示例。 它在调用线程中运行任务，并安排了一个取消任务，在运行指定的时间间隔后中断这个运行任务。

**这解决了从任务中抛出未检查异常的问题，因为该异常会被 timedRun 的调用者捕获。**

> 程序清单 7-8 在外部线程中安排中断（不要这么做）：

```java
// Scheduling an interrupt on a borrowed thread （不要这么做）
public class TimedRun1 {
    private static final ScheduledExecutorService cancelExec = Executors.newScheduledThreadPool();
		// 这个方法可以被任意的线程进行调用
    public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
        // 任务线程
        final Thread taskThread = Thread.currentThread();
        // 在指定时间后调用 任务线程的 interrupt 方法
        cancelExec.schedule(taskThread::interrupt, timeout, unit);
    }
}
```

这是一种非常简单的方法，但却破坏了以下规则：

- 在中断线程之前，应该了解它的中断策略。

由于 `timedRun`可以从任意一个线程中调用，因此它无法知道这个调用线程的中断策略。 如果任务在超时之前完成，那么中断 `timedRun` 所在线程的取消任务将在 `timedRun` 返回到调用者之后启动。 我们不知道在这种情况下将运行什么代码，但结果一定是不好的。（可以使用 `schedule` 返回的 `ScheduledFuture` 来取消这个取消任务以避免这种风险，这种方法虽然可行，但却非常复杂。）

如果**任务不响应中断**，那么 `timedRun` 会在任务结束时才返回，此时可能已经超过了指定的时限 `timeout` （或者还没超过，这是一个不确定的项）。

**如果某个限时运行的服务没有在指定的时间内返回，那么将对调用者带来负面影响。**

在 **程序清单 7-9 ** 中解决了 `aSecondOfPrimes` 的异常处理问题以及之前解决方案中的问题。

执行任务的线程拥有自己的执行策略，即使任务不响应中断，限时运行的方法扔能返回到它的调用者。 在启动任务线程之后， `timedRun` 将执行一个限时的 `join` 方法。 在 `join` 返回后，它将检查任务中是否有异常抛出，如果有的话，则会在调用 `timedRun` 的线程中再次抛出该异常。

由于 `Throwable` 将在两个线程之间共享，因此该变量需要被声明为 `voltaile` 来保证可见性，从而确保安全地将其从 任务线程 发布到 `timedRun` 线程。

> **程序清单 7-9** 在专门的线程中中断：

```java
// Interrupting a task in a dedicated thread 在独立的线程中 中断任务
public class TimedRun2 {
    private static final ScheduledExecutorService cancelExec = Executors.newScheduledThreadPool(1);

    public static void timedRun(final Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
        // 定义一个中断策略
        class RethrowableTask implements Runnable {
            // 定义一个 Throwable 对象，用来传递给调用者
            private volatile Throwable t;

            @Override
            public void run() {
                try {
                    r.run();
                } catch (Throwable t) {
                    this.t = t;
                }
            }

            // 异常重新抛出
            void rethrow() {
                if (t != null) {
                    throw LaunderThrowable.launderThrowable(t);
                }
            }
        }
        RethrowableTask task = new RethrowableTask();
        final Thread taskThread = new Thread(task);
        // 开启任务线程
        taskThread.start();

        // 指定时间后中断任务线程
        cancelExec.schedule(taskThread::interrupt, timeout, unit);

        // 等待工作线程运行超时时间后继续向下执行
        taskThread.join(unit.toMillis(timeout));

        // 重新抛出异常
        task.rethrow();
    }
}
```



在这个示例中的代码解决了之前示例的问题，但是它依赖于一个限时的 `join`，因此存在着 `join` 的不足： 无法知道执行控制是因为线程正常退出而返回还是 join 超时而返回。（这是 Thread API 的一个缺陷： 无论 `join` 是否成功完成，在 Java 内存模型中都会有内存可见性的结果，但 `join`本身不会返回某个状态来表明它是否成功）



#### 7.1.5 通过 Future 来实现取消

我们已经使用了一种抽象机制来管理任务的生命周期，处理异常，以及实现取消，即 `Future`。

通常，使用 `JDK库` 中的代码比自己编写更好，将继续使用 `Future` 和 `任务执行框架` 来构建 `timedRun`。

`ExecutorService.submit` 将返回一个 `Future` 来描述任务。 `Future` 拥有一个 `cancel` 方法，该方法带有一个 `boolean` 类型的参数 `mayInterruptIfRunning` ，表示取消操作是否成功。（这只是表示**任务是否能够接受中断**，而不是表示**任务是否能检测并处理中断**。） 如果这个参数为 `false` ， 那么意味着 "如果任务还没有启动，就不要运行它"，这种方式应该用于那些没有处理中断的任务中。

除非你清楚线程的中断策略，否则就不要中断线程。 那么什么情况下调用 `cancel` 可以将参数指定为 `true` ？

执行任务的线程是由标准的 `Executor` 创建的，它实现了一种中断策略使得任务可以通过中断被取消，所以如果任务在标准的 `Executor` 中运行，并通过它们的 `Future` 来取消任务，那么可以设置 `mayInterruptIfRunning` 当尝试取消某个任务时，不宜直接「中断线程池」，因为你并不知道当中断请求到达时正在运行什么任务 —— 只能通过任务的 `Future` 来实现取消。 这也是在**编写任务时要将中断视为一个取消请求**的另一个理由：可以通过任务的 `Future` 来取消它们。

**程序清单 7-10** 给出了另一个版本的 `timedRun` ： 将任务提交给一个 `ExecutorService`，并通过一个定时的 `Future.get` 来获得结果。 如果 `get` 在返回时抛出了一个 `TimedoutException` ，那么任务将通过它的 `Future` 来取消（为了简化代码，这个版本的 `timedRun` 在 `finally` 块中将直接调用 `Future.cancel` ，因为**取消一个已完成的任务不会带来任何影响。**） 如果任务在被取消前就抛出一个异常，那么该异常将被重新抛出以便由调用者来处理异常。

在**程序清单 7-10**中还给出了另一种良好的编程习惯：**取消那些不再需要的任务**。**（程序清单 6-13 和 6-16 中使用了相同的技术 【渲染页面中加载广告的例子】）**

> **程序清单 7-10** 通过 `Future` 来取消任务：

```
// 通过 Future 来取消任务
public class TimedRun {
    private static final ExecutorService taskExec = Executors.newCachedThreadPool();

    public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {
        // 任务的 Future
        final Future<?> task = taskExec.submit(r);

        try {
            task.get(timeout, unit);
        } catch (TimeoutException e) {
            // 在这里关闭任务
        } catch (ExecutionException e) {
            // 在任务中抛出的异常，在这里重新抛出
            throw LaunderThrowable.launderThrowable(e);
        } finally {
            // 如果任务已经完成，中断任务不会对任务造成影响
            task.cancel(true); // interrupt if running
        }
    }
}
```

> 当 `Future.get` 抛出 `InterruptedException` 或 `TImeoutException` 时，如果你知道不再需要结果，就可以调用 `Future.cancel` 来取消任务。

#### 7.1.6 处理不可中断的阻塞

在 `Java` 库中，许多可阻塞的方法都是通过 **「提前返回」** 或者 抛出`InterruptedException` 来响应中断请求的。 从而使开发人员更容易构建出响应取消请求的任务。

然而，并非所有的可阻塞方法或者阻塞机制都能响应中断：如果一个线程由于执行「同步的」 `Socket I/O` 或者等待获得内置锁 而 阻塞，那么中断请求只能设置线程的中断状态，除此之外没有任何其他作用。

对于那些由于执行不可中断操作而被阻塞的线程，可以使用**类似于中断**的手段来停止这些线程，但这要求我们必须知道**「线程阻塞的原因」**，下面是一些常见的阻塞原因：

- **`java.io` 包中的 同步 Socket I/O 。** 在服务器应用程序中。**最常见的阻塞 I/O 形式**就是对套接字进行「读取和写入」。 虽然 `InputStream` 和 `OutputStream` 中的 `read` 和 `write` 等方法都不会响应中断，但通过关闭底层的套接字，可以使得由于执行read 或 write 等方法被阻塞的线程抛出一个 `SocketException`。
- **java.io 包中的同步 I/O。** 当中断一个正在 `InterruptibleChannel` 上等待的线程时，将抛出 `ClosedByInterruptException` 并关闭链路。（这还会使得其他在这条链路上阻塞的线程同样抛出 `ClosedByInterruptedException`）。 当关闭一个 InterruptibleChannel 时，将导致所有在链路操作上阻塞的线程都抛出 `AsynchronousCloseException`。 大多数标准的 Channel 都实现了 `InterruptibleChannel`。
- `Selector` 的 异步 I/O。 如果一个线程在调用 `Selector.select` 方法（在 `java.nio.channels` 中）时阻塞了，那么调用 `close` 或 `wakeup` 方法会使线程抛出 `ClosedSelectorException` 并提前返回。
- **获取某个锁。** 如果一个线程由于等待某个内置锁而阻塞，那么将无法响应中断。 因为线程认为它肯定会获得锁，所以将不会理会中断请求。 但是在 `Lock` 类中提供了 `lockInterruptibly` 方法，该方法允许在等待一个锁的同时扔能响应中断，详情参见 第13章。

**程序清单 7-11** 的 `ReaderThread` 给出了如何封装非标准的取消操作。 `ReaderThread` 管理了一个套接字连接，它采用同步方式从该套接字中读取数据，并将接收到的数据传递给 `processBuffer` 。

为了结束某个用户的连接或者关闭服务器，`ReaderThread` 改写了 `interrupt` 方法，使其既能处理「标准的中断」，也能关闭底层的套接字。

因此，无论 `ReaderThread` 线程是在 `read` 方法中阻塞还是在某个可中断的阻塞方法中阻塞，都可以被中断并停止执行当前的工作。

> **程序清单 7-11** 通过改写 `interrupt` 方法将非标准的取消操作封装在 `Thread` 中：

```
// 改写 interrupt 方法将非标准的取消操作封装在 Thread 中：
public class ReaderThread extends Thread {
    private static final int BUFFER_SIZE = 512;
    private final Socket socket;
    private final InputStream in;

    public ReaderThread(Socket socket, InputStream in) throws IOException {
        this.socket = socket;
        this.in = socket.getInputStream();
    }

    // 非标准的取消操作
    public void interrupt() {
        try {
            socket.close();
        } catch (IOException e) {
            // 忽略
        } finally {
            // 中断线程
            super.interrupt();
        }
    }

    @Override
    public void run() {
        byte[] buf = new byte[BUFFER_SIZE];
        while (true) {
            try {
                int count = in.read(buf);
                if (count < 0) {
                    break;
                } else if (count > 0) {
                    processBuffer(buf, count);
                }
            } catch (IOException e) {
                // 允许线程退出
            }
        }
    }

    // 处理Buffer的逻辑
    public void processBuffer(byte[] buf, int count) {
    }
}
```

#### 7.1.7 采用 newTaskFor 来封装非标准的取消

我们可以通过 `newTaskFor` 方法进一步优化 ReaderThread 中封装的非标准取消技术，这是 `Java6` 在 `ThreadPoolExecutor` 中新增的功能。

当把一个 `Callable` 提交给 `ExecutorService` 时， `submit` 方法会返回一个 `Future`，我们可以通过这个 `Future` 来取消任务。
`newTaskFor`是一个**工厂方法** ， 它将创建 `Future` 来代表任务。 `newTaskFor` 还能返回一个 `RunnableFuture` 接口，该接口扩展了 `Future` 和 `Runnable` （并由 `FutureTask` 实现）。

通过定制表示任务的 `Future` 可以改变 `Future.cancel` 的行为：例如定制的取消代码可以实现**日志记录**或者 收集取消操作的统计信息，以及**取消一些不响应中断的操作**。

通过改写 `interrupt` 方法， `ReaderThread` 可以取消基于套接字的线程。 同样，通过改写任务的 `Future.cancel` 方法也可以实现类似的功能。

在**程序清单 7-12** 的 `CancellableTask` 中定义了一个 `Cancellable` 接口，该接口扩展了 `Callable`，并增加了一个 `cancel` 方法和一个 `newTask` 工厂方法来构造 `RunnableFuture` 。

`CancellingExecutor` 扩展了 `ThreadPoolExecutor`， 并通过改写 `newTaskFor` 使得 `CancellableTask` 可以创建自己的 `Future`。

> **程序清单 7-12** 通过 `newTaskFor` 将非标准的操作封装在一个任务中：

```
// 通过 newTaskFor 将非标准的操作封装在一个任务中：
public abstract class SocketUsingTask<T> implements CancellableTask<T> {
    @GuardedBy("this")
    private Socket socket;

    protected synchronized void setSocket(Socket s) {
        socket = s;
    }

    public synchronized void cancel() {
        if (socket != null) {
            try {
                socket.close();
            } catch (IOException e) {
                // 忽略
            }
        }
    }


    public RunnableFuture<T> newTask() {
        return new FutureTask<T>(this) {
            public boolean cancel(boolean mayInterruptIfRunning) {
                try {

                    SocketUsingTask.this.cancel();
                } finally {
                    return super.cancel(mayInterruptIfRunning);
                }
            }
        };
    }
}

interface CancellableTask<T> extends Callable<T> {
    void cancel();

    RunnableFuture<T> newTask();
}

@ThreadSafe
class CancellingExecutor extends ThreadPoolExecutor {

    public CancellingExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
            TimeUnit unit, BlockingQueue<Runnable> workQueue) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
    }

    public CancellingExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
            TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory);
    }

    public CancellingExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
            TimeUnit unit, BlockingQueue<Runnable> workQueue,
            RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, handler);
    }

    public CancellingExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime,
            TimeUnit unit, BlockingQueue<Runnable> workQueue, ThreadFactory threadFactory,
            RejectedExecutionHandler handler) {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    }

    // 这里是新添加的方法
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        if (callable instanceof CancellableTask) {
            return ((CancellableTask<T>) callable).newTask();
        } else {
            return super.newTaskFor(callable);
        }
    }
}
```

**【↑ 这个例子也挺复杂的 ，需要对这些类之间的关系比较了解】**

`SocketUsingTask` 实现了 `CancellableTask` 并定义了 `Future.cancel` 来关闭套接字和调用 `super.cancel`。

如果 `SocketUsingTask` 通过其自己的 `Future` 来取消，那么底层的套接字将被关闭并且线程将被中断。

**因此它提高了任务对取消操作的响应性：不仅能够在调用可中断方法的同时确保响应取消操作，而且还能在调用可阻塞的套接字 I/O 方法时响应取消操作。**