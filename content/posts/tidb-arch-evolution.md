---
title: "TiDB 的架构进化之道"
date: 2021-07-16T06:17:59Z
draft: false
---

> 整理自《高可用架构》的采访，原文：https://mp.weixin.qq.com/s/7nI_0jmr4Mo6zHzWCvTwpQ

**1. 简单介绍自己**

我叫张建，开源爱好者，PingCAP Engineering Manager，前阿里巴巴 ODPS 执行引擎研发工程师。在查询优化，分布式计算等方面有多年的研发经验。2017 年加入 PingCAP 从事 TiDB SQL 层的产品研发、架构改进、TiDB 社区建设等，目前在负责 TiKV 的产品演进。

**2. 聊聊你最近一年正在做的项目，它的技术价值怎样？它的行业发展状况是怎样？你负责项目的技术亮点和挑战能否展开讲讲？**

最近一年大量精力投入到了 TiDB SQL 层的产品研发中，一方面需要提升优化器的稳定性，另一方面需要增加和完善 TiDB SQL 功能。

我们都不希望在业务高峰的时候因为某个 SQL 的执行计划变了（通常是变得更差）导致用户服务受到影响，因此需要尽可能确保优化器能够稳定的生成高效的执行计划。有很多方法来提升优化器的稳定性，比如通过 SQL Plan Management 为高频 SQL 固定一个执行计划，或者提升优化器的基数估算以及代价模型的准确度。前者能确保执行计划不变，但是需要考虑 SQL 太多给系统带来的内存和资源消耗问题。后者能帮助优化器选到高效、消耗资源少的执行计划，但至今没有一个标准的解法。参考 Guy Lohman 2014 年的博客 [Is Query Optimization a “Solved” Problem?](https://wp.sigmod.org/?p=1075) – ACM SIGMOD Blog，SQL 层的优化依旧是数据库领域悬而未决的难题。

在今年发布的 TiDB 5.0 中 TiFlash 终于支持了 MPP 的分布式并行计算模型。这给 TiDB 优化器提出了更高的挑战：既要能同时产生 MPP 和 TiDB 的执行计划，还要能确保执行计划的稳定性。计算重的大查询要确保走到 MPP 模式，OLTP 的查询要能选到该选的索引。查询优化本质是个搜索算法，这个挑战相当于在扩大搜索空间后要求搜索出的执行计划尽可能高效和稳定。

此外还有 TiDB 执行引擎的资源控制尤其是内存 OOM 问题。为了避免因为数据库 OOM 导致用户服务受到影响，我们需要尽可能控制爆炸半径，限制内存占用的情况下完成 SQL 执行。我们逐渐统计各个 SQL 算子的内存使用，当发现内存使用过大时，通过取消当前 SQL 的执行来控制爆炸半径，或者进行中间结果落盘来确保当前 SQL 执行成功。这里面有很多技术上的挑战，比如内存统计准确度的问题，中间结果落盘后执行效率的问题，全局内存如何管理的问题等。

**3. 在技术方案落地的过程中，你通常关注哪些问题？如何保证技术方案顺利实施？**

在设计技术方案之前，需要明确要解决哪些用户、哪些应用场景的哪些问题。作为一个通用数据库，TiDB 面临着广泛的用户和应用场景，如果对当前重点优化的场景和要解决的问题没有选择，那么很可能我们辛辛苦苦开发了几个月后仍然没有一个用户对产品满意，仍然各种应用场景都有各种问题，这就太可怕了。有放弃才有选择和聚焦，才能对要解决的问题和收益达成一致，才能让产品飞速地迭代和发展。

除了技术方案，测试方案也非常重要。TiDB 已经拥有上千家用户，每个暴露出的缺陷都可能造成大范围的影响。为了提升产品的质量，每添加一个功能或改进一个模块都需要大量的测试确保结果不能错，性能不能回退，和其他组件或功能组合使用不会出现新问题，升级后不能 break 用户原有的应用程序等。我们会设计大量的功能测试、性能测试、系统测试、兼容性测试，覆盖到相关改动的方方面面。为了让方案落地的更快，这些测试用例最好提前设计，提前写好，面向测试用例，以终为始，驱动技术方案的开发和落地。

一个项目组的同学可能分布在全球各个地方，有时区、语言的天然困难。项目开发过程中的进度和风险管理也会遇到些挑战。这种情况下每天一个定时的信息同步会非常有帮助，可以是所有人在一起的视频会议，也可以各自总结到某文档上异步的讨论，能帮助我们尽早发现和解决开发过程中的问题，确保技术方案顺利落地。

**4. 架构师在最近的技术变化的浪潮中，需要面对的挑战都有哪些？如何应对这些挑战？**

常态化的挑战是如何在快速变化的技术环境中，选择对解决的问题，然后寻找和选择对应的技术方案。基础软件永远面对两个大开口，应用场景的多元化，解决方案的多元化，这两个开口的动态性带来了具体的技术挑战。

基础软件的应用场景特别多，产品在各个应用场景中会遇到着不一样的问题。技术方案的价值是解决用户问题，但比起具体的技术问题，我们更应该关注哪些场景中的问题要优先解决，哪些用户是这样的应用场景，他们分别遇到了什么问题。要回答这些问题需要我们往前走，从 oncall 问题中沉淀和总结，也去和具体的用户沟通了解业务架构，了解数据流转过程，未来的可能的发展变化等，在设计技术方案的时候，既要考虑解决眼下问题，也要尽可能考虑应对将来的问题。

随着技术的发展，可能会有越来越多更好的方案来解决同样的问题，为了保持架构的先进性，也为了避免一些可能的坑，保持开放的心态持续学习也很重要。阅读论文或博客，与人交流讨论都是非常不错的途径。

**5. 在做技术选型的过程中，你经常考虑的问题有些？**

在查询优化方面，会重点关注执行计划的稳定性和可预测性。执行计划的稳定性前文提过，这里不再展开。执行计划的可预测性指的是系统对用户来说是可预期的，比如 SQL 符合某个特征优化器就一定会选择用户希望使用的索引，MySQL 在这一点上做的就不错，比如 order by 的字段如果有索引就一定会用上。当系统的行为可预测时，系统提供者就说得清楚应用开发最佳实践是什么，喜欢上手实践的用户也能很快根据系统的行为建立自己的知识体系以优化应用程序。

数据库的可观测性也是非常重要的一个方面。我们需要有精简的指标来反映数据库是否运行良好，当系统的服务出现异常时，我们也要能快速定位到问题发生的原因。在 TiDB 产品发展过程中，我们逐渐为 SQL 执行过程中每个步骤都增加运行时信息的统计信息，完善了慢查询日志，通过 TiDB Dashboard 来帮助大家更直观更快的排查慢查询问题。还有 TiDB 的热力图，能够直观的显示出当前数据的读写热点，方便用户在不熟悉数据库原理的情况下快速定位热点问题。

**6. 云原生领域你看好哪个项目或技术，为什么？**

云原生数据库。

为了应对流量高峰，通常会按照峰值流量所需的硬件资源来部署数据库服务。但这样的高峰期并不是持续存在，在非高峰期时，多余的硬件资源仍然在支撑服务，但却并没有发挥出它们的价值，白白耗费了许多资源。云原生数据库因其存储计算分离，扩展性极强的特点，使得用户可以按需扩展数据库服务所需要的硬件资源，达到按需计费降低数据库服务所需要的费用开销的目的。而且随着越来越多应用上云以及技术的进步，云服务价格也会越来越便宜，云原生数据库的成本还将进一步降低。

此外云原生数据库在安全，故障恢复等方面也有极强的优势，这里不再展开。

**7. 请介绍下你这次在 GIAC 演讲的议题或者负责的专题内容**

这次主要为大家分享 TiDB 一路走到 5.0 的架构演进过程。会和大家讲讲我们是如何一步步构建 HTAP 数据库的，过程中遇到了哪些问题，如何解决的等。

**8. 对本次 GIAC 有什么寄语**

本次 GIAC 分会场众多，既可以专注于自己所处的技术领域，看看其他系统是如何解决类似问题的，也可以了解其他技术领域，关注新兴领域的技术趋势，以及一些新的应用问题，分别是如何解决的。希望大家在会议中充分交流，扩宽思路，互相提升。