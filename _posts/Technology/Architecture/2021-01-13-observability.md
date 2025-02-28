---

layout: post
title: BFF
category: 架构
tags: Architecture
keywords: observability

---

## 简介（未完成）

* TOC
{:toc}

## 可观察性

[云原生下的可观察性发展方向](https://mp.weixin.qq.com/s/4Hs-CC1CMBoaQkguHajN_A)当我在说可观察性的时候我在说啥？

![](/public/upload/linux/observability.png)

典型问题排查过程

![](/public/upload/linux/fix_bug.png)

在 IT 系统的可观察性上，也可以类似划分 6 级：

1. 等级 0：手工分析，依靠基础的 Dashboard、告警、日志查询、分布式链路追踪等方式进行手动告警、分析，也是目前绝大部分公司使用的场景
2. 等级 1：智能告警，能够自动去扫描所有的可观察性数据，利用机器学习的方式去识别一些异常并进行自动告警，免去人工设置 / 调整各种基线告警的工作
3. 等级 2：异常关联 + 统一视图，对于自动识别的异常，能够进行上下文的关联，形成一个统一的业务视图，便于快速的定位问题
4. 等级 3：根因分析 + 问题自愈，自动根据异常以及系统的 CMDB 信息直接定位问题的根因，根因定位准确后那边可以去做问题的自愈。这一阶段相当于是一次质的飞跃，在某些场景下可以在人不用参与的情况下实现问题的自愈。
5. 等级 4：故障预测，故障发生总会有损失，所以最好的情况是避免故障的发生，因此故障预测技术可以更好的来保证系统的可靠性，利用之前积累的一些故障先兆信息做到 “未卜先知”
6. 等级 5：变更影响预测，我们知道绝大部分的故障都是由变更引起的，因此如果能够模拟出每个变更对系统带来的影响以及可能产生的问题，我们就能够提前评估出是否能够允许此次变更。

## 监控理念

devops基本理念：
1. if you can't measure it,you can't improve it
2. you build it,you run it, you monitor it.  谁开发，谁运维，谁监控

四种主要的监控方式
1. Logging
2. Tracing
3. Metric
4. Healthchecks

监控是分层次的， 以metric 为例

1. 系统层，比如cpu、内存监控，面向运维人员
2. 应用层，应用出错、请求延迟等，业务开发、框架开发人员
3. 业务层，比如下了多少订单等，业务开发人员


许式伟：信噪比高，有故障就报警，有报警就直指根因。

采用合适的精度：应该仔细设计度量指标的精确度，这涉及到监控的成本问题。例如，每秒收集 CPU 负载信息可能会产生一些有意思的数据，但是这种高频率收集、存储、分析可能成本很高。如果我们的监控目标需要高精度数据，但是却不需要极低的延迟，可以通过采样 + 汇总的方式降低成本。例如：将当前 CPU 利用率按秒记录。按 5% 粒度分组，将对应的 CPU 利用率计数 +1。将这些值每分钟汇总一次。这种方式使我们可以观测到短暂的 CPU 热点，但是又不需要为此付出高额成本进行收集和保留高精度数据。

将需要立即处理和可以第二天处理的报警规则区分一下。每个紧急状态的报警的处理都应该需要某种智力分析过程。如果某个报警只是需要一个固定的机械化动作，那么它就应该被自动化。

**接警后的第一哲学，是尽快消除故障。找根因不是第一位的**。如果故障原因未知，我们可以尽量地保留现场，方便故障消除后进行事故的根因分析。一般来说，有清晰的故障场景的监控报警，都应该有故障恢复的预案。而在那些故障原因不清晰的情况下，消除故障的最简方法是基于流量调度，它可以迅速把用户请求从故障域切走，同时保留了故障现场。

## 监控的几个反模式

1. 事后监控，没有把监控作为系统的核心功能
2. 机械式监控，比如只监控cpu、内存等，程序出事了没报警。只监控http status=200，这样数据出错了也没有报警。
3. 不够准确的监控
4. 静态阈值，静态阈值几乎总是错误的，如果主机的CPU使用率超过80%就发出警报。这种检查 通常是不灵活的布尔逻辑或者一段时间内的静态阈值，它们通常会匹配特定的结果或范围，这种模式 没有考虑到大多数复杂系统的动态性。为了更好地监控，我们需要查看数据窗口，而不是静态的时间点。
5. 不频繁的监控
6. 缺少自动化或自服务

一个良好的监控系统 应该能提供 全局视角，从最高层（业务）依次（到os）展开。同时它应该是：内置于应用程序设计、开发和部署的生命周期中。

很多团队都是按部就班的搭建监控系统：一个常见的例子是监控每台主机上的 CPU、内存和磁盘，但不监控可以指示主机上应用程序是否正常运行的关键服务。如果应用程序在你 没有注意到的情况下发生故障，那么即使进行了监控，你也需要重新考虑正在监控的内容是否合理。**根据服务价值设计自上而下（业务逻辑 ==> 应用程序 ==> 操作系统）**的监控系统是一个很好的方式，这会帮助明确应用程 序中更有价值的部分，并优先监控这些内容，再从技术堆栈中依次向下推进。从业务逻辑和业务输出开始，向下到应用程序逻辑，最后到基础设施。这并不意味着你不需要收集基础设施或操作系统指标——它们在诊断和容量规划中很有帮助——但你不太可能使用这些来报告应用程序的价值。如果无法从业务指标开始，则可试着从靠近用户侧的地方开始监控。因为他们才是最终的客 户，他们的体验是推动业务发展的动力。PS：只要业务没事，底层os一定没事， 底层os没事，业务逻辑不一定没事，监控要尽量能够反应用户的体验。



## 报警哲学

一个好警报的关键是能够在正确的时间、以正确的理由和正确的速度发送，并在其中放入有 用的信息。警报方法中最常见的反模式：
1. 发送过多的警报。比如故障的主机或服务上游会触发其下游的所有内容的警报。你应确保警报系统识别并抑制这些重复 的相邻警报。关注症状而不是原因（因为引发一个症状的原因有很多）。噪声警报会导致警报疲劳，最终警报会被忽略。**修复警报不足比修复过度警报更容易**。
2. 警报的错误分类。比如应设置正确的警报优先级
3. 发送无用的警报。警报应包括适当的上下文。







