# x86_vcpu

## 介绍

定义 x86_64 架构的 vCPU 结构和虚拟化相关接口支持。

crate 用户必须使用 [`crate_interface::impl_interface`](https://crates.io/crates/crate_interface) 实现 `PhysFrameIf` trait，以提供 `PhysFrame` 分配和释放的低级实现，相关实现可参考 [ArceOS](https://github.com/arceos-org/arceos/blob/main/modules/axhal/src/paging.rs)。

## 架构

### `x86_vcpu/src/regs.rs`

我将为您提供关于x86_64架构通用寄存器代码的详细中文解析。
这段代码定义了一个用于处理x86_64架构通用寄存器的Rust结构和相关功能。我将分部分进行解析：

#### 结构定义

`GeneralRegisters`结构体用于表示和存储x86_64架构中的通用寄存器值：

+ 使用`#[repr(C)]`确保内存布局与C语言兼容
+ 实现了`Debug`、`Default`和`Clone`特性
+ 包含16个寄存器，每个都是64位无符号整数(u64)
+ 特别注意·是保留的，不直接操作RSP寄存器(栈指针)

#### 寄存器功能
代码中详细注释了各寄存器的常见用途：

+ `RAX`: 通常用于函数返回值
+ `RCX`: 常用作循环计数器
+ `RDX`: 常用于I/O操作
+ `RBX`: 用作基址指针或存储跨函数调用的值
+ `RBP`: 常用作函数调用中的帧指针
+ `RSI`: 在字符串操作中用作源索引
+ `RDI`: 在字符串操作中用作目标索引
+ `R8-R15`: 64位模式下的额外通用寄存器

#### 方法实现

结构体实现了两个关键方法:
1. `get_reg_of_index`: 根据索引获取指定寄存器的值
    + 索引范围为0-15(除4外)
    + 使用match语句映射索引到对应寄存器
    + 非法索引会导致panic
2. `set_reg_of_index`: 设置指定索引对应寄存器的值
    + 与get方法类似的索引映射
    + 同样会对非法索引panic

#### 宏定义

代码最后定义了两个汇编宏：
1. `save_regs_to_stack`: 将所有通用寄存器按特定顺序压入栈中
    + 注意压栈顺序是从R15到RAX
    + RSP位置用`sub rsp, 8`保留了空间
2. `restore_regs_from_stack`: 从栈中恢复所有寄存器
    + 顺序与压栈相反，从RAX到R15
    + RSP位置用`add rsp, 8`跳过
这些宏可能用于函数调用中保存和恢复寄存器状态，特别是在需要保存全部寄存器上下文的场景，如中断处理或上下文切换。
整体而言，这是一个用于x86_64寄存器管理的完善Rust实现，具有良好的注释和错误处理，适合在需要直接操作寄存器的底层系统编程中使用。

### `x86_vcpu/src/msr.rs`

这段代码是关于x86架构中模型特定寄存器(MSR)的Rust实现。以下是详细解析：

#### MSR枚举定义

代码首先定义了一个Msr枚举类型，表示各种x86 MSR寄存器：

+ 使用`#[repr(u32)]`确保枚举值被表示为32位无符号整数
+ 实现了`Debug`、`Copy`和`Clone`特性
+ 通过`#[allow(non_camel_case_types, dead_code)]`忽略了Rust命名规范警告和未使用代码警告

枚举包含多种MSR寄存器，可分为几大类：

1. 基本控制寄存器
    + `IA32_FEATURE_CONTROL`: 控制特定CPU功能的启用
    + `IA32_PAT`: 页属性表
2. VMX相关寄存器（虚拟化扩展）
    + 从`IA32_VMX_BASIC`到`IA32_VMX_TRUE_ENTRY_CTLS`的一系列寄存器
    + 用于控制和配置Intel VT-x虚拟化技术
3. 扩展功能寄存器
    + `IA32_XSS`: 扩展状态保存
4. 系统调用和长模式相关寄存器
    + `IA32_EFER`: 扩展特性启用寄存器
    + `IA32_STAR`, `IA32_LSTAR`, `IA32_CSTAR`: 系统调用目标地址
    + `IA32_FMASK`: RFLAGS掩码
5. 段基址寄存器
    + `IA32_FS_BASE`, `IA32_GS_BASE`: FS和GS段基址
    + `IA32_KERNEL_GSBASE`: 内核GS基址

#### 方法实现

`Msr`枚举实现了两个核心方法:

1. `read`: 读取指定`MSR`的64位值
    + 内联实现，调用底层`rdmsr`函数
    + 将枚举值转换为对应的`MSR`编号
2. `write`: 写入64位值到指定`MSR`
    + 标记为`unsafe`，因为写`MSR`可能导致系统状态不安全变更
    + 调用底层`wrmsr`函数
    + 提醒调用者必须确保写操作不会导致不安全的副作用

#### trait实现

代码定义了一个`MsrReadWrite` trait，用于更高级的`MSR`操作抽象：

+ 包含一个关联常量`MSR`，类型为`Msr`
+ 提供默认的`read_raw`方法读取关联的`MSR`
+ 提供默认的`write_raw`方法写入关联的`MSR`
+ 这种设计使得特定`MSR`可以实现此trait，获得更具体的读写功能

这种实现允许开发者以类型安全的方式操作MSR寄存器，特别适合于操作系统内核、虚拟机监视器、驱动程序等需要直接访问硬件寄存器的系统级软件。整体代码结构清晰，并通过适当的安全标记提醒开发者MSR操作的潜在风险。

### `x86_vcpu/src/frame.rs`

这段代码定义了一个用于管理物理内存页面的Rust实现。我将为您详细解析各个部分：

#### 核心设计

`PhysFrame<H>` 结构表示一个4KB大小的连续物理内存页面，具有以下特点：

+ 使用泛型参数 H: AxVCpuHal 支持不同的硬件抽象层实现
+ 实现了自动内存管理 - 当对象销毁时会自动释放物理页面
+ 支持零初始化和未初始化的页面分配

#### 结构定义

``` rust
pub struct PhysFrame<H: AxVCpuHal> {
    start_paddr: Option<HostPhysAddr>,
    _marker: PhantomData<H>,
}
```

+ `start_paddr`: 物理页面的起始地址，使用 `Option` 表示可能未初始化
+ `_marker`: 零大小的 `PhantomData` 字段，用于标记泛型参数 `H` 的使用

#### 方法实现

1. 页面分配方法:
    + `alloc()`: 分配一个新的物理页面，如果分配失败则返回错误
    + `alloc_zero()`: 分配并用零填充的页面
    + `uninit()`: 创建未初始化的页面对象（标记为 unsafe）

2. 访问方法:
    + `start_paddr()`: 获取页面的物理起始地址
    + `as_mut_ptr()`: 将物理地址转换为可变虚拟地址指针
    + `fill()`: 用指定字节值填充整个页面

#### Drop实现

通过实现 `Drop` trait，确保页面在对象销毁时自动释放:
``` rust
impl<H: AxVCpuHal> Drop for PhysFrame<H> {
    fn drop(&mut self) {
        if let Some(start_paddr) = self.start_paddr {
            H::dealloc_frame(start_paddr);
            debug!("[AxVM] deallocated PhysFrame({:#x})", start_paddr);
        }
    }
}
```

+ 只有当 `start_paddr` 不为 `None` 时（表示页面已分配）才会释放
+ 使用 HAL 的 `dealloc_frame` 方法释放物理页面
+ 输出调试信息记录页面释放

这种设计非常适合操作系统内核或虚拟机监视器中的内存管理子系统，提供了安全且灵活的物理内存分配机制。

### `x86_vcpu/src/ept.rs`

这段代码定义了一个名为`GuestPageWalkInfo`的结构体，用于存储和处理客户机页表遍历的相关信息。我来详细解析这个结构体的各个字段和用途：

#### 结构体概览

`GuestPageWalkInfo`结构体被用于虚拟化环境中，特别是在虚拟机监视器(VMM)或虚拟化层实现内存地址转换和页表操作时。它实现了`Debug`特性，便于调试和日志记录。
字段详解

+ `top_entry`: usize - 顶层页表结构条目的物理地址，是页表遍历的起点
+ `level`: usize - 客户机页表级别，表示页表的层次结构（如x86架构中可能是2级、3级或4级页表）
+ `width`: u32 - 页表宽度，表示地址位数（如32位或64位）
+ `is_user_mode_access`: bool - 标记是否允许用户模式访问，对应页表项的U/S位
+ `is_write_access`: bool - 标记是否允许写访问，对应页表项的R/W位
+ `is_inst_fetch`: bool - 标记是否允许指令获取，与执行权限相关
+ `pse`: bool - 对于32位分页，表示CR4.PSE（页大小扩展）标志；对于PAE/4级分页，总是为true
+ `wp`: bool - 控制器寄存器CR0.WP（写保护）标志，影响特权模式下的写操作行为
+ `nxe`: bool - MSR_IA32_EFER_NXE_BIT标志，启用不可执行页功能
+ `is_smap_on`: bool - 指示是否启用SMAP（管理员模式访问保护），防止特权代码访问用户空间内存
+ `is_smep_on`: bool - 指示是否启用SMEP（管理员模式执行保护），防止特权代码执行用户空间代码

### `x86_vcpu/src/vmx/mod.rs`

`has_hardware_support()` 函数：
+ 使用`raw_cpuid`库检测处理器是否支持VMX功能
+ 通过获取特性信息并检查`has_vmx()`来判断

`read_vmcs_revision_id()` 函数：

+ 读取VMCS修订ID，这对于正确初始化VMCS结构至关重要
+ 使用VmxBasic::read()获取基本VMX信息

`as_axerr()` 函数：
+ 将x86 VMX错误类型(VmFail)转换为自定义错误类型(AxError)

处理两种失败情况：

+ VmFailValid: VMCS有效但操作失败，返回具体指令错误信息
+ VmFailInvalid: VMCS指针无效

### `x86_vcpu/src/vmx/definitions.rs`

这段代码提供了一个用于处理VMX（Intel虚拟化技术）指令错误和退出原因的实现。我来为您详细解析这些内容：

#### VMX指令错误处理

`VmxInstructionError`结构定义了Intel处理器VMX指令可能的错误编号和描述：

+ 使用简单的封装结构存储错误代码（u32值）
+ 提供as_str方法将错误代码转换为描述性错误信息
+ 包含详尽的错误描述，参考了Intel SDM手册第3C卷第30.4节
+ 实现了From<u32>转换和Debug特性，方便使用和调试输出

#### VMX退出原因枚举

代码使用了`numeric_enum_macro`宏来定义VmxExitReason枚举，包含了几乎所有可能的VM退出原因：

+ 代表Intel VMX中所有可能的虚拟机退出原因
+ 每个变体都有详细注释说明其触发条件
+ 覆盖了从异常、中断到特定指令执行和状态转换的所有场景

主要分类包括：

+ 中断和异常类（如EXCEPTION_NMI, EXTERNAL_INTERRUPT）
+ 敏感指令类（如CPUID, HLT, INVD）
+ VMX操作类（如VMCALL, VMLAUNCH）
+ 资源访问类（如CR_ACCESS, MSR_READ）
+ 高级功能类（如EPT_VIOLATION, APIC_ACCESS）

#### 中断类型处理

`VmxInterruptionType`枚举和相关方法处理VM出入口中断信息字段的中断类型：

定义了8种中断/异常类型，对应Intel SDM中的定义
提供了几个辅助方法：

+ `vector_has_error_code`: 判断特定中断向量是否携带错误码
+ `from_vector`: 根据中断向量确定中断类型
+ `is_soft`: 判断是否为软件生成的中断/异常

### `x86_vcpu/src/vmx/instructions.rs`

具体是 INVEPT 指令 的封装。INVEPT 是 Intel VT-x 技术中的一个指令，用于 无效化（Invalidate） 由 扩展页表（EPT，Extended Page Table） 派生的 转换后备缓冲区（TLB，Translation Lookaside Buffer） 和 页表结构缓存。

这段代码实现了Intel VT-x虚拟化中的INVEPT指令操作。我来详细解析其组成部分和功能：
主要组件
1. VMX状态捕获函数
vmx_capture_status函数用于捕获VMX指令执行后的状态：

使用#[inline(always)]保证内联，避免函数调用导致RFLAGS寄存器被破坏
通过读取RFLAGS寄存器确定指令执行结果
根据Intel手册规范将标志位转换为对应的Result:

ZF=1 表示有效失败 (VmFailValid)
CF=1 表示无效失败 (VmFailInvalid)
两者都为0表示成功

2. INVEPT类型枚举
InvEptType枚举定义了INVEPT指令的操作类型：

SingleContext(1) - 只使EPT指针(EPTP)的特定映射失效
Global(2) - 使所有EPTP关联的映射失效

3. INVEPT指令包装
invept函数封装了INVEPT指令的调用：

接收无效化类型和EPT指针作为参数
创建INVEPT描述符(两个64位值组成的数组)
使用内联汇编直接执行INVEPT指令
调用vmx_capture_status捕获执行结果

### `x86_vcpu/src/vmx/percpu.rs`

这段代码实现了Intel VT-x虚拟化技术中的每CPU核心状态管理。我将为您详细解析：
VmxPerCpuState结构
VmxPerCpuState<H>结构表示运行在VMX模式下的每个CPU核心状态：

使用泛型参数H: AxVCpuHal支持不同硬件抽象层实现
包含两个关键字段：

vmcs_revision_id: VMCS版本标识符，确保软件与硬件VMCS兼容
vmx_region: 存储VMX操作所需的内存区域(VMXON区域)



AxArchPerCpu实现
该结构实现了AxArchPerCpu trait，提供了VMX功能的生命周期管理：

new方法：

创建一个未初始化的VMX状态实例
初始化VMCS修订ID为0
创建未初始化的VMX区域


is_enabled方法：

通过检查CR4寄存器的VMXE位判断VMX是否已启用


hardware_enable方法：

执行启用VMX所需的一系列步骤：

检查硬件是否支持VMX
确保VMX尚未启用
启用XSAVE/XRSTOR功能
配置并检查IA32_FEATURE_CONTROL MSR，确保允许VMX操作
验证CR0和CR4寄存器的值是否符合VMX要求
检查VMCS版本和内存类型等VMX基本参数
分配并初始化VMXON区域
设置CR4.VMXE位并执行VMXON指令


返回成功或详细的错误信息


hardware_disable方法：

执行禁用VMX所需的步骤：

确保VMX当前已启用
执行VMXOFF指令
清除CR4.VMXE位
释放VMXON区域

### `x86_vcpu/src/vmx/structs.rs`

这段代码实现了Intel VT-x虚拟化技术中的几个重要数据结构。我来详细解析各个部分：
VMX区域管理
`VmxRegion<H>`结构表示VMCS/VMXON区域（4KB大小）：

内部包含一个`PhysFrame<H>`用于分配和管理物理内存
提供uninit方法创建未初始化实例
new方法分配物理内存并初始化区域，设置修订ID和影子指示器
phys_addr方法返回区域的物理地址

I/O位图管理
`IOBitmap<H>`结构管理I/O端口虚拟化：

包含两个物理页帧：A位图(0-0x7FFF端口)和B位图(0x8000-0xFFFF端口)
提供`passthrough_all`和`intercept_all`方法分别允许或拦截所有I/O端口
`set_intercept`方法控制特定端口的拦截状态
`set_intercept_of_range`方法批量设置一系列端口

MSR位图管理
`MsrBitmap<H>`结构控制MSR寄存器虚拟化：

使用一个物理页帧存储位图信息
包含四个区域：低MSR读/写位图(0-0x1FFF)和高MSR读/写位图(0xC0000000-0xC0001FFF)
提供set_read_intercept和set_write_intercept方法控制特定MSR访问的拦截

VMX基本能力和特性控制

VmxBasic结构解析IA32_VMX_BASIC MSR：

包含VMCS修订ID、区域大小、地址宽度等关键信息
实现MsrReadWrite trait实现MSR读取功能
read方法将原始MSR值解析为结构化数据


FeatureControl结构和FeatureControlFlags管理IA32_FEATURE_CONTROL MSR：

控制VMX操作的启用和锁定状态
提供安全的读写方法，保留MSR中的保留位



EPT指针管理
EPTPointer结构封装扩展页表指针(EPTP)：

使用bitflags定义各种EPT配置选项
包含内存类型、页行走长度、访问/脏标志等设置
from_table_phys方法从PML4表物理地址创建EPTP，设置合适的默认值

### `x86_vcpu/src/vmx/vcpu.rs`

这段代码实现了Intel VMX技术的VCPU核心功能，涵盖了虚拟机执行环境的初始化、运行和退出处理。我将为您深入解析代码结构和关键功能。
主要结构定义
`VmxVcpu<H>`是代码的核心结构，表示一个虚拟CPU实例：

使用泛型参数`H: AxVCpuHal`支持不同硬件抽象层
包含关键字段：

`guest_regs`: 客户机通用寄存器状态
`vmcs`: 虚拟机控制结构区域
`io_bitmap`/msr_bitmap: I/O和MSR访问控制位图
`xstate`: 扩展状态管理(XSAVE相关)
`ept_root`: 扩展页表根指针



XState结构管理处理器扩展状态(XCR0/XSS)，维护主机和客户机的状态。
VmCpuMode枚举表示处理器的运行模式，包括实模式、保护模式、兼容模式和64位模式。
关键功能实现
1. VCPU创建与设置

new(): 创建新的VCPU实例，初始化VMCS和位图
setup(): 配置VCPU入口点和EPT根
setup_vmcs_XXX(): 一系列函数配置VMCS的不同部分(客户机状态、控制字段、主机状态)

2. VMX操作核心

vmx_launch()/vmx_resume(): 使用内联汇编实现VMX入口机制
vmx_exit(): VM退出后的处理入口
inner_run(): 执行VM入口，处理VM退出

3. 虚拟化支持功能

get_cpu_mode(): 判断客户机当前运行模式
queue_event(): 设置待注入中断/异常
inject_pending_events(): 在VM入口前注入待处理事件
set_io_intercept_of_range(): 配置I/O端口拦截
set_msr_intercept_of_range(): 配置MSR访问拦截

4. VM退出处理

exit_info(): 获取VM退出信息
builtin_vmexit_handler(): 处理基本VM退出
处理函数如handle_cpuid()、handle_xsetbv()等针对特定指令的处理

5. AxArchVCpu特性实现
该结构实现了AxArchVCpu trait，提供了标准接口用于:

创建和设置VCPU
绑定/解绑VCPU到当前处理器
执行VM并处理退出事件
设置通用寄存器

### `x86_vcpu/src/vmx/vmcs.rs`

这段代码实现了对VMX(虚拟机扩展)VMCS(虚拟机控制结构)字段的操作和管理。我将提供一个详细的中文解析。
整体架构
这段代码定义了VMCS字段的访问接口，包括各种枚举类型和相关功能，严格遵循Intel软件开发手册(SDM)的规范。主要包括以下部分：

VMCS字段枚举：将所有VMCS字段按类别和宽度组织成枚举类型
读写宏：实现对VMCS字段的读取和写入功能
控制函数：设置VMCS控制位字段的辅助函数
事件处理：处理VM退出和事件注入的相关功能

关键组件详解
VMCS字段枚举
代码按照Intel手册定义了8类VMCS字段枚举：

控制字段：VmcsControl16、VmcsControl32、VmcsControl64、VmcsControlNW
客户机状态字段：VmcsGuest16、VmcsGuest32、VmcsGuest64、VmcsGuestNW
主机状态字段：VmcsHost16、VmcsHost32、VmcsHost64、VmcsHostNW
只读数据字段：VmcsReadOnly32、VmcsReadOnly64、VmcsReadOnlyNW

每个枚举成员都对应一个具体的VMCS字段，其值为对应的VMCS字段编码。
字段访问宏
通过宏实现统一的字段读写功能：

vmcs_read!：实现不同宽度字段的读取
vmcs_write!：实现不同宽度字段的写入
define_vmcs_fields_ro/define_vmcs_fields_rw：为枚举类型添加读写方法

这些宏处理了32位和64位环境下的兼容性问题，确保正确处理寄存器宽度的差异。
事件和信息结构
定义了VM退出相关的信息结构：

VmxExitInfo：保存VM退出的基本信息，如退出原因、指令长度等
VmxInterruptInfo：描述中断或异常的详细信息
VmxIoExitInfo：I/O指令导致的VM退出的详细信息
CrAccessInfo：控制寄存器访问产生VM退出的详细信息

核心功能函数

控制字段设置：set_control函数实现了安全设置VMCS控制字段的功能，考虑了能力MSR的约束
EPT指针设置：set_ept_pointer函数用于设置和刷新扩展页表
VM退出信息获取：

exit_info：获取VM退出基本信息
interrupt_exit_info：获取中断类VM退出信息
io_exit_info：获取I/O指令VM退出信息
ept_violation_info：获取EPT违规VM退出信息


事件注入：inject_event函数实现向客户机注入中断或异常
CR访问处理：cr_access_info函数解析控制寄存器访问信息
EFER更新：update_efer函数用于处理长模式相关的EFER寄存器更新

## Example

``` 
use x86_vcpu::PhysFrameIf;

struct PhysFrameIfImpl;

#[crate_interface::impl_interface]
impl axvm::PhysFrameIf for PhysFrameIfImpl {
    fn alloc_frame() -> Option<PhysAddr> {
        // Your implementation here
    }
    fn dealloc_frame(paddr: PhysAddr) {
        // Your implementation here
    }
    fn phys_to_virt(paddr: PhysAddr) -> VirtAddr {
        // Your implementation here
    }
}
```

* [x86_64](https://github.com/arceos-hypervisor/x86_vcpu)