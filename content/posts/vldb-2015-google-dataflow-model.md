---
title: "[VLDB 2015] The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing"
date: 2023-03-04T17:00:00Z
categories: ["Paper Reading"]
draft: true
---

## INTRODUCTION

INTRODUCTION 部分主要讲了现代数据处理的复杂性和激动人心的领域，包括 MapReduce 和其后继者（如 Hadoop、Pig、Hive、Spark）、SQL 社区内关于流处理的大量工作（如查询系统、窗口处理、数据流、时间域和语义模型）以及低延迟处理方面的最新尝试，例如 Spark Streaming、MillWheel 和 Storm。此外，该部分还提到现有模型和系统在一些常见用例中仍然存在缺陷。

这些缺陷包括：过度依赖数据完整性的假设、对实时性和语义的要求不足、缺乏灵活性等。论文给出了一些例子来说明这些缺陷，如 MapReduce 模型需要等待所有输入数据都被处理完毕才能输出结果，而在实际场景中，数据可能是无限流式输入的；Storm 系统虽然可以实现流式处理，但其对事件时间排序和非对齐窗口等功能支持不足。因此，为了解决这些问题并提高大规模数据处理的效率和可靠性，需要开发一种新的模型和系统。

文中提到一个流媒体视频提供商和广告/内容提供商的例子。视频提供商希望通过在其视频内容中插入广告来实现盈利，而广告/内容提供商则希望了解其广告/内容的观看情况，包括观看时间、观看长度、观看者身份以及与之配对的视频等信息。为了实现这些目标，需要对每个视频观看会话进行跟踪和记录，并在大规模数据集上进行处理。

他们需要越快越好地得到这些统计结果，因为这些统计结果对于他们的业务决策非常重要。例如，广告/内容提供商可能需要根据观看情况来调整其广告/内容的投放策略，以提高其效果和收益。而视频提供商则需要根据观看情况来计算广告观看量并向广告商收费，以实现盈利。如果这些统计结果不能及时得到，他们将无法做出及时的业务决策，从而可能会影响其业务效益和竞争力。因此，越快越好地得到这些统计结果对于他们来说非常重要。

除了需要越快越好地得到统计结果之外，他们还对数据处理系统有其他要求。例如，他们需要数据处理系统具有高效、可靠和可扩展的特性，能够处理大规模、高度无序的数据集，并且能够在面对故障和错误时保持稳定和正确。此外，他们还需要数据处理系统具有灵活性和可定制性，能够适应不同的业务需求和场景，并且能够与现有的技术栈进行集成。最重要的是，由于涉及到金钱交易等敏感信息，他们需要数据处理系统具有高度的安全性和隐私保护机制，以确保其业务数据不会被泄露或滥用。

论文提出的 Dataflow 能够：
1. Allows for the calculation of event-time5 ordered re- sults, windowed by features of the data themselves, over an unbounded, unordered data source, with correctness, latency, and cost tunable across a broad spectrum of combinations.
2. 将 pipeline 实现分解为四个相关维度，包括计算哪些结果、在事件时间中何时计算结果、在处理时间中何时物化结果以及如何将早期结果与后续改进相关联，从而提供清晰性、可组合性和灵活性。
3. Separates the logical notion of data processing from the underlying physical implementation, allowing the choice of batch, micro-batch, or streaming engine to become one of simply correctness, latency, and cost.

为了达到上述目标，Dataflow 模型包含下面几个部分：
1. A windowing model which supports unaligned event- time windows, and a simple API for their creation and use (Section 2.2).
2. A triggering model that binds the output times of results to runtime characteristics of the pipeline, with a powerful and flexible declarative API for describing desired triggering semantics (Section 2.3).
3. An incremental processing model that integrates retractions and updates into the windowing and triggering models described above (Section 2.3).
4. Scalable implementations of the above atop the MillWheel streaming engine and the FlumeJava batch engine, with an external reimplementation for Google Cloud Dataflow, including an open-source SDK that is runtime-agnostic (Section 3.1).
5. A set of core principles that guided the design of this model (Section 3.2).
6. Brief discussions of our real-world experiences with massive-scale, unbounded, out-of-order data processing at Google that motivated development of this model (Section 3.3).

### Unbounded/Bounded vs Streaming/Batch

