# 一个独立的rust二进制程序

> 原文 https://os.phil-opp.com/freestanding-rust-binary/
> 原作者 phil-opp
> 译者 readlnh

创建一个不依赖于标准库的rust可执行文件是我们创建属于自己的操作系统内核的第一步。这将使得在不依赖于底层操作系统的情况下在裸机[bare metal](https://en.wikipedia.org/wiki/Bare_machine) 上运行一个rust程序成为可能。

<!-- more -->

这个系列的blog在[GitHub]上开放开发，如果你有任何问题，请在这里开一个issuse来讨论。当然你也可以在底部留言。你可以在[这里][post branch]找到这篇文章的完整源码。

[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-01

<!-- toc -->

## 简介
为了编写一个操作系统内核，我们的代码不能依赖与任何与操系统相关的功能。也就是说，我们不能使用线程，文件，内存堆栈，网络，随机数字，标准输入输出和其他的一些依赖于操作系统抽象和特定硬件特性的功能。这其实很好理解，毕竟我们要写的是自己的操作系统和自己的驱动。

这也就意味着我们不能使用大部分[Rust标准库]的大部分内容，不过还有很多我们可以用的Rust的特性。举例来说，我们可以使用[迭代器]，[闭包]，[模式匹配]，[option]和[result]，[字符串格式化]，当然还有[所有权系统]。这些功能能让我们在不需要担心[未定义行为]和[内存安全]的情况下写出富有表达性的高抽象层级的代码。

[option]: https://doc.rust-lang.org/core/option/
[result]:https://doc.rust-lang.org/core/result/
[Rust标准库]: https://doc.rust-lang.org/std/
[迭代器]: https://doc.rust-lang.org/book/ch13-02-iterators.html
[闭包]: https://doc.rust-lang.org/book/ch13-01-closures.html
[模式匹配]: https://doc.rust-lang.org/book/ch06-00-enums.html
[字符串格式化]: https://doc.rust-lang.org/core/macro.write.html
[所有权系统]: https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html
[未定义行为]: https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs
[内存安全]: https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention

为了用Rust来构建操作系统内核，我们需要先创建一个可以在不依赖于底层操作系统运行的可执行程序。这些可执行程序通常被称为"freestanding"或"bare-mental" 程序。

这篇文章描述了创建一个freetsanding的Rust二进制程序的必要步骤并解释了为什么需要这些步骤，如果你只对最终的代码实现感兴趣，你可以直接**[跳转到小结](#小结)**

## 禁用标准库
在默认情况下，所有的Rust crates都和[标准库]相关，标准库依赖于操作系统的功能诸如进程，文件，网络等。它还依赖于C标准库`libc`，一个与操作系统服务紧密相连的库。由于我们的目标是实现一个操作系统，所以我们不能使用任何依赖于操作系统的库。所以我们必须通过 [`no_std` attribute]来禁用标准库自动引用。

[标准库]: https://doc.rust-lang.org/std/
[`no_std` attribute]: https://doc.rust-lang.org/1.30.0/book/first-edition/using-rust-without-the-standard-library.html

我们从使用cargo创建一个新工程开始。这一步最简单的方法就是使用下面这条命令。
```
> cargo new blog_os --bin --edition 2018
```

我个人把这个项目命名为`blog_os`，当然你也可以选择你自己的名字。`--bin`flag代表我们是要创建一个二进制客执行程序，`--edition 2018`表示我们的crate要用的是[2018 edition]的Rust。当我们运行这条命令时，cargo会为我们创建如下结构。

[2018 edition]: https://rust-lang-nursery.github.io/edition-guide/rust-2018/index.html

```
blog_os
├── Cargo.toml
└── src
    └── main.rs
```

`Cargo.toml`包含了crate的设置，例如crate(包)名，作者，[semantic版本]，和依赖。 `src/main.rs`文件包含了crate的根模块和`main`函数。你可以通过`cargo build`命令来编译你的crate，然后运行位于`target/debug`子目录下的`blog_os`二进制文件。

[semantic版本]: http://semver.org/

###  `no_std` Attribute
目前我们的crate暗中链接了标准库。现在让我们通过添加 [`no_std`属性]来禁用它。

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

现在当我们试图去构建它的时候(通过运行`cargo build`命令)，会出现下列错误:

```
error: cannot find macro `println!` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

这个错误发生的原因是 [`println` macro]是标准库的一部分，而我们不再把标准库包含在内了。所以我们不能再打印任何东西了。这很好理解，因为`println`会向[标准输出]进行写操作，而这依赖于操作系统提供的特定的文件描述符。

[`println` macro]: https://doc.rust-lang.org/std/macro.println.html
[标准输出]: https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29

所以让我们把输出移除然后对这个空的main函数再试一次。

```rust
// main.rs

#![no_std]

fn main() {}
```

```
> cargo build
error: `#[panic_handler]` function required, but not found
error: language item required, but not found: `eh_personality`
```
现在编译器发现缺少一个 `#[panic_handler]`函数和一个language item。

### Panic的实现
`panic_handler` 属性定义了一个函数，当[painc]发生时它就会被调用。标准库会提供它自己的panic handler函数，但是在一个`no_std`环境里我们需要自己来定义它:

[panic]: https://doc.rust-lang.org/stable/book/ch09-01-unrecoverable-errors-with-panic.html

```rust
// in main.rs

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

[`PanicInfo` 的参数][PanicInfo]包含了panic发生的文件和行数以及可选的panic信息。这个函数永远会返回，所以它也通过将返回值类型设置为[“never” type] 记作`!`来被标记为[diverging function]。目前我们没有什么可以对这个函数做的，所以我们让它无限循环。

[PanicInfo]: https://doc.rust-lang.org/nightly/core/panic/struct.PanicInfo.html
[diverging function]: https://doc.rust-lang.org/1.30.0/book/first-edition/functions.html#diverging-functions
[“never” type]: https://doc.rust-lang.org/nightly/std/primitive.never.html

### `eh_personality` Language Item
Language items是一些编译器需要的特殊的函数或类型。举例来说， [`Copy`] trai就是一个典型的language item，用来告诉编译器那些类型需要遵循 [copy semantics(复制语义)][`Copy`]。当我们深入 [implementation(实现)][copy code]时，我们会发现一个特殊的 `#[lang = "copy"]` attribute(属性)将其定义为一个language item。

[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[copy code]: https://github.com/rust-lang/rust/blob/485397e49a02a3b7ff77c17e4a3f16c653925cb3/src/libcore/marker.rs#L296-L299

自己实现languange items是可能的，但这应该作为最后的手段。因为languages itmes是高度不稳定的语言细节实现甚至都没有类型检查(所以编译器甚至不会检查函数的参数类型是否正常)。幸运的是，还有别的更稳定的办法来修复上述language item错误。

 `eh_personality` language item会标记那些用于实现 [stack unwinding(栈展开)]的函数。在默认情况下，当[panic]发生时Rust会使用展开来析构那些活跃在栈上的变量。这会确保所有使用的内存都会被释放，并允许父进程捕获panic，处理并继续运行。然而，栈展开是一个非常复杂的过程，通常需要依赖操作系统的库(例如Liunx上的 [libunwind] ，Windows上的 [structured exception handling] )，所以我们不打算在我们的操作系统里使用它。

[[stack unwinding(栈展开)]: http://www.bogotobogo.com/cplusplus/stackunwinding.php
[libunwind]: http://www.nongnu.org/libunwind/
[structured exception handling]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657(v=vs.85).aspx

#### 禁用展开
在很多情况下我们并不需要栈展开，所以Rust提供了[abort on panic(panic时终止)]作为替代。这个标志能禁用栈展开相关的标识符的生成从而缩小生成的二进制从文件的大小。我们有很多办法来禁用展开，最简单的办法就是在`Cargo.toml`里添加以下内容:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

这些将panic策略设置为abort的设置不仅仅对dev配置(用于`cargo build`)生效，还对release配置(用于`cargo build --realease`)生效。现在，编译器不会再要求`eh_personality` language item了。

[[abort on panic(panic时终止)]: https://github.com/rust-lang/rust/pull/32900

现在我们已经修复了上述错误了。然而，当我们再次尝试编译，发现编译器又要求另一个language item。

```
> cargo build
error: requires `start` lang_item
```

### `start` attribute
很多人通常觉得`main`函数是程序运行时第一个被调用的函数。然而，实际上大部分语言都有一个 [runtime system(运行时系统)]，用来负责垃圾回收(比如Java)或软件线程(比如go的协程(ps:softsware threads这个词我有点拿捏不准 ))。这些runtime需要在`main`函数被调用前启动，并初始化自身。

[runtime system(运行时系统)]: https://en.wikipedia.org/wiki/Runtime_system

一个普通的连接到标准库的Rust二进制程序，是从一个叫`crt0` (“C runtime zero”)的C运行时库开始执行的，这个库会设置一个适合C语言应用程序运行的环境。这其中包括了栈的创建以及将参数放置到正确的寄存器里等。C运行时会调用 [entry point of the Rust runtime(Rust runtime入口)][rt::lang_start]，这个入口点被标记为 `start` language item。Rust只有一个非常小的运行时，仅仅管理一些很简单的事诸如设置栈溢出边界和在panic打印一个backtrace。这个运行时最后会调用`main`函数。

[rt::lang_start]: https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73

我们的freestanding可执行文件并没有进入Rust runtime和`crt0`，所以我们需要自己来定义入口。在这里实现`start`language item并没有什么帮助，程序仍然会要求`crt0`。因此，我们需要直接覆写`crt0`入口。

### 覆写入口
这里我们添加`#![no_main]` attribute来告诉Rust编译器我们不需要默认的入口链。

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

你可那已经注意到了我们移除了`main`函数，因为既然runtime不会再调用`main`了，它也就没用了。作为替代，我们需要重写操作系统的入口点。

入口点(entry point)的写法通常和操作系统有关。这里你最好先学习一下Linux的写法即使你用的是别的操作系统因为接下来在我们的内核里我们会采用这种写法。

#### Linux
在Linux上，默认的入口点是`_start`。链接器(linker)会寻找带有这个名字的函数，并将这个函数设置为可执行程序的入口点。所以，为了重写我们的入口点，我们需要定义我们自己的`_start`程序:

```rust
#[no_mangle]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

这里有一个重要的点，我们通过`no_mangle` attribute来关闭 [name mangling]，否则编译器会生成诸如`_ZN3blog_os4_start7hb173fedf945531caE`这样编译器识别不了的隐晦符号。这里我们还需要把函数标记为`ertern C`来告诉编译器为这个函数生成 [C calling convention(C调用约定)](而不是Rust调用约定)。

[name mangling]: https://en.wikipedia.org/wiki/Name_mangling
[C calling convention(C调用约定)]: https://en.wikipedia.org/wiki/Calling_convention

`!`返回类型表示这个函数是发散的，也就是说，它不允许返回。这是因为entry point不能被任何函数调用，而应该由操作系统或者bootloader直接调用。所以，entry point应该调用操作系统的 [`exit` system call]而不是程序返回。在我们的情况里，对于这样一个没有任何事可以做的freestanding二进制程序，关机或许是一个不错的选择。现在，我们可以添加一个无限循环来满足永不返回的条件。

[`exit` system call]: https://en.wikipedia.org/wiki/Exit_(system_call)

现在如果我们尝试构建它，会出现下面这样一段丑陋的错误:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" "-Wl,--as-needed" "-Wl,-z,noexecstack" "-m64" "-L"
    "/…/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib"
    "/…/blog_os/target/debug/deps/blog_os-f7d4ca7f1e3c3a09.0.o" […]
    "-o" "/…/blog_os/target/debug/deps/blog_os-f7d4ca7f1e3c3a09"
    "-Wl,--gc-sections" "-pie" "-Wl,-z,relro,-z,now" "-nodefaultlibs"
    "-L" "/…/blog_os/target/debug/deps"
    "-L" "/…/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib"
    "-Wl,-Bstatic"
    "/…/.rustup/toolchains/stable-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib/libcore-dd5bba80e2402629.rlib"
    "-Wl,-Bdynamic"
  = note: /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status

```

这个问题出现的原因是编译器依赖于一些C标准库`libc`的符号，因此它仍然会链接到C运行时的启动环境，而我们在代码里已经使用了 `no_std` attribute来避免链接到标准库。所以在这里我们需要完全抛弃C启动环境。我们可以通过把`-nostartfiles` flag传递链接器里来实现这点。

通过`cargo runstc`这个命令可以实现通过cargo来传递链接属性到编译器。这个命令的表现和`cargo build`类似，不过它允许向Rust的底层编译器`rustc`传递可选项。`rustc`有一个 `-C link-arg` flag可以向链接器传递参数。将两者稍作组合，我们的新构建命令如下:

```
> cargo rustc -- -C link-arg=-nostartfiles
```

现在我们成功从我们的crate中构建出了一个freestanding的可执行程序！

#### Windows
在Windows上，链接器需要两个入口点 [depending on the used subsystem]。
对于`CONSOLE`子系统，我们需要一个叫`mainCRTStartup`的函数，它会调用`main`函数。就像在Linux上一样，我们也需要定义 `no_mangle`函数来覆写入口点。

[depending on the used subsystem]: https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol

```rust
#[no_mangle]
pub extern "C" fn mainCRTStartup() -> ! {
    main();
}

#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}
```

#### macOS
macOS [不支持静态链接到二进制库],所以我们需要链接到`libSystem`库。它的入口点是`main'`:

[不支持静态链接到二进制库]: https://developer.apple.com/library/content/qa/qa1118/_index.html

```rust
#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}
```

为了构建它并链接到 `libSystem`，我们需要执行:

```
> cargo rustc -- -C link-arg=-lSystem
```


## 小结

一个独立(freestanding)微型的Rust二进制程序看起来如下:

`src/main.rs`:

```rust
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo;

/// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

入口点定义依赖于目标操作系统。Linux如下:

```rust
#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```

Windows如下:

```rust
#[no_mangle]
pub extern "C" fn mainCRTStartup() -> ! {
    main();
}

#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}
```

macOS如下:

```rust
#[no_mangle]
pub extern "C" fn main() -> ! {
    loop {}
}
```

独立于操作系统外的`Cargo.toml`如下:

```toml
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

二进制文件可由以下命令编译生成:

```bash
# Linux
> cargo rustc -- -C link-arg=-nostartfiles
# Windows
> cargo build
# macOS
> cargo rustc -- -C link-arg=-lSystem
```

注意，这仅仅是一个freestanding Rust二进制程序的小例子。这样一个二进制程序运行还需要很多条件，比如在`_start`函数被调用时需要有一个初始化完成的栈。**所以为了真正运行这样的一个二进制程序，还有很多步骤需要做**

## 下节预告

 [下一篇文章]会讲解如何在我们这个最小独立二进制程序的基础上构建一个最小操作系统内核的步骤。下一篇还会讲解如何配置目标操作系统的内核，如何使用bootloader，以及如何把一些内容输出到屏幕上。

[下一篇文章]: ./second-edition/posts/02-minimal-rust-kernel/index.md
