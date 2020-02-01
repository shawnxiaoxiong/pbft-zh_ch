发表在 *1999* 年 *2* 月于美国新奥尔良举行的第三届操作系统设计与实现研讨会上





# 实用的拜占庭容错



Miguel Castro and Barbara Liskov

*Laboratory for Computer Science,*

*Massachusetts Institute of Technology,*

*{castro,liskov}@lcs.mit.edu*

译者&校对：熊潇, [shawnxiong@qq.com](mailto:shawnxiong@qq.com)



## 摘要

本文介绍了一种能够实现拜占庭容错的新复制算法。我们相信，由于恶意攻击和软件错误越来越普遍，并且会导致故障节点表现出任意行为，因此，拜占庭容错算法在未来将变得越来越重要。鉴于先前的算法要么假定系统是同步的，要么性能太慢而无法运用到实际中，本文描述的算法是实用的：它可以在异步环境（如Internet 等）下工作，并且结合了多项重要的优化策略，可以将先前算法的响应时间缩短一个数量级以上。我们利用这个算法实现了拜占庭容错的NFS服务，并评估了其性能。结果表明，我们的实现仅比非复制式的标准 NFS 慢 3％。 

## 1、介绍 

恶意攻击和软件错误越来越普遍。行业和政府对在线信息服务的日益依赖增加了对恶意攻击的吸引力，并且使攻击得手的后果愈加严重。另外，由于软件规模和复杂性的提升，软件错误的数量也在不断增加。由于恶意攻击和软件错误都可能导致故障节点表现出拜占庭（即任意）行为，因此，拜占庭容错算法变得越来越重要。

本文提出了一种新的实用的状态机复制算法[17，34]，可以容忍拜占庭错误。该算法同时满足活性（以下引用原文称为Liveness）和安全性（以下引用原文称为Safty），支持n个副本（以下引用原文称为replica）中最多 $$[\frac{n-1}{3}]$$个同时出现恶化的容错机制。这意味着客户端（以下引用原文称为client）最终会收到对其请求的响应，并且根据线性一致性要求这些响应都是正确的[14，4]。该算法可在 Internet 等异步系统中使用，并且结合了重要的优化策略，可以使其高效运行。

> 译者注：原文是*faulty*，直译为故障，其核心是想表达可以表现出任意行为。为了避免与正常节点偶尔的网络延迟等问题造成的错误或失败发生混淆，本文统一翻译为恶化

在致力于解决拜占庭容错问题（从[19]开始）的协议和复制技术上，前人已经做了大量的工作。但是，大多数早期的工作（例如[3，24，10]）要么旨在证明其技术的理论可行性，而效率太低无法用于实际；要么依靠同步性假设，即依赖消息延迟和处理速度在一个已知的范围。Rampart [30]和 SecureRing [16]是为实用而设计的，也是最接近我们的系统 ，但它们依靠同步性假设来确保正确性，这在存在恶意攻击时很危险。攻击者可能会阻延非恶化节点或它们之间的通信，直到它们被标记为恶化节点并从副本群组中排除为止，从而损害服务的安全性。通常，这种拒绝服务攻击（denial-of-service）比获得对非恶化节点的控制要容易得多。

我们的算法不容易受到这种攻击，因为它不依赖同步性来保证安全。此外，如第7节所述，它将Rampart和 SecureRing的性能提高了一个数量级以上。它仅使用一次消息往返执行只读操作，以及使用两次消息往返执行读写操作。此外，公钥加密被认为是Rampart技术中的主要延迟[29]和吞吐量[22]瓶颈，而我们的算法仅在出现错误时才使用它，在正常操作时仅使用基于消息认证码（MACs）的高效身份验证方案。 

为了评估我们的方法，我们实现了一个复制库，并用其开发了一项真正的服务：一个支持NFS 协议的实现拜占庭容错的分布式文件系统。我们使用了Andrew Benchmark[15]来评估我们系统的性能。结果表明，在正 

常情况下，我们的系统仅比Digital Unix内核中的标准 NFS 守护程序慢 3％。 

因此，本文做出了以下贡献： 

- 它提出了第一个可以在异步网络中支持拜占庭容错的状态机复制协议。 
- 它提出了许多重要的优化策略，令该算法性能表现良好，使其可以在实际系统中使用。
- 它描述了支持拜占庭容错的分布式文件系统的实现。 
- 它提供了可以量化复制技术性能表现的实验结果。 

本文的其余部分安排如下。首先描述了我们的系统模型，包括我们对恶化行为的假设。第3节描述了算法解决的问题，并陈述了判定其正确的条件。该算法的具体内容在第4节中进行了阐述，而一些重要的优化策略在第5节做了讲解。第6节介绍了我们的复制库以及我们如何使用它来实现一个支持拜占庭容错的NFS。第7节介绍了我们的实验结果。第8节讨论相关工作。最后，我们总结了已完成的工作并讨论了未来的研究方向。

## 2、系统模型

我们假设一个异步的分布式系统，其中各节点通过网络连接。网络可能出现消息通讯出现丢失，延迟，重复或乱序的情况。

我们使用拜占庭故障模型，即故障（恶化）节点可能表现出仅受以下条件限制的任意行为。我们假设节点的恶化情况是相互独立的。为了使这种假设在存在恶意攻击的情况下成立，需要采取一些措施，例如，每个节点应运行服务代码和操作系统的不同实现，并且应具有不同的root密码和不同的管理员。可以从相同的代码库[28]获得不同的实现，并且可以从不同的供应商处购买操作系统以达到低复用性的效果。对于某些服务而言，另一种选择是N版本编程，即不同的开发团队进行不同的实现。 

