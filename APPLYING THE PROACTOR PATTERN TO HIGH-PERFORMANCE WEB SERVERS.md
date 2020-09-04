---
typora-copy-images-to: ..\..\markdownimg
---

# APPLYING THE PROACTOR PATTERN TO HIGH-PERFORMANCE WEB SERVERS

## ABSTRACT 摘要

Modern operating systems provide multiple concurrency mechanisms to develop high-performance Web servers. Synchronous multi-threading is a popular mechanism for developing Web servers that must perform multiple operations simultaneously to meet their performance requirements. In addition, an increasing number of operating systems support asynchronous mechanisms that provide the benefits of concurrency, while alleviating much of the performance overhead of synchronous multi-threading

现代操作系统提供了多种并发机制来开发高性能Web服务器。同步多线程是开发必须同时执行多个操作以满足性能需求的Web服务器的流行机制。此外，越来越多的操作系统支持提供并发优势的异步机制，同时减轻了同步多线程的大部分性能开销

This paper provides two contributions to the study of high-performance Web servers. First, it examines how synchronous and asynchronous event dispatching mechanisms impact the design and performance of JAWS, which is our high-performance Web server framework. The results reveal significant performance improvements when a proactive concurrency model is used to combine lightweight concurrency with asynchronous event dispatching.

本文对高性能Web服务器的研究提供了两个贡献。首先，本文将研究同步和异步事件分派机制如何影响我们的高性能Web服务器框架JAWS的设计和性能。结果表明，使用主动并发模型将轻量级并发与异步事件分派相结合，可以显著提高性能。

In general, however, the complexity of the proactive concurrency model makes it harder to program applications that can utilize asynchronous concurrency mechanisms effectively. Therefore, the second contribution of this paper describes how to reduce the software complexity of asynchronous concurrent applications by applying the Proactor pattern. This pattern describes the steps required to structure object-oriented applications that seamlessly combine concurrency with asynchronous event dispatching. The Proactor pattern simplifies concurrent programming and improves performance by allowing concurrent application to have multiple operations running simultaneously without requiring a large number of threads.

然而，一般来说，主动并发模型的复杂性使得编程能够有效利用异步并发机制的应用程序变得更加困难。
因此，本文的第二个贡献描述了如何通过应用Proactor模式来降低异步并发应用程序的软件复杂度。此模式描述了构建无缝结合并发性与异步事件分派的面向对象应用程序所需的步骤。Proactor模式允许并发应用程序在不需要大量线程的情况下同时运行多个操作，从而简化了并发编程并提高了性能。

## INTRODUCTION 引入

Computing power and network bandwidth on the Internet has increased dramatically over the past decade. High-speed networks (such as ATM and Gigabit Ethernet) and highperformance I/O subsystems (such as RAID) are becoming ubiquitous. In this context, developing scalable Web servers that can exploit these innovations remains a key challenge for developers. Thus, it is increasingly important to alleviate common Web server bottlenecks, such as inappropriate choice of concurrency and dispatching strategies, excessive filesystem access, and unnecessary data copying.

在过去的十年中，因特网的计算能力和网络带宽有了显著的提高。高速网络(如ATM和千兆以太网)和高性能I/O子系统(如RAID)正变得无处不在。在这种情况下，开发能够利用这些创新的可伸缩Web服务器对开发人员来说仍然是一个关键挑战。因此，缓解常见的Web服务器瓶颈(如并发和分派策略的不适当选择、过多的文件系统访问和不必要的数据复制)变得越来越重要。

Our research vehicle for exploring the performance impact of applying various Web server optimization techniques is the JAWS Adaptive Web Server (JAWS) [1]. JAWS is both an adaptive Web server and a development framework for Web servers that runs on multiple OS platforms including Win32, most versions of UNIX, and MVS Open Edition.

我们研究的工具是JAWS Adaptive Web服务器，JAWS Adaptive Web服务器可以帮助研究，应用各种Web服务器优化技术对性能的影响[1]。JAWS既是一个自适应Web服务器，也是一个Web服务器的开发框架，可在多个操作系统平台上运行，包括Win32、大多数版本的UNIX和MVS开放版。

Our experience [2] building Web servers on multiple OS platforms demonstrates that the effort required to optimize performance can be simplified significantly by leveraging OSspecific features. For example, an optimized file I/O system that automatically caches open files in main memory via mmap greatly reduces latency on Solaris. Likewise, support for asynchronous event dispatching on Windows NT can substantially increase server throughput by reducing context switching and synchronization overhead incurred by multithreading.

[2]在多个OS平台上构建Web服务器的经验表明，通过利用OSspecific特性，可以显著简化优化性能所需的工作。
例如，通过mmap自动在主内存中缓存打开的文件的优化文件I/O系统极大地降低了Solaris上的延迟。类似地，Windows NT上对异步事件分派的支持可以通过减少多线程引起的上下文切换和同步开销来显著提高服务器吞吐量。

Unfortunately, the increase in performance obtained through the use of asynchronous event dispatching on existing operating systems comes at the cost of increased software complexity. Moreover, this complexity is further compounded when asynchrony is coupled with multi-threading. This style of programming, i.e., proactive programming, is relatively unfamiliar to many developers accustomed to the synchronous event dispatching paradigm. This paper describes how the Proactor pattern can be applied to improve both the performance and the design of high-performance communication applications, such as Web servers

不幸的是，通过在现有操作系统上使用异步事件分派而获得的性能提高是以增加软件复杂性为代价的。而且，当异步与多线程耦合时，这种复杂性会进一步加剧。这种编程风格，即主动编程，对于习惯于同步事件分派范例的许多开发人员来说是相对陌生的。本文描述了如何应用Proactor模式来提高高性能通信应用程序(如Web服务器)的性能和设计

A pattern represents a recurring solution to a software development problem within a particular context [3]. Patterns identify the static and dynamic collaborations and interactions between software components. In general, applying patterns to complex object-oriented concurrent applications can significantly improve software quality, increase software maintainability, and support broad reuse of components and architectural designs [4]. In particular, applying the Proactor pattern to JAWS simplifies asynchronous application development by structuring the demultiplexing of completion events and the dispatching of their corresponding completion routines.

模式表示在特定上下文[3]中对软件开发问题的重复出现的解决方案。模式识别软件组件之间的静态和动态协作和交互。一般来说，将模式应用于复杂的面向对象并发应用程序可以显著提高软件质量，提高软件的可维护性，并支持组件和架构设计的广泛重用。特别地，将Proactor模式应用到JAWS可以简化异步应用程序开发，方法是构造完成事件的多路分解和相应的完成例程的分派。

The remainder of this paper is organized as follows: Section 2 provides an overview of the JAWS server framework design; Section 3 discusses alternative event dispatching strategies and their performance impacts; Section 4 explores how to leverage the gains of asynchronous event dispatching through application of the Proactor pattern; and Section 5 presents concluding remarks.

本文的其余部分组织如下:第2节概述了JAWS服务器框架的设计;第3节讨论了备选事件调度策略及其性能影响;
第4节探讨了如何通过应用Proactor模式来利用异步事件分派的好处;第5部分是结束语。

## JAWS FRAMEWORK OVERVIEW

Figure 1 illustrates the major structural components and design patterns that comprise the JAWS framework [1]. JAWS is designed to allow the customization of various Web server strategies in response to environmental factors. These factors include static factors (e.g., number of available CPUs, support for kernel-level threads, and availability of asynchronous I/O in the OS), as well as dynamic factors (e.g., Web traffic patterns and workload characteristics). 

