到目前为止，我们希望让您相信 consistency models 和 cache coherence protocols 是复杂而微妙的。在本章中，我们讨论了严格指定 consistency models 和 coherence protocols 的方法，并探索它们允许的行为。我们还讨论了指定其实现和验证其正确性的方法。

我们首先简要概述指定并发系统的两种方法，操作法 (operational method) 和公理法 (axiomatic method)，并重点关注如何使用这些方法指定 consistency models 和 coherence protocols（第 11.1 节）。对于给定的形式化的 consistency 规范，我们将讨论如何自动探索其行为（第 11.2 节）。最后，我们将快速浏览如何验证 (validating) consistency 的实现，其中，同时涵盖了形式化方法和基于测试的方法（第 11.3 节）。

## 11.1 Specification
一份规范，通过精确定义系统的合法行为的集合，作为系统用户与系统实现者之间的契约。一份规范需要回答以下问题：系统允许的行为是什么？相反，一个实现需要回答以下问题：系统如何执行规定的行为？

我们所说的行为 (behavior) 是什么意思呢？是说，系统通过一组动作 (actions) 与其用户（或环境）交互：(1) *输入动作*：从用户到系统的动作；(2) *内部动作*：系统内部发生的动作；(3) *输出动作*：从系统到用户的动作。其中，只有输入和输出动作对用户是可见的，而内部动作不是。因此，一系列（可观察的）输入和输出动作定义了系统的行为。我们感兴趣的是两类行为属性：*安全属性 (safety property)* 和*活跃属性 (liveness property)*。

安全属性断言坏事不应该发生，即，它指定了哪些可观察动作的序列是合法的。对于并发系统，由于不确定性，通常有多个合法序列。然而，安全属性不足以完全指定一个系统。考虑一个系统，它接受了一个输入动作并简单地 halts 了；尽管它没有表现出任何不良情况，但这样的系统显然没有用。这就是为什么规范还应当包括活跃属性，它断言好事最终一定会发生。

在我们进一步解释如何形式化指定行为之前，让我们首先考虑一下 consistency models 和 coherence protocols 的可观察动作是什么。

**Consistency models: Observable actions and behavior**

一个 memory consistency model 是软件（用户）和硬件（实现者）之间的契约。给定包含 per-thread 的 loads 和 stores 序列的多线程程序，memory model 指定每个 load 必须返回的值。因此，相关的输入动作是 loads 和 stores（以及它们的相关参数，包括处理器的 core ID、地址、以及在一次 store 中所要存储的值）。输出动作是响应 loads 的返回值。这些可观察的动作（stores、loads 和返回值）的序列，代表了 memory consistency model 的行为。

**Coherence protocols: Observable actions and behavior**

回想一下，处理器核心流水线与 coherence protocol 交互，以共同实现所需的 memory consistency model。因此，coherence protocol 的“用户”是与其交互的流水线，具体通过以下两个输入动作：(1) 程序中每个 load 的 read-request；(2) 程序中每个 store 的 write-request。Coherence protocol 的输出动作是：(1) read-return，为每个 read-request 返回一个值；(2) write-return，简单地对 write-request 进行确认 (acknowledge)。（流水线必须知道一个 write 何时完成。）这些可观察的动作的序列，代表了 coherence protocol 的外部的可观察行为。

重要的是，要注意 coherence protocol 与 consistency model 中，可观察的内容之间的区别。在 coherence protocol 中，一个 read-request 或 write-request 返回的瞬间是可观察的。然而，对于 consistency model，一个 load 或 store 返回的瞬间是不可观察的，只有 loads 返回的值是可观察的。

接下来，我们讨论指定系统行为的两种主要方法：*操作法 (operational method)* 和*公理法 (axiomatic method)*。在前者中，系统使用一个抽象参考实现 (abstract reference implementation) 来描述，而在后者中，数学公理 (mathematical axioms) 用于描述系统的行为。

### 11.1.1 Operational Specification

一份操作性的规范 (operational specification) 使用参考实现 (reference implementation) 来描述系统的行为，通常以状态机的形式表示。参考实现展示的行为，即输入/输出动作的序列，*定义*了系统所有合法行为的集合。操作模型 (operational models) 通常使用内部状态和内部动作来约束系统的行为，从而确保安全性 (safety)。操作模型的活跃性 (liveness) 取决于状态变化最终必须发生的事实。通常，活跃性是在外部（在状态机规范之外）表达的，它会使用以*时间逻辑 (temporal logic)* [31] 编写的数学公理。

>译者注：我们将 "temporal logic" 翻译为“时间逻辑”，以区别于通常被翻译为“时序逻辑”的 "sequential logic"。

**Specifying consistency models and coherence protocols operationally**

可以使用抽象实现 (abstract implementations) 来操作性地指定 consistency models。与它们的现实对应物一样，抽象实现通常同时具有处理器核心流水线组件 (processor core pipeline component) 和内存系统组件 (memory system component)。回想一下，内存系统组件与 coherence protocol 具有相同的接口（即，相同的可观察动作的集合，其中，可观察的动作是指：read-request、read-return、write-request、write-return），内存系统组件指定了 coherence protocol。

在下文中，我们将描述 sequential consistency (SC) 的两种操作性模型。两种模型具有相同的流水线组件，但它们的内存系统组件不同。前者指定了一个 consistency-agnostic coherence protocol，而后者指定了一个 consistency-directed coherence protocol。这将使我们能够操作性地描述两者之间的差异。

**SC Operational Spec1: In-order pipeline + atomic memory**

举个例子，一份操作性的 SC 规范 (operational SC specification)，类似于第 3.6 章 中描述的朴素的 SC 实现 (the switch)。除了可观察的动作（stores、loads 和返回值）之外，该模型还使用内部状态（内存）来约束 loads 可以读取的值。

操作性规范的工作方式如下。（假设每个核心都有一个程序计数器，指向下一条要取得的指令。）

* Step 1: *Fetch*. 其中一个处理器核心被非确定性地选中；该核心取得它的下一条指令，并将其插入局部指令队列。
* Step 2: *Issue from the pipeline*. 再一次地，一个核心被非确定性地选中；该核心从其指令队列中解码下一条指令。如果是内存指令，则流水线会发出 read-request (for a load) 或 write-request (for a store)，并阻塞。
* Step 3: *Atomic memory system*. 收到 read-request 后，内存系统会读取内存中的相应位置，并以返回值响应；收到 write-request 后，内存系统会将其写入内存，并以 ack 响应。
* Step 4: *Return to the pipeline*. 流水线在收到响应后解除阻塞。其中，store 会以 ack 响应，load 会以返回值响应。

本规范产生的行为，是遵循程序顺序的输入动作 (load/store at Step 1)、和输出动作 (value returned for a load at Step 4) 构成的序列。很容易看出它的行为满足 SC。例如，在表 11.1 的消息传递示例中，以下序列是可观察的：

<p align="center">«S1, S2, L1:r1=SET, L2:r2=NEW»</p>

同样地，L1 重复一次或多次，直到它看到 SET 的值，这种序列也是可能的。但是，以下 SC 违例是不可观察的：

<p align="center">«S1, S2, L1:r1=SET, L2:r2=0»</p>

![image](https://github.com/kaitoukito/A-Primer-on-Memory-Consistency-and-Cache-Coherence/assets/34410167/c34fdb5a-ba51-4d39-b5c7-a46c0a7ded6b)

**SC Operational Spec2: In-order pipeline + buffered memory**