这部分主要讲了在描述无限/有限数据集时，我们更倾向于使用无界/有界这些术语，而不是流式处理/批处理这些术语，因为后者会带有使用特定类型的执行引擎的暗示。实际上，自它们被提出以来，无界数据集就已经使用重复运行批处理系统进行处理，并且设计良好的流式处理系统完全能够处理有界数据。从模型的角度来看，流式处理或批处理的区别在很大程度上是不相关的，因此我们将这些术语专门用于描述运行时执行引擎。

### Windowing

这部分主要讲了窗口处理的概念和作用。当处理无界数据时，窗口处理是某些操作（例如聚合、外连接、时间限制操作等）所必需的，以便在大多数形式的分组中划定有限边界。对于有界数据，窗口处理基本上是可选的，但在许多情况下仍然是一个语义上有用的概念（例如向先前计算过的无界数据源的某些部分回填大规模更新）。窗口处理通常基于时间进行，虽然许多系统支持基于元组的窗口处理，但这实际上是在逻辑时间域上进行时间窗口处理，在其中按顺序排列的元素具有递增的逻辑时间戳。窗口可以是对齐或不对齐的，即可以应用于问题时间窗口中所有数据（对齐），也可以仅应用于特定子集的数据（例如不对齐）。

![Common Windowing Patterns](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303050031044.png)

当处理无界流时，有 3 种主要的窗口处理：
1. Fixed window：固定大小的窗口，窗口大小等于滑动周期。例如，每小时开始一个固定大小为一小时的窗口。固定窗口通常是对齐的，即每个窗口都适用于相应时间段内的所有数据。为了在时间上均匀地分配窗口完成负载，有时会通过为每个键相位移动窗口来使它们不对齐。也就是说，为了避免所有键同时完成窗口计算而导致负载过重，可以通过随机相位移动每个键的窗口来平衡计算负载。
2. Sliding window：滑动窗口，由窗口大小和滑动周期两个参数定义。例如，每分钟开始一个大小为一小时、滑动周期为一分钟的滑动窗口。滑动窗口通常是对齐的，即每个窗口的开始时间和结束时间都与其他窗口对齐。尽管在图示中画出了滑动的效果，但实际上所有五个窗口都会应用于图中的三个键（Key 1、Key 2 和 Key 3），而不仅仅是 Window 3。 此外，固定窗口实际上是滑动窗口的一种特殊情况，其中窗口大小等于滑动周期。也就是说，在固定大小为 1 小时的 fixed window 中，每个小时就会生成一个新的窗口，并且每个窗口都与前一个窗口对齐。
3. Session window：会话窗口，由超时间隔参数定义。任何在超时时间内发生的事件都将被视为同一个会话。例如，在用户停止活动一段时间后结束一个会话。会话窗口通常是不对齐的，即每个窗口的开始时间和结束时间都不一定与其他窗口对齐。例如，在图示中，Window 2 只适用于 Key 1，Window 3 只适用于 Key 2，而 Windows 1 和 4 只适用于 Key 3。 会话窗口通常用于捕获一段时间内的活动，并将这些活动分组为一个会话。例如，在 Web 应用程序中，可以使用会话窗口来捕获用户在一段时间内的所有操作，并将它们分组为一个会话。由于每个用户的操作时间和持续时间都可能不同，因此会话窗口通常是不对齐的。

### Time Domains

Time Domains 部分主要讲了在处理与时间相关的数据时需要考虑的两个时间域：事件时间和处理时间。事件时间是事件本身实际发生的时间，即发生事件时系统时钟（生成事件的任何系统）的记录。处理时间是在 Pipeline 内任何给定点观察到事件的时间，即根据系统时钟的当前时间。需要注意的是，在分布式系统中，我们不假设时钟同步。这些概念在文献中有所体现（特别是在时间管理和语义模型方面），但在第2.3节中提供了详细示例以更好地理解这些概念。

![Time Domain Skew](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303050047346.png)

