---
title: Virtual Threads虚拟线程(译)
date: 2023-10-07 12:00:00
tags: 
- Java
- JDK21
- 并发
description: Oracle文档翻译

---

原文 [Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)

--- 

Virtual threads 虚拟线程是轻量级的线程，可以减少编写、维护和调试高吞吐量并发应用程序的工作量

有关 Virtual Threads 的背景信息，请参阅 [JEP 444](https://openjdk.org/jeps/444)

thread是可调度的最小处理单元，thread之间同时（并且独立）运行

thread有两种，Platform Threads 平台线程和 Virtual Threads 虚拟线程

# Platform Threads 是什么

Platform Threads 是操作系统 (OS) 线程的简单封装，在底层OS线程上运行 Java 代码，因此 Platform Threads 的可用数量受限于OS线程的数量

Platform Threads 通常具有大型线程堆栈和由OS维护的其他资源，适合运行所有类型的任务，但资源有限

# Virtual Threads 是什么

与 Platform Threads 一样，Virtual Threads 也是 java.lang.Thread 的一个实例。 但是，Virtual Threads 并不依赖于特定的OS线程。Virtual Threads 仍然在OS线程上运行，但是当 Virtual Thread 中运行的代码调用阻塞 I/O 操作时，Java 运行时会挂起 Virtual Threads，直到可以恢复为止。与挂起的 Virtual Thread 关联的OS线程现在可以自由地为其他 Virtual Threads 执行操作

Virtual Thread 的实现方式与虚拟内存类似。为了模拟大量内存，操作系统将较大的虚拟地址空间映射到有限的 RAM。同样，为了模拟大量线程，Java运行时将大量 Virtual Threads 映射到少量OS线程。

与 Platform Thread 不同，Virtual Threads 通常具有浅调用堆栈，只执行单个 HTTP 客户端调用或单个 JDBC 查询。 尽管Virtual Threads 支持线程本地变量和可继承的线程本地变量，但您应该仔细考虑使用它们，因为单个 JVM 可能支持数百万个 Virtual Threads

Virtual Threads 适合运行大部分时间处于阻塞状态、通常等待 I/O 操作完成的任务，不适用于长时间运行的 CPU 密集型操作

# 为什么要使用 Virtual Threads

在高并发IO应用程序中使用 Virtual Threads，尤其是那些包含大量并发任务且大部分时间都在等待的应用程序。服务器应用程序是高吞吐量应用程序的示例，因为它们通常处理许多执行阻塞 I/O 操作（例如获取资源）的客户端请求

Virtual Threads 运行代码的速度并不比 Platform Threads 快。 它们的存在是为了提供scale规模（更高的吞吐量），而不是速度（更低的延迟）

# 创建并运行 Virtual Thread

Thread 和 Thread.Builder API 提供了创建 Platform Thread 和 Virtual Threads 的方法。

java.util.concurrent.Executors 类还定义了创建 ExecutorService 的方法，该服务为每个任务启动一个新的 Virtual Threads

## 使用 Thread 类和 Thread.Builder 接口创建 Virtual Threads

调用 Thread.ofVirtual() 方法创建 Thread.Builder 实例来创建 Virtual Threads

以下示例创建并启动一个打印消息的 Virtual Threads。 它调用 join 方法来等待 Virtual Threads 终止（这使您能够在主线程终止之前看到打印的消息）

```java
Thread thread = Thread.ofVirtual().start(() -> System.out.println("Hello"));
thread.join();
```

### Thread.Builder 接口允许您创建具有常见线程属性（例如线程名称）的线程。 Thread.Builder.OfPlatform 子接口创建 Platform Threads，而 Thread.Builder.OfVirtual 创建 Virtual Threads

以下示例使用 Thread.Builder 接口创建一个名为 MyThread 的 Virtual Threads

```java
Thread.Builder builder = Thread.ofVirtual().name("MyThread");
Runnable task = () -> System.out.println("Running thread");
Thread t = builder.start(task);
System.out.println("Thread t name: " + t.getName());
t.join();
```

以下示例使用 Thread.Builder 创建并启动两个 Virtual Threads

```java
Thread.Builder builder = Thread.ofVirtual().name("worker-", 0);
Runnable task = () -> System.out.println("Thread ID: " + Thread.currentThread().threadId());

// name "worker-0"
Thread t1 = builder.start(task);   
t1.join();
System.out.println(t1.getName() + " terminated");

// name "worker-1"
Thread t2 = builder.start(task);   
t2.join();  
System.out.println(t2.getName() + " terminated");
```

输出结果
```txt
Thread ID: 21
worker-0 terminated
Thread ID: 24
worker-1 terminated
```

## 使用 Executors.newVirtualThreadPerTaskExecutor() 方法创建并运行 Virtual Threads

Executors 允许您将线程的管理和创建与应用程序的其余部分分开。

以下示例使用 Executors.newVirtualThreadPerTaskExecutor() 方法创建 ExecutorService。 每当调用 ExecutorService.submit(Runnable) 时，就会创建一个新的 Virtual Thread 并开始运行任务。 该方法返回一个 Future 的实例。 请注意，Future.get() 方法等待线程任务完成。 因此，一旦 Virtual Threads 的任务完成，此示例就会打印一条消息。

```java
try (ExecutorService myExecutor = Executors.newVirtualThreadPerTaskExecutor()) {
    Future<?> future = myExecutor.submit(() -> System.out.println("Running thread"));
    future.get();
    System.out.println("Task completed");
    // ...
}
```

## 多线程 Client Server 示例

以下示例由两个类组成。 EchoServer 是一个服务器程序，它监听端口并为每个连接启动一个新的 Virtual Thread

EchoClient 是一个客户端程序，它连接到服务器并发送在命令行中输入的消息

EchoClient 创建一个 Socket，从而获得与 EchoServer 的连接。 它在标准输入流上读取用户的输入，然后通过将文本写入 Socket 来将该文本转发到 EchoServer。 EchoServer 通过 Socket 将输入回显给 EchoClient。 EchoClient 读取并显示从服务器传回给它的数据。 EchoServer 可以通过 Virtual Threads 同时为多个客户端提供服务，每个客户端连接一个线程。

```java
public class EchoServer {

    public static void main(String[] args) throws IOException {
        if (args.length != 1) {
            System.err.println("Usage: java EchoServer <port>");
            System.exit(1);
        }

        int portNumber = Integer.parseInt(args[0]);
        try (
            ServerSocket serverSocket = new ServerSocket(Integer.parseInt(args[0]));
        ) {                
            while (true) {
                Socket clientSocket = serverSocket.accept();
                // Accept incoming connections
                // Start a service thread
                Thread.ofVirtual().start(() -> {
                    try (
                        PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true);
                        BufferedReader in = new BufferedReader(
                            new InputStreamReader(clientSocket.getInputStream()));
                    ) {
                        String inputLine;
                        while ((inputLine = in.readLine()) != null) {
                            System.out.println(inputLine);
                            out.println(inputLine);
                        }
                    } catch (IOException e) { 
                        e.printStackTrace();
                    }
                });
            }
        } catch (IOException e) {
            System.out.println("Exception caught when trying to listen on port "
                + portNumber + " or listening for a connection");
            System.out.println(e.getMessage());
        }
    }
}

public class EchoClient {

    public static void main(String[] args) throws IOException {
        if (args.length != 2) {
            System.err.println("Usage: java EchoClient <hostname> <port>");
            System.exit(1);
        }
        String hostName = args[0];
        int portNumber = Integer.parseInt(args[1]);
        try (
            Socket echoSocket = new Socket(hostName, portNumber);
            PrintWriter out = new PrintWriter(echoSocket.getOutputStream(), true);
            BufferedReader in =
                new BufferedReader(
                    new InputStreamReader(echoSocket.getInputStream()));
        ) {
            BufferedReader stdIn =
                new BufferedReader(
                    new InputStreamReader(System.in));
            String userInput;
            while ((userInput = stdIn.readLine()) != null) {
                out.println(userInput);
                System.out.println("echo: " + in.readLine());
                if (userInput.equals("bye")) break;
            }
        } catch (UnknownHostException e) {
            System.err.println("Don't know about host " + hostName);
            System.exit(1);
        } catch (IOException e) {
            System.err.println("Couldn't get I/O for the connection to " + hostName);
            System.exit(1);
        } 
    }
}
```

## Virtual Threads 的调度和固定（pin）

OS调度 platform threads 何时运行。 但是，Java 运行时会调度 virtual threads 的运行时间。 当Java运行时调度 virtual threads 时，它会在 platform threads 上分配或安装 virtual threads，然后OS照常调度该 platform threads。 该 platform threads 称为 carrier 载体。运行一些代码后，virtual threads 可以从其 carrier 上卸载。 这通常发生在 virtual threads 执行阻塞 I/O 操作时。Virtual threads 从其 carrier 上卸载后，carrier 就空闲了，这意味着Java运行时调度程序可以在其上挂载不同的 virtual threads

当 virtual thread 固定到其 carrier 时，无法在阻塞操作期间卸载该 virtual thread。Virtual threads 在以下情况下被固定

1. Virtual threads 在同步块或方法内运行代码

2. Virtual threads 运行 native 方法或 foreign 函数（请参阅外部函数和内存 API）

固定不会使应用程序不正确，但可能会妨碍其可扩展性。 尝试通过修改频繁运行的同步块或方法并使用 java.util.concurrent.locks.ReentrantLock 给潜在的长 I/O 操作加锁来避免频繁且长期的固定

# Debugging Virtual Threads

Virtual Threads 仍然是 thread, debugger 可以像 platform thread 一样单步调试它们。Java Flight Recorder 和 jcmd 工具具有附加功能，可帮助您观察应用程序中的 Virtual Threads

## Virtual Threads 的 Java Flight Recorder 事件

Java Flight Recorder (JFR) 可以发出与 Virtual Threads 相关的以下事件：

- jdk.VirtualThreadStart 和 jdk.VirtualThreadEnd 指示 Virtual Threads 何时开始和结束。 默认情况下禁用这些事件。

- jdk.VirtualThreadPinned 指示 Virtual Threads 被固定（并且其载体线程未释放）的时间超过阈值持续时间。 默认情况下启用此事件，阈值为 20 毫秒。

- jdk.VirtualThreadSubmitFailed 表示启动或取消停放 Virtual Threads 失败，可能是由于资源问题。 停放 Virtual Threads 会释放底层承载线程以执行其他工作，而取消停放 Virtual Threads 会安排其继续。 该事件默认启用。

通过 JDK Mission Control 或使用自定义 JFR 配置启用事件 jdk.VirtualThreadStart 和 jdk.VirtualThreadEnd，如 Java 平台标准版 Flight Recorder API 程序员指南中的 Flight Recorder 配置中所述。

要打印这些事件，请运行以下命令，其中recording.jfr 是录制文件的文件名：

jfr print --events jdk.VirtualThreadStart,jdk.VirtualThreadEnd,jdk.VirtualThreadPinned,jdk.VirtualThreadSubmitFailed recording.jfr

## 查看 jcmd 线程 dump 中的 virtual threads

您可以以纯文本和 JSON 格式创建线程dump：

jcmd <PID> Thread.dump_to_file -format=text <file>

jcmd <PID> Thread.dump_to_file -format=json <file>

jcmd 线程转储列出了网络 I/O 操作中被阻塞的 virtual threads 以及 ExecutorService 接口创建的 virtual threads。 它不包括对象地址、锁、JNI 统计信息、堆统计信息以及传统线程dump中出现的其他信息

# Virtual Threads 采用指南

Virtual Threads 是由 Java 运行时而不是OS实现的 Java 线程。Virtual Threads 和传统线程（我们称之为 Platform Threads）之间的主要区别在于，我们可以轻松地在同一个 Java 进程中运行大量活动 Virtual Threads，甚至数百万个。Virtual Threads 的数量众多，赋予了 Virtual Threads 强大的力量：通过允许服务器同时处理更多请求，它们可以更有效地运行以 thread-per-request 风格编写的服务器应用程序，从而提高吞吐量并减少硬件浪费。

由于 Virtual Threads 是 java.lang.Thread 的实现，并且遵守自 Java SE 1.0 以来指定 java.lang.Thread 的相同规则，因此开发人员无需学习新概念即可使用它们。 然而，由于无法生成大量 Platform Threads （多年来 Java 中唯一可用的线程实现），已经产生了旨在应对其高成本的实践。 这些做法在应用于 Virtual Threads 时会适得其反，必须摒弃。 此外，成本上的巨大差异提供了一种新的思考线程的方式，而这些线程一开始可能是陌生的。

本指南无意全面涵盖 Virtual Threads 的每个重要细节。 其目的只是提供一套介绍性指南，以帮助那些希望开始使用 Virtual Threads 的人充分利用它们。

## 使用阻塞 I/O API 以 thread-per-request 的方式编写简单的同步代码

Virtual Threads 可以显著提高以 thread-per-request 风格编写的服务器的吞吐量（而不是延迟）。 在这种风格中，服务器专用一个线程在整个持续时间内处理每个传入请求。 它至少专用一个线程，因为在处理单个请求时，您可能希望使用更多线程来同时执行某些任务。

阻塞 Platform Threads 的成本很高，因为它保留了线程（一种相对稀缺的资源），而它没有做太多有意义的工作。 因为 Virtual Threads 可能很丰富，所以阻塞它们是廉价的并且值得鼓励。 因此，您应该以简单的同步风格编写代码并使用阻塞 I/O API

例如，以下以非阻塞异步风格编写的代码不会从 Virtual Threads 中受益太多

```java
CompletableFuture.supplyAsync(info::getUrl, pool)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofString()))
   .thenApply(info::findImage)
   .thenCompose(url -> getBodyAsync(url, HttpResponse.BodyHandlers.ofByteArray()))
   .thenApply(info::setImageData)
   .thenAccept(this::process)
   .exceptionally(t -> { t.printStackTrace(); return null; });
```

另一方面，以下以同步风格编写并使用简单阻塞 IO 的代码将受益匪浅

```java
try {
   String page = getBody(info.getUrl(), HttpResponse.BodyHandlers.ofString());
   String imageUrl = info.findImage(page);
   byte[] data = getBody(imageUrl, HttpResponse.BodyHandlers.ofByteArray());   
   info.setImageData(data);
   process(info);
} catch (Exception ex) {
   t.printStackTrace();
}
```

此类代码也更容易在调试器中调试、在分析器中分析或通过线程dump进行观察。要观察 virtual thread，请使用 jcmd 命令创建线程dump：

`jcmd <pid> Thread.dump_to_file -format=json <文件>`

以这种方式编写的堆栈越多， virtual threads 的性能和可观察性就越好。 以其他风格编写的程序或框架，如果每个任务没有专用一个线程，则不应期望从  virtual threads 中获得显著的好处。 避免将同步、阻塞代码与异步框架混合

## 将每个并发任务表示为一个 Virtual Thread, 不池化 Virtual Threads

关于 Virtual Threads 最难理解的事情是，虽然它们具有与 Platform Threads 相同的行为，但它们不应该代表相同的程序概念

 Platform Threads 稀缺，因此是宝贵的资源。 宝贵的资源需要管理，管理 Platform Threads 最常见的方法是使用线程池。 然后您需要回答的一个问题是，池中应该有多少个线程？

但 Virtual Threads 非常丰富，因此每个 Virtual Threads 不应代表某些共享的、池化的资源，而应代表一个任务。 线程从托管资源转变为应用程序域对象。 我们应该有多少个 Virtual Threads 的问题变得显而易见，就像我们应该使用多少个字符串在内存中存储一组用户名的问题一样显而易见：在您的应用程序中，Virtual Threads 的数量始终等于并发任务的数量

将 n 个平台线程转换为 n 个 Virtual Threads 不会产生什么好处；相反，它是需要转换的任务

要将每个应用程序任务表示为一个线程，请不要使用共享线程池执行器，如下例所示

```java
Future<ResultA> f1 = sharedThreadPoolExecutor.submit(task1);
Future<ResultB> f2 = sharedThreadPoolExecutor.submit(task2);
// ... use futures
```

相反，请使用 Virtual Threads 执行器，如下例所示

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
   Future<ResultA> f1 = executor.submit(task1);
   Future<ResultB> f2 = executor.submit(task2);
   // ... use futures
}
```

该代码仍然使用 ExecutorService，但从 Executors.newVirtualThreadPerTaskExecutor() 返回的 service 不使用线程池。 相反，它为每个提交的任务创建一个新的 Vritual Thread

此外，ExecutorService 本身是轻量级的，我们可以像创建任何简单对象一样创建一个新的。这使我们能够依赖新添加的 ExecutorService.close() 方法和 try-with-resources 构造。 在 try 块末尾隐式调用的 close 方法将自动等待提交给 ExecutorService 的所有任务（即 ExecutorService 生成的所有 Virtual Threads）终止

对于扇出（fanout）场景来说，这是一种特别有用的模式，在这种场景中，您希望同时对不同的服务执行多个传出调用，如下例所示

```java
void handle(Request request, Response response) {
    var url1 = ...
    var url2 = ...

    try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
        var future1 = executor.submit(() -> fetchURL(url1));
        var future2 = executor.submit(() -> fetchURL(url2));
        response.send(future1.get() + future2.get());
    } catch (ExecutionException | InterruptedException e) {
        response.fail(e);
    }
}

