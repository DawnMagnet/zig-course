---
outline: deep
---

# 汇编

> 尽管现代高级语言的特性已经非常丰富，但我们仍需要汇编语言的帮助，在某些特殊的场景下，汇编语言可以发挥出比高级语言更好的性能，这是因为汇编语言更加接近硬件，它允许我们对硬件直接进行操作。

一般是在以下场景下，才会涉及到使用汇编语言：

1. 时效性高的程序，例如工业控制的程序。
2. 驱动程序，这需要直接操控硬件，由于高级语言的抽象层次过高，导致其不如汇编语言来的方便。
3. 内核的开发，现代化内核编写时均会使用汇编来完成一些初始化工作，如 bootloader，分段分页，中断处理等。
4. 程序的优化，高级语言的编译器并不是完美的，它有时会做出反而使程序变慢的“优化”，而汇编语言完全由程序员控制。

在 zig 中使用汇编有两种方式，引入外部的内联汇编，内联汇编大概是使用最多的情况。

::: info 🅿️ 提示

对于 x86 和 x86_64 ，当前汇编语法为 AT＆T 语法，而不是更流行的 Intel 语法。这是由于技术限制，汇编解析由LLVM提供，其对 Intel 语法的支持存在 bug 且测试​​结果并不理想。

在未来的某一天 Zig 可能有自己的汇编器。这将使汇编能够更加无缝地集成到语言中，并与流行的 Nasm 语法兼容。

:::

## 外部汇编

两种方式引入外部汇编，一种是在 `build.zig` 中使用 `addAssemblyFile` 添加汇编文件，另一种是通过 zig 本身的全局汇编功能。

这里讲述全局汇编功能：当汇编表达式出现在容器级 comptime 块中时，就是全局汇编。

::: info 🅿️ 提示

你可能对于**容器**这个概念比较疑惑，在 Zig 中，容器是充当保存变量和函数声明的命名空间的任何语法结构。容器也是可以实例化的类型定义。结构体、枚举、联合、不透明，甚至 Zig 源文件本身都是容器，但容器并不能包含语句（语句是描述程序运行操作的一个单位）。

当然，你也可以这样理解：容器是一个只包含变量或常量定义以及函数定义的命名空间。

注意：容器和块（block）不同！

:::

它的实际作用就和使用 `addAssemblyFile` 的效果类似，在编译期它们会被提取出来作为单独的汇编文件进行编译和链接。

```zig
const std = @import("std");

comptime {
    asm (
        \\.global my_func;
        \\.type my_func, @function;
        \\my_func:
        \\  lea (%rdi,%rsi,1),%eax
        \\  retq
    );
}

extern fn my_func(a: i32, b: i32) i32;

pub fn main() void {
    std.debug.print("{}\n", .{my_func(2, 5)});
}
```

以上这段函数中，我们通过全局汇编定义了一个汇编函数，以实现加法功能，并在 `main` 中实现了调用，如果你想了解更多这些相关的内容，你可以继续查询有关**调用约定**（**Calling convention**）的资料。

## 内联汇编

内联汇编给予了我们可以将 `low-level` 的汇编代码和高级语言相组合，实现更加高效或者更直白的操作。

```zig
pub fn main() noreturn {
    const msg = "hello world\n";
    _ = syscall3(SYS_write, STDOUT_FILENO, @intFromPtr(msg), msg.len);
    _ = syscall1(SYS_exit, 0);
    unreachable;
}

pub const SYS_write = 1;
pub const SYS_exit = 60;

pub const STDOUT_FILENO = 1;

pub fn syscall1(number: usize, arg1: usize) usize {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> usize),
        : [number] "{rax}" (number),
          [arg1] "{rdi}" (arg1),
        : "rcx", "r11"
    );
}

pub fn syscall3(number: usize, arg1: usize, arg2: usize, arg3: usize) usize {
    return asm volatile ("syscall"
        : [ret] "={rax}" (-> usize),
        : [number] "{rax}" (number),
          [arg1] "{rdi}" (arg1),
          [arg2] "{rsi}" (arg2),
          [arg3] "{rdx}" (arg3),
        : "rcx", "r11"
    );
}
```

上面这段代码是通过内联汇编实现在 x86-64 linux 下输出 `hello world`，接下来讲解一下它们的组成和使用。

内联汇编是以 `asm` 关键字开头的一个表达式，这说明它可以返回值（也可以不返回值），`volatile` 关键字会通知编译器，内联汇编的表达式会被某些编译器未知的因素更改（例如操作系统，硬件 MMIO 或者其他线程等等），这样编译器就不会额外优化这段内联汇编。

上面就是基本的内联汇编的一个外部结构说明，接下来我们介绍具体的内部结构：

```zig
asm volatile ("assembly code"
        : [ret] "={rax}" (-> usize),
        : [number] "{rax}" (number),
          [arg1] "{rdi}" (arg1),
          [arg2] "{rsi}" (arg2),
          [arg3] "{rdx}" (arg3),
        : "rcx", "r11"
    );
```

结构大体是这样的:

```asm
# 别忘记三个冒号，即便对应的部分不存在也需要有冒号
AssemblerTemplate
: OutputOperands
[ : InputOperands
[ : Clobbers ] ]
```

1. 首先是一个内联汇编的语句，但它和普通的内联语句不同，它可以使用“占位符”，类似`%[value]`，这就是一个占位符，以 `%` 开头，如果需要使用寄存器，则需要使用两个 `%` ，例如使用 CR3 寄存器就是 `%%cr3`。
2. 之后是一个输出位置，它表示你需要将值输出到哪里，也可以没有返回值，例如上方的示例中 `[ret] "={rax}" (-> usize)` 代表我们使用 `[ret]` 标记了返回值，并且返回值就是 rax 寄存器中的值，其后的 `(-> usize)` 代表我们整个内联汇编表达式需要返回一个值，当然这里如果是一个变量，就会将rax寄存器的值通过`[ret]`标记绑定到变量上。（注意，此处的 `=` 代表只能进行写入操作数，属于是一个约束。）
3. 这是输入操作数，它和输出位置类似，但它可以存在多个输入，并且它也支持“占位符”和相关的约束。
4. 这里是改动的寄存器，用于通知编译器，我们在执行此内联汇编会使用（或者称之为破坏更合适）的寄存器，默认包含了输入和输出寄存器。还有一个特殊标记 `memory`，它会通知编译器内联汇编会写入任意为声明的内存位置。

::: info 🅿️ 提示

关于更多的内联汇编约束信息，你可以阅读这里：[LLVM documentation](http://releases.llvm.org/10.0.0/docs/LangRef.html#inline-asm-constraint-string)，[GCC documentation](https://gcc.gnu.org/onlinedocs/gcc/Extended-Asm.html)。

:::

::: warning

内联汇编特性在未来可能会发生更改以支持新的特性（如多个返回值），具体见此 [issue](https://github.com/ziglang/zig/issues/215)。

:::

了解更多？可以查看我的这篇文章 [Handle Interrupt on x86-64 kernel with zig](https://blog.nvimer.org/2023/11/12/handle-interrupt-on-x86-64-kernel-with-zig/)，这是一篇关于如何在 x86-64 内核上利用 zig 的特性实现中断处理的文章，在这其中我用了内联汇编，算是一个比较巧妙的例子。