---
title: chapter6任务执行
date: 2020-12-25 14:35:00
tags: [并发]
---

大多数并发应用程序都是围绕 **任务执行（Task Execution）**来进行构造的，**任务**通常是一些抽象的且离散的工作单元。<!--我们一直在说并发，但是巨人们已经总结了并发的使用场景和构造方法-->

通过把应用程序的工作分解到多个任务中，可以简化程序的组织结构，提供一种**「自然的事务边界」**来优化错误的恢复过程，以及提供一种自然的**「并行工作结构」**来提升并发性。<!--有了使用场景和构造方法，如何解决构造过程中出现的问题，这里巨人们也给出了方案-->

### 在线程中执行任务

当围绕 **「任务执行」** 来设计应用程序结构时，第一部就是要找出**清晰**的**「任务边界」**。在理想的情况下，各个任务之间是独立的：任务并不依赖于其他任务的状态，结果或边界效应。

**「独立性」**有助于实现并发，如果存在足够多的「资源」，那么这些独立的任务都可以并行执行。为了在**「调度」**与**「负载均衡」**等过程中实现更高的**「灵活性」**,每项任务还表示应用程序的一小部分处理能力。

在**「正常的负载」**下，服务器应用程序应该同时表现出良好**「吞吐量」** 和 快速的 **「响应性」**。 应用程序提供商希望支持尽可能多的用户，而用户希望得到尽可能快的响应。当「负荷过载」时，应用程序的性能应该是逐渐降低，而不是直接失败。

