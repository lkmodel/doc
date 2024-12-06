## 概述

FrameKernel的目标是同时达到微内核安全与宏内核的性能与效率。

包括两个部分OS Services和Framework：

<img src="./Asterinas FrameKernel学习记录.assets/image-20241129074830552.png" alt="image-20241129074830552" style="zoom:80%;" />

FrameKernel在设计上基于一个概念TCB，在保持TCB最小化的同时，提高整体的扩展性。

<img src="./Asterinas FrameKernel学习记录.assets/image-20241129075550413.png" alt="image-20241129075550413" style="zoom:80%;" />



## TCB概念

TCB (Trusted Computing Base)，为计算系统提高安全环境的必要的集合，形式上可能包括：软件、硬件、固件、内核、物理位置、内在安全控制措施、安全过程 ( Safety & Security Procedures) 等等。即TCB不只是软硬件，还有规范措施、方法过程等。

> TCB是一个超出内核范畴的更大的概念，FrameKernel只是借用了该概念。

<img src="./Asterinas FrameKernel学习记录.assets/image-20241129081458991.png" alt="image-20241129081458991" style="zoom:80%;" />

计算系统由TCB和其它部分构成：TCB内部包含的是对整个计算系统安全起到关键作用和有全局影响的部分；而其它部分如果被破坏，对整体安全只有局部和次要性影响。

TCB是整个计算系统安全的**基石**，它内部包含部分必须是高信任的。其它部分则是不被信任的，必须基于TCB来提供检查和保证，并且不能绕过TCB。

> 对信任的理解：
>
> 1. 我们必须首先相信某些部分是安全可靠的，以它们作为建立系统的起点。一般来说，必须对硬件平台、BIOS、BootLoader、工具链采取信任的态度。以信任的部分逐步验证确认其它的部分，最终达到确信整个系统安全可靠的目标。
> 2. 低特权级必须信任高特权级，因为超出低特权级范围的操作必须转入高特权级处理；相对的，高特权级天然的不相信低特权级。

TCB有个关键要素：**TCB边界**以及外部通过边界访问TCB服务的**路径控制**。

TCB边界是保护TCB的接口和调用入口，是必须精确设计和严格控制访问方式的，以防止被其它部分破坏或影响。



## TCB的特点

* TamperProof：外部不可修改TCB内部的代码和状态
* Not Bypassable：不存在任何绕过TCB去攻击计算系统的路径
* Verifiable：管理者有可行的手段去验证、确认TCB的正确性和安全性
* **Simple**：TCB相对于计算系统必须是简单的，和相对容易实现的

其它三点显而易见，但Simple是必须注意的部分，现实中经常陷入的误区。



## TCB体现的设计原则

TCB体现了工程领域中对 收益/代价成本 的关注。一定条件下能够投入的资源有限，所以必须区分哪些部分是对整体安全目标起到关键核心作用的部分，把它们识别出来重点关注和设计，作为TCB；然后对TCB进行保护，并设计机制让TCB成为系统中不可绕过的基础部分，使得基于它能够推导、验证、保证其它部分的安全性。以较小的代价获得最大的收益。



## FrameKernel中的TCB

FrameKernel在论文阐述同类对比时和在设计时借用了TCB的概念。

**FrameWork就是它的TCB部分**。

采用TCB概念应该有如下目的：

1. 强调它相对其它内核设计的优势

   FrameKernel的优势是，保持TCB较小的同时，还能基于API接口对上层提供良好的可描述能力（Expressible）。

   即它能够做到：随着内核的演进，它的TCB不会随着功能和代码规模扩张而变大变复杂。而其它内核设计达不到该效果。

2. 借用TCB说明其设计理念

   论文中强调了TCB最小化的设计原则，并针对该原则提出如何达到最小化的具体措施。

3. 对于TCB边界的重视

   在设计上首先区分哪些属于TCB内部，然后对于可以暴露给Services的部分重点进行了接口设计，以防止破坏TCB内部。



## FrameKernel的整体设计

<img src="./Asterinas FrameKernel学习记录.assets/image-20241129092553028.png" alt="image-20241129092553028" style="zoom:80%;" />

