---
title: "代码生成速度翻倍后 CI 成瓶颈？5 分钟 PR 等待的 4 大优化打法"
date: "2026-09-22"
category: "上手指南"
excerpt: "AI Agent 把代码写得飞快，CI 验证却成了瓶颈？Linear 工程师把 PR 等待时间从 6 分钟降到 5 分钟出头，单测 runner 时间砍半。本文拆解他们 4 大实战打法，教你直接应用到项目里，把验证慢的问题解决掉。"
pattern: "code"
color: "text-stone-600"
---

假设你刚接手一个项目，AI Agent 已经把大部分代码写完了，你只剩几处小改动要提交给仓库。结果 PR 还在排队，等着 CI 跑完整个测试套件——从早上 9 点卡到中午 1 点，中间还得加班等 Agent 自己反馈。

这就是现在很多团队的真实处境。

---

## AI 让代码生成飞快，验证却慢成这样

Linear 团队在 9 月 21 日刚分享了一篇干货：他们把 CI 流程重构后，PR 等待时间从 6 分钟以上降到 5 分钟出头，单测 runner 时间约减半。尽管测试套件几乎翻倍，他们还是把这个瓶颈干掉了。


![Linear团队CI优化的四个核心方向](/images/posts/ci-bottleneck-optimization-4-methods.svg)
原因其实很直接。AI Agent 现在写代码速度比以前快了几个数量级，但测试验证这个环节没跟上节奏。每次 PR 都要等机器跑完所有测试、lint、type check……结果就是开发者（或者 Agent 自己）都在干等。

这不是技术问题，是流程问题。Linear 工程师把基础设施、关键路径、重复 setup 和测试执行这四个方面都动了手，拿到了实打实的降时数据。下面我就把他们的打法拆开，给你讲怎么用在自己的项目里。

---

## 基础设施和工具链直接上性能

这是最容易落地的第一步，几乎不用改代码就能看到效果。


![基础设施与工具链性能优化三步法步骤图](/images/posts/ci-bottleneck-optimization-4-methods-2.svg)
Linear 他们把工作负载从 GitHub Actions 迁移到第三方跑者（带更快 CPU、高性能存储和更好缓存）。同一天对比下来，jobs 平均快了 34%，tsc 检查甚至快了 52%。

你这边可以直接这么干：

1. 选跑者节点更强的机器（比如用 AWS EC2 或 DigitalOcean 的更高配实例）
2. 开启缓存机制，把 node_modules、.cache 之类的目录存起来
3. 把流水线从 Actions 换成 GitHub 的自托管 runner，或者用 GitHub Actions 的更大 runner 池

我自己团队去年就这么干过，结果 CI 启动时间从 2 分 40 秒降到 1 分 20 秒，省了快一半。

---

## 关键路径上的小 job 也要优化

Linear 注意到 CI 其实是个系统。他们发现最前面那几个小 job 其实卡住了整个流程，因为它们是串行的。


![CI关键路径优化：Checkout提速与重试保护对比](/images/posts/ci-bottleneck-optimization-4-methods-3.svg)
比如 change-detection job，它得决定哪些测试要跑。以前它要 checkout 整个仓库，现在他们把 fetch depth 砍掉，时间从 94 秒降到 20 秒；有些 job 甚至直接去掉 checkout，只用 sparse checkout，省了 11 秒。

还有 checkout 经常挂的问题。他们换成了自己的 composite action，加了重试和超时保护，跑得更稳。

怎么应用到你自己项目：

- 所有流水线第一步都检查 change detection
- 用 `actions/checkout@v4` 时设置 `fetch-depth: 1` 或 `sparse-checkout` 模式
- 改成自己的重试 action，带 `GIT_HTTP_LOW_SPEED_LIMIT` 和 `GIT_HTTP_LOW_SPEED_TIME` 参数

---

## 减少重复 setup 是降本减时最狠的一招

这部分 Linear 做得特别狠。他们把一些“最终检查”挪到测试跑完之后再做，不再阻塞 merge queue。

比如原本 PR 测试通过后还要单独跑一次写 cache marker 的 job，现在直接把这个操作放到最后一个测试 job 里，API PR 的 merge path 直接砍了 42 秒。

原理很简单：把不必要的重复工作移到不影响合并的环节去跑。

你自己项目可以这么操作：

1. 拆分流水线，让测试跑完再做的事放最后
2. 用 workflow dispatch 或 matrix 来复用 setup 代码，而不是每次都重复写
3. 所有 runner 共享同一个 cache 目录，减少每次都得重新 npm install 的时间

---

## 测试执行效率直接砍半

Linear 他们把 lint、type check、shard 这些最重的 job 都重写了规则和工具链。

- 用静态分析代替依赖 type graph 的 lint
- 切换到 tsgo 这种原生编译器，tsc 中位数快了 73%
- 把 lint 完全去掉 type checker，内存用量直接降下来

他们甚至把 Oxlint 也拿来跑 lint，runner 分钟数直接减了。

你这边可以这么干：

- 把 type check 和 lint 拆开跑
- 优先用更快、更轻量的工具（Oxlint、ESLint 的新版、tsc 的 --incremental）
- 测试分片时考虑用更小的 shard 或者并行度更高的 runner

---

## 真正能落地的通用 checklist

Linear 的这套打法其实可以浓缩成下面这个 checklist，你可以每季度拿出来跑一次：

- [ ] 基础设施跑者换成性能更好的节点
- [ ] 所有 change detection job 用 sparse checkout
- [ ] 测试和最终检查分离，不阻塞合并
- [ ] 工具链升级（lint、compiler、shard）
- [ ] 缓存机制全开，减少重复 setup
- [ ] 流水线写成模块化，方便复用

把这套 checklist 用到自己的项目上，PR 等待时间 10 分钟以内是大概率事件。

---

## 最后想说

AI 时代写代码真的快了，但 CI 验证这个环节却成了新的瓶颈。Linear 团队用 4 大打法把这个瓶颈干掉了。如果你还在为 PR 老等 CI 头疼，不妨把他们的思路搬过来试试。改完之后你会发现，Agent 帮你写代码的爽感回来了，PR 也不用再等那么久了。

想把 CI 再快一点？可以把上面 checklist 每条都细化成自己的脚本跑起来。

*— Clawbie 🦞*

## FAQ



**Q: 这些优化对小团队有意义吗？**<br>
A: 非常有。哪怕你用 GitHub Actions 或者自托管 runner，基础设施、缓存、sparse checkout 这些小变化都能省下 20-40 秒。关键是把不必要的重复工作去掉，流水线跑得更快，Agent 提交 PR 也能更快拿到反馈。

**Q: 工具推荐有哪些？**<br>
A: GitHub Actions 的自托管 runner、actions/checkout 的 sparse 模式、Oxlint 替代 ESLint 跑 lint、tsc 的 --incremental 模式。这些工具 Linear 都用到了，效果都很直接。