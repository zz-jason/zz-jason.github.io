---
title: "DuckDB | Push-Based Execution Model"
date: 2022-11-13T08:36:00+08:00
categories: ["Technology"]
draft: true
---
## 简介

DuckDB 是我非常喜欢的一个数据库，它架构简单、分析性能优秀、代码干净、编译迅速、极易上手，另外 logo 也挺好看的。DuckDB 基于 libpg_query 实现 SQL Parser 在语法上和 PostgreSQL 保持一致，内嵌 sqlite 的 REPL CLI，在编译好 duckdb 的 executable 后，直接执行即可交互式的输入 SQL 得到执行结果，对于想要玩玩试试的人来说从 clone 代码到编译完成再到运行 SQL 得到结果的过程非常顺畅和简单。

10 月初偶然间翻看 duckdb 的代码，发现他的执行引擎和计算调度采用了类似 Hyper 的 Morsel-Driven（参考《Morsel-Driven Parallelism: A NUMA-Aware Query Evaluation Framework for the Many-Core Age》）的方式，实现了 push-based execution model，它的 pipeline breaker 语义和也 Hyper 在 《Efficiently Compiling Efficient Query Plans for Modern Hardware》 中定义的一致，不同的是 Hyper 期望通过 LLVM JIT 的方式对数据一行一行计算使其尽量保存在寄存器中，DuckDB 采用向量化使一批数据尽可能保存在 CPU Cache 中。

Hyper 和 Vectorize 的论文之前在做 TiDB 的时候研究过很多篇，也在内部做过几次分享，一直希望实现一个简单的 demo 验证下效果可一直没精力做，正好 DuckDB 采用了类似实现，这个勾起了我浓烈的好奇心。因此利用了周末时间研究了一下 DuckDB 是如何实现 push-based execution model 的。

这篇文章主要和大家分享我所看到和理解的 DuckDB 的 push-based execution model。我不是 DuckDB 专家，对 DuckDB 的代码不熟，源码分析是个逆向的过程，需要不断看代码作者是怎么写的代码，推测作者背后的思考和设计原则。文中可能会有错误之处，希望大家帮忙指出，感谢。

## 执行框架概览

DuckDB 启动时会创建一个全局的 TaskScheduler，启动 nproc-1（main 函数）个后台线程。这些后台线程启动后会不停地尝试从位于 TaskScheduler 的一个公共 Task 队列中取出一个 Task 并执行。DuckDB 中 Query 的执行就是靠这个后台线程池完成的。

要在这个框架下完成 Query 执行，DuckDB 基于 Query 的物理执行计划 PhysicalOperator DAG 构造了 Pipeline DAG。每个 Pipeline 代表了物理执行计划中一段连续的 PhysicalOperator 算子，由 source、operators 和 sink 构成。当且仅当 Pipeline 的所有 dependency 都执行完后，该 Pipeline 才可被执行。Pipeline 的 sink 代表了需要消费掉所有输入数据才能对外返回结果的 PhysicalOperator。Pipeline DAG 和 PhysicalOperator DAG 一样，可以看做是另一种视角下的物理执行计划。

每个 Pipeline 可以都同时被多个线程并发执行，Pipeline 的 source 和 sink 需要是并发安全的。每个 Pipeline 的执行都由对应的 PipelineExecutor 完成。为了让 PipelineExecutor 能够被线程池中后台线程执行，每个 PipelineExecutor 都被封装在了一个 ExecutorTask 中。这样后台线程只要执行完 TaskScheduler 队列中的一个 ExecutorTask 即可完成某个 Pipeline 其中一个并发任务。

为了正确调度和执行 Pipeline DAG 中所有的 Pipeline，并且考虑到每个 Pipeline 都可能有多个并发的 ExecutorTask，我们需要能够及时知道某个 Pipeline 的所有并发是否都执行完毕，并且在其父亲 Pipeline 的所有依赖都被执行完后及时调度父亲 Pipeline 的所有 ExecutorTask，DuckDB 采用 Event 来生成和调度对应的 ExecutorTask。

每个 Pipeline 的 Event 都记录了需要的总并发数和完成的并发数。在构造 Pipeline DAG 后，DuckDB 会为其构造一个对应的 Event DAG，Pipeline 的调度和执行就是通过 Event 来完成的。每当一个 ExecutorTask 完成，该 Event 的完成并发数就会加 1，当该 Pipeline 的所有 ExecutorTask 都完成后，Event 中的总并发数和已完成并发数相等，标志着该 Event 也完成，该 Event 会通知其父亲 Event，父亲 Event 一旦检测到所有 dependency Event 都执行完，就会调度自己的 ExecutorTask，从而驱动后续的 Pipeilne 计算。

ExecutorTask 中的 Pipeline 是以 push 的方式执行的：先从 Pipeline 的 source 获取一批数据，然后将该批数据依次的通过所有中间的 operators 计算，最终由 sink 完成这一批初始数据的最终计算。典型的 sink 就是构造 hash table 了：当前 Pipeline 的所有 ExecutorTask 执行完后，最终的 hash table 也构造好了。