要实现上述目标，应该选择清晰的**「任务边界」**以及明确的任务执行策略（[参见6.2.2 节](https://github.com/funnycoding/blog/issues/31)。）

大多数服务器应用程序都提供了一种自然的任务边界选择方式：以**「独立的客户请求」**为边界。

Web 服务器，邮件服务器，文件服务器，EJB 容器以及数据库服务器等，这些服务器都通过网络接受远程客户的连接请求。将**「独立的请求」**作为**「任务边界」**，既可以实现「任务的独立性」，又可以实现合理的「任务规模」。<!--非常巧妙的设计，即满足了并行工作结构的前提条件（任务边界），又尽量使用独立性有助于实现并发-->

例如：在向邮件服务器提交一个消息后得到的结果，并不受其他正在处理的消息的影响，而且在处理单个消息时通常只需要服务器总处理能力的很小一部分。

#### 串行地执行任务

在应用程序中可以通过多种策略来调度任务，而其中一些策略能够更好地利用潜在的并发性。最简单的策略就是在单个线程中**「串行」**地执行各项任务。 <!--- 也就是不使用多线程技术，所有业务逻辑都在一个线程中完成 -->

**程序清单 6-1** 中的 `SingleThreadWebServer` 将串行地处理它的任务（通过 80 端口接收到的 HTTP 请求）。 至于如何处理任务的细节问题，在这里并不重要，我们感兴趣的是**「如何表征不同调度策略的同步特性」**。

> 程序清单 6-1 串行的 Web 服务器：

```java
public class SingleThreadWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            // 接收客户端的请求
            Socket connection = socket.accept();
            // 以串行的形式处理请求,具体对请求做处理的逻辑，在这里我们不需要关心
            handleRequest(connection);
        }
    }
}
```

正确的，但执行性能却很糟糕，因为它「每次只能处理一个请求」。

主线程在「接受连接 `accept`」与处理相关请求 `handleRequest` 等操作之间不断地交替运行。当服务器正在处理请求时，新到来的连接必须等待直到服务器处理完成上一次的请求，然后服务器再次调用 `accpt`，来接受这一次请求。如果处理请求的速度很快，并且 `handleRequest` 可以立即返回，那么这种方法是可行的，但是现实世界中的 Web 服务器的情况却并非如此。

在 **「Web请求的处理」** 中包含了一组不同的运算与 I/O 操作。 服务器必须处理 套接字 I/O 以读取请求和写回响应，这些操作通常会由于 **「网络拥塞」** 或 **「连接性问题」**而被**「阻塞」**。 此外，服务器还可能处理 文件I/O 或者 数据库请求，这些操作同样会阻塞。 在单线程的服务器中，阻塞不仅会推迟当前请求的完成时间，而且还将彻底阻塞等待中的请求被处理。

- 如果请求**「阻塞」**的时间过长，用户将任务服务器是不可用的，因为服务器看似失去了响应。

- 同时，服务器的**「资源利用率」** 非常低，因为当单线程在等待 I/O 操作完成时， CPU 将处于空闲状态。


在服务器应用程序中，**「串行处理机制」**通常都无法提供「高吞吐率」**或**「快速响应性」。

<!--【在某些情况中，串行处理方式能带来「简单性」和 「安全性」。大多数 GUI 框架都通过单一的线程来串行地处理任务。 第9章 将再次减少串行模型】-->

#### 显示地为任务创建线程

##### a. 通过无限制创建线程的方式

通过为每个请求创建一个新的线程来提供服务，从而实现更高的响应性。

> 程序清单6-2 在 Web服务器中为每个请求都启动一个新的线程**（不要这么做）**：

```java
public class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(80);
        while (true) {
            final Socket connection = serverSocket.accept();
            Runnable task = () -> handleRequest(connection);
            new Thread(task).start();
        }
    }
}
```

`ThreadPerTaskWebServer` 在结构上类似之前的单线程版本 —— 主线程仍然不断地 交替执行 「接受外部链接」与 「分发请求」 这两个操作。 区别在于，对于每个连接，主循环都将创建一个新线程来处理请求，而不是在 「主循环」 中进行处理，由此可得到**3个主要结论**：

- 任务处理过程从主线程中分离出来，到每个新创建的子线程中去，使得主循环能更快地重新等待下一个到来的连接，使得不必等待上一个连接处理完成就可以接受新的请求，提高了**「响应性」**。
- 任务可以**「并行处理」**，从而能同时服务多个请求。如果有「多个处理器」，或者由于某种原因被「阻塞」，例如 「等待I/O完成」，获取锁资源 或者资源可用性等，程序的吞吐量将得到提高。【也就是同时可以处理更多的任务】
- **<u>任务处理代码</u>，即handleRequest(connection), 必须是「线程安全」**的，因为当有多个任务时，会并发地调用这段代码。<!--这块和前面4章的线程安全联系起来了-->

在「正常负载情况下」，「为每个任务分配一个线程」的方法能提升「串行执行」的性能。只要请求到达速率不超出服务器的请求处理能力，那么这种方法可以同时带来 **「更快的响应性」** 和 **「更高的吞吐率」**。

##### b. 无限制创建线程的不足

可以想到，这样为每个请求都创建一个线程的方法肯定有诸多不妥，尤其是有大量请求时，那就需要创建大量的线程：

- **线程生命周期的开销非常高。** 线程的创建与销毁并非没有代价，线程的创建过程都会需要「时间」，「延迟处理的请求」，并且需要 「JVM」 和 「操作系统」提供一些辅助操作。如果请求的到达率非常高且请求的处理过程是轻量级的，大多数服务器应用程序就是这种情况，那么为每个请求创建一个线程这种操作将消耗大量的计算资源。<!--极端情况，请求的处理速度handleRquest()比线程的创建销毁的速度都快，哈哈-->
- **资源消耗。** 「活跃的线程」会消耗「系统资源」，尤其是内存。如果可运行的线程数量多于可用处理的数量，那么有些线程将「闲置」。大量的空余线程会占用许多「内存」，给「垃圾回收器」带来压力， 而且大量线程在竞争 CPU 资源时还将产生 「其他性能开销」。所以如果已经有足够多的线程使 CPU 保持忙碌状态，那么此时创建「更多的线程」反而会「降低」性能。<!--这里就用到了JVM的运行时数据区的知识了，虚拟机栈是线程私有的，不管线程是否运行，只要线程被创建了而且存在着，线程都会占着茅坑不拉屎，消耗内存-->
- **稳定性**。 在「可创建线程」的数量上存在着一个限制。这个限制随着「平台」不同而不同，并且受多个因素制约，包括「JVM 的启动参数」，「`Thread`构造函数中请求的「栈」大小」，以及「底层系统」 对线程的限制等①。 如果破坏了这些限制，那么很可能抛出 `OutOfMemoryError`异常，要想从这种错误中恢复过来很危险，更好的方法是通过「构造程序」 避免超过这些限制。

①【在 32位的机器上，其中一个主要的 **「限制因素」** 是 **线程栈的 地址空间**。 每个线程都维护 **「两个」** 线程栈，一个用于 **「Java 代码」**，另一个用于 **「原生代码」**。通常 JVM 在「默认情况」下会生成一个 **「复合的栈」**，大小约为 0.5MB (可以通过 JVM 标志 `-Xss` 或者通过 `Thread` 的构造函数来修改这个值。)如果将 2^32（32位系统下内存的最大值） 除以每个栈的大小**，那么线程数量将被限制为 「几千」 到 「几万」**。其他的一些因素，例如操作系统的限制等，则可能施加更加严格的约束。】

**在一定范围内，增加线程可以提高系统的吞吐率，当超过这个阈值，再创建更多的线程只会降低程序的执行速度，过多的创建线程，则会使整个应用程序崩溃。**

要想避免这种危险，就需要对程序可以创建的线程数量进行「限制」，并且全面地测试应用程序，从而确保达到线程最大限制数量时，程序也不会因「耗尽资源」而崩溃。

「为每个任务分配一个线程」 这种方法的问题在于"没有限制可创建线程的数量，只限制了远程用户提交 HTTP 请求的速率。"<!--划重点，这是主要原因，没有限制是多么可怕的事情，所以管理资源是多么重要的事情--> 与其他 「并发危险」 一样，在原型设计和开发阶段，无限制的创建线程或许还能正常运行，当到了应用程序部署后并处于高负载下运行时，才会有问题不断地暴露出来。某个恶意用户或者过多的用户同时访问，都会使 Web 服务器的负载达到阈值，从而崩溃。

如果服务器需要提供 高可用性，并且在**「高负载」**情况下**「平缓地降低」**性能，那么这将是一个严重的故障。



### 任务执行框架：Executor 框架

**「任务」**是一组**「逻辑工作」**单元，而线程则是使「任务」**异步执行**的机制。<!--看看巨人们总结的，任务和线程之间的关系-->

之前已经分析过「两种」 通过线程来执行任务的策略：

- 把所有任务在单个线程中串行执行。
- 将每个任务放在各自的线程中执行。

上面这两种方式都存在一些严格的限制：**「串行执行」**的问题在于及其糟糕的响应性和吞吐量，而**「为每个任务分配一个线程」**的问题在于**「资源管理的复杂性」**。

第五章中，我们通过「有界队列」来防止高负荷的应用程序耗尽内存。 「线程池」简化了线程的管理工作，并且 `java.util.concurrent` 提供了一种灵活的线程池作为 `Executor`框架的一部分。

在 Java 类库中，**任务执行**的主要抽象不是 `Thread`，而是 `Executor`，如下面的程序所示：<!--小伙伴们，这里有个新概念，任务执行，千万要搞清楚 任务、线程、任务执行 这3者之间的关系-->

> 程序清单6-3 `Executor` 接口：

```
public interface Executor {
  void execute (Runnable commoand);
}
```

`Executor`为灵活且强大的**「异步任务执行框架」**提供了基础，该框架能支持多种不同类型的任务**「执行策略」**。它提供了一种标准的方法将任务的**「提交」**与**「执行」**过程解耦，使用 `Runnbale`来表示任务。<!--提交与执行的解耦，为能够方便地修改执行策略打下了基础-->

`Executor`的实现还提供了对**「生命周期」**的支持，以及**「统计信息收集」**、**「应用程序管理机制」**和**「性能监视」**等机制。

`Executor`基于**「生产者—消费者」**模式，**提交任务的操作**相当于**「生产者」**（生产待完成的工作单元），**执行任务的线程**则相当于 「消费者」（执行完这些工作单元）。

#### 6.2.1 示例：基于 Executor的Web服务器

基于 `Executor`构建 Web服务器 非常容易。

> 程序清单 6-4 基于线程池的 **Web服务器**：

```java
public class TaskExecutionWebServer {
    private static final int N_THREADS = 100;
    private static final Executor exec = Executors.newFixedThreadPool(N_THREADS);

    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = () -> handleRequest(connection);
            exec.execute(task);
        }
    }

}
```

在 `TaskExecutionWebServer`中，通过使用 `Executor` 将请求处理任务的「提交」 与任务的实际 「执行」 解耦开，并且只需要采用另一种不同的 `Executor` 实现，就可以改变服务器的行为。 改变 `Executor` 实现或配置所带来的影响要远远小于改变「任务提交方式」所带来的影响。

通常，`Executor`的配置是一次性的，因此在部署阶段可以完成，而提交任务的代码却会不断地「扩散」到整个程序，增加了修改的难度。

我们可以很容易地将 `TaskExecutionWebServer`修改为类似 `ThreadPerTaskWebServer` 的行为，只需要使用为每个请求都创建新线程的线程池，编写这样的 `Executor`也很简单，如下所示：

> 程序清单6-5 为每个请求启动一个新线程的 `Executor`：

```java
// 为每个请求都创建一个线程的 Executor
public class ThreadPerTaskExecutor implements Executor {
    @Override
    public void execute(Runnable command) {
        new Thread(command).start();
    }
}
```

同样，还可以编写一个 `Executor`使 `TaskExecutionWebServer`的行为类似于单线程程序的行为 —— 以「同步」 的方式执行每个任务，然后再返回，如下所示：

> 程序清单6-6 在调用线程中以同步方式执行所有任务的 `Executor`：

```java
// 让线程池以单线程的串行形式执行任务
public class WithinThreadExecutor implements Executor {
    @Override
    public void execute(Runnable command) {
        command.run();
    }
}
```

#### 6.2.2 执行策略

通过将任务的「提交」与 「执行」 解耦，从而无需太大的困难就可以为某种类型的任务指定和修改执行策略。<!--这就是把提交与执行解耦的一个原因-->

在执行策略中定义了任务执行的**「What」**、**「Where」**、**「When」**、**「How」**等方面，具体意义如下：

- 在什么（What）线程中执行任务？
- 任务按照什么（What）顺序执行（FIFO、LIFO、优先级）？
- 有多少个（How Many）任务能「并发」执行？
- 在队列中有多少个（How Many）任务在等待执行？
- 如果系统由于「过载」而需要拒绝一个任务，那么应该选择哪一个（Which）任务？另外，如何（How）通知应用程序有任务被拒绝？
- 在执行一个任务之前或之后，应该进行哪些（What）动作？

各种**「执行策略」**都是一种**「资源管理工具」**，最佳策略取决于**「可用的计算资源」**以及**「服务质量的需求」**。

通过**「限制并发任务的数量」**，可以确保应用程序不会由于**「资源耗尽」**而失败，或者由于在稀缺资源上发生竞争而严重影响性能。

通过将任务的提交与执行的策略分离开，有助于在**「部署阶段」**选择与「**可用硬件资源」**最匹配的执行策略。

> 每当看到下面这种形式的代码时：
>
> ```
> new Thread(runnable).start()
> ```
>
> 如果你希望获得一种更灵活的执行策略时，请考虑使用 `Executor`来代替直接使用 `Thread`。

<!--看看大师们的开发经验，非常实际的应用-->

#### 6.2.3 线程池

**「线程池」**指管理一组**「同构」**工作线程的资源池。线程池与**「工作队列（Work Queue)」**关系密切。<!--大意了啊，不要忘记线程池和工作队列可是息息相关的-->

