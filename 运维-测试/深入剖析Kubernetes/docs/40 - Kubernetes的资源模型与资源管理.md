你好，我是张磊。今天我和你分享的主题是：Kubernetes的资源模型与资源管理。

作为一个容器集群编排与管理项目，Kubernetes为用户提供的基础设施能力，不仅包括了我在前面为你讲述的应用定义和描述的部分，还包括了对应用的资源管理和调度的处理。那么，从今天这篇文章开始，我就来为你详细讲解一下后面这部分内容。

而作为Kubernetes的资源管理与调度部分的基础，我们要从它的资源模型开始说起。

我在前面的文章中已经提到过，在Kubernetes里，Pod是最小的原子调度单位。这也就意味着，所有跟调度和资源管理相关的属性都应该是属于Pod对象的字段。而这其中最重要的部分，就是Pod的CPU和内存配置，如下所示：

```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

> 备注：关于哪些属性属于Pod对象，而哪些属性属于Container，你可以在回顾一下第14篇文章[《深入解析Pod对象（一）：基本概念》](https://time.geekbang.org/column/article/40366)中的相关内容。

在Kubernetes中，像CPU这样的资源被称作“可压缩资源”（compressible resources）。它的典型特点是，当可压缩资源不足时，Pod只会“饥饿”，但不会退出。

而像内存这样的资源，则被称作“不可压缩资源（incompressible resources）。当不可压缩资源不足时，Pod就会因为OOM（Out-Of-Memory）被内核杀掉。

而由于Pod可以由多个Container组成，所以CPU和内存资源的限额，是要配置在每个Container的定义上的。这样，Pod整体的资源配置，就由这些Container的配置值累加得到。

其中，Kubernetes里为CPU设置的单位是“CPU的个数”。比如，cpu=1指的就是，这个Pod的CPU限额是1个CPU。当然，具体“1个CPU”在宿主机上如何解释，是1个CPU核心，还是1个vCPU，还是1个CPU的超线程（Hyperthread），完全取决于宿主机的CPU实现方式。Kubernetes只负责保证Pod能够使用到“1个CPU”的计算能力。

此外，Kubernetes允许你将CPU限额设置为分数，比如在我们的例子里，CPU limits的值就是500m。所谓500m，指的就是500 millicpu，也就是0.5个CPU的意思。这样，这个Pod就会被分配到1个CPU一半的计算能力。

当然，**你也可以直接把这个配置写成cpu=0.5。但在实际使用时，我还是推荐你使用500m的写法，毕竟这才是Kubernetes内部通用的CPU表示方式。**

而对于内存资源来说，它的单位自然就是bytes。Kubernetes支持你使用Ei、Pi、Ti、Gi、Mi、Ki（或者E、P、T、G、M、K）的方式来作为bytes的值。比如，在我们的例子里，Memory requests的值就是64MiB (2的26次方bytes) 。这里要注意区分MiB（mebibyte）和MB（megabyte）的区别。

> 备注：1Mi=1024\*1024；1M=1000\*1000

此外，不难看到，**Kubernetes里Pod的CPU和内存资源，实际上还要分为limits和requests两种情况**，如下所示：

```
spec.containers[].resources.limits.cpu
spec.containers[].resources.limits.memory
spec.containers[].resources.requests.cpu
spec.containers[].resources.requests.memory
```

这两者的区别其实非常简单：在调度的时候，kube-scheduler只会按照requests的值进行计算。而在真正设置Cgroups限制的时候，kubelet则会按照limits的值来进行设置。

更确切地说，当你指定了requests.cpu=250m之后，相当于将Cgroups的cpu.shares的值设置为(250/1000)\*1024。而当你没有设置requests.cpu的时候，cpu.shares默认则是1024。这样，Kubernetes就通过cpu.shares完成了对CPU时间的按比例分配。

而如果你指定了limits.cpu=500m之后，则相当于将Cgroups的cpu.cfs\_quota\_us的值设置为(500/1000)\*100ms，而cpu.cfs\_period\_us的值始终是100ms。这样，Kubernetes就为你设置了这个容器只能用到CPU的50%。

而对于内存来说，当你指定了limits.memory=128Mi之后，相当于将Cgroups的memory.limit\_in\_bytes设置为128 * 1024 * 1024。而需要注意的是，在调度的时候，调度器只会使用requests.memory=64Mi来进行判断。

**Kubernetes这种对CPU和内存资源限额的设计，实际上参考了Borg论文中对“动态资源边界”的定义**，既：容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，这是因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额。

基于这种假设，Borg在作业被提交后，会主动减小它的资源限额配置，以便容纳更多的作业、提升资源利用率。而当作业资源使用量增加到一定阈值时，Borg会通过“快速恢复”过程，还原作业原始的资源限额，防止出现异常情况。

而Kubernetes的requests+limits的做法，其实就是上述思路的一个简化版：用户在提交Pod时，可以声明一个相对较小的requests值供调度器使用，而Kubernetes真正设置给容器Cgroups的，则是相对较大的limits值。不难看到，这跟Borg的思路相通的。

在理解了Kubernetes资源模型的设计之后，我再来和你谈谈Kubernetes里的QoS模型。在Kubernetes中，不同的requests和limits的设置方式，其实会将这个Pod划分到不同的QoS级别当中。

**当Pod里的每一个Container都同时设置了requests和limits，并且requests和limits值相等的时候，这个Pod就属于Guaranteed类别**，如下所示：

```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