String fetchURL(URL url) throws IOException {
    try (var in = url.openStream()) {
        return new String(in.readAllBytes(), StandardCharsets.UTF_8);
    }
}
```

如上所示，您应该为即使是小型、短期的并发任务创建一个新的 Vritual Thread

为了获得更多帮助编写扇出（fanout）模式和其他常见并发模式，并具有更好的可观察性，请使用结构化并发（structured concurrency）

根据经验，如果您的应用程序从未拥有 10,000 个或更多 Vritual Threads，则它不太可能从 Vritual Threads 中受益。 要么它的负载太轻而需要更高的吞吐量，要么您没有向 Vritual Threads 表示足够多的任务

## 用 Semaphores 限制并发

有时需要限制某个操作的并发数。 例如，某些外部服务可能无法处理超过 10 个并发请求。 由于 Platform Threads 是一种宝贵的资源，通常在池中进行管理，因此线程池已经变得如此普遍，以至于它们被用于限制并发的目的，如下例所示

```java
ExecutorService es = Executors.newFixedThreadPool(10);
...
Result foo() {
    try {
        var fut = es.submit(() -> callLimitedService());
        return f.get();
    } catch (...) { ... }
}
```

此示例确保有限服务最多有 10 个并发请求

但限制并发只是线程池操作的副作用。 池旨在共享稀缺资源，而 Virtual Threads 并不稀缺，因此永远不应该池化！

使用 Virtual Threads 时，如果要限制访问某些服务的并发性，则应该使用专门为此目的设计的构造：Semaphore 类。 下面的例子演示了这个类

```java
Semaphore sem = new Semaphore(10);
...
Result foo() {
    sem.acquire();
    try {
        return callLimitedService();
    } finally {
        sem.release();
    }
}
```

调用 foo 的线程将被阻塞，因此一次只有 10 个线程可以取得进展，而其他线程将不受阻碍地继续自己的业务

简单地使用信号量阻塞某些 Virtual Threads 可能看起来与将任务提交到固定线程池有很大不同，但事实并非如此。 将任务提交到线程池会将它们排队以供稍后执行，但内部信号量（或与此相关的任何其他阻塞同步构造）会创建一个在其上阻塞的线程队列，该队列镜像等待池线程执行的任务队列来执行。 因为 Virtual Threads 是任务，所以结果结构是等效的

![线程池与信号量的比较](https://docs.oracle.com/en/java/javase/21/core/img/java-core-libraries-virtual-threads-thread-pool-and-semaphore.png)

尽管您可以将 Platform Threads 池视为处理从队列中提取的任务的工作人员，并将 Virtual Threads 视为任务本身，在它们可以继续之前被阻塞，但计算机中的底层表示实际上是相同的。 认识排队任务和阻塞线程之间的等效性将帮助您充分利用 Virtual Threads

数据库连接池本身充当信号量。 连接池限制为十个连接将阻止第十一个线程尝试获取连接。 无需在连接池之上添加额外的信号量

## 不要在线程局部变量中缓存昂贵的可重用对象

Virtual Threads 支持线程局部变量，就像平台线程一样。 有关详细信息，请参阅线程局部变量。 通常，线程局部变量用于将一些特定于上下文的信息与当前运行的代码关联起来，例如当前事务和用户ID。 对于 Virtual Threads 来说，线程局部变量的使用是完全合理的。 但是，请考虑使用更安全、更有效的范围值。 有关详细信息，请参阅范围值。

线程局部变量的另一种用途与 Virtual Threads 根本上是不一致的：缓存可重用对象。 这些对象的创建成本通常很高（并且消耗大量内存），并且是可变的，并且不是线程安全的。 它们被缓存在线程局部变量中，以减少它们实例化的次数以及它们在内存中的实例数量，但它们可以被线程上不同时间运行的多个任务重用。

例如，SimpleDateFormat 的实例创建成本很高，而且不是线程安全的。 出现的一种模式是将此类实例缓存在 ThreadLocal 中，如下例所示

```java
static final ThreadLocal<SimpleDateFormat> cachedFormatter = ThreadLocal.withInitial(SimpleDateFormat::new);

