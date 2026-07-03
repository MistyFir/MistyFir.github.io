# 从0到1实现NPT无痕Hook——内存虚拟化Hook技术全解析

> 本文从实现者的角度，讲清楚基于AMD NPT（Nested Page Tables）的内存虚拟化Hook技术"为什么这样设计"以及"应该怎么实现"。重点是思路，具体代码示例在我的 [**Github仓库**](https://github.com/MistyFir/SVMHypervisor-Preview) 上。

---

## 技术背景：为什么需要"无痕"Hook？

### Hook技术的演进——一场"修改与检测"的军备竞赛

Hook（钩子）技术几乎和Windows操作系统一样古老。从应用层的API Hook到内核层的SSDT Hook，从简单的地址替换到复杂的Inline Patch，二十多年来Hook技术一直在和"反Hook"技术做猫鼠游戏。理解这段历史，才能明白为什么NPT/EPT虚拟化Hook会出现，以及它解决了什么问题。

**第一代：表项替换。** 早期的Hook极其简单——操作系统用表来存储函数指针（比如SSDT系统服务表、IDT中断描述符表、驱动对象里的IRP处理函数指针），你只需要把表里的指针改成你自己的函数地址就完事了。这种方法在Windows XP时代非常流行，但致命问题是太容易被检测：反作弊和安全软件只要把表中的地址和磁盘上的原始文件对比一下，立刻就能发现指针被改过。

**第二代：Inline Hook。** 既然改表太显眼，那就不改表，直接改函数开头的机器码——在函数入口写入一条跳转指令（jmp rel32，5字节），CPU执行到这里就会跳到你的函数。执行完你的逻辑后，再通过一个"跳板"（trampoline）执行被覆盖的原始字节，跳回原函数继续执行。这就是经典的Inline Hook，也是Detours、MinHook等库的原理。Inline Hook比表项替换隐蔽得多——它不修改任何表指针，只是修改了代码字节。但问题随之而来：你改了代码字节，就一定有办法被检测到。任何对代码段做CRC校验或Hash比对的机制（比如Windows的PatchGuard/KPP、现代反作弊的.text段完整性校验）都能发现这5字节的jmp指令与磁盘上的原始文件不一致。

**第三代：绕过修改。** 为了对抗代码段校验，各种"无痕"手段被发明出来：
- **硬件断点Hook**：利用CPU的DR0-DR3调试寄存器设置执行断点，不修改任何内存。但调试寄存器数量有限（只有4个），且很容易被`GetThreadContext`检测到——反作弊只要读一下DR寄存器就知道你在Hook。
- **VEH/Page Guard Hook**：通过修改页属性为PAGE_GUARD触发异常，在异常处理器中重定向执行流。但异常处理链本身可以被遍历和检测，且性能开销大。
- **HAL/内核回调**：利用Windows提供的回调机制（如进程创建回调、注册表回调）做监控。但回调数量有限，无法拦截任意函数。

这些方法虽然在"不修改代码"上前进了一步，但都有各自的致命短板——要么能被轻易检测，要么能Hook的位置受限，要么性能差到无法接受。

**第四代：硬件虚拟化Hook。** 到这里，一个根本性的问题摆上台面：**有没有一种方法，能让CPU在"读"某段代码时看到原始字节，但在"执行"这段代码时跳转到我们的逻辑——而且全程不修改任何内存？**

答案在2005年前后出现了。Intel和AMD相继推出硬件辅助虚拟化技术（VT-x和AMD-V），其中包含一个关键特性——**二级地址翻译（SLAT, Second Level Address Translation）**。Intel叫它EPT（Extended Page Tables），AMD叫它NPT（Nested Page Tables）。这个机制原本是为了让虚拟机更高效地运行——Hypervisor通过EPT/NPT控制Guest物理地址到主机物理地址的映射，不需要像早期软件虚拟化那样"影子页表"。但安全研究者很快意识到：**既然Hypervisor能控制每个物理页的"读/写/执行"权限，而且这个控制发生在CPU硬件层面、Guest完全看不见——那这不就是做Hook的完美武器吗？**

2006年，Joanna Rutkowska提出了著名的"Blue Pill"概念——在运行时通过AMD-V把正在运行的操作系统整个包裹进一个Hypervisor里，操作系统对此完全无感知。这个概念震惊了安全界，也开启了硬件虚拟化Rootkit的研究。此后十多年里，基于EPT/NPT的内存Hook技术逐渐成熟：从学术项目（如2013年的SPIDER框架）到开源Hypervisor（如SimpleSvmHook、DdiMon、HyperDbg），虚拟化Hook从理论走向了工程实现。

### NPT Hook的核心优势

相比前三代Hook技术，基于SLAT的虚拟化Hook有几个颠覆性的优势：

1. **代码页零修改**：被Hook函数的内存字节从头到尾没有被改过一个bit。任何读取这个地址的操作（不管是CRC校验、Hash比对、反汇编扫描、还是直接dump到文件）看到的都是原始代码。
2. **PatchGuard无感知**：PatchGuard检查的是它能读到的内存内容——而它读到的始终是干净的原始页，所以永远不会报被篡改。
3. **NPT不可见**：NPT页表由Hypervisor持有，Guest操作系统没有任何指令可以读取NPT的内容（不像CR3指向的Guest页表可以被遍历）。Guest无法通过软件手段知道GPA背后映射到了哪个HPA。
4. **Ring -1特权级**：Hypervisor运行在比内核（Ring 0）更高的特权级（严格意义来讲还是Ring 0层级，Ring -1只是行业把Hypervisor所在的Ring 0层级区分开来所定义的一个层级），Guest的任何操作——哪怕是内核态代码——都无法绕过或关闭NPT（除非Hypervisor自身存在安全漏洞）。
5. **不依赖未文档化接口**：整个机制基于AMD/Intel公开的硬件规范，不需要hook SSDT、不需要Patch内核、不需要依赖任何未导出函数。

当然，NPT Hook也不是万能的——它需要在内核中加载一个驱动来初始化Hypervisor（虽然启动后可以移除驱动痕迹），会引入VMEXIT的性能开销，而且也不是完全不可检测（第14章会讨论检测与对抗）。但就目前而言，它是Windows平台上隐蔽性最强、最稳定的Hook手段之一。

本文要讲的，就是如何从零实现这样一个基于AMD NPT的无痕Hook框架的思路。

---

## 目录

- [技术背景：为什么需要"无痕"Hook？](#技术背景：为什么需要无痕hook)
- [零、你需要先知道什么](#零你需要先知道什么)
- [一、核心原理：CPU到底在什么地方给了我们可乘之机](#一核心原理cpu到底在什么地方给了我们可乘之机)
- [二、整体方案的演化：从粗糙到精细](#二整体方案的演化从粗糙到精细)
- [三、第一步：把整个系统塞进虚拟机](#三第一步把整个系统塞进虚拟机)
- [四、第二步：构建你自己完全掌控的NPT页表](#四第二步构建你自己完全掌控的npt页表)
- [五、第三步：利用NX位制造"取指陷阱"](#五第三步利用nx位制造取指陷阱)
- [六、第四步：影子页——让"读"和"执行"看到不同内容](#六第四步影子页让读和执行看到不同内容)
- [七、第五步：跳板——从Hook点安全地跳到你的代码](#七第五步跳板从hook点安全地跳到你的代码)
- [八、第六步：把一切拼起来——NPF与#BP的联动](#八第六步把一切拼起来npf与bp的联动)
- [九、第七步：Hook执行完之后——复位与卸载](#九第七步hook执行完之后复位与卸载)
- [十、隐藏你自己——Hypervisor的反侦察](#十隐藏你自己hypervisor的反侦察)
- [十一、多核、TLB、中断——那些会让你蓝屏的坑](#十一多核tlb中断那些会让你蓝屏的坑)
- [十二、不同Hook设计方案的对比与取舍](#十二不同hook设计方案的对比与取舍)
- [十三、工程实现清单——从零开始要写哪些模块](#十三工程实现清单从零开始要写哪些模块)
- [十四、调试方法论](#十四调试方法论)
- [十五、写在最后](#十五写在最后)

---

## 零、你需要先知道什么

阅读本文前，假设你已经了解：

- x64分页机制（PML4/PDPT/PD/PT四级页表、PTE位域、2MB大页）
- Windows内核驱动开发基础（DriverEntry、IRQL、非分页内存、自旋锁）
- x64调用约定（RCX/RDX/R8/R9传参、shadow space、栈对齐）
- 基本的操作系统概念（Ring 0/3、物理地址/虚拟地址、CR3寄存器）

不需要提前了解SVM/AMD-V细节，我会在用到时讲。

---

## 一、核心原理：CPU到底在什么地方给了我们可乘之机

做任何Hook，本质上都是要回答一个问题：**如何让CPU在执行到某个地址时，不执行那里原本的指令，而是执行我们的指令？**

传统Inline Hook的回答是：**把那个地址的指令改掉**，写一条jmp跳过来。简单粗暴，但有致命问题——任何人（PatchGuard、反作弊、你的调试器）只要去读那个地址，就能看到你写的jmp。

NPT Hook换了一个角度来回答这个问题：

> **不改指令，改CPU看到的物理内存。**

x64 CPU在开启分页后执行一条指令，要经过"虚拟地址→物理地址"的翻译。这个翻译是CPU的内存管理单元（MMU）自动完成的。在没有虚拟化时，CR3指向一套页表，MMU按照这套页表把VA（虚拟地址）翻译成PA（物理地址）。

但当CPU运行在SVM Guest模式时，翻译变成了两级：

```
GVA ──(Guest CR3 页表)──► GPA ──(NPT，nCR3 指向)──► HPA
```

关键点在这里：**第二级翻译（NPT）是Hypervisor完全控制的，Guest操作系统根本看不见也改不了NPT**。Guest知道的只是GPA，至于GPA背后到底对应哪块HPA上的物理内存，是Hypervisor说了算。

这就给了我们一个上帝视角：**同一个GPA，我们可以让它在"读/写"时指向物理页A（原始代码），在"取指执行"时指向物理页B（我们动过手脚的页）。**

CPU怎么区分"读/写"和"取指"？因为NPT的页表项里有NX（No Execute，bit63）位。如果我们把一页设为"可读可写但不可执行"（NX=1），CPU在这页上取指时会触发一个Nested Page Fault（#NPF），并且在错误码里用一个bit告诉我们："这次违规是因为取指（execute）"。

这就是整个NPT Hook技术的根基。后面所有复杂的机制（影子页、跳板、TLB刷新），都是在这个根基上长出来的工程细节。

---

## 二、整体方案的演化：从粗糙到精细

在动手之前，先让我们用"方案迭代"的方式推导出正确的设计，这样你理解每个部分时就知道它是为了解决什么问题。

### 方案A：直接在原始页上写jmp + 用NPT保护？不行。

你可能想：我在原始代码页上写jmp到我的函数，然后用NPT把这页设为只读，防止别人覆盖我的jmp。但这没有解决根本问题——任何人读这页都能看到jmp指令，CRC校验立刻发现异常。**核心矛盾：写了jmp就会被看到。**

### 方案B：双页切换——读时给原页，执行时给修改页。

好，那我不修改原始页。我准备两个物理页：
- 页A：原始代码（完整拷贝，未修改）
- 页B：代码的拷贝，但在函数入口处写了跳转到我的Hook函数

平时NPT把GPA映射到页A（读/写/执行都指向A）。当CPU要执行目标函数时，我想办法把NPT切到页B，执行完了再切回A。

问题：怎么知道CPU"要执行目标函数"了？一个办法是把页A的NX位置1，禁止执行。这样取指时触发NPF，在NPF处理中我把NPT切到页B（可执行），让CPU重新取指——就跳到了页B上的jmp。

但这还有两个问题：
1. **切到页B之后，任何人读这页都会读到jmp**——如果在执行Hook的同时，另一个线程/核心恰好读这页（比如反作弊做CRC扫描），就暴露了。
2. **页B上的jmp是直接跳到Hook函数的**，Hook函数执行完怎么回到原函数？需要跳板。

还有更严重的问题：**如果我们把B页做成含jmp的页，那B页上jmp跳到Hook函数后，Hook函数返回时需要执行被jmp覆盖掉的原始指令**——这就是经典的Trampoline（跳板）问题，和传统Inline Hook的跳板是一个道理。

### 方案C：读/写始终指向原页，执行瞬间指向含INT3的影子页，BP处理中跳到跳板。

问题1的解决思路：让页B（执行视图）上**不是jmp，而是INT3（0xCC）**。执行到INT3会触发#BP异常，我们拦截#BP，在VMEXIT中把Guest RIP改成跳板地址（由Hypervisor接管跳转控制，隐藏跳板地址），然后立刻切回页A。这样页B暴露的时间窗口被压缩到"执行一条单字节INT3的瞬间。

等等，这里还有问题：即使不切回A，如果另一个核心同时读这页，它的NPT也是独立的（每核心NPT），可以让它始终指向A。但同一核心上，NPF切到B之后，如果在执行INT3之前有一个读操作进来怎么办？实际上，INT3是单字节指令，从NPT切到B到#BP触发VMEXIT之间，CPU只会执行这一条INT3，不可能发生读操作（因为执行流已经陷入VMEXIT了）。

### 最终方案（经典实现）

综合以上推演，最终方案包含这几个核心部件：

1. **NPT页表**（每核心一份）：Hypervisor自己的四级页表，实现GPA→HPA翻译
2. **初始恒等映射**：启动时全部用2MB大页做GPA==HPA的恒等映射
3. **影子页0（读/写视图）**：原始代码页的完整拷贝，NX=1（禁止执行），读/写都看这个
4. **影子页1（执行视图）**：同样是拷贝，但在Hook函数入口写INT3（0xCC），NX=0（可执行）
5. **NPF处理**：取指触发NX缺页→切NPT到影子页1→刷新TLB→返回Guest重新执行
6. **#BP拦截**：INT3触发#BP→VMEXIT中匹配RIP是Hook点→把RIP改成跳板地址→切回影子页0
7. **跳板（Trampoline）**：保存寄存器→调用Hook函数→执行被覆盖的原始指令→跳回原函数→VMMCALL通知Hypervisor
8. **复位机制**：确保影子页及时复位，缩短暴露窗口
9. **Hypervisor自身隐藏**：把所有私有数据结构映射到全零页

后面的章节我们逐个展开。

---

## 三、第一步：把整个系统塞进虚拟机

### 3.1 为什么可以"运行时虚拟化"？

很多人以为Hypervisor必须在系统启动前加载（像VMware、Hyper-V那样）。其实不然。SVM（以及VT-x）允许我们在系统运行时动态启用虚拟化——已经运行的操作系统可以作为Guest被"包裹"进Hypervisor里。这就是所谓的**Blue Pill**类型Hypervisor。

核心思路：
1. 写一个内核驱动，在DriverEntry中创建系统线程
2. 在这个线程中，在当前CPU上启用SVM、配置VMCB、执行VMRUN
3. VMRUN后，原来正在运行的Windows就变成了Guest，继续执行——它感觉不到任何变化
4. 但从此，所有敏感事件都会触发VMEXIT到我们的Hypervisor代码

这是完全可行的，因为VMCB保存了Guest的所有寄存器状态，VMRUN时从VMCB加载这些状态后，Guest就从VMRUN的下一条指令继续执行，跟什么都没发生一样。

### 3.2 前置检测

在做任何事情之前，先确认CPU支持：

```
1. CPUID leaf 0: EBX/ECX/EDX == "AuthenticAMD"（AMD CPU）
2. CPUID leaf 0x80000001: ECX bit2 == 1（支持SVM）
3. RDMSR(0xC0010114) bit4 == 0（BIOS没有锁定SVM，即SVMDIS位为0）
4. CPUID leaf 0x8000000A: EDX bit0 == 1（支持NPT）
```

如果第3步失败（SVMDIS=1），说明BIOS禁用了SVM，需要提示用户重启进BIOS打开。

### 3.3 启用SVM

设置EFER.SVME位（EFER的bit12，MSR地址0xC0000080）：

```c
ULONG64 efer = __readmsr(MSR_EFER);  // 0xC0000080
__writemsr(MSR_EFER, efer | EFER_SVME);  // bit12
```

这一步是全局开关。设置后CPU进入"SVM已启用"状态，可以执行VMRUN指令了。

### 3.4 分配和初始化VMCB

VMCB是一块4KB的物理连续内存，按照AMD APM定义的格式组织。分配时必须用`MmAllocateContiguousMemory`（保证物理连续），然后按字节清0。

VMCB分两部分：
- **偏移0-0x3FF（Control Area）**：配置哪些事件拦截到Hypervisor
- **偏移0x400-0xFFF（State Save Area）**：Guest的寄存器状态

初始化State Save Area的思路：**把当前CPU正在运行的状态"冻结"下来作为Guest的初始状态**。这意味着你需要：
- 读取当前的CR0、CR3、CR4、EFER等值写入VMCB
- 读取当前所有段寄存器（CS/SS/DS/ES/FS/GS/TR/LDTR/GDTR/IDTR）
- 读取当前的RIP、RSP、RFLAGS
- MSR相关：STAR、LSTAR、CSTAR、SFMask、KernelGsBase、Sysenter CS/ESP/EIP等

这样VMRUN之后Guest就从你当前的执行点继续运行——Windows对此完全无感。

Control Area需要设置的关键项：
```
NpEnable = 1              // 启用NPT
NCr3    = NPT_PML4_PA    // NPT的PML4物理地址（后面构建）
GuestAsid = cpuIndex + 1  // ASID，每CPU唯一，用于TLB标记
TlbControl = 0x01         // 首次进入时Flush整个ASID的TLB

// 拦截位图
InterceptException: bit3 = 1   // 拦截 #BP (INT3)
InterceptMisc1:    拦截MSR读写、IO指令、SHUTDOWN
InterceptMisc2:    拦截VMMCALL、VMRUN

// IOPM和MSRPM的物理地址
IopmBasePa  = IOPM_PA;
MsrpmBasePa = MSRPM_PA;
```

### 3.5 HSAVE区域

还需要分配一页物理连续内存作为HSAVE区，物理地址写入MSR_VM_HSAVE_PA（0xC0010117），用于硬件在VMRUN/VMSAVE之间保存Host状态。

### 3.6 Host栈和Host VMCB

VMEXIT发生时CPU要切到Host栈执行我们的处理代码。需要为每个CPU分配一块足够大的栈（如64KB-128KB，非分页内存），以及一个Host VMCB（主要用来保存Host的RSP）。

### 3.7 进入VM循环

这些准备好之后，通过一段汇编来进入VM循环(伪代码)：

```js
AsmLaunchVm PROC
    push    rbx
    ; 保存当前原始RSP（用于退出VM后恢复）
    mov     [rcx].SavedRsp, rsp
    
    ; 切换到Host栈
    mov     rsp, [rcx].HostStackTop
    
    ; 压入context指针作为参数
    push    rcx
    
VmLoop:
    vmload  VMCB_GUEST       ; 加载Guest状态
    vmrun                    ; 进入Guest（这里会"阻塞"直到VMEXIT）
    clgi                     ; 关中断
    vmsave  VMCB_GUEST       ; 保存Guest状态
    
    ; 检查是否需要退出VM循环
    test    [rcx].Running, 1
    jz      VmExit
    
    ; 保存所有通用寄存器和XMM寄存器到栈
    push    rax; push rbx; push rcx; push rdx; ...
    sub     rsp, 16*16
    movaps  [rsp+0],  xmm0
    movaps  [rsp+16], xmm1
    ; ... 保存XMM0-15
    
    ; 调用C的VmExitHandler(context, regs)
    mov     rdx, rsp         ; regs指针
    call    VmExitHandler
    
    ; 恢复寄存器
    movaps  xmm0, [rsp+0]
    ; ...
    add     rsp, 16*16
    pop     rdx; pop rcx; pop rbx; pop rax; ...
    
    jmp     VmLoop
    
VmExit:
    ; 恢复原始RSP，返回到调用者
    mov     rsp, [rcx].SavedRsp
    pop     rbx
    ret
AsmLaunchVm ENDP
```

执行完VMRUN之后，Guest就开始运行了。你的系统此时已经在虚拟化环境中运行。先什么都不做（NPT是恒等映射），确认系统能稳定运行——如果连这一步都过不了，后面的Hook也不用谈了。

---

## 四、第二步：构建你自己完全掌控的NPT页表

### 4.1 为什么要自己构建NPT？

VMCB中NCr3指向的NPT是我们掌控Guest物理内存视图的根本。如果不设置NpEnable（或者NPT有问题），Guest的GPA就是HPA，跟没有虚拟化一样——这当然不是我们想要的。

### 4.2 初始映射策略：恒等映射+2MB大页

启动时NPT的最简实现是**恒等映射**（GPA == HPA），让Guest看到真实的物理内存，不做任何篡改。为了减少页表开销和TLB miss，使用2MB大页：

```
PML4（1页）
  └── PML4E[0] → PDPT
PDPT（1页）
  └── PDPTE[i] → PD[i]   （i = 0..511）
PD[i]（512页，每页512项）
  └── PDE[j] → 2MB物理页 (i*512 + j)*0x200000  （LargePage位=1）
```

这样514页（约2MB）就能映射512GB物理内存，覆盖绝大多数消费级硬件的内存容量。每个PDE的权限位设置为P=1、R/W=1、U/S=1（因为我们需要允许Guest内核和用户都访问）、A=1、D=1、NX=0（允许执行）。

### 4.3 页表项格式

NPT的PTE格式跟x64普通PTE完全一样，这是NPT相比EPT的一个友好之处：

```
bit0:  P (Present)
bit1:  R/W (1=writable)
bit2:  U/S (1=user accessible)
bit3:  PWT (Page-Level Write-Through)
bit4:  PCD (Page-Level Cache Disable)
bit5:  A (Accessed，硬件置位)
bit6:  D (Dirty，硬件置位，仅对叶子PTE有效)
bit7:  LargePage/Pat (PD项为1表示2MB大页)
bit8:  G (Global)
bit12-51 (63 for 4KB): PFN
bit63: NX (No Execute)
```

### 4.4 大页拆分

2MB大页对于"整页权限一致"的场景够用了，但当我们需要Hook一个函数时，我们只需要对函数所在的4KB页做手脚，不希望影响同一个2MB内其他4KB页。因此需要大页拆分机制。

拆分的思路：
1. 找到要拆分的2MB大页对应的PDE
2. 分配一个新的PT页（512个PTE，每项映射4KB）
3. 把原大页覆盖的512个4KB页的信息填入PT
4. 原子地把PDE从"大页模式"改成"指向PT的4KB模式"（用InterlockedCompareExchange64防止多核竞争）
5. 刷新TLB

伪代码如下：
```c
BOOL Split2MBLargePage(PPTE pde, ULONG64 largePagePA) {
    PPT pt = AllocateOnePageFromPool();
    
    // 初始化PT：512个PTE，映射到原来2MB内的各4KB页
    for (int k = 0; k < 512; k++) {
        pt[k].Flags = pde->Flags & 0x8000000000000FFF;  // 保留NX等权限位
        pt[k].PageFrameNumber = (largePagePA >> 12) + k;
        pt[k].Present = 1;
        pt[k].Write = 1;
    }
    
    // 原子替换：PDE从大页变为指向PT
    PTE newPde = {0};
    newPde.Present = 1;
    newPde.Write = 1;
    newPde.PageFrameNumber = GetPhysicalAddress(pt) >> 12;
    
    PTE oldPde;
    oldPde.Flags = InterlockedCompareExchange64(
        (volatile LONG64*)pde, newPde.Flags, pde->Flags
    );
    
    if (oldPde.Flags != pde->Flags) {
        // 另一个核心已经拆分了，释放分配的PT
        FreePageToPool(pt);
    }
    return TRUE;
}
```

为什么要原子操作？因为可能有多个核心同时在NPF处理中尝试拆分同一个大页——第一个成功的真正做了拆分，其他的检测到PDE已经变了就直接放弃。

### 4.5 页池管理

页表页、影子页都需要物理页，但VMEXIT运行在高IRQL（关中断），不能直接分配物理内存。解决方案：**预分配多个4KB大小的物理页**，管理哪些页空闲、哪些已使用。


### 4.6 懒更新（Lazy Page Table Construction）

初始只映射前512GB。如果Guest访问了未映射的GPA（NPF，Present=0），就在NPF处理中动态补建页表：遍历四级PML4→PDPT→PD→PT，遇到中间项不存在就从页池分配，最后PTE填恒等映射。这样不需要启动时映射全部物理内存。

---

## 五、第三步：利用NX位制造"取指陷阱"

### 5.1 基本思路

有了NPT之后，我们的第一件武器就是NX位。假设我们想Hook函数F（地址为0xFFFFF800`12345000）：

1. 把F所在的4KB页对应的NPT PTE的NX位设为1（禁止执行）
2. 保持P=1、R/W=1（可读可写）——这样读/写这个页完全正常
3. CPU取指到这页时，NX=1触发NPF，ExitInfo1.Id=1（instruction fetch）
4. 在NPF VMEXIT中我们就能知道"有人要执行这页的代码了"

这就是一个最简单的"执行陷阱"。

### 5.2 修改NPT权限的通用函数

需要写一个通用函数，修改指定GPA在NPT中的权限，伪代码如下：

```c
VOID SetNptPageProtection(CPU_CONTEXT* cpu, ULONG64 gpa, BOOL present, BOOL writable, BOOL noExecute, ULONG64 newHpa) {
    // 1. 遍历NPT四级，找到对应PTE（需要时Split大页）
    PPTE pml4 = cpu->Npt.Pml4;
    // ... walk ...
    
    // 2. 修改PTE
    PTE newPte = {0};
    newPte.Present = present ? 1 : 0;
    newPte.Write = writable ? 1 : 0;
    newPte.NoExecute = noExecute ? 1 : 0;
    newPte.PageFrameNumber = (newHpa != 0 ? newHpa : gpa) >> 12;
    newPte.Accessed = 1;
    newPte.Dirty = 1;
    
    InterlockedExchange64((volatile LONG64*)pte, newPte.Flags);
    
    // 3. 标记需要TLB刷新
    cpu->PendingTlbFlush = TRUE;
}
```

注意`newHpa`参数：这是核心——它允许我们把GPA映射到一个**不同的HPA**（比如影子页的物理地址）。这就是视图切换的底层原语。

### 5.3 NPF错误码解读

NPF VMEXIT时，ExitInfo1是错误码，结构如下：

```
bit0 (Present): 0=页不存在    1=页存在但权限冲突
bit1 (Write):   0=读/取指访问  1=写访问
bit2 (User):    0=内核态      1=用户态
bit4 (ID):      0=数据访问    1=指令取指（Instruction Fetch）
```

我们关心的是：当`Present=1`且`ID=1`时，就是"在一个存在的页上取指但`NX=1`"——这正是我们要的执行陷阱。

---

## 六、第四步：影子页——让"读"和"执行"看到不同内容

### 6.1 为什么需要影子页？

光有NX陷阱只能让我们知道"CPU要执行这个地址了"，但CPU触发NPF后，我们还得让它能**继续执行**。如果我们直接把NX清0让它执行原页，那执行的就是原始指令——没有Hook效果。

我们需要的是：NPF之后，CPU重新取指时看到的不是原始指令，而是"能跳转到Hook函数"的代码。

方案：**不修改原始物理页，而是准备一个影子物理页，内容是原页的拷贝，但在Hook点动了手脚。NPF时把NPT从原始物理页切到这个影子页。**

### 6.2 影子页0（读/写视图）

分配一个新物理页（从页池），memcpy原始页的内容过去。这页内容完全干净——读它就等于读原页。NPT在正常状态下始终指向这一页，NX=1（禁止执行）。

等等——为什么读/写不直接指向原始物理页，要多此一举拷贝一份？

确实可以直接指向原始物理页（很多实现就是这么做的）。但用影子页0有一个额外好处：**可以在这页上设置与原页不同的权限**（比如禁止写），或者让它映射到零页（用于隐藏Hypervisor自身）。对于代码Hook场景，影子页0可以直接等于原始物理页（不分配新页），但框架设计上预留这个能力更灵活。

### 6.3 影子页1（执行视图）

再分配一个物理页，同样`memcpy`原始页内容。但在**被Hook函数的入口偏移处**，写入一个字节`0xCC（INT3）`和`nop`对齐第一条指令。

为什么只写一个0xCC而不是一条完整的jmp？三个原因：

1. **触发精确**：INT3执行时立即产生#BP异常，CPU会VMEXIT到我们手里，RIP可精确匹配Hook点
2. **不需要修改更多字节**：jmp rel32需要5字节，可能截断多条指令。INT3只需1字节，其他字节填NOP即可
3. **跳转发生在Hypervisor层**：我们不需要影子页上真有jmp——在#BP的VMEXIT处理中直接改Guest RIP到跳板地址就行，比在影子页上写jmp更干净

### 6.4 默认状态

Hook安装完成后，目标GPA在NPT中的默认映射：

```
PTE.PFN = 影子页0的物理页帧号（或原始物理页帧）
PTE.Present = 1
PTE.R/W = 1（可读可写）
PTE.NX = 1（禁止执行）
```

此时Guest读/写目标地址都正常（看到干净代码），但一执行就NPF。

---

## 七、第五步：跳板——从Hook点安全地跳到你的代码

### 7.1 跳板要解决的问题

当我们在#BP VMEXIT中把Guest RIP改成跳板地址后，Guest开始执行跳板代码。跳板需要完成：

1. **保存现场**：所有寄存器（尤其是x64调用约定的RCX/RDX/R8/R9参数寄存器和栈上参数）
2. **调用Hook函数**：把参数传给你的Hook函数，让它决定放行/拦截/修改
3. **执行被"跳过"的原始指令**：因为INT3占了原函数入口位置，原函数开头的指令没有被执行，跳板里必须有这些指令的副本
4. **跳回原函数**：执行完复制的序言指令后，跳到原函数序言之后的位置继续执行
5. **正常返回**：原函数执行完ret后，能回到调用者

### 7.2 跳板内存布局

跳板是一段动态生成的机器码，分配在非分页可执行内存（ExAllocatePool2 with POOL_FLAG_NON_PAGED_EXECUTE）。结构如下：

```
偏移 0           24             24 + prolog_size
┌────────────────┬────────────────┬──────────────────────┬──────────┐
│ 数据区(24字节) │  指令保存区     │ 原函数序言副本         │ 返回逻辑  │
│ 3个8字节指针   │  (push/保存)   │ (被复制的原始指令)     │ vmmcall  │
└────────────────┴────────────────┴──────────────────────┴──────────┘
```

数据区存放3个地址：
- `hookFunc`：你的Hook处理函数地址
- `originalAfterProlog`：原函数地址 + 被复制的序言长度（回跳点）
- `trampolineReturn`：跳板"返回后处理"标签地址

### 7.3 跳板执行流（伪代码）

```js
; === 跳板入口 ===
TrampolineEntry:
    sub  rsp, 0x80 + 0x88      ; 分配栈空间（含x64 shadow space + 寄存器保存）
    mov  [rsp + 0x80], rax     ; 保存RAX
    
    ; 保存参数（x64 calling convention）
    mov  [rsp + 0x88 + 0x00], rcx
    mov  [rsp + 0x88 + 0x08], rdx
    mov  [rsp + 0x88 + 0x10], r8
    mov  [rsp + 0x88 + 0x18], r9
    ; 注意：栈上的第5+个参数还在原来的位置，因为我们没动返回地址
    
    ; 准备调用Hook函数
    lea  rax, [AfterHookCall]
    push rax                   ; HookFunc ret后回到AfterHookCall
    jmp  [data.hookFunc]       ; 调用HookFunc（tail call）

AfterHookCall:
    cmp  eax, STATUS_ACCESS_DENIED
    je   Blocked
    
    ; 放行：恢复寄存器，执行原始指令
    mov  rax, [rsp + 0x80]
    add  rsp, 0x80 + 0x88
    
    sub  rsp, 0x28             ; x64 shadow space (32 bytes) + 8 for alignment
    mov  rcx, [saved_rcx]
    mov  rdx, [saved_rdx]
    mov  r8,  [saved_r8]
    mov  r9,  [saved_r9]
    
    ; === 这里执行复制过来的原函数序言 ===
    ; （原始序言指令被memcpy到这里）
    ; 例如：push rbp; mov rbp,rsp; sub rsp,0x20; ...
    ;
    ; 序言最后一条指令应该跳到原函数+序言长度
    jmp  [data.originalAfterProlog]

Blocked:
    mov  eax, STATUS_ACCESS_DENIED
    add  rsp, 0x80 + 0x88
    
HookComplete:
    push rax                   ; 保存返回值
    mov  rax, VMMCALL_HOOK_DONE
    vmmcall                    ; 通知Hypervisor：本线程Hook执行完毕
    pop  rax
    ret                        ; 返回调用者
```

这里有个很微妙的地方：跳板本身不是用CALL指令调用原函数，而是用JMP。当原函数执行完RET时，栈顶是调用者的返回地址——RET正确返回到调用被Hook函数的地方。

### 7.4 关键问题：复制多少字节序言？

这是一个工程上的难点。你需要从原函数入口开始，复制若干字节到跳板的OriginalCode区域，这些字节必须覆盖整数条完整指令（不能在指令中间截断），并且总长度要足够容纳跳板开头的"hook跳转"逻辑。

具体来说：
- 影子页1的INT3只需要1字节，但如果Hook函数需要检查参数或做特殊处理，你需要确定至少要复制多长的原函数序言才能安全地跳回原函数
- 通常需要复制5字节以上（一条jmp rel32的长度），但更安全的做法是用指令长度解码器，循环解码直到覆盖至少一个安全的跳转位置
- 在实际工程中，由于我们在影子页1里只写了1字节INT3，实际上只"覆盖"了1字节，我们必须复制完整的一条指令到跳板中，至于计算指令长度，可以用HDE等反汇编引擎来实现。

这就是为什么项目里需要一个x64指令长度解码器（不需要完整反汇编，只需要算出每条指令多长）。

### 7.5 为什么跳板不隐藏？

跳板分配在`NonPagedExecute`池，需要让Guest能执行（否则跳过去会NPF）。但是：
- 跳板地址不在任何模块的代码段范围内
- Hook函数本身放在一个NPT隐藏的内存区域（映射零页），所以即使有人追踪到跳板，也跳不到真正的Hook处理函数——因为Hook函数地址那块内存对Guest来说是零页。

---

## 八、第六步：把一切拼起来——NPF与#BP的联动

这是整个Hook触发的核心流程。让我们从头追踪一次完整的Hook触发：

### 8.1 Guest调用被Hook的函数

```
Guest线程调用目标函数F(0xFFFFF800`12345000)
    │
    ▼
CPU 取指：GVA = F的地址
    │
    ▼ Guest页表翻译（CR3）
GPA = F所在物理页
    │
    ▼ NPT翻译（nCR3）
PTE检查：NX=1（禁止执行！）
    │
    ▼
触发 #NPF VMEXIT
ExitCode = 0x400
ExitInfo1.Present=1, Write=0, ID=1（取指违例）
ExitInfo2 = 故障GPA
GuestRIP = F的地址
```

### 8.2 NPF VMEXIT处理

```c
HandleNPF(cpu, vmcb):
    faultGPA = vmcb->ControlArea.ExitInfo2;
    faultRIP = vmcb->StateSaveArea.Rip;
    exitInfo = vmcb->ControlArea.ExitInfo1;
    
    if (!exitInfo.Present) {
        // 真正的缺页：补建NPT或注入Guest #PF
        if (NeedLazyMap(faultGPA)) {
            BuildNptEntry(cpu, faultGPA);  // 懒更新
            return;
        }
        InjectException(vmcb, EXCEPTION_PF, 1, MakePfErrorCode(exitInfo));
        return;
    }
    
    if (exitInfo.ID) {
        // === 取指违例：NX触发，Hook的第一道门 ===
        
        // 查找这个RIP是否对应已注册的Hook
        HOOK* hook = FindHookByRip(faultRIP);
        
        if (hook != NULL) {
            // 把NPT切到影子页1（含INT3，可执行）
            RemapPage(cpu, faultGPA, hook->ShadowPage1PA, 
                      present=TRUE, writable=FALSE, noExecute=FALSE);
            
            // 需要TLB Flush
            vmcb->ControlArea.TlbControl = TLB_FLUSH_ASID;
            return;  // 回到Guest，CPU重新取指
        }
        
        // 不是Hook点，注入#GP
        InjectException(vmcb, EXCEPTION_GP, 1, 0);
        return;
    }
    
    if (exitInfo.Write) {
        // 写违例（比如写了只读页）
        // 根据需要处理，比如Hypervisor自身的保护页
        HandleWriteViolation(cpu, vmcb, faultGPA);
        return;
    }
```

### 8.3 Guest重新执行——取到INT3

NPF处理完后，VMRUN回到Guest。CPU重新从RIP=F取指：

```
这次NPT指向影子页1（NX=0，可执行）
影子页1在F入口处是0xCC（INT3）
    │
    ▼
CPU 执行 INT3 → 触发 #BP（Breakpoint Exception）
因为VMCB中设置了InterceptException bit3（拦截#BP）
    │
    ▼
VMEXIT！
ExitCode = 0x43（VMEXIT_EXCP_BP）
GuestRIP = F的入口地址（VMCB.StateSaveArea.Rip保存断点地址）
```

### 8.4 #BP VMEXIT处理

```c
HandleBP(cpu, vmcb):
    faultRIP = vmcb->StateSaveArea.Rip;
    
    // 查找这个RIP对应的Hook
    HOOK_FUNC* func = FindHookFunc(faultRIP);
    
    if (func != NULL) {
        // 核心：直接把Guest RIP改成跳板地址
        vmcb->StateSaveArea.Rip = func->TrampolineAddr + func->TrampolineExecOffset;
        
        // 关键：在切走之前，把NPT切回影子页0（NX=1），为下次Hook做准备
        ULONG64 gpa = VirtualToPhysical((PVOID)func->OriginalAddr);
        RemapPage(cpu, gpa, func->HookInfo->ShadowPage0PA, 
                  present=TRUE, writable=TRUE, noExecute=TRUE);
        vmcb->ControlArea.TlbControl = TLB_FLUSH_ASID;
        return;
    }
    
    // 不是Hook点的INT3：吞掉或者转发给Guest
    if (g_DebugMode) {
        InjectException(vmcb, EXCEPTION_BP, 1, 0);
    } else {
        vmcb->StateSaveArea.Rip = vmcb->ControlArea.NRip;  // 跳过INT3
    }
```

### 8.5 执行跳板

VMRUN回Guest后，RIP指向跳板。跳板开始执行：保存寄存器→调用Hook函数→执行原始序言→跳回原函数+序言偏移→原函数正常执行→ret返回到调用者。

整个Hook触发流程完成。

---

## 九、第七步：Hook执行完之后——复位与卸载

VMMCALL是Guest主动与Hypervisor通信的指令。跳板最后可以通过`vmmcall`通知Hypervisor执行复位操作，Hypervisor在VMEXIT_VMMCALL中处理：

```c
HandleVmmcall(cpu, vmcb, regs):
    if (vmcb->StateSaveArea.Cpl != 0) {  // 只有Ring0允许
        InjectException(vmcb, EXCEPTION_GP, 1, 0);
        return;
    }
    
    switch (regs->Rax) {
    case VMMCALL_RESET_SHADOWS:
        ResetAllHooksToShadow0(cpu);
        break;
    }
    
    vmcb->StateSaveArea.Rip = vmcb->ControlArea.NRip;
```

### 9.1 周期性复位线程

你可能注意到：在#BP处理中我们已经把NPT切回了影子页0（NX=1）。但如果因为某种原因（比如多核并发、异常流程）NPT没有正确复位，怎么办？

更安全的做法是创建一个高优先级系统线程，每隔很短时间（如5ms）在每个CPU上执行`VMMCALL_RESET_SHADOWS`，强制把所有Hook页的NPT复位到影子页0（NX=1）。这个"双保险"确保影子页1暴露的时间窗口极短。

```c
VOID ResetShadowThread(PVOID ctx) {
    while (!g_Unload) {
        for (ULONG i = 0; i < g_NumCpus; i++) {
            if (!g_HvRunning) break;
            KeSetSystemAffinityThread(MAKELONG(1<<i, 0));
            AsmVmmcallResetShadows();  // vmmcall VMMCALL_RESET_SHADOWS
        }
        // 使用KeDelayExecutionThread休眠5ms
    }
}
```

### 9.2 Hook卸载流程
伪代码如下：
```c
VOID Unhook(HOOK_FUNC* func) {
    // 1. 从链表摘除，防止新的NPF匹配到它
    RemoveEntryList(&func->ListEntry);
    
    // 2. 所有CPU上恢复NPT到真实物理页
    for each CPU:
        RemapPage(cpu, gpa, originalPA, present=1, writable=1, noExecute=0);
        FlushTLB(cpu);
    
    // 3. 释放跳板
    ExFreePool(func->TrampolineAddr);
    
    // 4. 释放影子页
    FreeShadowPages(func->HookInfo);
    
    // 5. 释放HOOK结构
    ExFreePool(func);
}
```

---

## 十、隐藏你自己——Hypervisor的反侦察

做Hook的人最怕的不是Hook失效，而是被发现。NPT Hook的优势在于Hypervisor本身对Guest是不可见的，但前提是你要做好隐藏。

### 10.1 把私有内存映射到零页

Hypervisor的所有私有数据结构（VMCB、NPT页表、Host栈、影子页、Hook元数据、甚至你自己的代码段）都不应该被Guest访问到。最简单粗暴的方法：**把这些地址的NPT PTE指向一个全零页**。

这样Guest读这些地址读到全0（看起来是未分配内存），写触发NPF（注入#PF），执行触发NPF（注入#GP）。

```c
// 分配一个全零页
g_ZeroPage = AllocateZeroedPage();

// 隐藏某块内存
VOID HideMemory(PVOID va, SIZE_T size) {
    for each 4KB page in [va, va+size):
        pa = MmGetPhysicalAddress(va);
        for each CPU:
            RemapPage(cpu, pa, g_ZeroPagePA, 
                      present=1, writable=0, noExecute=1);
}
```

### 10.2 MSR拦截

Guest如果读EFER MSR，会看到EFER.SVME=1（因为SVM确实在运行），这直接暴露了虚拟化。处理方式：

- **拦截MSR_EFER的WRMSR**：强制保留SVME位（防止Guest关闭SVM）
- **拦截MSR_EFER的RDMSR**：返回值中清除SVME位，让Guest看到SVME=0
- **伪造MSR_VM_CR**：返回VM_CR.LOCK=1 | VM_CR.SVMDIS=1，模拟"BIOS锁定了SVM"
- **VM_HSAVE_PA MSR**：读返回0，写注入#GP（模拟SVME=0状态下的行为）

### 10.3 CPUID伪造

CPUID是最常见的虚拟化检测手段：
- CPUID leaf 1.ECX[31]（Hypervisor Present位）：返回0
- CPUID leaf 0x40000000-0x400000FF（Hypervisor Leaf）：不拦截的话会返回0（这其实是正常的——非HV环境下这些leaf也是0），但最好注入#GP
- 拦截CPUID指令需要在VMCB.InterceptMisc1中设置CPUID拦截位

### 10.4 SVM指令模拟

在Guest中执行VMRUN/VMSAVE/VMLOAD/VMMCALL等SVM指令时，由于我们已经让Guest看到SVME=0，这些指令应该触发#UD（无效操作码异常）。在VMMCALL拦截中：
- 先检查CPL是否为0（Ring0），不是则#GP
- 我们自己的VMMCALL功能码（如HOOK_DONE、RESET_SHADOWS）正常处理
- 其他未识别的VMMCALL：注入#UD（模拟真实硬件SVME=0时的行为）

### 10.5 一致性原则

反检测最重要的原则不是把某个特定的bit伪造好，而是**所有的返回值必须一致**。如果你让CPUID返回"SVM不支持"，那EFER.SVME必须返回0，VM_CR必须返回锁定状态，VMRUN执行必须触发#UD——只要有一个地方矛盾，检测方就能判断出有Hypervisor在运行。

---

## 十一、多核、TLB、中断——那些会让你蓝屏的坑

### 11.1 每CPU独立结构

每个CPU核心需要独立的：VMCB（Guest+Host）、HSAVE、Host栈、NPT页表。NPT的PML4和PDPT层可以共享（因为它们映射的是全局地址空间），但PD和PT层需要独立——因为同一时刻，CPU A可能在执行Hook（NPT指向影子页1），而CPU B在读同一函数（NPT指向影子页0）。

启动SVM时，通过`KeSetSystemAffinityThread`依次把线程绑定到每个CPU，在该CPU上执行VMRUN。

### 11.2 TLB一致性

每次修改NPT PTE后，必须刷新TLB。否则CPU可能使用缓存的旧翻译，导致：
- 切到影子页1后还在执行影子页0的内容（Hook不触发）
- 切回影子页0后还在执行影子页1的内容（暴露时间窗口变长）

AMD SVM通过VMCB中的TlbControl字段控制TLB刷新：
- 0x01：Flush当前ASID的所有TLB条目
- 0x03：Flush所有ASID的所有TLB条目
- 0x07：Flush当前ASID的所有TLB条目，包括全局页

设置TlbControl后，下次VMRUN时硬件自动完成刷新，不需要执行INVLPGA指令。每个CPU使用不同的ASID（GuestAsid = cpuIndex + 1），这样可以利用ASID标记避免不必要的全局刷新。

### 11.3 VMEXIT期间的中断

VMEXIT发生时，硬件自动禁用中断（相当于执行了CLGI）。在VmExitHandler执行期间，中断是关闭的——这意味着：
- VMEXIT处理必须尽快完成，不能做耗时操作（不能调用会Wait的内核函数，不能打印日志到串口等慢设备）
- 需要分配内存时只能用预分配的页池，不能调用ExAllocatePool
- VmExitHandler中不能持有自旋锁后再尝试获取另一个锁（死锁风险）

如果需要在VMEXIT中做复杂操作，应该把工作通过队列DPC/工作项延后到IRQL降低后执行。

### 11.4 NPF注入的Guest #PF

如果NPF不是Hook引起的（比如Guest真的发生了缺页），你需要向Guest注入一个#PF异常，让Guest的缺页处理函数来处理。事件注入通过VMCB.EventInj字段：

```c
EVENTINJ inj = {0};
inj.Vector = EXCEPTION_PF;  // 14
inj.Type = 3;               // Exception type
inj.Valid = 1;              // 注入有效
inj.ErrorCode = pfErrorCode; // 和正常#PF一样的错误码
vmcb->ControlArea.EventInj = inj;
```

VMRUN时硬件会自动把这个异常注入到Guest中，Guest的#PF处理函数正常执行，跟真实发生缺页一模一样。

### 11.5 电源状态

Windows休眠（S3）时CPU状态会丢失，唤醒后VMCB/NPT等状态无效。需要注册`\Callback\PowerState`回调：
- 休眠前：在每个CPU上通过VMMCALL让VM循环退出，清除EFER.SVME
- 唤醒后：重新初始化VMCB/NPT，重新进入VM

不处理电源状态的后果：唤醒后导致三重故障。

---

## 十二、不同Hook设计方案的对比与取舍

了解了基本方案后，你需要知道：NPT Hook并非只有一种实现方式，不同方案有不同的性能和隐蔽性。

### 方案1：影子页切换 + INT3（本文主讲的方案）

```
默认：NPT→影子页0(NX=1) → NPF取指 → 切影子页1(NX=0) → INT3 → #BP → 跳跳板 → 复位
```

- **优点**：实现简单，稳定，影子页1暴露窗口极短（单条INT3的时间），隐藏了跳板的位置。
- **缺点**：每次Hook触发产生两次VMEXIT（NPF + BP），开销略大。

### 方案2：影子页切换+jmp

不修改页内容，而是准备两套NPT：
- NPT-A（不可执行视图）：W=0，NX=1（禁止执行）
- NPT-B（读/执行视图）：被Hook页映射到含jmp的影子页，W=0，NX=0。

NPF时根据错误码切换VMCB.nCR3指向不同NPT根，利用ASID标记避免TLB flush。

- **优点**：不需要INT3，不需要BP拦截；TLB条目可以同时缓存两套翻译（因为ASID不同），减少flush开销
- **缺点**：实现更复杂（需要覆盖2到3条汇编指令，容易出错），且跳板位置暴露，存在隐蔽性问题。

---

## 十三、工程实现清单——从零开始要写哪些模块

以下是一个NPT Hook框架的完整模块清单，按照实现顺序排列：

### 阶段1：基础骨架（不做Hook，先让Hypervisor跑起来）

| # | 模块 | 内容 |
|---|------|------|
| 1 | CPUID检测 | 检测AMD/SVM/NPT/SVMDIS |
| 2 | 物理内存分配器 | 物理连续页分配（MmAllocateContiguousMemory）、页池（预分配+位图管理） |
| 3 | 汇编入口 | .asm文件：VM循环、VMRUN进入、VMEXIT保存/恢复寄存器、VMMCALL封装 |
| 4 | VMCB管理 | VMCB分配、Control Area初始化、State Save Area填充（读当前寄存器） |
| 5 | Host状态 | Host栈分配、Host VMCB、HSAVE、MSRPM/IOPM |
| 6 | NPT页表 | 四级页表构建（2MB大页恒等映射）、大页拆分、地址翻译、PTE修改、懒更新 |
| 7 | VMEXIT分发 | 基本的ExitCode switch-case，处理NPF（先全部注入Guest #PF）、MSR直通、SHUTDOWN蓝屏 |
| 8 | DriverEntry + 启动线程 | 异步启动、逐核绑定亲和性、StartSVM |
| 9 | 基础MSR拦截 | 拦截EFER写（保留SVME位） |
| 10 | 电源回调 | Sleep前Suspend、唤醒后Resume |

**里程碑**：系统在Hypervisor下稳定运行，功能跟没虚拟化一样。

### 阶段2：内存隐藏能力

| # | 模块 | 内容 |
|---|------|------|
| 11 | 零页隐藏 | SetNestedPageProtection(Hide=TRUE)映射零页 |
| 12 | PE段规划 | code_seg/data_seg把Hypervisor代码和数据分到专用段，4KB对齐 |
| 13 | 自身隐藏 | 把VMCB/NPT/Host栈/私有数据段/代码段全部Hide |

**里程碑**：Hypervisor自身代码和数据对Guest不可见。

### 阶段3：Hook框架

| # | 模块 | 内容 |
|---|------|------|
| 14 | Hook管理器 | HOOK_INFO/HOOK_FUNC结构、链表管理、AddHook/RemoveHook |
| 15 | 影子页管理 | CreateShadowPage（分配+memcpy+隐藏）、SetGuestShadowPage（切换NPT映射） |
| 16 | x64指令长度解码器 | Legacy prefix→REX→Opcode→ModRM/SIB→Immediate长度计算 |
| 17 | 跳板生成 | 汇编跳板模板、AllocateJmpTrampoline（分配+memcpy模板+填3个地址+复制序言） |
| 18 | #BP拦截 | HandleBP：匹配RIP、改写Guest RIP到跳板、复位影子页 |
| 19 | NPF Hook触发 | HandleNPF中Id=1分支：查找Hook、切到影子页1 |
| 20 | VMMCALL处理 | ResetShadows复位影子页 |
| 21 | 复位线程 | 高优先级系统线程，周期VMMCALL复位影子页0 |
| 22 | Hook安装/卸载API | InstallHook/UninstallHook完整流程 |

**里程碑**：可以Hook任意内核函数，读原函数看到干净字节，执行流被重定向。

### 阶段4：反检测与优化（可选，根据需求）

| # | 模块 | 内容 |
|---|------|------|
| 23 | CPUID伪造 | 拦截CPUID、清除Hypervisor位、伪造SVM状态 |
| 24 | EFER/RDMSR伪造 | 让Guest读EFER看到SVME=0 |
| 25 | SVM指令#UD | CPL split + SVME=0时VMRUN/VMSAVE等触发#UD |
| 26 | RDTSC虚拟化 | 拦截RDTSC，补偿VMEXIT时间差 |
| 27 | TLB优化 | ASID管理、减少不必要的全局flush |
| 28 | VMMCALL认证 | 防伪造的hypercall调用（cookie机制） |

---

## 十四、调试方法论

开发Hypervisor级别的代码，蓝屏是常态。以下是实战中总结的调试策略：

### 14.1 分阶段验证

不要一次写完所有模块再测试。按阶段1→2→3的顺序，每完成一个阶段就验证：
- 阶段1：能启动VM并稳定运行10分钟以上，运行各种软件不蓝屏
- 阶段2：在Guest中尝试读Hypervisor地址，确认读到0
- 阶段3：先Hook一个自己写的简单调用`DbgPrintEx`的函数，确认Hook触发和跳板正常，再Hook复杂内核函数。

### 14.2 双机内核调试

用两台物理机通过串口/网络/USB做内核调试（WinDbg）。Hypervisor的Bug几乎不可能在单机上调试——因为一旦蓝屏你就什么都看不到了。

设置方法：
- 被调试机：bcdedit /debug on; Set com port。
- 调试机：WinDbg → Attach to kernel → Connect com。

### 14.3 日志策略

VMEXIT中不能直接DbgPrint（太慢且在高IRQL）。使用环形缓冲区日志：
- 预分配一块4KB-16KB的非分页内存作为log buffer
- VMEXIT中写日志到buffer（无锁或per-CPU buffer）
- 在PASSIVE_LEVEL的线程中定期把buffer输出到DbgPrint
- 关键路径用DbgPrint直接打（启动阶段、Hook安装阶段，这些不在VMEXIT中）

### 14.4 常见Bug排查

| 症状 | 可能原因 |
|------|---------|
| VMRUN后立刻蓝屏、三重故障、无响应 | VMCB.StateSaveArea填错了（尤其是CR3、段寄存器、EFER.SVME）亦或者是处理特定VMEXIT时未更新Rip |
| 运行几秒后随机蓝屏 | NPT映射错误（比如漏了某些物理内存区域的映射） |
| Hook第一次触发正常，第二次蓝屏 | 影子页复位失败；跳板OriginalCode指令截断 |
| 多核环境一个核正常其他核蓝屏 | 每CPU结构没独立分配；NPT页表共享了不该共享的层 |
| 休眠唤醒后蓝屏 | 电源回调没正确Suspend/Resume |
| Hook后系统变卡 | VMEXIT太多（检查拦截位，不要拦截不需要的事件）；复位线程间隔太短 |
| Guest能读到Hypervisor内存 | Hide后忘了Flush TLB；某些页没有加入隐藏列表 |


---

## 十五、写在最后

### 技术的本质

NPT Hook之所以"无痕"，不是因为它用了什么魔法，而是因为它把Hook从"修改代码"这个维度搬到了"控制内存映射"这个维度。在传统模型里，代码和它所在的物理页是绑定的——你改了代码字节，任何人读那个地址都能看到。但在虚拟化模型里，"地址"和"物理内容"的绑定关系是Hypervisor可以随时切换的，而且这个切换发生在MMU硬件层面，Guest的任何软件操作都无法绕过。

这种思想——**多加一层抽象，在这层抽象里做手脚**——其实在计算机科学里无处不在。NPT Hook只是"加一层"思想在安全领域的一个具体应用。

### 性能代价

每次Hook触发涉及两次VMEXIT（NPF + BP），每次VMEXIT大约500-2000个CPU周期。对于不频繁调用的函数（进程创建、文件打开、注册表操作），这个开销完全可以忽略。但对于每秒调用数万次的函数（如`NtQueryInformationProcess`、某些图形函数），累计开销也可能会很显著，但整体影响可以在能接受的范围内。

### 为什么理解这些很重要

即使你不打算自己写一个Hypervisor，理解NPT Hook的原理也很有价值：
- 安全软件开发者需要知道攻击者可能用什么手段绕过你的检测
- 逆向工程师需要理解为什么有些代码"读起来和执行起来不一样"
- 内核开发者需要理解硬件虚拟化提供的能力边界
- 这种"控制映射关系而非内容"的思想可以启发很多其他场景的设计

### 延伸阅读

- **AMD Architecture Programmer's Manual Volume 2**: SVM/NPT的权威参考，所有实现细节都在这里
- **AMD64 Architecture Programmer's Manual Volume 3**: 指令集参考，手写反汇编引擎和跳板时必查
- **SimpleSvmHook/DdiMon**: 开源参考实现，读源码是最好的学习方式

---

*本文讲解的是技术原理和实现思路。请将所学用于正当的安全研究和系统开发。*