在「工作队列」中保存了所有等待执行的任务。**「工作者线程（Work Thread）」**的任务很简单：从**「工作队列」**中获取一个任务，执行任务，然后返回线程池并等待下一个任务。

"在线程池中执行任务"与比"为每个任务分配一个线程"有众多优势。首先重用线程而不是创建新线程，可以在处理多个请求时「分摊」线程创建和销毁过程中的巨大开销。另外当用户请求到达时，线程已经被创建好，因此省去了等待线程创建的时间，提高了「响应性」。

通过适当调整线程池的大小，可以创建足够多的线程以便使处理器保持忙碌姿态，同时还可以「防止过多」线程相互竞争资源而使应用程序「耗尽内存」或「失败」。

类库提供了一个灵活的线程池以及一些有用的「默认配置」。可以通过 Executors 中的静态工厂方法之一来创建一个线程池，下面是不同的线程池类型：

- `newFixedThreadPool`，创建一个固定长度的线程池，每提交一个任务该线程池中就创建一个线程，直到达到线程池「最大数量」。这时线程池的规模将不再变化（如果有线程在执行时遇到「未预期」的 `Exception`而结束，那么线程池会补充一个新的线程） <---【也就是对于意外减员的应对情况】
- `newCachedThreadPool`，创建了一个「可缓存」的线程池，如果线程池的当前规模超过了处理器的需求，那么将「回收」空闲的线程，而当需求增加时，则可以添加新的线程，该线程池的规模不存在任何限制。
- `newSingleThreadExecutor`，一个单线程的线程池，它创建单个工作线程来执行任务，如果这个线程「异常结束」，会创建另一个线程进行代替。`newSingleThreadExecutor`能确保依照任务在队列中的顺序来「串行」执行（例如FIFO，LIFO，优先级） <!--单线程的 `Executor`提供了大量的「内部」同步机制，从而确保了任务执行的任何内存写入操作对于后续任务来说都是「可见」的。这意味着，即使这个线程会时不时被「另一个线程替代」，但对象总是可以「安全」地「封闭」在「任务线程」 中。-->
- `newScheduledThreadPool`，创建了一个固定长度的线程池，而且以「延迟」或定时的方式来执行任务，类似于 `Timer`。

`newFixedThreadPool` 和 `newCachedThreadPool` 这两个工厂方法返回「通用」 的 `ThreadPoolExecutor` 实例，这些实例可以直接用来构造专门用途的 `executor`。 将在 第8章 中深入讨论「线程池的各个配置选项」。

`TaskExecutionWebService` 中的 Web服务器使用了一个带有 有界线程池 的 `Executor`。通过 `execute` 方法将任务提交到工作队列中，工作线程反复地从工作队列汇总取出并执行它们。

#### 6.2.4 Executor 的生命周期

之前已经说了如何「创建」一个 `Executor`，但是没有讨论如何关闭它。 `Executor` 的实现通常会创建线程来执行任务，但 `JVM` 只有在所有（非守护）线程全部终止之后才会退出。

**「因此，如果无法正确地关闭 `Executor`，那么 `JVM` 将无法结束。」** <!--所以关闭任务执行框架非常的重要-->

由于 `Executor`以 「异步」 方式来执行任务，因此在任何时刻，之前提交任务的状态不是「立即可见」的。有些任务可能已经完成，有些可能正在运行，而其他的任务可能在队列中等待执行。

当关闭应用程序时，可能使用的是「最平缓」的关闭形式（完成所有已经启动的任务，并且不再接受「任何形式」 的新任务），也可能采用的是「最粗暴」的关闭形式 —— 直接关掉机房的电源，以及其他各种可能的形式。

既然 `Executor`是为应用程序提供服务的，所以它们也是可关闭的（无论是采用平缓的还是粗暴的方式），并将在关闭操作中「受影响的任务」的状态反馈给「应用程序」。

为了解决 任务执行框架 `Executor` 的生命周期问题，`ExecutorService`扩展了 `Executor` 接口，添加了一些用于「管理」 生命周期的方法（还有一些用于提交任务的便利方法），具体如下：

```
public interface ExecutorService extends Executor {
    void shutdown();
    List<Runnable> shutdownNow();
    boolean isShutdown();
    boolean isTerminated();
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
		
  // ... 其他用于任务提交的便利方法
}
```