图1说明了组成JAWS框架[1]的主要结构组件和设计模式。JAWS旨在允许定制各种Web服务器策略，以响应环境因素。这些因素包括静态因素(例如，可用cpu的数量，对内核级线程的支持，以及操作系统中异步I/O的可用性)，以及动态因素(例如，Web流量模式和工作负载特征)。

![image-20200831121351900](E:\markdownimg\image-20200831121351900.png)

JAWS is structured as a framework of frameworks. The overall JAWS framework contains the following components and frameworks: an Event Dispatcher, Concurrency Strategy, I/O Strategy, Protocol Pipeline, Protocol Handlers, and Cached Virtual Filesystem. Each framework is structured as a set of collaborating objects implemented using components in ACE [5]. The collaborations among JAWS components and frameworks are guided by a family of patterns, which are listed along the borders in Figure 1. An outline of the key frameworks, components, and patterns in JAWS is presented below; Section 4 then focuses on the Proactor pattern in detail.1 

JAWS是一个框架的框架。整个JAWS框架包含以下组件和框架:事件分派器、并发策略、I/O策略、协议管道、协议处理程序和缓存的虚拟文件系统。每个框架都被构造成一组使用ACE[5]中的组件实现的协作对象。JAWS组件和框架之间的协作由一系列模式指导，这些模式在图1的边框中列出。JAWS中的关键框架、组件和模式的概述如下;
然后，第4节将详细介绍Proactor模式

**Event Dispatcher**: This component is responsible for coordinating JAWS’ Concurrency Strategy with its I/O Strategy. The passive establishment of connection events with Web clients follows the Acceptor pattern [6]. New incoming HTTP request events are serviced by a concurrency strategy. As events are processed, they are dispatched to the Protocol Handler, which is parameterized by an I/O strategy. JAWS ability to dynamically bind to a particular concurrency strategy and I/O strategy from a range of alternatives follows the Strategy pattern [3].

**事件分派器**:此组件负责协调JAWS的并发策略与其I/O策略。Web客户端连接事件的被动建立遵循接受者模式[6]。
新的传入HTTP请求事件由并发策略提供服务。在处理事件时，它们被分配到协议处理程序，该处理程序由I/O策略参数化。JAWS能够动态绑定到特定的并发策略，而来自一系列备选方案的I/O策略遵循策略模式[3]。

**Concurrency Strategy**: This framework implements concurrency mechanisms (such as single-threaded, thread-perrequest, or thread pool) that can be selected adaptively at run-time using the State pattern [3] or pre-determined at initialization-time. The Service Configurator pattern [7] is used to configure a particular concurrency strategy into a Web server at run-time. When concurrency involves multiple threads, the strategy creates protocol handlers that follow the Active Object pattern [8].

**并发策略**:这个框架实现了并发机制(如单线程、线程请求或线程池)，这些机制可以被自适应地选择使用状态模式[3]或预先确定的运行时初始化时间。服务配置器模式[7]是用于将特定的并发策略配置为运行时的Web服务器。当并发性涉及多个线程时，该策略创建随后的协议处理程序活动对象模式[8]。

**I/O Strategy**: This framework implements various I/O mechanisms, such as asynchronous, synchronous and reactive I/O. Multiple I/O mechanisms can be used simultaneously. In JAWS, asynchronous I/O is implemented using the Proactor pattern [9], while reactive I/O is accomplished through the Reactor pattern [10]. These I/O strategies may utilize the Memento [3] and Asynchronous Completion Token [11] patterns to capture and externalize the state of a request so that it can be restored at a later time.

**I/O策略**:该框架实现了各种I/O机制，如异步、同步和响应式I/O。可以同时使用多个I/O机制。在JAWS，异步I/O是使用Proactor实现的模式[9]，而无反应I/O是通过反应器模式[10]。这些I/O策略可以利用记忆体[3]和异步完成令牌[11]模式捕获并外部化请求的状态，以便它可以在以后的时间恢复。

**Protocol Handler**: This framework allows system developers to apply the JAWS framework to a variety of Web system applications. A Protocol Handler is parameterized by a concurrency strategy and an I/O strategy. These strategies are decoupled from the protocol handler using the Adapter pattern [3]. In JAWS, this component implements the parsing and handling of HTTP/1.0 request methods. The abstraction allows for other protocols (such as HTTP/1.1, DICOM, and SFP [12]) to be incorporated easily into JAWS. To add a new protocol, developers simply write a new Protocol Handler implementation, which is then configured into the JAWS framework.

协议处理程序:该框架允许系统开发人员将JAWS框架应用于各种Web系统应用程序。协议处理程序由并发策略和I/O策略参数化。这些策略是使用适配器模式[3]与协议处理程序解耦。在JAWS中，该组件实现解析和处理HTTP/1.0请求方法。抽象允许使用其他协议(如HTTP/1.1、DICOM和SFP[12])可以很容易地并入钳口。
要添加一个新的协议，开发人员只需编写一个新的协议处理程序实现，然后将其配置到JAWS框架中。

Protocol Pipeline: This framework allows filter operations to be incorporated easily with the data being processed by the Protocol Handler. This integration is achieved by employing the Adapter pattern. Pipelines follow the Pipes and Filters pattern [13] for input processing. Pipeline components can be linked dynamically at run-time using the Service Configurator pattern.

协议管道:这个框架允许过滤操作轻松地与协议处理程序正在处理的数据合并。这种集成是通过使用适配器模式实现的。管道遵循管道和过滤器模式[13]进行输入处理。管道组件可以在运行时使用服务配置器模式进行动态链接。

Cached Virtual Filesystem: This component improves Web server performance by reducing the overhead of filesystem access. Various caching strategies, such as LRU, LFU, Hinted, and Structured, can be selected following the Strategy pattern [3]. This allows different caching strategies to be profiled and selected based on their performance. Moreover, optimal strategies to be configured statically or dynamically using the Service Configurator pattern. The cache for each Web server is instantiated using the Singleton pattern [3].

缓存的虚拟文件系统:该组件改进Web通过减少文件系统访问的开销来提高服务器性能。各种缓存策略，比如LRU, LFU，和结构化的，可以选择遵循[3]的策略模式。这允许分析不同的缓存策略根据他们的表现来选择。此外,优方法静态或动态配置的策略服务配置器模式。每个Web服务器的缓存使用单例模式[3]实例化。

Tilde Expander: This component is another cache component that uses a perfect hash table [14] that maps abbreviated user login names (e.g., schmidt) to user home directories (e.g., /home/cs/faculty/schmidt). When personal Web pages are stored in user home directories, and user directories do not reside in one common root, this component substantially reduces the disk I/O overhead required to access a system user information file, such as /etc/passwd. By virtue of the Service Configurator pattern, the Tilde Expander can be unlinked and relinked dynamically into the server when a new user is added to the system.

波浪扩展器:该组件是另一个缓存组件，它使用完美的哈希表[14]，映射缩略用户登录名称(例如,施密特)用户的主目录(例如,/home/cs/faculty/schmidt)。当个人Web页面存储在用户主目录中，而用户目录并不驻留在一个公共根目录(即此组件)中大大减少了访问所需的磁盘I/O开销系统用户信息文件，如/etc/passwd.通过借助服务配置器模式，波浪扩展器何时可以解除链接并重新链接到服务器将向系统添加一个新用户。

Our previous work on high-performance Web servers has focused on (1) the design of the JAWS framework [1] and (2) detailed measurements on the performance implications of alternative Web server optimization techniques [2]. In our earlier work, we discovered that a concurrent proactive Web server can achieve substantial performance gains [15].

我们之前在高性能Web服务器上的工作主要集中在(1)JAWS框架[1]的设计和(2)对替代Web服务器优化技术[2]的性能影响的详细测量。在我们早期的工作中，我们发现一个并发的主动Web服务器可以实现[15]的显著性能提升。