在数据处理过程中，由于使用的系统存在通信延迟、调度算法、处理时间、管道串行化等因素，导致两个领域之间存在固有的和动态变化的偏差。为了解决这个问题，论文提出了一些技术和方法来减少时间偏差，并确保计算结果正确和可重复。例如，在水印（watermark）机制中引入了延迟参数（delay parameter），用于控制水印与实际时间之间的滞后程度。此外，在窗口操作中还引入了触发器（trigger）机制，用于在满足特定条件时触发窗口操作，并确保计算结果正确和完整。但是完整性的概念通常与正确性不兼容，因此我们不会仅仅依赖于水印（watermark）机制来确保计算结果的正确性。然而，水印机制可以提供一个有用的概念，即系统认为在事件时间上到达某个给定点之前所有数据已经被观察到。因此，水印机制不仅可以用于可视化时间偏差，还可以用于监控系统整体健康状况和进展情况，并作出一些不需要完全准确性的进展决策，例如基本垃圾收集策略。

在理想情况下，时间域偏差应该始终为零；我们应该始终在事件发生后立即处理所有事件。然而，现实情况并不总是如此理想，通常我们得到的结果更像图2所示。从12:00左右开始，由于管道滞后，水印开始偏离实时时间，并在12:02左右回到接近实时时间，然后在12:03左右再次明显地滞后。这种动态偏差在分布式数据处理系统中非常常见，并且将在定义提供正确、可重复结果所需的功能方面发挥重要作用。

## DATAFLOW MODEL

DATAFLOW MODEL 部分主要讲了数据流模型的设计原则、语义和实现。数据流模型是一种用于处理大规模数据集的编程模型，它将计算表示为一系列有向无环图（DAG）中的操作，其中每个节点表示一个操作，每个边表示数据元素的传输。该部分介绍了数据流模型的核心原则，包括可组合性、确定性和并行性，并提供了对其语义进行详细审查的示例。此外，该部分还介绍了 Google Cloud Dataflow 的实现，它基于 FlumeJava 和 MillWheel 技术构建而成。

### Core Primitives

Core Primitives 部分主要讲了数据流模型的 2 个核心原语。为了简化数据处理过程，我们将所有元素都视为 (key, value) 对的形式进行处理，即使某些操作（例如 ParDo）实际上并不需要键（key）。大多数有趣的讨论都围绕 GroupByKey 进行，而 GroupByKey 确实需要键（key），因此假设它们存在可以更简单地进行处理。

第 1 个原语是 ParDo。每个要处理的输入元素（它本身可能是一个有限集合）都会被提供给一个用户定义的函数（在 Dataflow 中称为 DoFn），该函数可以针对每个输入元素产生零个或多个输出元素。ParDo 操作对每个输入元素进行逐个处理，因此自然地适用于无限流数据。

例如，考虑一个操作，它将输入键的所有前缀扩展开来，并在它们之间复制相同的值：

```txt
(fix, 1), (fit, 2)
↓
ParDo(ExpandPrefixes)
↓
(f, 1), (fi, 1), (fix, 1), (f, 2), (fi, 2), (fit, 2)
```

在这个例子中，ParDo 操作会将输入数据 `(fix, 1)` 和 `(fit, 2)` 提供给 `ExpandPrefixes` 函数进行处理。该函数会将每个键的所有前缀扩展开来，并在它们之间复制相同的值。最终，输出结果为 `(f, 1)`, `(fi, 1)`, `(fix, 1)`, `(f, 2)`, `(fi, 2)`, 和 `(fit, 2)`。

第 2 个原语是 GroupByKey。GroupByKey 操作会将具有相同 key 的所有数据收集起来，然后将它们发送到下游进行聚合操作。如果输入源是无界的，我们无法知道它何时结束。解决这个问题的常见方法是对数据进行窗口化。一个 GroupByKey 的例子如下：

```txt
(f,1),(fi,1),(fix,1),(f,2),(fi,2),(fit,2)
↓
GroupByKey
↓
(f,[1,2]),(fi,[1,2]),(fix,[1]),(fit,[2])
```

### Windowing

Windowing 部分主要讲了数据流模型中的窗口处理。窗口处理是一种将无限数据集划分为有限边界的方法，以便在这些边界内执行聚合、排序和其他操作。该部分介绍了两种窗口处理模型：基于时间的窗口和基于元素数量的窗口。基于时间的窗口将数据元素划分为固定大小或滑动的时间段，而基于元素数量的窗口将数据元素划分为固定大小或滑动的元素组。此外，该部分还介绍了如何使用触发器来控制何时输出结果，并提供了示例代码来说明如何在 Dataflow 模型中实现窗口处理。

