# 摘要
今天给你介绍mesos，这是一个在各种各样的集群计算框架之间共享集群的平台，比如说 Hadoop 和 MPI，这里说的的共享指的是提升利用率和避免在不同计算框架之间复制数据，让框架通过轮流读取存储在本地机器上的数据达到数据本地性（Locality）。为了支持当今框架复杂的调度实现，Mesos 提出了一个分布式的两阶段调度机制叫做 resource offers。Mesos 决定每个框架需要多少 resources，同时框架自己决定接受哪些 resources 和 运行怎样的计算，我们的结果显示 Mesos 在不同框架之间分享集群资源就快要达到最优的数据本地性（Locality），且可以扩展到5w个node（emulated 模拟），且具备失败弹性恢复。

# 简介
现在的计算平台基本上都是商业机器集群，为庞大的联网服务和快速增长的计算任务提供支持。在程序猿在努力下，发展起来的很多的计算框架来简化集群的操作，比如 MapReduce，Dryad，MapReduce Online 和其他框架。

可以预见的是，新的计算框架肯定会持续的出现，没有框架会适合所有类型的应用程序（app）。所以，你的老板想要在一个集群里面跑起来所有的框架，你要负责选择出最合适的框架来运行你的 app。在框架之间复用集群可以显而易见的避免复制数据的消耗，从而来提高利用率。
两个比较常见的解决方案是，1，静态的划分集群资源分片，每个分片跑一个框架。2，给每个框架分配一些虚拟机。抱歉，这些方案即不能提升机器利用率也不能提高数据效率。这里面的主要问题是资源分配粒度和框架之间的错配。现有的框架，比如 Hadoop 和 Dryad，引入了一个细粒度的资源共享模型，就是把机器（node）划分成 slots，任务 jobs 由精简的 tasks 来组成，并且和 slots 相匹配。task 运行时间通常很短，一台机器上可以同时运行多个 tasks，这样一来，Job 就可以在同一个机器（node）上读写数据，达到一个比较好的数据 Locaility。再次抱歉，由于这些框架都是各自开发的，在框架之间没法得到一个细粒度的资源共享，所以就比较难搞。

在这篇论文里面，Mesos 给所有的框架一个统一的访问集群资源的接口，然后自己在底层实现一个细粒度的跨框架资源共享方案。

这里面的主要问题是 Mesos 怎样去构建一个可扩展的高校系统，可以同时支持各种各样的计算框架，这里面比较屌的是还要支持还没有出现的未来的框架，Mesos 也要支持。这就延伸出来一些子问题，一是各个框架之间基于他们的编程模型，交互模式，任务依赖和数据存储有不同的资源调度需求，二是调度系统要能扩展到成千上万的节点，数以百万计的 tasks。最后，因为这些框架都依赖 Mesos，所以 Mesos 本身必须容错且高可用。

一个可选的方法是实现一个中央调度器，统计框架的需求，资源利用情况和组织规则，然后计算出一个调度方案。尽管这种方案可以实现在框架之间的最优化调度，但是不得不面临许多的挑战。首当其冲的就是复杂度，调度器需要提供一个可以有效表达的 API 来满足所有框架的需求，还要在线百万级的 task 优化问题，即使这种方案可行，这种程度的复杂性会给扩展性和可靠性带来巨大的负面影响。其次，新的框架和调度策略在不断的出现，我们不能明确的保障说可以满足所有未来出现的框架。再次，许多现有的框架实现了它们自己的复杂的调度算法，将框架已有的调度实现重构移植到全局的调度器上需要巨大的工作量。

所以！Mesos 另辟蹊径，将调度控制委托给框架自身。这是一个全新的抽象概念，叫做 resource offer，这是将资源打包给框架，框架再在集群上分配资源运行 task。Mesos 基于组织策略比说公平，来决定给框架分配多少资源，同时框架自己决定接受多少的资源同时运行什么样的 task。尽管这种去中心化的调度模型达不到全局的最优化方案，但是在实际中我们去发现它运行得非常棒棒哒，可以让框架近乎完美的达到数据 Locality。此外，resource offer 也足够简单，可以非常方便稳定的实现出来，同时也可以达到高度可扩展和稳定性。

