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

Most fields are required by LLVM to generate code for that platform. For example, the [`data-layout`] field defines the size of various integer, floating point, and pointer types. Then there are fields that Rust uses for conditional compilation, such as `target-pointer-width`. The third kind of fields define how the crate should be built. For example, the `pre-link-args` field specifies arguments passed to the [linker].

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

Note that we changed the OS in the `llvm-target` and the `os` field to `none`, because we will run on bare metal.

We add the following build-related entries:


```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

Instead of using the platform's default linker (which might not support Linux targets), we use the cross platform [LLD] linker that is shipped with Rust for linking our kernel.

[LLD]: https://lld.llvm.org/

```json
"panic-strategy": "abort",
```

This setting specifies that the target doesn't support [stack unwinding] on panic, so instead the program should abort directly. This has the same effect as the `panic = "abort"` option in our Cargo.toml, so we can remove it from there.

[stack unwinding]: http://www.bogotobogo.com/cplusplus/stackunwinding.php

```json
"disable-redzone": true,
```

We're writing a kernel, so we'll need to handle interrupts at some point. To do that safely, we have to disable a certain stack pointer optimization called the _“red zone”_, because it would cause stack corruptions otherwise. For more information, see our separate post about [disabling the red zone].

[disabling the red zone]: ./second-edition/extra/disable-red-zone/index.md

```json
"features": "-mmx,-sse,+soft-float",
```

The `features` field enables/disables target features. We disable the `mmx` and `sse` features by prefixing them with a minus and enable the `soft-float` feature by prefixing it with a plus.

The `mmx` and `sse` features determine support for [Single Instruction Multiple Data (SIMD)] instructions, which can often speed up programs significantly. However, using the large SIMD registers in OS kernels leads to performance problems. The reason is that the kernel needs to restore all registers to their original state before continuing an interrupted program. This means that the kernel has to save the complete SIMD state to main memory on each system call or hardware interrupt. Since the SIMD state is very large (512–1600 bytes) and interrupts can occur very often, these additional save/restore operations considerably harm performance. To avoid this, we disable SIMD for our kernel (not for applications running on top!).

[Single Instruction Multiple Data (SIMD)]: https://en.wikipedia.org/wiki/SIMD

A problem with disabling SIMD is that floating point operations on `x86_64` require SIMD registers by default. To solve this problem, we add the `soft-float` feature, which emulates all floating point operations through software functions based on normal integers.

For more information, see our post on [disabling SIMD](./second-edition/extra/disable-simd/index.md).

#### Putting it Together
Our target specification file now looks like this:

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

### Building our Kernel
Compiling for our new target will use Linux conventions (I'm not quite sure why, I assume that it's just LLVM's default). This means that we need an entry point named `_start` as described in the [previous post]:

[previous post]: ./second-edition/posts/01-freestanding-rust-binary/index.md

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

Note that the entry point needs to be called `_start` regardless of your host OS. The Windows and macOS entry points from the previous post should be deleted.

We can now build the kernel for our new target by passing the name of the JSON file as `--target`:

```
> cargo build --target x86_64-blog_os.json