void foo() {
  ...
        cachedFormatter.get().format(...);
        ...
}
```

仅当线程（以及因此在线程本地缓存的昂贵对象）被多个任务共享和重用时（就像平台线程被池化时的情况一样），这种缓存才有用。 许多任务在线程池中运行时可能会调用 foo，但由于池中仅包含几个线程，因此该对象只会被实例化几次（每个池线程一次）并被缓存和重用。

但是，Virtual Threads 永远不会被池化，也不会被不相关的任务重用。 因为每个任务都有自己的 Virtual hreads ，所以每次从不同任务调用 foo 都会触发新 SimpleDateFormat 的实例化。 而且，由于可能有大量的 Virtual Threads 同时运行，昂贵的对象可能会消耗相当多的内存。 这些结果与线程本地缓存想要实现的结果恰恰相反。

没有提供单一的通用替代方案，但对于 SimpleDateFormat，您应该将其替换为 DateTimeFormatter，DateTimeFormatter 是不可变的，因此单个实例可以由所有线程共享

```java
static final DateTimeFormatter formatter = DateTimeFormatter….;

void foo() {
  ...
        formatter.format(...);
        ...
}
```

请注意，使用线程局部变量来缓存共享的昂贵对象有时是由异步框架在幕后完成的，其隐含的假设是它们由极少数池线程使用。 这就是为什么混合 Virtual Threads 和异步框架不是一个好主意的原因之一：对方法的调用可能会导致在本来要缓存和共享的线程局部变量中实例化昂贵的对象

## 避免长时间和频繁的固定

当前 Virtual Threads 实现的一个限制是，在同步块或方法内执行阻塞操作会导致 JDK 的 Virtual Threads 调度程序阻塞宝贵的OS线程，而如果阻塞操作是在同步块之外完成则不会，我们称这种情况为“固定”。 如果阻塞操作既长期又频繁，则固定可能会对服务器的吞吐量产生不利影响。 保护短期操作（例如内存中操作）或使用同步块或方法的不频繁操作应该不会产生不利影响。

为了检测可能有害的固定实例，（JDK Flight Recorder (JFR) 在固定阻塞操作时发出 jdk.VirtualThreadPinned 线程；默认情况下，当操作时间超过 20 毫秒时启用此事件

或者，您可以使用系统属性 jdk.tracePinnedThreads 在线程被固定时阻塞时发出堆栈跟踪。 使用选项 -Djdk.tracePinnedThreads=full 运行会在线程被固定时阻塞时打印完整的堆栈跟踪，突出显示本机帧和持有监视器的帧。 使用选项 -Djdk.tracePinnedThreads=short 运行将输出限制为仅有问题的帧。

如果这些机制检测到固定既长期又频繁的地方，请在这些特定地方将同步的使用替换为 ReentrantLock（同样，无需在保护短期或不频繁操作的地方替换同步）。 以下是长期且频繁使用同步块的示例。

```java
synchronized(lockObj) {
    frequentIO();
}
```

您可以将其替换为以下内容

```java
lock.lock();
try {
    frequentIO();
} finally {
    lock.unlock();
}
```