Mesos 还有其他好处，即使你实际上只使用了一个框架，也可以使用 Mesos，或者你可以运营一个框架的不同版本，我们在雅虎和 Facebook 使用这个方案来区分生产和实验的集群，同时隔离掉生产和集群的负载，同时用来滚动升级 Hadoop。还有，Mesos 可以用来快速的开发和实验部署新的框架。这种可以在不同框架之间共享资源的能力释放了开发者构建和运行特定框架的工作，同时也比给所有程序员的所有问题领域都只提供一种方案更加的有效。在这种情况，不同的框架能够给不同领域的问题提供更好的支持。

我们实现 Mesos 只用了 1w 行的 c++ 代码，这个系统可以扩展到 5w 个虚拟节点，使用 zookeeper 来支持容错。为了评估 Mesos，我们移植了3个集群计算框架到上面，Hadoop，MPI 和 Torque 批调度器。为了验证我们特定的框架能比通用的一个框架提供更高的价值的设想，我们在 Mesos 之上创建了一个新的框架叫做 Spark，针对迭代数据型的 job 做了优化，特别是在多个并行任务中需要反复读取同一个数据集的情况，结果显示在迭代型的机器学习负载情况下获得了10倍的加速。

这篇文章的结构如下，第二部分介绍了 Mesos 针对的数据中心环境。第三部分展示了 Mesos 的架构。第四部分我们的分布式调度模型，并总结它的优势运行环境。第五部分介绍 Mesos 的实现。第六部分评估性能，相关工作在第七部分，最后总结是第八部分。

# 目标环境

拿一个我们准备支持的负载类型举例，考虑在 Facebook 使用的 Hadoop 数据仓库，fb 将2000个节点的 web 服务器日志存储在 Hadoop 集群当中，被用来做商业智能，垃圾检测，广告优化。除了用生产环境定时的任务使用之外，一些实验性的工作也在同时进行，从数小时的机器学习训练任务到 Hive 提交的需要一两分钟的 SQL 即席查询。大多数的任务都是比较短的（这里给出一个中位数时间是84s），job 也是由细粒度的 MapReduce task 组成。如图1（这里么有图，可以去原文看）。

为了达到这些 job 的性能要求，fb 使用了一个公平调度器，同时也是在 task 级别的细粒度资源分配，可以做到不错的数据Locality。但是抱歉 again，如果用户想要使用 MPI，也许是因为 MPI 的交互模式有更好的效率，用户就必须单独设置一个 MPI 集群，然后倒入 TB 级别的数据到集群中。这个问题不是我们假设出来的，而是有真实的用户报告。Mesos 的目标就是针对这种情况，在多种的框架之间提供细粒度的资源调度和共享。

# 架构

我们开始我们的设计哲学来开始讨论，然后说一下 Mesos 的组件，资源分配机制，还有 Mesos 怎么做到隔离，可扩展和容错。

## 设计哲学

Mesos 旨在提供一个可以让不同框架之间高效共享集群的可扩展的弹性的核心。由于框架的发展速度非常的快，而且分化严重，所以我们的哲学就是定义一个最小化的接口，这个接口可以在框架之间高效的共享集群资源，剩下的调度和任务执行则是交给框架自己来完成。将具体的任务控制交给框架有以下的几点好处，一是允许框架针对不同的问题实现多种多样的解决方案，而且可以独立的针对方案进行持续的演化。二是可以保持 Mesos 足够简单，从而减少系统变更的频率，达到 Mesos 可扩展且稳定的特性。

尽管 Mesos 提供了一个底层的接口，但是我们还是期望使用比较高层次的 lib 来实现一个通用的功能（比如说容错）。这些 lib 类似系统扩展包。将这些功能放在外部扩展包中可以让 Mesos 保持简单稳定，同时可以让这些 lib 可以独立的发展。（就是偷懒）