FrameKernel主要参照微内核的设计架构，主要区别是面向Services的边界机制不同。它没有采用传统的基于硬件特权级进行隔离的方式，借助类型安全语言Rust来建立边界，这点与Thesues的方式类似。由此既建立了隔离保护，又能保证效率。

### 设计目标

1. Soundness：基于Framework界面提供的API + Rust SafeMode编程，就不会触发未定义行为。工具链在编译阶段就能保证。
2. Expressive：面向Services的 API支持内核开发的表达能力足够。
3. Minimalism：最小化TCB。
4. Efficiency：充分利用Rust的零开销的抽象并扩展其理念和机制。

### 具体设计

<img src="./Asterinas FrameKernel学习记录.assets/image-20241129095442365.png" alt="image-20241129095442365" style="zoom:80%;" />

五个方面的设计：

1. 内存分为untyped 和 typed

   主要从安全和可表达能力两个方面考虑了分类处理。

   * 有类型内存 - 主要是内核的代码段、栈段和Heap段等等，这些段内会保存具有类型安全保证的对象。不能对TCB外暴露，更不能支持指针操作，因为这样可能破坏内核的内存安全。
   * Untyped内存：分给用户空间的和分给DMA的内存，Framework(TCB)对它们天然不信任，它们也不会存放内核安全类型的对象。允许它们通过指针随便修改，这样写坏了也不会影响内核和其它进程的安全。允许指针则提高了表达能力。

2. CPU state分为用户态的和内核态

   用户态的部分CPU寄存器状态是允许被查看和修改的，这个不影响系统安全。

   需要完全保护在TCB内部的只是内核态CPU state。

3. 对任务区别调度策略和上下文切换

   与我们的设想类似，调度本身不是内核专有的部分，其实只是针对调度单元的算法，所以可以暴露给services实现。

   只是上下文切换 需要保护在TCB之内。

4. 异常/中断的分阶段处理

   类似Linux的对中断处理的设计，分为前半段和后半段，前半段涉及对系统级寄存器的直接修改，影响整个系统安全，必须包含在TCB内部，而后半段其实主要是业务处理，不必要采取UnsafeMode编程，所以分离到Services中。

   <img src="./Asterinas FrameKernel学习记录.assets/image-20241129100058162.png" alt="image-20241129100058162" style="zoom:80%;" />

5. 对系统级设备和普通外设的区别对待

   系统设备就是IOMMIO和APIC之类可能破坏系统安全的设备。对它们的操作保护在TCB之内。

   Framework强制要使用IOMMU，以保证TCB对应Framework安全。

   其它普通外设，如网卡、显卡、存储之类，都没有全局破坏能力，按照受控接口暴露给Services处理。



## OSTD与RustSTD的对比

在新版本里面，Frameworkd改称OSTD，即OS STD，相当于应用开发领域的Rust Std库。

OSTD是跨越安全与非安全边界的，是Safe和Unsafe层之间的中介。

## OSTD优势

1. 降低OS开发门槛

   给OS开发者一个基本可运行的东西，可以以它为基础做扩展。

2. 提高内存安全性

   封装低级的、error-prone、arch-specific的部分，支持Rust规则。

3. 提高OS间组件的移植能力

   基于ostd相当于应用级开发中基于 std，可以提高OS各模块或组件之间的可移植性。

4. 支持用户态开发user-mode development

   在用户态开发如驱动、文件系统等，提高开发效率。OSTD后端支持Linux、逻辑和虚拟机。



## FrameKernel设计在实现层面的体现

之前FrameKernel的framework和Services的分层，在新版本的代码实现中，体现为ostd和kernel实现。分层设计主要解决的矛盾：

对于计算机系统底层机制的安全封装，基于封装接口对上层提供接口，让主要的内核开发工作都在上层完成，这在原理上可以提高整体的安全性；但是，接口对底层的封装势必影响上层对底层机制的控制能力。即，基于接口封装获得的安全性与通过接口获得的访问能力，是一对矛盾，无法兼得。这也是论文强调的两点：Sound和Expressive的平衡问题。

<img src="./Asterinas FrameKernel学习记录.assets/image-20241206091114992.png" alt="image-20241206091114992" style="zoom:80%;" />