我们使用密码学技术来防止欺骗和重放攻击以及检测损坏的消息。我们的消息包含公钥签名[33]，消息认证码[36]和由抗碰撞哈希函数生成的消息摘要[32]。我们用$$<{m}>_{\sigma_i}$$表示一条由节点 i 签名的消息 m，消息摘要为D(m) 。我们遵循对消息摘要进行签名并将其附加到消息明文之后的惯例，而不是对整个消息进行签名（$$<{m}>_{\sigma_i}$$用的就是这种方式 ）。所有replica都知道其他人的公钥从而可以验证其签名。

我们假想有一个非常强大的对手，它可以协调所有恶化节点，延迟通讯，或者说延迟那些正常的节点，从而对复制服务造成最大的破坏。我们必须假设这个对手不能无限期地延迟非恶化的节点。我们还假设该对手（及其控制的恶化节点）存在算力上限，因而（大概率）不能颠覆上述密码学技术。例如，该对手无法生成非恶化节点的有效签名，无法从摘要中反推出其所包含的信息， 或找到两条具有相同摘要的消息。我们使用的密码学技术正是具有这些特性[33、36、32]。 

## 3、服务属性

我们的算法适用于任何包含状态及其操作的确定性复制服务。这里所指的操作并不局限于对服务状态的各部分进行简单的读写；而是可以利用状态和操作参数执行任意的确定性计算。client向复制服务发出请求以调用操作，然后进入阻塞状态，等待响应。复制服务由n个replica实现。如果client和replica都遵循第4节的算法，并且没有攻击者可以伪造他们的签名，则它们不会出错。 

在最多不超过$$[\frac{n-1}{3}]$$个replica出现恶化时，该算法能够同时满足Safty和Liveness两大属性。Safty意味着复制服务满足线性一致性要求[14]（用于解决拜占庭故障client[4]的改进）：它的表现就像一个中心化的实现，一次执行一个原子操作。Safty要求限制恶化replica的数量，因为恶化replica可以做出任意行为，例如，它可以破坏其状态。

无论有多少恶化client使用该服务（即使它们与恶化replica合并），都能满足Safty：非恶化client以一致的方式观察恶化client执行的所有操作。特别是，如果服务操作旨在保留服务状态的某些不变式，则表示恶化client无法破坏那些不变式。 

Safty属性不足以抵御恶化client，例如，在文件系统中，恶化client可以将垃圾数据写入某些共享文件。但是，我们可以通过使用访问控制机制来限制恶化client可能造成的损害：我们对client进行身份验证，如果发出请求的client无权调用该操作，则拒绝访问。相应的，需要提供用于更改client访问权限的操作。由于该算法确保所有client能够一致的观察到撤消访问权限操作的后果，因而提供了一种可从恶化client的攻击中恢复的强大机制。 

该算法不依靠同步性来满足Safty，那它就必须依靠同步性来满足Liveness；否则它可能被用来在异步系统中实现共识，但这是不可能的[9]。我们确保Liveness，即client最终会收到对他们请求的响应，是在最多不超过$$[\frac{n-1}{3}]$$个恶化replica，并且delay(t)的增速不会永远都比 t 的增速更快的前提下。这里的delay(t)是指第一次发送消息的时刻 t 与它的目标接收到消息之间的时间间隔（假设发送方一直重新发送消息直到被接收）。（可以在[4]中找到更精确的定义。）这是关于同步性的一个相当弱的假设，在任何实际系统中都可能是成立的，因为网络问题最终会被修复，但它可以使我们能够规避[9]中的不可能原理。  

我们的算法的弹性是最优的：当最多 f 个replica出现恶化时，3f+1 是使异步系统满足Safty和Liveness属性的最少replica数量（请参见[2]作为证明）。之所以需要这么多replica，是因为在有 f 个replica可能是恶化的并且不响应的情况下，n-f 个replica相互通信之后必须能够继续运转下去。但是，没有响应的 f 个replica可能不是恶化的，反而有响应的 f 个replica可能才是恶化的。即使这样，仍然必须保证有足够的响应，以使非恶化replica的响应数大于恶化replica的响应数，即 n-2f>f 。因此 n>3f 。 

该算法无法解决隐私容错性的问题：恶化replica可能会将信息泄漏给攻击者。在一般情况下，提供隐私容错性是不可行的，因为服务操作需要使用其参数和服务状态执行任意计算。replica显然需要此信息才能有效执行此类操作。使用秘密共享方案[35]获得隐私性是有可能的，即使在恶化replica[13]存在阈值，参数和部分状态对服务操作不透明的情况下。 我们计划在未来研究这些技术。 

## 4、算法 

我们的算法是状态机复制的一种形式[17，34]：将服务建模为状态机，该状态机在分布式系统中的不同节点之间进行复制。每个状态机replica都维护服务状态并实现服务操作。我们用 R 表示一组replica，并使用{0,...,|R|-1} 中的整数标识每个replica。为简单起见，我们假设 |R|=3f+1 ,其中 f 表示最多可能是恶化replica的个数；尽管可以允许超过3f+1个以上的replica，但多出来的replica只会降低性能（因为要交换更多、更大的消息），而没有提供更高的容错能力。 

replica的运转是通过一系列被称为视图的结构实现。在一个视图中，一个replica是主副本（以下引用原文称为primary），其他replica是备份副本（以下引用原文称为backup）。视图是连续编号的。假定视图的视图编号为 v ，primary的副本编号为 p ，那么计算公式为 p=v mod |R| 。当primary出现错误时，将执行视图切换。Viewstamped Replication算法[26]和Paxos算法[18]使用类似的方法来容忍良性故障（如第8节讨论）。

