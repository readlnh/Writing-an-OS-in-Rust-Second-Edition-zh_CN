#  测试
本文主要讲述了在`no_std`环境下进行单元测试和集成测试的方法。我们将通过Rust的自定义测试框架来在我们的内核中执行一些测试函数。为了将结果反馈到QEMU上，我们需要使用QEMU的一些其他的功能以及`bootimage`工具。

<!-- more -->

这个系列的blog在[GitHub]上开放开发，如果你有任何问题，请在这里开一个issuse来讨论。当然你也可以在[底部]留言。你可以在[这里][post branch]找到这篇文章的完整源码。

[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-04

<!-- toc -->

## 阅读要求

这篇文章替换了此前的(现在已经过时了) [_单元测试(Unit Testing)_] 和 [_集成测试(Integration Tests)_] 两篇文章。这里我将假定你是在2019-04-27日后阅读的[_最小Rust内核_]一文。总而言之，本文要求你已经有一个[设置默认目标]的 `.cargo/config` 文件且[定义了一个runner可执行文件]。

 [_单元测试(Unit Testing)_]: ./second-edition/posts/deprecated/04-unit-testing/index.md
[_集成测试(Integration Tests)_]: ./second-edition/posts/deprecated/05-integration-tests/index.md
[_最小Rust内核_]: ./second-edition/posts/02-minimal-rust-kernel/index.md
[设置默认目标]: ./second-edition/posts/02-minimal-rust-kernel/index.md#set-a-default-target
[定义了一个runner可执行文件]: ./second-edition/posts/02-minimal-rust-kernel/index.md#using-cargo-run

## Rust中的测试

Rust有一个[内置的测试框架(built-in test framework)]无需任何设置就可以进行单元测试，只需要创建一个通过assert来检查结果的函数并在函数的头部加上`#[test]`属性即可。然后`cargo test`会自动找到并执行你的crate中的所有测试函数。

[内置的测试框架(built-in test framework)]: https://doc.rust-lang.org/book/second-edition/ch11-00-testing.html

不幸的是，对于一个`no_std`的应用，比如我们的内核，这有点点复杂。现在的问题是，Rust的测试框架会隐式的调用内置的[`test`]库，但是这个库依赖于标准库。这也就是说我们的 `#[no_std]`内核无法使用默认的测试框架。

[`test`]: https://doc.rust-lang.org/test/index.html

当我们试图在我们的项目中执行`cargo xtest`时，我们可以看到如下信息:

```
> cargo xtest
   Compiling blog_os v0.1.0 (/…/blog_os)
error[E0463]: can't find crate for `test`
```

由于`test`crate依赖于标准库，所以它在我们的裸机目标上并不可用，虽然将`test`crate移植到一个 `#[no_std]` 上下文环境中是[可能的][utest]，但是这样做是高度不稳定的并且还会需要一些特殊的hacks，例如重定义 `panic` 宏。 

[utest]: https://github.com/japaric/utest

### 自定义测试框架

幸运的是，Rust支持通过使用不稳定的 [`自定义测试框架(custom_test_frameworks)`] 功能来替换默认的测试框架。该功能不需要额外的库，因此在 `#[no_std]`环境中它也可以工作。它的工作原理是收集所有标注了 `#[test_case]`属性的函数，然后将这个测试函数的列表作为参数传递给用户指定的runner函数。因此，它实现了对测试过程的最大控制。

[`自定义测试框架(custom_test_frameworks)`]: https://doc.rust-lang.org/unstable-book/language-features/custom-test-frameworks.html

与默认的测试框架相比，它的缺点是有一些高级功能诸如 [`should_panic` tests]都不可用了。相对的，如果需要这些功能，我们需要自己来实现。当然，这点对我们来说是好事，因为我们的环境非常特殊，在这个环境里，这些高级功能的默认实现无论如何都是无法工作的，举个例子， #[should_panic]属性依赖于堆栈展开来捕获内核panic，而我的内核早已将其禁用了。

[`should_panic` tests]: https://doc.rust-lang.org/book/ch11-01-writing-tests.html#checking-for-panics-with-should_panic

要为我们的内核实现自定义测试框架，我们需要将如下代码添加到我们的`main.rs`中去:

```rust
// in src/main.rs

#![feature(custom_test_frameworks)]
#![test_runner(crate::test_runner)]

#[cfg(test)]
fn test_runner(tests: &[&dyn Fn()]) {
    println!("Running {} tests", tests.len());
    for test in tests {
        test();
    }
}
```

我们的runner会打印一个简短的debug信息然后调用列表中的每个测试函数。参数类型 `&[&dyn Fn()]` 是[_Fn()_] trait的 [_trait object_] 引用的一个 [_slice_]。它基本上可以被看做一个可以像函数一样被调用的类型的引用列表。由于这个函数在不进行测试的时候没有什么用，我们使用 `#[cfg(test)]`属性保证它只会出现在测试中。 

[_slice_]: https://doc.rust-lang.org/std/primitive.slice.html
[_trait object_]: https://doc.rust-lang.org/1.30.0/book/first-edition/trait-objects.html
[_Fn()_]: https://doc.rust-lang.org/std/ops/trait.Fn.html

现在当我们运行 `cargo xtest` ，我们可以发现运行成功了。然而，我们看到的仍然是"Hello World"而不是我们的 `test_runner`传递来的信息。这是由于我们的入口点仍然是 `_start` 函数。自定义测试框架会生成一个`main`来供 `test_runner` `test_runner`调用，但是由于我们使用了 `#[no_main]`并提供了我们自己的入口点，所以这个`main`函数就被忽略了。

为了修复这个问题，我们需要通过 `reexport_test_harness_main`属性来将生成的函数的名称更改为与`main`不同的名称。然后我们可以在我们的`_start`函数里调用这个重命名的函数:

```rust
// in src/main.rs

#![reexport_test_harness_main = "test_main"]

#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    #[cfg(test)]
    test_main();

    loop {}
}
```

我们将测试框架的入口函数的名字设置为`test_main`并在我们的 `_start`入口点里调用它。我们使用[条件编译]的方式来限定 只在测试上下文环境中调用`test_main`，因为该函数在非测试环境中根本不会生成。

现在当我们执行 `cargo xtest`时，我们可以看到我们的`test_runner`将 "Running 0 tests"信息显示在屏幕上了。现在我们可以来创建我们的第一个测试函数了:

```rust
// in src/main.rs

#[test_case]
fn trivial_assertion() {
    print!("trivial assertion... ");
    assert_eq!(1, 1);
    println!("[ok]");
}
```

现在，当我们运行 `cargo xtest`时，我们可以看到如下输出:

![QEMU printing "Hello World!", "Running 1 tests", and "trivial assertion... [ok]"](qemu-test-runner-output.png)

传递给 `test_runner`函数的`tests`切片里包含了一个 `trivial_assertion` 函数的引用，从屏幕上输出的 `trivial assertion... [ok]`信息可见，我们的测试调用已经成功了。

在执行完tests后， `test_runner`会将结果返回给 `test_main`函数，而该函数又返回到 `_start`入口点函数，这样我们就进入了一个死循环，因为入口点函数是不允许返回的。这就导致了一个问题，因为我们希望`cargo xtest`在运行完所有的测试后会返回并退出。

## 退出QEMU

现在我们在`_start`函数结束后进入了一个死循环，所以每次执行完`cargo xtest`后我们都需要手动去关闭QEMU。很不走运，我们还想在没有用户交互的环境下用脚本去执行 `cargo xtest`。解决这个问题的最佳方式是实现一个合适的方法来关闭我们的操作系统。不幸的是，这个方式实现起来相对有些复杂，因为这要求我们实现对[APM]或[ACPI]电源管理标准的支持。

[APM]: https://wiki.osdev.org/APM
[ACPI]: https://wiki.osdev.org/ACPI

幸运的是，还有一个绕开这些的办法:QEMU支持一种名为 `isa-debug-exit`的特殊设备，它提供了一种从客户系统里退出QEMU的简单方式。为了使用这个设备，我们需要向QEMU传递一个`-device`参数。当然，我们也可以通过将 `package.metadata.bootimage.test-args` 配置关键字添加到我们的`Cargo.toml`来达到目的:

```toml
# in Cargo.toml

[package.metadata.bootimage]
test-args = ["-device", "isa-debug-exit,iobase=0xf4,iosize=0x04"]
```

 `bootimage runner` 会在QEMU的所有测试命令后默认添加`test-args` 参数。而对于`cargo xrun`命令，这个参数就会被忽略。

在传递设备名 (`isa-debug-exit`)的同时，我们还传递了两个参数，`iobase` 和 `iosize` 。这两个参数指定了通过我们的内核访问设备的_I/O 端口_。

### I/O 端口
在x86平台上，CPU和外围硬件通信通常有两种方式，**内存映射I/O**和**端口映射I/O**。之前，我们已经使用内存映射的方式，通过内存地址`0xb8000`访问了[VGA文本缓冲区]。该地址并没有映射到RAM，而是映射到了VGA设备的一部分内存上。

[VGA text buffer]: ./second-edition/posts/03-vga-text-buffer/index.md

与内存映射不同，端口映射I/O使用独立的I/O总线来进行通信。每个外围设备都有一个或数个端口号。CPU采用了一些特殊的`in`和`out`指定来和端口进行通信，这些指令带有一个端口号和一个字节的数据(有些这种指令的变体也允许发送`u16`或是`u32`的数据)。

`isa-debug-exit`设备使用的就是端口映射I/O。其中， `iobase` 参数指定了设备对应的端口地址(在x86上，`0xf4`是一个[通常未被使用的端口][list of x86 I/O ports])，而`iosize`则指定了端口的大小(`0x04`代表4字节)。

[list of x86 I/O ports]: https://wiki.osdev.org/I/O_Ports#The_list

### 使用退出(Exit)设备

 `isa-debug-exit`设备的功能非常简单。当一个 `value`写入`iobase`指定的端口时，它会导致QEMU以[退出状态(exit status)] `(value << 1) | 1`退出。也就是说，当我们向端口写入`0`时，QEMU将以退出状态`(0 << 1) | 1 = 1`退出，而当我们向端口写入`1`时，它将以退出状态`(1 << 1) | 1 = 3`退出。

[退出状态(exit status)] : https://en.wikipedia.org/wiki/Exit_status

这里我们使用 [`x86_64`] crate提供的抽象，而不是手动调用`in`或`out`指令。为了添加对该crate的依赖，我们可以将其添加到我们的 `Cargo.toml`中的 `dependencies` 小节中去:


[`x86_64`]: https://docs.rs/x86_64/0.7.0/x86_64/

```toml
# in Cargo.toml

[dependencies]
x86_64 = "0.7.0"
```

现在我们可以使用crate中提供的[`Port`] 类型来创建一个`exit_qemu` 函数了:

[`Port`]: https://docs.rs/x86_64/0.7.0/x86_64/instructions/port/struct.Port.html

```rust
// in src/main.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u32)]
pub enum QemuExitCode {
    Success = 0x10,
    Failed = 0x11,
}

pub fn exit_qemu(exit_code: QemuExitCode) {
    use x86_64::instructions::port::Port;

    unsafe {
        let mut port = Port::new(0xf4);
        port.write(exit_code as u32);
    }
}
```

该函数在`0xf4`处创建了一个新的端口，该端口同时也是 `isa-debug-exit` 设备的 `iobase` 。然后它会向端口写入传递的退出代码。这里我们使用`u32`来传递数据，因为我们之前已经将 `isa-debug-exit`设备的 `iosize` o指定为4字节了。上述两个操作都是`unsafe`的，因为I/O端口的写入通常会导致一些不可预知的行为。

为了指定退出状态，我们创建了一个 `QemuExitCode`枚举。思路大体上是，如果所有的测试都成功了就以成功退出码退出，否则就以失败退出码退出。这个枚举类型被标记为 `#[repr(u32)]`，代表每个变量都是一个`u32`的整数类型。我们使用退出代码`0x01`代表成功，`0x11`代表失败。 实际的退出代码并不重要，只要它们不与QEMU的默认退出代码冲突即可。 例如，使用退出代码0表示成功可能并不是一个好主意，因为它在转换后就变成了`(0 << 1) | 1 = 1` ，而`1`是QEMU运行失败时的默认退出代码。 这样，我们就无法将QEMU错误与成功的测试运行区分开来了。

现在我们将我们的`test_runner`更新为运行完所有测试后会退出QEMU的状态:

```rust
fn test_runner(tests: &[&dyn Fn()]) {
    println!("Running {} tests", tests.len());
    for test in tests {
        test();
    }
    /// new
    exit_qemu(QemuExitCode::Success);
}
```

现在，当我们运行`cargo xtest`时，我们会发现在所有测试执行完后QEMU会立刻退出。现在的问题是， `cargo test`会将所有的测试都视为为失败即使我们传递了我们的`成功(Success)`退出代码: 

```
> cargo xtest
    Finished dev [unoptimized + debuginfo] target(s) in 0.03s
     Running target/x86_64-blog_os/debug/deps/blog_os-5804fc7d2dd4c9be
Building bootloader
   Compiling bootloader v0.5.3 (/home/philipp/Documents/bootloader)
    Finished release [optimized + debuginfo] target(s) in 1.07s
Running: `qemu-system-x86_64 -drive format=raw,file=/…/target/x86_64-blog_os/debug/
    deps/bootimage-blog_os-5804fc7d2dd4c9be.bin -device isa-debug-exit,iobase=0xf4,
    iosize=0x04`
error: test failed, to rerun pass '--bin blog_os'
```

这里的问题在于，`cargo test`会将所有非`0`的错误码都视为测试失败。

### 成功退出(Exit)代码

为了解决这个问题， `bootimage`提供了一个 `test-success-exit-code`配置项，可以将指定的退出代码映射到退出代码`0`:

```toml
[package.metadata.bootimage]
test-args = […]
test-success-exit-code = 33         # (0x10 << 1) | 1
```

有了这个配置，`bootimage`就会将我们的成功退出码映射到退出码0,这样一来， `cargo xtest`就能够正确的识别出测试成功的情况而不会将其视为测试失败。

我们的测试runner现在会在正确报告测试结构后自动关闭QEMU。我们可以看到QEMU的窗口会打开一小会，但是这点时间不足以我们看清测试的结果。如果测试结果会打印在控制台上而不是QEMU里，让我们能在QEMU退出后仍然能看到测试结果就好了。

## 打印到控制台

要在控制台上查看测试输出，我们需要以某种方式将数据从内核发送到宿主系统。 有多种方法可以实现这一点，例如通过TCP网络接口来发送数据。 但是，设置网络堆栈是一项很复杂的任务，这里我们选择更简单的解决方案。

### 串口

发送数据的一个简单的方式是通过[串行端口]，这是一个现代电脑中已经不存在的旧标准接口(译者注:玩过单片机的同学应该知道，其实译者上大学的时候有些同学的笔记本电脑还有串口的，没有串口的同学在烧录单片机程序的时候也都会需要usb转串口线,一般是51,像stm32有st-link，这个另说，不过其实也可以用串口来下载)。串口非常易于编程，QEMU可以将通过串口发送的数据重定向到宿主机的标准输出或是文件中。

[串行端口]: https://en.wikipedia.org/wiki/Serial_port

用来实现串行接口的芯片被称为 [UARTs]。在x86上，有[很多UART模型]，但是幸运的是，它们之间仅有的那些不同之处都是我们用不到的高级功能。目前通用的UARTs都会兼容[16550 UART]，所以我们在我们测试框架里采用该模型。

[UARTs]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter
[很多UART模型]: https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter#UART_models
[16550 UART]: https://en.wikipedia.org/wiki/16550_UART

我们使用[`uart_16550`] crate来初始化UART，并通过串口来发送数据。为了将该crate添加为依赖，我们将我们的`Cargo.toml`和`main.rs`修改为如下:

[`uart_16550`]: https://docs.rs/uart_16550

```toml
# in Cargo.toml

[dependencies]
uart_16550 = "0.2.0"
```

 `uart_16550` crate包含了一个代表UART寄存器的`SerialPort`结构体，但是我们仍然需要自己来创建一个相应的实例。我们使用以下内容来创建一个新的`串口`模块:

```rust
// in src/main.rs

mod serial;
```

```rust
// in src/serial.rs

use uart_16550::SerialPort;
use spin::Mutex;
use lazy_static::lazy_static;

lazy_static! {
    pub static ref SERIAL1: Mutex<SerialPort> = {
        let mut serial_port = unsafe { SerialPort::new(0x3F8) };
        serial_port.init();
        Mutex::new(serial_port)
    };
}
```

就像[VGA文本缓冲区][vga lazy-static]一样，我们使用 `lazy_static` 和一个自旋锁来创建一个 `static` writer实例。通过使用 `lazy_static` ，我们可以保证`init`方法只会在该示例第一次被使用使被调用。

和 `isa-debug-exit`设备一样，UART也是用过I/O端口进行编程的。由于UART相对来讲更加复杂，它使用多个I/O端口来对不同的设备寄存器进行编程。不安全的`SerialPort::new`函数需要UART的第一个I/O端口的地址作为参数，从该地址中可以计算出所有所需端口的地址。我们传递的端口地址为`0x3F8` ，该地址是第一个串行接口的标准端口号。

[vga lazy-static]: ./second-edition/posts/03-vga-text-buffer/index.md#lazy-statics

为了使串口更加易用，我们添加了 `serial_print!` 和 `serial_println!`宏:

```rust
#[doc(hidden)]
pub fn _print(args: ::core::fmt::Arguments) {
    use core::fmt::Write;
    SERIAL1.lock().write_fmt(args).expect("Printing to serial failed");
}

/// Prints to the host through the serial interface.
#[macro_export]
macro_rules! serial_print {
    ($($arg:tt)*) => {
        $crate::serial::_print(format_args!($($arg)*));
    };
}

/// Prints to the host through the serial interface, appending a newline.
#[macro_export]
macro_rules! serial_println {
    () => ($crate::serial_print!("\n"));
    ($fmt:expr) => ($crate::serial_print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => ($crate::serial_print!(
        concat!($fmt, "\n"), $($arg)*));
}
```

该实现和我们此前的`print`和`println`宏的实现非常类似。 由于`SerialPort`类型已经实现了`fmt::Write` trait，所以我们不需要提供我们自己的实现了。

[`fmt::Write`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html

现在我们可以从测试代码里向串行接口打印而不是向VGA文本缓冲区打印了:

```rust
// in src/main.rs

#[cfg(test)]
fn test_runner(tests: &[&dyn Fn()]) {
    serial_println!("Running {} tests", tests.len());
    […]
}

#[test_case]
fn trivial_assertion() {
    serial_print!("trivial assertion... ");
    assert_eq!(1, 1);
    serial_println!("[ok]");
}
```

注意，由于我们使用了 `#[macro_export]` 属性， `serial_println`宏直接位于根命名空间下，所以，通过`use crate::serial::serial_println` 来导入该宏是不起作用的。

### QEMU参数

为了查看QEMU的串行输出，我们需要使用`-serial`参数将输出重定向到stdout：

```toml
# in Cargo.toml

[package.metadata.bootimage]
test-args = [
    "-device", "isa-debug-exit,iobase=0xf4,iosize=0x04", "-serial", "stdio"
]
```

现在，当我们运行 `cargo xtest`时，我们可以直接在控制台里看到测试输出了:

```
> cargo xtest
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running target/x86_64-blog_os/debug/deps/blog_os-7b7c37b4ad62551a
Building bootloader
    Finished release [optimized + debuginfo] target(s) in 0.02s
Running: `qemu-system-x86_64 -drive format=raw,file=/…/target/x86_64-blog_os/debug/
    deps/bootimage-blog_os-7b7c37b4ad62551a.bin -device
    isa-debug-exit,iobase=0xf4,iosize=0x04 -serial stdio`
Running 1 tests
trivial assertion... [ok]
```

然而，当测试失败时，我们仍然会在QEMU内看到输出结果，因为我们的panic handler还是用了`println`。为了模拟这个过程，我们将我们的 `trivial_assertion` test中的断言(assertion)修改为 `assert_eq!(0, 1)`:


![QEMU printing "Hello World!" and "panicked at 'assertion failed: `(left == right)`
    left: `0`, right: `1`', src/main.rs:55:5](qemu-failed-test.png)

可以看到，panic信息被打印到了VGA缓冲区里，而测试输出则被打印到串口上了。panic信息非常有用，所以我们希望能够在控制台中来查看它。

### 在panic时打印一个错误信息

为了在panic时使用错误信息来退出QEMU，我们可以使用[条件编译(conditional compilation)]在测试模式下使用(与非测试模式下)不同的panic处理方式:

[条件编译(conditional compilation)]: https://doc.rust-lang.org/1.30.0/book/first-edition/conditional-compilation.html

```rust
// our existing panic handler
#[cfg(not(test))] // new attribute
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}

// our panic handler in test mode
#[cfg(test)]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    serial_println!("[failed]\n");
    serial_println!("Error: {}\n", info);
    exit_qemu(QemuExitCode::Failed);
    loop {}
}
```

在我们的测试panic处理中，我们用 `serial_println`来代替`println` 并使用失败代码来退出QEMU。注意，在`exit_qemu`调用后，我们仍然需要一个无限循环的`loop`因为编译器并不知道 `isa-debug-exit`设备会导致程序退出。

现在，即使在测试失败的情况下QEMU仍然会存在，并会将一些有用的错误信息打印到控制台:

```
> cargo xtest
    Finished dev [unoptimized + debuginfo] target(s) in 0.02s
     Running target/x86_64-blog_os/debug/deps/blog_os-7b7c37b4ad62551a
Building bootloader
    Finished release [optimized + debuginfo] target(s) in 0.02s
Running: `qemu-system-x86_64 -drive format=raw,file=/…/target/x86_64-blog_os/debug/
    deps/bootimage-blog_os-7b7c37b4ad62551a.bin -device
    isa-debug-exit,iobase=0xf4,iosize=0x04 -serial stdio`
Running 1 tests
trivial assertion... [failed]

Error: panicked at 'assertion failed: `(left == right)`
  left: `0`,
 right: `1`', src/main.rs:65:5
```

由于我们现在将所有的测试输出到控制台上了，我们不再需要让QEMU窗口弹出来一小会了。所以，我们可以将其完全隐藏。

### 隐藏 QEMU

由于我们使用`isa-debug-exit`设备和串行端口来报告完整的测试结果，所以我们不再需要QMEU的窗口了。我们可以通过向QEMU传递 `-display none`参数来将其隐藏:

```toml
# in Cargo.toml

[package.metadata.bootimage]
test-args = [
    "-device", "isa-debug-exit,iobase=0xf4,iosize=0x04", "-serial", "stdio",
    "-display", "none"
]
```

现在QEMU完全在后台运行且没有任何窗口会被打开。这不仅不那么烦人，还允许我们的测试框架在没有图形界面的环境里，诸如CI服务器或是[SSH]连接里运行。

[SSH]: https://en.wikipedia.org/wiki/Secure_Shell

### 超时

由于 `cargo xtest` 会等待test runner退出，如果一个测试永远不返回那么它就会一直阻塞test runner。幸运的是，在实际应用中这并不是一个大问题因为无限循环通常是很容易避免的。在我们的这个例子里，无限循环会发生在以下几种不同的情况中:


- bootloader加载内核失败，导致系统不停重启。
- BIOS/UEFI固件加载bootloader失败，同样会导致无限重启。
- CPU在某些函数结束时进入一个`loop{}`语句，例如因为QEMU的exit设备无法正常工作而导致死循环。
- 硬件触发了系统重置，例如未捕获CPU异常时(在后续的文章里会进行解释)。

由于无限循环可能会在各种情况中发生，因此， `bootimage` 工具默认为每个可执行测试设置了一个长度为5分钟的超时时间。如果测试未在此时间内完成，则将其标记为失败，并向控制台输出"Timed Out(超时)"错误。这个功能确保了那些卡在无限循环里的测试不会一直阻塞`cargo xtest`。

你可以将`loop{}`语句添加到 `trivial_assertion`测试中来进行尝试。当你运行 `cargo xtest`时，你可以发现该测试会在五分钟后被标记为超时。超时持续的时间可以通过Cargo.toml中的`test-timeout`来进行[配置][bootimage config]:

[bootimage config]: https://github.com/rust-osdev/bootimage#configuration

```toml
# in Cargo.toml

[package.metadata.bootimage]
test-timeout = 300          # (in seconds)
```

如果你不想为了观察`trivial_assertion` 测试超时等待5分钟之久，你可以暂时降低将上述值。

此后，我们不再需要 `trivial_assertion` 测试，所以我们可以将其删除。

## 测试VGA缓冲区

现在我们已经有了一个可以工作的测试框架了，我们可以为我们的VGA缓冲区实现创建一些测试。首先，我们创建了一个非常简单的测试来验证 `println`是否正常运行而不会panic:

```rust
// in src/vga_buffer.rs

#[cfg(test)]
use crate::{serial_print, serial_println};

#[test_case]
fn test_println_simple() {
    serial_print!("test_println... ");
    println!("test_println_simple output");
    serial_println!("[ok]");
}
```

这个测试所做的仅仅是将一些内容打印到VGA缓冲区。如果它正常结束并且没有panic，也就意味着`println`调用也没有panic。由于我们只需要将 `serial_println` 导入到测试模式里，所以我们添加了 `cfg(test)` attribute(属性)来避免正常模式下 `cargo xbuild`会出现的未使用导入警告(unused import warning)。

为了确保即使打印很多行且有些行超出屏幕的情况下也没有panic发生，我们创建了另一个测试:

```rust
// in src/vga_buffer.rs

#[test_case]
fn test_println_many() {
    serial_print!("test_println_many... ");
    for _ in 0..200 {
        println!("test_println_many output");
    }
    serial_println!("[ok]");
}
```

我们还创建了一个测试函数用来验证打印的几行字符是否真的出现在了屏幕上:

```rust
// in src/vga_buffer.rs

#[test_case]
fn test_println_output() {
    serial_print!("test_println_output... ");

    let s = "Some test string that fits on a single line";
    println!("{}", s);
    for (i, c) in s.chars().enumerate() {
        let screen_char = WRITER.lock().buffer.chars[BUFFER_HEIGHT - 2][i].read();
        assert_eq!(char::from(screen_char.ascii_character), c);
    }

    serial_println!("[ok]");
}
```

该函数定义了一个测试字符串，并通过 `println`将其输出，然后遍历静态 `WRITER`也就是vga字符缓冲区的屏幕字符。由于`println`在将字符串打印到屏幕上最后一行后会立刻附加一个新行(即输出完后有一个换行符)，所以这个字符串应该会出现在第 `BUFFER_HEIGHT - 2`行。

通过使用[`enumerate`] ，我们统计了变量`i`的迭代次数，然后用它来加载对应于`c`的屏幕字符。 通过比较屏幕字符的`ascii_character`和`c` ，我们可以确保字符串的每个字符确实出现在vga文本缓冲区中。

[`enumerate`]: https://doc.rust-lang.org/core/iter/trait.Iterator.html#method.enumerate

如你所想，我们可以创建更多的测试函数，例如一个用来测试当打印一个很长的且包装正确的行时是否会发生panic的函数。或是一个用于测试换行符，不可打印字符，非unicode字符是否能被正确处理的函数。

在这篇文章的剩余部分，我们还会解释如何创建一个 _集成测试_以测试不同组建之间的交互。 


## 集成测试

在Rust中，[集成测试]的约定是将其放到项目根目录中的`tests`目录下(即`src`的同级目录)。无论是默认测试框架还是自定义测试框架都将自动获取并执行该目录下所有的测试。

[集成测试]: https://doc.rust-lang.org/book/ch11-03-test-organization.html#integration-tests

所有的集成测试都是它们自己的可执行文件，并且与我们的`main.rs`完全独立。这也就意味着每个测试都需要定义它们自己的函数入口点。让我们创建一个名为`basic_boot`的例子来看看集成测试的工作细节吧:

```rust
// in tests/basic_boot.rs

#![no_std]
#![no_main]
#![feature(custom_test_frameworks)]
#![test_runner(crate::test_runner)]
#![reexport_test_harness_main = "test_main"]

use core::panic::PanicInfo;

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    test_main();

    loop {}
}

fn test_runner(tests: &[&dyn Fn()]) {
    unimplemented!();
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    loop {}
}
```

由于集成测试都是单独的可执行文件，所以我们需要再次提供所有的crate属性(`no_std`, `no_main`, `test_runner`, 等等)。我们还需要创建一个新的入口点函数`_start`，用于调用测试入口函数`test_main`。我们不需要任何的`cfg(test)` attributes(属性)，因为集成测试的二进制文件在非测试模式下根本不会被编译构建。 

We use the [`unimplemented`] macro that always panics as a placeholder for the `test_runner` function and just `loop` in the `panic` handler for now. Ideally, we want to implement these functions exactly as we did in our `main.rs` using the `serial_println` macro and the `exit_qemu` function. The problem is that we don't have access to these functions since tests are built completely separately of our `main.rs` executable.

[`unimplemented`]: https://doc.rust-lang.org/core/macro.unimplemented.html

If you run `cargo xtest` at this stage, you will get an endless loop because the panic handler loops endlessly. You need to use the `Ctrl+c` keyboard shortcut for exiting QEMU.

### Create a Library

To make the required functions available to our integration test, we need to split off a library from our `main.rs`, which can be included by other crates and integration test executables. To do this, we create a new `src/lib.rs` file:

```rust
// src/lib.rs

#![no_std]
```

Like the `main.rs`, the `lib.rs` is a special file that is automatically recognized by cargo. The library is a separate compilation unit, so we need to specify the `#![no_std]` attribute again.

To make our library work with `cargo xtest`, we need to also add the test functions and attributes:

```rust
// in src/lib.rs

#![cfg_attr(test, no_main)]
#![feature(custom_test_frameworks)]
#![test_runner(crate::test_runner)]
#![reexport_test_harness_main = "test_main"]

use core::panic::PanicInfo;

pub fn test_runner(tests: &[&dyn Fn()]) {
    serial_println!("Running {} tests", tests.len());
    for test in tests {
        test();
    }
    exit_qemu(QemuExitCode::Success);
}

pub fn test_panic_handler(info: &PanicInfo) -> ! {
    serial_println!("[failed]\n");
    serial_println!("Error: {}\n", info);
    exit_qemu(QemuExitCode::Failed);
    loop {}
}

/// Entry point for `cargo xtest`
#[cfg(test)]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    test_main();
    loop {}
}

#[cfg(test)]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    test_panic_handler(info)
}
```

To make our `test_runner` available to executables and integration tests, we don't apply the `cfg(test)` attribute to it and make it public. We also factor out the implementation of our panic handler into a public `test_panic_handler` function, so that it is available for executables too.

Since our `lib.rs` is tested independently of our `main.rs`, we need to add a `_start` entry point and a panic handler when the library is compiled in test mode. By using the [`cfg_attr`] crate attribute, we conditionally enable the `no_main` attribute in this case.

[`cfg_attr`]: https://doc.rust-lang.org/reference/conditional-compilation.html#the-cfg_attr-attribute

We also move over the `QemuExitCode` enum and the `exit_qemu` function and make them public:

```rust
// in src/lib.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u32)]
pub enum QemuExitCode {
    Success = 0x10,
    Failed = 0x11,
}

pub fn exit_qemu(exit_code: QemuExitCode) {
    use x86_64::instructions::port::Port;

    unsafe {
        let mut port = Port::new(0xf4);
        port.write(exit_code as u32);
    }
}
```

Now executables and integration tests can import these functions from the library and don't need to define their own implementations. To also make `println` and `serial_println` available, we move the module declarations too:

```rust
// in src/lib.rs

pub mod serial;
pub mod vga_buffer;
```

We make the modules public to make them usable from outside of our library. This is also required for making our `println` and `serial_println` macros usable, since they use the `_print` functions of the modules.

Now we can update our `main.rs` to use the library:

```rust
// src/main.rs

#![no_std]
#![no_main]
#![feature(custom_test_frameworks)]
#![test_runner(blog_os::test_runner)]
#![reexport_test_harness_main = "test_main"]

use core::panic::PanicInfo;
use blog_os::println;

#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    #[cfg(test)]
    test_main();

    loop {}
}

/// This function is called on panic.
#[cfg(not(test))]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}

#[cfg(test)]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    blog_os::test_panic_handler(info)
}
```

The library is usable like a normal external crate. It is called like our crate, which is `blog_os` in our case. The above code uses the `blog_os::test_runner` function in the `test_runner` attribute and the `blog_os::test_panic_handler` function in our `cfg(test)` panic handler. It also imports the `println` macro to make it available to our `_start` and `panic` functions.

At this point, `cargo xrun` and `cargo xtest` should work again. Of course, `cargo xtest` still loops endlessly (you can exit with `ctrl+c`). Let's fix this by using the required library functions in our integration test.

### Completing the Integration Test

Like our `src/main.rs`, our `tests/basic_boot.rs` executable can import types from our new library. This allows us to import the missing components to complete our test.

```rust
// in tests/basic_boot.rs

#![test_runner(blog_os::test_runner)]

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    blog_os::test_panic_handler(info)
}
```

Instead of reimplementing the test runner, we use the `test_runner` function from our library. For our `panic` handler, we call the `blog_os::test_panic_handler` function like we did in our `main.rs`.

Now `cargo xtest` exits normally again. When you run it, you see that it builds and runs the tests for our `lib.rs`, `main.rs`, and `basic_boot.rs` separately after each other. For the `main.rs` and the `basic_boot` integration test, it reports "Running 0 tests" since these files don't have any functions annotated with `#[test_case]`.

We can now add tests to our `basic_boot.rs`. For example, we can test that `println` works without panicking, like we did in the vga buffer tests:

```rust
// in tests/basic_boot.rs

use blog_os::{println, serial_print, serial_println};

#[test_case]
fn test_println() {
    serial_print!("test_println... ");
    println!("test_println output");
    serial_println!("[ok]");
}
```

When we run `cargo xtest` now, we see that it finds and executes the test function.

The test might seem a bit useless right now since it's almost identical to one of the VGA buffer tests. However, in the future the `_start` functions of our `main.rs` and `lib.rs` might grow and call various initialization routines before running the `test_main` function, so that the two tests are executed in very different environments.

By testing `println` in a `basic_boot` environment without calling any initialization routines in `_start`, we can ensure that `println` works right after booting. This is important because we rely on it e.g. for printing panic messages.

### Future Tests

The power of integration tests is that they're treated as completely separate executables. This gives them complete control over the environment, which makes it possible to test that the code interacts correctly with the CPU or hardware devices.

Our `basic_boot` test is a very simple example for an integration test. In the future, our kernel will become much more featureful and interact with the hardware in various ways. By adding integration tests, we can ensure that these interactions work (and keep working) as expected. Some ideas for possible future tests are:

- **CPU Exceptions**: When the code performs invalid operations (e.g. divides by zero), the CPU throws an exception. The kernel can register handler functions for such exceptions. An integration test could verify that the correct exception handler is called when a CPU exception occurs or that the execution continues correctly after resolvable exceptions.
- **Page Tables**: Page tables define which memory regions are valid and accessible. By modifying the page tables, it is possible to allocate new memory regions, for example when launching programs. An integration test could perform some modifications of the page tables in the `_start` function and then verify that the modifications have the desired effects in `#[test_case]` functions.
- **Userspace Programs**: Userspace programs are programs with limited access to the system's resources. For example, they don't have access to kernel data structures or to the memory of other programs. An integration test could launch userspace programs that perform forbidden operations and verify that the kernel prevents them all.

As you can imagine, many more tests are possible. By adding such tests, we can ensure that we don't break them accidentally when we add new features to our kernel or refactor our code. This is especially important when our kernel becomes larger and more complex.

## Testing Our Panic Handler

Another thing that we can test with an integration test is that our panic handler is called correctly. The idea is to deliberately cause a panic in the test function and exit with a success exit code in the panic handler.

Since we exit from our panic handler, the panicking test never returns to the test runner. For this reason, it does not make sense to add more than one test because subsequent tests are never executed. For cases like this, where only a single test function exists, we can disable the test runner completely and run our test directly in the `_start` function.

### No Harness

The `harness` flag defines whether a test runner is used for an integration test. When it's set to `false`, both the default test runner and the custom test runner feature are disabled, so that the test is treated like a normal executable.

Let's create a panic handler test with a disabled `harness` flag. First, we create a skeleton for the test at `tests/panic_handler.rs`:

```rust
// in tests/panic_handler.rs

#![no_std]
#![no_main]

use core::panic::PanicInfo;
use blog_os::{QemuExitCode, exit_qemu};

#[no_mangle]
pub extern "C" fn _start() -> ! {
    exit_qemu(QemuExitCode::Failed);
    loop {}
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    exit_qemu(QemuExitCode::Failed);
    loop {}
}
```

The code is similar to the `basic_boot` test with the difference that no test attributes are needed and no runner function is called. We immediately exit with an error from the `_start` entry point and the panic handler for now and first try to get it to compile.

If you run `cargo xtest` now, you will get an error that the `test` crate is missing. This error occurs because we didn't set a custom test framework, so that the compiler tries to use the default test framework, which is unavailable for our panic. By setting the `harness` flag to `false` for the test in our `Cargo.toml`, we can fix this error:

```toml
# in Cargo.toml

[[test]]
name = "panic_handler"
harness = false
```

Now the test compiles fine, but fails of course since we always exit with an error exit code.

### Implementing the Test

Let's complete the implementation of our panic handler test:

```rust
// in tests/panic_handler.rs

use blog_os::{serial_print, serial_println, QemuExitCode, exit_qemu};

const MESSAGE: &str = "Example panic message from panic_handler test";
const PANIC_LINE: u32 = 14; // adjust this when moving the `panic!` call

#[no_mangle]
pub extern "C" fn _start() -> ! {
    serial_print!("panic_handler... ");
    panic!(MESSAGE); // must be in line `PANIC_LINE`
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    serial_println!("[ok]");
    exit_qemu(QemuExitCode::Success);
    loop {}
}
```

We immediately `panic` in our `_start` function with a `MESSAGE`. In the panic handler, we exit with a success exit code. We don't need a `qemu_exit` call at the end of our `_start` function, since the Rust compiler knows for sure that the code after the `panic` is unreachable. If we run the test with `cargo xtest --test panic_handler` now, we see that it succeeds as expected.

We will need the `MESSAGE` and `PANIC_LINE` constants in the next section. The `PANIC_LINE` constant specifies the line number that contains the `panic!` invocation, which is `14` in our case (but it might be different for you).

### Checking the `PanicInfo`

To ensure that the given `PanicInfo` is correct, we can extend the `panic` function to check that the reported message and file/line information are correct:

```rust
// in tests/panic_handler.rs

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    check_message(info);
    check_location(info);

    // same as before
    serial_println!("[ok]");
    exit_qemu(QemuExitCode::Success);
    loop {}
}
```

We will show the implementation of `check_message` and `check_location` in a moment. Before that, we create a `fail` helper function that can be used to print an error message and exit QEMU with an failure exit code:

```rust
// in tests/panic_handler.rs

fn fail(error: &str) -> ! {
    serial_println!("[failed]");
    serial_println!("{}", error);
    exit_qemu(QemuExitCode::Failed);
    loop {}
}
```

Now we can implement our `check_location` function:

```rust
// in tests/panic_handler.rs

fn check_location(info: &PanicInfo) {
    let location = info.location().unwrap_or_else(|| fail("no location"));
    if location.file() != file!() {
        fail("file name wrong");
    }
    if location.line() != PANIC_LINE {
        fail("file line wrong");
    }
}
```

The function takes queries the location information from the `PanicInfo` and fails if it does not exist. It then checks that the reported file name is correct by comparing it with the output of the compiler-provided [`file!`] macro. To check the reported line number, it compares it with the `PANIC_LINE` constant that we manually defined above.

[`file!`]: https://doc.rust-lang.org/core/macro.file.html

#### Checking the Panic Message

Checking the reported panic message is a bit more complicated. The reason is that the [`PanicInfo::message`] function returns a [`fmt::Arguments`] instance that can't be compared with our `MESSAGE` string directly. To work around this, we need to create a `CompareMessage` struct:

[`PanicInfo::message`]: https://doc.rust-lang.org/core/macro.file.html
[`fmt::Arguments`]: https://doc.rust-lang.org/core/fmt/struct.Arguments.html

```rust
// in tests/panic_handler.rs

use core::fmt;

/// Compares a `fmt::Arguments` instance with the `MESSAGE` string
///
/// To use this type, write the `fmt::Arguments` instance to it using the
/// `write` macro. If the message component matches `MESSAGE`, the `expected`
/// field is the empty string.
struct CompareMessage {
    expected: &'static str,
}

impl fmt::Write for CompareMessage {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        if self.expected.starts_with(s) {
            self.expected = &self.expected[s.len()..];
        } else {
            fail("message not equal to expected message");
        }
        Ok(())
    }
}
```

The trick is to implement the [`fmt::Write`] trait like we did for our [VGA buffer writer]. The [`write_str`] method is called with a `&str` parameter that we can compare with the expected message. An important detail is that the method is called _multiple times_ with the individual string components. For example, when we do `print!("{}z", "xy")` the method on our VGA buffer writer is invoked once with the string `"xy"` and once with the string `"z"`.

[`fmt::Write`]: https://doc.rust-lang.org/core/fmt/trait.Write.html
[VGA buffer writer]: ./second-edition/posts/03-vga-text-buffer/index.md#formatting-macros
[`write_str`]: https://doc.rust-lang.org/core/fmt/trait.Write.html#tymethod.write_str

This means that we can't directly compare the `s` argument with the expected message, since it might only be a substring. Instead, we use the [`starts_with`] method to verify that the given string component is a substring of the expected message. Then we use [string slicing] to remove the already printed characters from the `expected` string. If the `expected` field is an empty string after writing the panic message, it means that it matches the expected message.

[`starts_with`]: https://doc.rust-lang.org/std/primitive.str.html#method.starts_with
[string slicing]: https://doc.rust-lang.org/book/ch04-03-slices.html#string-slices


With the `CompareMessage` type, we can finally implement our `check_message` function:

```rust
// in tests/panic_handler.rs

#![feature(panic_info_message)] // at the top of the file

use core::fmt::Write;

fn check_message(info: &PanicInfo) {
    let message = info.message().unwrap_or_else(|| fail("no message"));
    let mut compare_message = CompareMessage { expected: MESSAGE };
    write!(&mut compare_message, "{}", message)
        .unwrap_or_else(|_| fail("write failed"));
    if !compare_message.expected.is_empty() {
        fail("message shorter than expected message");
    }
}
```

The function uses the [`PanicInfo::message`] function to get the panic message. If no message is reported, it calls `fail` to fail the test. Since the function is unstable, we need to add the `#![feature(panic_info_message)]` attribute at the top of our test file. Note that you need to adjust the `PANIC_INFO` line number after adding the attribute and the imports on top.

[`PanicInfo::message`]: https://doc.rust-lang.org/core/panic/struct.PanicInfo.html#method.message

After querying the message, the function constructs a `CompareMessage` instance with the `expected` field set to the `MESSAGE` string. Then it writes the message to it using the [`write!`] macro. After the write, it reads the `expected` field and fails the test if it is not the empty string.

[`write!`]: https://doc.rust-lang.org/core/macro.write.html

Now we can run the test using `cargo xtest --test panic_handler`. We see that it passes, which means that the reported panic info is correct. If we use a wrong line number in `PANIC_LINE` or panic with an additional character through `panic!("{}x", MESSAGE)`, we see that the test indeed fails.

## Summary

Testing is a very useful technique to ensure that certain components have a desired behavior. Even if they cannot show the absence of bugs, they're still an useful tool for finding them and especially for avoiding regressions.

This post explained how to set up a test framework for our Rust kernel. We used the custom test frameworks feature of Rust to implement support for a simple `#[test_case]` attribute in our bare-metal environment. By using the `isa-debug-exit` device of QEMU, our test runner can exit QEMU after running the tests and report the test status out. To print error messages to the console instead of the VGA buffer, we created a basic driver for the serial port.

After creating some tests for our `println` macro, we explored integration tests in the second half of the post. We learned that they live in the `tests` directory and are treated as completely separate executables. To give them access to the `exit_qemu` function and the `serial_println` macro, we moved most of our code into a library that can be imported by all executables and integration tests. Since integration tests run in their own separate environment, they make it possible to test the interactions with the hardware or Rust's panic system.

We now have a test framework that runs in a realistic environment inside QEMU. By creating more tests in future posts, we can keep our kernel maintainable when it becomes more complex.

## What's next?

In the next post, we will explore _CPU exceptions_. These exceptions are thrown by the CPU when something illegal happens, such as a division by zero or an access to an unmapped memory page (a so-called “page fault”). Being able to catch and examine these exceptions is very important for debugging future errors. Exception handling is also very similar to the handling of hardware interrupts, which is required for keyboard support.