`Executor` 的**「生命周期」** 有3种状态：运行、关闭和已终止。

`ExecutorService`在初始创建时处于「运行」状态。

`shutdown`方法将执行平缓的关闭过程：不再接受新的任务，同时等待已经提交的任务执行完成——包括那些「还未开始执行」的任务。

`shutdownNow`方法则是粗暴的关闭过程：它将尝试「取消」所有「运行中的」任务，并且不再启动队列中尚未开始执行的任务。

在 `ExecutorService` 关闭后提交的任务将由 "拒绝执行处理器"（Rejected Execution Handler）处理，它会「抛弃」任务，或者使得 `execute`方法抛出一个未检查的 `RejectedExcutionException` 。等所有任务都完成后，`ExecutorService`将转入终止状态。可以调用 `awaitTermination` 来等待 `ExecutorService`到达终止状态，或者调用 `isTerminated` 来 「轮询」 `ExecutorService` 是否已经终止。

通常在调用 `awaitTermination` 之后会立即调用 `shutdown`，从而产生「同步地关闭」 `ExecutorService` 的效果。（第7章 将进一步介绍 Executor 的关闭和任务取消方面的内容）

程序清单 6-8 的 `LifecycleWebServer` 通过增加生命周期支持来 「扩展」 Web服务器的功能。 可以通过两种方法来关闭 Web 服务器：

- 在程序中调用 `stop`
- 以客户端请求的形式向 Web服务器 发送一个特定格式的 HTTP 请求

> 程序清单 6-8 支持关闭操作的 Web 服务器：

```java
// 支持关闭操作的 Web 服务器
public class LifecycleWebServer {
    private final ExecutorService exec = Executors.newCachedThreadPool();

    // 开始执行任务
    public void start() throws IOException {
        ServerSocket socket = new ServerSocket(80);
        // 只要任务执行框架没有被通知关闭，则一直执行主循环，同时可能因为 ExecutorService 被关闭抛出 拒绝异常，此时需要对异常进行处理
        while (!exec.isShutdown()) {
            try {
                final Socket conn = socket.accept();
                exec.execute(() -> handleRequest(conn));
            } catch (RejectedExecutionException e) {
                if (!exec.isShutdown()) {
                    log("task submission reject", e);
                }
            }

        }
    }


    // 对连接进行业务处理，首先判断封装的程序是否处于关闭状态，如果是，则调用 ExecutorService 的shutdown() 方法，否则进行任务分发
    void handleRequest(Socket connection) {
        Request req = readRequest(connection);
        if (isShutdownRequest(req)) {
            stop();
        } else {
            dispatchRequest(req);
        }
    }

    private void stop() {
        exec.shutdown();
    }

    private boolean isShutdownRequest(Request req) {
        return false;
    }
}
```

#### 6.2.5 示例：延迟任务与周期任务

`Timer` 类负责管理延迟任务（例如在 100ms 后执行的任务）以及周期任务（"以每 10ms 执行一次的任务"）。

然而 `Timer` 存在一些缺陷，因此应该考虑使用 `ScheduledThreadPoolExecutor` 来代替它。（`Timer`支持的是基于「绝对时间」的调度机制，因此任务的执行对「系统时钟」的变化很敏感，而 `ScheduledThreadPool` 只支持基于相对时间的调度）。

可以通过 `ScheduledThreadPoolExecutor` 的构造函数或 `newScheduledThreadPool` 工厂方法来创建该类的对象。

`Timer` 在执行「所有定时任务」时只会创建一个线程。如果某个线程执行时间过长，那么将**「破坏」**其他 `TimerTask` 的**「定时精确性」**。<!--单线程。。。。Timer的设计师怎么想的。。。-->

例如某个周期 TimerTask 需要 每 10ms 执行一次，而另一个 TimerTask 需要执行 40ms，那么这个周期任务或者在 40ms 任务执行完成后快速连续地调用 4次，或者彻底「丢失」4次调用（取决于它是基于**「固定速率」**还是基于**「固定延迟」来**进行调度）。线程池能弥补这个缺陷，它可以提供「多个线程」来执行 延时任务 和 周期任务。

`Timer` 的另一个问题是：如果 `TimerTask` 抛出了一个「未检查的异常」，那么 `Timer` 将表现出糟糕的行为。

`Timer`线程并不捕获异常，因此当 `TimerTask` 抛出未检查异常时将 「终止」 定时线程。 这种情况下 `Timer` 不会恢复线程的执行，而是会「错误地」 认为整个 Timer 都被取消了。<!--坑爹，抛出未受检查的异常。。。。，找事情啊-->

因此「已经调度」但「尚未执行」的 `TimerTask` 将不会再执行，而新的任务也不能被调度（这个问题称为「线程泄漏」）

在 **程序清单6-9** 的 `OutOfTime` 中给出了 `Timer` 为什么出现这种问题的原因，以及如何使得试图提交 `TimerTask` 的调用者也出现问题。 你可能认为程序会运行6秒后退出，但实际情况是运行1秒就结束了，并抛出一个异常消息 —— "Timer already cancelled"。

`ScheduledThreadPoolExector` 能正确处理这些表现出错误的行为的任务，在 `Java5.0` 或更高的 `JDK`中，很少使用 `Timer`。

如果要构建自己的「调度服务」，可以使用 `DelayQueue`，它实现了 `BlockingQueue` 并为 `SCheduledThreadPoolExecutor` 提供 「调度」 功能。

`DelayQueue` 管理着一组 `Delayed` 对象。每个 `Delayed` 对象都有一个相应的延迟时间：在 `DelayQueue` 中，只有某个元素「逾期」后，才能从 `DelayQueue` 中执行 `take` 操作，从 `DelayQueue` 中返回的对象将根据它们的 「延迟时间」 进行排序。

### 找出可利用的并行性

`Executor` 框架使确定执行策略更加容易，但如果要使用 `Executor`，必须将任务表述为一个 `Runnable`。在大多数服务器应用程序中都存在一个「明显的任务边界」：单个用户请求。

但有时候，任务边界并非是显而易见的，例如在很多「桌面应用程序中」。即使是 「服务器应用程序」，在单个客户请求中可能存在可发掘的「并行性」，例如 「数据库服务器」。

> 程序清单 6-9 错误的 `Timer` 行为：

```java
// Timer 因抛出异常错误结束的情景
public class OutOfTime {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("开始执行");
        Timer timer = new Timer();
        // 执行到这里时 Timer因为抛出了 未预料的异常，之后的代码也无法继续进行了
        timer.schedule(new ThrowTask(),1);
        SECONDS.sleep(1);
        timer.schedule(new ThrowTask(),1);
        SECONDS.sleep(5);
    }

    static class ThrowTask extends TimerTask {
        @Override
        public void run() {
            throw new RuntimeException();
        }
    }
}
```