当这个Pod创建之后，它的qosClass字段就会被Kubernetes自动设置为Guaranteed。需要注意的是，当Pod仅设置了limits没有设置requests的时候，Kubernetes会自动为它设置与limits相同的requests值，所以，这也属于Guaranteed情况。

**而当Pod不满足Guaranteed的条件，但至少有一个Container设置了requests。那么这个Pod就会被划分到Burstable类别**。比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

**而如果一个Pod既没有设置requests，也没有设置limits，那么它的QoS类别就是BestEffort**。比如下面这个例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```

那么，Kubernetes为Pod设置这样三种QoS类别，具体有什么作用呢？

实际上，**QoS划分的主要应用场景，是当宿主机资源紧张的时候，kubelet对Pod进行Eviction（即资源回收）时需要用到的。**

具体地说，当Kubernetes所管理的宿主机上不可压缩资源短缺时，就有可能触发Eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等。

目前，Kubernetes为你设置的Eviction的默认阈值如下所示：

```
memory.available<100Mi
nodefs.available<10%
nodefs.inodesFree<5%
imagefs.available<15%
```

当然，上述各个触发条件在kubelet里都是可配置的。比如下面这个例子：

```
kubelet --eviction-hard=imagefs.available<10%,memory.available<500Mi,nodefs.available<5%,nodefs.inodesFree<5% --eviction-soft=imagefs.available<30%,nodefs.available<10% --eviction-soft-grace-period=imagefs.available=2m,nodefs.available=2m --eviction-max-pod-grace-period=600
```

在这个配置中，你可以看到**Eviction在Kubernetes里其实分为Soft和Hard两种模式**。

其中，Soft Eviction允许你为Eviction过程设置一段“优雅时间”，比如上面例子里的imagefs.available=2m，就意味着当imagefs不足的阈值达到2分钟之后，kubelet才会开始Eviction的过程。

而Hard Eviction模式下，Eviction过程就会在阈值达到之后立刻开始。

> Kubernetes计算Eviction阈值的数据来源，主要依赖于从Cgroups读取到的值，以及使用cAdvisor监控到的数据。

当宿主机的Eviction阈值达到后，就会进入MemoryPressure或者DiskPressure状态，从而避免新的Pod被调度到这台宿主机上。

而当Eviction发生的时候，kubelet具体会挑选哪些Pod进行删除操作，就需要参考这些Pod的QoS类别了。

- 首当其冲的，自然是BestEffort类别的Pod。
- 其次，是属于Burstable类别、并且发生“饥饿”的资源使用量已经超出了requests的Pod。
- 最后，才是Guaranteed类别。并且，Kubernetes会保证只有当Guaranteed类别的Pod的资源使用量超过了其limits的限制，或者宿主机本身正处于Memory Pressure状态时，Guaranteed的Pod才可能被选中进行Eviction操作。

当然，对于同QoS类别的Pod来说，Kubernetes还会根据Pod的优先级来进行进一步地排序和选择。

在理解了Kubernetes里的QoS类别的设计之后，我再来为你讲解一下Kubernetes里一个非常有用的特性：cpuset的设置。

我们知道，在使用容器的时候，你可以通过设置cpuset把容器绑定到某个CPU的核上，而不是像cpushare那样共享CPU的计算能力。

这种情况下，由于操作系统在CPU之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。事实上，**cpuset方式，是生产环境里部署在线应用类型的Pod时，非常常用的一种方式。**

可是，这样的需求在Kubernetes里又该如何实现呢？

其实非常简单。

- 首先，你的Pod必须是Guaranteed的QoS类型；
- 然后，你只需要将Pod的CPU资源的requests和limits设置为同一个相等的整数值即可。

比如下面这个例子：

```
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "2"
      requests:
        memory: "200Mi"
        cpu: "2"