error[E0463]: can't find crate for `core` OR
error[E0463]: can't find crate for `compiler_builtins`
```

It fails! The error tells us that the Rust compiler no longer finds the `core` or the `compiler_builtins` library. Both libraries are implicitly linked to all `no_std` crates. The [`core` library] contains basic Rust types such as `Result`, `Option`, and iterators, whereas the [`compiler_builtins` library] provides various lower level functions expected by LLVM, such as `memcpy`.

[`core` library]: https://doc.rust-lang.org/nightly/core/index.html
[`compiler_builtins` library]: https://github.com/rust-lang-nursery/compiler-builtins

The problem is that the core library is distributed together with the Rust compiler as a _precompiled_ library. So it is only valid for supported host triples (e.g., `x86_64-unknown-linux-gnu`) but not for our custom target. If we want to compile code for other targets, we need to recompile `core` for these targets first.

#### Cargo xbuild
That's where [`cargo xbuild`] comes in. It is a wrapper for `cargo build` that automatically cross-compiles `core` and other built-in libraries. We can install it by executing:

[`cargo xbuild`]: https://github.com/rust-osdev/cargo-xbuild

```
cargo install cargo-xbuild
```

The command depends on the rust source code, which we can install with `rustup component add rust-src`.

Now we can rerun the above command with `xbuild` instead of `build`:

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

We see that `cargo xbuild` cross-compiles the `core`, `compiler_builtin`, and `alloc` libraries for our new custom target. Since these libraries use a lot of unstable features internally, this only works with a [nightly Rust compiler]. Afterwards, `cargo xbuild` successfully compiles our `blog_os` crate.

[nightly Rust compiler]: ./second-edition/posts/01-freestanding-rust-binary/index.md#installing-rust-nightly

Now we are able to build our kernel for a bare metal target. However, our `_start` entry point, which will be called by the boot loader, is still empty. So let's output something to screen from it.

### Printing to Screen
The easiest way to print text to the screen at this stage is the [VGA text buffer]. It is a special memory area mapped to the VGA hardware that contains the contents displayed on screen. It normally consists of 25 lines that each contain 80 character cells. Each character cell displays an ASCII character with some foreground and background colors. The screen output looks like this:

[VGA text buffer]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode

![screen output for common ASCII characters](https://upload.wikimedia.org/wikipedia/commons/6/6d/Codepage-737.png)

We will discuss the exact layout of the VGA buffer in the next post, where we write a first small driver for it. For printing “Hello World!”, we just need to know that the buffer is located at address `0xb8000` and that each character cell consists of an ASCII byte and a color byte.

The implementation looks like this:

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

First, we cast the integer `0xb8000` into a [raw pointer]. Then we [iterate] over the bytes of the [static] `HELLO` [byte string]. We use the [`enumerate`] method to additionally get a running variable `i`. In the body of the for loop, we use the [`offset`] method to write the string byte and the corresponding color byte (`0xb` is a light cyan).

[iterate]: https://doc.rust-lang.org/stable/book/ch13-02-iterators.html
[static]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime
[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate
[byte string]: https://doc.rust-lang.org/reference/tokens.html#byte-string-literals
[raw pointer]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`offset`]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset

Note that there's an [`unsafe`] block around all memory writes. The reason is that the Rust compiler can't prove that the raw pointers we create are valid. They could point anywhere and lead to data corruption. By putting them into an `unsafe` block we're basically telling the compiler that we are absolutely sure that the operations are valid. Note that an `unsafe` block does not turn off Rust's safety checks. It only allows you to do [four additional things].

[`unsafe`]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html
[four additional things]: https://doc.rust-lang.org/stable/book/ch19-01-unsafe-rust.html#unsafe-superpowers

I want to emphasize that **this is not the way we want to do things in Rust!** It's very easy to mess up when working with raw pointers inside unsafe blocks, for example, we could easily write behind the buffer's end if we're not careful.

So we want to minimize the use of `unsafe` as much as possible. Rust gives us the ability to do this by creating safe abstractions. For example, we could create a VGA buffer type that encapsulates all unsafety and ensures that it is _impossible_ to do anything wrong from the outside. This way, we would only need minimal amounts of `unsafe` and can be sure that we don't violate [memory safety]. We will create such a safe VGA buffer abstraction in the next post.

[memory safety]: https://en.wikipedia.org/wiki/Memory_safety

### Creating a Bootimage
Now that we have an executable that does something perceptible, it is time to turn it into a bootable disk image. As we learned in the [section about booting], we need a bootloader for that, which initializes the CPU and loads our kernel.

[section about booting]: #the-boot-process

Instead of writing our own bootloader, which is a project on its own, we use the [`bootloader`] crate. This crate implements a basic BIOS bootloader without any C dependencies, just Rust and inline assembly. To use it for booting our kernel, we need to add a dependency on it:

[`bootloader`]: https://crates.io/crates/bootloader

```toml
# in Cargo.toml