在这一节中将开发一些「不同版本」 的组件，并且每个版本都实现了「不同程度」的并发性。

该示例组件实现浏览器程序中的 页面渲染（Page-Rendering）功能，它的作用是将 HTML 页面绘制到图像缓存中。为了简便，假设 HTML 页面只包含「标签文本」，以及预定义大小的「图片」和 URL。

#### 示例1：渲染文本、下载、渲染页面都是串行

最简单的方法就是对 `HTML` 文档进行「串行处理」。当遇到文本标签时，将其绘制到「图像缓存」中。 当遇到图像引用时，先通过网络获取它，然后再将其绘制到图像缓存中。

优点是这种方法非常简单：程序只需要将输入中的每个元素处理一次（甚至不需要缓存文档），**但这种方法会让用户等待很长的时间**，因为获取图片可能需要的时间很久，他们必须一直等待，直到显示所有文本。

另一种「串行」执行的方法更好一些，它先绘制文本元素，同时为图像预留出矩形的占位空间，在处理完了第一遍文本后，程序再开始下载图像，并将他们绘制到相应的占位空间中。

> --->renderText(source)   ---> 下载图片1 ---> 下载图片2  .......---> 下载图片n ---> renderImage(图片)



在 **程序清单 6-10** 的 `SingleThreadRenderer`中给出了上述这种方法的实现。
程序清单 6-10 串行地渲染页面元素：

```java
// 串行渲染页面元素
public abstract class SingleThreadRenderer {
    void renderPage(CharSequence source) {
        // 先渲染文字
        renderText(source);
        // 定义页面图像引用
        List<ImageData> imageData = new ArrayList<>();
        // 通过source分析出其包含的图像信息 并将其添加到之前定义的 imageData 中
        for (ImageInfo imageInfo : scanForImageInfo(source)) {
            imageData.add(imageInfo.downloadImage() //网络IO阻塞);
        }
        // 渲染页面图片
        for (ImageData data : imageData) {
            renderImage(data);
        }

    }
}
```

**缺点：**

> 图像下载过程的大部分时间都是在 「等待 I/O」 操作执行完成。 在这期间 CPU 几乎没有任何工作。
>
> 因此，这种串行执行方法没有充分地利用 CPU，使得用户看到最终页面之前要等待过长的时间。
>
> 通过将问题「分解」为多个独立的任务「并发执行」，能够获得更高的 「CPU 利用率」和「响应灵敏度」。



#### 携带结果的任务 Callable 与Future

`Executor` 框架使用 `Runnable` 作为其「基本的任务表示形式」。 `Runnable` 是一种有很大局限性的抽象，虽然 `run` 能写入到日志文件或者将结果放入某个 「共享的数据结构」，但它**不能** 「返回一个值」 或 「抛出一个受检查的异常」。

许多任务实际上都是存在「延迟计算」这种情况的，例如「执行数据库查询」，「从网络上获取资源」，或者计算某个复杂的功能。

对于上述类型的任务 `Callable`是一种更好的抽象：它认为主入口点 `call` 方法将返回一个值，并可能「抛出一个异常」。（要使用 `Callable` 来表示一个无返回值的任务，可使用 `Callable<Void>`）。

```java
// java.util.concurrent/Callable.java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

在 `Executor` 中包含了一些「辅助方法」能将其他类型的任务「封装」为一个`Callable`，例如 `Runnable` 和 `java.security.PrivilegedAction`。

`Runnable` 和 `Callable` 描述的都是「抽象的计算任务」。 这些任务通常是「有范围」的 —— 都有一个明确的起始点，并且最终会结束。

`Executor` 执行的任务有 4 个生命周期阶段：**「创建、提交、开始、完成」**。

由于有些任务可能执行时间很长，因此通常希望能够取消这些任务。 在 `Executor` 框架中，已提交但尚未开始的任务可以取消，但是对于那些**「已经开始」**的任务，只有当它们能**「响应」**中断操作时，才能取消。取消一个已经「完成」的任务不会有任何影响（第7章将进一步介绍取消操作）。

`Future` 表示一个任务的生命周期，并提供了相应的方法来判断是否已经完成或取消，以及「获取任务的结果」和「取消任务」等。 在 **程序清单 6-11** 中给出了 `Callable` 和 `Future`。<!--Callable与Future各自的作用一定要清楚，Callable是被提交给Executor的任务单元；Future是Executor获取Callable以后，Callable对应的生命周期-->

`Future` 规范中包含的**「隐含意义」** 是：任务的生命周期「只能前进」，不能后退，就像 `ExecutorService` 的生命周期一样，当某个任务完成后，它就只能永远停留在「完成」状态上。<!--生命周期只能前进，合理！！！-->

`get`方法的行为「取决于任务的状态」（尚未开始、正在运行、已完成）。 

- 如果任务已经完成，那么 `get` 会立即返回或者抛出一个 `Exception` ，
- 如果任务没有完成，那么 `get` 将阻塞并直到任务完成。
- 如果任务抛出了异常，那么 `get` 将该异常封装为 `ExecutionException`并重新抛出。 
- 如果任务被取消，那么 `get` 将抛出 `CancellationException`。如果 `get` 抛出了 `ExecutionException`，那么可以通过 `getCause` 来获得被封装的「初始异常」。

> **程序清单 6-11** `Callable` 与 `Future` 接口

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}

public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();

    V get() throws InterruptedException, ExecutionException;

    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

可以通过许多方法创建一个 `Future` 来 「描述」 任务。 `ExecutorService` 中的所有 `submit` 方法都将返回一个 `Future`，从而将一个 `Runnable` 或 `Callable` 提交给 `Executor`，并得到一个 `Future` 用来获得任务的执行结果或者取消任务。

还可以显式地位某个指定的 `Runnable` 或 `Callable` 实例化一个 `FutureTask`。（由于 `FutureTask` 实现了`Runnable` ，因此可以将它提交给 `Executor` 来执行，或者直接调用它的 `run` 方法）

从 `Java6` 开始，`ExecutorService` 的实现可以改写为 `AbstractExecutorService` 中的 `newTaskFor`方法，从而根据已提交的 `Runnable` 或 `Callable` 来控制 `Future`的「实例化过程」。

在默认实现中仅创建了一个新的 `FutureTask` 如下所示：

> 程序清单 6-12 `ThreadPoolExecutor` 中的 `newTaskFor` **的默认实现**：

```
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
```

<!--实际上 `newTaskFor` 这个方法不再 `ThreadPoolExecutor.java` 中，而是在 ``AbstractExecutorService.java`中，但是 `ThreadPoolExecutor` 继承 该类，说是默认实现也没问题。-->