支持分组操作的系统通常会将其 GroupByKey 操作重新定义为 GroupByKeyAndWindow 操作。我们在这里的主要贡献是支持非对齐窗口（unaligned windows），其中有两个关键见解。第一个见解是，从模型的角度来看，将所有窗口化策略都视为非对齐窗口更简单，并允许底层实现在适用的情况下应用与对齐情况相关的优化。

这段话的意思是，窗口化可以分解为两个相关的操作：
* `Set<Window> AssignWindows(T datum)`：它将元素分配给一个或多个窗口。这本质上是《Semantics and Evaluation Techniques for Window Aggregates in Data Streams》中提出的 Bucket Operator。
* `Set<Window> MergeWindows(Set<Window> windows)`：它在分组时合并窗口。这允许数据驱动的窗口随着数据到达和分组而随时间构建。

对于任何给定的窗口化策略，这两个操作都是密切相关的；滑动窗口分配需要滑动窗口合并，会话窗口分配需要会话窗口合并等。为了原生支持事件时间窗口化，我们现在通过系统传递 (key, value, event time, window) 四元组，而不是传递 (key, value) 对。元素被提供给系统时带有事件时间戳，并且可以在 pipeline 的任何时刻进行修改。这些元素最初被分配到一个默认的全局窗口中，该窗口覆盖了所有的事件时间，并提供了与标准批处理模型中默认值相匹配的语义。

#### 窗口分配

![Window Assignment](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303050120988.png)
从模型的角度来看，窗口分配会在每个被分配到的窗口中创建元素的新副本。例如，考虑通过滑动两分钟宽度和一分钟周期的窗口对数据集进行窗口化，如上图所示（为了简洁起见，时间戳以HH:MM格式给出）。在窗口化操作中，每个 (key, value) 对都会在与其时间戳重叠的两个窗口中创建副本。由于窗口直接与其所属的元素相关联，因此这意味着窗口分配可以在应用分组之前的 pipeline 中的任何位置发生。这一点非常重要，因为分组操作可能会被嵌套在复合转换（例如 Sum.integersPerKey()）的下游某个位置。

#### 窗口合并

窗口合并是 GroupByKeyAndWindow 操作的一部分，并且最好通过示例来解释。我们将使用会话窗口化作为示例，因为这是我们的主要应用场景。图4展示了四个示例数据，其中三个属于 k1，一个属于 k2，并按照会话窗口化进行了分组，每个会话窗口的超时时间为30分钟。所有元素最初都被系统放置在一个默认的全局窗口中。AssignWindows 的会话实现将每个元素放入一个单独的窗口中，该窗口延伸到其自身时间戳之后的30分钟；如果后续事件要被视为同一会话的一部分，则这个窗口表示它们可能出现的时间范围。然后，我们开始 GroupByKeyAndWindow 操作，这实际上是一个由五个部分组成的复合操作。

![Window Merging](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303050129866.png)

-   DropTimestamps：删除元素的时间戳，因为从这里开始只有窗口是相关的。
-   GroupByKey：将 (value, window) 元组按照键值进行分组。
-   MergeWindows：合并一个键的当前窗口集。具体的合并逻辑由窗口策略定义。例如，如果使用会话窗口化策略，那么当两个或多个窗口重叠时，它们就会被合并为一个更大的窗口。上图展示了这种情况，其中 v1 和 v4 的窗口重叠，并且被合并为一个新的、更大的会话窗口。
-   GroupAlsoByWindow：对于每个键，按照窗口分组值。在上一步合并之后，v1 和 v4 现在在相同的窗口中，因此在这一步被分组在一起。
-   ExpandToElements：将每个键和窗口组中的值扩展为 (key, value, event time, window) 元组，并使用新的窗口时间戳。时间戳可以设置为任何大于或等于窗口中最早事件的值，以保证 watermark 的正确性。在这个例子中，我们将时间戳设置为窗口结束时的时间。

### Triggers & Incremental Processing

Triggers & Incremental Processing 部分主要讲了如何使用触发器来控制何时输出结果，并介绍了增量处理的概念。触发器是一种机制，用于确定何时将窗口中的结果输出到下游操作。该部分介绍了三种类型的触发器：早期触发器、延迟触发器和累积触发器，并提供了示例代码来说明如何在 Dataflow 模型中实现这些触发器。此外，该部分还介绍了增量处理的概念，它允许在窗口处理过程中对数据进行更新和撤销操作。增量处理可以通过将更新和撤销操作与窗口处理模型集成来实现，从而提高计算效率并减少资源消耗。