This paper focuses on a previously unexamined point in the high-performance Web server design space: the application of the Proactor pattern to simplify Web server software development, while maintaining high-performance. Section 3 motivates the need for concurrent proactive architectures by analyzing empirical benchmarking results of JAWS and outlining the software design challenges involved in developing proactive Web servers. Section 4 then demonstrates how these challenges can be overcome by designing the JAWS Web server using the Proactor pattern.

本文主要关注高性能Web服务器设计领域中以前没有研究过的一点:应用Proactor模式来简化Web服务器软件开发，同时保持高性能。第3节通过分析JAWS的经验基准测试结果和概述开发主动Web服务器所涉及的软件设计挑战，激发了对并发主动架构的需求。第4节将演示如何通过使用Proactor模式设计JAWS Web服务器来克服这些挑战。

## CONCURRENCY ARCHITECTURES

Developing a high-performance Web server like JAWS requires the resolution of the following forces:

开发像JAWS这样的高性能Web服务器需要解决以下问题:

- Concurrency: The server must perform multiple client requests simultaneously;  
- 并发性:服务器必须同时执行多个客户端请求;
- Efficiency: The server must minimize latency, maximize throughput, and avoid utilizing the CPU(s) unnecessarily.   
- 效率:服务器必须最小化延迟，最大化吞吐量，并避免不必要地使用CPU。
- Adaptability: Integrating new or improved transport protocols (such as HTTP 1.1 [16]) should incur minimal enhancement and maintenance costs.
- 适应性:集成新的或改进的传输协议(如HTTP 1.1[16])应该带来最小的增强和维护成本。
- Programming simplicity: The design of the server should simplify the use of various concurrency strategies, which may differ in performance on different OS platforms;
- 编程简单:服务器的设计应该简化各种并发策略的使用，这些策略在不同的OS平台上可能会有不同的性能;

The JAWS Web server can be implemented using several concurrency strategies, such as multiple synchronous threads, reactive synchronous event dispatching, and proactive asynchronous event dispatching. Below, we compare and contrast the performance and design impacts of using conventional multi-threaded synchronous event dispatching versus proactive asynchronous event dispatching, using our experience developing and optimizing JAWS as a case-study.

JAWS Web服务器可以使用多种并发策略实现，例如多个同步线程、被动同步事件分派和主动异步事件分派。
下面，我们将以我们开发和优化JAWS的经验作为案例研究，对比使用传统多线程同步事件调度和主动异步事件调度对性能和设计的影响。

## CONCURRENT SYNCHRONOUS EVENTS

Overview: An intuitive and widely used concurrency architecture for implementing concurrent Web servers is to use synchronous multi-threading. In this model, multiple server threads can process HTTP GET requests from multiple clients simultaneously. Each thread performs connection establishment, HTTP request reading, request parsing, and file transfer operations synchronously. As a result, each operation blocks until it completes

概述:用于实现并发Web服务器的直观且广泛使用的并发架构是使用同步多线程。在这个模型中，多个服务器线程可以同时处理来自多个客户端的HTTP GET请求。每个线程同步执行连接建立、HTTP请求读取、请求解析和文件传输操作。因此，每个操作都会阻塞，直到完成为止

The primary advantage of synchronous threading is the simplification of server code. In particular, operations performed by a Web server to service client A’s request are mostly independent of the operations required to service client B’s request. Thus, it is easy to service different requests in separate threads because the amount of state shared between the threads is low, which minimizes the need for synchronization. Moreover, executing application logic in separate threads allows developers to utilize intuitive sequential commands and blocking operations

同步线程的主要优点是服务器代码的简化。具体来说，Web服务器为服务客户机a的请求而执行的操作是
大部分独立于服务客户端的操作B的请求。因此，很容易为不同的请求提供服务分离线程，因为线程之间共享的状态量线程很低，这使得同步的需求最小化。此外，单独执行应用程序逻辑线程允许开发人员使用直观的顺序命令和阻塞操作

Figure 2 shows how JAWS can be configured to use synchronous threads to process multiple clients concurrently. This figure shows a Sync Acceptor object that encapsulates the server-side mechanism for synchronously accepting network connections.

图2显示了如何配置JAWS以使用同步线程并发地处理多个客户机。该图显示了一个同步接受器对象，它封装了用于同步接受网络连接的服务器端机制。

The sequence of steps that each thread uses to service an HTTP GET request using a Thread Pool concurrency model can be summarized as follows: 

每个线程使用线程池并发模型为HTTP GET请求提供服务的步骤序列可以总结如下:

- Each thread synchronously blocks in the Sync Acceptor waiting for a client connection request;
- 每个线程同步阻塞在同步接受等待客户端连接请求;
- A client connects to the server, and the Sync Acceptor selects one of the waiting threads to accept
  the connection;
- 客户端连接到服务器，同步接收器选择一个等待的线程接受连接;
- The new client’s HTTP request is synchronously read from the network connection by the selected thread;
- 新客户端的HTTP请求被选择的线程从网络连接同步读取;
- The request is parsed;
- 请求被解析;
- The requested file is synchronously read;
- 被请求的文件被同步读取;
- The file is synchronously sent to the client.
- 文件被同步发送到客户端。

![image-20200831140552159](E:\markdownimg\image-20200831140552159.png)

As described above, each concurrently connected client is serviced by a dedicated server thread. The thread completes a requested operation synchronously before servicing other HTTP requests. Therefore, to perform synchronous I/O while servicing multiple clients, JAWS must spawn multiple threads

如上所述，每个并发连接的客户机都由专用服务器线程提供服务。线程在处理其他HTTP请求之前同步完成请求的操作。因此，要在服务多个客户端时执行同步I/O, JAWS必须生成多个线程

Evaluation: Although the synchronous multi-threaded model is intuitive and maps relatively efficiently onto multi-CPU platforms, it has the following drawbacks:

评估:尽管采用同步多线程模型是直观的和相对有效地映射到多cpu平台，它有以下缺点:

The threading policy is tightly coupled to the concurrency policy: The synchronous model requires a dedicated thread for each connected client. A concurrent application may be better optimized by aligning its threading strategy to available resources (such as the number of CPUs) rather than to the number of clients being serviced concurrently;

线程策略与并发策略紧密耦合:同步模型需要一个专用的每个连接的客户端的线程。并发应用程序可能是更好的通过优化调整其线程策略可用资源(例如cpu的数量)，而不是同时提供服务的客户数目;

Increased synchronization complexity: Threading can increase the complexity of synchronization mechanisms necessary to serialize access to a server’s shared resources (such as cached files and logging of Web page hits);

增加同步的复杂性:线程化会增加同步机制的复杂性，而同步机制是序列化访问服务器的共享资源(例如
缓存文件和记录网页点击);

Increased performance overhead: Threading can perform poorly due to context switching, synchronization, and data movement among CPUs [5];

增加的性能开销:可以执行线程由于上下文切换、同步和数据在cpu之间移动而导致的结果很差

Non-portability: Threading may not be available on all OS platforms. Moreover, OS platforms differ widely in terms of their support for preemptive and non-preemptive threads. Consequently, it is hard to build multi-threaded servers that behave uniformly across OS platforms.

不可移植性:线程可能不是在所有OS平台上都可用。此外，操作系统平台在对抢占式线程和非抢占式线程的支持方面差别很大。因此，很难构建跨操作系统平台行为一致的多线程服务器。

As a result of these drawbacks, multi-threading may not always be the most efficient nor the least complex solution to1develop concurrent Web servers. The solution may not be obvious, since the disadvantages may not result in any actual performance penalty except under certain conditions, such as  a particularly high number of long running requests intermixed with rapid requests for smaller files. Therefore, it is important to explore alternative Web server architecture designs, such as the concurrent asynchronous architecture described next.