## 概要
图2 展示了 Mesos 主要的组件，Mesos 包含一个 master 进程，它管理在集群 node 上运行的 slave 的守护进程，同时框架 task 也运行在这些 slave node 上。

由 master 实现的 resource offer 机制提供了细粒度的资源共享，每份 resource offer 是一个包含了空闲多个 slave node 空闲资源的列表。master 根据组织规则来决定提供给各个框架的资源具体有多少，这些规则就包括公平原则或者优先原则。为了支持众多的框架间资源分配策略，Mesos 提供了插件机制来让管理者自己来定义自己的分配策略。

每个运行在 Mesos 的框架包含了两个组件，一个在 master 上的调度器（scheduler）来负责资源的分配（提供 resource offer），一个 slave 上的 executor 进程来负责运行框架的任务。master 决定给多少资源给每个框架，调度器再来进一步的决定使用哪一个 resource offer。当一个框架接受了 resource offer，它会传递给 Mesos 它想要运行的任务的描述。

图3 展示了一个框架怎么调度来运行任务的例子，在第一步，slave 1 向 master 报告它自己有 4 个 CPU 和 4GB 内存可用，然后 master 调用分配模块，这个模块告诉 master 框架1需要所有的资源，第二步，master 发送一个 resource offer 给到框架1，这个 offer 包含了上述的计算资源。第三步，框架1 的调度器回复 master 有2个任务运行在这份资源上面，分别是 <2 CPU, 1GB RAM> 给到第一个任务，<1CPU，2GB RAM> 给到第二个任务。最后，在第四步，master 发送任务给到 slave，同时配合分配给到合适的资源，slave 上的执行器，会轮流把任务起起来。由于还有 1 个 CPU 和 1GB 的内存处于空闲状态，分配模块可以继续将剩下的空闲资源分配给框架2。多说一点，当任务结束和新的资源加入到空闲列表的时候，resource offer 会不停的重复上述过程。

为了维护一个最小集合的接口从而使得框架可以独立的演化，Mesos 不会要求框架指定它们的资源需求或者限制。取而代之的是，Mesos 给予框架拒绝 resource offer 的选择权。框架可以拒绝不满足它自己要求的资源从而等待可以满足自己运行的 resource offer。这样一来，这种拒绝机制就可以使得框架可以支持任意复杂度的资源限制，而且还可以让 Mesos 保持简单可扩展。

仅仅使用拒绝机制来满足所有的框架资源需求还面临着一个潜在的挑战，那就是【效率】，框架可能需要等待很久的时间才能等到自己需要的 resource offer，同时 Mesos 也可能发送很多的无效的 resource offer。为了避免这种情况，Mesos 允许框架设置自己的资源过滤器，可以在给出 resource offer 前，前置过滤掉一个不满足条件的 resource offer。举个例子，一个框架可以给出一个包含自己可以运行在的 node 的白名单。

有两个点值得注意，第一，过滤器（filter）仅是一个性能提升的手段，最终的资源接受还是拒绝的权利还是保留在框架的手里。第二，接下来会讲到，当负载包含了非常细粒度的任务，resource offer 模型表现的非常好，即使没有 filter 的存在也非常好。特别是，我们发现了一个简单的策略叫做延迟调度（deplay scheduling），在框架在等待有限的资源，同时需要在 node 本地读取数据的情况下，在等待 1-5s的情况下几乎所有的任务都可以在本地读到数据。

在接下来的子章节中，我们会描述 Mesos 怎样完成两个关键的功能，资源分配（resource allocation）和资源隔离（resource isolation）。然后描述 filter 和其他的一些让 resource offer 保持可扩展和稳定性的机制。最后，我们讨论容错和总结 Mesos 的 API。