将 `Runnable` 或 `Callbale` 提交到 `Executor` 的过程中，包含了一个 「安全发布」的过程（参见3.5） —— 将 `Runnable` 或 `Callable` 从「提交线程」 发布到最终 「执行任务的线程」。 类似地，在设置 `Future` 结果的过程中也包含了一个安全发布，将这个结果从计算它的线程发布到任何通过 `get` 获得它的线程。

#### 示例2： 使用 Future 实现下载图片和渲染文本并行

为了使页面渲染器实现更高的「并发性」，我们将渲染过程分解为「两个任务」：

- 渲染所有文本 （CPU密集型）
- 下载所有的图像 （I/O 密集型）

这样将 CPU密集型 和 I/O 密集型的任务分解开，即使在单 CPU 系统上也能提升性能。

`Callable` 和 `Future` 有助于表示这些协同任务之间的交互，在 **程序清单 6-13** 的 `Future Renderer` 中创建了一个 `Callable` 来下载所有的图像，并将其提交到一个 `ExecutorService`。 这将返回一个 「描述任务执行情况」的 `Future`。 当主任务需要图像时，它会等待 `Future.get` 的调用结果。

如果幸运的话，当开始请求时所有图像就已经下载完成了，即使没有，下载图像的任务也已经提前开始了。

> ---> renderText(source)   
>
> ---> 下载图片1 ---> 下载图片2 ... ---> 下载图片n   ---> renderImage(图片)



> 程序清单 6-13 使用 `Future` 等待图像下载：

```java
// 使用 Future 等待下载图像任务的完成
public abstract class FutureRenderer {
    private final ExecutorService executor = Executors.newCachedThreadPool();

    void renderPage(CharSequence source) {
        final List<ImageInfo> imageInfos = scanForImageInfo(source);
        // 创建一个 Callable 开启一个线程专门下载图片；直到所有图片IO下载完成，这个任务结束
        Callable<List<ImageData>> task = () -> {
            final List<ImageData> result = new ArrayList<>();
            for (ImageInfo imageInfo : imageInfos) {
                result.add(imageInfo.downloadImage());
            }
            return result;
        };
        // 提交这个 Callable 到 Executor 中，获得一个返回的 Future
        final Future<List<ImageData>> future = executor.submit(task);

        // 渲染文字
        renderText(source);

        try {
            // 开始获取下载图片的结果
            final List<ImageData> imageData = future.get();
            // 渲染图片
            imageData.forEach(this::renderImage);
        } catch (InterruptedException e) {
            // 重新声明线程的中断状态
            Thread.currentThread().interrupt();
            // 此时这个 Future 的结果已经不需要了，所以关闭这个任务
            future.cancel(true);
        } catch (ExecutionException e) {
            // 抛出异常
            throw LaunderThrowable.launderThrowable(e);
        }
    }
}
```

`get` 方法拥有 「状态依赖」 的内在特性，因而调用者不需要知道任务的状态，此外在任务 「提交」 和 「获得结果」 中包含的安全发布属性也确保了这个方法是 「线程安全」的。

`Future.get` 的异常处理代码将处理两个可能的问题：

- 任务遇到了一个 Exception
- 或者调用 `get` 的线程在获得结果之前被中断（参见 5.5.2 和 5.4 节）

`FutureRenderer` 使得渲染文本任务与下载图像数据的任务「并发」 地执行。 当所有图像下载完成后，会显示到页面上。 这将提升「用户体验」，不仅使用户更快地看到结果，还有效地利用了「并行性」，但我们还可以做的更好：**「用户不必等待所有图像都下载完成，而是希望每下载完成一个就显示出一个」**。

#### 讨论：在异构任务并行化中存在的局限

上个例子中，我们尝试并行地执行两个**「不同类型」**的任务 —— 「下载图像」与 「渲染页面」。然而，通过对 **「异构任务」** 进行并行化来获得重大的性能提升是很困难的。<!--异构，这个词语，记住了啊-->

如果工作类型相同，比如都是洗完，那么两个人可以很好地分摊工作，一个负责清洗，一个负责烘干，增加人手可以直接提升工作效率。然而，如果将「不同类型的任务」 平均分配给每个工人却并不容易。

当人数增加时，如果没有在「相似」 的任务之间找出细粒度的「并行性」，那么这种方法带来的好处将减少。

当在多个工人之间分配「异构」 任务时，还有一个问题就是各个任务的「大小」可能完全不同。

如果将两个任务 A 和 B 分配给两个工人，但是「 A 的执行时间是 B 的10倍。」，那么整个过程也只能加速 **9%**

当在多个工人之间分解任务时，还需要一定的任务协调开销：「为了使任务分解能提高性能，这种开销不能高于并行性实现的提升。」

`FutureRenderer` 使用了两个任务，其中一个负责**「渲染文本」**，另一个负责**「下载图像」**。 如果渲染文本的速度远高于下载图像的速度（这个可能性很大），那么程序的最终性能与串行执行时的性能差别不大（因为需要等待 下载图像，而渲染文本的耗时可以忽略不计，所以和串行执行几乎差不多），而并行执行任务的代码却更复杂了。因此，虽然做了许多工作来并发执行「异构任务」 以提高并发度，但从中获得的 「并发性」 却十分有限。 （在 11.4.2 节 和 11.4.3节） 中的示例说明了同一个问题。

只有当大量「相互独立」 且 「同构」（相同类型工作）的任务可以进行并发处理时，才能体现出将程序的工作负载分配到「多个任务」 中带来的真正性能提升。

#### 任务完成框架：CompletionService：Executor 与 BlockingQueue

如果向 `Executor` 提交了一组计算任务，并且希望在计算完成后获得结果，那么可以保留与任务关联的 `Future`，然后反复使用 `get`方法，同时将参数 `timeout` 指定为 0 <!--立即查看任务状态-->，从而通过轮询来判断任务是否完成，从而仅仅获得已完成的结果。

这种方法虽然可行，却有些「繁琐」。还有一种更好的方法：**「完成服务」**（`CompletionService`）

`CompletionService` 将 `Executor` 和 `BlockingQueue` 的功能融合在一起。 你可以将 `Callable` 任务提交给它来执行，然后使用类似于队列操作的 `take` 和 `poll` 等方法来获得已完成的结果，而这些结果会在完成时被封装为 `Future`。