我们需要一种方法来确定何时发出窗口的结果。由于数据在事件时间上是无序的，因此我们需要其他信号来告诉我们何时窗口完成。一个初步的解决方案可能是使用一种全局的事件时间进度指标，比如 watermark。但是，watermark 本身在正确性方面有两个主要的缺点：
1. 它们有时候太快了，意味着可能有一些迟到的数据落在 watermark 后面。对于许多分布式数据源来说，要得到一个完全准确的事件时间 watermark 是不可行的，因此如果我们想要输出数据100%正确，就不能只依赖 watermark。
2. 它们有时候太慢了。因为它们是一个全局的进度指标，watermark 可能会被整个 pipeline 中的一个单个慢数据所阻塞。而且即使对于事件时间偏差很小的健康 pipeline 来说，基线水平的偏差可能仍然有几分钟或更多，这取决于输入源。因此，使用 watermark 作为唯一的信号来发出窗口结果可能会导致总体结果的延迟比如说一个可比较的 Lambda 架构 pipeline 更高。

一个解决方案是参考 Lambda 架构，它不仅提供了流式处理管道所能提供的最佳低延迟结果估计，而且还承诺在批处理管道运行后实现最终一致性和正确性。这种方法可以有效地避免数据完整性问题，并提高计算效率和准确性。

如果我们想要在单个管道中执行相同的操作（无论执行引擎如何），那么我们需要一种方法为任何给定的窗口提供多个答案（或窗格）。我们称之为触发器，因为它们允许指定何时触发给定窗口的输出结果。

Triggering 是一种用于在内部或外部信号的作用下刺激 GroupByKeyAndWindow 结果的生成的机制，和 Windowing 相辅相成：Windowing 是将数据按照时间切片进行分组处理的过程，Triggering 则是指定何时触发给定窗口的输出结果。

除了控制何时发出结果之外，触发器系统还可以通过三种不同的细化模式控制同一窗口的多个窗格之间的关系：
1. Discarding 模式：在触发时，窗口内容被丢弃，并且后续结果与先前结果无关。Discarding 在窗口结束时丢弃之前的输出结果，只保留最新的结果。这种模式适用于那些只关心最新数据而不在乎历史数据的场景。例如，如果我们想要每分钟统计一次某个指标的值，我们可以使用 Discarding 模式
1. Accumulating 模式：在触发时，窗口内容保留，并且后续结果与先前结果相关。这种模式适用于需要对窗口内容进行累加或聚合操作（例如计算平均值或求和）的情况。Accumulating 在窗口结束时保留之前的输出结果，并将新的结果与之前的结果合并。这种模式适用于那些关心数据的累积变化的场景。例如，如果我们想要每分钟统计一次某个指标的全局总和，我们可以使用 Accumulating 模式，这实际上相当于给了事件时间窗口语义，输出窗格会重叠，因为它们的结果包含了事件时间上相同或部分相同的数据区域。
2. Accumulating & Retracting 模式：和 Accumulating 模式类似也会保留之前的输出结果，在窗口结束时将新的结果与之前的结果合并。相比 Accumulating 模式多了一个 Retracting 操作：当窗口再次触发时，首先会发出一个对之前值的撤回（retraction），然后再发出新值作为一个普通数据。这种模式适用于那些关心数据的准确性而不在乎数据的重复性或延迟性的场景。例如，如果我们想要每分钟统计一次某个指标的全局总和，并且能够处理乱序数据和延迟数据。

### Examples

前面一堆理论介绍，对于第一次接触这些理论的朋友可能看完后还是很陌生，论文的 Examples 部分占据了 4 页的篇幅。我们一起来看看这些例子，希望能够帮助读者朋友们更好的建立起脑海中对 Dataflow 模型的认知体系。

```txt
PCollection<KV<String, Integer>> output = input
  .apply(Sum.integersPerKey());
```

