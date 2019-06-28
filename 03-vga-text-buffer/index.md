# VGA字符模式

> 原文 https://os.phil-opp.com/minimal-rust-kernel/
> 原作者 phil-opp
> 译者 readlnh
> 翻译项目地址 https://github.com/readlnh/Writing-an-OS-in-Rust-Second-Edition-zh_CN

[VGA文本模式]是将字符打印到屏幕上的一个简单的方法，在这篇文章里，我们通过将所有的不安全的内容封装在一个单独的模块中从而创建一个安全，简单易用的接口。我们还为Rsut的[格式化宏]实现了相应的支持。

[VGA文本模式]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode
[格式化宏]: https://doc.rust-lang.org/std/fmt/#related-macros

<!-- more -->

这个系列的blog在[GitHub]上开放开发，如果你有任何问题，请在这里开一个issuse来讨论。当然你也可以在[底部]留言。你可以在[这里][post branch]找到这篇文章的完整源码。

[GitHub]: https://github.com/phil-opp/blog_os
[底部]: #comments
[post branch]: https://github.com/phil-opp/blog_os/tree/post-03

<!-- toc -->

## VGA文本缓冲区
为了在VGA文本模式下将字符打印到屏幕上，我们需要将其写入到VAG硬件上的相应的文本缓冲区。VGA文本缓冲区是一个25行，80列的二位数组，其中的内容会直接显示到屏幕。这个数组里的每一个元素通过以下格式描述单个显示到屏幕上的字符：

位数(Bit(s)) | 值(Value)
------ | ----------------
0-7    | ASCII码点
8-11   | 前景色
12-14  | 背景色
15     | (是否)闪烁

有以下几种可选的颜色(指前景色和背景色):

数字(Number) | 颜色(Color)      | (Number) + (Bright Bit) | (Bright Color)
------ | ---------- | ------------------- | -------------
0x0    | 黑色(Black)      | 0x8                 | 暗灰色(Dark Gray)
0x1    | 蓝色(Blue)       | 0x9                 | 亮蓝色(Light Blue)
0x2    | 绿色(Green)      | 0xa                 | 亮绿色(Light Green)
0x3    | 青色(Cyan)       | 0xb                 | 亮青色(Light Cyan)
0x4    | 红色(Red)        | 0xc                 | 亮红色(Light Red)
0x5    | 品红(Magenta)    | 0xd                 | 粉色(Pink)
0x6    | 棕色(Brown)      | 0xe                 | 黄色(Yellow)
0x7    | 亮灰色(Light Gray) | 0xf                 | 白色(White)

位4(即第4位)是_明暗(bright bit)_，例如它可以将蓝色转化成亮蓝色。

VGA文本缓冲区通过[I/O内存映射]到地址`0xb8000`上。这意味着对该地址进行读写不会访问RAM而是直接访问VGA硬件上的文本缓冲区。这也意味着我们可以通过一般内存操作来对该地址进行读写。

[I/O内存映射]: https://en.wikipedia.org/wiki/Memory-mapped_I/O

注意，内存映射硬件不一定支持所有的一般RAM操作。举个例子，某个硬件只支持逐字节读取且当读取一个`u64`的数据时就会返回一个无用的信息。幸运的是，文本缓冲区[支持一般读写操作]，所以我们不需要对其进行特殊对待。

[支持一般读写操作]: https://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/vgamem.htm#manip

## A Rust Module
现在我们已经知道了VGA缓冲区的工作原理，我们可以创建一个Rust模块来处理打印事件了:

```rust
// in src/main.rs
mod vga_buffer;
```

关于这个模块的内容，我们创建了一个新的 `src/vga_buffer.rs`文件。下面所有的代码都会进入到我们的这个新模块中(除非有其他说明)。

### 颜色
首先，我们使用一个枚举来表示不同的颜色:

```rust
// in src/vga_buffer.rs

#[allow(dead_code)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Color {
    Black = 0,
    Blue = 1,
    Green = 2,
    Cyan = 3,
    Red = 4,
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    LightBlue = 9,
    LightGreen = 10,
    LightCyan = 11,
    LightRed = 12,
    Pink = 13,
    Yellow = 14,
    White = 15,
}
```

这里我们使用[C-like 枚举]来显式指定每种颜色的数字。因为 `repr(u8)`属性的缘故，所有的枚举变量会被保存为一个`u8`数据。实际上，4位的数据就足够了，但是Rust并没有`u4`这种类型。

