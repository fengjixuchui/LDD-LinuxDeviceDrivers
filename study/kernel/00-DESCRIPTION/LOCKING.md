---

title: 锁机制
date: 2021-02-15 00:32
author: gatieme
tags:
    - linux
    - tools
categories:
        - 技术积累
thumbnail:
blogexcerpt: 虚拟化 & KVM 子系统

---

<br>

本作品采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可, 转载请注明出处, 谢谢合作

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a>

因本人技术水平和知识面有限, 内容如有纰漏或者需要修正的地方, 欢迎大家指正, 鄙人在此谢谢啦

**转载请务必注明出处, 谢谢, 不胜感激**

<br>

| 日期 | 作者 | GitHub| CSDN | BLOG |
| ------- |:-------:|:-------:|:-------:|:-------:|
| 2021-02-15 | [成坚-gatieme](https://kernel.blog.csdn.net) | [`AderXCoding/system/tools/fzf`](https://github.com/gatieme/AderXCoding/tree/master/system/tools/fzf) | [使用模糊搜索神器 FZF 来提升办公体验](https://blog.csdn.net/gatieme/article/details/113828826) | [Using FZF to Improve Productivit](https://oskernellab.com/2021/02/15/2021/0215-0001-Using_FZF_to_Improve_Productivity)|


<br>

2   **锁**
=====================




**-*-*-*-*-*-*-*-*-*-*-*-*-*-*-*　重要功能和时间点　-*-*-*-*-*-*-*-*-*-*-*-*-*-*-***





下文将按此目录分析 Linux 内核中 MM 的重要功能和引入版本:




**-*-*-*-*-*-*-*-*-*-*-*-*-*-*-* 正文 -*-*-*-*-*-*-*-*-*-*-*-*-*-*-***

# 1 SPINLOCK
-------

[Linux 中的 spinlock 机制 [一] - CAS 和 ticket spinlock](https://zhuanlan.zhihu.com/p/80727111)

[Linux 中的 spinlock 机制 [二] - MCS Lock](https://zhuanlan.zhihu.com/p/89058726)

[Linux 中的 spinlock 机制 [三] - qspinlock](https://zhuanlan.zhihu.com/p/100546935)

[Non-scalable locks are dangerous](https://pdos.csail.mit.edu/papers/linux:lock.pdf)
[Non-scalable locks are dangerous](https://andreybleme.com/2021-01-24/non-scalable-locks-are-dangerous-summary)
[Non-scalable locks are dangerous](https://www.jianshu.com/p/d058fb620f89)
[Scalable   Locking](https://pdos.csail.mit.edu/6.828/2018/lec/l-mcs.pdf)

## 1.1 CAS(compare and swap) LOCK
-------

CAS 的原理是, 将旧值与一个期望值进行比较, 如果相等, 则更新旧值.

这种是实现 spinlock 用一个整形变量表示, 其初始值为 1, 表示 available 的状态(可以被 1 用户独占).

1.  当一个 CPU A 获得 spinlock 后, 会将该变量的值设为 0(不能被任何用户再占有), 之后其他 CPU 试图获取这个 spinlock 时, 会一直等待, 直到 CPU A 释放 spinlock, 并将该变量的值设为 1.

2.  等待该 spinlock 的 CPU B 不断地把 「期望的值」 1 和 「实际的值」 (1 或者 0)进行比较(compare), 当它们相等时, 说明持有 spinlock 当前未被任何人持有(持有的 CPU 已经释放了锁或者锁一直未被任何人持有), 那么试图获取 spinlock 的 CPU 就会尝试将 "new" 的值(0) 写入 "p"(swap), 以表明自己成为占有了 spinlock, 成为新的 owner.

这里只用了 0 和 1 两个值来表示 spinlock 的状态, 没有充分利用 spinlock 作为整形变量的属性, 为此还有一种衍生的方法, 可以判断当前 spinlock 的争用情况. 具体规则是: 每个 CPU 在试图获取一个 spinlock 时, 都会将这个 spinlock 的值减1, 所以这个值可以是负数, 而「负」的越多(负数的绝对值越大), 说明当前的争抢越激烈.

存在的问题
基于CAS的实现速度很快, 尤其是在没有真正竞态的情况下(事实上大部分时候就是这种情况),  但这种方法存在一个缺点：它是「不公平」的.  一旦spinlock被释放, 第一个能够成功执行CAS操作的CPU将成为新的owner, 没有办法确保在该spinlock上等待时间最长的那个CPU优先获得锁, 这将带来延迟不能确定的问题.

## 1.2 ticket LOCK
-------

[Ticket spinlocks](https://lwn.net/Articles/267968)


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2007/11/01 | Nick Piggin <npiggin@suse.de> | [ticket spinlocks for x86](https://lore.kernel.org/patchwork/cover/95892) | X86 架构 ticket spinlocks 的实现. | v1 ☑ 2.6.25-rc1(部分合入) | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/85789)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/cover/95892), [PatchWork](https://lore.kernel.org/patchwork/cover/95894) |
| 2021/09/19 | Guo Ren <guoren@kernel.org> | [riscv: locks: introduce ticket-based spinlock implementation](https://patchwork.kernel.org/project/linux-riscv/patch/20210919165331.224664-1-guoren@kernel.org) | riscv 架构 ticket spinlocks 的实现. | v1 ☐ |[PatchWork](https://patchwork.kernel.org/project/linux-riscv/patch/20210919165331.224664-1-guoren@kernel.org) |
| 2013/10/09 | Nick Piggin <npiggin@suse.de> | [arm64: locks: introduce ticket-based spinlock implementation](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=52ea2a560a9dba57fe5fd6b4726b1089751accf2) | ARM64 架构 ticket spinlocks 的实现. | v1 ☑ 3.13-rc1 | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=52ea2a560a9dba57fe5fd6b4726b1089751accf2) |
| 2015/02/10 | Nick Piggin <npiggin@suse.de> | [arm64: locks: patch in lse instructions when supported by the CPU](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=81bb5c6420635dfd058c210bd342c29c95ccd145) | ARM64 ticket spinlocks 使用 LSE 进行优化. | v1 ☑ 4.3-rc1 | [COMMIT](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=81bb5c6420635dfd058c210bd342c29c95ccd145) |
| 2022/04/14 | Palmer Dabbelt <palmer@rivosinc.com> | [RISC-V: Generic Ticket Spinlocks](https://patchwork.kernel.org/project/linux-riscv/cover/20220414220214.24556-1-palmer@rivosinc.com/) | 632429 | v3 ☐☑ | [LORE v3,0/7](https://lore.kernel.org/all/20220414220214.24556-1-palmer@rivosinc.com) |


[Linux中的spinlock机制[一] - CAS和ticket spinlock](https://zhuanlan.zhihu.com/p/80727111)

[BAKERY ALGORITHM](https://remonstrate.wordpress.com/tag/bakery-algorithm)
[Lamport 面包店算法](https://blog.csdn.net/pizi0475/article/details/17649949)

## 1.3 MCS lock
-------

[MCS locks and qspinlocks](https://lwn.net/Articles/590243)

spinlock 的值出现变化时, 所有试图获取这个 spinlock 的 CPU 都需要读取内存, 刷新自己对应的 cache line, 而最终只有一个 CPU 可以获得锁, 也只有它的刷新才是有意义的. 锁的争抢越激烈(试图获取锁的CPU数目越多), 无谓的开销也就越大.

如果在 ticket spinlock 的基础上进行一定的修改, 让每个 CPU 不再是等待同一个 spinlock 变量, 而是基于各自不同的 per-CPU 的变量进行等待, 那么每个 CPU 平时只需要查询自己对应的这个变量所在的本地 cache line, 仅在这个变量发生变化的时候, 才需要读取内存和刷新这条 cache line, 这样就可以解决上述的这个问题.

要实现类似这样的 spinlock的 「分身」, 其中的一种方法就是使用 MCS lock. 试图获取一个 spinlock 的每个CPU, 都有一份自己的 MCS lock.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2014/01/21 | Tim Chen <tim.c.chen@linux.intel.com> | [MCS Lock: MCS lock code cleanup and optimizations](https://lore.kernel.org/patchwork/cover/435770) | MCS LOCK 重构, 增加了新的文件 `include/linux/mcs_spinlock.h` | v9 ☑ 3.15-rc1 | [LKML v6 0/6](https://lkml.org/lkml/2013/9/25/532)<br>*-*-*-*-*-*-*-* <br>[PatchWork v9 0/6](https://lore.kernel.org/patchwork/cover/435770), [关键 commit](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=e72246748ff006ab928bc774e276e6ef5542f9c5) |
| 2014/02/10 | Peter Zijlstra <peterz@infradead.org> | [locking/core patches](https://lore.kernel.org/patchwork/cover/440565) | PV SPINLOCK | v1 ☑ 4.2-rc1 | [PatchWork](https://lore.kernelorg/lkml/20140210195820.834693028@infradead.org) |


## 1.4 qspinlock
-------

### 1.4.1 X86
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/08/28 | Nick Piggin <npiggin@suse.de> | [queueing spinlocks?](https://lore.kernel.org/patchwork/cover/127444) | X86 架构 qspinlocks 的实现. | RFC ☐ | [PatchWork RFC](https://lore.kernel.org/lkml/20080828073428.GA19638@wotan.suse.de) |
| 2015/04/24 | Waiman Long <Waiman.Long@hp.com> | [qspinlock: a 4-byte queue spinlock with PV support](https://lore.kernel.org/patchwork/cover/127444) | X86 架构 qspinlocks 的实现. | v16 ☑ 4.2-rc1 | [PatchWork RFC](https://lore.kernel.org/lkml/20140310154236.038181843@infradead.org)<br>*-*-*-*-*-*-*-* <br>[LORE v16 00/14](https://lore.kernel.org/all/1429901803-29771-1-git-send-email-Waiman.Long@hp.com/) |

### 1.4.2 ARM64
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/04/10 | Yury Norov <ynorov@caviumnetworks.com> | [arm64: queued spinlocks and rw-locks](http://patches.linaro.org/cover/98492) | ARM64 架构 qspinlocks 的实现. | RFC ☐ | [PatchWork RFC](https://patchwork.kernel.org/project/linux-arm-kernel/patch/1491860104-4103-4-git-send-email-ynorov@caviumnetworks.com)<br>*-*-*-*-*-*-*-* <br>[LORE 0/3](https://lore.kernel.org/lkml/20170503145141.4966-1-ynorov@caviumnetworks.com) |
| 2018/04/26 | Will Deacon <will.deacon@arm.com> | [kernel/locking: qspinlock improvements](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=baa8c6ddf7be33f2b0ddeb68906d668caf646baa) | 优化并实现了 qspinlocks 的通用框架. | v3 ☑ 4.18-rc1 | [LWN](https://lwn.net/Articles/751105), [LORE v3,00/14](https://lore.kernel.org/all/1524738868-31318-1-git-send-email-will.deacon@arm.com) |
| 2018/06/26 | Will Deacon <will.deacon@arm.com> | [Hook up qspinlock for arm64](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=5d168964aece0b4a41269839c613683c5d7e0fb2) | ARM64 架构 qspinlocks 的实现. | v1 ☑ 4.19-rc1 | [LORE 0/3](https://lore.kernel.org/linux-arm-kernel/1530010812-17161-1-git-send-email-will.deacon@arm.com) |


### 1.4.3
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/06/21 | guoren@kernel.org <guoren@kernel.org> | [riscv: Add qspinlock support](https://lore.kernel.org/all/20220621144920.2945595-1-guoren@kernel.org) | TODO | v6 ☐☑✓ | [LORE v5](https://lore.kernel.org/lkml/20220620155404.1968739-1-guoren@kernel.org)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/2](https://lore.kernel.org/all/20220621144920.2945595-1-guoren@kernel.org) |


## 1.5 PV_SPINLOCK
-------

[PV qspinlock 原理](https://blog.csdn.net/bemind1/article/details/118224344)

[Hook 内核之 PVOPS](https://diting0x.github.io/20170101/pvops)

spinlock 在非虚拟化的环境下, 它是可以认为 CPU 不会被抢占的, 所以 A 拿锁干活, B 死等 A, A 干完自己的活, 就释放了, 中间不会被调度.

但是在虚拟化下, CPU 对应到 vcpu, 每个 vcpu 跟之前裸机上的进程调度一样, 所以 A 拿锁干活, 并不一定不会被抢占, 很有可能被调度走了, 因为 cpu 这时候还不知道 vcpu 在干嘛. B 死等 A, 但是 A 被调度了, 运行了 C, C 也要死等 A, 在一些设计不够好的系统里面, 这样就会变得很糟糕.


为了保证spinlock unlock的公平性, 有一种队列的spinlock, ticketlock,  http://www.ibm.com/developerworks/cn/linux/l-cn-spinlock/这篇文章介绍的非常详细, 总之根据next, own, 来判断是否到自己了. 这样一种机制在裸机上是可以解决公平的问题, 但是放到虚拟化环境里, 它会使问题变得更糟. C必须等到B完成才可以, 如果中间B被调度了, 又开始循环了, 当然更糟的定义也是相对的, 如果vcpu的调度机制能够vcpu正在拿锁的话, 会怎样？


jeremy很早就写了一个pv ticketlock, 原理大概就是vcpu在拿锁了一段时间, 会放弃cpu, 并blocked, unlocked的时候会被唤醒, 这个针对PV制定的优化, 在vcpu拿不到锁的场景下, 并没有任何的性能损耗, 并且能够解决之前的问题, 但是当运行native linux的时候, 就会有性能损耗, 所以当时在config里面添加了一个编译选项CONFIG_PARAVIRT_SPINLOCK, 话说我们的系统里面, 这个是没打开的啊, 后面要再好好评估下


之后, 这个patch进行了改良, 在原有native linux的ticketlock的基础, 增加了一种模式, 通过检测cpu是否spinned一段时间, 判断是否要进入slow path, 之前的fast path的逻辑和原来保持不变, 进入slowpath后, 会在ticketlock里面置位, 并block vcpu, 等unlock的时候, 这个位会被clear, 因为占用了一个位, 所以能用的ticket少了一半.

这个方案在一些硬件(XEON x3450)上进行各种benchmark测试后, 结论是不再有任何的性能损耗.



好吧, 说了这么多理论性的东西, 再来说下, 我们实际遇到的问题.  很早以前经常有windows用户的工单投诉, 说自己的vm里面cpu没有怎么使用, 为什么cpu显示百分之百.

由于是windows系统, 加上我是小白, 很难给出一些技术细节上的分析, 只能通过简单滴一些测试实验进行调查.



最后的结果, 就是windows很多核的情况下, 比如12、16,  在一个稍微有点load的物理机上面, 跑一些cpu压力, 就很容易出现cpu百分百的问题, 后面降core之后, 情况有所缓解. 后面大致的分析结果是, windows里面很多操作是用到spinlock的, 当一个core拿到锁, 事情没有做完, 被调度了, 这时其他的core也需要拿锁, 当core越来越多的时候, 情况就越来越糟, 最后看上去就大家都很忙, 但实际什么事情也没做.



目前来看, 已经有一种较为成熟的软件方法来解决类似问题, 期待后续是否会有硬件的一些特性来支持, 或许已经有了.


PV_SPINLOCKS 的合入引起了[性能问题 Performance overhead of paravirt_ops on native identified](https://lore.kernel.org/patchwork/patch/156045), yinru


当开启了 CONFIG_PARAVIRT_SPINLOCKS 之后, queued_spin_lock_slowpath() 将作为[宏函数被展开多份](https://elixir.bootlin.com/linux/v4.2/source/kernel/locking/qspinlock.c#L281), 一份 [native_queued_spin_lock_slowpath()](https://elixir.bootlin.com/linux/v4.2/source/kernel/locking/qspinlock.c#L255) 用于传统的 spinlock 场景, 一份 [`__pv_queued_spin_lock_slowpath()`](https://elixir.bootlin.com/linux/v4.2/source/kernel/locking/qspinlock.c#L468) 用于虚拟化场景. 参见 [v4.2: commit a23db284fe0d locking/pvqspinlock: Implement simple paravirt support for the qspinlock](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a23db284fe0d1879ca2002bf31077b5efa2fe2ca). 在启动阶段, 通过 PVOPS 机制动态的的将 spinlock 替换为内核实际所需的 spinlock 处理函数. 虚拟化 guest 中将在 kvm_spinlock_init() 中被替换为虚拟化场景[所需的 `__pv_queued_spin_lock_slowpath()` ](https://elixir.bootlin.com/linux/v4.2/source/arch/x86/kernel/kvm.c#L868) 等函数.



| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2008/07/07 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [Paravirtual spinlocks](https://lore.kernel.org/patchwork/patch/121810) | PV_SPINLOCK 实现. | RFC ☑ 2.6.27-rc1 | [PatchWork RFC](https://lore.kernel.org/patchwork/cover/121810) |
| 2009/05/15 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [x86: Fix performance regression caused by paravirt_ops on native kernels](https://lore.kernel.org/patchwork/patch/156045/) | NA | v13 ☑ 3.12-rc1 | [PatchWork v13](https://lore.kernel.org/patchwork/cover/156402) |
| 2013/08/09 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [Paravirtualized ticket spinlocks](https://lore.kernel.org/patchwork/cover/398912) | PV_SPINLOCK 的 ticket lock 实现. | v13 ☑ 3.12-rc1 | [PatchWork v13](https://lore.kernel.org/patchwork/cover/398912) |
| 2015/04/07 | Waiman Long <Waiman.Long@hp.com> | [qspinlock: a 4-byte queue spinlock with PV support](https://lore.kernel.org/patchwork/cover/558505) | PV SPINLOCK | v15 ☑ 4.2-rc1 | [PatchWork v15](https://lore.kernel.org/patchwork/cover/558505) |
| 2015/11/10 | Waiman Long <Waiman.Long@hpe.com> | [locking/qspinlock: Enhance pvqspinlock performance](https://lore.kernel.org/patchwork/cover/616398) | PV SPINLOCK | v10 ☑ 4.5-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/588106)<br>*-*-*-*-*-*-*-* <br>[PatchWork v10](https://lore.kernel.org/patchwork/cover/616398) |
| 2018/10/08 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [Enable PV qspinlock for Hyper-V](https://lore.kernel.org/patchwork/cover/996494) | Hyper-V 的 PV spiclock 实现. | v2 ☑ 4.20-rc1 | [PatchWork v2](https://lore.kernel.org/patchwork/cover/996494) |
| 2019/10/23 | Zhenzhong Duan <zhenzhong.duan@oracle.com> | [Add a unified parameter "nopvspin"](https://lore.kernel.org/patchwork/cover/1143398) | PV SPINLOCK | v8 ☑ 5.9-rc1 | [PatchWork v8](https://lore.kernel.org/patchwork/cover/1143398) |
|


## 1.5 NumaAware SPINLOCK
-------

[关于多核 CPU 自旋锁 (spinlock) 的优化](https://blog.csdn.net/cpongo1/article/details/89539933)

[NUMA-aware qspinlocks](https://lwn.net/Articles/852138)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/08/24 | H.J. Lu  | [numa-spinlock](https://sanidhya.github.io/pubs/2019/shfllock.pdf) | 阿里实现的用户态 numa aware spinlock. | ☐ |[GitLab](https://gitlab.com/numa-spinlock/numa-spinlock) |
| 2019/08/24 | sanidhya | [Scalable and Practical Locking with Shuffling](https://sanidhya.github.io/pubs/2019/shfllock.pdf) | Shuffling 锁实现了洗牌技术. 将等待锁的线程更指定的策略进行重新排序. 类似于通过定义的比较功能对 waiter 进行排序. 实现 NUMA 感知的唤醒和阻塞策略. 洗牌的动作通常不会在关键路径上执行. | ☐ | [GitHub](https://github.com/sslab-gatech/shfllock) |
| 2019/08/08 | dozenow | ShortCut: Accelerating Mostly-Deterministic Code Regions | NA | ☐ | [GitHub](hhttps://github.com/shortcut-sosp19/shortcut) |
| 2021/05/14 | Alex Kogan <alex.kogan@oracle.com> | [Add NUMA-awareness to qspinlock](https://lwn.net/Articles/856387) | NUMA 感知的 spinlock, 基于 [CNA-compact-numa-aware-locks](https://deepai.org/publication/compact-numa-aware-locks). | v15 ☐ | [PatchWork v15](https://lore.kernel.org/patchwork/cover/1428910), [PatchWork v15,0/6 ARM](https://patchwork.kernel.org/project/linux-arm-kernel/cover/20210514200743.3026725-1-alex.kogan@oracle.com) |



## 1.6 SPINlOCK DEBUG
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2021/05/14 | Alex Kogan <alex.kogan@oracle.com> | [x86, locking/qspinlock: Allow lock to store lock holder cpu number](https://lkml.org/lkml/2020/7/16/1066) | struct raw_spinlock 中有 owner_cpu, 但是需要开启 CONFIG_DEBUG_SPINLOCK, 这个选项对 spinlock 的性能影响较大. 这个补丁集修改 x86 的 qspinlock 和 qrwlock 代码, 允许它在可行的情况下将锁持有者的 cpu 号 (qrwlock 的锁写入器 cpu 号) 存储在锁结构本身, 这对调试和崩溃转储分析很有用. 通过定义宏 `__cpu_number_sadd1` (用于 qrwlock) 和 `__cpu_number_sadd2` (用于 qspinlock), 以达到饱和的 + 1 和 + 2 cpu 数, 可以在 qspinlock 和 qrwlock 的锁字节中使用. 可以在每个体系结构的基础上启用该功能, 当前只提供了 x86 下的实现.| v2 ☐ | [PatchWork v2,0/5](https://lkml.org/lkml/2020/7/16/1066) |


# 2 RWSEM
-------

## 2.1 RWSEM
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2020/11/21 | Waiman Long <longman@redhat.com> | [locking/rwsem: Rework reader optimistic spinning](https://lore.kernel.org/patchwork/cover/1342950) | 当读者的评论部分很短, 周围没有那么多读者时, 读者乐观旋转(osq_lock)是有帮助的. 它还提高了读者获得锁的机会, 因为写入器乐观旋转对写入器的好处远远大于读者. 由于提交d3681e269fff ("locking/rwsem: Wake up almost all reader in wait queue"), 所有等待的reader都会被唤醒, 这样它们就都能获得读锁并并行运行. 当竞争的读者数量很大时, 允许读者乐观自旋很可能会导致读者碎片, 多个较小的读者组可以以顺序的方式(由写入器分隔)获得读锁. 这降低了读者的并行性. 解决这个缺点的一种可能方法是限制能够进行乐观旋转的读者的数量(最好是一个). 这些读者作为等待队列中所有等待的读者的代表, 因为一旦获得锁, 它们将唤醒所有等待的读者.  | v2 ☐ | [PatchWork v2,0/5](https://lore.kernel.org/patchwork/cover/1342950) |


## 2.2 PER-CPU RWSEM
-------

percpu rw 信号量是一种新的读写信号量设计, 针对读取锁定进行了优化.

传统的读写信号量的问题在于, 当多个内核读取锁时, 包含信号量的 cache-line 在内核的 L1 缓存之间弹跳, 导致性能下降. 锁定读取速度非常快, 它使用 RCU, 并且避免了锁定和解锁路径中的任何原子指令. 另一方面, 写入锁定非常昂贵, 它调用 synchronize_rcu () 可能需要数百毫秒. 使用 RCU 优化 rw-lock 的想法是由 Eric Dumazet <eric.dumazet@gmail.com>. 代码由 Mikulas Patocka <mpatocka@redhat.com> 编写

锁是用 “struct percpu_rw_semaphore” 类型声明的. 锁被初始化为 percpu_init_rwsem, 它在成功时返回 0, 在分配失败时返回 -ENOMEM. 必须使用 percpu_free_rwsem 释放锁以避免内存泄漏.

使用 percpu_down_read 和 percpu_up_read 锁定读取, 使用 percpu_down_write 和 percpu_up_write 锁定写入.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/06/22 | Peter Zijlstra <peterz@infradead.org> | [percpu rwsem -v2](https://lore.kernel.org/all/20150622121623.291363374@infradead.org) | TODO | v2 ☐☑✓ | [LORE v2,0/13](https://lore.kernel.org/all/20150622121623.291363374@infradead.org) |
| 2012/08/31 | Mikulas Patocka <mpatocka@redhat.com> | [Fix a crash when block device is read and block size is changed at the same time](https://lore.kernel.org/patchwork/cover/323377) | NA | v2 ☑ 3.8-rc1 | [PatchWork 0/4](https://lore.kernel.org/patchwork/cover/323377) |
| 2020/11/07 | Oleg Nesterov <oleg@redhat.com> | [percpu_rw_semaphore: reimplement to not block the readers unnecessarily](https://lore.kernel.org/patchwork/cover/1342950) | NA | v2 ☑ 3.8-rc1 | [PatchWork v2,0/5](https://lore.kernel.org/patchwork/cover/339247), [PatchWork](https://lore.kernel.org/patchwork/cover/339702)<br>*-*-*-*-*-*-*-* <br>[PatchWork](https://lore.kernel.org/patchwork/patch/339064) |
| 2020/11/18 | Oleg Nesterov <oleg@redhat.com> | [percpu_rw_semaphore: lockdep + config](https://lore.kernel.org/patchwork/cover/1342950) | NA | v1 ☑ 3.8-rc1 | [PatchWork 0/3](https://lore.kernel.org/patchwork/cover/341521) |
| 2020/01/31 | Peter Zijlstra <peterz@infradead.org> | [locking: Percpu-rwsem rewrite](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=41f0e29190ac9e38099a37abd1a8a4cb1dc21233) | TODO | v1 ☐☑✓ | [LORE v1,0/7](https://lore.kernel.org/all/20200131150703.194229898@infradead.org) |


# 3 MUTEX
-------


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2013/4/17 | Waiman Long <Waiman.Long@hp.com> | [mutex: Improve mutex performance by doing less atomic-ops & better spinning](https://lkml.org/lkml/2013/4/17/418) | NA | v4 ☑ 3.10-rc1 | [LKML 0/3 v2](https://lkml.org/lkml/2013/4/15/245)<br>*-*-*-*-*-*-*-* <br>[LKML v4 0/4](https://lkml.org/lkml/2013/4/17/418) |


# 4 membarrier
-------




| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/10/19 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [membarrier: Provide register expedited private command](https://lore.kernel.org/patchwork/cover/843003) | 引入 MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED. | v6 ☑ 4.14-rc6 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/835747)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6](https://lore.kernel.org/patchwork/cover/398912) |
| 2017/11/21 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [membarrier: Provide core serializing command](https://lore.kernel.org/patchwork/cover/843003) | 引入 MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED. | v6 ☑ 4.16-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/835747)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6](https://lore.kernel.org/patchwork/cover/398912) |
| 2018/01/29 | Raghavendra K T <raghavendra.kt@linux.vnet.ibm.com> | [membarrier: Provide core serializing command](https://lore.kernel.org/patchwork/cover/843003) | 引入 MEMBARRIER_CMD_REGISTER_PRIVATE_EXPEDITED. | v6 ☑ 4.16-rc1 | [PatchWork v5](https://lore.kernel.org/patchwork/cover/835747)<br>*-*-*-*-*-*-*-* <br>[PatchWork v6](https://lore.kernel.org/patchwork/cover/398912) |


# 5 RCU()
-------

[What is RCU, Fundamentally?](https://lwn.net/Articles/262464)

[What is RCU? Part 2: Usage](https://lwn.net/Articles/263130)

[Recent RCU changes](https://lwn.net/Articles/894379)

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2019/06/01 | "Joel Fernandes (Google)" <joel@joelfernandes.org> | [Harden list_for_each_entry_rcu() and family](https://lore.kernel.org/patchwork/cover/1082845) | 本系列增加了一个新的内部函数rcu_read_lock_any_held(), 该函数在调用这些宏时检查reader节是否处于活动状态. 如果不存在reader section, 那么list_for_each_entry_rcu()的可选第四个参数可以是一个被计算的lockdep表达式(类似于rcu_dereference_check()的工作方式). . | RFC ☑ 5.4-rc1 | [PatchWork RFC,0/6](https://lore.kernel.org/patchwork/cover/1082845) |



# 6 FUTEX
-------

[FUTEX2's sys_futex_waitv() Sent In For Linux 5.16 To Help Linux Gaming](https://www.phoronix.com/scan.php?page=news_item&px=Linux-5.16-sys_futex_waitv)


[FUTEX2 Begins Sorting Out NUMA Awareness](https://www.phoronix.com/scan.php?page=news_item&px=FUTEX2-NUMA-Awareness-RFC)

# 7 Semaphores
-------

[The Little Book of Semaphores](https://www.greenteapress.com/semaphores/LittleBookOfSemaphores.pdf)

# 8 LockLess
-------


[An introduction to lockless algorithms](https://lwn.net/Articles/844224)


# 9 ATOMIC
-------

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2015/08/06 | Will Deacon <will.deacon@arm.com> | [Add generic support for relaxed atomics](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=0ca326de7aa9cb253db9c1a3eb3f0487c8dbf912) | ARM64 引入 relaxed atomics. | v5 ☑ 4.3-rc1 | [LORE 0/5](https://lore.kernel.org/lkml/1436790687-11984-1-git-send-email-will.deacon@arm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v2,0/7](https://lore.kernel.org/lkml/1437060758-10381-1-git-send-email-will.deacon@arm.com)<br>*-*-*-*-*-*-*-* <br>[LORE v5,0/8](https://lore.kernel.org/all/1438880084-18856-1-git-send-email-will.deacon@arm.com) |

# 10 LOCKDEP
-------


## 10.1 LOCKDEP
-------

Lockdep 跟踪锁的获取顺序, 以检测死锁, 以及 IRQ 和 IRQ 启用/禁用状态, 并考虑故障现场分析或获取. Lockdep 在检测到并报告死锁后应立即关闭, 因为由于设计复杂, 检测后数据结构和算法不可重用.


| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2006/07/03 | Paul Mackerras <paulus@samba.org> | [Add generic support for relaxed atomics](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=829035fd709119d9def124a6d40b94d317573e6f) | LOCKDEP 死锁检测机制. | v5 ☑ 2.6.18-rc1 | [LORE 0/5](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=829035fd709119d9def124a6d40b94d317573e6f) |


参见 [[PATCH RFC v6 00/21] DEPT(Dependency Tracker)](https://lore.kernel.org/lkml/1651652269-15342-1-git-send-email-byungchul.park@lge.com), 分析了 Lockdep 的问题以及引入 Dependency Tracker 的背景和设计思路.

但是 Lockdep 依旧有太多问题

1.  对于与实际锁无关的等待和事件, 比如时间等待等机制, 如果无法完成, 最终也会导致死锁. 但是 Lockdep 只能通过分析锁的获取顺序来完成思索检测, 对于与实际锁无关的等待和事件, 无法识别和处理, 只能通过过模拟锁来完成.

2.  更糟糕的是, Lockdep 存在太多假阳性检测, 这可能阻止了本身更有价值的进一步检测.

3.  此外, 通过跟踪获取顺序, 它不能正确地处理读锁和交叉事件, 例如 wait_for_completion ()/complete () 用于死锁检测. Lockdep 不再是实现这一目的的好工具.


## 10.2 Crossrelease Feature
-------

不仅是锁操作, 而且任何导致等待或旋转的操作都可能导致死锁, 除非它最终被某人“释放”. 这里重要的一点是, 等待或旋转必须由某人"释放". 但是很明显 LOCKDEP 无法检测到此类问题.

因此社区开发者 Byungchul Park 开发了交叉释放功能(Crossrelease Feature), 使得 LOCKDEP 不仅可以检查典型锁的依赖关系并检测死锁可能性, 还可以检查 lock_page()、wait_for_xxx() 等等待事件, 这些锁/等待可能在任何上下文中被释放.

这个特性最终经历了 v8 之后在 v4.14 被合入主线. 一开始的时候它报告很多有价值的隐藏死锁问题, 但随后也报告了诸多假阳性的死锁. 当然, 没有人喜欢 Lockdep 的假阳性报告, 因为它使 Lockdep 停止, 阻止报告更多的真实问题.

这造成越来越多地开发者直接关闭甚至不再使用 LOCKDEP 特性. 最终 Ingo 跟作者讨论后, 在 v4.15 回退了交叉释放功能. 参见 [locking/lockdep: Remove the cross-release locking checks](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=e966eaeeb623f09975ef362c2866fae6f86844f9).

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2017/08/07 | Byungchul Park <byungchul.park@lge.com> | [lockdep: Implement crossrelease feature](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=ef0758dd0fd70b98b889af26e27f003656952db8) | 交叉释放功能(Crossrelease Feature). | v8 ☑✓ 4.14-rc1 | [LORE v8,0/14](https://lore.kernel.org/all/1502089981-21272-1-git-send-email-byungchul.park@lge.com) |
| 2017/12/13 | Ingo Molnar <mingo@kernel.org> | [locking/lockdep: Remove the cross-release locking checks](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/log/?id=e966eaeeb623f09975ef362c2866fae6f86844f9) | 回退交叉释放功能(Crossrelease Feature) | v1 ☑✓ 4.15-rc4 | [LORE](https://lore.kernel.org/all/20171213104617.7lffucjhaa6xb7lp@gmail.com) |


## 10.3 Dependency Tracker
-------


接着 2022 年, 原交叉释放功能(Crossrelease Feature)的作者 Byungchul Park 提出了新的解决方案 Dept(依赖跟踪器), 它专注于等待和事件本身, 跟踪等待和事件, 并在任何事件永远无法到达时报告它.

1.  以正确的方式处理读锁.

2.  适用于任何等待和事件也就是交叉事件.

3.  报告多次后仍可继续工作.

4.  提供简单直观的 api.

5.  做了依赖检查器应该做的事情.

| 时间  | 作者 | 特性 | 描述 | 是否合入主线 | 链接 |
|:----:|:----:|:---:|:----:|:---------:|:----:|
| 2022/05/04 | Byungchul Park <byungchul.park@lge.com> | [DEPT(Dependency Tracker)](https://lore.kernel.org/all/1651652269-15342-1-git-send-email-byungchul.park@lge.com) | 一种死锁检测工具, 通过跟踪等待/事件而不是锁的获取顺序来检测死锁的可能性, 试图覆盖所有锁(spinlock, mutex, rwlock, seqlock, rwsem)以及同步机制(包括 wait_for_completion, PG_locked,  PG_writeback, swait/wakeup 等). | v6 ☐☑✓ | [RFC 00/14](https://lore.kernel.org/lkml/1643078204-12663-1-git-send-email-byungchul.park@lge.com)<br>*-*-*-*-*-*-*-* <br>[LORE v6,0/21](https://lore.kernel.org/all/1651652269-15342-1-git-send-email-byungchul.park@lge.com) |


# 11 深入理解并行编程
-------

Paul McKenney's parallel programming book, [LWN](https://lwn.net/Articles/421425), [PerfBook](https://mirrors.edge.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html), [cgit, perfbook](https://git.kernel.org/pub/scm/linux/kernel/git/paulmck/perfbook.git/)

计算机体系结构基础, [第 10 章 并行编程基础](https://foxsen.github.io/archbase/并行编程基础.html)



<br>

*   本作品/博文 ( [AderStep-紫夜阑珊-青伶巷草 Copyright ©2013-2017](http://blog.csdn.net/gatieme) ), 由 [成坚(gatieme)](http://blog.csdn.net/gatieme) 创作.

*   采用<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="知识共享许可协议" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">知识共享署名-非商业性使用-相同方式共享 4.0 国际许可协议</a>进行许可. 欢迎转载、使用、重新发布, 但务必保留文章署名[成坚gatieme](http://blog.csdn.net/gatieme) ( 包含链接: http://blog.csdn.net/gatieme ), 不得用于商业目的.

*   基于本文修改后的作品务必以相同的许可发布. 如有任何疑问, 请与我联系.

*   **转载请务必注明出处, 谢谢, 不胜感激**
<br>