由于这些缺点，多线程可能并不总是开发并发Web服务器的最有效或最简单的解决方案。解决方案可能并不明显，因为这种缺点可能不会导致任何实际的性能损失，除非在某些情况下，比如大量长时间运行的请求与对较小文件的快速请求混合在一起。因此，研究其他Web服务器体系结构设计很重要，比如下面描述的并发异步体系结构。

## CONCURRENT ASYNCHRONOUS EVENTS

Overview: When the OS platform supports asynchronous operations, an efficient and convenient way to implement a high-performance Web server is to use proactive event dispatching. Web servers designed using this dispatching model handle the completion of asynchronous operations with one or more threads of control.

概述:当操作系统平台支持异步时。一种高效、方便的实现方法高性能Web服务器是使用主动事件调度。Web服务器使用这种分派模型设计用一个或处理异步操作的完成更多的控制线程。

JAWS implements proactive event dispatching by first issuing an asynchronous operation to the OS and registering a callback (which is the Completion Handler) with the Event Dispatcher. 2 This Event Dispatcher notifies JAWS when the operation completes. The OS then performs the operation and subsequently queues the result in a well-known location. The Event Dispatcher is responsible for dequeuing completion notifications and executing the appropriate Completion Handler

JAWS通过首先向OS发出一个异步操作并向事件分派器注册一个回调(即完成处理程序)来实现主动事件分派。操作完成时，这个事件分派器通知JAWS。然后操作系统执行该操作，并随后将结果队列到一个已知的位置。事件分派器负责退出完成通知的队列，并执行适当的完成处理程序

Figures 3 and 4 show how the JAWS Web server configured using proactive event dispatching handles multiple clients concurrently within one or more threads. Figure 3 shows the sequence of steps taken when a client connects to JAWS.

图3和图4显示了使用主动事件分派配置的JAWS Web服务器如何在一个或多个线程中并发地处理多个客户机。
图3显示了客户机连接到JAWS时所采取的一系列步骤。

![image-20200831142312249](E:\markdownimg\image-20200831142312249.png)

- JAWS instructs the Async Acceptor to initiate an asynchronous accept;
- JAWS指示异步接受者发起一个异步接受;
- The Async Acceptor initiates an asynchronous accept with the OS and passes itself as a Completion Handler and a reference to the Event Dispatcher that will be used to notify the Async Acceptor upon
  completion of the asynchronous accept;
- Async Acceptor用OS启动一个异步接受，并将自己作为一个完成处理程序和一个事件分派器的引用传递给Async Acceptor完成异步接收;
- JAWS invokes the event loop of the Event Dispatcher;
- JAWS调用事件调度程序的事件循环;
- The client connects to JAWS;
- 客户端连接到JAWS;
- When the asynchronous accept operation completes, the OS notifies the Event Dispatcher;
- 当异步接受操作完成时，OS会通知事件分派器;
- The Event Dispatcher notifies the Async Acceptor;
- 事件分派器通知异步接受者;
- The Async Acceptor creates an appropriate Protocol Handler;
- 异步接受者创建一个适当的协议处理程序;
- The Protocol Handler initiates an asynchronous operation to read the request data from the client and passes itself as a Completion Handler and a reference to the Event Dispatcher that will be used to notify the Protocol Handler upon completion of the asynchronous read.
- 协议处理程序启动一个异步操作，从客户端读取请求数据，并将自身作为一个完成处理程序和一个事件分派器的引用传递，事件分派器将用于在异步读取完成时通知协议处理程序。

Figure 4 shows the sequence of steps that the JAWS uses to service an HTTP GET request when it is configured using proactive event dispatching. These steps are outlined below:

图4显示了JAWS在使用主动事件分派配置时为HTTP GET请求提供服务的步骤序列。这些步骤概述如下:

![image-20200831144400759](E:\markdownimg\image-20200831144400759.png)

- The client sends an HTTP GET request;
- 客户端发送一个HTTP GET请求;
- The read operation completes and the OS notifies the Event Dispatcher;
- 读取操作完成，操作系统通知事件分派器;
- The Event Dispatcher notifies the Protocol Handler (steps 2 and 3 will repeat until the entire request has been received);
- 事件调度程序通知协议处理程序(步骤2和步骤3将重复，直到整个请求被接收);
- The Protocol Handler parses the request;
- 协议处理程序解析请求;
- The Protocol Handler initiates an asynchronous TransmitFile operation to read the file data and write it out to the client connection. When doing so, it passes itself as a Completion Handler and a reference to the Event Dispatcher that will be used to notify the Protocol Handler upon completion of the
  asynchronous operation;
- 协议处理程序启动一个异步TransmitFile操作来读取文件数据并将其写入客户端连接。当这样做时，它将自己作为一个完成处理程序和一个事件分派器的引用传递，事件分派器将用于在协议处理程序完成时通知协议处理程序异步操作;
- When the write operation completes, the OS notifies the Event Dispatcher;
- 写操作完成后，OS会通知事件分派器;
- The Event Dispatcher then notifies the Completion Handler
- 事件分派器然后通知完成处理程序

Evaluation: The primary advantage of using proactive event dispatching is that multiple operations can be initiated and run concurrently without requiring the application to have as many threads as there are simultaneous I/O operations. The operations are initiated asynchronously by the application and they run to completion within the I/O subsystem of the OS. Once the asynchronous operation is initiated, the thread that started the operation become available to service additional requests

评估:使用前瞻性事件调度的主要优势，可以并发启动和执行多个操作运行，而不需要应用程序因为同时有多个I/O操作，而具有许多线程。应用程序异步启动操作，它们在OS的I/O子系统中运行直至完成。一旦异步操作被启动，线程就可以重新开启，这样就可以再次服务其他请求。

In the proactive example above, for instance, the Event Dispatcher could be single-threaded, which may be desirable on a uniprocessor platform. When HTTP requests arrive, the single Event Dispatcher thread parses the request, reads the file, and sends the response to the client. Since the response is sent asynchronously, multiple responses could potentially be sent simultaneously. Moreover, the synchronous file read could be replaced with an asynchronous file read to further increase the potential for concurrency. If the file read is performed asynchronously, the only synchronous operation performed by a Protocol Handler is the HTTP protocol request parsing.

例如，在上面的主动示例中，事件分派器可以是单线程的，这在单处理器平台上可能是可取的。当HTTP请求到达时，单个事件分派器线程解析请求，读取文件，并将响应发送给客户端。由于响应是异步发送的，因此可能同时发送多个响应。此外，可以将同步文件读取替换为异步文件读取，以进一步提高并发性。如果文件读取是异步执行的，协议处理程序执行的唯一同步操作就是HTTP协议请求解析。

The primary drawback with the proactive event dispatching model is that the application structure and behavior can be considerably more complicated than with the conventional synchronous multi-threaded programming paradigm. In general, asynchronous applications are hard to develop since programmer’s must explicitly retrieve OS notifications when asynchronous events complete. However, completion notifications need not appear in the same order that the asynchronous events were requested. Moreover, combining concurrency with asynchronous events is even harder since the thread that issues an asynchronous request may ultimately handle the completion of an event started by a different thread. The JAWS framework alleviates many of the complexities of concurrent asynchronous event dispatching by applying the Proactor pattern described in Section 4

主动事件调度模型的主要缺点是，应用程序结构和行为可能比传统的同步多线程编程范式复杂得多。一般来说，异步应用程序很难开发，因为程序员必须在异步事件完成时显式地检索OS通知。但是，完成通知的出现顺序不需要与请求异步事件的顺序相同。此外，由于发出异步请求的线程可能最终处理由不同线程启动的事件的完成，因此将并发性与异步事件结合起来就更加困难了。JAWS框架通过应用第4节中描述的Proactor模式，减轻了并发异步事件分派的许多复杂性