FrameKernel的思路是，对各类内核资源进行分析，区分出哪些是全局性的、关键性的资源，哪些是局部性的，非关键的资源。关键的完全封装到framework或称ostd中，在services或称kernel实现中是无须也不应该直接接触这些资源的，如此形成严密的保护；而对于局部性的、非关键的资源，主要是指与应用进程相关的、与非关键外设相关的资源，很大程度上放宽了对它们操作的限制，以此为kernel层面的开发提供更强的表达能力。

此外，framework(ostd)和services(kernel)之间不是基于特权级的隔离，仍然都在内核特权级。主要是借助了Rust语言等在类型安全方面的封装，来限制上层对下层的访问，所以效率上的影响较小。

### 1. 内存分为untyped 和 typed

对于分配给应用或DMA的页面，作为untyped内存，它们不包含系统关键的、类型化的资源对象，即使损坏也只会影响自己，不会整个系统的安全。所以，对于untyped的内存页，**允许**指针操作。

示例：kernel实现stat系统调用。

```rust
pub fn sys_fstat(fd: FileDesc, stat_buf_ptr: Vaddr, ctx: &Context) -> Result<SyscallReturn> {
    ... ...
    ctx.user_space().write_val(stat_buf_ptr, &stat)?;
    ... ...
}
```

第3行，允许直接对应用的地址空间直接写入值。并且，可以看出是直接针对指针的操作。

这样kernel的开发将十分方便，且保证了不会对系统整体安全造成影响。



### 2. CPU state分为用户态的和内核态

与内存情况类似，大多数仅属于应用进程本身的CPU状态，其实可以允许直接访问和修改，即使损坏也只会影响自己。这样就可以提高上层开发的灵活性和控制力。

示例：kernel实现execve系统调用。

```rust
fn do_execve(
    elf_file: Dentry,
    argv_ptr_ptr: Vaddr,
    envp_ptr_ptr: Vaddr,
    ctx: &Context,
    user_context: &mut UserContext,
) -> Result<()> {
    ... ...
    // set cpu context to default
    *user_context.general_regs_mut() = RawGeneralRegs::default();
    user_context.set_tls_pointer(0);
    *user_context.fpu_state_mut() = FpuState::default();
    // FIXME: how to reset the FPU state correctly? Before returning to the user space,
    // the kernel will call `handle_pending_signal`, which may update the CPU states so that
    // when the kernel switches to the user mode, the control of the CPU will be handed over
    // to the user-registered signal handlers.
    user_context.fpu_state().restore();
    // set new entry point
    user_context.set_instruction_pointer(elf_load_info.entry_point() as _);
    debug!("entry_point: 0x{:x}", elf_load_info.entry_point());
    // set new user stack top
    user_context.set_stack_pointer(elf_load_info.user_stack_top() as _);
    ... ...
}
```

第10~22行，可以直接对用户态的CPU各个状态寄存器进行访问甚至修改。



### 3. 对任务区别调度策略和上下文切换

上下文切换的概念和机制完全封装在底层，上层看不到。但是上层可以基于一个Trait定制调度策略，让底层来调用。

示例：kernel不能直接干预，也看不到上下文切换这种底层机制；但是它可以实现调度策略，让底层在适当时机调用。

kernel默认实现了一个调度策略PreemptScheduler。

```rust
impl<T: Sync + Send + PreemptSchedInfo + FromTask<U>, U: Sync + Send + CommonSchedInfo> Scheduler<U>
    for PreemptScheduler<T, U>
{
    fn enqueue(&self, task: Arc<U>, flags: EnqueueFlags) -> Option<CpuId> {
        ... ...
    }
    ... ...
}
```

Trait Scheduler是由ostd提供，来自ostd::task::scheduler::Scheduler。也即，这个Trait相当于一个回调函数，沟通了上下两层，完成了隔离和分工。



### 4. 异常/中断的分阶段处理

从实现上区分为TopHalf阶段和BottomHalf阶段，TopHalf封装在底层，而BottomHalf暴露给上层实现。

示例：ostd提供注册BottomHalf的方式，而kernel可以调用它，注册后半部的处理函数。

```rust
pub fn register_bottom_half_handler<F>(func: F)
where
    F: Fn() + Sync + Send + 'static,
{
    BOTTOM_HALF_HANDLER.call_once(|| Box::new(func));
}
```



### 5. 对系统级设备和普通外设的区别对待