为了返回结果给客户端，当前 Query 的主线程会不断调用 root PipelineExecutor 的 pull 接口。需要注意的是，这个接口名字的 pull 指的仅仅是从最顶层 Pipeline 拿结果数据，在计算顶层 Pipeline 的时候仍然是从 source 到最后一个计算 PhysicalOperator push 计算过去的。root PipelineExecutor 拿到一批 source 数据代表着 root Pipeline 依赖的所有 PipelineTask 都执行完毕，之后 root PipelineExecutor 内部以 push 的方式执行完这一批数据得到结果，将结果返回给客户端，用户就可以看到 Query 执行结果了。

以上就是 DuckDB 执行框架的大致介绍。实际实现时因为要特殊考虑一些算子的优化方案，所以实现起来会稍微复杂一些。比如考虑到 UNION ALL，DuckDB 会在一段 PhysicalOperator 链条上构造多个 Pipeline。考虑到 partitioned hash join 的高效实现，DuckDB 也会在一段 PhysicalOperator 链条上构造多个 Pipeline，和 UNION ALL 不同的是，这些 Pipeline 之间还有执行顺序的依赖关系。最终构造出来的可能就是有多个 root 的 Pipeline DAG

## 第 1 部分：TaskScheduler 和后台线程池


在 DuckDB 启动时会创建一个全局的 TaskScheduler，在后台启动 nproc-1（main 函数）个后台线程，启动线程是在 TaskScheduler::SetThreadsInternal() 函数中进行的，从主线程启动线程池的调用堆栈如下，感兴趣的读者可以根据这些关键函数看看线程是如何启动起来的：
```
(lldb) bt
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
  * frame #0: 0x00000001074e29ee duckdb`duckdb::TaskScheduler::SetThreadsInternal(this=0x00006060000015e0, n=12) at task_scheduler.cpp:239:4
    frame #1: 0x00000001074e5335 duckdb`duckdb::TaskScheduler::SetThreads(this=0x00006060000015e0, n=12) at task_scheduler.cpp:199:2
    frame #2: 0x0000000107099f45 duckdb`duckdb::DatabaseInstance::Initialize(this=0x0000616000000698, database_path=0x0000000000000000, user_config=0x00007ff7bfefd420) at database.cpp:159:13
    frame #3: 0x000000010709f2ce duckdb`duckdb::DuckDB::DuckDB(this=0x0000602000000a90, path=0x0000000000000000, new_config=0x00007ff7bfefd420) at database.cpp:170:12
    frame #4: 0x000000010709f689 duckdb`duckdb::DuckDB::DuckDB(this=0x0000602000000a90, path=0x0000000000000000, new_config=0x00007ff7bfefd420) at database.cpp:169:100
    ...
