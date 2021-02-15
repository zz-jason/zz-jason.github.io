---

title: "重构 Projection Elimination"
date: 2017-08-12T11:03:22+08:00
draft: false
toc: false

---

在 TiDB rc4 版本以前，Projection Elimination 是在 Physical Optimization 阶段完成以后做的。老的 Projection Elimination 只能消除那种做纯拷贝，不交换列的顺序，只改变列的名字的 Projection，Projection 消除后，他的 child 直接使用这个被消除后的 Projection 的 Schema。

为什么要重构呢？一个原因现在的 Projection Elimination 是在 Physical Optimization 完成以后再做的，如果把这个优化挪到 Logical Optimization 阶段，将会给接下的其他类型的 Logical 或者 Physical 的 Optimization 提供更多更优的选择；另一个原因是现在的 Projection Elimination 并没有把能消除的 Projection 给消除干净，在 Logical Plan build 完成后，整个 Logical Plan 中可能存在的 Projection 可以分为如下几类：

- 做表达式计算
- 剪裁或复制 child 的某些列（schema 中 column 的数量和 child 的不等）
- 改变 child 某些列的名字（schema 中 column 的数量和 child 的相等）

因为涉及到表达式计算，第一种是无论如何也不能消除的。老的 Projection Elimination 只能消除最后一种，但有些时候第二种我们也能消除。于是我在这 [#3687](https://github.com/pingcap/tidb/pull/3687) 把 Projection Elimination 给重构了。

做这个工作的时候我对 TiDB 的 Plan 也不是很熟悉，重构的过程艰辛无比，大家可以在 pr 的 Conversation 页面看到我和 [@hanfei1991](https://github.com/hanfei1991) [@lamxTyler](https://github.com/lamxTyler) 的大量讨论，重构后，我们的 Projection Elimination 被拆成了两个阶段：一个在 Logical Optimization 阶段，一个在 Physical Optimization 完成后

为什么要分成两个阶段来做呢？在生成 Physical Plan 的时候，某些 Logical Operator 的 schema 可能跟它对应的 Physical Operator 不同，一个典型的例子就是 LogicalJoin，如果生成的是 PhysicalIndexJoin，可能颠倒左右 child 的列在 schema 中出现的顺序，因此，在 Logical Projection Elimination 阶段，我们要保证整个 Plan 的最顶上一定要有最少一个 Projection，但在做完 Physical Optimization 后这个 Projection 又可能只是改变 child 某些列的名字，因此我们在做完 Physical Optimization 后还要消除一次 Projection，并且这个时候只会存在那种改变 child 某些列名的 Projection

Logical Optimization 由一个个的 rule 构成，按照先后顺序分别执行这些 rule。Logical Projection Elimanation 对应的 rule 是 projectionEliminater，放在了 columnPruner 后面。还是按照前文说的，我们需要保留至少一个 Projection 来保序。整个 logical plan 需要保序，如果有 Union，那么以 Union 的 child 为 root 的 plan 也需要保序。Projection 的消除是从底向上的，在消除当前这个 Projection 的时候，以当前 operator 为 root 的 plan 已经完成了 Logical Projection Elimination 的过程，当判断到当前的 Projection 属于可以消除的那两种后，还需要判断消除当前这个 Projection 后整个 plan 是否可以保序，只有在能保序的情况下，这个 Plan 才能够消除，换句话说，只有在这个 Projection 和 root 或者 Union 的路径上还有 Projection，它才可以被消除

最后，在 Physical Optimization 完成后，其实也是整个 Plan 做完了以后，再来扫一下尾。这时候只有第二种 Projection 能够消除了

Projection Elimination 重构后，如果你用的是 rc3 版本的 TiDB 并且包含了这个 commit，那么可能会出现 bug。在 rc3，如果使用 tikv 的话，会走老 plan，这时候在 Logical Optimization 的 ppdSolver rule 里面会做 Join Reorder，Join Reorder 后不会去更新父亲的 schema，如果父亲不是 Projection，这个时候就可能出现 bug 了，如果遇到这种情况，请升级 rc4，因为 rc4 废弃老 planer 使用新 planner，这个问题在 rc3 并没有修复