`ExecutorCompletionService` 实现了 `CompletionService`，并将**「计算部分」**委托给了一个 `Executor`。<!--其实这块的设计，需要仔细想一下设计模式的概念，如果你想为一个类开发一个全新的功能，你会怎么做呢？JAVA的设计师使用了组合方法，计算部分仍然交给Executor，全新的功能交给CompletionService,那么CompletionService持有一个Exectuor对象即可-->



`ExecutorCompletionService` 的实现非常简单。 在构造函数中创建一个 `BlockingQueue` 来保存计算完成的结果。<!--Executor有任务队列，ExcutorCompletionService比Exectutor多了一个结果队列-->

当计算完成时，调用 `FutureTask` 中的 `done` 方法。

当提交某个任务时，该任务将首先包装为一个 `QueueingFuture` ，这是 `FutureTask` 的一个子类，然后再改写子类的 `done` 方法，并将结果放入 `BlockingQueue` 中，如 **程序清单 6-14** 所示 —— `take` 和 `pool` 方法委托于 `BlockingQueue` ，这些方法在得到结果之前将被「阻塞」。

> 程序清单 6-14 由 `ExecutorCompletionService` 使用的 `QueueingFuture` 类：

```java
// JDK 8 的 QueueingFuture
    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
    }
// JDK 6的 QueueingFuture ，书中给的例子
     private class QueueingFuture extends FutureTask<V> {
        QueueingFuture(Callable<V> c) {super(c);}
        QueueingFuture(RunnableFuture t,V r) {
            super(t r);
        }
        protected void done() { completionQueue.add(this); }
    }
```

可以看到随着 `JDK` 的演化，底层的实现还是有些许不同的地方的。

#### 示例3：使用 CompletionService 实现页面渲染器

可以通过 `CompletionService` 从两个方面来提高页面渲染器的性能：

- 缩短总运行时间
- 提高响应性

为每一个图像的「下载」都创建一个「独立任务」，并在线程中执行它们，从而将「串行」的下载过程转变为「并行」过程。

此外，通过从 `CompletionService` 中获取结果以及使每张图片在下载完成后「立刻」 显示出来，能使用户获得一个更加「动态」和「更高响应性」 的用户界面，如下面的代码所示：

> ---> renderText(source)   
>
> ---> 下载图片1  ---> renderImage(图片)
>
> ---------> 下载图片2  ---> renderImage(图片)
>
> ... 
>
> ------------------------------------> 下载图片n   ---> renderImage(图片)



> **程序清单 6-15** 使用 `CompletionService` 使页面元素在下载完成后立即显示出来：

```java
// 为每个图片分配一个线程进行下载，并且当其下载完成后立即进行渲染
public abstract class Renderer {
    private final ExecutorService executor;

    // 通过传入 ExecutorService 获得不同的特性
    public Renderer(ExecutorService executor) {
        this.executor = executor;
    }

    void renderPage(CharSequence source) {
        final List<ImageInfo> info = scanForImageInfo(source);
        // 初始化 ExecutorCompletionService
        final ExecutorCompletionService<ImageData> completionService =
                new ExecutorCompletionService<>(executor);
        // 为每个图片分配一个Callable线程进行下载
        for (final ImageInfo imageInfo : info) {
            completionService.submit(imageInfo::downloadIamge); //不会阻塞
        }
        // 渲染页面文字
        renderText(source);

        try {
            for (int t = 0; t < info.size(); t++) {
                // 获取下载任务关联的 Future
                final Future<ImageData> f = completionService.take();
                // 获取下载任务的结果 ——> ImageData
                final ImageData imageData = f.get();
                // 渲染页面图片
                renderImage(imageData);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } catch (ExecutionException e) {
            throw LaunderThrowable.launderThrowable(e);
        }
    }
  
		interface ImageInfo {
        ImageData downloadIamge();
    }

}
```

多个 `ExecutorCompletionService` 可以 「共享」 一个 `Executor`。 因此可以创建一个对于「特定计算」私有，又能「共享」一个公共 `Executor` 的 `ExecutorCompletionService` 。因此，`CompletionService` 的作用就相当于一组**「计算句柄」**，这与 `Future` 作为单个计算句柄是非常类似的。 通过记录提交给 `CompletionService` 的任务数量，并计算出已经获得的已完成结果的数量，即使使用一个 「共享的」 `Executor`，也能知道已经获得了所有任务结果的「时间」。

#### 为任务设置超时时间

有时候，如果某个任务无法在指定时间内完成，那么将不再需要它结果，此时可以放弃这个任务。<!--非常常见的业务场景，具有强时效性的任务-->

例如：某个 Web 应用程序从外部的广告服务器上获取广告信息，但如果该应用程序在两秒内得不到响应，那么将显示一个默认的广告，这样即使不能获得广告信息，也不会「降低」 网站的响应性能。 类似地，一个门户网站可以从多个数据源「并行地」获取数据，但可能值会在「指定的时间」内等待数据，一旦超出了等待时间，那么将只显示已经获得的数据。

在有限的时间内执行任务的主要困难在于：**「确保得到答案的时间不会超过限定的时间」**，或者在限定的时间内无法获得答案时做出对应的处理。

在支持「时间限制」的 `Future.get` 中支持这种需求： 当结果可用时，它将立即返回，如果在指定时限内没有计算出结果，将抛出 `TimeoutException`。

在使用时限任务时需要注意，当这些任务超时后应该 「立即停止」，从而避免为无效的任务浪费计算资源。 要实现这个功能，可以由 「任务本身」 来管理它自己的限定时间，并且在超时后 「中止」 或 「取消」 任务。<!--这里，用"中止"和"取消"，比"放弃"任务更加准确，因为"中止"是java线程的一种状态-->

此时可再次使用 `Future` ,如果一个限时的 `get` 方法抛出了 `TimeoutException` ,那么可以通过 `Future` 来取消任务。

如果编写的任务是「可取消」的（参见第7章），那么可以提前中止它，以免消耗过多的资源。 在程序清单 6-13 和 6-16 的代码中使用了这项技术。【提前中止】

程序清单 6-16 给出了限时 `Future.get` 的一种 「典型应用」 —— 在生成的页面中包括「响应用户请求的内容」以及从广告服务器上获得的「广告」。 它将获取广告的「任务」 提交给一个 Executor，然后计算剩余的文本页面内容，最后等待广告信息，直到超出指定的时间。（传递给 `get` 的 `timeout` 参数的计算方法是： 将指定时间 - 当前时间。 这样可能得到 「负数」，但 `java.util.concurrent` 中所有与「时限」 相关的方法都将负数视为零，因此不需要额外的代码来处理这种情况）。

如果 `get` 超时，那么将取消获取广告的任务，并转而使用默认的广告信息。（`Future.cancel` 的参数为 `true` 表示任务线程可以在运行中中断，详见 第7章）