```
后台线程启动后的主逻辑在 TaskScheduler::ExecuteForever() 中，在每个后台线程的生命周期内，它们会不停尝试从位于 TaskScheduler 的公共队列中取出一个 Task，调用 Task::Execute() 函数完成 Task 的执行：
```cpp
void TaskScheduler::ExecuteForever(atomic<bool> *marker) {
#ifndef DUCKDB_NO_THREADS
    unique_ptr<Task> task;
    // loop until the marker is set to false
    while (*marker) {
        // wait for a signal with a timeout
        queue->semaphore.wait();
        if (queue->q.try_dequeue(task)) {
            task->Execute(TaskExecutionMode::PROCESS_ALL);
            task.reset();
        }
    }
#else
    throw NotImplementedException("DuckDB was compiled without threads! Background thread loop is not allowed.");
#endif
}
```
Pipeline 的各个 Task 就是这样被后台线程并发执行的。要想控制 Pipeline 之间的执行顺序和 Pipeline 内的并发度，只需要设计和控制好各个 Task 入队的顺序即可完成 Pipeline 计算的调度。Pipeline 的执行主要依靠 ExecutorTask，各个算子如果需要自定义计算逻辑和调度规则也是通过实现新的 ExecutorTask 完成。

## 第 2 部分：ExecutorTask 和 Event 驱动的调度模型

![Event based scheduling model](/images/duckdb-push-based-execution-model/Task-Event.jpg)

在 Task 的执行框架内，后台线程会通过 ExecutorTask::Execute() 驱动当前 ExecutorTask 的执行。为了给各个 Pipeline 和 PhysicalOperator 提供灵活的执行方式，DuckDB 内各个 PhysicalOperator 可以各自实现特定的 ExecutorTask 用于完成自身特殊的执行和后续 Pipeline Task 的计算调度。ExecutorTask::Execute() 的执行会直接调用子类的 ExecutorTask::ExecuteTask() 函数完成当前 ExecutorTask 的实际执行。

对于一般的 Pipeline 来说，会直接构造一个叫 PipelineTask 的 ExecutorTask 子类。PipelineTask::ExecuteTask() 的大致逻辑如下：

```cpp
TaskExecutionResult ExecuteTask(TaskExecutionMode mode) override {
    if (!pipeline_executor) {
        pipeline_executor = make_unique<PipelineExecutor>(pipeline.GetClientContext(), pipeline);
    }
    if (mode == TaskExecutionMode::PROCESS_PARTIAL) {
        bool finished = pipeline_executor->Execute(PARTIAL_CHUNK_COUNT);
        if (!finished) {
            return TaskExecutionResult::TASK_NOT_FINISHED;
        }
    } else {
        pipeline_executor->Execute();
    }
    event->FinishTask();
    pipeline_executor.reset();
    return TaskExecutionResult::TASK_FINISHED;
}
```

通过 PipelineExecutor::Execute() 完成当前 ExecutorTask 的执行后，它会去调用 Event::FinishTask() 函数来标记当前已经执行完成，在 Event::FinishTask() 函数如果发现当前 Event 的所有 Task 都执行完毕就会清理当前 Event 相关内容，并调用父亲 Event 的 CompleteDependency()：

```cpp
void Event::Finish() {
    D_ASSERT(!finished);
    FinishEvent();
    finished = true;
    // finished processing the pipeline, now we can schedule pipelines that depend on this pipeline
    for (auto &parent_entry : parents) {
        auto parent = parent_entry.lock();
        if (!parent) { // LCOV_EXCL_START
            continue;
        } // LCOV_EXCL_STOP
        // mark a dependency as completed for each of the parents
        parent->CompleteDependency();
    }
    FinalizeFinish();
}
```

在 Event::CompleteDependency() 中，如果发现所有 dependency Event 都已经执行完毕，则会开始调度执行父亲 Event 的 Task。如果父亲 Event 没有 task 需要执行，则会直接再调用父亲 Event 的 Finish() 函数在当前线程中完成父亲 Event 的执行：

```cpp
void Event::CompleteDependency() {
    idx_t current_finished = ++finished_dependencies;
    D_ASSERT(current_finished <= total_dependencies);
    if (current_finished == total_dependencies) {
        // all dependencies have been completed: schedule the event
        D_ASSERT(total_tasks == 0);
        Schedule();
        if (total_tasks == 0) {
            Finish();
        }
    }
}
```

Event 的 Task 调度是通过 Event::Schedule() 函数完成的，这是个 Event 的纯虚函数，不同的子类 Event 需要自行实现。Pipeline 执行过程中使用的 Event 类型不多，最常见的是：

* PipelineInitializeEvent：主要用来初始化当前 Pipeline 的 sink，会调度 1 个 PipelineInitializeTask
* PipelineEvent：主要用来表示 Pipeline 的执行操作，可能会调度多个 ExecutorTask 到执行队列中。PipelineEvent 的 Schedule() 函数主要调用 Pipeline::Schedule() 完成 ExecutorTask 的计算调度，这里不再展开，感兴趣的读者可以继续追踪代码看看其中的实现细节
* PipelineFinishEvent：主要用来标记当前 Pipeline 执行结束，在 Event::Finish() 检测到当前 Event 结束，调用到 PipelineFinishEvent::FinishEvent() 时完成 Pipeline::Finalize()，用来做 Pipeline 的清理操作
* PipelineCompleteEvent：用来更新 Executor 中已结束的 Pipeline 的 counter completed_pipelines，Executor 主线程会不断检测 completed_pipelines，当发现所有中间 Pipeline 都执行完后，主线程会开始执行 root Pipeline，返回结果给客户端。

## 第 3 部分：PipelineExecutor 和 Pipeline 内基于 Push 的执行模型

PipelineExecutor::Execute() 函数通过调用 Pipeline 中各个 PhysicalOperator 的相应接口，以 Push 的方式完成了当前 Pipeline 的执行，执行逻辑可以概括为：

* 先调用 FetchFromSource() 从 Pipeline 的 source PhysicalOperator 中获取计算结果作为 source DataChunk，这里会调用 source 的 GetData() 接口。
* 再调用 ExecutePushInternal() 依次执行 Pipeline 中 operators 列表中的各个 PhysicalOperator 和最后一个 sink PhysicalOperator 完成这批数据后续的所有计算操作。对于普通 operator 会调用它的 Execute() 接口，对最后的 sink 会调用它的 Sink() 接口。
* 最后调用 PushFinalize() 完成当前 ExecutorTask 的执行，这里会调用 sink 的 Combine 接口，用以完成一个 ExecutorTask 结束后的收尾清理工作。

```cpp
bool PipelineExecutor::Execute(idx_t max_chunks) {
    D_ASSERT(pipeline.sink);
    bool exhausted_source = false;
    auto &source_chunk = pipeline.operators.empty() ? final_chunk : *intermediate_chunks[0];
    for (idx_t i = 0; i < max_chunks; i++) {
        if (IsFinished()) {
            break;
        }
        source_chunk.Reset();
        FetchFromSource(source_chunk);
        if (source_chunk.size() == 0) {
            exhausted_source = true;
            break;
        }
        auto result = ExecutePushInternal(source_chunk);
        if (result == OperatorResultType::FINISHED) {
            D_ASSERT(IsFinished());
            break;
        }
    }
    if (!exhausted_source && !IsFinished()) {
        return false;
    }
    PushFinalize();
    return true;
}
```

PhysicalOperator 同时包含了 source、operator、sink 所需要的所有接口，需要根据具体 PhysicalOperator 的需要实现对应的接口。比如 partitioned hash join 因为会分成 3 个阶段分别作为 sink、operator 和 source 角色，它同时实现了所有的接口。

## 第 4 部分：主线程和 root Pipeline 的执行



## 第 5 部分：Pipeline 的构造



## 最后，DuckDB 执行模式引发的一些思考



## 参考材料



细节 1：从物理执行计划构造 Pipeline

我们知道 SQL 需要先被 Parse 成 AST，再经过 rewrite、optimize 生成出物理执行计划后才被执行。这篇文章主要分析 DuckDB 的执行框架，我们跳过 parser 和优化器部分，直接看物理执行计划是如何被执行的。
物理执行计划在 DuckDB 中被表示成一个 PhysicalOperator 树。和大多数数据库一样，每个算子都有自己的孩子节点表示算子的输入源和计算的依赖关系。
Pipeline 是 DuckDB 中计算调度和并发执行的最小单元，主要由三部分构成：
source：PhysicalOperator，表示该 Pipeline 的数据源
opetators：vector<PhysicalOperator>，表示该 Pipeline 从 source 读取到数据后需要进行的计算
sink: PhysicalOperator，表示该 Pipeline 最后一个算子，该算子也是下游 Pipeline 的 source
从物理执行计划划分 Pipeline 第一个遇到的问题是如何确定 Pipeline 的 sink 和 source。DuckDB 采用了和 Hyper 一样的 Pipeline breaker 定义，哪些需要消化掉所有孩子节点的数据后才能进行下一步计算输出结果的算子。典型的比如构造 hash join 或 hash aggregate 的 hash table，需要完全消费掉孩子节点的数据将其构造成 hash table 后，才能 probe hash table 的数据输出结果给父亲节点。类似的还有 sort，topN 这样的算子都是 Pipeline breaker。算子的实现原理将深刻影响 Pipeline 的构造，我们在后面 Union All 和 Hash Join 的 Pipeline 构造中将会深刻的感受到这一点。
物理执行计划转成 Pipeline 是由当前查询的 Executor 完成的，几个关键函数：
Executor::InitializeInternal()：把物理执行计划（PhysicalOperator tree）转成 Pipeline 的入口，所有构造出来的 Pipeline 都存储在该查询的 Executor 中。
PhysicalOperator::BuildPipelines()：构造 Pipeline 的是通过 top down 的遍历 PhysicalOperator tree 完成的，Pipeline 的 sink 会先被确定下来（要么是整个物理执行计划的根节点，要么是上一个 Pipeline 的 source 节点）。Executor 通过该函数遍历每个 PhysicalOperator，决定将其加入当前 Pipeline 的 operators 列表还是做为当前 Pipeline 的 source。遇到当前 Pipeline 的 source 时就需要结束构造当前 Pipeline 了，然后将该 source 作为下一个 Pipeline 的 sink，继续 top down 的遍历 PhysicalOperator tree 和构造新的 Pipeline。
PhysicalOperator::BuildChildPipeline()：切分 Pipeline，构造 Pipeline 之间的依赖关系。
Executor::InitializeInternal() 函数的代码如下。它是将物理执行计划构造成 Pipeline 并完成 Pipeline 计算调度的总指挥。可以看到，在这里 root Pipeline 的 sink 设置为 null 后就通过调用 PhysicalOperator::BuildPipelines() 来 top down 的遍历 PhysicalOperator tree 并构造 Pipeline 了。PhysicalOperator::BuildPipelines() 中传递了当前的 Executor，用来存储构造出来的所有 Pipeline。
void Executor::InitializeInternal(PhysicalOperator *plan) {

    auto &scheduler = TaskScheduler::GetScheduler(context);
    {
        lock_guard<mutex> elock(executor_lock);
        physical_plan = plan;

        this->profiler = ClientData::Get(context).profiler;
        profiler->Initialize(physical_plan);
        this->producer = scheduler.CreateProducer();

        auto root_pipeline = make_shared<Pipeline>(*this);
        root_pipeline->sink = nullptr;

        PipelineBuildState state;
        physical_plan->BuildPipelines(*this, *root_pipeline, state);

        this->total_pipelines = pipelines.size();

        root_pipeline_idx = 0;
        ExtractPipelines(root_pipeline, root_pipelines);

        VerifyPipelines();

        ScheduleEvents();
    }
}
PhysicalOperator::BuildPipelines() 是个虚函数，搜索代码可以看到，PhysicalExecute、PhysicalResultCollector、PhysicalCrossProduct、PhysicalDelimJoin、PhysicalIEJoin、PhysicalIndexJoin、PhysicalJoin、PhysicalExport、PhysicalColumnDataScan、PhysicalRecursiveCTE、PhysicalUnion 他们都分别重载了这个虚函数，这些算子大多是像 Join、Union All 这种有多个 Child。其他没有重载该函数的算子，默认的 BuildPipelines 函数如下，主要逻辑是：
如果当前 PhysicalOperator 是 Pipeline breaker，也就是 IsSink() 返回 true，那么就结束当前 Pipeline 的构造：将当前当前算子作为当前 Pipeline 的 source，调用 PhysicalOperator::BuildChildPipeline() 构造下一个 Pipeline。在构造下一个 Pipeline 的 Pipeline 的时候，会新生成一个 Pipeline，并将当前算子作为新 Pipeline 的 sink，然后继续 top down 的遍历当前算子的孩子节点构造新的 Pipeline。
如果不是 Pipeline breaker，并且没有孩子节点了，那么说明遍历到了 PhysicalOperator tree 的一个叶子结点，将当前算子作为 Pipeline 的 source，结束当前 Pipeline 的构造。为了正确遍历完所有 PhysicalOperator tree 的节点构造所有的 Pipeline，当我们遇到有多个孩子节点的算子时，Pipeline 的构造逻辑需要特殊处理下，毕竟 Pipeline 中的计算逻辑是链式的，不再有多个数据源了。
否则，当前算子加入到 Pipeline 的 operators 列表中，继续遍历算子的孩子节点。
void PhysicalOperator::BuildPipelines(Executor &executor, Pipeline &current, PipelineBuildState &state) {
    op_state.reset();
    if (IsSink()) {
        // operator is a sink, build a pipeline
        sink_state.reset();

        // single operator:
        // the operator becomes the data source of the current pipeline
        state.SetPipelineSource(current, this);
        // we create a new pipeline starting from the child
        D_ASSERT(children.size() == 1);

        BuildChildPipeline(executor, current, state, children[0].get());
    } else {
        // operator is not a sink! recurse in children
        if (children.empty()) {
            // source
            state.SetPipelineSource(current, this);
        } else {
            if (children.size() != 1) {
                throw InternalException("Operator not supported in BuildPipelines");
            }
            state.AddPipelineOperator(current, this);
            children[0]->BuildPipelines(executor, current, state);
        }
    }
}
PhysicalOperator::BuildPipelines() 不仅构建了 PhysicalOperator 和 Pipeline 的关系，也构建了 Pipeline 之间的依赖关系：如果某个 PhysicalOperator 是 Pipeline breaker，那么它不仅会作为当前 Pipeline 的 source，也会作为下一个 Pipeline 的 sink，Pipeline breaker 算子隐含了 Pipeline 之间的计算先后关系，只有上游 Pipeline 完全完成计算后才能开启下游 Pipeline 的计算。这一点我们在后面 Pipeline 的计算调度上能更清楚的看到。
PhysicalOperator::BuildChildPipeline() 函数的关键代码如下。可以看到，当前 PhysicalOperator 既作为当前 Pipeline 的 source，也作为了新 Pipeline 的 sink，当前 Pipeline 通过 AddDependency() 函数将新 Pipeline 添加为了依赖的 Pipeline，供后续 Pipeline 的计算调度使用：
void PhysicalOperator::BuildChildPipeline(Executor &executor, Pipeline &current, PipelineBuildState &state,
                                          PhysicalOperator *pipeline_child) {
    auto pipeline = make_shared<Pipeline>(executor);
    state.SetPipelineSink(*pipeline, this);
    // the current is dependent on this pipeline to complete
    current.AddDependency(pipeline);
    // recurse into the pipeline child
    pipeline_child->BuildPipelines(executor, *pipeline, state);
    AddPipeline(executor, move(pipeline), state);
}
以上就是基本的 Pipeline 构造方法，为了更加深刻的理解我们可以看一个单表聚合的简单例子，假设我们有如下这张 events 表：
create table events (id bigint, event_type string);
对于下面这样的单表聚合来说：
select count(*), event_type from events group by event_type order by count(*) limit 1;
​

编辑

切换为全宽
Aggregate and TopN to Pipelines
它物理执行计划如下：
TOP_N <- PROJECTION <- HASH_GROUP_BY <- PROJECTION <- TABLE_SCAN
上面的物理执行计划中，TOP_N、HASH_GROUP_BY 都是 Pipeline breaker，最终构造成的 Pipeline 如下。其中 Pipeline 0 使用 HASH_GROUP_BY 作为 source，Pipeline 1 使用 HASH_GROUP_BY 做为 sink，他们使用同一个 HASH_GROUP_BY 对象，共享一块内存和执行状态，在 Pipeline 1 执行完后才能执行 Pipeline 0，两个 Pipeline 可以各自设置执行的并发度并发执行：
Pipeline 0
|  source: TABLE_SCAN
|  operators: PROJECTION
|  sink: HASH_GROUP_BY
|
Pipeline 1
  source: HASH_GROUP_BY
  operators: PROJECTION
  sink: TOP_N
刚才我们提到，PhysicalOperator::BuildPipelines() 是个虚函数，基本上有多个孩子节点的 PhysicalOperator 都重载了该函数。multi-child 的 PhysicalOperator 情况会比较复杂，DuckDB 特殊处理了 Join 和 Union All 的 Pipeline 构造，我们先从简单的 Union All 入手。
### 从 PhysicalUnion 构造 Pipeline


编辑

切换为全宽
Union All to Pipelines
DuckDB 的 Union All 用 PhysicalUnion 来表示，每个 PhysicalUnion 有 2 个孩子节点。如果用户 SQL 中有 N 个表 Union All，那么就会构造出 N-1 个 PhysicalUnion 算子。PhysicalUnion 仅仅用来汇总多个数据源，传递孩子节点的数据给它的父节点完成计算。
PhysicalUnion 有多个 child 数据源，意味着 PhysicalUnion 往下 top down 构造 Pipeline 的时候需要分别给各个孩子节点传递不同的 Pipeline，那这 2 个 Pipeline 的 sink 应该是什么呢。考虑到 PhysicalUnion 没有计算逻辑仅汇总数据的特殊性，我们可以让这 2 个 Pipeline 共享当前传递过来的 Pipeline 的 sink 和当前的 operators 列表，然后各自在自己的 operators 列表中新增自己的算子，设置自己的 sink。
PhysicalUnion::BuildPipelines() 的代码如下。具体的做法是：从当前 Pipeline 克隆出一个新的 Pipeline，新 Pipeline 和当前 Pipeline 有同样的 sink 和 operators 列表，遍历 PhysicalUnion 的 child 0 继续构造当前 Pipeline，遍历 PhysicalUnion 的 child 1 构造克隆出来的新 Pipeline。：
void PhysicalUnion::BuildPipelines(Executor &executor, Pipeline &current, PipelineBuildState &state) {
    if (state.recursive_cte) {
        throw NotImplementedException("UNIONS are not supported in recursive CTEs yet");
    }
    op_state.reset();
    sink_state.reset();

    auto union_pipeline = make_shared<Pipeline>(executor);
    auto pipeline_ptr = union_pipeline.get();
    auto &union_pipelines = state.GetUnionPipelines(executor);
    // for the current pipeline, continue building on the LHS
    state.SetPipelineOperators(*union_pipeline, state.GetPipelineOperators(current));
    children[0]->BuildPipelines(executor, current, state);
    // insert the union pipeline as a union pipeline of the current node
    union_pipelines[&current].push_back(move(union_pipeline));

    // for the union pipeline, build on the RHS
    state.SetPipelineSink(*pipeline_ptr, state.GetPipelineSink(current));
    children[1]->BuildPipelines(executor, *pipeline_ptr, state);
}
这样的 Pipeline 分裂可以使 PhysicalUnion 父节点的计算逻辑和对应的中间状态在这 2 个 Pipeline 之间复用，虽然 PhysicalUnion 孩子节点的计算逻辑位于不同 Pipeline 之间各自独立产生计算结果，但 PhysicalUnion 之后的计算逻辑和中间状态在不同 Pipeline 之间是共用的，可以确保计算的正确性。
不过这样的 Pipeline 构造带来了额外的问题，我们上面提到 Pipeline breaker 确定了 Pipeline 之间的计算调度关系，并且每个 Pipeline 还可以独立设置自己的并发度。对于 PhysicalUnion 所处的 Pipeline 来说，这个 Pipeline 的 sink 同时属于多个 Pipeline（PhysicalUnion 分裂出来的），只有这些 Pipeline 都完成执行后才能执行他们的下游 Pipeline。所以后面在 Pipeline 调度的时候这里还需要特殊处理下。DuckDB 先将这些被 PhysicalUnion 分裂出来的 Pipeline 存储在了 Executor 的 union_pipelines 中，供后续 Pipeline 的计算调度使用。
为了方便理解，我们来看看下面这个例子：
select count(*), event_type
from (
    select * from events
    union all
    select * from events
) t
group by event_type order by count(*) limit 1;
物理执行计划如下：
TOP_N <- PROJECTION <- HASH_GROUP_BY <- PROJECTION <- UNION <- SEQ_SCAN
                                                            <- SEQ_SCAN
这里的 HASH_GROUP_BY 是一个 Pipeline breaker，将算子树切割成了 2 块，从 HASH_GROUP_BY 开始往下构造 Pipeline 时就会遇到 UNION 算子。遍历到 UNION 时 Pipeline 中的算子已经包含了 HASH_GROUP_BY（sink）和 PROJECTION（位于 operators 列表中），接着再克隆当前 Pipeline，然后分别遍历左右两个孩子节点分别构造 Pipeline，最终构造成的 Pipeline 如下。Pipeline 0 和 Pipeline 2 就是 UNION 分裂出的 2 个 Pipeline，并且在 Executor 的 union_pipelines 里存储了 Pipeline 0 到 Pipeline 2 的映射，方便后续计算调度使用：
Pipeline 0
|  source: TABLE_SCAN
|  operators: PROJECTION
|  sink: HASH_GROUP_BY
|
Pipeline 1
  source: HASH_GROUP_BY
  operators: PROJECTION
  sink: TOP_N

Pipeline 2
  source: TABLE_SCAN
  operators: PROJECTION
  sink: HASH_GROUP_BY

union_pipelines
  Pipeline 0 -> Pipeline 2
理解了 Union All 的 Pipeline 构造，我们再来看看稍微复杂点的 Join。
### 从 PhysicalJoin 构造 Pipeline
PhysicalJoin 的 Pipeline 构造相对来说要复杂一点，我们先从 DuckDB 中 PhysicalJoin 的实现讲起。DuckDB 的 Hash Join 采用了 partitioned hash join，当数据量比较大的时候可以通过 repartition 将数据落盘避免 OOM，然后对 hash join build side 的每个 partition 构造 hash table。然后读取 hash join probe side 的数据 probe 该 partition 对应的 hash table 得到结果返回。probe side 读上来的数据也包含了非当前 partition 的数据，所以这部分数据还需要落盘。等所有 probe side 的数据都 probe 过当前 partition 的数据后，再对 probe side 的数据 repartition，然后多个线程同时处理一个 partition 的数据。
​

编辑

切换为全宽
Hash Join to Pipelines
为了正确产生结果，hash join 在这里有 3 个阶段：
Pipeline 1：并发读取 build 端的数据，每个并发内都一边读一遍构造 thread-local 的 hash table，当所有数据都读完后，在 Pipeline 2 的 Finalize 阶段检查总数据量是否能全部放在内存中，如果不能就将 build 端的数据 repartition，并选出第一批能放在内存中的 partition，为它们构造 hash table，供接下来的 Pipeline 1 使用。
Pipeline 2：处理位于内存中的第一批 partition。读取读取所有 probe 端的数据，probe 当前 build 端位于内存中的 partition 所对应的 hash table，产生结果后交给下一个算子继续处理。
Pipeline 3：处理位于磁盘上的剩余 partition。对 probe 端的数据再次 partition，对每个 partition：将 build 端位于同样 partition 的数据和 hash table 置换进内存中，进行当前 partition 的 hash join，产生结果后交给下一个算子继续处理。
DuckDB 中 PhysicalJoin 根据 1 号（右节点）孩子节点的计算结果 build hash table，然后根据 0 号（左节点）孩子节点的计算结果 probe hash table 完成 join 并对外返回结果。当遍历到 PhysicalJoin 时，probe hash table 的操作应该归属于传下来的 Pipeline，而 build hash table 的操作应该是一个 sink，属于另一个新的 Pipeline。一个基本思路是：
将当前 PhysicalJoin 加入到当前 Pipeline 的 operators 列表中，遍历 PhysicalJoin 的左子树（probe side，也就是 0 号孩子节点）尝试继续构造当前 Pipeline
构造一个新的 Pipeline，设置当前 PhysicalJoin 为其 sink，遍历 PhysicalJoin 的右子树（build side，也就是 1 号孩子节点）构造新的 Pipeline
void PhysicalJoin::BuildJoinPipelines(Executor &executor, Pipeline &current, PipelineBuildState &state,
                                      PhysicalOperator &op) {
    op.op_state.reset();
    op.sink_state.reset();

    // 'current' is the probe pipeline: add this operator
    state.AddPipelineOperator(current, &op);

    // Join can become a source operator if it's RIGHT/OUTER, or if the hash join goes out-of-core
    // this pipeline has to happen AFTER all the probing has happened
    bool add_child_pipeline = false;
    if (op.type != PhysicalOperatorType::CROSS_PRODUCT) {
        auto &join_op = (PhysicalJoin &)op;
        if (IsRightOuterJoin(join_op.join_type)) {
            if (state.recursive_cte) {
                throw NotImplementedException("FULL and RIGHT outer joins are not supported in recursive CTEs yet");
            }
            add_child_pipeline = true;
        }

        if (join_op.type == PhysicalOperatorType::HASH_JOIN) {
            auto &hash_join_op = (PhysicalHashJoin &)join_op;
            hash_join_op.can_go_external = !state.recursive_cte && !IsRightOuterJoin(join_op.join_type) &&
                                           join_op.join_type != JoinType::ANTI && join_op.join_type != JoinType::MARK;
            if (hash_join_op.can_go_external) {
                add_child_pipeline = true;
            }
        }

        if (add_child_pipeline) {
            state.AddChildPipeline(executor, current);
        }
    }

    // continue building the LHS pipeline (probe child)
    op.children[0]->BuildPipelines(executor, current, state);

    // on the RHS (build side), we construct a new child pipeline with this pipeline as its source
    op.BuildChildPipeline(executor, current, state, op.children[1].get());
}
### Pipeline Breaker
在前面看到，通过 PhysicalOperator 构造 Pipeline 时，主要通过 PhysicalOperator 的 IsSink() 函数来判断当前算子是否是一个 pipeline breaker，IsSink() 是个虚函数，如果某个算子没有重载该函数的话默认就会返回 false，所以我们只需要看哪些算子重载了这个虚函数，返回了 true 就可以找到所有的 pipeline breaker 了。搜索代码可以发现 pipeline breaker 主要是这些算子：
Single child PhysicalOperator
PhysicalHashAggregate
PhysicalPerfectHashAggregate
PhysicalUngroupedAggregate
PhysicalWindow
PhysicalExplainAnalyze
PhysicalLimitPercent
PhysicalLimit
PhysicalReservoirSample
PhysicalResultCollector
PhysicalOrder
PhysicalTopN
PhysicalCopyToFile
PhysicalDelete
PhysicalInsert
PhysicalUpdate
PhysicalCreateIndex
PhysicalCreateTableAs
PhysicalExport
PhysicalRecursiveCTE
Multi children PhysicalOperator
PhysicalBlockwiseNLJoin
PhysicalCrossProduct
PhysicalDelimJoin
PhysicalHashJoin
PhysicalIEJoin
PhysicalNestedLoopJoin
PhysicalPiecewiseMergeJoin
比较特殊的一类：
PhysicalVacuum：需要看它自己是否需要 buffer 数据
大体上来说，主要是这些类别：DML、DDL、order by、limit、聚合函数、join、窗口函数（看起来还没有做太多流式优化）
PipelineBreaker 的作用：
反应计算的执行链路：一批数据（一块内存）在 Pipeline 内部经过连续的计算，数据尽可能保持在 Memory/L3/L2/L1/Register 中，只有在 PipelineBreaker 这里才会停止当前一批数据的计算，开始计算下一批数据。
反应计算的依赖关系：只有当前 Pipeline 的计算完成后，它之后的计算才能开始。Pipeline 计算完成意味着当前 Pipeline 处理完了它的所有输入数据，并且能够产生新的数据给之后的计算逻辑使用。
细节 2 ：Pipeline 的调度与执行
###
### Construct Event from Pipelines
在 schedule pipeline 执行的时候，pipeline 已经按照拓扑排序放到了 Executor 的 pipelines 变量中，先从没有 dependency 的 pipeline 开始 schedule
初始时每个 Pipeline 会创建 2 个 Events：PipelineFinishEvent 和 PipelineCompleteEvent，其中 complete 依赖于 finish，finish 依赖当前 pipeline 的 event。finish 先被放到队列，接着是 complete，接着再是当前 pipeline 对应的 event
在为每个 pipeline 创建完（pipeline event，finish event，complete event）的三元组后，再遍历 pipeline 的依赖关系，并让 parent pipeline event 依赖于 dependency pipeline 的 complete event
然后遍历这个初始的 events 列表，找到没有任何 dependency 的，schedule 它。schedule 最终也会调用 pipeline 的 schedule parallel 函数。
举个例子：
CREATE TABLE t(a BIGINT, b BIGINT, c BIGINT, d BIGINT);

SELECT d, AVG(a), SUM(b), COUNT(*)
FROM t
WHERE c > 0
GROUP BY d
ORDER BY COUNT(*) DESC
LIMIT 1;
DuckDB 为其构造出来的 Pipeline 如下：
Pipeline 0：
source: TABLE_SCAN
operators: PROJECTION
sink: HASH_GROUP_BY
dependencies: 无
parents: Pipeline 0
Pipeline 1:
source: HASH_GROUP_BY
operators: 无
sink: TOP_N
dependencies: Pipeline 0
parents: 无
为这些 Pipeline 生成的 Events 如下：
Pipeline 0:
PipelineEvent
PipelineFinishEvent
PipelineCompleteEvent
Pipeline 1:
PipelineEvent
PipelineFinishEvent
PipelineCompleteEvent
如果我们用 其中 B -> A 表示 A Event 依赖于 B Event，Event 之间的依赖关系可以描述为 2 类：
Pipeline 内：PipelineEvent -> PipelineFinishEvent -> PipelineCompleteEvent
Pipeline 之间：如果 Pipeline 0 依赖 Pipeline 1，那么 Pipeline 0 的 PipelineEvent 1 依赖 Pipeline 1 的 PipelineCompleteEvent 1
最终所有的 Event 之间的依赖关系是：
PipelineEvent 0
-> PipelineFinishEvent 0
-> PipelineCompleteEvent 0
-> PipelineEvent 1
-> PipelineFinishEvent 1
-> PipelineCompleteEvent 1
依赖关系反应了数据流向，反应了计算调度的先后顺序，也反应了调度的并发能力。
### Construct and schedule EventTask
构造好 Pipeline 对应的 Event 后，所有 Pipeline 的 Events 倍存储在了一个 vector 中。初始时只调度所有没有依赖的 Event，见 Executor::ScheduleEventsInternal() 函数：
// schedule the pipelines that do not have dependencies
for (auto &event : events) {
    if (!event->HasDependencies()) {
        event->Schedule();
    }
}
其实就是先执行所有 scan 数据的 pipeline，因为只有他们的 event 才没有 dependency。Schedule() 是个虚函数，不同的 Event 有各自不同的 Schedule 函数。初始时这里的 Event 是 PipelineEvent，它的 Schedule 函数会调用到对应 Pipeline 的 Schedule 函数。在 Pipeline 的 Schedule 函数中，会先尝试 ScheduleParallel()，如果失败再尝试 ScheduleSequentialTask()
一个 Pipeline 能够并行执行的条件：source、所有的 operatoers、以及最终的 sink 都能并行执行。如果都满足，那么 ScheduleParallel() 就会继续调用 LaunchScanTasks()

### Event-driven scheduling
grep 代码可以看到总共有这些 Event：
Event
BasePipelineEvent
PipelineEvent
PipelineFinishEvent
OrderMergeEvent
RangeJoinMergeEvent
HashJoinFinalizeEvent
HashJoinPartitionEvent
HashAggregateFinalizeEvent
WindowMergeEvent
DistinctAggregateFinalizeEvent
DistinctCombineFinalizeEvent
PipelineCompleteEvent

## References
Recent issue, contains implementation details: issues/1583, related PR: pull/2393
Early discussion: issues/11
Push-Based Execution in DuckDB by Mark Raasveldt: slides, video