```

这时候，该Pod就会被绑定在2个独占的CPU核上。当然，具体是哪两个CPU核，是由kubelet为你分配的。

以上，就是Kubernetes的资源模型和QoS类别相关的主要内容。

## 总结

在本篇文章中，我先为你详细讲解了Kubernetes里对资源的定义方式和资源模型的设计。然后，我为你讲述了Kubernetes里对Pod进行Eviction的具体策略和实践方式。

正是基于上述讲述，在实际的使用中，我强烈建议你将DaemonSet的Pod都设置为Guaranteed的QoS类型。否则，一旦DaemonSet的Pod被回收，它又会立即在原宿主机上被重建出来，这就使得前面资源回收的动作，完全没有意义了。

## 思考题

为什么宿主机进入MemoryPressure或者DiskPressure状态后，新的Pod就不会被调度到这台宿主机上呢？

感谢你的收听，欢迎你给我留言，也欢迎分享给更多的朋友一起阅读。
<div><strong>精选留言（15）</strong></div><ul>
<li><span>DJH</span> 👍（82） 💬（5）<p>“为什么宿主机进入 MemoryPressure 或者 Dis...“

这是因为给宿主机打了污点标记吗？</p>2018-11-23</li><br/><li><span>gogo</span> 👍（41） 💬（5）<p>老师您好，cpu设置limit之后，容器的cpu使用率永远不会超过这个限制对吗？而mem设置limit之后，容器mem使用率有可能超过这个限制而被kill掉，也就是说设置了cpu limit之后，容器永远不会因为cpu超过限制而被kill对吗</p>2018-11-23</li><br/><li><span>刘岚乔月</span> 👍（21） 💬（3）<p>1、3、5都在追文章，一直都有一个疑问，请作者能解惑下。
对于主java是其他语言（非运维）的同学来说，我们是否需要深入了解k8s和docker（还是停留在使用层面） 我想一直跟着学的同学大部门还是冲着能找到更好的工作去的（有情怀的同学请忽略）
目前更大公司的招聘对于要求掌握k8s和docker的基本上都是运维岗位。
并没有招聘java要求掌握k8s和docker，面试中也不曾问到。感觉很尴尬 - -！
毕竟时间成本在哪，请作者能阐述下自己的观点！
</p>2018-11-25</li><br/><li><span>虎虎❤️</span> 👍（11） 💬（4）<p>在namespace limitRange 里面设置了default request 和 default limit之后，创建出来的pod即使不显式指定limit和request，是不是也是guaranteed？</p>2018-11-27</li><br/><li><span>unique</span> 👍（10） 💬（5）<p>这时候，该 Pod 就会被绑定在 2 个独占的 CPU 核上。

独占的意思就是其它pod 不能使用这两个CPU了么？</p>2018-11-23</li><br/><li><span>beenchaos</span> 👍（4） 💬（1）<p>请问张老师，cpuset是否只适用于nginx或者redis这类单线程的应用，为这类进程单独绑定一个CPU。而针对多线程的应用程序，设置cpuset反而会限制该应用程序的并发能力？这样理解准确么？</p>2019-05-28</li><br/><li><span>初学者</span> 👍（4） 💬（1）<p>老师，能不能讲一下kubernetes是如何划分和管理宿主机上的cgroups结构的？</p>2018-12-01</li><br/><li><span>汪浩</span> 👍（3） 💬（1）<p>被称作“不可压缩资源（compressible resources）

应该是 uncompressible</p>2018-11-25</li><br/><li><span>Pixar</span> 👍（1） 💬（2）<p>如果某个Guaranted Pod 的 Mem 设定为了256，在宿主机资源不紧张但该Pod 的的 mem 使用量达到了256以后会出现什么情况？会被oom杀掉吗？</p>2018-11-23</li><br/><li><span>chenhz</span> 👍（43） 💬（2）<p>
Pod 是最小的原子调度单位，所有跟调度和资源管理相关的属性，都是 Pod 对象属性的字段。其中最重要的是 Pod 和 CPU 配置。其中，CPU 属于可压缩资源，内存属于不可压缩资源。当可压缩资源不足时，Pod 会饥饿；当不可压缩资源不足时，Pod 就会因为 OOM 被内核杀掉。

Pod ，即容器组，由多个容器组成，其 CPU 和内存配额在 Container 上定义，其 Container 配置的累加值即为 Pod 的配额。

## limits 和 requests

- requests：kube-scheduler 只会按照 requests 的值进行计算。
- limits：kubelet 则会按照 limits 的值来进行设置 Cgroups 限制.


## QoS 模型
- Guaranteed： 同时设置 requests 和 limits，并且 requests 和 limit 值相等。优势一是在资源不足 Eviction 发生时，最后被删除；并且删除的是 Pod 资源用量超过 limits 时才会被删除；优势二是该模型与 docker cpuset 的方式绑定 CPU 核，避免频繁的上下午文切换，性能会得到大幅提升。
- Burstable：不满足 Guaranteed 条件， 但至少有一个 Container 设置了 requests
- BestEffort：没有设置 requests 和 limits。

## Eviction 

```bash
kubelet --eviction-hard=imagefs.available&lt;10%,memory.available&lt;500Mi,nodefs.available&lt;5%,nodefs.inodesFree&lt;5% --eviction-soft=imagefs.available&lt;30%,nodefs.available&lt;10% \
--eviction-soft-grace-period=imagefs.available=2m,nodefs.available=2m \
--eviction-max-pod-grace-period=600
```

两种模式：
- soft: 如 `eviction-soft-grace-period=imagefs.available=2m`  eviction 会在阈值达到 2 分钟后才会开始
- hard：evivtion 会立即开始。

**eviction 计算原理**： 将 Cgroups （limits属性）设置的值和 cAdvisor 监控的数据相比较。


最佳实践：DaemonSet 的 Pod 都设置为 Guaranteed， 避免进入“回收-&gt;创建-&gt;回收-&gt;创建”的“死循环”。</p>2020-03-19</li><br/><li><span>wilder</span> 👍（30） 💬（1）<p>极客时间里面最爱的课程，没有之一，哈哈哈哈哈哈</p>2018-11-23</li><br/><li><span>Flying</span> 👍（12） 💬（2）<p>请用老师，cpuset为2，这个Pod就独占两个cpu核上，假如宿主机总共只有10个cpu核，那么这台机就只能运行5个cpuset=2的Pod吗</p>2018-11-30</li><br/><li><span>虎虎❤️</span> 👍（8） 💬（3）<p>能否分享一下给namespace 设置quota的经验呢？

如果设置的太小，会造成资源的浪费。如果设置太大，又怕起不到限制的作用。一个namespace使用资源太多可能会影响其他namespace用户的使用。

这是否也是namespace只能做soft multi-tenant的佐证呢？Cloudfoundry应该是按照实际的usage来设置space的quota，如果有监控插件，k8s可以也按照实际的usage来设置quota吗？</p>2018-11-23</li><br/><li><span>黑米</span> 👍（7） 💬（8）<p>如果一个java应用JAVA_OPTS配置了-Xms4g -Xmx4g，k8s这边要配置多少的limit比较合适？直接4Gi的话应用内存达到一个阈值会被重启。</p>2019-12-12</li><br/><li><span>Adam</span> 👍（4） 💬（3）<p>宿主机进入了MemoryPressure后被打上污点标记，新的POD不会被调度到此节点，那假如宿主机资源恢复正常后，这个污点标记会自己消失吗？还是说需要人工介入去处理。</p>2019-10-10</li><br/>
</ul>