## Benchmarking Results  

To gain an understanding of how different concurrency and event dispatching mechanisms impact the performance of Web servers subject to heavy load conditions, JAWS was designed to use both synchronous and proactive I/O and event dispatching. Each one was benchmarked and the performance of each was compared

为了理解不同的并发性和事件分派机制如何在高负载条件下影响Web服务器的性能，JAWS被设计为同时使用同步和主动I/O和事件分派。每一个都是基准测试，并比较各自的表现

### Hardware Testbed  

Our hardware testbed is shown in Figure 5.  

![image-20200831160207370](E:\markdownimg\image-20200831160207370.png)



The testbed consists of two Micron Millennia PRO2 plus workstations. Each PRO2 has 128 MB of RAM and is equipped with 2 Pentium Pro processors. The client machine has a clock speed of 200 MHz, while the server machine runs 180 MHz. In addition, each PRO2 has an ENI-155P-MF-S ATM card made by Efficient Networks, Inc. and is driven by Orca 3.01 driver software. The two workstations were connected via an ATM network running through a FORE Systems ASX-200BX, with a maximum bandwidth of 622 Mbps. However, due to limitations of LAN emulation mode, the peak bandwidth of our testbed is approximately 120 Mbps.

该试验台由两个微米的千年PRO2 plus工作站组成。每个PRO2都有128mb的RAM，并配备了2个奔腾Pro处理器。
客户端机器有200mhz的时钟速度，而服务器机器运行180mhz。此外，每个PRO2有一个eni - 155p - fm - s ATM卡由高效网络公司，并由Orca 3.01驱动软件。这两个工作站通过一个ATM网络连接，该网络通过一个FORE系统ASX-200BX运行，最大带宽为622 Mbps。然而，由于局域网仿真模式的限制，我们的测试台的峰值带宽约为120mbps。

### Software Request Generator  

We used the WebSTONE [17] v2.0 benchmarking software to collect client- and server-side metrics. These metrics included average server throughput, and average client latency. WebSTONE is a standard benchmarking utility, capable of generating load requests that simulate typical Web server file access patterns. Our experiments used WebSTONE to generate loads and gather statistics for particular file sizes to determine the impacts of different concurrency and event dispatching strategies

我们使用WebSTONE[17] 2.0基准测试软件来收集客户端和服务器端指标。这些指标包括平均服务器吞吐量和平均客户机延迟。WebSTONE是一个标准的基准测试工具，能够生成模拟典型Web服务器文件访问模式的负载请求。
我们的实验使用WebSTONE生成负载并收集特定文件大小的统计信息，以确定不同并发性和事件分派策略的影响

The file access pattern used in the tests is shown in Table 1. This table represents actual load conditions on popular servers, based on a study of file access patterns conducted by SPEC

测试中使用的文件访问模式如表1所示。这个表 表示流行服务器上的实际负载条件，它基于SPEC对文件访问模式的研究

![image-20200831163817056](E:\markdownimg\image-20200831163817056.png)

### Experimental Results  

![image-20200831164058604](E:\markdownimg\image-20200831164058604.png)

![image-20200831164119140](E:\markdownimg\image-20200831164119140.png)

![image-20200831164136333](E:\markdownimg\image-20200831164136333.png)

![image-20200831164152495](E:\markdownimg\image-20200831164152495.png)

![image-20200831164207672](E:\markdownimg\image-20200831164207672.png)

The results presented below compare the performance of several different adaptations of the JAWS Web server. We discuss the effect of different event dispatching and I/O models on throughput and latency. Throughput is defined as the average number of bits received per second by the client. A high-resolution timer for throughput measurement was started before the client benchmarking software sent the HTTP request. The high-resolution timer stops just after the connection is closed at the client end. The number of bits received includes the HTML headers sent by the server

下面给出的结果比较了JAWS Web服务器的几种不同适应性的性能。讨论了不同的事件分派和I/O模型对吞吐量和延迟的影响。吞吐量定义为客户端每秒接收的平均位数。在客户端基准测试软件发送HTTP请求之前，启动了一个用于吞吐量测量的高分辨率计时器。高分辨率计时器会在客户端连接关闭后立即停止。接收的比特数包括服务器发送的HTML头

Latency is defined as the average amount of delay in milliseconds seen by the client from the time it sends the request to the time it completely receives the file. It measures how long an end user must wait after sending an HTTP GET request to a Web server, and before the content begins to arrive at the client. The timer for latency measurement is started just before the client benchmarking software sends the HTTP request and stops just after the client receives the first response from the server

延迟被定义为客户端从发送请求到完全接收到文件的平均延迟量(以毫秒为单位)。
它测量终端用户在向Web服务器发送HTTP GET请求之后以及内容开始到达客户机之前必须等待的时间。
延迟度量计时器在客户端基准测试软件发送HTTP请求之前启动，在客户端从服务器接收到第一个响应后停止

The five graphs shown for each of throughput and latency represent different file sizes used in each experiment, 500 bytes through 5 Mbytes by factors of 10. These files sizes represent the spectrum of files sizes benchmarked in our experiments, to discover what impact file size has on performance.

每个吞吐量和延迟所显示的5个图表示每个实验中使用的不同文件大小，从500字节到5兆字节，按10的倍数。
这些文件大小代表了我们实验中以基准测试的文件大小的范围，以了解文件大小对性能的影响。

Throughput Comparisons: Figures 6-10 demonstrate the variance of throughput as the size of the requested file and the server hit rate are systematically increased. As expected, the throughput for each connection generally degrades as the connections per second increases. This stems from the growing number of simultaneous connections being maintained, which decreases the throughput per connection

吞吐量比较:图6-10显示了当请求文件的大小和服务器命中率有系统地增加时吞吐量的差异。正如预期的那样，每个连接的吞吐量通常会随着每秒连接数的增加而降低。这是由于同时维护的连接数量不断增加，从而降低了每个连接的吞吐量

As shown in Figure 8, the throughput of Thread-perRequest can degrade rapidly for smaller files as the connection load increases. In contrast, the throughput of the synchronous Thread Pool implementation degrade more gracefully. The reason for this difference is that Thread-per-Request incurs higher thread creation overhead since a new thread is spawned for each GET request. In contrast, thread creation overhead in the Thread Pool strategy is amortized by pre-spawning threads when the server begins execution

如图8所示，对于较小的文件，随着连接负载的增加，Thread-perRequest的吞吐量会迅速降低。相比之下，同步线程池实现的吞吐量下降得更平稳。产生这种差异的原因是，每个请求的线程会导致更高的线程创建开销，因为每个GET请求都会生成一个新线程。相反，线程池策略中的线程创建开销通过在服务器开始执行时预生成线程来平摊

The results in figures 6-10 illustrate that TransmitFile performs extremely poorly for small files (i.e., <50 Kbytes). Our experiments indicate that the performance of TransmitFile depends directly upon the number of simultaneous requests. We believe that during heavy server loads (i.e., high hit rates), TransmitFile is forced to wait while the kernel services incoming requests. This creates a high number of simultaneous connections, degrading server performance

图6-10中的结果表明，对于小文件(即<50 Kbytes)， TransmitFile的性能非常差。我们的实验表明，TransmitFile的性能直接取决于同时请求的数量。我们认为，在服务器负载较大的情况下(即，高命中率)，当内核服务传入请求时，TransmitFile被迫等待。这将创建大量的并发连接，降低服务器性能