> 译者注：需要很明确的区分replica、backup、primary的区别，对于理解算法非常重要。简单来说，replica=primary+backup

该算法的工作原理大致如下： 

\1. client向primary发起服务调用请求 

\2. primary将请求广播给其他backup 

\3. replica执行请求并向client返回响应

\4. client等待来自不同replica的 f+1 个相同的响应；这个响应就是本次请求的结果。

跟所有状态机复制技术[34]一样，我们对replica施加两个条件：（1）它们必须是确定性的（即，在指定状态、指定参数的条件下，执行结果必须始终相同），并且（2）它们必须以相同的状态开始。在这两个条件的限定下，该算法可以保证所有非恶化replica即使发生异常失败，也都能够对所有请求的执行顺序达成共识，从而确保Safty属性。 

本节的其余部分描述了该算法的简化版本。我们省略了有关空间不足节点如何从故障中恢复的讨论。我们还省略了与消息重传有关的详细描述。此外，我们假定使用数字签名而不是基于消息认证码这种更高效的方案来实现消息认证；第5节将深入探讨这个问题。在[4]中对于该算法使用I/O自动机模型[21]进行了更详细正式的论述。 

### 4.1 **客户端（**The Client）

一个client c 通过向primary发送$$<{REQUEST,o,t,c}>_{\sigma_c}$$消息来请求执行操作 o。时间戳 t 用于确保client的请求只被执行一次。c 发起的所有请求的时间戳是完全有序的，确保后续的请求比之前的请求具有更高的时间戳； 例如，时间戳可以是请求发起时client的本地时间。 

replica发送给client的每条消息都包含当前视图编号，从而允许client跟踪视图及当前primary。client将请求通过点对点消息发送给它认为的当前primary。primary使用下一部分中描述的协议将请求原子性的广播到所有replica。 

replica将请求的响应直接发送给client，响应格式为$$<{REPLY,v,t,c,i,r}>_{\sigma_i}$$，其中，v 表示当前视图编号，t 表示对应请求的时间戳，i 表示replica编号，r 表示请求的执行结果。 

client在等到来自不同replica的 f+1 个具有有效签名，并带有相同 t 和 r 的响应之后，才会接受执行结果 r。这足以确保结果是有效的，因为最多只有 f 个replica可能是恶化的。 

如果client没有足够快地收到响应，则它将请求广播到所有replica。如果请求已被处理，则replica仅重新发送响应；replica会记录它们发送给每个client的最后一条响应消息。否则，如果replica不是primary，它将请求转发给primary。如果primary没有将请求广播到整个replica群组，则其将最终会被足够多的replica怀疑是恶的，从而引发视图切换。 

在本文中，我们假设client在发送下一个请求之前等待前一个请求完成。但是我们可以允许client发起异步请求，但保留对这些请求进行排序的约束。 

### 4.2 **常规操作** 

replica的状态包括服务状态，消息日志（包含其所有已接受的消息）以及当前视图编号。我们将在第 4.3 节中描述如何清理日志。

当primary p 收到client的请求 m 时，primary将启动一个三阶段协议，原子性的将请求广播到所有replica。primary将立即启动协议，除非当前正在处理的消息数超过了协议约定的最大值。出现这种情况时，它将缓冲请求。缓冲的请求随后作为一个组进行广播，以减少高负载下的消息数量和CPU开销。这种优化策略类似于事务类系统中的组提交[11]。为简单起见，我们在下面的描述中忽略了这种优化。 

三阶段是指预准备（以下引用原文称为pre-prepare），准备（以下引用原文称为prepare）和提交（以下引用原文称为commit）。pre-prepare和prepare阶段用于对同一视图中发起的请求进行定序，即使发起定序请求的primary是恶化的也没有问题。prepare和commit阶段用于确保在跨视图时，提交的请求也是被定序的。

在pre-prepare阶段，primary为每个请求分配一个请求序号 n，然后将pre-prepare消息连同 m 一起广播给所有replica，并将该消息追加写入到其日志中。该消息的格式为$$<<{PRE-PREPARE,v,n,d}>_{\sigma_p},m>$$，其中v 表示当前请求所处的视图编号， m 是client的请求内容，而 d 是 m 的摘要。 

为了使pre-prepare消息尽量小，请求内容 m 并不包含在该消息中。这很重要，因为pre-prepare消息的作用是在发生视图切换时用于证明在视图 v 中已为该请求分配了序号 n。此外，它将对请求定序协议与请求传输协议进行了分离，允许我们可以针对协议消息这种小消息和大型请求这种大消息分别进行传输优化。

backup接受一个pre-prepare消息的前提条件：

- 请求和pre-prepare消息中的签名均正确，且 d 是 m 的摘要; 
- 当前处在视图 v 中; 
- 尚未接受视图编号为 v ，请求序号为 n ，但却包含不同请求摘要 d 的pre-prepare消息；
- pre-prepare消息中的序号 n 在低水位线 h 和高水位线 H 之间。

最后一个条件是防止恶化primary通过选择一个很大的序号来耗尽请求序号空间的问题。我们将在4.3 节中讨论 H 和 h 如何解决这个问题。

如果backup i 接受了$$<<{PRE-PREPARE,v,n,d}>_{\sigma_p},m>$$消息，它将通过将$$<{PREPARE,v,n,d,i}>_{\sigma_i}$$消息广播给所有其他replica而后进入prepare阶段，并将这两个消息追加写入到其日志中。否则，它什么都不做。

