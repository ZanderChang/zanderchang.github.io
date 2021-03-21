---
title: AFL 及其相关拓展项目总结
date: 2019-04-29 14:44:42
categories:
- 漏洞挖掘
tags:
- 外文翻译
- 漏洞挖掘
- AFL
---

*原文标题 Zoo AFL (Apr 24, 2019)*

作者：[@d1g1](https://habr.com/en/users/d1g1/)

本文主要讨论的不是经典 AFL 本身，而是为其设计的工具以及对其本身做出的改进，我们认为这些改进可以显著提高模糊测试的质量。如果你想知道如何提高 AFL 效率以及如何更快地找到更多漏洞，请继续阅读！（持续更新中）

<!-- more -->

# AFL 是什么，它有什么用？

AFL是一个覆盖导向或基于反馈的模糊测试工具，关于这些概念的更多信息可以在参考这篇论文 [Fuzzing: Art, Science, and Engineering](https://arxiv.org/abs/1812.00140)。总结一下 AFL：

* 它改进可执行文件以了解它如何影响覆盖范围
* 变异输入数据以最大化覆盖范围
* 重复上一步以找到程序崩溃的位置
* 通过实践证明它非常有效
* 它易于使用

如果你不清楚 AFL 是什么的话，这里有一些资源帮助你上手和理解：

* [项目主页](http://lcamtuf.coredump.cx/afl/)
* [AFL 练习](https://github.com/ThalesIgnite/afl-training)
* [AFL Demo](https://gitlab.com/wolframroesler/afl-demo) — 使用 AFL fuzz C++ 程序
* [AFL 发现的漏洞](https://github.com/mrash/afl-cve) —（截至 2017）
* 在[这里](https://tunnelshade.in/blog/2018/01/afl-internals-compile-time-instrumentation/)你可以读到关于 AFL 在构建过程中添加到程序中的内容
* [一些](https://copyninja.info/blog/afl-and-network-programs.html)关于 fuzz 网络程序的技巧

在写这篇文章时 AFL 的最新版本是[2.52b](http://lcamtuf.coredump.cx/afl/)。随着时间的推移，一些分支正在被纳入到 AFL 主分支。接下来我们会列举几个有用的附件工具。

**关于 Rode0day 竞赛**

[Rode0day](https://rode0day.mit.edu/) 竞赛每月举办一次，参赛者要在花费更少时间的情况下在预先制作的语料库（无论是否获取源代码）中找到最多的漏洞。就其本质而言，Rode0day 是 AFL 不同分支之间的争斗。

一些AFL用户[指出](https://groups.google.com/forum/#!topic/afl-users/6NTPAkK7JEk)，AFL 的作者 Michal Zalewski 显然已经放弃该项目， 因为上次的改进日期还得追溯到 2017 年 11 月 5 日。这可能与他离开 Google 并开展一些新项目有关。 因此用户开始为最新的版本 2.52b 制作新的[补丁](https://github.com/vanhauser-thc/afl-patches))。

AFL 还有不同的变体和衍生物，允许模糊测试 Python，Go，Rust，OCaml，GCJ Java，内核系统调用，甚至整个虚拟机。

**AFL 对其它语言的支持**

* [python-afl](https://github.com/jwilk/python-afl) — Python
* [afl.rs](https://github.com/rust-fuzz/afl.rs) — Rust
* [afl-fuzz-js](https://github.com/connor4312/js-fuzz) — Javascript
* [java-afl](https://github.com/Barro/java-afl) — Java
* [kelinci](https://github.com/isstac/kelinci) — Java（[相关文章](https://www.modzero.ch/modlog/archives/2018/09/20/java_bugs_with_and_without_fuzzing/index.html)）
* [javan-warty-pig](https://github.com/cretz/javan-warty-pig) — JVM
* [afl-swift](https://github.com/Proteas/afl-swift) — Swift
* [ocamlopt-afl](https://github.com/kayceesrk/ocamlopt-afl) — OCaml
* [sharpfuzz](https://github.com/Metalnem/sharpfuzz) — .Net

# AFL 附属工具

本节我们收集了 AFL 的各种脚本和工具，并将它们分为几类：

## 崩溃处理

* [afl-utils](https://gitlab.com/rc0r/afl-utils) — 自动处理/分析崩溃和减少测试用例数量
* [afl-crash-analyzer](https://github.com/floyd-fuh/afl-crash-analyzer) — AFL 崩溃分析程序
* [fuzzer-utils](https://github.com/ThePatrickStar/fuzzer-utils) — 分析结果
* [atriage](https://github.com/Ayrx/atriage) — 简单的分类工具
* [afl-kit](https://github.com/kcwu/afl-kit) — Python 实现的 afl-cmin
* [AFLize](https://github.com/d33tah/aflize) — 生成适合 AFL 的 debian 包的构建
* [afl-fid](https://github.com/FoRTE-Research/afl-fid) — 处理输入数据

## 代码覆盖率

* [afl-cov](https://github.com/mrash/afl-cov) — 提供人性化的覆盖率数据
* [count-afl-calls](https://github.com/Barro/count-afl-calls) — 使用脚本计算二进制文件中的插桩块数，评估其占比
* [afl-sancov](https://github.com/bshastry/afl-sancov) — 类似于 afl-cov 但使用 clang的消毒机制（sanitizer）
* [covnavi](https://github.com/Cisco-Talos/covnavi) — 思科 Talos 开发的用于代码覆盖和分析的脚本
* [LAF LLVM Passes](https://gitlab.com/laf-intel/laf-llvm-pass/tree/master) — 类似于 AFL 的补丁集合，可以改进代码使其更容易找到分支

## 一些用于最小化测试用例的脚本

* [afl-pytmin](https://github.com/ilsani/afl-pytmin) — 一个 afl-tmin 的包装器，它试图通过使用多 CPU 内核来加速最小化测试用例的过程
* [afl-ddmin-mod](https://github.com/MarkusTeufelberger/afl-ddmin-mod) — 基于 ddmin 算法的 afl-tmin
* [halfempty](https://github.com/googleprojectzero/halfempty) — 来自 Tavis Ormandy 基于并行化的快速最小化测试用例的程序

## 分布式执行

* [disfuzz-afl](https://github.com/MartijnB/disfuzz-afl) — AFL 的分布式模糊测试
* [AFLDFF](https://github.com/quantumvm/AFLDFF) — AFL 分布式模糊测试框架
* [afl-launch](https://github.com/bnagy/afl-launch) — AFL 多实例执行
* [afl-mothership](https://github.com/afl-mothership/afl-mothership) — AWS 云上多同步 AFL 的管理和执行
* [afl-in-the-cloud](https://github.com/abhisek/afl-in-the-cloud) — 另一个在 AWS 中运行 AFL 的脚本
* [VU\_BSc\_project](https://github.com/clvang000/VU_BSc_project) — 使用 libFuzzer 和 AFL 对开源库进行模糊测试

最近有一篇文章 [Scaling AFL to a 256 thread machine](https://gamozolabs.github.io/fuzzing/2018/09/16/scaling_afl.html)。

## 部署，管理，监控，报告

* [afl-other-arch](https://github.com/shellphish/afl-other-arch) — 用于添加对各种非x86体系结构的支持
* [afl-trivia](https://github.com/bnagy/afl-trivia) — 简化 AFL 管理的脚本
* [afl-monitor](https://github.com/reflare/afl-monitor) — AFL 监控脚本
* [afl-manager](https://github.com/zx1340/afl-manager) — Python 实现的用于管理 multi-afl 的 Web 服务器
* [afl-tools](https://hub.docker.com/r/moflow/afl-tools/) — 带有 afl-latest, afl-dyninst 和 Triforce-afl 的 docker 镜像
* [afl-remote](https://github.com/block8437/afl-remote) — 远程管理 AFL 实例的 Web 服务器

## 包装器（wrapper）

* [phuzzer](https://github.com/angr/phuzzer) - 基于 Python 与 AFL 交互

# AFL 改进

AFL 对漏洞研究人员和模糊测试本身产生了巨大的影响，他们开始根据原始 AFL 进行改进。在不同的情况下，与原始AFL相比，每个改进都有其自身的优缺点。

几乎所有改进版本可以在 [hub.docker.com](https://hub.docker.com) 找到。

目的是什么？

* 提高速度并/或提高代码覆盖率
  * 算法
  * 环境
    * 操作系统
    * 硬件

* 无源码环境
  * 代码模拟
  * 代码插桩
    * 静态
    * 动态

## AFL 默认模式

在继续研究 AFL 的不同改进和分支之前，我们必须讨论两种重要的模式，这些模式在过去也有过改进但最终被合并。它们是 Syzygy 和 QEMU。

### [Syzygy](https://doar-e.github.io/blog/2017/08/05/binary-rewriting-with-syzygy/) 模式 — instrument.exe 中工作的模式

```
instrument.exe --mode=afl --input-image=test.exe --output-image=test.instr.exe
```

Syzygy 允许使用 AFL 静态重写 PE32 二进制文件，但需要符号和额外环境的才能识别 WinAFL 内核。

### QEMU 模式 — QEMU 下的工作方式见 [Internals of AFL fuzzer — QEMU Instrumentation](https://tunnelshade.in/blog/2018/02/afl-internals-qemu-instrumentation/)

使用 QEMU 实现对二进制文件的支持在版本 1.31b 中被添加到 AFL 处理流程上游。AFL QEMU 模式使用 QEMU tcg（一个微小的代码生成器）二进制转换引擎的二进制插桩功能。为此，AFL 有一个 QEMU 的构建脚本，它提取特定版本 QEMU（2.10.0）的源代码，将它们放到几个小的补丁上，并为指定架构构建。然后，创建名为 afl-qemu-trace 的文件，该文件实际上是 QEMU 用户模式模拟（仅可执行ELF文件）的文件。因此，AFL 可以使用不同体系架构（QEMU 支持）中 elf 二进制文件的反馈进行模糊测试。此外，还有许多 AFL 工具，比如从显示器上获取有关当前会话的信息，以及 afl-analyze 等高级内容。但同时 AFL 也会受到 QEMU 的局限， 比如如果工具链使用硬件 SoC 功能构建文件（该功能启动二进制并且 QEMU 不支持），则只要有特定指令或使用特定 MMIO，模糊测试就会中断。

[这里](https://abiondo.me/2018/09/21/improving-afl-qemu-mode/)是 QEMU 模式的另一个有趣的分支，其中使用TCG代码插桩和缓存，速度提高了3-4倍。

## 分支

AFL 的分支首先与经典 AFL 算法的变化和改进有关。

* [pe-afl](https://github.com/wmliang/pe-afl) — 对在 Windows 操作系统中没有源代码的 PE 文件进行模糊测试的改进。fuzzer 使用 IDA Pro 分析目标程序，并生成用于接下来静态插桩的信息。然后用 AFL 对插桩后的程序进行模糊测试。
* [afl-cygwin](https://github.com/arizvisa/afl-cygwin) — 尝试使用 Cygwin 将经典 AFL 移植到Windows。然而它有很多错误且很慢，作者已经放弃了开发
* [AFLFast](https://github.com/mboehme/aflfast)（使用 Power Schedule 改进 AFL 的调度算法和搜索策略）— 最早的 AFL 分支之一，它增加了启发式功能，实现在短时间内找到更多路径。它在目标 binutils 上获得了比 AFL 高 1-2 个数量级的发现，在24小时内发现了原始 AFL 未能发现的 4 个 CVE，达到平均 19 倍的效率提升。Team Codejitsu 在美国国防部和 DARPA 组织的 CGC（基于人工智能的自动化漏洞挖掘大赛）上，使用 AFLFast 获得了漏洞发现数目单项挑战的亚军。
* [FairFuzz](https://github.com/carolemieux/afl-rb) — AFL 的扩展，针对难抵达的分支（利用Branch Mask）
* [AFLGo](https://github.com/aflgo/aflgo) — AFL 的扩展，获取代码的某些部分而不是完整的程序覆盖，可用于测试补丁或新添加的代码片段
* [PerfFuzz](https://github.com/carolemieux/perffuzz) — AFL 的扩展，用于查找可能显着减慢程序速度的测试用例
* [Pythia](https://github.com/mboehme/pythia) — AFL 的扩展，预测找到新路径的难度
* [Angora](https://github.com/AngoraFuzzer/Angora) — 最新的 fuzzer 之一，用 Rust 语言编写，使用新的策略进行变异并增加覆盖范围。
* [Neuzz](https://github.com/Dongdongshe/neuzz) — 用神经网络进行模糊测试
* [UnTracer-AFL](https://github.com/FoRTE-Research/UnTracer-AFL) — 将 AFl 与 UnTracer 集成以进行有效跟踪
* [Qsym](https://github.com/sslab-gatech/qsym/) — 针对混合模糊测试的实用混合执行（concolic execution）引擎。从本质上讲，它是一个符号执行引擎（基本组件作为 intel pin 的插件来实现），它与 AFL 一起执行混合模糊测试。这是基于反馈的模糊测试演变的一个阶段，需要单独讨论。它的主要优点是可以相对快速地进行混合执行。这是由于本机执行命令而没有代码、快照和一些启发式的中间表示。它使用旧版本的Intel pin（由于 libz3 和其他 DBT 之间的支持问题），目前支持ELF x86 和 x86_64 架构
* [Superion](https://github.com/zhunki/Superion) — 灰盒模糊测试器，其优点是除了插桩程序外，它还使用 ANTLR 语法获取输入数据的规范，然后在此语法的帮助下执行变异
* [AFLSmart](https://github.com/aflsmart/aflsmart) — 另一个灰盒模糊测试器，它以 Peach 使用的格式获得输入数据的规范

有许多研究论文致力于实现 AFL 被改进的新方法和模糊测试技术。由于它们只有白皮书，所以我们甚至没有提到这些。如果你愿意，你可以搜索它们。例如最新的一些有 [CollAFL: Path Sensitive Fuzzing](http://chao.100871.net/papers/oakland18.pdf)、[EnFuzz](https://arxiv.org/pdf/1807.00182.pdf)、[Efficient approach to fuzzing interpreters](https://i.blackhat.com/asia-19/Fri-March-29/bh-asia-Dominiak-Efficient-Approach-to-Fuzzing-Interpreters-wp.pdf)、[ML](https://arxiv.org/pdf/1811.08973.pdf) 用于 AFL 等。

## 基于 QEMU 的改进

* [TriforceAFL](https://github.com/nccgroup/TriforceAFL) — AFL/QEMU对系统完全模拟来进行模糊测试。是由 nccgroup 提供的一个分支。它允许在 QEMU 模式下对整个操作系统进行模糊测试。它是通过一个特殊指令（aflCall (0f 24)）（在 QEMU x64 CPU 中添加）实现的。然而它不再受支持; 支持 AFL 的最后一个版本是2.06b
* [TriforceLinuxSyscallFuzzer](https://github.com/nccgroup/TriforceLinuxSyscallFuzzer) — Linux 系统调用的模糊测试
* [afl-qai](https://github.com/kanglictf/afl-qai) — 一个带有 QEMU 增强插桩（qai）的小型演示项目。

## 基于 KLEE 的改进

* [kleefl](https://github.com/julieeen/kleefl) — 通过符号执行生成测试用例（在大程序上非常慢）

## 基于 Angr 的改进

* [Driller](https://github.com/shellphish/driller) - 将 AFL 和 Angr 结合在一起，还被用于 CGC(Cyber Grand Challenge) 自动网络攻防竞赛上

## 基于 Unicorn 的改进

* [afl-unicorn](https://github.com/Battelle/afl-unicorn) — 允许通过在 [Unicorn Engine](https://www.unicorn-engine.org/) 上模拟代码片段进行模糊测试

我们在实践中成功使用了 AFL 的这种改进，由于目标是在 SOC 上执行的某个 RTOS（real-time operating system，实时操作系统）的代码区域，因此我们无法使用QEMU模式。在我们没有源代码的情况下（我们无法构建用于解析器分析的独立二进制文件）并且程序不直接获取输入数据（例如，数据被加密或是如在 CGC 二进制文件中的信号样本），然后我们可以逆向并找到所谓的位置函数，函数中中数据以便于模糊测试器使用的格式进行处理。这是 AFL 最普遍/通用的修改，即它允许模糊测试任何东西。它独立于架构、源、输入数据格式和二进制格式（例如裸机，只是来自控制器内存的代码片段）。研究人员首先检查这个二进制文件并编写一个模糊测试器，它在解析过程的输入端模拟状态。显然与 AFL 不同，这需要对二进制进行一定的检查。对于裸机固件，如 Wi-FI 或基带，我们需要记住一些缺点：

* 我们必须本地处理控制和的检查
* 模糊测试器的状态是保存在内存转储中的内存状态，这可以防止模糊测试器进入某些路径
* 动态内存调用没有消毒，但它可以手动实现，它将取决于 RTOS（必须进行研究）
* 未模拟任务内 RTOS 交互，这也可能阻止寻找某些路径

使用这种修改的例子有：

[afl-unicorn: Fuzzing Arbitrary Binary Code](https://hackernoon.com/afl-unicorn-fuzzing-arbitrary-binary-code-563ca28936bf)

[afl-unicorn: Part 2 — Fuzzing the 'Unfuzzable'](https://hackernoon.com/afl-unicorn-part-2-fuzzing-the-unfuzzable-bea8de3540a5)

[unicorefuzz](https://github.com/fgsect/unicorefuzz) 使用 afl-unicorn 来 fuzz 内核

在我们继续基于动态二进制检测（DBI）框架进行修改之前，不要忘记这些框架的最快速度由 DynamoRIO，Dynlnst 以及 PIN 实现。

## 基于 PIN 的改进

* [aflpin](https://github.com/mothran/aflpin) — 采用 Intel PIN 插桩的 AFL
* [afl\_pin\_mode](https://github.com/spinpx/afl_pin_mode) — 另一个采用 Intel PIN 插桩的 AFL
* [afl-pin](https://github.com/vanhauser-thc/afl-pin) — 采用 PINtool 的 AFL
* [NaFl](https://github.com/carlosgprado/NaFl) — AFL 模糊测试器的克隆（基本核心）
* [PinAFL](https://github.com/houcy/PinAFL) — 该工具的作者试图将 AFL 移植到 Windows，以便对已编译的二进制文件进行模糊测试。该项目好像是为了好玩而在一夜之间完成的且未进一步发展。存储库里没有源文件，只有已编译的二进制文件和启动指令。我们不知道它基于哪个版本的 AFL，它只支持 32 位应用程序。

正如我们所看到的，AFL 有许多不同的修改，但它们在现实生活中并不是非常有用。

## 基于 Dyninst 的改进

[afl-dyninst](https://github.com/talos-vulndev/afl-dyninst) — AFL + Dyninst == AFL 黑盒模糊测试。这个版本的特点是首先使用 Dyninst 对一个研究过的程序（没有源代码）进行静态插桩（静态二进制插桩，静态二进制重写），然后使用经典 AFL 进行模糊测试，AFL 会认为程序是用 afl-gcc/afl-g++/afl-as 构建的。因此，它在没有源代码的情况下以非常好的生产率工作：与本地编译相比，经典 AFL 是 0.25 倍的速度。与QEMU相比，它具有显着优势：它允许动态链接库的插桩，而 QEMU 只能插桩与库静态链接的基本可执行文件。然而现在它只支持 Linux，如果增加对于 Windows 支持，则需要对 Dyninst 本身进行更改，这个工作[正在进行](https://github.com/dyninst/dyninst/issues/120)中。

还有另一个[分支](https://github.com/vanhauser-thc/afl-dyninst)具有提升的速度和某些功能（AARCH64 和 PPC 架构的支持）。

## 基于 DynamoRIO 的改进

* [drAFL](https://github.com/mxmssh/drAFL) — AFl + DynamoRIO – 在 Linux 上没有源代码的模糊测试
* [afl-dr](https://github.com/atrosinenko/afl-dr) — 基于 DynamoRIO 的另一种实现，在 [Habr](https://habrahabr.ru/post/332076/) 上有很好的描述
* [afl-dynamorio](https://github.com/vanhauser-thc/afl-dynamorio) — 来自 vanhauser-thc 的修改。 据他所说：“当正常的 afl-dyninst 能够使二进制崩溃并且 QEMU 模式 -Q 无法执行时，用 DynamoRIO 运行 AFL。”它支持 ARM 和 AARCH64。DynamoRIO 比 QEMU 慢大约 10 倍，比 dyninst 慢 25 倍，但比 Pintool 快 10 倍。
* [WinAFL](https://github.com/ivanfratric/winafl) — 最著名的 AFL分支。（DynamoRIO，也是 syzygy 模式）。这个修改的出现只是时间问题，因为许多人想在 Windows 上尝试 AFL 并将其应用于没有源代码的应用程序。目前，这个工具正在被积极改进，尽管 AFL 代码库（撰写本文时为 2.43b）相对过时，但它都有助于发现多个漏洞（CVE-2016-7212，CVE-2017-0073，CVE- 2017-0190，CVE-2017-11816）。Google Zero Project 团队和 MSRC 漏洞与解决团队的专家正在参与此项目，因此我们希望它能够得到进一步的发展。开发人员使用动态插桩（基于DynamoRIO）而不是编译时插桩，这显着减慢了被分析软件的执行速度，但产生的开销（加倍）与二进制模式下的经典 AFL 相当。他们还解决了快速启动过程的问题，称其为持续模糊测试模式；他们选择对函数进行模糊测试（通过文件内部的偏移或导出表中存在的函数名称）并对其进行插桩，以便可以在循环中调用它，从而启动多个输入数据样本而无需重新启动程序。最近发表的一篇[文章](https://research.checkpoint.com/50-adobe-cves-in-50-days/)描述了作者如何使用 WinAFL 在约 50 天内发现约 50 个漏洞。发布之后很快 WinAFL 就支持 Intel PT（Processor Tracing，处理器跟踪）模式，在这里可以找到[细节信息](https://github.com/googleprojectzero/winafl/blob/master/readme_pt.md)。

专业读者可能注意到这些改进使用了除了Frida之外的各种流行的插桩框架。 唯一提到 Frida 与 AFL 的结合使用是在 [Chizpurfle: A Gray-Box Android Fuzzer for Vendor Service Customizations](http://wpage.unina.it/roberto.natella/papers/natella_androidfuzzing_issre2017.pdf) 中找到的。 带 Frida 的 AFL 版本非常有用，因为 Frida 支持多种 RISC 架构。

许多研究者也期待由 Capstone，Unicorn 和 Keystone 的创建者发布的 DBI Scopio 框架。基于这个框架，作者已经创建了一个模糊测试器（Darko），据他们说能够成功地使用它来对嵌入式设备进行模糊测试。有关这方面的更多信息，请参阅 [Digging Deep: Finding 0days in Embedded Systems with Code Coverage Guided Fuzzing](https://conference.hitb.org/hitbsecconf2018pek/materials/D2T1%20-%20Finding%200days%20in%20Embedded%20Systems%20with%20Code%20Coverage%20Guided%20Fuzzing%20-%20Dr%20Quynh%20and%20Kai%20Jern%20Lau.pdf)。

## 基于处理器硬件特征的改进

关于支持处理器硬件功能的 AFL 修改方面，首先，它允许对内核代码进行模糊测试；其次，它允许在没有源代码的情况下更快地对应用程序进行模糊测试。

当然，谈到处理器硬件功能，我们最感兴趣的是 [Intel PT](https://software.intel.com/en-us/node/721535)。从第 6 代处理器开始（大约自2015年起）支持该功能。 因此，为了能够使用下面列出的模糊测试器，我们需要一个支持 Intel PT 的处理器。

* [WinAFL-IntelPT](https://github.com/intelpt/winafl-intelpt) — 使用 Intel PT 而不是 DynamoRIO 的第三方 WinAFL。
* [ptfuzzer](https://github.com/hunter-ht-2018/ptfuzzer) - 使用 Intel PT 对二进制进行黑盒测试。
* [kAFL](https://github.com/RUB-SysSec/kAFL) — 一个旨在解决独立于操作系统的模糊测试内核时的覆盖率导向问题的学术项目。 它通过使用管理程序（hypervisor）和 Intel PT 来解决该问题。有关它的更多信息可以在白皮书 [kAFL: Hardware-Assisted Feedback Fuzzing for OS Kernels](https://www.usenix.org/system/files/conference/usenixsecurity17/sec17-schumilo.pdf) 中找到。

# AFL 具体应用

* [GUSTAVE](https://github.com/airbus-seclab/gustave) - 一个嵌入式操作系统内核的 Fuzzer，基于 QEMU 和 AFL。
* [android-afl](https://github.com/ele7enxxh/android-afl) - 在 Android 上使用 AFL。[在Linux上使用AFL对Stagefright进行模糊测试](https://paper.seebug.org/245/)
* 使用 AFL 对嵌入式系统的 TEE 进行模糊测试。[PDF](https://nullcon.net/website/archives/pdf/bangalore-2019/fuzzing-embedded-(trusted)-operating-systems%20using-AFL.pdf) [YOUTUBE](https://youtu.be/AZhxZlwZ160)
* [Janus](https://github.com/sslab-gatech/janus) - 基于 AFL 和 Syzkaller 对 linux 的文件系统进行模糊测试， 支持 8 种文件系统 ext4、XFS、btrfs、F2FS、GFS2、HFS+、ReiserFS 和 vFAT；以及 34 个文件操作的系统调用。
* 利用 AFL 模糊测试 [statzone](https://github.com/fcambus/statzone)。[地址](https://www.cambus.net/fuzzing-dns-zone-parsers/)
* 利用 AFL 模糊测试 Linux 内核。[地址](https://blog.cloudflare.com/a-gentle-introduction-to-linux-kernel-fuzzing/)
* [FIRM-AFL](https://github.com/zyw-200/FirmAFL) - 基于 AFL 实现的用于 IoT 固件灰盒模糊测试的工具。[PDF](https://www.cs.ucr.edu/~heng/pubs/FirmAFL.pdf)
* 对 OP-TEE 的系统调用接口进行模糊测试。[地址](https://github.com/Riscure/optee_fuzzer)
* 利用 WinAFL 对二进制程序进行模糊测试，利用内存访问热图来精简输入种子大小。[地址](https://sec-consult.com/en/blog/2017/09/hack-the-hacker-fuzzing-mimikatz-on-windows-with-winafl-heatmaps-0day/)
* [FuzzFactory](https://github.com/rohanpadhye/FuzzFactory) - AFL 扩展，使用 Waypoints 针对特定领域进行通用的以覆盖率位导向的模糊测试。[YOUTUBE](https://www.youtube.com/watch?v=34A1jtgfgsU) [Paper](https://dl.acm.org/citation.cfm?doid=3366395.3360600)
* [PowerFL](https://www.petergoodman.me/docs/qpss-2019-slides.pdf) - PowerFL = AFL + QEMU + VxWorks，使用 AFL 和 QEMU 对 PowerPC 和 Intel i386 VxWorks 进行 fuzz。
* [frida-js-afl-instr](https://github.com/andreafioraldi/frida-js-afl-instr) - 使用 [AFL++](https://github.com/vanhauser-thc/AFLplusplus) 和 Frida 进行 Linux 内存中的模糊测试。
* [libprotobuf-mutator_fuzzing_learning](https://github.com/bruce30262/libprotobuf-mutator_fuzzing_learning) - 将 AFL++ 和 LibFuzzer 结合使用（通过protobuf）。
* [AFLNet](https://github.com/aflnet/aflnet) - 面向网络协议的**状态**灰盒 Fuzzer，以 Server 和 Client 之间正常的协议通信数据（tcpdump 监听端口并用 Wireshark 做进一步处理）作为语料，不需要知道协议具体数据格式和语法。
* [fuzz实战之afl](https://www.secpulse.com/archives/71903.html) - 利用 AFL fuzz 网络程序 libmodbus 库，利用 [preeny](https://github.com/zardus/preeny) `LD_PRELOAD` 把从 socket 获取数据，转变为从 stdin 获取数据。
* [Fuzzing sockets, part 1: FTP servers](https://securitylab.github.com/research/fuzzing-sockets-FTP) - 利用 AFL++ fuzz FTP 服务器（命令），作者详细介绍如何改写 FTP 服务器源码来支持 fuzz（工程量大，修改约 1500 行代码），并介绍了`CVE-2020-9273`，`CVE-2020-9365`和`CVE-2020-9274`的原理。

# AFL 原理与实现

* [AFL(American Fuzzy Lop)实现细节与文件变异](https://paper.seebug.org/496/)
* [AFL afl_fuzz.c 详细分析](https://bbs.pediy.com/thread-254705.htm)
* [AFL 漏洞挖掘技术漫谈（一）：用 AFL 开始你的第一次 Fuzzing](https://paper.seebug.org/841/)
* [AFL 漏洞挖掘技术漫谈（二）：Fuzz 结果分析和代码覆盖率](https://paper.seebug.org/842/)：各种辅助分析的工具
  * `afl-cmin -i input_dir -o output_dir -- /path/to/tested/program [params] @@`
* [AFL学习笔记（上）](https://bbs.pediy.com/thread-257399.htm)：持续更新中
* [漏洞挖掘技术之 AFL 项目分析](https://bbs.pediy.com/thread-249912.htm)

# Fuzz 相关文章

* [Fuzzing战争: 从刀剑弓斧到星球大战](https://mp.weixin.qq.com/s/nREiT1Uj25igCMWu1kta9g)

# 总结

如你所见，研究人员仍在积极地改进 AFL，而且 AFL 还存在进行实验和创新的空间，我们也可以创建一个我们自己的实用有趣的 AFL。

感谢阅读，祝大家 Fuzz 顺利。

[原文链接](https://habr.com/ru/company/dsec/blog/449134/)