As the size of the file grows, however, TransmitFile rapidly outperforms the synchronous dispatching models. For instance, at heavy loads with the 5 Mbyte file (shown in Figure 10), it outperforms the next closest model by nearly 40%. TransmitFile is optimized to take advantage of Windows NT kernel features, thereby reducing the number of data copies and context switches.

然而，随着文件大小的增长，TransmitFile的性能迅速超过同步分派模型。例如，在使用5mbyte文件的重载情况下(如图10所示)，它的性能比最近的模型高出近40%。TransmitFile被优化以利用Windows NT内核特性，从而减少了数据副本和上下文切换的数量。

Latency Comparisons: Figures 6-10 demonstrate the variance of latency performance as the size of the requested file and the server hit rate increase. As expected, as the connections per second increases, the latency generally increases, as well. This reflects the additional load placed on the server, which reduces its ability to service new client requests.

延迟比较:图6-10显示了延迟性能随请求文件大小和服务器命中率的增加而变化。正如预期的那样，随着每秒连接数的增加，延迟通常也会增加。这反映了服务器上的额外负载，从而降低了其为新客户机请求提供服务的能力。

As before, TransmitFile performs extremely poorly for small files. However, as the file size grows, its latency rapidly improves relative to synchronous dispatching during light loads

和以前一样，对于小文件，TransmitFile的性能非常差。但是，随着文件大小的增长，相对于轻负载期间的同步调度，其延迟会迅速提高

### SUMMARY OF PERFORMANCE RESULTS  

As illustrated in the benchmarking results presented above, there is significant variance in throughput and latency depending on the concurrency and event dispatching mechanisms. For small files, the synchronous Thread Pool strategy provides better overall performance. Under moderate loads, the synchronous event dispatching model provides slightly better latency than the asynchronous model. Under heavy loads and with large file transfers, however, the asynchronous model using TransmitFile provides better quality of service. Thus, under Windows NT, an optimal Web server should adapt itself to either event dispatching and file I/O model, depending on the server’s workload and distribution of file requests.

如上面的基准测试结果所示，根据并发性和事件分派机制，吞吐量和延迟存在显著差异。对于小文件，同步线程池策略提供了更好的总体性能。在中等负载下，同步事件调度模型提供的延迟比异步模型稍微好一些。然而，在重载和大型文件传输情况下，使用TransmitFile的异步模型提供了更好的服务质量。因此，在windows nt下，一个最佳的Web服务器应该根据服务器的工作负载和文件请求的分布，使自己适应事件分派和文件I/O模型。

Despite the potential for substantial performance improvements, it is considerably harder to develop a Web server that manages concurrency using asynchronous event dispatching, compared with traditional synchronous approaches. This is due to the additional details associated with asynchronous programming (e.g. explicitly retrieving OS notifications that may appear in non-FIFO order), and the added complexity of combining the approach with multi-threaded concurrency. Moreover, proactive event dispatching can be difficult to debug since asynchronous operations are often nondeterministic. Our experience with designing and developing a proactive Web server indicates that the Proactor pattern provides an elegant solution to managing these complexities.

尽管有显著性能改进的潜力，但与传统同步方法相比，使用异步事件分派开发管理并发性的Web服务器要困难得多。这是由于与异步编程相关的额外细节(例如显式地检索可能以非fifo顺序出现的OS通知)，以及将该方法与多线程并发相结合所增加的复杂性。此外，由于异步操作通常是不确定的，主动事件分派可能难以调试。我们设计和开发主动Web服务器的经验表明，Proactor模式为管理这些复杂性提供了一种优雅的解决方案。

## THE PROACTOR PATTERN  

In general, patterns help manage complexity by providing insight into known solutions to problems in a particular software domain. In the case of concurrent proactive architectures, the complexity of the additional details of asynchronous programming are compounded by the complexities associated with multi-threaded programming. Fortunately, patterns identified in software solutions to other proactive architectures have yielded the Proactor pattern, which is described below.

通常，模式通过提供对特定软件领域中问题的已知解决方案的洞察来帮助管理复杂性。在并发主动架构的情况下，异步编程附加细节的复杂性由于与多线程编程相关的复杂性而变得更加复杂。幸运的是，在其他主动架构的软件解决方案中确定的模式产生了Proactor模式，如下所述。

### INTENT  

The Proactor pattern supports the demultiplexing and dispatching of multiple event handlers, which are triggered by the completion of asynchronous events. This pattern simplifies asynchronous application development by integrating the demultiplexing of completion events and the dispatching of their corresponding event handlers

Proactor模式支持多个事件处理程序的多路分解和分派，这些事件处理程序是由异步事件的完成触发的。此模式通过集成完成事件的多路分解和相应事件处理程序的分派，简化了异步应用程序开发

### APPLICABILITY  

Use the Proactor pattern when one or more of the following conditions hold:

当具备以下一个或多个条件时，请使用Proactor模式

- An application needs to perform one or more asynchronous operations without blocking the calling thread;
- 应用程序需要在不阻塞调用线程的情况下执行一个或多个异步操作;
- The application must be notified when asynchronous operations complete;
- 必须在异步操作完成时通知应用程序;
- The application needs to vary its concurrency strategy independent of its I/O model;
- 应用程序需要改变其独立于I/O模型的并发策略;
- The application will benefit by decoupling the application-dependent logic from the applicationindependent infrastructure;
- 应用程序将受益于分离依赖于应用程序的逻辑与独立于应用程序的基础结构;
- An application will perform poorly or fail to meet its performance requirements when utilizing either the multithreaded approach or the reactive dispatching approach.
- 当使用多线程方法或反应调度方法时，应用程序的性能会很差，或无法满足其性能要求。

## STRUCTURE AND PARTICIPANTS  

The structure of the Proactor pattern is illustrated in Figure 11 using OMT notation.  

![image-20200904143005233](E:\markdownimg\image-20200904143005233.png)

The key participants in the Proactor pattern include the following:
**Proactive Initiator** (Web server application’s main thread):  

**主动发起者** (Web服务器应用程序主线程):

A Proactive Initiator is any entity in the application that initiates an Asynchronous Operation. The Proactive Initiator registers a Completion Handler and a Completion Dispatcher with a Asynchronous Operation Processor, which notifies it when the operation completes

主动启动程序是应用程序中发起异步操作的任何实体。主动启动程序向异步操作处理器注册一个完成处理程序和一个完成分派程序，当操作完成时，异步操作处理器会通知主动启动程序

**Completion Handler** (the Acceptor and HTTP Handler):

完成处理程序(Acceptor和HTTP处理程序):

The Proactor pattern uses Completion Handler interfaces that are implemented by the application for
Asynchronous Operation completion notification

Proactor模式使用由应用程序实现的完成处理程序接口异步操作完成通知

**Asynchronous Operation** (the methods Async Read, Async Write, and Async Accept):  

**异步操作**(方法异步读，异步写，异步接受):

Asynchronous Operations are used to execute requests (such as I/O and timer operations) on behalf of applications. When applications invoke Asynchronous Operations, the operations are performed without borrowing the application’s thread of control.4 Therefore, from the application’s perspective, the operations are performed asynchronously. When Asynchronous Operations complete, the Asynchronous Operation Processor delegates application notifications to a Completion Dispatcher.

异步操作用于代表应用程序执行请求(如I/O和定时器操作)。当应用程序调用异步操作时，操作的执行不需要借用应用程序的控制线程。因此，从应用程序的角度来看，操作是异步执行的。当异步操作完成时，异步操作处理器将应用程序通知委托给一个完成分派器。

**Asynchronous Operation Processor** (the Operating System):
Asynchronous Operations are run to completion by the Asynchronous Operation Processor.