replica（包括primary）接受所有有效的prepare消息，并将它们添加到其日志中，前提是其签名正确，其视图编号等于该replica的当前视图编号，并且其请求序号在 h 和 H 之间。 

我们定义谓词prepared(m,v,n,i) ，其为 true的条件是，当且仅当replica i 将以下信息已写入其消息日志中： 请求 m ，与 m 对应的视图编号为 v 、请求序号为 n 的pre-prepare消息，以及 2f 个来自不同**backup**与该pre-prepare消息匹配的prepare消息。replica匹配prepare和pre-prepare的规则是检查是否具有相同的视图编号，请求序号和请求摘要。 

> 译者注：注意这里说的是*backup*，因此是不包含*primary*的，因为*primary*不会发送*prepare*消息。论文这里没有表述很清楚，需要分情况详细描述：*1.* 对于*primary*来说，需要*2f*个*backup*的*prepare*，因为*primary*自身是没有*prepare*的；*2.*对于*backup*来说，需要*2f*个包含自己在内的*backup*的*prepare*。总的来说，加上该*replica*自身其实都是满足了*2f+1*个确认，与全文思想是统一的。

该算法的pre-prepare和prepare阶段确保了所有非恶化replica对一个视图中的请求定序达成一致。更准确地说，它们确保了以下不变式：如果prepared(m,v,n,i) 为true，则对于任何非恶化replica j（包括 i=j ）和任何使得 D(m)$$\neq$$D(m')成立的 m'，prepared(m’,v,n,j)都为false。这是毋庸置疑的，因为prepared(m,v,n,i)和|R|=3f+1这两个条件就意味着至少有 f+1 个非恶化replica已经在视图 v 中用请求序号 n 为 m 发送了pre-prepared或prepared消息。因此，要使prepared(m',v,n,j)为true，这些非恶化的replica中至少有一个需要发送两个矛盾的prepare消息（如果对于视图 v 的primary来说，则为pre-prepare消息），即两个prepare消息具有相同的视图编号和请求序号，但消息摘要不同。但这是不可能的，因为这些replica是非恶化的。最后，我们对消息摘要算法强度的假设确保了“m'$$\neq$$m但D(m)=D(m')”的概率可以忽略。 

当prepared(m,v,n,i) 变为true时，replica i 广播一个$$<{COMMIT,v,n,D(m),i}>_{\sigma_i}$$消息给其他replica。这将启动commit阶段。replica接受commit消息并将它们插入其日志中，前提是其签名正确，其视图编号等于replica当前的视图编号，并且请求序号介于 h 和 H 之间。

我们按以下方式定义committed 和 committed-local 两个谓词：committed (m,v,n)为true，当且仅当在非恶化replica集合中存在 f+1 个replica，其中所有 i 的prepared(m,v,n,i) 都为true；committed-local (m,v,n,i)为true，当且仅当prepared(m,v,n,i) 为true，并且 i 已经接受了2f+1个来自于不同replica的与对应的pre-prepare消息匹配的commit消息（可能包括其自己的commit消息）；commit与pre-prepare匹配的规则是，具有相同的视图编号，请求序号和请求摘要。 

commit阶段可确保以下不变式：对于某些非恶化replica i ，如果committed-local (m,v,n,i)为true，则committed (m,v,n)也为true。这个不变式和在第 4.4 节中将描述的视图切换协议可确保所有无恶化replica能够对各自在本地提交的请求的请求序号达成一致，即使这些请求在每个replica上是在不同的视图中提交也没有问题。此外，它还保证在一个非恶化replica上本地提交的任何请求最终都会在f+1个或更多非恶化replica上提交。 

每个replica i 在 committed-local (m,v,n,i)为true，并且请求序号更小的请求都依序串行执行完成之后，将执行 m 请求的操作。这样可以确保所有非恶化replica按照相同的顺序执行请求，从而满足Safty属性。执行完成后，replica发送一个响应给client。replica将丢弃那些请求，其请求时间戳小于最后一次响应的的请求时间戳，以保证不会重复执行。 

我们不依赖于有序消息传递，因此replica可能会无序commit请求。这无关紧要，因为它会保留pre-prepare，prepared和commit消息记录，直到相关的请求可以被执行为止。 

图 1 显示了该算法在没有primary故障的正常情况下的操作。replica 0 为主副本，replica 3 为恶化副本，而 C 为client。



图1 常规操作

### 4.3 **垃圾回收** 

本节讨论从日志中清理消息的机制。为了保证Safty属性，必须将消息保存在replica的日志中，直到确信相关的请求已由至少f+1个非恶化replica执行，并且在视图切换时能够向别人证明这一点。此外，如果某些replica丢失了一些被所有的非恶化replica都已经清理掉的消息，则需要通过同步全部或部分服务状态使其得到更新。因此，replica也需要一些证明显示其状态是正确的。 

在每次执行操作后都去生成这些证明是代价高昂的。取而代之的是定期生成方式，比如当被执行的请求序号能够被某个常数（例如 100）整除时。我们将达到定期生成条件的请求被执行之后所产生的状态称为检查点，并规定带有证明的检查点就是稳定检查点。 

一个replica会维护服务状态的多个逻辑备份：最后一个稳定检查点，零个或多个不稳定检查点以及当前状态。正如第6.3节所述，写时复制技术（Copy-on-write）可用于减少存储状态的额外备份而带来的空间开销。 

检查点的正确性证明生成方式如下。当一个replica i 产生检查点时，它将广播一条检查点消息（以下引用原文称为checkpoint消息）$$<{CHEKCPOINT,n,d,i}>_{\sigma_i}$$到其他replica，其中 n 是最后一个请求序号，该请求的执行结果将反映在其状态中，而 d 是状态的摘要。每个replica都将一直接收并保存检查点消息在其日志中，直到其拥有 2f+1 个（可以包括自己在内）请求序号为 n、状态摘要为 d、且由不同replica签名的checkpoint消息。这 2f+1 条消息是检查点的正确性证明。 

具有证明的检查点将变得稳定，replica将丢弃其日志中请求序号小于或等于n的所有pre-prepare、prepare和commit消息；它还会丢弃所有较早的检查点和checkpoint消息。

检查点证明的计算是高效的，因为状态摘要可以使用第6.3节中讨论的增量加密[1]进行计算，并且很少生成证明。检查点协议用于推进低水位线标记和高水位线标记（这限制了哪些消息可以被接收）。低水位标记 h 等于最后一个稳定检查点的请求序号。高水位线标记H=h+k，其中 k 足够大，因此replica不会因等待检查点稳定而停止工作。例如，如果检查点是每100个请求生成，k 则可能是200。 

### 4.4 **视图切换** 

视图切换协议确保了当primary出现故障时，系统仍然能够继续运转，从而满足Liveness属性。视图切换是由超时触发的，超时防止了backup无限期地等待执行请求。如果backup收到一个有效请求而尚未执行它，则backup处于等待执行请求状态。当backup接收到请求时，它将启动一个全新的计时器。当其不再等待执行该请求时，backup将停止计时器，但如果此时开始等待执行其他请求，则将计时器重启。 

如果backup i 的计时器在视图 v 中超时，则 i 将发起视图切换以将系统切换至视图 v+1 。它将停止接收消息（checkpoint，view-change和new-view消息除外），并广播一个视图切换消息（以下引用原文称为view-change消息）$$<{VIEW-CHANGE,v+1,C,P,i}>_{\sigma_i}$$给所有replica。n 是 i 已知的最近一个稳定检查点 S 的请求序号， C 是证明 S 正确性的 2f+1 个有效的checkpoint消息集合，P 是一个包含集合$$P_m$$的集合，其中 m 表示 i 已经prepare过的每一个请求序号大于 n 的请求。每个集合$$P_m$$包含一个有效的pre-prepare消息（不包含对应的client请求）和 2f 个匹配项，即是视图编号、请求序号和请求摘要相同，由不同backup签名的有效的prepare消息。

当视图 v+1 的primary p 从其他replica接收到 2f 个切换至视图 v+1 的有效view-change消息时，它将广播一个新视图消息（以下引用原文称为new-view消息）$$<{NEW-VIEW,v+1,V,O}>_{\sigma_p}$$给所有其他replica，其中 V 是一个集合，包含primary接收的有效view-change消息以及primary已经发送（或还未来得及发送的）的切换至视图 v+1 的view-change消息，而 O 是一组pre-prepare消息（不附带请求内容）。O 的计算如下： 

\1. primary确定 V 中最新的稳定检查点的请求序号min-s和 V 中所有prepare消息里请求序号的最大值max-s。 

\2. primary为介于min-s和max-s之间的每个序号 n 以视图 v+1 分别创建新的pre-prepare消息。有两种情况：（1）在 V 中的某些view-change消息的 P 集合中，已经至少存在一个请求序号 n 了，或者（2）不存在这样的情况。在第一种情况下，primary创建一个新消息$$<{PRE-PREPARE,v+1,n,d}>_{\sigma_p}$$，其中 d 是 V 中请求序号为 n ，且具有最高视图编号的pre-prepare消息的请求摘要。在第二种情况下，它将创建一个新的pre-prepare消息$$<{PRE-PREPARE,v+1,n,d^{null}}>_{\sigma_p}$$，其中$$d^{null}$$是一个特殊的空请求的请求摘要；空请求像其他请求一样遵从协议，但是它执行的是空操作。（Paxos [18]使用了类似的技术来补齐。） 

接下来，primary将 O 中的消息追加写入到其日志中。如果min-s大于其最新的稳定检查点的请求序号，则primary还将在其日志中插入请求序号为min-s的检查点的稳定性证明，并如第4.3节中所述清理日志信息。然后它进入视图 v+1 ：此时，它可以接受视图 v+1 的消息。

backup将接受切换至视图 v+1 的new-view消息，如果其签名正确，包含的view-change消息都是切换至视图 v+1 的有效消息，并且集合 O 是正确的；backup通过执行类似于primary创建 O 的计算过程来验证 O 的正确性。然后，跟primary一样将这些新信息追加写入到其日志中，为 O 中每个消息广播一个prepare消息到所有其他replica，把这些prepare消息追加写入到其日志中，并进入视图 v+1 。 

此后，按第4.2节中所述继续执行协议。replica其实是按照协议重做了min-s和max-s之间的所有消息，但是它们避免重新执行client的请求（通过使用它们存储并最后发送给每个client的响应信息）。 

一个replica可能缺少某些请求消息 m 或一个稳定检查点（因为他们不会在 new-view消息中发送）。它可以从另一个replica中获取丢失的信息。例如，replica i 可以从任意一个在 V 中已经证明了其检查点消息的正确性的那些replica中获得丢失的检查点状态 S 。由于这样的replica中有 f+1 个是正确的，因此replica i 将始终能够获得 S 或一个稍后验证过的稳定检查点。我们可以通过对状态进行分区并为每个分区标记上修改它的最后一个请求序号，来避免发送整个检查点。要将replica同步至最新，只需发送并更新它那些已经过期的分区即可，而不是整个检查点。 

### 4.5 **正确性** 

本节概述了该算法满足安全性（Safty）和活性（Liveness）的证明。详细信息可以在[4]中找到。 

#### 4.5.1 **安全性** **（**Safty）

如前所述，如果所有非恶化replica对本地提交的请求序号达成一致，则该算法满足安全性。在第4.2 节中 ，我们提出了如果prepared(m,v,n,i) 为true，则对于任何非恶化replica j（包括 i=j ）和任何使得D(m)$$\neq$$D(m')成立的m'，prepared(m',v,n,j)都为false。这意味着两个非恶化replica在同一视图中对各自本地提交的请求序号达成了共识。

视图切换协议确保了所有非恶化replica在不同视图中对各自本地提交的请求序号也能达成共识。一个请求m要在视图 v 中的一个非恶化replica上，以请求序号 n 进行本地提交，只有在committed (m,v,n)为true时。这就意味着存在一个集合$$R_1$$，至少包含 f+1 个非恶化replica，使得集合中的每个replica i 的prepared(m,v,n,i) 都是true。 

非恶化replica在未收到切换至视图v’的new-view消息时（因为只有那时才能进入视图 v’），将不接受视图 v’（v’>v）的pre-prepare消息。但是，切换至视图v’（v’>v）的任何正确的new-view消息，都会包含来自集合$$R_2$$（包含 2f+1 个replica）中的每个replica i 的正确的view-change消息。 由于有 3f+1 个replica，集合$$R_1$$和$$R_2$$的交集至少包含一个非恶化replica K 。K 的view-change消息将确保以下事实：（1）前一个视图中已经prepare的请求 m 将会传播到后续视图继续处理，（2）除非new-view消息中包含一个view-change消息，其中带有请求序号大于 n 的稳定检查点。在第一种情况下，该算法将使用相同的请求序号 n 以及新的视图编号为请求 m 重新执行三阶段协议。这很重要，因为它可以防止在之前的视图中任何已经分配了请求序号 n 的其他请求无法被提交。在第二种情况下，新视图中没有replica会接受序列号小于 n 的任何消息。无论哪种情况，replica都将对本地提交的请求序号为 n 的请求达成一致。

#### 4.5.2 **活性（**Liveness）

为了满足活性，如果replica无法执行请求时，则必须切换到一个新的视图。但更重要的是，当在同一视图中至少存在 2f+1 个非恶化replica时，应使超时时间最大化，并确保在执行某些请求执行之前，该时间应呈指数增长。我们通过三种方式实现这些目标。 

首先，避免过早发起视图切换，一个广播了切换至视图 v+1 的view-change消息的replica将要等待 2f+1 个切换至视图 v+1 的view-change消息, 然后启动它的计时器等待一段超时时间T。如果在这个replica收到有效的切换至视图 v+1 的new-view消息之前，或者这个replica在新视图中开始执行之前未执行过的请求之前，定时器就超时了，它将开始发起切换至视图 v+2 的请求，但是这次它将等待 2T 的超时时间，才能启动下一次切换至视图 v+3 的视图切换。 

其次，如果一个replica从其他replica接收到一组 f+1 个有效的、视图编号大于当前视图编号的view-change 消息，它也会用该组中最小的视图编号发起 view-change 消息，即使其计时器还未超时。这样可以防止太晚启动视图切换。 

第三，恶化replica无法通过强行频繁的视图切换来阻碍算法的执行。恶化replica无法通过发送view-change消息就能引起视图切换，因为只有至少 f+1 个replica发送view-change消息时才会启动视图切换，但是当primary是恶化的时，它可能导致视图切换（通过不发送消息或发送错误消息）。但是，因为视图 v 的primary p 是通过 p=v mod |R| 产生，所以primary不会连续超过 f 个视图都是恶化的。 

以上三种技术可以保证Liveness属性，除非消息延迟时间的增速永远都比超时时间的增速更快，但这在实际系统中是不可能的。

### 4.6 非确定性

状态机副本必须是确定性的，但是许多服务都涉及某种形式的非确定性。例如，通过读取服务器的本地时钟来设置NFS中的最后修改时间。如果每个replica都独立完成此操作，则非恶化replica的状态将有所不同。因此，需要一种机制来确保所有replica选择相同的值。通常，client无法选择该值，因为它没有足够的信息。例如，它不知道它的请求跟其他client发起的并发请求相比将会被如何定序。取而代之的是，由primary来独立地或基于backup提供的值来进行选择。 

如果primary独立选择不确定性值，则它将该值与相关的请求关联起来，并执行三阶段协议以确保所有非恶化replica对于关联了请求和值的请求序号达成一致。这样可以防止恶化的primary通过给不同的replica发送不同值的方式，致使replica的状态出现不一致。但是，一个恶化的primary可能会向所有replica发送相同的但不正确的值。因此，replica必须能够仅基于服务状态来确定值是否正确（以及如果不正确该如何处理）。 

该协议适用于大多数服务（包括NFS），但有时replica必须参与选择值以满足服务的规范。这可以通过在协议中添加额外的阶段来实现：primary获取backup建议的、经过验证的值，将 2f+1 个建议值与对应的请求关联在一起，并为这个关联后的消息启动三阶段协议。replica通过对 2f+1 个值及其自身的状态进行确定性计算来选择值，例如，取中位数。在通常情况下，这个额外的阶段可以被优化。例如，如果replica需要的值与他们的本地时钟 “足够接近”，当replica之间会在某段增量区间内进行时钟同步时，可以不必增加这个额外的阶段。

## 5、优化策略

本节介绍了一些优化策略，可提高算法在常规情况下的性能。所有优化都保留了Liveness和Safty属性。

### 5.1 减少通信

我们使用三个优化策略来降低通信成本。第一个是避免发送绝大多数的大型响应。client可以指定replica返回包含完整结果的响应；所有其他replica发送仅包含结果摘要的响应。对于大型响应而言，摘要可以使clients验证结果正确性的同时，显著减少网络带宽和 CPU消耗。如果client没有收到来自指定replica的正确响应结果，它将用普通的模式重新发送请求，请求所有replica发送完整响应。

第二个优化策略是将请求被执行的消息延迟的次数从5次减少到4次。一旦一个请求的prepared谓词被满足时，replica就临时执行该请求，replica的状态反映了所有那些较低请求序号的请求的执行结果，并且这些请求都已经被提交过了。执行请求后，replica将临时发送响应给client。client等待 2f+1 个相匹配的临时响应。如果收到这么多请求，则可以保证该请求最终会被提交。否则，clients将重新发送该请求，并等待 f+1 个正式响应。 

如果视图发生切换，则临时执行的请求将会中止，并由空请求代替。在这种情况下，replica将其状态还原到new-view消息中最后一个稳定检查点或它的最后一个检查点状态（取决于哪个具有更高的序号）。 

第三个优化策略提高了不会修改服务状态的只读操作的性能。client将只读请求广播到所有replica。在检查了请求已被验证，client具有访问权限以及确实是只读操作之后，这些replica立即以其暂时状态执行请求。但它们只会在其暂时状态对应的所有请求都已提交后才发送响应。这对于防止client脏读（获取未提交状态）是必要的。client等待来自不同replica的 2f+1 个相同响应。如果同时发生了对于执行结果有影响的并发写入，则client可能无法获得 2f+1 个这样的响应。在这种情况下，在它的重发计时器到期后，它将以常规的读写请求的方式重发该请求。 

### 5.2 **加密技术** 

在第4节中，我们描述了一种使用数字签名对所有消息进行身份验证的算法。但是，实际上，我们仅将数字签名用于很少发送的view-change和new-view消息上，而使用消息验证码（MACs）对所有其他消息进行验证。这消除了以前系统中的主要性能瓶颈[29，22]。 

但是，相对于数字签名，MACs 有一个基本限制-无法对第三方证明消息的真实性。在第4节中介绍的算法和之前的拜占庭容错算法 [31，16]中的状态机复制都依赖了数字签名的额外功能。我们改进了我们的算法，以利用以下特定的不变式的优势来规避该问题，例如，不存在两个不同的请求可以以相同的视图编号和请求序号在两个非恶化replica上完成prepare的不变式。 在[5]中详细描述了改进的算法。在这里，我们概述了使用MACs的主要实现。 

MACs的生成速度比数字签名快三个数量级。例如，一个200MHz的Pentium Pro要花费43ms来为一个MD5摘要生成1024-bit modulus RSA签名，并花费0.6ms来验证签名[37]，而使用相同的硬件，我们的方法仅需要 10.3$$\mu s$$就可以为一个64-byte的消息生成MAC。还有其他生成签名更快的公钥加密系统，例如椭圆曲线公钥加密系统，但验签速度较慢[37]，而我们的算法要求每个签名都要经过多次校验。

每个节点（包括活跃的client）与每个replica共享一个16-byte的秘密会话密钥。我们通过将消息和密钥拼接起来之后应用MD5算法，从而计算MAC。我们仅使用MD5摘要低位有效的10个byte，而不是全部的16个byte。这种截断具有减少 MAC 大小的明显优势，并且还提高了它们对某些攻击的弹性[27]。这是秘密后缀方法[36]的一种变体，只要MD5具有抗冲突性，该方法就很安全[27，8]。 

响应消息中的数字签名被单个MAC替换就足够了，因为这些消息具有单一的预期接收方。所有其他消息（包括client请求，但不包括视图切换相关的消息）中的签名被我们称为身份验证器（以下称authenticator）的MAC向量替换。authenticator为除发送者以外的每个replica都生成一个条目；每个条目都是使用发送者和与该条目对应的replica之间的共享密钥计算出的MAC。 

验证authenticator的时间是恒定的，但是生成authenticator的时间随replica数量的增加而线性增加。这不是问题，因为我们预计不会有大量的replica，并且MAC和数字签名计算之间存在巨大的性能差距。此外，我们可以高效地生成authenticator；通过对消息做一次MD5计算，然后将生成的上下文和相应的会话密钥再为向量中的每个条目计算MD5。例如，在具有37个replica的系统中（即可以容忍同时出现12个恶化的系统），authenticator的生成速度仍然比1024-bit modulus RSA签名快两个数量级。

authenticator的大小与replica个数呈线性增长，但增长缓慢：相当于$$30\times[\frac{n-1}{3}]$$bytes。当$$n\leq 13$$时（即最多可容许同时出现4个恶化的系统） ，authenticator比1024-bit modulus RSA签名更小，这也是我们预计的大多数的使用场景。



## References

[1] M. Bellare and D. Micciancio. A New Paradigm for Collisionfree Hashing: Incrementality at Reduced Cost. In *Advances in* 

*Cryptology – Eurocrypt 97*, 1997. 

[2] G. Bracha and S. Toueg. Asynchronous Consensus and Broadcast Protocols. *Journal of the ACM*, 32(4), 1995. 

[3] R. Canneti and T. Rabin. Optimal Asynchronous Byzantine Agreement. Technical Report #92-15, Computer Science Department, Hebrew University, 1992. 

[4] M. Castro and B. Liskov. A Correctness Proof for a Practical Byzantine-Fault-Tolerant Replication Algorithm. Technical Memo MIT/LCS/TM-590, MIT Laboratory for Computer Science, 1999. 

[5] M. Castro and B. Liskov. Authenticated Byzantine Fault Tolerance Without Public-Key Cryptography. Technical Memo MIT/LCS/TM-589, MIT Laboratory for Computer Science, 1999. 

[6] F. Cristian, H. Aghili, H. Strong, and D. Dolev. AtomicBroadcast: From Simple Message Diffusion to Byzantine Agreement. In *International Conference on Fault Tolerant Computing*, 1985. 

[7] S. Deering and D. Cheriton. Multicast Routing in Datagram Internetworks and Extended LANs. *ACM Transactions on Computer Systems*, 8(2), 1990. 

[8] H. Dobbertin. The Status of MD5 After a Recent Attack. *RSA Laboratories’ CryptoBytes*, 2(2), 1996. 

[9] M. Fischer, N. Lynch, and M. Paterson. Impossibility of Distributed Consensus With One Faulty Process. *Journal of the ACM*, 32(2), 1985. 

[10] J. Garay and Y. Moses. Fully Polynomial Byzantine Agreement for n 3t Processorsin t+1 Rounds. *SIAM Journal of Computing*, 

27(1), 1998. 

[11] D. Gawlick and D. Kinkade. Varieties of Concurrency Control in IMS/VS Fast Path. *Database Engineering*, 8(2), 1985. 

[12] D. Gifford. Weighted Voting for Replicated Data. In *Symposium on Operating Systems Principles*, 1979. 

[13] M. Herlihy and J. Tygar. How to make replicated data secure. *Advances in Cryptology (LNCS 293)*, 1988. 

[14] M. Herlihy and J. Wing. Axiomsfor Concurrent Objects. In *ACM Symposium on Principles of Programming Languages*, 1987. 

[15] J. Howard et al. Scale and performance in a distributed fifile system. *ACM Transactions on Computer Systems*, 6(1), 1988. 

[16] K. Kihlstrom, L. Moser, and P. Melliar-Smith. The SecureRing Protocols for Securing Group Communication. In *Hawaii International Conference on System Sciences*, 1998. 

[17] L. Lamport. Time, Clocks, and the Ordering of Events in a Distributed System. *Commun. ACM*, 21(7), 1978. 

[18] L. Lamport. The Part-Time Parliament. Technical Report 49, DEC Systems Research Center, 1989. 

[19] L. Lamport, R. Shostak, and M. Pease. The Byzantine Generals Problem. *ACM Transactions on Programming Languages and Systems*, 4(3), 1982. 

[20] B. Liskov et al. Replication in the Harp File System. In *ACM Symposium on Operating System Principles*, 1991. 

[21] N. Lynch. *Distributed Algorithms*. Morgan Kaufmann Publishers, 1996. 

[22] D. Malkhi and M. Reiter. A High-Throughput Secure Reliable Multicast Protocol. In *Computer Security Foundations Workshop*, 1996. 

[23] D. Malkhi and M. Reiter. Byzantine Quorum Systems. In *ACM Symposium on Theory of Computing*, 1997. 

[24] D. Malkhi and M. Reiter. Unreliable Intrusion Detection in Distributed Computations. In *Computer Security Foundations Workshop*, 1997. 

[25] D. Malkhi and M. Reiter. Secure and Scalable Replication in Phalanx. In *IEEE Symposium on Reliable Distributed Systems*, 1998. 

[26] B. Oki and B. Liskov. Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems. In *ACM Symposium on Principles of Distributed Computing*, 1988. 

[27] B. Preneel and P. Oorschot. MDx-MAC and Building Fast MACs from Hash Functions. In *Crypto 95*, 1995. 

[28] C. Pu, A. Black, C. Cowan, and J. Walpole. A Specialization Toolkit to Increase the Diversity of Operating Systems. In *ICMAS Workshop on Immunity-Based Systems*, 1996. 

[29] M. Reiter. Secure Agreement Protocols. In *ACM Conference on Computer and Communication Security*, 1994. 

[30] M. Reiter. The Rampart Toolkit for Building High-Integrity Services. *Theory and Practice in Distributed Systems (LNCS 938)*, 1995. 

[31] M. Reiter. A Secure Group Membership Protocol. *IEEE Transactions on Software Engineering*, 22(1), 1996. 

[32] R. Rivest. The MD5 Message-Digest Algorithm. Internet RFC-1321, 1992. 

[33] R. Rivest, A. Shamir, and L. Adleman. A Method for Obtaining Digital Signatures and Public-Key Cryptosystems. *Communications of the ACM*, 21(2), 1978. 

[34] F. Schneider. Implementing Fault-Tolerant Services Using The State Machine Approach: A Tutorial. *ACM Computing Surveys*, 22(4), 1990. 

[35] A. Shamir. How to share a secret. *Communications of the ACM*, 22(11), 1979. 

[36] G. Tsudik. Message Authentication with One-Way Hash Functions. *ACM ComputerCommunicationsReview*, 22(5), 1992. 

[37] M. Wiener. Performance Comparison of Public-Key Cryptosystems. *RSA Laboratories’ CryptoBytes*, 4(1), 1998.