[dependencies]
bootloader = "0.4.0"
```

Adding the bootloader as dependency is not enough to actually create a bootable disk image. The problem is that we need to combine the bootloader with the kernel after it has been compiled, but cargo has no support for additional build steps after successful compilation (see [this issue][post-build script] for more information).

[post-build script]: https://github.com/rust-lang/cargo/issues/545

To solve this problem, we created a tool named `bootimage` that first compiles the kernel and bootloader, and then combines them to create a bootable disk image. To install the tool, execute the following command in your terminal:

```
cargo install bootimage --version "^0.5.0"
```

The `^0.5.0` is a so-called [_caret requirement_], which means "version `0.5.0` or a later compatible version". So if we find a bug and publish version `0.5.1` or `0.5.2`, cargo would automatically use the latest version, as long as it is still a version `0.5.x`. However, it wouldn't choose version `0.6.0`, because it is not considered as compatible. Note that dependencies in your `Cargo.toml` are caret requirements by default, so the same rules are applied to our bootloader dependency.

[_caret requirement_]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#caret-requirements

After installing the `bootimage` tool, creating a bootable disk image is as easy as executing:

```
> bootimage build --target x86_64-blog_os.json
```

You see that the tool recompiles your kernel using `cargo xbuild`, so it will automatically pick up any changes you make. Afterwards it compiles the bootloader, which might take a while. Like all crate dependencies it is only built once and then cached, so subsequent builds will be much faster. Finally, `bootimage` combines the bootloader and your kernel to a bootable disk image.

After executing the command, you should see a bootable disk image named `bootimage-blog_os.bin` in your `target/x86_64-blog_os/debug` directory. You can boot it in a virtual machine or copy it to an USB drive to boot it on real hardware. (Note that this is not a CD image, which have a different format, so burning it to a CD doesn't work).

#### How does it work?
The `bootimage` tool performs the following steps behind the scenes:

- It compiles our kernel to an [ELF] file.
- It compiles the bootloader dependency as a standalone executable.
- It appends the bytes of the kernel ELF file to the bootloader.

[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[rust-osdev/bootloader]: https://github.com/rust-osdev/bootloader

When booted, the bootloader reads and parses the appended ELF file. It then maps the program segments to virtual addresses in the page tables, zeroes the `.bss` section, and sets up a stack. Finally, it reads the entry point address (our `_start` function) and jumps to it.

#### Bootimage Configuration
The `bootimage` tool can be configured through a `[package.metadata.bootimage]` table in the `Cargo.toml` file. We can add a `default-target` option so that we no longer need to pass the `--target` argument:

```toml
# in Cargo.toml

[package.metadata.bootimage]
default-target = "x86_64-blog_os.json"
```

Now we can omit the `--target` argument and just run `bootimage build`.

## Booting it!
We can now boot the disk image in a virtual machine. To boot it in [QEMU], execute the following command:

[QEMU]: https://www.qemu.org/

```
> qemu-system-x86_64 -drive format=raw,file=bootimage-blog_os.bin
warning: TCG doesn't support requested feature: CPUID.01H:ECX.vmx [bit 5]
```

![QEMU showing "Hello World!"](qemu.png)

Alternatively, you can invoke the `run` subcommand of the `bootimage` tool:

```
> bootimage run
```

By default it invokes the exact same QEMU command as above. Additional QEMU options can be passed after a `--`. For example, `bootimage run -- --help` will show the QEMU help. It's also possible to change the default command through an `run-command` key in the `package.metadata.bootimage` table in the `Cargo.toml`. For more information see the `--help` output or the [Readme file].

[Readme file]: https://github.com/rust-osdev/bootimage/blob/master/Readme.md

### Real Machine
It is also possible to write it to an USB stick and boot it on a real machine:

```
> dd if=target/x86_64-blog_os/debug/bootimage-blog_os.bin of=/dev/sdX && sync
```

Where `sdX` is the device name of your USB stick. **Be careful** to choose the correct device name, because everything on that device is overwritten.

## What's next?
In the next post, we will explore the VGA text buffer in more detail and write a safe interface for it. We will also add support for the `println` macro.