假设我们有一个整数输入源，它会前后生成 10 个整数，这些整数 Value 都属于同一个 Key。处理时间和生成时间可能不同，下图的横坐标表示了这些数字的生成时间，纵坐标表示了这些数字的处理时间，图里面数字 9 比较有意思，它是一个典型的迟到的数据：

![Example Inputs](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303051103131.png)

#### Classic Batch Execution

在经典的批处理系统中，我们需要等到所有数据都拿到后才开始计算，所以下图中的 actual watermark 直到 12:09 接受到最后一个整数 1 以后才能确认，计算也才能开始，它的处理时间包含了前面的等待时间和最后计算所有整数和的时间：

![Classic Batch Execution](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303051112863.png)

#### GlobalWindows, AtPeriod, Accumulating

```txt
PCollection<KV<String, Integer>> output = input
  .apply(Window.trigger(Repeat(AtPerid(1, MINUTE)))
			   .accumulating())
  .apply(Sum.integersPerKey());
```

也可以把这个整数生成器当做是个无界数据流，通过 stream processing 的方式来处理它。我们通过 `Window.trigger()` 操作在这个全局 Window 上生成一个 1 分钟触发一次的 Trigger，这个 Trigger 使用 Accumulating 模式周期性的更新当前求得的整数总和，这样我们就能每分钟更新一次所有整数的和。下图中总共持续了 4 分钟，前 3 分钟得到的总和分别是 11、22 和 33，最后第 4 分钟我们看到了和上面 batch 模式一样的结果 51：

![GlobalWindows, AtPeriod, Accumulating](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303051125661.png)

#### GlobalWindows, AtPeriod, Discarding

```txt
PCollection<KV<String, Integer>> output = input
  .apply(Window.trigger(Repeat(AtPerid(1, MINUTE)))
			   .discarding())
  .apply(Sum.integersPerKey());
```

那如果我们只希望看到每分钟新增数据的总和呢？我们可以把这个固定 Trigger 的模式调整为 Discarding，这样就会忽略之前的计算总和，每分钟求得的总和独立输出：

![GlobalWindows, AtPeriod, Discarding](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303052342145.png)

#### GlobalWindows, AtCount, Discarding

```txt
PCollection<KV<String, Integer>> output = input
  .apply(Window.trigger(Repeat(AtPerid(1, MINUTE)))
			   .discarding())
  .apply(Sum.integersPerKey());
```

我们还可以使用 tuple-based 的 Trigger 方式，比如这个例子里我们使用 `AtCount(2)` 重新定义了这个 Trigger，使其在当前窗口中处理完 2 条数据后就对外输出结果，这个 Trigger 采用 discarding 的模式忽略之前的计算结果，这样我们就得到了每 2 条数据的中间结果：

![GlobalWindows, AtCount, Discarding](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303052349523.png)

#### FixedWindows, Batch

除了使用全局 Windowing 以外，我们还可以使用 Fixed Window 策略。比如这个例子中使用了一个 2 分钟长度的 Fixed Window，这个 Window 采用 accumulating 策略，每次都在上次结果的基础上更新求得的总和：

```txt
PCollection<KV<String, Integer>> output = input
  .apply(Window.into(FixedWindows.of(2, MINUTES)
               .accumulating())
  .apply(Sum.integersPerKey());
```

默认的 Trigger 的行为等价如下：

```txt
PCollection<KV<String, Integer>> output = input
  .apply(Window.into(FixedWindows.of(2, MINUTES)
               .trigger(Repeat(AtWatermark())))
               .accumulating())
  .apply(Sum.integersPerKey());
```

上面的 Window 和 Trigger 配置告诉我们，在每 2 分钟 Window 结束的时候得到一个 watermark，触发器会在 watermark 通过窗口的末尾时被触发。批处理和流处理引擎都实现了 watermark 触发器，Trigger 定义中的 `Repeat` 指的是如果在 watermark 之后仍有数据到达，则会实例化并立即触发对应的 watermark 触发器。

在 Dataflow 的当前实现中，数据源必须是有界的。因此，类似于最开始经典的批处理示例，我们需要等待所有数据都到达后才开始计算，计算时按照事件时间顺序处理数据，模拟 watermark 推进，到达 Window 边界时触发 watermark trigger，得到当前 Window 的结果。如下图所示，当所有数据都接收到以后才会开始计算第 1 个 window：

![FixedWindows, Batch](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303060001019.png)