异步操作由异步操作处理器运行到完成。

This component is typically implemented by the OS.  

该组件通常由操作系统实现。

**Completion Dispatcher** (the Notification Queue):

**完成调度程序**(通知队列):

  The Completion Dispatcher is responsible for calling back to the application’s Completion Handlers when Asynchronous Operations complete. When the Asynchronous Operation Processor completes an asynchronously initiated operation, the Completion Dispatcher performs an application callback on its behalf.

完成分派器负责在异步操作完成时回调应用程序的完成处理程序。当异步操作处理器完成异步启动的操作时，完成分派器代表它执行一个应用程序回调。

![image-20200904155438780](E:\markdownimg\image-20200904155438780.png)

## COLLABORATIONS  协作

There are several well-defined steps that occur for all Asynchronous Operations. At a high level of abstraction, applications initiate operations asynchronously and are notified when the operations complete. Figure 12 shows the following interactions that must occur between the pattern participants:

![image-20200904161434100](E:\markdownimg\image-20200904161434100.png)

对于所有异步操作，都有几个定义良好的步骤。在较高的抽象级别上，应用程序异步地启动操作，并在操作完成时得到通知。图12显示了模式参与者之间必须发生的以下交互:

- Proactive Initiators initiates operation: To perform asynchronous operations, the application initiates the operation on the Asynchronous Operation Processor. For instance, a Web server might ask the OS to transmit a file over the network using a particular socket connection. To request such an operation, the Web server must specify which file and network connection to use. Moreover, the Web server must specify (1) which Completion Handler to notify when the operation completes and (2) which Completion Dispatcher should perform the callback once the file is transmitted
- 主动启动器启动操作:为了执行异步操作，应用程序在异步操作处理器上启动操作。例如，Web服务器可能要求OS使用特定的套接字连接在网络上传输文件。要请求这样的操作，Web服务器必须指定要使用的文件和网络连接。此外，Web服务器必须指定(1)在操作完成时通知哪个完成处理程序，(2)在文件传输后哪个完成分派程序应该执行回调
- Asynchronous Operation Processor performs operation: When the application invokes operations on the Asynchronous Operation Processor it runs them asynchronously with respect to other application operations. Modern operating systems (such as Solaris and Windows NT) provide asynchronous I/O subsystems with the kernel.
- 异步操作处理器执行操作:当应用程序调用异步操作处理器上的操作时，它会相对于其他应用程序操作异步地运行这些操作。现代操作系统(例如Solaris和Windows NT)在内核中提供了异步I/O子系统。
- The Asynchronous Operation Processor notifies the Completion Dispatcher: When operations complete, the Asynchronous Operation Processor retrieves the Completion Handler and Completion Dispatcher that were specified when the operation was initiated. The Asynchronous Operation Processor then passes the Completion Dispatcher the result of the Asynchronous Operation and the Completion Handler to call back. For instance, if a file was transmitted asynchronously, the Asynchronous Operation Processor may report the completion status (such as success or failure), as well as the number of bytes written to the network connection
- 异步操作处理器通知完成分派器:当操作完成时，异步操作处理器检索在操作启动时指定的完成处理程序和完成分派器。然后，异步操作处理器将异步操作的结果和要回调的完成处理程序传递给完成分派器。
  例如，如果一个文件是异步传输的，异步操作处理器可能会报告完成状态(例如成功或失败)，以及写入网络连接的字节数
- Completion Dispatcher notifies the application: The Completion Dispatcher calls the completion hook on the Completion Handler, passing it any completion data specified by the application. For instance, if an asynchronous read completes, the Completion Handler will typically be passed a pointer to the newly arrived data
- 完成分派器通知应用程序:完成分派器调用完成处理程序上的完成钩子，将应用程序指定的任何完成数据传递给它。例如，如果异步读取完成，完成处理程序通常会传递一个指向新到达数据的指针

### CONSEQUENCES  结果

#### BENEFITS  

The Proactor pattern offers the following benefits:  

Proactor模式提供了以下好处:

- **Increased separation of concerns**: The Proactor pattern decouples application-independent asynchrony mechanisms from application-specific functionality. The applicationindependent mechanisms become reusable components that know how to demultiplex the completion events associated with Asynchronous Operations and dispatch the appropriate callback methods defined by the Completion Handlers. Likewise, the application-specific functionality knows how to perform a particular type of service (such as HTTP processing)
- **增加了关注点分离**:Proactor模式将独立于应用程序的异步机制与特定于应用程序的功能解耦。独立于应用程序的机制成为可重用的组件，这些组件知道如何将与异步操作关联的完成事件分解，并分派完成处理程序定义的适当回调方法。同样，特定于应用程序的功能知道如何执行特定类型的服务(如HTTP处理)
- **Improved application logic portability**: It improves application portability by allowing its interface to be reused independently of the underlying OS calls that perform event demultiplexing. These system calls detect and report the events that may occur simultaneously on multiple event sources. Event sources may include I/O ports, timers, synchronization objects, signals, etc. On real-time POSIX platforms, the asynchronous I/O functions are provided by the aio family of APIs [19]. In Windows NT, I/O completion ports and overlapped I/O are used to implement asynchronous I/O [20].
- **改进的应用程序逻辑可移植性**:通过允许其接口独立于执行多路分解事件的底层OS调用进行重用，改进了应用程序的可移植性。这些系统调用检测并报告可能在多个事件源上同时发生的事件。事件源可以包括I/O端口、计时器、同步对象、信号等。在实时POSIX平台上，异步I/O函数由api[19]的aio家族提供。在Windows NT中，I/O完成端口和重叠I/O用于实现异步I/O[20]。
- **The Completion Dispatcher encapsulates the concurrency mechanism**: A benefit of decoupling the Completion Dispatcher from the Asynchronous Operation Processor is that applications can configure Completion Dispatchers with various concurrency strategies without affecting other participants. The Completion Dispatcher can be configured to use several concurrency strategies including single-threaded and Thread Pool solutions
- 完成分派器封装了并发机制:完成分派器与异步操作处理器解耦的一个好处是，应用程序可以使用各种并发策略配置完成分派器，而不影响其他参与者。完成分派器可以配置为使用多种并发策略，包括单线程和线程池解决方案
- **Threading policy is decoupled from the concurrency policy**: Since the Asynchronous Operation Processor completes potentially long-running operations on behalf of Proactive Initiators, applications are not forced to spawn threads to increase concurrency. This allows an application to vary its concurrency policy independently of its threading policy. For instance, a Web server may only want to have one thread per CPU, but may want to service a higher number of clients simultaneously
- 线程策略与并发策略解耦:由于异步操作处理器代表主动启动器完成可能长时间运行的操作，应用程序不必被迫派生线程来提高并发性。这允许应用程序独立于线程策略改变其并发策略。例如，Web服务器可能只希望每个CPU有一个线程，但可能希望同时为更多的客户机提供服务
- **Increased performance**: Multi-threaded operating systems perform context switches to cycle through multiple threads of control. While the time to perform a context switch remains fairly constant, the total time to cycle through a large number of threads can degrade application performance significantly if the OS context switches to an idle thread. For instance, threads may poll the OS for completion status, which is inefficient. The Proactor pattern can avoid the cost of context switching by activating only those logical threads of control that have events to process. For instance, a Web server does not need to activate an HTTP Handler if there is no pending GET request
- **提高性能**:多线程操作系统执行上下文切换以循环通过多个控制线程。虽然执行上下文切换的时间相当稳定，但是如果操作系统上下文切换到空闲线程，那么通过大量线程进行循环的总时间会显著降低应用程序性能。
  例如，线程可能会轮询操作系统的完成状态，这是低效的。Proactor模式可以通过只激活那些需要处理事件的逻辑控制线程来避免上下文切换的代价。例如，如果没有挂起的GET请求，Web服务器不需要激活HTTP处理程序