## 资源分配（resource allocation）
Mesos 将资源分配委托给可插拔的插件，这样组织者就可以按需定制分配方案。到现在为止，我们实现了两个分配模块，一个是最大最小化的公平分配方案，另一个是严格优先级分配方案，类似的方案在 Hadoop 里或者 Dryad 里可以找到。

在普通的操作中，Mesos 占了大多数任务都比较短暂这一特点的便宜，伴随着仅当任务结束时才重新分配资源的策略。这些情况发生得比较频繁，以便框架可以快速的获取需要的资源。比如，一个框架的在集群中的占比是 10%，它大约需要等待平均任务时间的10%的时长才能获取到它自己份额。然而，如果集群被长时间的任务挤占，这种情况可能是任务出现 bug 或者是框架自身比较的贪婪，分配模块可以结束掉任务，在结束任务之前，Mesos 也会给一点时间给模块做清理工作。

我们将结束 task 的策略实现交给分配模块来做，但是在这里会描述两个相关的机制。第一，结束一个 task 对许多框架来讲影响并不大，对任务有相互依赖关系的框架会比较大（MPI）。我们允许这些框架通过让分配模块暴露出一个“被保证的分配”的机制给到每个框架来避免被杀掉，即是框架不会丢失任务的一个资源数量。框架通过 API 调用读取它们的被保证分配资源数量。分配模块来负责这些“被保证的资源”供应。现在为止，我们的“被保证资源”的语义非常简单：如果一个框架的资源在“被保证的资源”水位线下，就没有任务会被强制结束；如果一个框架的资源在“被保证的资源”水位线上，它的任务就有可能会被强制结束。

第二，为了决定什么时候执行撤销（revocation），Mesos 必须知道哪个框架会使用更多的资源。框架通过 API 来表明自己感兴趣的 resource offer。

## 隔离（isolation）

Mesos 提供了在同一个 slave 上运行的框架之间的性能隔离，通过现有的 OS 隔离机制来实现。由于这些机制是平台独立的，所以我们通过插件实现的隔离模块实现了许多隔离机制。

我们现在使用 OS 的容器技术来实现资源隔离，尤其是 Linux Container 技术和 Solaris Project。这些技术可以限制 CPU，内存，网络带宽和进程树的 IO 使用情况。这些隔离技术并不完美，但是在框架之上（比如说 Hadoop ）使用容器已经是一个优点，不同 jobs 的 task 只需要简单的运行在不同的进程中即可。

## 使 Resource offer 可扩展和稳定

因为 Mesos 中的的任务调度是一个分布式过程，它需要在面对失败的时候拥有稳定性和高效率。Mesos 使用了三种技术来达到这个目标。

第一，因为一些框架会一直拒绝某些资源，Mesos 会短路化处理拒绝过程，通过 filter 机制来避免和 master 的交互过程。我们现在提供了两种过滤器，第一个是“仅向列表 L 中的 nodes 发放 offer”；“仅使用最新的空闲的 R resource 向 node 提供资源”。然后，其他类型的预测型过滤器也是有的。注意和常规的限制性描述语言不一样的是，过滤器提供的语义仅仅是 true 或者 false，表示某个 node 上的一个框架会接受或者拒绝一份 offer 上的资源包，所以可以非常快的在 master 上运行。任何 resource 一旦没有通过过滤器，就会被看作会被框架拒绝。

—— translated in 2022/3/9

第二，因为框架对 resource offer 作出响应也需要一定的时间，Mesos 面向框架的分配集群来计算 resource offer（这句话没看懂，啥意思？原文是 Mesos counts resources offered to a framework towards its allocation of the cluster。直译的话就是，Mesos 面向框架的的集群分配计算给到它的 resource）。对于提升框架对于 resource offer 的响应时间是个不错的激励，而且可以快速的过滤掉框架不需要的 resource offer。

第三，如果一个框架长时间没有对一个 resource offer 进行有效的响应，Mesos 会撤销 resource offer，然后把资源给到其他的框架。

