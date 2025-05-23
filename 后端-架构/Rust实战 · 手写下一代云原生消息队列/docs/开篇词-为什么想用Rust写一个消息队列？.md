> 本课程为精品小课，不标配音频

你好，我是文强，一个长期在基础架构领域摸爬滚打的技术人，也是极客时间[《深入拆解消息队列47讲》](https://time.geekbang.org/column/intro/100552001)的作者。这次为你带来《深入拆解消息队列47讲》的后续课程：《Rust 实战 · 手写下一代云原生消息队列》。

没错，这门课程的关键词将是 Rust 编程。

## 学习 Rust 的关键问题是什么？

近几年，Rust 这门语言不断地出现在我们的视野中，我们经常会看到 “Rust 重写一切” 这句话。作为使用过多门编程语言的老研发，我对这种口号一般是免疫的。因为每一种新语言出现时，都会有类似的口号，比如 Scala、Golang、Haskell 等等。

真正让我想去尝试 Rust 这门语言的契机是：**Rust 进入了 Linux 内核**。当时我就在想，这么多年来，除了 C 和 C++ 外，Rust 还是第一门进入内核的新兴语言。像 Linus这么轴的人竟然能让 Rust 进入内核，说明这个语言一定有非常厉害的地方。

在学习了 Rust 一段时间后，我遇到了第一个关键问题：**语法记不住，各种语法糖、技巧不会用，不会写。**其实对于我们这种长期编程的人来说，一般接触一门语言，花个几天就能开始产出了。但是，在 Rust 中我花了很长时间看完基础语法和各种特性后，第一个感觉就是好像懂了，但是真正要写点东西的时候，又发现好像啥都没懂。

本着实践是效率最高的学习方式，我开始去找能提升 Rust 技能的实战项目，这时我遇到了第二个关键问题：**没有合适的项目让我去学习和实践**。而且，这个问题还挺普遍。业界有很多偏基础语法、语言特性的资料，却少有让我们能够真正熟练掌握和应用这门语言的资料。

## 为何想用 Rust 编写项目？

怎么办？不能放弃啊！尤其是在体验了 Rust 的各种优点后，比如生命周期管理、无GC、借用等等特性。

这让我想起了我在一几年的时候，负责过的一个用 C++ 重写的 Kafka 项目。这个项目最终商业化了，还产生了不错的收益，但是最后，这个 C++ 重写的 Kafka 却不再维护了，用回了社区版本 Java 写的 Kafka。放弃的主要原因是 C++ 版本的 Kafka 特性支持跟不上社区的进度。而跟不上的一个核心原因就是 C++ 的开发效率确实比 Java 低很多，但是 C++ 语言特性带来的性能提升却是实实在在的，比 Java 写的社区版本高很多。

得益于多语言经历，我发现 Rust 具有不输 C++ 的性能，但是编码效率却比C++高很多。

所以我萌生了用 Rust 写一个消息队列的想法。一方面，我希望能让自己快速掌握这门语言，并应用于实际开发；另一方面，也是想把自己在消息队列这块的积累用 Rust 刷新一下，并沉淀成具体的项目。

很高兴，在过去一年的时间里，这个消息队列已经初具雏形了。

![图片](https://static001.geekbang.org/resource/image/9a/0d/9a245d93f733cf940ce0ccf72bb58d0d.png?wh=1534x414)

在这段编码过程中，我经历了从 Rust 小白到用 Rust 写成一个分布式基础软件的过程。由于Rust 确实是一门学习曲线很陡的语言，我在这期间踩了很多坑，也走了很多弯路，所以我想将这个过程分享出来，希望能带给你直接的帮助。

## 这门课是如何设计的？

这就不得不提到我们的课程设计。

诚如你所见，这个专栏目前只有十几讲，手写一个分布式基础软件又该是怎样的一个工作量，想必你有概念。所以，这将是一个系列课程。我们的整体学习路径是：从0开始，用 Rust 写成一个分布式的基础软件（消息队列）。期间会讲解 Rust 的实战技巧，带你融会贯通这门语言。最终，我们一起打造出一个牛逼的开源基础软件。

系列中的每一门课，都将按照 10 讲左右的篇幅去设计，定位实战，所以会包括技术方案设计、Rust 代码实现及讲解两大部分。

看到这里，你可能还会有点顾虑，如果没有消息队列的技术背景，我能学明白吗？

在我看来，影响不大。消息队列隶属基础软件，而分布式基础软件的基础模块都是通用的，比如单机网络、存储、分布式一致性、分布式高性能读写等等。换句话说，你可以忽略技术背景，而去关注项目编程本身，去关注 Rust 这门语言有哪些不同之处和使用优势。

那么，作为系列课程的第一门课，我们要怎么出发呢？

既然是实战，就不能只讲实现思路，而忽略实现过程，所以我们要聚焦一下，这门课程我们将主要讲解如何用 Rust 实现一个消息队列架构中的必要组件：类Zookeeper的分布式协调服务。而且这部分从代码实现逻辑上看，也是和消息队列业务逻辑无关的，所以更无需担心自己是否具有消息队列基础。

课程设计思路如下：

1. 先完成消息队列的整体技术方案设计，讲清楚我们整个系列课程是要做成什么样的消息队列。
2. 在开始编码之前，我们需要重点掌握 Rust 的哪些知识点，并对这些知识点做一个精简的讲解和资料链接。
3. 教你如何组织、管理、编译一个复杂的 Rust 项目，并完成最基础的命令行参数、配置、日志、测试用例部分的编写，即完成代码的初始框架。
4. 通过单机网络层、单机存储层、分布式集群这三个步骤，构建一个简单的分布式集群化的存储集群。
5. 在当前的分布式集群化的存储集群的基础上，实现存储消息队列集群元数据所需要的 KV 存储能力。
6. 从客户端和服务端的角度讲解提高性能的常用技巧，如客户端的连接池、连接复用、失败重试等，服务端的主节点读写、从节点可读的能力等。

完成本课程的学习后，你就掌握了如何用 Rust 编写一个分布式的 KV 模型存储的元数据集群。具体内容还可以参考下本课程的知识脑图。

![图片](https://static001.geekbang.org/resource/image/c7/6c/c71e02e92c17b49a4db09af425260a6c.jpg?wh=1920x1139)

在这里，你会看到项目核心逻辑以及 Rust 编程实现过程，还会有可运行的 Demo 支持大家学习和讨论。而随着系列课程的展开，你还会看到项目中更多功能模块的实现，更多 Rust 语法和特性的使用，期待你能在实践中收获全新的编程体验。

本课程所有可运行的源代码都在这里，请参考[robustmq-geek](https://github.com/robustmq/robustmq-geek)。

好，现在就开启我们的 Rust 编程之旅吧！
<div><strong>精选留言（7）</strong></div><ul>
<li><span>pedro</span> 👍（0） 💬（1）<p>有点疑惑，标题是手写消息队列，但实际的目录只有一个阶段，具体课程安排是咋样的？规划是什么？</p>2024-09-10</li><br/><li><span>拾掇拾掇</span> 👍（1） 💬（0）<p>第一个感觉就是好像懂了，但是真正要写点东西的时候，又发现好像啥都没懂！！！ 就是我现在的状况，哈哈</p>2024-09-22</li><br/><li><span>Geek_96e92a</span> 👍（1） 💬（0）<p>怎么成为 Contributor呢？</p>2024-09-19</li><br/><li><span>请务必优秀</span> 👍（0） 💬（0）<p>打卡，开始学习！</p>2024-09-18</li><br/><li><span>guanjun</span> 👍（0） 💬（0）<p>期待</p>2024-09-15</li><br/><li><span>小可爱(`へ´*)ノ</span> 👍（0） 💬（0）<p>跟上</p>2024-09-14</li><br/><li><span>太空牛仔</span> 👍（0） 💬（0）<p>跟进</p>2024-09-09</li><br/>
</ul>