底层即ostd中封装了对系统级设备IOMMU的操作，对上层不暴露。但是上层即kernel有能力对其它各种普通外设可以通过较为宽松的接口进行访问和处理。

示例：kernel层面开发各种外设驱动，可以类似应用开发那样直接对分配给自己的虚拟地址范围做访问和修改。





## 引导过程

以下以Riscv64体系架构为主进行分析。

引导主要分为3个阶段：

### 1. 一级引导(基于体系结构)

基于汇编代码，从0x8020_0000体系结构默认的内核入口开始执行。

```rust
// arch/riscv/boot/mod.rs
global_asm!(include_str!("boot.S"));
```

#### 1.1 启用分页，进入虚拟地址空间

启用早期页表，采取SV48模式。

early页表两级：boot_pagetable和boot_pagetable_2nd。

根页表boot_pagetable的最高项（顶端512G空间）-> 二级页表boot_pagetable_2nd

<img src="./Asterinas FrameKernel学习记录.assets/image-20241205095806677.png" alt="image-20241205095806677" style="zoom:80%;" />

总的地址空间256T，平分为高端和低端两个部分。每个部分的第一个页表项直接映射前512G的物理内存空间，即通过两个虚拟地址范围

1. 0x0000_0000_0000_0000开始的512G
2. 0xffff_8000_0000_0000开始的512G

都可以直接访问到最大512G的物理内存空间，对于多数平台都足够了。

根页表的最高项指向二级页表，二级页表的第510项指向0x8020_0000开始的1G空间，即内核所在的1G空间，可以通过0xffff_ffff_8020_0000虚拟地址访问。



地址空间布局的lds文件./osdk/src/base_crate/riscv64.ld.template。其中内容决定了内核各个段的布局情况。



> 伪指令la、lga以及lla
>
> lga以全局的绝对寻址方式获取目标变量的地址；lla以局部的相对PC寻址的方式获取目标变量的地址。
>
> la是根据代码的编译选择情况，自动选择lga或者是lla。
>
> 从实践看，直接调用la就可以。

#### 1.2 初始化内核栈

初始化64K的内核栈，随后可以调用函数，这是为进入Rust编码做准备。该内核栈仅针对BSP。

#### 1.3 跳转到下一级入口riscv_boot，从汇编进入到Rust世界

riscv_boot就已经是基于Rust编码了。纯粹汇编阶段的代码比较精简，体现了降低非安全代码的设计思路。



### 2. 二级引导(体系结构相关)

由riscv_boot负责继续引导，现在可以使用Rust语言编写代码。流程包括三步：

#### 2.1 初始化设备树

对于riscv体系结构，获取系统配置的主要方式是解析设备树。

此处基于从内核入口传入的设备树内存块指针，初始化一个设备树实例。

以下各个获取配置的方法，几乎都是从设备树中获得。

#### 2.2 注册引导阶段的回调函数

抽象了各个体系结构具有共性的需要获取的系统配置信息，分为如下6个方面：

1. bootloader的名称
2. 内核命令行
3. 是否有initramfs以及具体信息
4. acpi的参数
5. 由bootloader提供的数据缓冲区
6. 物理内存的总量、各个分段起止范围

第6项是本阶段的关键，所有体系结构都需要该项，它指示了后面如何进行内存管理子系统的初始化，是整个系统初始化的核心部分。

对于Riscv64，第2项和第6项是通过设备树解析获取，其余各项其实基本没有用。

#### 2.3 调用体系结构无关的引导主函数call_ostd_main进入下一阶段

从体系结构相关的代码转入体系结构无关的代码。



### 3. 三级引导(体系结构无关)

引导的最后统一阶段，此时，x86和riscv64会汇集到统一的引导过程call_ostd_main，包括2步：

#### 3.1 调用上阶段注册的回调函数

按照既定流程逐个调用体系结构相关的引导阶段准备的初始化函数。

这样从架构设计上，比较好的区分了体系结构相关和无关代码。

#### 3.2 从ostd转入services阶段调用内核的主函数继续引导过程

至此ostd完成了它的引导工作，此后就是kernel即services层次的引导。

kernel的入口主函数main使用ostd::main的marker来标记。

从设计上，这是仿照了Rust应用开发的模式。ostd起到了类似std标准库的作用，而开发者只需要基于它去开发自己的内核。