#### FixedWindows, Micro-Batch

上面例子中，只有在所有数据都拿到后才能开始计算并得到每个窗口最终的精确结果。如果我们利用每 1 分钟执行一次的 micro-batch execution engine 来执行的话，每个窗口的 watermark 在 micro-batch 触发执行时都会被重新计算，如果有更新则会去更新当前窗口的计算结果。

还是按照 event-time 顺序来处理数据，和上图一样从左到右划分成了 4 个窗口，每个窗口从下到上执行了 4 次，对于第 1 个窗口来说，第 1 个 1 分钟执行时拿到了 5 这个数据，最后一个 1 分钟 micro-batch 执行时拿到了 9 这个数据，整个窗口结果会被更新 4 次，分别是 5、5、5、14。

当所有的输入数据都处理完成时，每个窗口的最终结果和上面 batch 模式的结果一样，从这个角度来说 micro-batch 模式保证了 eventual correctness，并且每分钟更新一次结果也能保证一定的实时性：

![FixedWindows, Micro-Batch](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303060024243.png)

#### FixedWindows, Streaming

和上面 micro-batch 不同，streaming execution engine 是每遇到一个 watermark 内的数据就触发一次计算，更新当前窗口的结果并输出。不像 micro-batch 那样是固定时间触发更新并输出结果，在这一点上实时性会好很多。streaming execution engine 同样也处理迟到的数据，还是第一个窗口，9 这个数字是在很靠后的时间才接收到，和 micro-batch 一样也满足 eventual correctness：

![FixedWindows, Streaming](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303060039098.png)

#### FixedWindows, Streaming, Partial

![FixedWindows, Streaming, Partial](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303060059262.png)

#### Sessions, Retracting

![Sessions, Retracting](https://raw.githubusercontent.com/zz-jason/blog-images/master/images/202303060057838.png)

## IMPLEMENTATION & DESIGN

IMPLEMENTATION & DESIGN 部分主要讲了如何在实现和设计 Dataflow 模型时考虑到可扩展性、容错性和性能等方面的问题。该部分介绍了 Dataflow 模型的三个实现：FlumeJava、MillWheel 和 Google Cloud Dataflow，并提供了有关这些实现的详细信息。此外，该部分还介绍了如何使用 Dataflow 模型进行流处理和批处理，并提供了一些最佳实践和技巧，以帮助读者更好地使用 Dataflow 模型进行大规模数据处理。

### Implementation

### Design Principles

Design Principles 部分主要讲了 Dataflow 模型的设计原则。这些原则包括可组合性、确定性、并行性和容错性等方面，旨在确保 Dataflow 模型具有高效、可靠和可扩展的特性。该部分介绍了每个原则的含义和重要性，并提供了示例代码和最佳实践，以帮助读者更好地理解这些原则并将其应用于实际数据处理场景中。此外，该部分还介绍了如何使用 Dataflow 模型进行流处理和批处理，并提供了一些有用的提示和技巧，以帮助读者更好地使用 Dataflow 模型进行大规模数据处理

### Motivating Experiences

Motivating Experiences 部分主要讲了设计 Dataflow 模型的动机和背景。该部分介绍了 Google 在处理大规模数据时遇到的挑战和问题，并提供了 FlumeJava 和 MillWheel 等先前技术的概述。此外，该部分还介绍了如何使用这些技术来解决实际问题，并提供了一些示例代码和最佳实践，以帮助读者更好地理解这些技术并将其应用于实际数据处理场景中。最后，该部分还介绍了如何使用 Dataflow 模型进行流处理和批处理，并提供了一些有用的提示和技巧，以帮助读者更好地使用 Dataflow 模型进行大规模数据处理。

## CONCLUSIONS

CONCLUSIONS 部分主要总结了 Dataflow 模型的设计和实现，并提出了未来数据处理的发展方向。该部分指出，Dataflow 模型具有高效、可靠和可扩展的特性，可以用于大规模数据处理场景中。同时，该部分还提出了未来数据处理的趋势，即无界数据将成为数据处理的主流，并需要更加强大和灵活的工具来满足消费者对事件时间排序和非对齐窗口等功能的需求。最后，该部分强调了需要改变整体思维方式以适应未来数据处理工具的发展，并鼓励研究人员和开发者继续探索和创新。