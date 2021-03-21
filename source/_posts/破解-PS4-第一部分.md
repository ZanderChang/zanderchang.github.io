---
title: 破解 PS4 - 第一部分：PS4 安全和用户态 ROP 简介
date: 2019-05-06 10:56:54
categories:
- 硬件分析
tags:
- 外文翻译
- PS4
- 硬件分析
- 漏洞挖掘
---

*原文标题 Hacking the PS4, part 1 Introduction to PS4's security, and userland ROP (日期未知)*

作者：[CTurt](https://github.com/CTurt)

由于目前很长时间没有关于 PS4 破解的重大公告，我想解释一下 PS4 破解已经走了多远，以及阻止它进一步发展的原因。

我将解释一些通用于所有现代系统的安全概念，以及我在 PS4 上运行 ROP 测试所做的发现。

本系列文章的目标是展示完整的漏洞利用链，从访问 Internet 浏览器的网页，到最终在 PS4 上实现内核代码执行。

如果您对漏洞挖掘不是特别熟悉，那么您应该先阅读我的关于[通过保存文件中的栈粉碎漏洞来破解 DS 游戏](http://cturt.github.io/DS-exploit-finding.html)的文章。

您可以在此下载我的[完整设置](https://github.com/CTurt/PS4-playground)以自行运行这些测试；它目前仅适用于固件 1.76。如果您正在使用较旧的固件并希望更新到 1.76，则可以下载 [1.76 PUP](http://www.psdevwiki.com/ps4/1.760.000) 文件并[通过 USB 更新](https://www.playstation.com/en-us/support/system-updates/ps4/)。

<!-- more -->

# PS4 背景

> 您可能知道 PS4 使用的是定制的 AMD x86-64 CPU（8核），即使此特定版本可能与已知标准稍有不同，目前仍有许多研究成果可用于此 CPU 架构。 例如，PFLA（Page Fault Liberation Army）在 29C3 大会期间发布了仅使用页错误和 x86 MMU 实现完整图灵机的概念证明，您可以在 YouTube 上查看他们的精彩视频。 如果您尝试在虚拟机中运行代码并希望在主机 CPU 上执行指令，也会很有趣。
> 
> -- <cite>[EurAsia 新闻 3251 号文章](http://www.eurasia.nu/modules.php?name=News&file=article&sid=3251)</cite>

除了表现良好的 CPU 架构外，[PS4 中使用的大部分软件](https://en.wikipedia.org/wiki/PlayStation_4_system_software)都是[开源](http://www.scei.co.jp/ps4-license/)的。

最值得注意的是，PS4 的 Orbis OS 基于 FreeBSD 9.0，就像 PS3 的操作系统一样（也有部分 NetBSD）；并包括各种许多其他开源软件，如 [Mono VM](http://www.mono-project.com/docs/advanced/runtime/) 和 [WebKit](https://www.webkit.org/)。

# 从 WebKit 入手

WebKit 是开源的布局引擎，它被用于 iOS，Wii U，3DS，PS Vita 和 PS4 的浏览器中来呈现网页。

尽管 WebKit 被广泛使用，但它确实存在漏洞；通过阅读 [Pwn2Own write-ups](https://en.wikipedia.org/wiki/Pwn2Own#Summary_of_successful_exploits)，你可以了解其中的许多内容。

特别是 PS4 1.76 版本固件中的浏览器使用的 WebKit 版本易受到 [CVE-2012-3748](https://www.exploit-db.com/exploits/28081/) 的攻击，这是由 `JSArray :: sort(...)` 方法中基于堆的缓冲区溢出造成的。

2014年，NAS 和 Proxima [宣布](http://wololo.net/talk/viewtopic.php?p=368577)他们已经成功地将一个利用该漏洞破解 Mac OS X Safari 代码移植到 PS4 的浏览器，并公布了作为破解 PS4 第一个入口点的 PoC 代码。

这使我们可以任意读写 WebKit 进程可以读取和写入的内容，这些内容可以用于转储模块，并覆盖栈上的返回地址，让我们控制指令指针寄存器（`rip`）来实现 RO P执行。

从那时起，WebKit 中[发现](http://webkitgtk.org/security/WSA-2016-0003.html)了许多[其他漏洞](http://webkitgtk.org/security/WSA-2016-0002.html)，这些漏洞可能被用作 PS4 后期固件的入口点，但截至撰写时，没有人公开宣称将任何这些漏洞利用代码移植到 PS4。

如果您从未登录 PSN，您的 PS4 将无法打开 Internet 浏览器，但您可以转到“设置”，然后选择“用户指南”打开功能有限的 Web 浏览器视图，您可以用代理控制其中的内容。

# ROP 是什么？

与 DS 等早期设备不同，PS4 有一个内核来控制内存不同区域的属性。其中标记为可执行的内存页面不能被覆写，并且标记为可写的内存页面不能被执行; 这称为[数据执行保护（DEP）](https://en.wikipedia.org/wiki/Data_Execution_Prevention)。

这意味着我们不能只将有效负载复制到内存中并执行。但是，我们可以执行已加载到内存中并标记为可执行的代码。

如果我们不能将自己的代码编写到某个地址地址，那么跳转到该地址并不是非常有用，因此我们使用 ROP。

[ROP（Return-Oriented Programming）](https://en.wikipedia.org/wiki/Return-oriented_programming)是传统栈粉碎技术的扩展，我们可以将许多不同的地址（称为 gadgets）链接在一起，而不是仅覆盖 `rip` 将跳转到的单个值。

一条 gadget 通常只是一个所需的指令以及一个 `ret`。

在 x86_64 程序集中，当到达 `ret` 指令时，程序会从栈中弹出一个 64 位的值并且将 `rip` 跳转到它；由于我们可以控制堆栈，所以我们可以使每个 `ret` 指令跳转到下一个所需的 gadget。

例如 `0x80000` 处：

    mov rax, 0
    ret

`0x90000` 处：

    mov rbx, 0
    ret

If we overwrite a return address on the stack to contain `0x80000` followed by `0x90000`, then as soon as the first `ret` instruction is reached execution will jump to `mov rax, 0`, and immediately afterwards, the next `ret` instruction will pop `0x90000` off the stack and jump to `mov rbx, 0`.

Effectively this chain will set both `rax` and `rbx` to 0, just as if we had written the code into a single location and executed it from there.

ROP chains aren't just limited to a list of addresses though; assuming that from `0xa0000` contains these instructions:

    pop rax
    ret

We can set the first item in the chain to `0xa0000` and the next item to any desired value for `rax`.

Gadgets also don't have to end in a `ret` instruction; we can use gadgets ending in a `jmp`:

    add rax, 8
    jmp rcx

By making `rcx` point to a `ret` instruction, the chain will continue as normal:

    chain.add("pop rcx", "ret");
    chain.add("add rax, 8; jmp rcx");

Sometimes you won't be able to find the exact gadget that you need on its own, but with other instructions after it. For example, if you want to set `r8` to something, but only have this gadget, you will have to set `r9` to some dummy value:

    pop r8
    pop r9
    ret

Although you may have to be creative with how you write ROP chains, it is generally accepted that within a sufficiently large enough code dump, there will be enough gadgets for [Turing-complete](https://en.wikipedia.org/wiki/Turing_completeness) functionality; this makes ROP a viable method of defeating DEP.

# 寻找 gadgets

Think of ROP as writing a new chapter to a book, using only words that have appeared at the end of sentences in the previous chapters.

It's obvious from the structure of most sentences that we probably won't be able to find words like 'and' or 'but' appearing at the end of any sentences, but we will need these connectives in order to write anything meaningful.

It is quite possible however, that a sentence has ended with 'sand'. Although the author only ever intended for the word to be read from the 's', if we start reading from the 'a', it will appear as an entirely different word by coincidence, 'and'.

These principles also apply to ROP.

Since almost all functions are structured with a prologue and epilogue:

    ; Save registers
    push    rbp
    mov     rbp, rsp
    push    r15
    push    r14
    push    r13
    push    r12
    push    rbx
    sub     rsp, 18h

    ; Function body

    ; Restore registers
    add     rsp, 18h
    pop     rbx
    pop     r12
    pop     r13
    pop     r14
    pop     r15
    pop     rbp
    ret

You'd expect to only be able to find `pop` gadgets, or more rarely, something like `xor rax, rax` to set the return value to 0 before returning.

Having a comparison like:

    cmp [rax], r12
    ret

Wouldn't make any sense since the result of the comparison isn't used by the function. However, there is still a possibility that we can find gadgets like these.

x86_64 instructions are similar to words in that they have variable lengths, and can mean something entirely different depending on where decoding starts.

> The x86_64 architecture is a variable-length CISC instruction set. Return-oriented programming on the x86_64 takes advantage of the fact that the instruction set is very "dense", that is, any random sequence of bytes is likely to be interpretable as some valid set of x86_64 instructions.
> 
> -- <cite>[Wikipedia](https://en.wikipedia.org/wiki/Return-oriented_programming)</cite>

 To demonstrate this, take a look at the end of this function from the WebKit module:

    000000000052BE0D                 mov     eax, [rdx+8]
    000000000052BE10                 mov     [rsi+10h], eax
    000000000052BE13                 or      byte ptr [rsi+39h], 20h
    000000000052BE17                 ret

Now take a look at what the code looks like if we start decoding from `0x52be14`:

    000000000052BE14                 cmp     [rax], r12
    000000000052BE17                 ret

Even though this code was never intended to be executed, it is within an area of memory which has been marked as executable, so it is perfectly valid to use as a gadget.

Of course, it would be incredibily time consuming to look at every possible way of interpreting code before every single `ret` instruction manually; and that's why tools exist to do this for you. The one which I use to search for ROP gadgets is [rp++](https://github.com/0vercl0k/rp/); to generate a text file filled with gadgets, just use:

    rp-win-x64 -f mod14.bin --raw=x64 --rop=1 --unique > mod14.txt

# General protection faults

If we _do_ perform an access violation, such as by trying to execute a non-executable page of memory, or by trying to write to a non-writable page of memory, a general protection fault, or more specifically in this instance, a [segmentation fault](https://en.wikipedia.org/wiki/Segmentation_fault), will occur.

For example, trying to execute code on the stack, which is mapped as read and write only:

    setU8to(chain.data + 0, 0xeb);
    setU8to(chain.data + 1, 0xfe);

    chain.add(chain.data);

And trying to write to code, which is mapped as read and execute only:

    setU8to(moduleBases[webkit], 0);

If a general protection fault occurs, a message saying "There is not enough free system memory" will appear, and the page will fail to load:

![](/img/hacking-ps4-1/segfault.png)

This message will also be displayed for other hard faults, such as division by 0, or execution of an invalid instruction or unimplemented system call, but most commonly it will be encountered by performing a segmentation fault.

# ASLR

[Address Space Layout Randomization (ASLR)](https://en.wikipedia.org/wiki/Address_space_layout_randomization) is a security technique which causes the base addresses of modules to be different every time you start the PS4.

It has been reported to me that [very old firmwares (1.05) don't have ASLR enabled](https://www.youtube.com/watch?v=aEu208625XA), but it was introduced sometime before firmware 1.70. Note that kernel ASLR is not enabled (for firmwares 1.76 and lower at least), which will be proved later in the article.

For most exploits ASLR would be a problem because if you don't know the addresses of the gadgets in memory, you would have no idea what to write to the stack.

Luckily for us, we aren't limited to just writing static ROP chains. We can use JavaScript to read the modules table, which will tell us the base addresses of all loaded modules. Using these bases, we can then calculate the addresses of all our gadgets before we trigger ROP execution, defeating ASLR.

The modules table also includes the filenames of the modules:

*   WebProcess.self
*   libkernel.sprx
*   libSceLibcInternal.sprx
*   libSceSysmodule.sprx
*   libSceNet.sprx
*   libSceNetCtl.sprx
*   libSceIpmi.sprx
*   libSceMbus.sprx
*   libSceRegMgr.sprx
*   libSceRtc.sprx
*   libScePad.sprx
*   libSceVideoOut.sprx
*   libScePigletv2VSH.sprx
*   libSceOrbisCompat.sprx
*   libSceWebKit2.sprx
*   libSceSysCore.sprx
*   libSceSsl.sprx
*   libSceVideoCoreServerInterface.sprx
*   libSceSystemService.sprx
*   libSceCompositeExt.sprx

Although the PS4 predominantly uses the [**S**igned] **P**PU **R**elocatable E**x**ecutable ([S]PRX) format for modules, some string references to [**S**igned] **E**xecutable and **L**inking **F**ormat ([S]ELF) object files can also be found in the `libSceSysmodule.sprx` dump, such as `bdj.elf`, `web_core.elf` and `orbis-jsc-compiler.self`. This combination of modules and objects is similar to what is used in the PSP and PS3.

You can view [a complete list of all modules available](http://www.psdevwiki.com/ps4/Libraries) (not just those loaded by the browser) in `libSceSysmodule.sprx`. We can load and dump some of these through several of Sony's custom system calls, which will be explained later in this article.

# JuSt-ROP

Using JavaScript to write and execute dynamic ROP chains gives us a tremendous advantage over a traditional, static buffer overflow attack.

As well as being necessary to defeat ASLR, JavaScript also lets us read the user agent of the browser, and provide different ROP chains for different browser versions, giving our exploit a greater range of compatibility.

We can even use JavaScript to read the memory at our gadgets' addresses to check that they are correct, giving us almost perfect reliability. Theoretically, you could take this even further by writing a script to dynamically find ROP gadgets and then build ROP chains on the fly.

Writing ROP chains dynamically, rather than generating them with a script beforehand, just makes sense.

I created a JavaScript framework for writing ROP chains, [JuSt-ROP](https://github.com/CTurt/JuSt-ROP), for this very reason.

# JavaScript caveats

JavaScript represents numbers using the [IEEE-754](https://en.wikipedia.org/wiki/IEEE_floating_point) double-precision (64 bit) format. This provides us with 53 bit precision, meaning that it isn't possible to represent every 64 bit value, approximations will have to be used for some.

If you just need to set a 64 bit value to something low, like `256`, then `setU64to` will be fine.

But for situations in which you need to write a buffer or struct of data, there is the possibility that certain bytes will be written incorrectly if it has been written in 64 bit chunks.

Instead, you should write data in 32 bit chunks (remembering that the PS4 is little endian), to ensure that every byte is exact.

# 系统调用

Interestingly, the PS4 uses the same [calling convention as Linux and MS-DOS for system calls](https://www.freebsd.org/doc/en/books/developers-handbook/x86-system-calls.html#x86-alternate-calling-convention), with arguments stored in registers, rather than the traditional UNIX way (which FreeBSD uses by default), with arguments stored in the stack:

*   `rax` - System call number
*   `rdi` - Argument 1
*   `rsi` - Argument 2
*   `rdx` - Argument 3
*   `r10` - Argument 4
*   `r8` - Argument 5
*   `r9` - Argument 6

We can try to perform any system call with the following JuSt-ROP method:

    this.syscall = function(name, systemCallNumber, arg1, arg2, arg3, arg4, arg5, arg6) {
    	console.log("syscall " + name);

    	this.add("pop rax", systemCallNumber);
    	if(typeof(arg1) !== "undefined") this.add("pop rdi", arg1);
    	if(typeof(arg2) !== "undefined") this.add("pop rsi", arg2);
    	if(typeof(arg3) !== "undefined") this.add("pop rdx", arg3);
    	if(typeof(arg4) !== "undefined") this.add("pop rcx", arg4);
    	if(typeof(arg5) !== "undefined") this.add("pop r8", arg5);
    	if(typeof(arg6) !== "undefined") this.add("pop r9", arg6);
    	this.add("mov r10, rcx; syscall");
    }

Just make sure to set the stack base to some free memory beforehand:

    this.add("pop rbp", stackBase + returnAddress + 0x1400);

Using system calls can tell us a huge amount about the PS4 kernel. Not only that, but using system calls is most likely the only way that we can interact with the kernel, and thus potentially trigger a kernel exploit.

If you are reverse engineering modules to identify some of Sony's custom system calls, you may come across an alternative calling convention:

Sometimes Sony performs system calls through regular system call 0 (which usually does nothing in FreeBSD), with the first argument (`rdi`) controlling which system call should be executed:

*   `rax` - 0
*   `rdi` - System call number
*   `rsi` - Argument 1
*   `rdx` - Argument 2
*   `r10` - Argument 3
*   `r8` - Argument 4
*   `r9` - Argument 5

It is likely that Sony did this to have easy compatibility with the function calling convention. For example:

    .global syscall
    syscall:
    	xor     rax, rax
    	mov     r10, rcx
    	syscall
    	ret

Using this, they can perform system calls from C using the function calling convention:

    int syscall();

    int getpid(void) {
    	return syscall(20);
    }

When writing ROP chains, we can use either convention:

    // Both will get the current process ID:
    chain.syscall("getpid", 20);
    chain.syscall("getpid", 0, 20);

It's good to be aware of this, because we can use whichever one is more convenient for the gadgets that are available.

# getpid

Just by using system call 20, [getpid(void)](http://www.freebsd.org/cgi/man.cgi?query=getpid&sektion=2), we can learn a lot about the kernel.

The very fact that this system call works at all tells us that Sony didn't bother mixing up the system call numbers as a means of [security through obscurity](https://en.wikipedia.org/wiki/Security_through_obscurity) (under the BSD license they could have done this without releasing the new system call numbers).

So, we automatically have a [list of system calls in the PS4 kernel](http://fxr.watson.org/fxr/source/kern/syscalls.master?v=FREEBSD9) to try.

Secondly, by calling `getpid()`, restarting the browser, and calling it again, we get a return value 2 higher than the previous value.

This tells us that the Internet Browser app actually consists of 2 separate processes: the WebKit core (which we take over), that handles parsing HTML and CSS, decoding images, and executing JavaScript for example, and another one to handle everything else: displaying graphics, receiving controller input, managing history and bookmarks, etc.

Also, although FreeBSD has supported [PID randomisation](http://security.stackexchange.com/questions/88692/do-randomized-pid-bring-more-security) since 4.0, sequential PID allocation is the default behaviour.

The fact that PID allocation is set to the default behaviour indicates that Sony likely didn't bother adding any additional security enhancements such as those encouraged by projects like [HardenedBSD](https://hardenedbsd.org/), other than userland ASLR.

## How many custom system calls are there?

The [last standard FreeBSD 9 system call](http://fxr.watson.org/fxr/source/kern/syscalls.master?v=FREEBSD9#L952) is `wait6`, number `532`; anything higher than this must be a custom Sony system call.

Invoking most of Sony's custom system calls without the correct arguments will return error `0x16`, `"Invalid argument"`; however, any compatibility or unimplemented system calls will report the "There is not enough free system memory" error.

Through trial and error, I have found that system call number `617` is the last Sony system call, anything higher is unimplemented.

From this, we can conclude that there are 85 custom Sony system calls in the PS4's kernel (617 - 532).

## libkernel.sprx

To identify how custom system calls are used by libkernel, you must first remember that it is just a modification of the standard FreeBSD 9.0 libraries.

Here's an extract of `_libpthread_init` from [thr_init.c](https://github.com/freebsd/freebsd/blob/release/9.0.0/lib/libthr/thread/thr_init.c#L297):

    /*
     * Check for the special case of this process running as
     * or in place of init as pid = 1:
     */
    if ((_thr_pid = getpid()) == 1) {
    	/*
    	 * Setup a new session for this process which is
    	 * assumed to be running as root.
    	 */
    	if (setsid() == -1)
    		PANIC("Can't set session ID");
    	if (revoke(_PATH_CONSOLE) != 0)
    		PANIC("Can't revoke console");
    	if ((fd = __sys_open(_PATH_CONSOLE, O_RDWR)) &lt; 0)
    		PANIC("Can't open console");
    	if (setlogin("root") == -1)
    		PANIC("Can't set login to root");
    	if (_ioctl(fd, TIOCSCTTY, (char *) NULL) == -1)
    		PANIC("Can't set controlling terminal");
    }

The same function can be found at offset `0x215F0` from `libkernel.sprx`. This is how the above extract looks from within a libkernel dump:

    call    getpid
    mov     cs:dword_5B638, eax
    cmp     eax, 1
    jnz     short loc_2169F

    call    setsid
    cmp     eax, 0FFFFFFFFh
    jz      loc_21A0C

    lea     rdi, aDevConsole ; "/dev/console"
    call    revoke
    test    eax, eax
    jnz     loc_21A24

    lea     rdi, aDevConsole ; "/dev/console"
    mov     esi, 2
    xor     al, al
    call    open

    mov     r14d, eax
    test    r14d, r14d
    js      loc_21A3C
    lea     rdi, aRoot       ; "root"
    call    setlogin
    cmp     eax, 0FFFFFFFFh
    jz      loc_21A54

    mov     edi, r14d
    mov     esi, 20007461h
    xor     edx, edx
    xor     al, al
    call    ioctl
    cmp     eax, 0FFFFFFFFh
    jz      loc_21A6C

## Reversing module dumps to analyse system calls

libkernel isn't completely open source though; there's also a lot of custom code which can help disclose some of Sony's system calls.

Although this process will vary depending on the system call you are looking up; for some, it is fairly easy to get a basic understanding of the arguments that are passed to it.

The system call wrapper will be declared somewhere in `libkernel.sprx`, and will almost always follow this template:

    000000000000DB70 syscall_601     proc near
    000000000000DB70                 mov     rax, 259h
    000000000000DB77                 mov     r10, rcx
    000000000000DB7A                 syscall
    000000000000DB7C                 jb      short error
    000000000000DB7E                 retn
    000000000000DB7F
    000000000000DB7F error:
    000000000000DB7F                 lea     rcx, sub_DF60
    000000000000DB86                 jmp     rcx
    000000000000DB86 syscall_601     endp

Note that the `mov r10, rcx` instruction doesn't necessarily mean that the system call takes at least 4 arguments; all system call wrappers have it, even those that take no arguments, such as `getpid`.

Once you've found the wrapper, you can look up xrefs to it:

    0000000000011D50                 mov     edi, 10h
    0000000000011D55                 xor     esi, esi
    0000000000011D57                 mov     edx, 1
    0000000000011D5C                 call    syscall_601
    0000000000011D61                 test    eax, eax
    0000000000011D63                 jz      short loc_11D6A

It's good to look up several of these, just to make sure that the registers weren't modified for something unrelated:

    0000000000011A28                 mov     edi, 9
    0000000000011A2D                 xor     esi, esi
    0000000000011A2F                 xor     edx, edx
    0000000000011A31                 call    syscall_601
    0000000000011A36                 test    eax, eax
    0000000000011A38                 jz      short loc_11A3F

Consistently, the first three registers of the system call convention (`rdi`, `rsi`, and `rdx`) are modified before invoking the call, so we can conclude with reasonable confidence that it takes 3 arguments.

For clarity, this is how we would replicate the calls in JuSt-ROP:

    chain.syscall("unknown", 601, 0x10, 0, 1);
    chain.syscall("unknown", 601, 9, 0, 0);

As with most system calls, it will return 0 on success, as seen by the `jz` conditional after `test`ing the return value.

Looking up anything beyond than the amount of arguments will require a much more in-depth analysis of the code before and after the call to understand the context, but this should help you get started.

## Brute forcing system calls

Although reverse engineering module dumps is the most reliable way to identify system calls, some aren't referenced at all in the dumps we have so we will need to analyse them blindly.

If we guess that a certain system call might take a particular set of arguments, we can brute force all system calls which return a certain value (`0` for success) with the arguments that we chose, and ignore all which returned an error.

We can also pass 0s for all arguments, and brute force all system calls which return useful errors such as `0xe`, `"Bad address"`, which would indicate that they take at least one pointer.

Firstly, we will need to execute the ROP chain as soon as the page loads. We can do this by attaching our function to the `body` element's `onload`:

    <body onload="exploit()">

Next we will need to perform a specific system call depending on an HTTP GET value. Although this can be done with JavaScript, I will demonstrate how to do this using PHP for simplicity:

    var Sony = 533;
    chain.syscall("Sony system call", Sony + &lt;?php print($_GET["b"]); ?&gt;, 0, 0, 0, 0, 0, 0);
    chain.write_rax_ToVariable(0);

Once the system call has executed, we can check the return value, and if it isn't interesting, redirect the page to the next system call:

    if(chain.getVariable(0) == 0x16) window.location.assign("index.php?b=" + (&lt;?php print($_GET["b"]); ?&gt; + 1).toString());

Running the page with `?b=0` appended to the end will start the brute force from the first Sony system call.

Although this method requires a lot of experimentation, by passing different values to some of the system calls found by brute forcing and analysing the new return values, there are a few system calls which you should be able to partially identify.

## System call 538

As an example, I'll take a look at system call 538, without relying on any module dumps.

These are the return values depending on what is passed as the first argument:

*   0 - `0x16`, `"Invalid argument"`
*   1 - `0xe`, `"Bad address"`
*   Pointer to 0s - `0x64` initially, but each time the page is refreshed this value increases by 1

Other potential arguments to try would be PID, thread ID, and file descriptor.

Although most system calls will return `0` on success, due to the nature of the return value increasing after each time it is called, it seems like it is allocating a resource number, such as a file descriptor. 

The next thing to do would be to look at the data before and after performing the system call, to see if it has been written to.

Since there is no change in the data, we can assume that it is an input for now.

I then tried passing a long string as the first argument. You should always try this with every input you find because there is the possibility of discovering a buffer overflow.

    writeString(chain.data, "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa");
    chain.syscall("unknown", 538, chain.data, 0, 0, 0, 0, 0);

The return value for this is `0x3f`, `ENAMETOOLONG`. Unfortunately it seems that this system call correctly limits the name (32 bytes including `NULL` truncator), but it does tell us that it _is_ expecting a string, rather than a struct.

We now have a few possibilities for what this system call is doing, the most obvious being something related to the filesystem (such as a custom `mkdir` or `open`), but this doesn't seem particularly likely seeing as a resource was allocated even before we wrote any data to the pointer.

To test whether the first parameter is a path, we can break it up with multiple `/` characters to see if this allows for a longer string:

    writeString(chain.data, "aaaaaaaaaa/aaaaaaaaaa/aaaaaaaaaa");
    chain.syscall("unknown", 538, chain.data, 0, 0, 0, 0, 0);

Since this also returns `0x3f`, we can assume that the first argument isn't a path; it is a name for _something_ that gets allocated a sequential identifier.

After analysing some more system calls, I found that the following all shared this exact same behaviour:

*   533
*   538
*   557
*   574
*   580

From the information that we have so far, it is almost impossible to pinpoint exactly what these system calls do, but as you run more tests, further information will slowly be revealed.

To save you some time, system call 538 is allocating an event flag (and it doesn't just take a name).

Using general knowledge of how a kernel works, you can guess, and then verify, what the system calls are allocating (semaphores, mutexes, etc).

## Dumping additional modules

We can dump additional modules by following these stages:

*   Load the module
*   Get the module's base address
*   Dump the module

I've extracted and posted a list of all module names on [psdevwiki](http://www.psdevwiki.com/ps4/Libraries#Libraries_on_firmware_1.76).

To load a module we will need to use the `sceSysmoduleLoadModule` function from `libSceSysmodule.sprx + 0x1850`. The first parameter is the module ID to load, and the other 3 should just be passed 0.

The following JuSt-ROP method can be used to perform a function call:

    this.call = function(name, module, address, arg1, arg2, arg3, arg4, arg5, arg6) {
    	console.log("call " + name);

    	if(typeof(arg1) !== "undefined") this.add("pop rdi", arg1);
    	if(typeof(arg2) !== "undefined") this.add("pop rsi", arg2);
    	if(typeof(arg3) !== "undefined") this.add("pop rdx", arg3);
    	if(typeof(arg4) !== "undefined") this.add("pop rcx", arg4);
    	if(typeof(arg5) !== "undefined") this.add("pop r8", arg5);
    	if(typeof(arg6) !== "undefined") this.add("pop r9", arg6);
    	this.add(module_bases[module] + address);
    }

So, to load `libSceAvSetting.sprx` (`0xb`):

    chain.call("sceSysmoduleLoadModule", libSysmodule, 0x1850, 0xb, 0, 0, 0);

Unfortunately, a fault will be triggered when trying to load certain modules; this is because the `sceSysmoduleLoadModule` function doesn't load dependencies, so you will need to manually load them first.

Like most system calls, this should return `0` on success. To see the loaded module ID that was allocated, we can use one of Sony's custom system calls, number `592`, to get a list of currently loaded modules:

    var countAddress = chain.data;
    var modulesAddress = chain.data + 8;

    // System call 592, getLoadedModules(int *destinationModuleHandles, int max, int *count);
    chain.syscall("getLoadedModules", 592, modulesAddress, 256, countAddress);

    chain.execute(function() {
    	var count = getU64from(countAddress);
    	for(var index = 0; index &lt; count; index++) {
    		logAdd("Module handle: 0x" + getU32from(modulesAddress + index * 4).toString(16));
    	}
    });

Running this without loading any additional modules will produce the following list:

    0x0, 0x1, 0x2, 0xc, 0xe, 0xf, 0x11, 0x12, 0x13, 0x14, 0x15, 0x16, 0x17, 0x18, 0x19, 0x1a, 0x1b, 0x1e, 0x37, 0x59

But if we run it after loading module `0xb`, we will see an additional entry, `0x65`. Remember that module ID is _not_ the same as loaded module handle.

We can now use another of Sony's custom system calls, number `593`, which takes a module handle and a buffer, and fills the buffer with information about the loaded module, including its base address. Since the next available handle is always `0x65`, we can hardcode this value into our chain, rather than having to store the result from the module list.

The buffer must start with the size of the struct that should be returned, otherwise error `0x16` will be returned, `"Invalid argument"`:

    setU64to(moduleInfoAddress, 0x160);
    chain.syscall("getModuleInfo", 593, 0x65, moduleInfoAddress);

    chain.execute(function() {
    	logAdd(hexDump(moduleInfoAddress, 0x160));
    });

It will return `0` upon success, and fill the buffer with a struct which can be read like so:

    var name = readString(moduleInfoAddress + 0x8);
    var codeBase = getU64from(moduleInfoAddress + 0x108);
    var codeSize = getU32from(moduleInfoAddress + 0x110);
    var dataBase = getU64from(moduleInfoAddress + 0x118);
    var dataSize = getU32from(moduleInfoAddress + 0x120);

We now have everything we need to dump the module!

    dump(codeBase, codeSize + dataSize);

There is another Sony system call, number `608`, which works in a similar way to `593`, but provides slightly different information about the loaded module:

    setU64to(moduleInfoAddress, 0x1a8);
    chain.syscall("getDifferentModuleInfo", 608, 0x65, 0, moduleInfoAddress);
    logAdd(hexDump(moduleInfoAddress, 0x1a8));

It's not clear what this information is.

## Browsing the filesystem

The PS4 uses the standard FreeBSD 9.0 system calls for reading files and directories.

However, whilst using `read` for some directories such as `/dev/` will work, others, such as `/` will fail.

I'm not sure why this is, but if we use `getdents` instead of `read` for directories, it will work much more reliably:

    writeString(chain.data, "/dev/");
    chain.syscall("open", 5, chain.data, 0, 0);
    chain.write_rax_ToVariable(0);

    chain.read_rdi_FromVariable(0);
    chain.syscall("getdents", 272, undefined, chain.data + 0x10, 1028);

This is the resultant memory:

    0000010: 0700 0000 1000 0205 6469 7073 7700 0000  ........dipsw...
    0000020: 0800 0000 1000 0204 6e75 6c6c 0000 0000  ........null....
    0000030: 0900 0000 1000 0204 7a65 726f 0000 0000  ........zero....
    0000040: 0301 0000 0c00 0402 6664 0000 0b00 0000  ........fd......
    0000050: 1000 0a05 7374 6469 6e00 0000 0d00 0000  ....stdin.......
    0000060: 1000 0a06 7374 646f 7574 0000 0f00 0000  ....stdout......
    0000070: 1000 0a06 7374 6465 7272 0000 1000 0000  ....stderr......
    0000080: 1000 0205 646d 656d 3000 0000 1100 0000  ....dmem0.......
    0000090: 1000 0205 646d 656d 3100 0000 1300 0000  ....dmem1.......
    00000a0: 1000 0206 7261 6e64 6f6d 0000 1400 0000  ....random......
    00000b0: 1000 0a07 7572 616e 646f 6d00 1600 0000  ....urandom.....
    00000c0: 1400 020b 6465 6369 5f73 7464 6f75 7400  ....deci_stdout.
    00000d0: 1700 0000 1400 020b 6465 6369 5f73 7464  ........deci_std
    00000e0: 6572 7200 1800 0000 1400 0209 6465 6369  err.........deci
    00000f0: 5f74 7479 3200 0000 1900 0000 1400 0209  _tty2...........
    0000100: 6465 6369 5f74 7479 3300 0000 1a00 0000  deci_tty3.......
    0000110: 1400 0209 6465 6369 5f74 7479 3400 0000  ....deci_tty4...
    0000120: 1b00 0000 1400 0209 6465 6369 5f74 7479  ........deci_tty
    0000130: 3500 0000 1c00 0000 1400 0209 6465 6369  5...........deci
    0000140: 5f74 7479 3600 0000 1d00 0000 1400 0209  _tty6...........
    0000150: 6465 6369 5f74 7479 3700 0000 1e00 0000  deci_tty7.......
    0000160: 1400 020a 6465 6369 5f74 7479 6130 0000  ....deci_ttya0..
    0000170: 1f00 0000 1400 020a 6465 6369 5f74 7479  ........deci_tty
    0000180: 6230 0000 2000 0000 1400 020a 6465 6369  b0.. .......deci
    0000190: 5f74 7479 6330 0000 2200 0000 1400 020a  _ttyc0..".......
    00001a0: 6465 6369 5f73 7464 696e 0000 2300 0000  deci_stdin..#...
    00001b0: 0c00 0203 6270 6600 2400 0000 1000 0a04  ....bpf.$.......
    00001c0: 6270 6630 0000 0000 2900 0000 0c00 0203  bpf0....).......
    00001d0: 6869 6400 2c00 0000 1400 0208 7363 655f  hid.,.......sce_
    00001e0: 7a6c 6962 0000 0000 2e00 0000 1000 0204  zlib............
    00001f0: 6374 7479 0000 0000 3400 0000 0c00 0202  ctty....4.......
    0000200: 6763 0000 3900 0000 0c00 0203 6463 6500  gc..9.......dce.
    0000210: 3a00 0000 1000 0205 6462 6767 6300 0000  :.......dbggc...
    0000220: 3e00 0000 0c00 0203 616a 6d00 4100 0000  >.......ajm.A...
    0000230: 0c00 0203 7576 6400 4200 0000 0c00 0203  ....uvd.B.......
    0000240: 7663 6500 4500 0000 1800 020d 6e6f 7469  vce.E.......noti
    0000250: 6669 6361 7469 6f6e 3000 0000 4600 0000  fication0...F...
    0000260: 1800 020d 6e6f 7469 6669 6361 7469 6f6e  ....notification
    0000270: 3100 0000 5000 0000 1000 0206 7573 6263  1...P.......usbc
    0000280: 746c 0000 5600 0000 1000 0206 6361 6d65  tl..V.......came
    0000290: 7261 0000 8500 0000 0c00 0203 726e 6700  ra..........rng.
    00002a0: 0701 0000 0c00 0403 7573 6200 c900 0000  ........usb.....
    00002b0: 1000 0a07 7567 656e 302e 3400 0000 0000  ....ugen0.4.....
    00002c0: 0000 0000 0000 0000 0000 0000 0000 0000  ................

You can read some of these devices, for example: reading `/dev/urandom` will fill the memory with random data.

It is also possible to parse this memory to create a clean list of entries; look at `browser.html` in the repository for a complete file browser:

![](/img/hacking-ps4-1/browser.png)

Unfortunately, due to sandboxing we don't have complete access to the file system. Trying to read files and directories that [do exist](http://www.psdevwiki.com/ps4/Talk:Files_on_the_PS4) but are restricted will give you error 2, `ENOENT`, `"No such file or directory"`.

We do have access to a lot of interesting stuff though including encrypted save data, trophies, and account information. I will go over more of the filesystem in my next article.

## Sandboxing

As well as file related system calls failing for certain paths, there are other reasons for a system call to fail.

Most commonly, a disallowed system call will just return error `1`, `EPERM`, `"Operation not permitted"`; such as trying to use `ptrace`, but other system calls may fail for different reasons:

Compatibilty system calls are disabled. If you are trying to call `mmap` for example, you must use [system call number 477](http://fxr.watson.org/fxr/source/kern/syscalls.master?v=FREEBSD9#L851), not [71](http://fxr.watson.org/fxr/source/kern/syscalls.master?v=FREEBSD9#L172) or [197](http://fxr.watson.org/fxr/source/kern/syscalls.master?v=FREEBSD9#L379); otherwise a segfault will be triggered.

Other system calls such as `exit` will also trigger a fault:

    chain.syscall("exit", 1, 0);

Trying to create an SCTP socket will return error `0x2b`, `EPROTONOSUPPORT`, indicating that SCTP sockets have been disabled in the PS4 kernel:

    //int socket(int domain, int type, int protocol);
    //socket(AF_INET, SOCK_STREAM, IPPROTO_SCTP);
    chain.syscall("socket", 97, 2, 1, 132);

And although calling `mmap` with `PROT_READ | PROT_WRITE | PROT_EXEC` will return a valid pointer, the `PROT_EXEC` flag is ignored. Reading its protection will return 3 (RW), and any attempt to execute the memory will trigger a segfault:

    chain.syscall("mmap", 477, 0, 4096, 1 | 2 | 4, 4096, -1, 0);
    chain.write_rax_ToVariable(0);
    chain.read_rdi_FromVariable(0);
    chain.add("pop rax", 0xfeeb);
    chain.add("mov [rdi], rax");
    chain.add("mov rax, rdi");
    chain.add("jmp rax");

The list of [open source software used in the PS4](http://www.scei.co.jp/ps4-license/) doesn't list any kind of sandboxing software like [Capsicum](https://www.freebsd.org/cgi/man.cgi?query=capsicum), so the PS4 must use either pure [FreeBSD jails](https://www.freebsd.org/doc/handbook/jails.html), or some kind of custom, proprietary, sandboxing system (unlikely).

## Jails

We can prove the existence of FreeBSD jails being actively used in the PS4's kernel through the `auditon` system call being impossible to execute within a jailed environment:

    chain.syscall("auditon", 446, 0, 0, 0);

The first thing the `auditon` system call does is check `jailed` [here](https://github.com/freebsd/freebsd/blob/release/9.0.0/sys/security/audit/audit_syscalls.c#L164), and if so, return `ENOSYS`:

    if (jailed(td->td_ucred))
    	return (ENOSYS);

Otherwise the system call would most likely return `EPERM` from the `mac_system_check_auditon` [here](https://github.com/freebsd/freebsd/blob/release/9.0.0/sys/security/audit/audit_syscalls.c#L169):

    error = mac_system_check_auditon(td->td_ucred, uap->cmd);
    if (error)
    	return (error);

Or from the `priv_check` [here](https://github.com/freebsd/freebsd/blob/release/9.0.0/sys/security/audit/audit_syscalls.c#L174):

    error = priv_check(td, PRIV_AUDIT_CONTROL);
    if (error)
    	return (error);

The absolute furthest that the system call could reach would be immediately after the `priv_check`, [here](https://github.com/freebsd/freebsd/blob/release/9.0.0/sys/security/audit/audit_syscalls.c#L178), before returning `EINVAL` due to the length argument being 0:

    if ((uap->length <= 0) || (uap->length > sizeof(union auditon_udata)))
    	return (EINVAL);

Since `mac_system_check_auditon` and `priv_check` will never return `ENOSYS`, having the `jailed` check pass is the only way `ENOSYS` could be returned.

When executing the chain, `ENOSYS` _is_ returned (`0x48`).

This tells us that whatever sandbox system the PS4 uses is at least based on jails because the `jailed` check passes.

## FreeBSD 9.0 kernel exploits

Before trying to look for new vulnerabilities in the [FreeBSD 9.0 kernel source code](https://github.com/freebsd/freebsd/tree/release/9.0.0/sys/kern), we should first check whether any of the [kernel vulnerabilities already found](https://www.exploit-db.com/platform/?p=freebsd) could be used on the PS4.

We can immediately dismiss some of these for obvious reasons:

*   [FreeBSD 9.0-9.1 mmap/ptrace - Privilege Escalation Exploit](https://www.exploit-db.com/exploits/26368/) - this won't work since, as previously stated, we don't have access to the `ptrace` system call.
*   [FreeBSD 9.0 - Intel SYSRET Kernel Privilege Escalation Exploit](https://www.exploit-db.com/exploits/28718/) - won't work because the PS4 uses an AMD processor.
*   [FreeBSD Kernel - Multiple Vulnerabilities](https://www.exploit-db.com/exploits/35938/) - maybe the first vulnerability will lead to something, but the other 2 rely on SCTP sockets, which the PS4 kernel has disabled (as previously stated).

However, there are [some smaller vulnerabilites](http://www.cvedetails.com/vulnerability-list/vendor_id-6/product_id-7/version_id-118765/Freebsd-Freebsd-9.0.html), which could lead to something:

## getlogin

One vulnerability which looks easy to try is using the [getlogin system call to leak a small amount of kernel memory](http://www.cvedetails.com/cve/CVE-2014-8476/).

The [`getlogin`](https://www.freebsd.org/cgi/man.cgi?query=getlogin&amp;sektion=2) system call is intended to copy the login name of the current session to userland memory, however, due to a bug, the whole buffer is always copied, and not just the size of the name string. This means that we can read some uninitialised data from the kernel, which might be of some use.

Note that the system call (49) is actually `int getlogin_r(char *name, int len);` and not `char *getlogin(void);`.

So, let's try copying some kernel memory into an unused part of userland memory:

    chain.syscall("getlogin", 49, chain.data, 17);

Unfortunately 17 bytes is the most data we can get, since:

> Login names are limited to `MAXLOGNAME` (from `&lt;sys/param.h&gt;`) characters, currently 17 including null.
> 
> -- <cite>[FreeBSD Man Pages](https://www.freebsd.org/cgi/man.cgi?query=getlogin&amp;sektion=2)</cite>

After executing the chain, the return value was 0, which means that the system call worked! An excellent start. Now let's take a look at the memory which we pointed to:

Before executing the chain:

    00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
    00

After executing the chain:

    72 6f 6f 74 00 fe ff ff 08 62 61 82 ff ff ff ff
    00

After decoding the first 4 bytes as ASCII:

    root

So the browser is executed as root! That was unexpected.

But more interestingly, the memory leaked looks like a pointer to something in the kernel, which is always the same each time the chain is run; this is evidence to support [Yifanlu's claims that the PS4 has no Kernel ASLR](https://twitter.com/yifanlu/status/551498289284009984)!

## Summary

From the information currently available, the PS4's kernel seems to be very similar to the stock FreeBSD 9.0 kernel.

Importantly, the differences that _are_ present appear to be from standard [kernel configuration](https://www.freebsd.org/doc/handbook/kernelconfig-custom-kernel.html) changes (such as disabling SCTP sockets), rather than from modified code. Sony have also added several of their own custom system calls to the kernel, but apart from this, the rest of the kernel seems fairly untouched.

In this respect, I'm inclined to believe that the PS4 shares most of the same juicy vulnerabilities as FreeBSD 9.0's kernel!

Unfortunately, most kernel exploits cannot be triggered from the WebKit entry point that we currently have due to sandboxing constraints (likely to be just stock FreeBSD jails).

And with FreeBSD 10 being out, it's unlikely that anyone is stashing away any private exploits for FreeBSD 9, so unless a new one is suddenly released, we're stuck with what is currently available.

The best approach from here seems to be reverse engineering all of the modules which can be dumped, in order to document as many of Sony's custom system calls as possible; I have a hunch that we will have more luck targeting these, than the standard FreeBSD system calls.

Recently [Jaicrab has discovered two UART ports on the PS4](http://hackinformer.com/2015/06/24/breaking-news-discovered-two-communication-ports-uart-playstation-4/) which shows us that there are hardware hackers interested in the PS4. Although the role of hardware hackers has traditionally been to dump the RAM of a system, [like with the DSi](http://farm4.static.flickr.com/3441/3869187499_da1665050d.jpg), which we can already do thanks to the WebKit exploit, there's also the possibility of a hardware triggered kernel vulnerability being found, like [geohot's original PS3 hypervisor hack](http://rdist.root.org/2010/01/27/how-the-ps3-hypervisor-was-hacked/). It remains most likely that a kernel exploit will be found on the PS4 through system call vulnerabilities though.

[原文地址](http://cturt.github.io/ps4.html)