- **Simplification of application synchronization**: As long as Completion Handlers do not spawn additional threads of control, application logic can be written with little or no regard to synchronization issues. Completion Handlers can be written as if they existed in a conventional singlethreaded environment. For instance, a Web server’s HTTP GET Handler can access the disk through an Async Read operation (such as the Windows NT TransmitFile function
- **简化应用程序同步**:只要完成处理程序不产生额外的控制线程，应用程序逻辑就可以在编写时很少或根本不考虑同步问题。完成处理程序可以像在传统的单线程环境中一样编写。例如，Web服务器的HTTP GET处理程序可以通过异步读取操作(如Windows NT TransmitFile函数)访问磁盘

#### DRAWBACKS  

The Proactor pattern has the following drawbacks:  

**Hard to debug**: Applications written with the Proactor pattern can be hard to debug since the inverted flow of control oscillates between the framework infrastructure and the method callbacks on application-specific handlers. This increases the difficulty of “single-stepping” through the run-time behavior of a framework within a debugger since application developers may not understand or have access to the framework code. This is similar to the problems encountered trying to debug a compiler lexical analyzer and parser written with LEX and YACC. In these applications, debugging is straightforward when the thread of control is within the user-defined action routines. Once the thread of control returns to the generated Deterministic Finite Automata (DFA) skeleton, however, it is hard to follow the program logic.

**难以调试**:使用Proactor模式编写的应用程序可能难以调试，因为控制的反向流在框架基础结构和特定于应用程序的处理程序的方法回调之间来回波动。这增加了在调试器中通过框架运行时行为的“单步”操作的难度，因为应用程序开发人员可能不理解或不能访问框架代码。这类似于调试使用LEX和YACC编写的编译器词法分析器和解析器时遇到的问题。在这些应用程序中，当控制线程位于用户定义的动作例程内时，调试非常简单。然而，一旦控制线程返回到生成的确定性有限自动机(Deterministic Finite Automata, DFA)框架，就很难遵循程序逻辑。

**Scheduling and controlling outstanding operations**: Proactive Initiators may have no control over the order in which Asynchronous Operations are executed. Therefore, the Asynchronous Operation Processor must be designed carefully to support prioritization and cancellation of Asynchronous Operations

**调度和控制未完成的操作**:主动启动器可能无法控制异步操作的执行顺序。因此，必须仔细设计异步操作处理器，以支持异步操作的优先级划分和取消

### KNOWN USES  

The following are some widely documented uses of the Proctor pattern:  

以下是一些广泛记录的Proctor模式的用法:

I/O Completion Ports in Windows NT: The Windows NT operating system implements the Proactor pattern. Various Asynchronous Operations such as accepting new network connections, reading and writing to files and sockets, and transmission of files across a network connection are supported by Windows NT. The operating system is the Asynchronous Operation Processor. Results of the operations are queued up at the I/O completion port (which plays the role of the Completion Dispatcher)

Windows NT中的I/O完成端口:Windows NT操作系统实现Proactor模式。Windows NT支持各种异步操作，如接受新的网络连接，对文件和套接字的读写，以及通过网络连接传输文件。操作系统是异步操作处理器。操作的结果在I/O完成端口(扮演完成分派器的角色)上排队。

ACE Proactor: The Adaptive Communications Environment (ACE) [5] implements a Proactor component that encapsulates I/O Completion Ports on Windows NT. The ACE Proactor abstraction provides an OO interface to the standard C APIs supported by Windows NT. The source code for this implementation can be acquired from the ACE website at www.cs.wustl.edu/schmidt/ACE.html

ACE Proactor自适应通信环境(ACE)[5]实现了前摄器组件,封装了I / O完成端口在Windows NT。ACE Proactor抽象提供了一个面向对象接口标准C api支持Windows NT。这种实现的源代码可以从ACE获得网站www.cs.wustl.edu/schmidt/ACE.html  

The UNIX AIO Family of Asynchronous I/O Operations: On some real-time POSIX platforms, the Proactor pattern is implemented by the aio family of APIs [19]. These OS features are very similar to the ones described above for Windows NT. One difference is that UNIX signals can be used to implement an truly asynchronous Completion Dispatcher (the Windows NT API is not truly asynchronous)

异步I/O操作的UNIX AIO家族:在一些实时POSIX平台上，Proactor模式是由api[19]的AIO家族实现的。
这些操作系统特性与上面为Windows NT所描述的特性非常相似。

Asynchronous Procedure Calls in Windows NT: Some systems (such as Windows NT) support Asynchronous Procedure Calls (APC)s. An APC is a function that executes asynchronously in the context of a particular thread. When an APC is queued to a thread, the system issues a software interrupt. The next time the thread is scheduled, it will run the APC. APCs made by operating system are called kernel-mode APCs. APCs made by an application are called user-mode APCs.

在Windows NT中的异步过程调用:一些系统(如Windows NT)支持异步过程调用(APC)。APC是在特定线程的上下文中异步执行的函数。当一个APC排队到一个线程，系统发出一个软件中断。下一次调度线程时，它将运行APC。操作系统制造的apc称为内核模式apc。由应用程序制作的APCs称为用户模式APCs。

## CONCLUDING REMARKS  **结束语**

Over the past several years, computer and network performance has improved substantially. However, the development of high-performance Web servers has remained expensive and error-prone. The JAWS framework described in this paper aims to support Web server developers by simplifying the application of various server designs and optimization strategies

在过去的几年中，计算机和网络性能有了很大的提高。然而，高性能Web服务器的开发仍然昂贵且容易出错。本文所描述的JAWS框架旨在通过简化各种服务器设计和优化策略的应用来支持Web服务器开发人员

This paper illustrates that a Web server based on traditional synchronous event dispatching performs adequately under light server loads. However, when the Web server is subject to heavy loads, a design based on a concurrent proactive architecture provides significantly better performance. However, programming this model adds complexity to the software design and increases the effort of developing high-performance Web servers.

本文演示了基于传统同步事件调度的Web服务器在轻量级服务器负载下的性能。然而，当Web服务器承受较大的负载时，基于并发主动架构的设计可以提供更好的性能。但是，编写这个模型增加了软件设计的复杂性，增加了开发高性能Web服务器的工作量。

Much of the development effort stems from the repeated rediscovery and reinvention of fundamental design patterns. Design patterns describe recurring solutions found in existing software systems. Applying design patterns to concurrent software systems can reduce software development time, improve code maintainability, and increase code reuse over traditional software engineering techniques

许多开发工作源于对基本设计模式的重复发现和再创造。设计模式描述在现有软件系统中发现的重复出现的解决方案。与传统的软件工程技术相比，在并发软件系统中应用设计模式可以减少软件开发时间，提高代码的可维护性，并增加代码的重用性

The Proactor pattern described in this paper embodies a powerful technique that supports both efficient and flexible event dispatching strategies for high-performance concurrent applications. In general, applying this pattern enables developers to leverage the performance benefits of executing operations concurrently, without constraining them to synchronous multi-threaded or reactive programming. In our experience, applying the Proactor pattern to the JAWS Web server framework has made it considerably easier to design, develop, test, and maintain.

本文描述的Proactor模式包含一种强大的技术，该技术支持高效、灵活的高性能并发应用程序事件调度策略。
通常，应用此模式使开发人员能够利用并发执行操作的性能优势，而不将它们限制为同步多线程或响应式编程。
根据我们的经验，将Proactor模式应用到JAWS Web服务器框架可以大大简化设计、开发、测试和维护。

