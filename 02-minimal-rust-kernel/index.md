# 最小Rust内核

在这篇文章里我们将在x86架构上创建一个最小的64位Rust内核。我们将在上一篇文章[一个独立的rust二进制程序]的基础上创建一个磁盘启动镜像，并将一些内容显示在屏幕上。

[一个独立的rust二进制程序]: ./second-edition/posts/01-freestanding-rust-binary/index.md

<!-- more -->

这个系列的blog在[GitHub]上开放开发，如果你有任何问题，请在这里开一个issuse来讨论。当然你也可以在底部留言。你可以在[这里][post branch]找到这篇文章的完整源码。


[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-02

<!-- toc -->

## Boot进程

当你打开计算机时，计算机会执行存储在主板[ROM]里的固件代码。这些代码扮演了一个[上电自检]的角色，它会检查可用的RAM，预初始化CPU和硬件。然后，它会查找可引导磁盘并开始引导操作系统内核。

[ROM]: https://en.wikipedia.org/wiki/Read-only_memory
[上电自检]: https://en.wikipedia.org/wiki/Power-on_self-test

在x86上，有两种固件标准:"简单输入/输出系统" (**[BIOS]**)和"统一可扩展固件接口" (**[UEFI]**)。BIOS标准比较古老且已经过时了，但是它很简单并且自1980年代起就对x86机器有着良好的支持。相反，UEFI则更现代，拥有更多的功能，但是配置起来也更复杂(至少在我看来是这样)。

[BIOS]: https://en.wikipedia.org/wiki/BIOS
[UEFI]: https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface

目前，我们只提供BIOS支持，但是UEFI支持也在计划之中。如果你希望帮助我实现它，请确认这个 [Github issue](https://github.com/phil-opp/blog_os/issues/349).。


### BIOS Boot
几乎所有的x86系统都支持BIOS启动，包括比较新的基于UEFI的机器也会通过模拟BIOS的方式来支持。这很棒，因为你可以和从上世纪出现的各种老机器保持一致的启动逻辑。然而这种广泛的兼容性也是BIOS启动最大的缺点，因为这意味着在启动前CPU会处于被称为[实模式]的16位兼容模式下，正因如此那些1980年代的古老的bootloader才仍然能够工作。

不过，现在让我们从头开始吧:

当你打开计算机时，它会从主板上某些特殊的flash(闪存)里加载BIOS。BIOS会运行硬件自检和初始化程序，然后，它会寻找可引导的磁盘。当它找到时，它会将控制权转交给*bootloader*--一段存储在磁盘开头的512字节的代码。大部分bootloader都比512字节要大，所以一般会把bootloader分为第一小段(为了适应512字节)和第二段（随后从第一级加载）。

bootloader必须确定内核映像在磁盘上的位置，并将其加载到内存中。它还需要先将CPU置为16位[实模式]，再将其切换到32位[保护模式]，最后切换到64位[长模式]，只有在64位模式下，64位寄存器和完整的内存才可以使用。它的第三个任务则是从BIOS查询一些信息(例如内存映射)并将其传递给操作系统内核。

[实模式]: https://en.wikipedia.org/wiki/Real_mode
[保护模式]: https://en.wikipedia.org/wiki/Protected_mode
[l长模式]: https://en.wikipedia.org/wiki/Long_mode
[memory segmentation]: https://en.wikipedia.org/wiki/X86_memory_segmentation

实现一个bootloader有一点麻烦因为这会需要使用汇编语言以及很多没什么意义的工作例如"将一些魔数写到寄存器里"。因此，我们不会在本文中讲解如何实现一个bootloader，而是提供了一个叫做[bootimage]的工具，该工具可以把bootloader自动添加到内核里。

[bootimage]: https://github.com/rust-osdev/bootimage

如果你对构建一个自己的bootloader感兴趣，请继续关注该教程，我已经计划了一系列关于此内容的文章！ <!-- , check out our “_[Writing a Bootloader]_” posts, where we explain in detail how a bootloader is built. -->


#### Multiboot 标准
为了避免每个操作系统都实现一个自己的bootlaoder，每个bootloader都只兼容单个操作系统，[自由软件基金会]于1995年建立了一个叫[Multibooot]的开放bootloader标准。该标准定义了bootloader和操作系统之间的接口，所以任何Multiboot兼容的bootloader可以加载任何Multiboot兼容的操作系统。最典型的参考实现就是[GNU GRUB],它也是目前在Linux系统中最受欢迎的bootloader。


[自由软件基金会]: https://en.wikipedia.org/wiki/Free_Software_Foundation
[Multiboot]: https://wiki.osdev.org/Multiboot
[GNU GRUB]: https://en.wikipedia.org/wiki/GNU_GRUB

要使内核兼容Multiboot，只需要在内核文件的开头插入所谓的[Multiboot header] 。这样就可以非常简单的在GRUB里启动操作系统了。然而，GRUB和Multiboot规范也有一些问题:

[Multiboot header]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.html#OS-image-format

- 他们只支持32位保护模式。这意味着你仍然需要进行一些CPU配置来切换到64位长模式。
- 他们是为了使bootloader更简单而不是为了使内核更简单而设计的。例如，内核需要以[调整后的默认页大小(adjusted default page size)]链接,否则GRUB找不到Multiboot头。另一个例子则是[boot information]，它包含大量与架构有关的数据，会被直接传递给内核而不是经过一层更清晰的抽象。
- GRUB和Multiboot标准的文档说明都很少。
- 只有在宿主机上安装了GRUB才能从内核文件里创建磁盘引导镜像。这就导致在Windows和Mac环境下的开发变得很困难。

[调整后的默认页大小(adjusted default page size)]: https://wiki.osdev.org/Multiboot#Multiboot_2
[boot information]: https://www.gnu.org/software/grub/manual/multiboot/multiboot.html#Boot-information-format

由于这些缺点，我们决定不使用GRUB或是Multiboot规范。但是，我们计划在我们的[bootimage]工具里添加Multiboot支持，这样你就可以在一个GRUB系统里加载你的内核。如果你对实现一个Multiboot兼容的内核感兴趣，请查看这个博客系列文章的[first editon]。

[first edition]: ./first-edition/_index.md

### UEFI

(目前我们不打算支持UEFI，但是我们还是很乐意支持UEFI的！如果你想要帮忙，请在[Github issue]里告诉我们。

## 最小内核

现在我们已经大致了解了计算机的启动，是时候来创建我们自己的小内核了。我们的目标是创建一个在启动时会在屏幕上输出"Hello World"的磁盘镜像。我们在上一篇博客[一个独立的rust二进制程序]的基础上来继续。

你也许还记得，我们是通过`cargo`来构建独立二进制程序的，然而这仍然依赖于操作系统:我们需要不同的入口点，不同的编译选项(flag)。这是因为*cargo*默认是以宿主系统，即你正在运行的那个操作系统为目标构建平台的。这不是我们想要的内核，因为内核如果要运行在例如Windows这样的操作系统上那就没有意义了。所以，我们需要明确的定义一个*目标编译系统(target system)*。



### 安装Rust Nightly
Rust有三个发行通道： *stable*, *beta*, 和 *nightly*。<<Rust编程语言>>一书对这些版本之间的区别有详细的解释，你可以花上一分钟来[查看一下](https://doc.rust-lang.org/book/appendix-07-nightly-rust.html#choo-choo-release-channels-and-riding-the-trains)。为了构建一个操作系统，我们需要一些只有nightly才会提供的实验性功能，所以我们需要安装nightly版的Rust。

我强烈推荐用[rustup]来管理Rust安装。它允许你同时安装nightly, beta, 和stable版本并很方便的更新它们。通过rustup，你可以执行`rustup override add nightly`命令实现在当前目录下使用nightly 编译器。另外你可以将内容为`nightly`的名为`rust-toolchain`的文件添加到项目的根目录里。你还可以通过运行`rust --version`命令(版本号的最后应该要包含`-nightly`)来确认你安装的nightly版本。

[rustup]: https://www.rustup.rs/

nightly允许我们在文件的开头添加所谓的*feature flags*来选择使用某些实验性的功能。例如，我们可以通过在`main.rs`文件头部添加例如 `#![feature(asm)]` 来启用 实验性的内敛汇编[`asm!` macro(宏)]注意，像这样的实验性功能是完全不稳定的，这也以为着这些功能可能在没有提取说明的情况下就在某个版本里变动甚至被移除了。基于这个理由，我们只在必须的情况下才使用他们。

[`asm!` macro(宏)]: https://doc.rust-lang.org/nightly/unstable-book/language-features/asm.html

### Target Specification
cargo可以通过 `--target`参数来支持不同的目标系统。这个目标系统由所谓[目标三元组]来描述，它描述了CPU的架构，供应商，操作系统以及[ABI]。举个例子，`x86_64-unknown-linux-gnu`三元目标组描述了一个基于`x86_64` CPU，不明确的供应商，采用GNU ABI的Linux操作系统的结构。Rust支持[很多不同的目标三元组][平台支持]，包括安卓的 `arm-linux-androideabi`和[WebAssembly的`wasm32-unkonw`](https://www.hellorust.com/setup/wasm-target/)平台。



[目标三元组]: https://clang.llvm.org/docs/CrossCompilation.html#target-triple
[ABI]: https://stackoverflow.com/a/2456882
[平台支持]: https://forge.rust-lang.org/platform-support.html

对于我们的目标系统，我们需要一些特殊的配置参数(例如，不依赖于底层操作系统)，所以现有的[目标三元组][平台支持]都不适用。幸运的是，Rust允许我们通过一个JOSN文件来定义我们自己的目标平台。例如，一个描述``x86_64-unknown-linux-gnu`目标平台的JSON文件如下:

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

大部分字段需要有LLVM来为该平台生成代码。例如， [`data-layout`]字段定义了各种整型，浮点型，指针类型的大小。还有一些用于Rust条件编译的字段，诸如 `target-pointer-width`。第三种类型的字段则定义了该如何构建一个crate。例如， `pre-link-args` 字段就指定了传递给[linker(链接器)]的参数。


[`data-layout`]: https://llvm.org/docs/LangRef.html#data-layout
[linker(链接器)]: https://en.wikipedia.org/wiki/Linker_(computing)

`x86_64`体系也是我们内核的目标平台之一，所以我们的target specfiation(目标配置)看起来和第一个非常相似。让我们从创建包含如下内容的 `x86_64-blog_os.json`(你可以任选一个你喜欢的名字)文件开始吧。 

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
}
```

注意我们改变了`llvm-target`中的OS关键字，同时将`os`字段修改为`none`，因为在后面我们的操作系统是要跑在裸机上的。

我们添加了以下与构建相关的条目:

```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

这里我们使用Rust自带的跨平台[LLD] linker而不是平台默认的linker(平台自带的有可能不支持Linux目标平台)。

[LLD]: https://lld.llvm.org/

```json
"panic-strategy": "abort",
```

该设置指定了目标平台不支持panic时的[栈展开(stack unwinding)],相应的，程序在panic时会直接终止。这个设置的效果和Cargo.toml里的`panic = "abort"`选项效果一样，所以我们可以从Cargo.toml里将其移除。

[栈展开(stack unwinding)]: http://www.bogotobogo.com/cplusplus/stackunwinding.php

```json
"disable-redzone": true,
```

由于我们是在编写内核，所以我们在某些时候会需要处理中断。为了安全的执行该操作，我们必须禁用被称为*red zone*的堆栈指针优化，否则会导致堆栈损坏。想了解更多的信息，请阅读另一篇文章[禁用red zone]。

[禁用red zone]: ./second-edition/extra/disable-red-zone/index.md

```json
"features": "-mmx,-sse,+soft-float",
```
`feature`字段可以启用/禁用目标平台的功能。我们通过添加减号前缀来禁用`mmx`和`sse`功能，通过添加加号前缀来启用`soft-float`功能。

`mmx`和`sse`决定了是否支持[单指令多数据(SIMD)]，SIMD通常可以显著的提高程序运行的速度。然而，在操作系统内核里使用大型SIMD寄存器会导致性能问题。原因是系统在从中断程序返回前必须将所有的寄存器恢复原状。这也意味着，在每次系统调用和硬件中断发生时，内核都必须保存整个SIMD的状态。SIMD的状态非常大(一般在512-1600字节之间)而中断可能又会频繁发生，这些额外的保存/恢复操作会严重影响性能。为了避免这种情况，我们将为我们的内核(不是为上层应用)禁用SIMD。

[单指令多数据(SIMD)]: https://en.wikipedia.org/wiki/SIMD

禁用SIMD会导致的一个问题是在`X86_64`下浮点运算默认依赖于SIMD寄存器。为了解决这个问题，我们引入了`soft-float`功能。`soft-float`可以在通用整型数据的基础上通过软件模拟的方式来实现浮点运算。

想要了解更多信息，请阅读[禁用SIMD](./second-edition/extra/disable-simd/index.md)。

#### 统合起来
我们的target specification(目标平台配置)文件现在看起来应该如下:

```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
  "arch": "x86_64",
  "target-endian": "little",
  "target-pointer-width": "64",
  "target-c-int-width": "32",
  "os": "none",
  "executables": true,
  "linker-flavor": "ld.lld",
  "linker": "rust-lld",
  "panic-strategy": "abort",
  "disable-redzone": true,
  "features": "-mmx,-sse,+soft-float"
}
```

### 构建我们的内核
为了编译我们的系统，我们需要使用Linux的规范(我也不清楚为什，我猜可能是LLVM的默认设置？)。这也就意味着我们需要一个像[上一篇文章]里描述的名为`_start`的入口点。

[上一篇文章]: ./second-edition/posts/01-freestanding-rust-binary/index.md

```rust
// src/main.rs

#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```

注意，不管你的宿主机是什么操作系统，你的入口点都必须叫`_start`。上一篇文章里提到的Windows和macOS的入口点都应该删掉。

我们现在可以通过把这个JSON文件名传入`--target`参数来编译我们的内核了:

```
> cargo build --target x86_64-blog_os.json

error[E0463]: can't find crate for `core` OR
error[E0463]: can't find crate for `compiler_builtins`
```

编译失败！错误信息告诉我们Rust编译器没有找到`core`或是 `compiler_builtins`库。这两个库都会隐式的链接到所有的`no_std`包。[`core` library]包含了Rust的基础类型诸如`Result`,`Option`和迭代器，而 [`compiler_builtins` library]则提供了很多LLVM需要的底层函数，例如`memcpy`。

[`core` library]: https://doc.rust-lang.org/nightly/core/index.html
[`compiler_builtins` library]: https://github.com/rust-lang-nursery/compiler-builtins

现在的问题是core library是作为*预编译库*和Rust编译器一起发布的。所以它只对它支持的目标三元组宿主(例如`x86_64-unknown-linux-gnu`)有效，而对我们自定义的平台无效。如果我们想要为我们自己的目标平台编译代码，我们需要先为目标平台重新编译`core`才行。

#### Cargo xbuild
这就是我们为什么需要引入 [`cargo xbuild`]的原因了。它是一个可以自动交叉编译`core`和其他内置库的`cargo build`的封装。我们可以通过执行如下命令来安装它:

[`cargo xbuild`]: https://github.com/rust-osdev/cargo-xbuild

```
cargo install cargo-xbuild
```

这个命令依赖于rust源码，我们可以通过`rustup component add rust-src`命令来安装rust源码。

我们现在可以把上面的命令中的`build`替换成`xbuild`并重新运行:

```
> cargo xbuild --target x86_64-blog_os.json
   Compiling core v0.0.0 (/…/rust/src/libcore)
   Compiling compiler_builtins v0.1.5
   Compiling rustc-std-workspace-core v1.0.0 (/…/rust/src/tools/rustc-std-workspace-core)
   Compiling alloc v0.0.0 (/tmp/xargo.PB7fj9KZJhAI)
    Finished release [optimized + debuginfo] target(s) in 45.18s
   Compiling blog_os v0.1.0 (file:///…/blog_os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.29 secs
```

我们看到`cargo xbuild`为我们自定义的目标平台交叉编译了 `core`, `compiler_builtin`, 和 `alloc` 库。由于这些库用了大量的unstable的功能，所以我们必须使用[nightly Rust compiler]。最后，我们终于成功编译了我们的`blog_os`crate。

[nightly Rust compiler]: ./second-edition/posts/01-freestanding-rust-binary/index.md#installing-rust-nightly

我们现在终于可以为裸机构建我们的内核了。然而，我们提供给boot loader调用的`_start`入口点仍然是空的。所以，我们来让它向屏幕输出一些东西。

### 向屏幕打印
现阶段像屏幕打印字符的最简单的方式就是通过[VGA text buffer]了。这是一块映射到VGA硬件的特殊的内存区域，它里面包含了要显示到屏幕上的内容。它通常由25行组成，每行包含80个字符单元。每个字符单元显示一个包含前景和背景色的ASCII字符。在屏幕上显示效果如下:

[VGA text buffer]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode

![screen output for common ASCII characters](https://upload.wikimedia.org/wikipedia/commons/6/6d/Codepage-737.png)

我们会在下一章详细讨论VGA缓冲区的内存布局，届时我们将会为其写一个简单的驱动。现在，为了输出"Hello World!",我们只需要知道缓冲区的起始位置为`0xb8000`且每个字符单元包含一个ASCII字节和一个颜色字节。

代码实现如下:

```rust
static HELLO: &[u8] = b"Hello World!";

#[no_mangle]
pub extern "C" fn _start() -> ! {
	let vga_buffer = 0xb8000 as *mut u8;

    for (i, &byte) in HELLO.iter().enumerate() {
        unsafe {
            *vga_buffer.offset(i as isize * 2) = byte;
            *vga_buffer.offset(i as isize * 2 + 1) = 0xb;
        }
    }

	loop {}
}
```

首先我们将整型数据`0xb8000`转成一个[裸指针]。然后，我们[迭代]遍历[静态(satic)][字节字符串]`HELLO`。我们使用 [`enumerate`]来获得另一个循环变量`i`。在for循环体中，我们用[`offset`]方法来将字符串字节和对应的颜色字节写入内存中(`0xb`代表浅青色)。

[迭代]: https://doc.rust-lang.org/stable/book/ch13-02-iterators.html
[静态(static)]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate
[字节字符串]: https://doc.rust-lang.org/reference/tokens.html#byte-string-literals
[裸指针]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`offset`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

注意，在所有的写内存操作外都有一个[`unsafe`]区块包裹。原因是Rust编译器无法证明我们创建的裸指针是有效的。他们可能会指向任何地方并导致数据损坏。将代码放到`unsafe`块里基本上可以代表我们告诉编译器我们绝对确定自己做的操作都是合法的。注意`unsafe`块并没有关闭Rust的安全检查，它只是允许你做[四种例外事件]:

[`unsafe`]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html
[四种例外事件]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers

我要强调**随便使用unsafe并不是我们在Rust里的工作方式！**在unsafe块中使用裸指针很容易搞砸，举个例子，如果我们不小心的话我们会很容易往缓冲区外的内存写入数据。

所以我们应该尽量最小化`unsafe`块。Rust给了我们创建安全抽象的能力来实现这个目标。举个例子，我们可以创建一个VGA缓冲区类型将所有的不安全的代码封装起来从而确保在外部操作时不会有任何不安全的错误发生。这样我们只需要最少的`unsafe`代码从而确保我们不会破坏[内存安全]。在下一篇文章里，我们将会创建这样的一个安全的VGA缓冲区抽象。

[内存安全]: https://en.wikipedia.org/wiki/Memory_safety

## Running our Kernel

Now that we have an executable that does something perceptible, it is time to run it. First, we need to turn our compiled kernel into a bootable disk image by linking it with a bootloader. Then we can run the disk image in the [QEMU] virtual machine or boot it on real hardware using an USB stick.

### Creating a Bootimage

To turn our compiled kernel into a bootable disk image, we need to link it with a bootloader. As we learned in the [section about booting], the bootloader is responsible for initializing the CPU and loading our kernel.

[section about booting]: #the-boot-process

Instead of writing our own bootloader, which is a project on its own, we use the [`bootloader`] crate. This crate implements a basic BIOS bootloader without any C dependencies, just Rust and inline assembly. To use it for booting our kernel, we need to add a dependency on it:

[`bootloader`]: https://crates.io/crates/bootloader

```toml
# in Cargo.toml

[dependencies]
bootloader = "0.6.0"
```

Adding the bootloader as dependency is not enough to actually create a bootable disk image. The problem is that we need to link our kernel with the bootloader after compilation, but cargo has no support for [post-build scripts].

[post-build scripts]: https://github.com/rust-lang/cargo/issues/545

To solve this problem, we created a tool named `bootimage` that first compiles the kernel and bootloader, and then links them together to create a bootable disk image. To install the tool, execute the following command in your terminal:

```
cargo install bootimage --version "^0.7.3"
```

The `^0.7.3` is a so-called [_caret requirement_], which means "version `0.7.3` or a later compatible version". So if we find a bug and publish version `0.7.4` or `0.7.5`, cargo would automatically use the latest version, as long as it is still a version `0.7.x`. However, it wouldn't choose version `0.8.0`, because it is not considered as compatible. Note that dependencies in your `Cargo.toml` are caret requirements by default, so the same rules are applied to our bootloader dependency.

[_caret requirement_]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#caret-requirements

For running `bootimage` and building the bootloader, you need to have the `llvm-tools-preview` rustup component installed. You can do so by executing `rustup component add llvm-tools-preview`.

After installing `bootimage` and adding the `llvm-tools-preview` component, we can create a bootable disk image by executing:

```
> cargo bootimage
```

We see that the tool recompiles our kernel using `cargo xbuild`, so it will automatically pick up any changes you make. Afterwards it compiles the bootloader, which might take a while. Like all crate dependencies it is only built once and then cached, so subsequent builds will be much faster. Finally, `bootimage` combines the bootloader and your kernel to a bootable disk image.

After executing the command, you should see a bootable disk image named `bootimage-blog_os.bin` in your `target/x86_64-blog_os/debug` directory. You can boot it in a virtual machine or copy it to an USB drive to boot it on real hardware. (Note that this is not a CD image, which have a different format, so burning it to a CD doesn't work).

#### How does it work?
The `bootimage` tool performs the following steps behind the scenes:

- It compiles our kernel to an [ELF] file.
- It compiles the bootloader dependency as a standalone executable.
- It links the bytes of the kernel ELF file to the bootloader.

[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[rust-osdev/bootloader]: https://github.com/rust-osdev/bootloader

When booted, the bootloader reads and parses the appended ELF file. It then maps the program segments to virtual addresses in the page tables, zeroes the `.bss` section, and sets up a stack. Finally, it reads the entry point address (our `_start` function) and jumps to it.

### 在QEMU里启动内核

现在我们可以在虚拟机里启动这个磁盘镜像。可以通过执行以下命令来使其在[QEMU]中启动:

[QEMU]: https://www.qemu.org/

```
> qemu-system-x86_64 -drive format=raw,file=bootimage-blog_os.bin
warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
```

这会打开一个独立的窗口并显示如下画面:

![QEMU showing "Hello World!"](qemu.png)

我们可以看到"Hello World!"已经显示在屏幕上了。


### 物理机
我们还可以将其写入到U盘中并在物理机上启动它:


```
> dd if=target/x86_64-blog_os/debug/bootimage-blog_os.bin of=/dev/sdX && sync
```

`sdx`是你的U盘设备名。**注意**选择正确的设备名，因为指定设备上的数据都将全部被擦除覆写。

在将镜像写入U盘后，你可以在任意机器上通过U盘来启动。为了从U盘启动系统你可能需要设置启动菜单或是在BIOS设置里修改启动顺序。注意，由于`bootloader`crate不支持UEFI，所以无法在UEFI的机器上启动。

### Using `cargo run`

To make it easier to run our kernel in QEMU, we can set the `runner` configuration key for cargo:

```toml
# in .cargo/config

[target.'cfg(target_os = "none")']
runner = "bootimage runner"
```

The `target.'cfg(target_os = "none")'` table applies to all targets that have set the `"os"` field of their target configuration file to `"none"`. This includes our `x86_64-blog_os.json` target. The `runner` key specifies the command that should be invoked for `cargo run`. The command is run after a successful build with the executable path passed as first argument. See the [cargo documentation][cargo configuration] for more details.

The `bootimage runner` command is specifically designed to be usable as a `runner` executable. It links the given executable with the project's bootloader dependency and then launches QEMU. See the [Readme of `bootimage`] for more details and possible configuration options.

[Readme of `bootimage`]: https://github.com/rust-osdev/bootimage

Now we can use `cargo xrun` to compile our kernel and boot it in QEMU. Like `xbuild`, the `xrun` subcommand builds the sysroot crates before invoking the actual cargo command. The subcommand is also provided by `cargo-xbuild`, so you don't need to install an additional tool.


## 下期预告
在下一篇文章中，我们将探索VGA字符缓冲的更多细节并为它写一个安全的接口。我们将会为其添加一个`println`宏。