## 容错性
由于所有的框架都以来 Mesos master，所以 master 的容错就是整个 Mesos 容错的关键，为了做到容错性，我们把 master 设计成了 **软状态（soft state）**，这样的话，一个新的 master 就可以从 slaves 和框架调度器中的信息完全重建出它的内部状态。特别是，master 仅有的状态仅是处于活动状态的 slave 节点，框架和运行中的 tasks。这些信息对于计算每个框架正在使用多少资源是有效的，还能用来运行分配策略。我们同时运行许多个热备的 master 节点，通过 zookeeper 来做 leader 选举，当现在的 master 节点失败的时候，slaves 和调度器连接到下一个被选举出来的 master 节点，然后重新填充状态信息。

除了处理 master 失败的情况之外，Mesos 也报告节点失败，处理框架的调度器崩溃。框架然后可以选一个策略对这些失败作出响应动作。

最后，为了处理调度器失败，Mesos 允许一个框架注册多个调度器，这样当其中一个失败的时候，另一个就会被 master 通知到来处理后续的动作。框架必须使用它们自己的机制来分享调度器之间的状态。

## API 总结
表1总结了 Mesos 的API，callback 列列出了框架必须实现的功能，actions 是他们可以调用的方法。

# Mesos 的行为

在这个部分，我们研究了 Mesos 在不同的工作负载下的行为表现，我们的目标不是开发一个系统的精确的模型，而是一个提供一个对他粗浅的了解，为了总结 Mesos 的分布式调度器模型能表现良好的工作环境。

简单来说，我们发现当框架满足下面条件的时候 Mesos 表现非常好，框架能够弹性扩缩容的，任务都是同质的，还有框架喜欢所有的节点都是同样的节点。我们展示了 Mesos 能够模拟啊一个中心化的调度器，这个调度器能够模拟在不同的框架之间实现公平性调度。此外，我们展示了 Mesos 能够处理异质的任务时长，而且还不会影响框架在处理短时任务的性能，我们同时讨论了框架在 Mesos 下运行能够获得的好处，还证明了这些好处可以对所有的框架生效，我们用一些 Mesos 的分布式调度器限制总结这部分内容。

## 定义，度量和假设

在我们的讨论中，我们考虑三个度量（metrics）
框架启动时间，框架自身分配资源所花费的时间
Job 完成时间，Job 完成需要的时间，假设每个框架仅有一个 Job
系统效率，整个集群的效率

我们概括了在两个维度上的负载：弹性和任务的分布式耗时。一个弹性的框架，比如 Hadoop 和 Dryad，能够扩展和收缩它自己的资源，比如说，它能够在请求到节点的时候立马就使用，任务结束的时候就立马释放这些节点。相较而言，一个刚性（rigid）的框架，比如说 MPI，只有在它请求一个固定数量的资源节点后才能够启动它的 Jobs，而且不能动态的扩展从而获得新加入的资源的好处，或者收缩来释放集群的性能。从任务耗时的角度来说，我们同时考虑同质化任务和异质化的任务。（笔者自己也不是太理解同质化的任务和异质化的任务是啥意思，原文是 homogeneous and heterogeneous）.

我们同样区分两种类型的资源：强制型和偏好型（mandatory & oerferred）。一个强制型的资源是框架运行必须的资源。比如，一个GPU资源对于一个没有GPU就不能运行的任务的来说就是强制型的资源。相较而言，一个偏好型的资源对于框架来说就是有了更好，没有也行，同时也可以运行在其他相等的资源上面，比如说，一个框架也许偏好运行在有本地数据的节点上面，但是如果有需要的话，也可以运行在没有本地数据的节点上。

我们假设一个框架要求的强制型资源永远也不好超过它的被保证的份额（前文有叙述到什么是被确保的份额）。这个保证可以保证框架之间不会因为等待资源可用出现死锁的情况。为了简便，我们假设所有的任务都有同样的资源需求而且可以运行在同样的机器分片上，我们把这样的机器分片叫做 *slots*，还有每个框架仅运行一个 Job。

## 同质化任务