> **程序清单 6-16** 在指定时间内获取广告信息：

```java
// 使用有时限的任务来放弃超时的失效 Task
public class RederWithTimeBudget {
    // 广告信息，初始化时使用默认广告信息
    private static final Ad DEFAULT_AD = new Ad();
    // 超时时间
    private static final long TIME_BUDGET = 1000;
    // 初始化任务执行框架
    private static final ExecutorService exec = Executors.newCachedThreadPool();


    // 这里没有处理被中断的异常，而是将其抛给了调用者进行处理
    Page renderPageWithAd() throws InterruptedException {
        // 结束时间
        final long endNanos = System.nanoTime() + TIME_BUDGET;
        // 提交一个获取广告的任务到任务执行框架中
        final Future<Ad> f = exec.submit(new FetchAdTask());
        // Render the page while waiting for the ad 在等待获取广告的同事，渲染这个页面
        final Page page = renderPageBody();
        Ad ad;
        try {
            // Only wait for the remaining time budget
            // 获取任务执行时间，如果超时则直接抛弃任务 ，如果获取成功则将其赋值给之前定义的广告引用
            final long timeLeft = endNanos - System.nanoTime();
            ad = f.get(timeLeft, TimeUnit.NANOSECONDS);
        } catch (ExecutionException e) {
            // 发生异常时，将广告信息设置为默认信息
            ad = DEFAULT_AD;
        } catch (TimeoutException e) {
            // 如果获取广告的任务超时，不仅将广告设置为默认信息，同时关闭这个获取广告的任务
            ad = DEFAULT_AD;
            f.cancel(true);
        }
        page.setAd(ad);
        return page;
    }

}
```



#### 示例4：旅行预订门户网站
「预定时间」 方法可以很容易地 「扩展」 到任意数量的任务上。

例如这样一个旅行预定门户网站：用户输入旅行的「日期」 和其他要求，门户网站获取并显示来自多条航线，旅店或汽车租赁公司的报价。

在获取不同公司报价的过程中，可能会调用「Web服务」，「访问」 数据库，执行一个 EDI 事务或其他机制。在这种情况下，不应该让页面的响应时间 受限于 「最慢服务」的响应时间，而应该只显示在 「指定时间」内接收到的信息。 对于没有及时响应的服务提供者，页面可以忽略它们，或者显示一个提示信息，例如"未在指定时间内获取到 xxx 信息"。<!--一组任务分解为多个任务上，并且抛弃超时的任务-->

从一个公司获得报价的过程 与 从其他公司获得报价的过程无关。因此可以将获取报价的过程当成「一个任务」，从而使获得报价的过程能「并发执行」。

创建 n 个任务，将其提交到一个线程池，保留 n 个 `Future`，并使用限时的 `get` 方法通过 `Future` 串行地获取每一个结果 ，这一切都很简单，但还有个更简单的方法 ——> `invokeAll`。

下面的示例代码中使用了支持限时的 `invokeAll` ，将多个任务提交到 「一个」 `ExecutorService` 并获得结果。

`InvokeAll` 方法的参数为 「一组任务」，并返回一组 `Future`。 这两个集合有着相同的结构，`invokeAll` 按照任务集合中迭代器的顺序将所有的 `Future` 添加到返回的集合中，从而使调用者能降各个 `Future` 与其表示的 `Callable` 关联起来。<!--太好了，而且集合能将Callable任务单元 与 Future任务结果状态一一对应-->

当所有任务都执行完毕时，或者调用线程被中断时，又或者超过指定时限时， `invokeAll` 将返回。

当超过 「指定时限」后，任务还未完成的任务都会「取消」。 当 `invokeAll` 返回后，每个任务要么正常地完成，要么被取消。 而客户端代码可以调用 `get` 或 `isCancelled` 来判断具体是什么情况。

> 程序清单 6-17 在预定时间内请求旅游报价：

```java
// Requesting travel quotes under a time budget
// 使用 invokeAll 来获取一组报价，这个类的设计非常严谨
public class TimeBudget {

    private static ExecutorService exec = Executors.newCachedThreadPool();

    // 获取报价的方法 在这里调用 QuoteTask 中的方法
    public List<TravelQuote> getRankedTravelQuotes(TravelInfo travelInfo, Set<TravelCompany> companies,
            Comparator<TravelQuote> ranking, long time, TimeUnit unit) throws InterruptedException {
        final List<QuoteTask> tasks = new ArrayList<>();

        // 轮询调用每个旅行社指定 TravelInfo 的报价
        for (TravelCompany company : companies) {
            tasks.add(new QuoteTask(company, travelInfo));
        }

        // 通知任务执行框架开始这一组任务，并获取其 Future
        final List<Future<TravelQuote>> futures = exec.invokeAll(tasks, time, unit);

        // 用来保存真正获取到的报价信息 其数量与获取报价任务的数量相等
        final List<TravelQuote> quotes = new ArrayList<>(tasks.size());

        // 获取任务的迭代器
        final Iterator<QuoteTask> taskIter = tasks.iterator();
        // 遍历 Future 获取其任务执行完成的信息
        for (Future<TravelQuote> f : futures) {
            final QuoteTask task = taskIter.next();
            try {
                quotes.add(f.get());
            } catch (ExecutionException e) {
                // 发生异常时 ，在 task列表中 增加一个 获取失败的报价类
                quotes.add(task.getFailureQuote(e.getCause()));
            } catch (CancellationException e) {
                // 收集因任务关闭导致获取报价失败的类
                quotes.add(task.getTimeoutQuote(e));
            }
        }
        // 排序
        Collections.sort(quotes, ranking);
        return quotes;
    }

}

// 获取报价类的具体实现
class QuoteTask implements Callable<TravelQuote> {
    // 旅行社
    private final TravelCompany company;
    // 不同航线
    private final TravelInfo info;

    public QuoteTask(TravelCompany company, TravelInfo info) {
        this.company = company;
        this.info = info;
    }

    // 获取失败的报价信息
    TravelQuote getFailureQuote(Throwable t) {
        return null;
    }

    // 获取超时的报价信息
    TravelQuote getTimeoutQuote(CancellationException e) {
        return null;
    }

    @Override
    public TravelQuote call() throws Exception {
        // 调用旅行社的获取具体航线信息报价的方法
        return company.solicitQuote(info);
    }
}

// 代表不同旅行社的类
interface TravelCompany {
    // 返回具体报价信息
    TravelQuote solicitQuote(TravelInfo travelInfo) throws Exception;
}

// 报价
interface TravelQuote {

}

// 不同航线的信息
interface TravelInfo {

}
```