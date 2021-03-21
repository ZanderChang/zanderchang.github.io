---
title: VBA Stomping 简介
date: 2019-04-30 10:17:09
categories:
- 安全技术
tags:
- 外文翻译
- VBA
- 安全技术
---

*原文标题 VBA Stomping — Advanced Maldoc Techniques (Oct 6, 2018)*

作者：Kirk Sayre [@bigmacjpg](https://twitter.com/bigmacjpg)，Harold Ogden [@haroldogden](https://twitter.com/haroldogden) 和 Carrie Roberts [@OrOneEqualsOne](https://twitter.com/OrOneEqualsOne)

VBA Stomping 是一种可以绕过反病毒检测恶意文档生成技术，它最初由 Vesselin Bontchev 博士引起我们的注意（见[此处](https://github.com/bontchev/pcodedmp)）。VBA stomping 是指销毁 Microsoft Office 文档中的 VBA 源代码，只留下文档文件中称为 p-code 的宏代码的编译版本。在这种情况下，仅基于VBA源代码的恶意文档检测会失败。在这篇博文中，我们将详细演示 VBA stomping 并介绍一些其他技术。

<!-- more -->

## VBA Stomping

首先，我们将在一个简单的非恶意宏上演示 VBA Stomping。此文档在打开时显示带有文本 ABC 的消息框。VBA 源代码和生成的消息框如下所示。

![](/img/vba-stomping/1_P_mM0Ffo9kzzhtMM1O7oCg.png)

现在我们修改上面显示的VBA源代码，同时保持中间 p-code 不变。我们将集中讨论当前的 .docx/.xlsx/.docm/.xlsm（Office 2007 以上）格式。但是，此处讨论的技术可以轻松应用于较旧的 .doc/.xls格式。在 Office 2007 以上文件中，VBA 源代码和 p-code 通常位于名为 vbaProject.bin 的文件中。请注意这是默认文件名，可以被重命名。如果要手动修改此文件，我们需要首先解压 .docm/.xlsm 文件，然后在十六进制编辑器中打开 vbaProject.bin 文件。在此示例中，我们将 ABC 改为 XYZ，但仅限于存储 VBA 源代码的位置，而不是 p-code 部分。VBA 源代码以压缩形式存储，即下图中不可打印或奇怪的字符。

![](/img/vba-stomping/1_KGc1XphbyNT5Dws5W8lOPQ.png)

![](/img/vba-stomping/1_iHWUB-LOeGuw_8K02Gixpg.png)

现在我们已经手动编辑了 VBA 源代码将 ABC 更改为 XYZ，我们打开文档并在单击“启用内容”按钮之前检查 VBA 源代码。

![](/img/vba-stomping/1_jqYto7TMBR3MUZPqzdhogw.png)

我们打开文档但未启用宏。在代码编辑器中检查宏，代码显示宏将显示带有文本 XYZ 的消息框，但事实并非如此。实际上，只要启用了内容，就会显示一个 ABC 的消息框，并且代码编辑器中的源代码会更新以匹配！

![](/img/vba-stomping/1_VE19x2i45U7r6FXDxov1UA.png)

源代码表示 XYZ 将显示在消息框中，但显示 ABC 。到底是怎么回事？

正如 Bontchev 博士所解释的那样，只要 p-code 与系统上的当前 VBA 版本兼容，文档实际执行的是存储的 p-code。此外，宏编辑器中显示的内容（一旦启用内容）并不是解压的 VBA 源代码，而是反编译的 p-code。

如果我们在不同版本的 Word（使用不同的 VBA 版本）中打开文档，则 p-code 将不可重用。这将强制将 VBA 源代码解压缩并重新编译为 p-code，从而在消息框中显示 XYZ。所以现在我们有一个文档，在一个版本的Office上显示 ABC，但在另一个版本上显示 XYZ。

请注意，这很重要。**使用 VBA Stomping 技术的恶意文档只能使用用于创建文档时相同的 VBA 版本执行。** 我们可以通过在恶意文档生成之前对目标进行侦察来确定要使用的适当 Office 版本适当 Office 版本；或者通过生成具有多个 Office 版本的恶意文档并将其喷射到目标上来解决此限制。

从防守角度看 VBA stomping 是什么效果呢？通常恶意文档检测仅基于 VBA 源代码。甚至许多可用于分析文档的工具都无法识别 VBA 源和 p-code 之间的差异。由 Philippe Lagadec 编写的 [olevba 脚本](https://github.com/decalage2/oletools/wiki/olevba)对篡改文档的分析仅显示了解压缩的源代码，且缺少 p-code 细节。

![](/img/vba-stomping/1_TIq1-TyPn5gxopcr88-U9A.png)

除非使用 p-code dumper 插件运行，否则 Didier Stevens 的 Python 脚本 [oledump.py](https://blog.didierstevens.com/programs/oledump-py/) 也会给出类似的结果。在使用该插件的情况下，输出给出了一些指示，即 p-code 将使消息框显示 ABC，如下图所示。

![](/img/vba-stomping/1_ZmBHbwFxuqbZB9JZzN4yrw.png)

Bontchev 博士发布了一个名为 [pcodedmp.py](https://github.com/bontchev/pcodedmp) 的 Python 脚本用于显示 p-code，如下图所示。输出可以显示操作名称而不是上面的操作码。

![](/img/vba-stomping/1_1jU7iSKRyTb93EK5WfIZ3Q.png)

我们可以通过用零或随机字节覆盖 VBA 源代码来完全擦除（stomp），而不仅仅是修改它。以下截图显示了如何在十六进制编辑器中手动执行此操作。

![](/img/vba-stomping/1_xCsSmTV9k0qFMjUWfQDWuQ.png)

![](/img/vba-stomping/1_XLiGHjipu6lk7JU8rNxzXw.png)

在这种情况下，VBA 编辑器根本不显示宏代码，但启用内容后仍然显示带有文本 ABC 的消息框。现在，我们在此文件上重新运行 olevba，结果不显示任何VBA源代码，并显示“未找到可疑关键字或 IOC”，而不是之前突出显示我们在使用可疑关键字（AutoOpen）。

![](/img/vba-stomping/1_3t0jKe01yE0ydhE2pFXvxA.png)

# 这个问题有多严重?

我们现在将更深入地了解 Office 如何使用压缩的 VBA 源代码和 p-code。稍后当我们讨论如何滥用 Office 功能来轻松修改恶意文档以欺骗许多 AV 扫描时，这将会发挥作用。

Office 文档包含两个用于提取 VBA 的位置，即压缩的 VBA 源代码和 p-code。下表详细说明了在以下情况下使用哪个 VBA 数据源：

![](/img/vba-stomping/1_Eh5SA794dEMFOlkqR8ThDQ.png)

从该表中我们可以看到，如果我们有有效的 p-code，则在启用宏时它会忽略压缩的 VBA 源代码，并通过运行 p-code 来执行所有宏功能。作为攻击者，这告诉我们，我们可以完全破坏或修改压缩的 VBA 源代码，并且仍然让我们的恶意文档执行其指定的任务。

# 现实案例

上述表明这适用于简单示例，但这真的是威胁吗？让我们通过使用已知的恶意文档将其提升到新的水平。在这种情况下，我们将从最近的 [Emotet](https://www.cyber.nj.gov/threat-profiles/trojan-variants/emotet) 恶意文档开始，它目前在 Virus total 上被 36/59 的供应商检测为恶意。

![](/img/vba-stomping/1_vhePAlQl6xO7TLdXXW1mCg.png)

如果我们使用这个文档并且如上所述方法简单地处理 VBA 源代码，那么检测率将降至 58 种防病毒解决方案中的 7 种！

![](/img/vba-stomping/1_15XLHrfFRP4PcFoTSX1GCA.png)

如您所见，这给网络防御者带来了严重的问题。即使手动分析此类文档也可能存在问题。此时显而易见的问题是*可以通过工具自动执行 VBA stomping 还是我们必须手动编辑 Office 文档？* 这个问题的答案是肯定的。我们开发了一个 POC 工具来自动在任何 Office 文档中处理压缩的 VBA 源代码（它被用于 VBA stomping 示例的 Emotet 文档，我们不会发布此实用程序）。考虑到自动化 VBA stomping 十分容易，这是一个真正的威胁。

# 变得更糟

考虑恶意文档被设计为在执行恶意有效载荷之后立即关闭 MS Word 的情况。与 VBA stomping 结合使用时，没有脱机的 VBA 源代码提取工具将显示 VBA 源代码，并且在启用宏之前，VBA 源代码甚至不会显示在 Office 宏编辑器窗口中。但是启用宏会导致 Word 立即退出而不提供查看反编译源的机会。虽然我们已经证明“类似汇编”的 p-code 是可提取的，但是 p-code 很难解释，并且无法在 VBA 调试器中被分析。此外 p-code 仅在特定的VBA版本上运行。出于这些原因，访问 VBA 源代码对分析人员来说是一个很大的好处。但是如何才能为这样的文件做到这一点呢？

经过一些实验，我们找到了一个简单的解决方案，可以在宏启用后立即停止 MS Word 执行任何方法。请注意，此解决方案适用于 Office 2007 以上的文档，我们研究解决较旧的文档格式。Office 2007 以上的解决方案是从 .docm 文件的 word 目录中删除 vbaData.xml 文件，如下所示（这可以通过 7-Zip 程序完成，而无需手动解压缩和重新压缩文档）。

![](/img/vba-stomping/1_j953lLi4vkji5nySs3YPeg.png)

现在我们可以打开文档并启用内容，从而在 VBA 代码编辑器中显示反编译的 p-code，但不执行代码。如果恶意软件作者已采取措施阻止分析人员访问 VBA 源代码，则这种方法可以节省时间。

删除 vbaData.xml 的一个有趣的副作用是它会导致 MS Word 在宏对话框中错误地列出没有宏（参见下图）。

![](/img/vba-stomping/1_KIhYa2asfDGRAIWXERhXug.png)

# 检测

我们编写了一个开源工具，用于检测名为 VBA Seismograph 的 Office 文档中的 VBA stomping。此工具已在 Ubuntu 16.04 下测试，它检测声明的函数/变量名称、字符串文字以及出现在已编译的 p-code 和 Office 文档的 VBA 源代码中的注释行之间的差异。该工具可在[这里](https://github.com/kirk-sayre-work/VBASeismograph)找到。

# 总结

在这篇博客文章中，我们演示了如何修改或销毁 Office 文档中压缩的 VBA 源代码，来绕过 AV 扫描，并使恶意文档的手动分析更加困难。对此技术的防御涉及检测和标记具有有效 p-code 但无效或缺少 VBA 源代码的文档，或更一般地，检查压缩 VBA 源代码与反编译 p-code 之间的差异。目前，我们还没有发现任何进行此项检查的商用 AV 解决方案，因此这是 AV 扫描解决方案可以改进的领域。

# 参考资料
* https://vbastomp.com/

[原文地址](https://medium.com/walmartlabs/vba-stomping-advanced-maldoc-techniques-612c484ab278)