[C-like 枚举]: http://rustbyexample.com/custom_types/enum/c_like.html

通常来将，编译器会为每个未使用的变量报一个警告。这里我们通过使用 `#[allow(dead_code)]` 属性来为`Color`枚举禁用这一警告。

通过[派生]包括 [`Copy`], [`Clone`], [`Debug`], [`PartialEq`], 和[`Eq`]在内的traits，我们为这个类型启用了[复制语义]，并使其能够被打印和被比较。

[派生]: http://rustbyexample.com/trait/derive.html
[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[`Clone`]: https://doc.rust-lang.org/nightly/core/clone/trait.Clone.html
[`Debug`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Debug.html
[`PartialEq`]: https://doc.rust-lang.org/nightly/core/cmp/trait.PartialEq.html
[`Eq`]: https://doc.rust-lang.org/nightly/core/cmp/trait.Eq.html
[复制语义]: https://doc.rust-lang.org/book/first-edition/ownership.html#copy-types

为了表示能同时指定前景色和背景色的全色代码，我们在`u8`的基础上创建了一个[新的类型]:

[新的类型]: https://rustbyexample.com/generics/new_types.html

```rust
// in src/vga_buffer.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
struct ColorCode(u8);

impl ColorCode {
    fn new(foreground: Color, background: Color) -> ColorCode {
        ColorCode((background as u8) << 4 | (foreground as u8))
    }
}
```
`ColorCode`结构体包含了全色代码字节，它包含了前景色和背景色的信息。像之前一样，我们同样为其派生了`Copy`和`Debug`traits。为了确保`ColorCode`和`u8`具有完全一样的数据布局，我们使用了[`repr(transparent)`]属性。 

[`repr(transparent)`]: https://doc.rust-lang.org/nomicon/other-reprs.html#reprtransparent

### 文本缓冲区
现在我们添加几个结构体来表示屏幕字符和文本缓冲区:

```rust
// in src/vga_buffer.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
struct ScreenChar {
    ascii_character: u8,
    color_code: ColorCode,
}

const BUFFER_HEIGHT: usize = 25;
const BUFFER_WIDTH: usize = 80;

#[repr(transparent)]
struct Buffer {
    chars: [[ScreenChar; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

由于Rust默认没有指定结构体的字段(成员)顺序(在内存层面上来讲)，所以我们需要使用[`repr(C)`]属性。它能保证该结构体字段和C语言结构体的字段的布局完全相同，从而保证正确的字段排序。对于`Buffer`结构体(类型)，我们再次使用[`repr(transparent)`] 属性来确保它和单个字段(成员)具有相同的内存布局[^译者注1]。

[^译者注1]: 这里的意思就是说加了这个属性后，理论上这个struct的内存布局就和它的成员是一样的，因为这里就只有一个成员，这个struct仅仅只是一层封装。

[`repr(C)`]: https://doc.rust-lang.org/nightly/nomicon/other-reprs.html#reprc

为了将字符实际写到屏幕上，我们来创建一个writer类型:

```rust
// in src/vga_buffer.rs

pub struct Writer {
    column_position: usize,
    color_code: ColorCode,
    buffer: &'static mut Buffer,
}
```

这个writer会向屏幕的最后一行写入并在一行写满(或遇到换行符`\n`)时将每一行依次上移。 `column_position`字段将跟踪光标在最后一行中的位置。当前的前景色和背景色则由`color_code`指定，并将VGA缓冲区的一个引用保存到`buffer`中。注意，这里我们需要一个[显式生命周期]来告诉编译器这个引用的有效时间。 [`'static`] 生命周期指定了在整个程序的运行周期内该引用都是有效的(对于VGA文本缓冲区来说正是如此)。

[显式生命周期]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-annotation-syntax
[`'static`]: https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime

### 打印
现在我们可以通过`Writer`来修改缓冲区中的字符了。首先，我们创建一个方法用来写入单个ASCII字节:

```rust
// in src/vga_buffer.rs

impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(),
            byte => {
                if self.column_position >= BUFFER_WIDTH {
                    self.new_line();
                }

                let row = BUFFER_HEIGHT - 1;
                let col = self.column_position;

                let color_code = self.color_code;
                self.buffer.chars[row][col] = ScreenChar {
                    ascii_character: byte,
                    color_code,
                };
                self.column_position += 1;
            }
        }
    }

    fn new_line(&mut self) {/* TODO */}
}
```

如果这个字节是[换行]字节`\n`，那么writer不会打印任何东西。相应的，它会调用`new_line`方法，稍后我们会实现这个方法。其他的字节则按照match匹配的第二项里的，被打印到屏幕上。

[换行]: https://en.wikipedia.org/wiki/Newline

每打印一个字节，writer就会检查当前行是否已经满了。如果已经满了，就会先调用`new_line`来进行换行。然后，将一个新的`ScreenChar`写入到缓冲区的当前位置。最后，当前光标位置前进一位。

为了打印一整个字符串，我们可以先将他们转换成字节，并将它们逐个(按字节)打印:

```rust
// in src/vga_buffer.rs

impl Writer {
    pub fn write_string(&mut self, s: &str) {
        for byte in s.bytes() {
            match byte {
                // printable ASCII byte or newline
                0x20...0x7e | b'\n' => self.write_byte(byte),
                // not part of printable ASCII range
                _ => self.write_byte(0xfe),
            }

        }
    }
}
```

VGA文本缓冲区仅支持ASCII和 [代码页 437]的额外字节。Rsut的字符串默认是[UTF-8] 格式的，因此，其中可能包含不被VGA缓冲区所支持的字符，我们使用一个match语句来区分可打印的ASCII字节(换行符或是介于空格字符和`~`字符之间的任意字符)和不能打印的字节。对于不能打印的字节，我们打印一个`■`字符，该字符在VGA硬件中的16进制编码为`0xfe`。 

[代码页 437]: https://en.wikipedia.org/wiki/Code_page_437
[UTF-8]: http://www.fileformat.info/info/unicode/utf8.htm

#### 试试看!
你可以创建一个临时的函数来将一些字符写到屏幕上:

```rust
// in src/vga_buffer.rs

pub fn print_something() {
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };

    writer.write_byte(b'H');
    writer.write_string("ello ");
    writer.write_string("Wörld!");
}
```

该函数首先会创建一个新的指向地址`0xb8000`的VGA缓冲区的Writer。这里的语法看起来可能有一些奇怪:首先，我们将整数`0xb8000`转换为一个可变的[裸指针]。然后我们通过解引用(通过`*`)将其转换为一个可变引用然后立刻再次借用它(通过`&mut`)。这个转换过程需要一个 [`unsafe` block]，因为编译器无法保证裸指针的有效性。

[裸指针]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`unsafe` block]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html

然后它将字节`b'H'`写入其中。`b`前缀创建了一个[byte literal]，代表一个单独的ASCII字符。通过写入字符串 `"ello "'` 和 `"Wörld!"`，我们测试了我们的`write_string`方法和其对不可打印字符的处理。我们在`_start`函数中调用`print_something`函数来查看输出:

```rust
// in src/main.rs

#[no_mangle]
pub extern "C" fn _start() -> ! {
    vga_buffer::print_something();

    loop {}
}
```

现在，当我们运行我们的项目时，一个黄色的 `Hello W■■rld!`字符串就会被打印在屏幕的*左下角*。

[byte literal]: https://doc.rust-lang.org/reference/tokens.html#byte-literals

![QEMU output with a yellow `Hello W■■rld!` in the lower left corner](vga-hello.png)

注意， `ö` 被打印成了两个 `■` 字符。这是因为 `ö` 在[UTF-8]中由两个字节来表示，这两个字节都没有落在可打印的ASCII范围内，事实上，这是UTF-8的一个基本性质:多字节的值的每一个单独的字节都不可能是合法的ASCII值。

### Volatile
现在我们看到我们的信息已经能正确打印了。然而，在更进一步优化的Rust编译器上这些代码可能就不能正常工作了。

造成这个问题的原因是我们指向`Buffer`中写入数据而从来没有从其中再次将数据读取出来。编译器不知道我们其实已经访问了VGA缓冲区内存(而不是一般的RAM)也不知道这些操作带来的副反应--会有一些字符显示在屏幕上。所以，编译器可能会觉得这些操作是没必要的并将其优化掉。为了避免这种错误的优化，我们需要将这些写入操作指定为*[volatile(易失)]*。这将告诉编译器该写入操作有相应的副反应，不应该被优化掉。

[volatile(易失)]: https://en.wikipedia.org/wiki/Volatile_(computer_programming)

为了在VGA缓冲区中使用易失性写入操作，我们使用了 [volatile][volatile crate] 库。这个crate(这是Rust世界里包的称呼)提供了一个带有`read`和`write`方法的`Volatile`包装类型。这些方法的内部调用了核心库中的 [read_volatile]和[write_volatile]函数，因此可以保证读/写操作不会被优化掉。

[volatile crate]: https://docs.rs/volatile
[read_volatile]: https://doc.rust-lang.org/nightly/core/ptr/fn.read_volatile.html
[write_volatile]: https://doc.rust-lang.org/nightly/core/ptr/fn.write_volatile.html

我们可以将`volatile` crate添加到我们的 `Cargo.toml`的`dependencies` 小节中去:

```toml
# in Cargo.toml

[dependencies]
volatile = "0.2.3"
```

这里的`0.2.3`是一个[语义]版本号。更多相关信息请查看cargo文档中的[指定依赖项]指南。

[语义]: http://semver.org/
[指定依赖项]: http://doc.crates.io/specifying-dependencies.html

让我们用它来完成VGA缓冲区的volatile写入。将我们的`Buffer`类型更新为如下形式:

```rust
// in src/vga_buffer.rs

use volatile::Volatile;

struct Buffer {
    chars: [[Volatile<ScreenChar>; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```
我们现在使用 `Volatile<ScreenChar>`来代替`ScreenChar`。(`Volatile`类型是一个[泛型]，因此它可以包装(几乎)任何类型)。这也保证了我们不会通过"普通"的写入操作意外向其写入数据。现在我们转而使用(Volatile)提供`write`方法。

[泛型]: https://doc.rust-lang.org/book/ch10-01-syntax.html

这也意味着我们需要修改我们的 `Writer::write_byte`方法:

```rust
// in src/vga_buffer.rs

impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(),
            byte => {
                ...

                self.buffer.chars[row][col].write(ScreenChar {
                    ascii_character: byte,
                    color_code: color_code,
                });
                ...
            }
        }
    }
    ...
}
```

现在我们使用`write`方法而不是使用通常的`=`赋值。这也保证了编译器永远不会将这个写操作优化掉。

### 格式化宏
支持Rsut的格式化宏似乎也是个不错的主意。通过这种方式，我们可以轻松的打印各种不同的类型，例如整型和浮点型。为了支持它们，我们需要实现 [`core::fmt::Write`] trait。这个trait唯一需要的方法是`write_str`，这个方法看起来和我们的`write_string`方法非常相似，只是它采用`fmt::Result`作为返回类型:

[`core::fmt::Write`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html

```rust
// in src/vga_buffer.rs

use core::fmt;

impl fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.write_string(s);
        Ok(())
    }
}
```

这里的 `Ok(())`是一个包含值为`()`的变量的Result(枚举)类型。

现在，我们可以使用Rust内置的 `write!`/`writeln!` 格式化宏了:

```rust
// in src/vga_buffer.rs

pub fn print_something() {
    use core::fmt::Write;
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };

    writer.write_byte(b'H');
    writer.write_string("ello! ");
    write!(writer, "The numbers are {} and {}", 42, 1.0/3.0).unwrap();
}
```

现在，你应该可以在屏幕地步看到一行`Hello! The numbers are 42 and 0.3333333333333333`了。`write!`宏调用会返回一个`Result`类型的值，这个值如果不被使用就会导致一个警告，所以，我们为它调用 [`unwrap`] 函数，当错误发生时它就会panic。不过由于向VGA缓冲区中写入的操作永远不会失败，因此，在我们的场景中不会有这个问题。

[`unwrap`]: https://doc.rust-lang.org/core/result/enum.Result.html#method.unwrap

### 换行
到目前为止，我们忽略了换行也没有处理字符超过一行时的情况。而现在，我们希望在换行时把所有的字符向上移动一行(最顶上的一行字符被删除)，然后在最后一行的起始位置继续打印。为了实现这个目标，我们需要为`Writer`添加一个`new_line`方法的实现：

```rust
// in src/vga_buffer.rs

impl Writer {
    fn new_line(&mut self) {
        for row in 1..BUFFER_HEIGHT {
            for col in 0..BUFFER_WIDTH {
                let character = self.buffer.chars[row][col].read();
                self.buffer.chars[row - 1][col].write(character);
            }
        }
        self.clear_row(BUFFER_HEIGHT - 1);
        self.column_position = 0;
    }

    fn clear_row(&mut self, row: usize) {/* TODO */}
}
```

我们遍历了屏幕上所有的字符并将每个字符向上移动了一行。注意，范围标记符(..)是左闭右开的。我们还忽略了第0行(第一层循环的范围是从`1`开始的)，因为这一行已经被移出屏幕了。

我们添加`clear_row`方法来完成换行的代码:

```rust
// in src/vga_buffer.rs

impl Writer {
    fn clear_row(&mut self, row: usize) {
        let blank = ScreenChar {
            ascii_character: b' ',
            color_code: self.color_code,
        };
        for col in 0..BUFFER_WIDTH {
            self.buffer.chars[row][col].write(blank);
        }
    }
}
```

这个方法通过将整行所有的字符替换为空格字符来清空一行。

## 全局接口
为了提供一个可供任意模块作为接口使用的不包含`Writer`实例的全局writer，我们试着创建了一个静态的`WRITER`:

```rust
// in src/vga_buffer.rs

pub static WRITER: Writer = Writer {
    column_position: 0,
    color_code: ColorCode::new(Color::Yellow, Color::Black),
    buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
};
```

然而，如果我们现在尝试编译这段代码，就会出现如下错误:

```
error[E0015]: calls in statics are limited to constant functions, tuple structs and tuple variants
 --> src/vga_buffer.rs:7:17
  |
7 |     color_code: ColorCode::new(Color::Yellow, Color::Black),
  |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0396]: raw pointers cannot be dereferenced in statics
 --> src/vga_buffer.rs:8:22
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ dereference of raw pointer in constant

error[E0017]: references in statics may only refer to immutable values
 --> src/vga_buffer.rs:8:22
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ statics require immutable values

error[E0017]: references in statics may only refer to immutable values
 --> src/vga_buffer.rs:8:13
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ statics require immutable values
```

为了搞清楚这里到底发生了什么，我们需要清楚一件事：普通变量在运行时初始化而静态变量在编译时初始化。Rust编译器中用来处理这些初始化表达式的组件称为 “[const evaluator(常量求值器)]”。目前它的功能还很有限，但是对它的扩展工作正在进行中，例如，"[允许在常量中panic]”这个RFC文档的中的工作。

[const evaluatorr(常量求值器)]: https://rust-lang.github.io/rustc-guide/const-eval.html
[允许在常量中panic]: https://github.com/rust-lang/rfcs/pull/2345

这个关于 `ColorCode::new`的问题可以通过使用 [`const` functions(常函数)]来解决，但是这里最根本的问题是Rust无法在编译时将裸指针转换为引用。或许将来某一天这个功能会实现，但是现在，我们需要寻找另一种解决方法。

[`const` functions(常函数)]: https://doc.rust-lang.org/unstable-book/language-features/const-fn.html

### Lazy Statics
在非常量函数中对静态变量进行一次性初始化是Rust中非常常见的问题。幸运的是，已经有一个叫做 [lazy_static]的crate给出了一个很好的解决方案。该crate提供了一个`lazy_static`宏可以延迟初始化`静态变量`。这个变量将在第一次使用时才初始化而不是在编译时初始化。由于所有的初始化过程都在运行时中发生，故无论多么复杂的初始化代码都是可行的。

[lazy_static]: https://docs.rs/lazy_static/1.0.1/lazy_static/

让我们将 `lazy_static` crate添加到我们的项目中去:

```toml
# in Cargo.toml

[dependencies.lazy_static]
version = "1.0"
features = ["spin_no_std"]
```

由于我们没有连接到标准库，这里我们需要`spin_no_std`特性。

有了`lazy_static`，我们定义静态`WRITER`时就不会有任何问题了:

```rust
// in src/vga_buffer.rs

use lazy_static::lazy_static;

lazy_static! {
    pub static ref WRITER: Writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };
}
```

然而，由于这个`WRITER`是不可变的所以它可能并没有什么用。这也就意味这我们没法向其中写入任何东西(因为所有的write方法都采用的`&mut self`)。一个可能的解决方法是使用 [mutable static(可变静态变量)]。但是这样一来，它的所有读写操作都会变成不安全的，因为这样会很容易引起数据竞争以及其他的各种问题。使用`static mut`是极度不推荐的。甚至还有人提议将其[移除][remove static mut]。那么有什么作为替代呢？我们可以试着用不可变静态变量的cell类型比如[RefCell]抑或是提供[内部可变性]的[UnsafeCell]。但是这些类型都不是[Sync]的(这有充分的理由)，所以我们无法在静态变量中使用它们。

[mutable static(可变静态变量)]: https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable
[remove static mut]: https://internals.rust-lang.org/t/pre-rfc-remove-static-mut/1437
[RefCell]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html#keeping-track-of-borrows-at-runtime-with-refcellt
[UnsafeCell]: https://doc.rust-lang.org/nightly/core/cell/struct.UnsafeCell.html
[interior mutability(内部可变性)]: https://doc.rust-lang.org/book/ch15-05-interior-mutability.html
[Sync]: https://doc.rust-lang.org/nightly/core/marker/trait.Sync.html

### 自旋锁
标准库用户通常会使用[Mutex]来获取同步的内部可变性。Mutex通过在资源被锁定时阻塞线程来实现互斥。但是我们的简单内核不支持阻塞甚至都没有线程这个概念，所以我们也就没法使用它。好在在计算机科学中有一种不依赖于操作系统特性的互斥锁:[spinlock(自旋锁)]。自旋锁不采用阻塞，而是在一个小循环中不停的锁定该资源，因此它会一直占用CPU资源(时间)直到互斥被再次释放。

[Mutex]: https://doc.rust-lang.org/nightly/std/sync/struct.Mutex.html
[spinlock(自旋锁)]: https://en.wikipedia.org/wiki/Spinlock

为了使用自旋锁，我们将[spin crate]添加到依赖中去:

[spin crate]: https://crates.io/crates/spin

```toml
# in Cargo.toml
[dependencies]
spin = "0.4.9"
```

现在我们可以通过自旋锁来为我们的静态`WRITER`添加安全的[interior mutability(内部可变性)]了。

```rust
// in src/vga_buffer.rs

use spin::Mutex;
...
lazy_static! {
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    });
}
```

现在我们可以删除 `print_something`函数并在我们的`_start`函数里直接进行打印了:

```rust
// in src/main.rs
#[no_mangle]
pub extern "C" fn _start() -> ! {
    use core::fmt::Write;
    vga_buffer::WRITER.lock().write_str("Hello again").unwrap();
    write!(vga_buffer::WRITER.lock(), ", some numbers: {} {}", 42, 1.337).unwrap();

    loop {}
}
```

这里我们需要将 `fmt::Write` trait导入这样我们才能使用其中的函数。

### Safety
注意，目前我们的代码中只有一个不安全的代码块，这是创建指向地址`0xb8000`的`缓冲区`的引用所必需的。除此之外，所有的操作都是安全的。Rust默认会对数组访问进行边界检查，所以我们不会意外的在缓冲区外进行写入操作(即越界)。因此，我们只需要实现类型系统所需的条件，就能向外部提供一个安全的接口。

### println宏
现在，我们已经有一个全局的writer了，我们可以在此基础上实现一个可以在代码的任意位置里使用的`println`宏。Rust的[macro syntax(宏语法)]有些奇怪，所以我们不会尝试从零开始编写一个宏。相对的，我们来看看标准库中的[`println!` 宏]的源码:

[macro syntax(宏语法)]: https://doc.rust-lang.org/nightly/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming
[`println!` 宏]: https://doc.rust-lang.org/nightly/std/macro.println!.html

```rust
#[macro_export]
macro_rules! println {
    () => (print!("\n"));
    ($($arg:tt)*) => (print!("{}\n", format_args!($($arg)*)));
}
```

宏是通过一个或多个规则定义的，类似于`match`语句的分支。`println`宏有两条规则:第一条规则是没有参数的调用(例如`println!()`)，它可以被扩展为 `print!("\n")`，因此只会打印一个换行符。第二条规则则是有参数的调用，诸如 `println!("Hello")`或是`println!("Number: {}", 4)`。它同样可以被扩展成 `print!`宏，可以传递所有的参数并在结尾附加一个换行符`\n`。

 `#[macro_export]`属性使得整个crate(不仅限于定义它的模块)以及外部的crate都可以使用它。它还将宏置于crate root下，这也就意味这我们需要通过 `use std::println`来导入而不是`std::macros::println`。

[`print!` 宏] 的定义如下:

[`print!` 宏]: https://doc.rust-lang.org/nightly/std/macro.print!.html

```rust
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print(format_args!($($arg)*)));
}
```

将这个宏展开后可以看到它是调用了`io`模块中的 [`_print` 函数]。[`$crate` 变量]可以通过扩展`std`来保证该宏在`std` crate之外也能正常使用。

 [`format_args` 宏]会从传入的参数中构建出一个  [fmt::Arguments]类型。libstd(标准库)中的[`_print` 函数]会调用`print_to`，这个函数非常复杂，因为它需要支持多种不同的`Stdout(标准输出)`设备。当然我们不需要这么搞得复杂，因为我们只需要打印到VGA缓冲区上就行了。

[`_print` 函数]: https://github.com/rust-lang/rust/blob/29f5c699b11a6a148f097f82eaa05202f8799bbc/src/libstd/io/stdio.rs#L698
[`$crate` 变量]: https://doc.rust-lang.org/1.30.0/book/first-edition/macros.html#the-variable-crate
[`format_args` 宏]: https://doc.rust-lang.org/nightly/std/macro.format_args.html
[fmt::Arguments]: https://doc.rust-lang.org/nightly/core/fmt/struct.Arguments.html

为了能够打印到VGA缓冲区，我们只需要复制 `println!` 和 `print!`宏，并将它们修改为使用我们自己的 `_print`函数即可:

```rust
// in src/vga_buffer.rs

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::vga_buffer::_print(format_args!($($arg)*)));
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}

#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    WRITER.lock().write_fmt(args).unwrap();
}
```

我们从最初的`println`定义中所做的一个重要的修改就是我们为`print!`宏也添加了`$crate` 前缀。这也就意味着如果我们只用`println`的话是不需要将`print!`宏也导入进来的。

就像标准库一样，我们为两种宏都添加了 `#[macro_export]` 属性，这样他们在我们的crate中的任意位置都是有效的。注意，这个操作也将宏放到了我们的crate的根命名空间下，所以通过`use crate::vga_buffer::println` 来导入是没用的。相应的，我们应该使用 `use crate::println`。

`_print`函数会锁定我们的 `WRITER`并调用 `write_fmt`方法。这个方法来自于`Writer` trait，所以我们需要导入这个trait。如果打印不成功，我们添加在结尾处的`unwrap()`就会panic。但是由于我们的 `write_str`总是返回`Ok`，所以理论上并不会发生这种情况。

由于该宏需要在模块外调用`_print`，所以这个函数应该被设置成共有的。然而，考虑到这是一个应该为私有的细节实现，我们为其添加 [`doc(hidden)` 属性]来防止它出现在生成的文档中。

[`doc(hidden)` 属性]: https://doc.rust-lang.org/nightly/rustdoc/the-doc-attribute.html#dochidden

### 用`println`实现的Hello World
现在我们可以在我们的`_start`函数中使用`println`了:

```rust
// in src/main.rs

#[no_mangle]
pub extern "C" fn _start() {
    println!("Hello World{}", "!");

    loop {}
}
```

注意我们不需要在main函数中导入该宏，因为它已经存在于根命名空间中了。

和我们预期的一样，现在我们在屏幕上看到了一行 *“Hello World!”*:

![QEMU printing “Hello World!”](vga-hello-world.png)

### 打印Panic信息

现在我们有了`println`宏，这样一来我们就可以在我们的panic函数中打印panic信息和panic发生的位置了:

```rust
// in main.rs

/// This function is called on panic.
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}
```

当我们将 `panic!("Some panic message");`插入到我们的`_start`函数中时，我们会得到以下的输出:

![QEMU printing “panicked at 'Some panic message', src/main.rs:28:5](vga-panic.png)

好了，现在我们不仅可以知道panic发生了，还可以看到panic信息并知道是在代码的哪个部分发生的panic。
## 总结
在这篇文章中，我们了解了VGA文本缓冲区的结构以及如何通过地址`0xb8000`的内存映射来对其进行写操作。我们还创建了一个Rust模块，它封装了向内存映射缓冲区写入的不安全的操作并向外部提供一个安全且方便的接口。

我们也发现了，多亏了cargo，向项目中添加第三方库依赖变得非常简单。我们添加的两个依赖`lazy_static` 和 `spin`在操作系统开发中都非常有用。在后续的文章中，我们还会在更多的地方用到它们。

## 下期预告
在下一篇文章中，我们会讲述如何配置Rust内置的单元测试框架。然后我们会为本文中的VGA缓冲区模块创建一些基本的单元测试。
