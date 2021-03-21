---
title: Hypervisor From Scratch
mathjax: false
date: 2020-04-16 11:31:06
categories:
- Hypervisor
tags:
- Hypervisor
---

Hypervisor From Scratch 系列文章学习笔记

[实验代码](https://github.com/SinaKarvandi/Hypervisor-From-Scratch/)

<!-- more -->

# [Part 1: 基本概念和环境搭建](https://rayanfam.com/topics/hypervisor-from-scratch-part-1/)

||||
|-|-|-|
|Intel|VT-x|Virtual Machine eXtension (VMX)|
|AMD|AMD-V|Secure Virtual Machine (SVM)|

- [x64 Inline Assembly in Windows Driver Kit](https://rayanfam.com/topics/inline-assembly-in-x64/)
- [Writing a Windows Kernel Driver](https://resources.infosecinstitute.com/writing-a-windows-kernel-driver/)

```
[HKEY_LOCAL_MACHINE/SYSTEM/CurrentControlSet/Control/Session Manager/Debug Print Filter] 
DEFAULT=dword:0000000f
```

|关键概念||
|-|-|
|Virtual Machine Monitor (VMM)|VMM acts as a host and has full control of the processor(s) and other platform hardware. A VMM is able to retain selective control of processor resources, physical memory, interrupt management, and I/O.|
|Guest Software|Each virtual machine (VM) is a guest software environment.|
|VMX Root Operation and VMX Non-root Operation|A VMM will run in VMX root operation and guest software will run in VMX non-root operation|
|VMX transitions|Transitions between VMX root operation and VMX non-root operation.|
|VM entries|Transitions into VMX non-root operation.|
|Extended Page Table (EPT)|A modern mechanism which uses a second layer for converting the guest physical address to host physical address.|
|VM exits|Transitions from VMX non-root operation to VMX root operation.|
|Virtual machine control structure (VMCS)| a data structure in memory that exists exactly once per VM (or more precisely one per each VCPU [Virtual CPU]), while it is managed by the VMM. With every change of the execution context between different VMs, the VMCS is restored for the current VM, defining the state of the VM’s virtual processor and VMM control Guest software using VMCS.|

VMCS由6部分组成: [VMCS结构图](https://rayanfam.com/wp-content/uploads/sites/2/2018/08/VMCS.pdf)

|||
|-|-|
|Guest-state area|Processor state saved into the guest state area on VM exits and loaded on VM entries.|
|Host-state area|Processor state loaded from the host state area on VM exits.|
|VM-execution control fields|Fields controlling processor operation in VMX non-root operation.|
|VM-exit control fields|Fields that control VM exits.|
|VM-entry control fields|Fields that control VM entries.|
|VM-exit information fields|Read-only fields to receive information on VM exits describing the cause and the nature of the VM exit.|

|VMX指令||
|-|-|
|INVEPT|Invalidate Translations Derived from EPT|
|INVVPID|Invalidate Translations Based on VPID|
|VMCALL|Call to VM Monitor|
|VMCLEAR|Clear Virtual-Machine Control Structure|
|VMFUNC|Invoke VM function|
|VMLAUNCH|Launch Virtual Machine|
|VMRESUME|Resume Virtual Machine|
|VMPTRLD|Load Pointer to Virtual-Machine Control Structure|
|VMPTRST|Store Pointer to Virtual-Machine Control Structure|
|VMREAD|Read Field from Virtual-Machine Control Structure|
|VMWRITE|Write Field to Virtual-Machine Control Structure|
|VMXOFF|Leave VMX Operation|
|VMXON|Enter VMX Operation|

![VMM生命周期](https://rayanfam.com/wp-content/uploads/sites/2/2018/08/vmm-life-cycle.png)

# [Part 2: 开启VMX](https://rayanfam.com/topics/hypervisor-from-scratch-part-2/)

- 关闭驱动签名验证 `bcdedit.exe /set nointegritychecks on`
- 打开测试模式 `bcdedit /set testsigning on`
- 一边按住`Shift`一边重启，选择`7`关闭签名验证（仅本次有效）

- IRP Major Functions
  - IRP_MJ_CREATE
  - IRP_MJ_DEVICE_CONTROL
- IRP Minor Functions (mainly used for PnP manager to notify for a special event)
- Fast I/O
  - 先、快于IRP

- 检查Hypervisor支持
  - `EAX = 0` `CPUID`检查字符串`GenuineIntel`，判断是否为Intel处理器
  - `CPUID.1:ECX.VMX[bit 5] = 1`，判断是否支持VMX
- 开启VMX
  - `CR4.VMXE[bit 13] = 1`

- MyHypervisorDriver调用`IoCreateDevice`函数创建设备并为其`IRP_MJ_CREATE`指定`DrvCreate`函数，`DrvCreate`函数开启VMX，并调用`DbgPrint`函数输出，用Dbgview中查看。
- MyHypervisorApp利用`__asm`检查CPU是否支持VMX，如果支持则调用`CreateFile`函数打开设备，触发驱动的`DrvCreate`函数，开启VMX。

# [Part 3: 建立第一个虚拟机](https://rayanfam.com/topics/hypervisor-from-scratch-part-3/)

- IOCTL(32bit)
  - METHOD_BUFFERED
  - METHOD_NIETHER
  - METHOD_IN_DIRECT
  - METHOD_OUT_DIRECT

|内核函数/宏||
|-|-|
|KeQueryActiveProcessorCount|获取逻辑处理器数|
|KeSetSystemAffinityThread|将当前线程分配到某个逻辑处理器上|
|KeRevertToUserAffinityThread|恢复线程运行的处理器|
|MmGetPhysicalAddress|虚拟地址->物理地址|
|MmGetVirtualForPhysical|物理地址->虚拟地址|
|MmAllocateContiguousMemory|申请对齐的连续内存|
|MmFreeContiguousMemory|释放上述内存|
|RtlSecureZeroMemory|初始化内存为0|
|__readmsr|读取IA32_FEATURE_CONTROL_MSR，用于检查VMX支持、保存RevisionId|
|__vmx_on||
|__vmx_vmptrld||
|__vmx_off||
|PAGED_CODE|确保调用线程运行在一个允许分页的足够低IRQL级别|

![VMX生命周期](https://rayanfam.com/wp-content/uploads/sites/2/2018/09/VMXLifecycle.png)

![VMCS结构](https://rayanfam.com/wp-content/uploads/sites/2/2018/09/Init-VMCS.png)

- MyHypervisorDriver调用`IoCreateDevice`函数创建设备并为其`IRP_MJ_CREATE`指定`DrvCreate`函数，为`IRP_MJ_CLOSE`指定`DrvClose`函数。`DrvCreate`函数初始化VMX，为每个逻辑处理器开启`VMX`并为`VMXON`和`VMCS`分配内存，`DrvClose`则相反。
- MyHypervisorApp调用`CreateFile`函数打开设备，触发驱动的`DrvCreate`函数，开启VMX，并为每个逻辑处理器的`VMXON`和`VMCS`分配内存；调用`CloseHandle`关闭设备，触发驱动的`DrvClose`函数，关闭`VMX`并释放内存。

- 用户态和VMM驱动交互
  - MyHypervisorDriver为`IRP_MJ_DEVICE_CONTROL`指定`DrvIOCTLDispatcher`函数，与MyHypervisorApp通过`IOCTL`通信（内存读取）。MyHypervisorApp调用`DeviceIoControl`函数并指定`IoControlCode`触发`DrvIOCTLDispatcher`。

# [Part 4: 使用EPT进行地址转换](https://rayanfam.com/topics/hypervisor-from-scratch-part-4/)

- [Turning the Pages: Introduction to Memory Paging on Windows 10 x64](https://connormcgarr.github.io/paging/)
  - Windbg
    - `!pte`
      - PXE = PML4E
      - PPE = PDPE
    - `!vtop` 将虚拟地址转换为物理地址
- [Introduction to IA-32e hardware paging](https://www.triplefault.io/2017/07/introduction-to-ia-32e-hardware-paging.html)
  - Intel 64位分页机制
  - PG flag: CR0[bit 31]: 开启分页
  - Physical Address Extension (PAE): CR4[bit 5]: 未设置则使用32bit分页
  - Long Mode Enable (LME): Extended Feature Enable Register (IA32_EFER MSR)[bit 8]: 未设置则使用PAE 36bit分页，否则使用64bit的4层分页机制
    - Page Frame Number (PFN): the next paging structure in the hierarchy (0x1000 4 KB)
    - 4096bytes 512 entries(PFN)
    - `CR3`保存第一个页结构的物理地址
    - 虚拟地址
      - [bits 63-48] 保留
      - [bits 47-39] a PML4 table (located in CR3) offset
      - [bits 38-30] a Page Directory Pointer Table (PDPT) offset
      - [bits 29-21] a Page Directory (PD) offset
      - [bits 20-12] a Page Table (PT) offset
      - [bits 11-00] 物理页中的偏移
  - 使用Windbg具体分析

- Second Level Address Translation (SLAT) or nested paging
  - an extended layer in the paging mechanism
  - hardware-based virtualization virtual addresses -> physical memory
  - 实现
    - AMD: Rapid Virtualization Indexing (RVI)/Nested Page Tables (NPT)
    - Intel: Extended Page Table (EPT)
    - ARM: Stage-2 page-tables
  - 两种方法
    - Shadow Page Tables (Software-assisted paging)
    - Extended Page Tables (Hardware-assisted paging)
      - Page table maintained by guest OS generate the guest-physical address.
      - Page table maintained by VMM map guest physical address to host physical address.

![EPT地址转换](https://rayanfam.com/wp-content/uploads/sites/2/2018/08/ept-translation.png)

![EPTP结构](https://rayanfam.com/wp-content/uploads/sites/2/2018/10/EPTP.png)

![EPT整体结构](https://rayanfam.com/wp-content/uploads/sites/2/2018/10/EPT-Layout.png)

MyHypervisorDriver添加了EPT的初始化代码，为64bit的4层地址转换的每个结构表申请内存空间。

# [Part 5: 建立VMCS并在虚拟机中执行代码](https://rayanfam.com/topics/hypervisor-from-scratch-part-5/)

|内核函数||
|-|-|
|__vmx_vmptrst||
|__vmx_vmclear||
|__vmx_vmptrld||
|__vmx_vmlaunch||
|__vmx_vmread||
|__vmx_vmwrite|将数据写入VMCS的指定字段|
|__vmx_vmresume||

|VMX Controls|||
|-|-|-|
|VM-Execution Controls|Primary Processor-Based VM-Execution Controls<br>Secondary Processor-Based VM-Execution Controls|设置VMCS|
|VM-entry Control Bits|||
|VM-exit Control Bits|||
|PIN-Based Execution Control||governs the handling of asynchronous events|
|Activity State|0:Active 1:HLT 2:Shutdown 3:Wait-for-SIPI||
|Interruptibility State|permit certain events to be blocked for a period of time||

MyHypervisorDriver通过`__vmx_vmwrite`设置`VMCS`的各项内容（非常复杂），并调用`LaunchVM`在第0号虚拟处理器上设置`VMCS`（设置`CPU_BASED_HLT_EXITING`即在`HLT`时调用`VM-Exit`，设置`HOST_RIP`指向`VMExitHandler`在处理`EXIT_REASON_HLT`中调用`VMXOFF`），最后调用`__vmx_vmlaunch`执行`HLT(\xF4)`，触发`VM-Exit`。

# [Part 6: 虚拟化正在运行的系统](https://rayanfam.com/topics/hypervisor-from-scratch-part-6/)

|||
|-|-|
|CPU_BASED_VM_EXEC_CONTROL|CPU_BASED_ACTIVATE_MSR_BITMAP|
|SECONDARY_VM_EXEC_CONTROL|CPU_BASED_CTL2_RDTSCP<br>CPU_BASED_CTL2_ENABLE_INVPCID<br>CPU_BASED_CTL2_ENABLE_XSAVE_XRSTORS|

[IRQL](https://blogs.msdn.microsoft.com/doronh/2010/02/02/what-is-irql/)(Interrupt Request Level): a Windows-specific mechanism to manage interrupts or giving priority by their level so raising IRQL means your routine will execute with higher priority than normal Windows codes (PASSIVE LEVEL & APC LEVEL).
```c
KIRQL OldIrql = KeRaiseIrqlToDpcLevel(); // raises the IRQL to Dispatch Level so the Windows Scheduler can’t kick in to change the context
KeLowerIrql(OldIrql);
```

CPUID is one the main instructions that cause the VM-Exit.

|内核函数|||
|-|-|-|
|_cpuidex|CPUID|HYPERV_HYPERVISOR_PRESENT_BIT|

[Defeating malware’s Anti-VM techniques (CPUID-Based Instructions)](https://rayanfam.com/topics/defeating-malware-anti-vm-techniques-cpuid-based-instructions/)