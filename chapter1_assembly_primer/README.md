<!-- Copyright © 2018 Clement Rey <cr.rey.clement@gmail.com>. -->
<!-- Licensed under the BY-NC-SA Creative Commons 4.0 International Public License. -->

# 第一章: Go 汇编入门

在深入学习 runtime 和标准库的实现之前，我们需要先对 Go 的汇编有一定的熟练度。这份快速指南希望能够加速你的学习进程。

- *本章假设你已经对某一种汇编器的基础知识有所了解*
- *涉及到架构相关的情况时，请假设我们是运行在 `linux/amd64` 平台上*
- *学习过程中编译器优化会**打开**。*

---

**目录**
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- ["伪汇编"](#%E4%BC%AA%E6%B1%87%E7%BC%96)
- [拆解一个简单程序](#%E6%8B%86%E8%A7%A3%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%A8%8B%E5%BA%8F)
  - [解剖 `add`](#%E8%A7%A3%E5%89%96-add)
  - [解剖 `main`](#%E8%A7%A3%E5%89%96-main)
- [关于协程, 栈及栈分裂](#%E5%85%B3%E4%BA%8E%E5%8D%8F%E7%A8%8B-%E6%A0%88%E5%8F%8A%E6%A0%88%E5%88%86%E8%A3%82)
  - [栈](#%E6%A0%88)
  - [栈分裂](#%E6%A0%88%E5%88%86%E8%A3%82)
  - [缺失的细节](#%E7%BC%BA%E5%A4%B1%E7%9A%84%E7%BB%86%E8%8A%82)
- [总结](#%E6%80%BB%E7%BB%93)
- [链接](#%E9%93%BE%E6%8E%A5)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

---

*本章中的引用段落/注释都引用自官方文档或者 Go 的代码库，除非另外注明*

## "伪汇编"

Go 编译器会输出一种抽象可移植的汇编代码，这种汇编并不对应某种真实的硬件架构。Go 的汇编器会使用这种伪汇编，再为目标硬件生成具体的机器指令。

伪汇编这一个额外层可以带来很多好处，最主要的一点是方便将 Go 移植到新的架构上。相关的信息可以参考文后列出的 Rob Pike 的 *The Design of the Go Assembler*。

> The most important thing to know about Go's assembler is that it is not a direct representation of the underlying machine. Some of the details map precisely to the machine, but some do not. This is because the compiler suite needs no assembler pass in the usual pipeline. Instead, the compiler operates on a kind of semi-abstract instruction set, and instruction selection occurs partly after code generation. The assembler works on the semi-abstract form, so when you see an instruction like MOV what the toolchain actually generates for that operation might not be a move instruction at all, perhaps a clear or load. Or it might correspond exactly to the machine instruction with that name. In general, machine-specific operations tend to appear as themselves, while more general concepts like memory move and subroutine call and return are more abstract. The details vary with architecture, and we apologize for the imprecision; the situation is not well-defined.

> The assembler program is a way to parse a description of that semi-abstract instruction set and turn it into instructions to be input to the linker.

## 拆解一个简单程序

思考一下下面这段 Go 代码 ([direct_topfunc_call.go](./direct_topfunc_call.go)):

```Go
//go:noinline
func add(a, b int32) (int32, bool) { return a + b, true }

func main() { add(10, 32) }
```
*(注意这里的 `//go:noinline` 编译器指令。。不要省略掉这部分)*

将这段代码编译到汇编:
```
$ GOOS=linux GOARCH=amd64 go tool compile -S direct_topfunc_call.go
```
```Assembly
0x0000 TEXT		"".add(SB), NOSPLIT, $0-16
  0x0000 FUNCDATA	$0, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
  0x0000 FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x0000 MOVL		"".b+12(SP), AX
  0x0004 MOVL		"".a+8(SP), CX
  0x0008 ADDL		CX, AX
  0x000a MOVL		AX, "".~r2+16(SP)
  0x000e MOVB		$1, "".~r3+20(SP)
  0x0013 RET

0x0000 TEXT		"".main(SB), $24-0
  ;; ...omitted stack-split prologue...
  0x000f SUBQ		$24, SP
  0x0013 MOVQ		BP, 16(SP)
  0x0018 LEAQ		16(SP), BP
  0x001d FUNCDATA	$0, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d FUNCDATA	$1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
  0x001d MOVQ		$137438953482, AX
  0x0027 MOVQ		AX, (SP)
  0x002b PCDATA		$0, $0
  0x002b CALL		"".add(SB)
  0x0030 MOVQ		16(SP), BP
  0x0035 ADDQ		$24, SP
  0x0039 RET
  ;; ...omitted stack-split epilogue...
```

接下来一行一行地对这两个函数进行解析来帮助我们理解编译器在编译期间都做了什么事情。

### 解剖 `add`

```Assembly
0x0000 TEXT "".add(SB), NOSPLIT, $0-16
```

- `0x0000`: 当前指令相对于当前函数的偏移量。

- `TEXT "".add`: `TEXT` 指令声明了 `"".add` 是 `.text` 段(程序代码在运行期会放在内存的 .text 段中)的一部分，并表明跟在这个声明后的是函数的函数体。
在链接期，`""` 这个空字符会被替换为当前的包名: 也就是说，`"".add` 在链接到二进制文件后会变成 `main.add`。

- `(SB)`: `SB` 是一个虚拟寄存器，保存了静态基地址(static-base) 指针，即我们程序地址空间的开始地址。
`"".add(SB)` 表明我们的符号位于某个固定的相对地址空间起始处的偏移位置 (最终是由链接器计算得到的)。换句话来讲，它有一个直接的绝对地址: 是一个全局的函数符号。
`objdump` 这个工具能帮我们确认上面这些结论:

```
$ objdump -j .text -t direct_topfunc_call | grep 'main.add'
000000000044d980 g     F .text	000000000000000f main.add
```
> All user-defined symbols are written as offsets to the pseudo-registers FP (arguments and locals) and SB (globals).  
> The SB pseudo-register can be thought of as the origin of memory, so the symbol foo(SB) is the name foo as an address in memory.

- `NOSPLIT`: 向编译器表明*不应该*插入 *stack-split* 的用来检查栈需要扩张的前导指令。
在我们 `add` 函数的这种情况下，编译器自己帮我们插入了这个标记: 它足够聪明地意识到，由于 `add` 没有任何局部变量且没有它自己的栈帧，所以一定不会超出当前的栈；因此每次调用函数时在这里执行栈检查就是完全浪费 CPU 循环了。
> "NOSPLIT": Don't insert the preamble to check if the stack must be split. The frame for the routine, plus anything it calls, must fit in the spare space at the top of the stack segment. Used to protect routines such as the stack splitting code itself.  

本章结束时会对 goroutines 和 stack-splits 进行简单介绍。

- `$0-16`: `$0` 代表即将分配的栈帧大小；而 `$16` 指定了调用方传入的参数大小。
> In the general case, the frame size is followed by an argument size, separated by a minus sign. (It's not a subtraction, just idiosyncratic syntax.) The frame size $24-8 states that the function has a 24-byte frame and is called with 8 bytes of argument, which live on the caller's frame. If NOSPLIT is not specified for the TEXT, the argument size must be provided. For assembly functions with Go prototypes, go vet will check that the argument size is correct.

```Assembly
0x0000 FUNCDATA $0, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
0x0000 FUNCDATA $1, gclocals·33cdeccccebe80329f1fdbee7f5874cb(SB)
```

> The FUNCDATA and PCDATA directives contain information for use by the garbage collector; they are introduced by the compiler.

现在还不要对这个太上心；在本书深入探讨垃圾收集时，会再回来了解这些知识。

```Assembly
0x0000 MOVL "".b+12(SP), AX
0x0004 MOVL "".a+8(SP), CX
```

Go 的调用规约要求每一个参数都通过栈来传递，这部分空间由 caller 在其栈帧(stack frame)上提供。

调用其它过程之前，caller 就需要按照参数和返回变量的大小来对应地增长(返回后收缩)栈。

Go 编译器不会生成任何 PUSH/POP 族的指令: 栈的增长和收缩是通过在栈指针寄存器 `SP` 上分别执行减法和加法指令来实现的。

> The SP pseudo-register is a virtual stack pointer used to refer to frame-local variables and the arguments being prepared for function calls. It points to the top of the local stack frame, so references should use negative offsets in the range [−framesize, 0): x-8(SP), y-4(SP), and so on.

尽管官方文档说 "*All user-defined symbols are written as offsets to the pseudo-register FP(arguments and locals)*"，实际这个原则只是在手写的代码场景下才是有效的。
与大多数最近的编译器做法一样，Go 工具链总是在其生成的代码中，使用相对栈指针(stack-pointer)的偏移量来引用参数和局部变量。这样使得我们可以在那些寄存器数量较少的平台上(例如 x86)，也可以将帧指针(frame-pointer)作为一个额外的通用寄存器。
如果你喜欢了解这些细节问题，可以参考本章后提供的 *Stack frame layout on x86-64* 一文。

`"".b+12(SP)` 和 `"".a+8(SP)` 分别指向栈的低 12 字节和低 8 字节位置(记住: 栈是向低位地址方向增长的！)。
`.a` 和 `.b` 是分配给引用地址的任意别名；尽管 *它们没有任何语义上的含义* ，但在使用虚拟寄存器和相对地址时，这种别名是需要强制使用的。
虚拟寄存器帧指针(frame-pointer)的文档对此有所提及:
> The FP pseudo-register is a virtual frame pointer used to refer to function arguments. The compilers maintain a virtual frame pointer and refer to the arguments on the stack as offsets from that pseudo-register. Thus 0(FP) is the first argument to the function, 8(FP) is the second (on a 64-bit machine), and so on. However, when referring to a function argument this way, it is necessary to place a name at the beginning, as in first_arg+0(FP) and second_arg+8(FP). (The meaning of the offset —offset from the frame pointer— distinct from its use with SB, where it is an offset from the symbol.) The assembler enforces this convention, rejecting plain 0(FP) and 8(FP). The actual name is semantically irrelevant but should be used to document the argument's name.

最后，有两个重点需要指出:
1. 第一个变量 `a` 的地址并不是 `0(SP)`，而是在 `8(SP)`；这是因为调用方通过使用 `CALL` 伪指令，把其返回地址保存在了 `0(SP)` 位置。
2. 参数是反序传入的；也就是说，第一个参数和栈顶距离最近。

```Assembly
0x0008 ADDL CX, AX
0x000a MOVL AX, "".~r2+16(SP)
0x000e MOVB $1, "".~r3+20(SP)
```

`ADDL` 进行实际的加法操作，L 这里代表 **L**ong，4 字节的值，其将保存在 `AX` 和 `CX` 寄存器中的值进行相加，然后再保存进 `AX` 寄存器中。
这个结果之后被移动到 `"".~r2+16(SP)` 地址处，这是之前调用方专门为返回值预留的栈空间。这一次 `"".~r2` 同样没什么语义上的含义。

为了演示 Go 如何处理多返回值，我们同时返回了一个 bool 常量 `true`。
返回这个 bool 值的方法和之前返回数值的方法是一样的；只是相对于 `SP` 寄存器的偏移量发生了变化。

```Assembly
0x0013 RET
```

最后的 `RET` 伪指令告诉 Go 汇编器插入一些指令，这些指令是对应的目标平台中的调用规约所要求的，从子过程中返回时所需要的指令。
一般情况下这样的指令会使在 `0(SP)` 寄存器中保存的函数返回地址被 pop 出栈，并跳回到该地址。

> The last instruction in a TEXT block must be some sort of jump, usually a RET (pseudo-)instruction.
> (If it's not, the linker will append a jump-to-itself instruction; there is no fallthrough in TEXTs.)

我们一次性需要消化的语法和语义细节有点多。下面将我们已经覆盖到的知识点作为注释加进了汇编代码中:
```Assembly
;; Declare global function symbol "".add (actually main.add once linked)
;; Do not insert stack-split preamble
;; 0 bytes of stack-frame, 16 bytes of arguments passed in
;; func add(a, b int32) (int32, bool)
0x0000 TEXT	"".add(SB), NOSPLIT, $0-16
  ;; ...omitted FUNCDATA stuff...
  0x0000 MOVL	"".b+12(SP), AX	    ;; move second Long-word (4B) argument from caller's stack-frame into AX
  0x0004 MOVL	"".a+8(SP), CX	    ;; move first Long-word (4B) argument from caller's stack-frame into CX
  0x0008 ADDL	CX, AX		    ;; compute AX=CX+AX
  0x000a MOVL	AX, "".~r2+16(SP)   ;; move addition result (AX) into caller's stack-frame
  0x000e MOVB	$1, "".~r3+20(SP)   ;; move `true` boolean (constant) into caller's stack-frame
  0x0013 RET			    ;; jump to return address stored at 0(SP)
```

总之，下面是 `main.add` 即将执行 `RET` 指令时的栈的情况。

```
   |    +-------------------------+ <-- 32(SP)              
   |    |                         |                         
 G |    |                         |                         
 R |    |                         |                         
 O |    | main.main's saved       |                         
 W |    |     frame-pointer (BP)  |                         
 S |    |-------------------------| <-- 24(SP)              
   |    |      [alignment]        |                         
 D |    | "".~r3 (bool) = 1/true  | <-- 21(SP)              
 O |    |-------------------------| <-- 20(SP)              
 W |    |                         |                         
 N |    | "".~r2 (int32) = 42     |                         
 W |    |-------------------------| <-- 16(SP)              
 A |    |                         |                         
 R |    | "".b (int32) = 32       |                         
 D |    |-------------------------| <-- 12(SP)              
 S |    |                         |                         
   |    | "".a (int32) = 10       |                         
   |    |-------------------------| <-- 8(SP)               
   |    |                         |                         
   |    |                         |                         
   |    |                         |                         
 \ | /  | return address to       |                         
  \|/   |     main.main + 0x30    |                         
   -    +-------------------------+ <-- 0(SP) (TOP OF STACK)

(diagram made with https://textik.com)
```
<!-- https://textik.com/#af55d3485eaa56f2 -->

### 解剖 `main`

这里略去了一些代码帮你节省滚鼠标的时间，我们再次回忆一下 `main` 函数的逆向结果:
```Assembly
0x0000 TEXT		"".main(SB), $24-0
  ;; ...omitted stack-split prologue...
  0x000f SUBQ		$24, SP
  0x0013 MOVQ		BP, 16(SP)
  0x0018 LEAQ		16(SP), BP
  ;; ...omitted FUNCDATA stuff...
  0x001d MOVQ		$137438953482, AX
  0x0027 MOVQ		AX, (SP)
  ;; ...omitted PCDATA stuff...
  0x002b CALL		"".add(SB)
  0x0030 MOVQ		16(SP), BP
  0x0035 ADDQ		$24, SP
  0x0039 RET
  ;; ...omitted stack-split epilogue...
```

```Assembly
0x0000 TEXT "".main(SB), $24-0
```

没什么新东西:

- `"".main` (被链接之后名字会变成 `main.main`) 是一个全局的函数符号，存储在 `.text` 段中，该函数的地址是相对于整个地址空间起始位置的一个固定的偏移量。
- 它分配了 24 字节的栈帧，且不接收参数，不返回值。

```Assembly
0x000f SUBQ     $24, SP
0x0013 MOVQ     BP, 16(SP)
0x0018 LEAQ     16(SP), BP
```

上面我们已经提到过，Go 的调用规约强制我们将所有参数都通过栈来进行传递。

`main` 作为调用者，通过对虚拟栈指针(stack-pointer)寄存器做减法，将其栈帧大小增加了 24 个字节(*回忆一下栈是向低地址方向增长，所以这里的 `SUBQ` 指令是将栈帧的大小调整得更大了*)。
这 24 个字节中:

- 8 个字节(`16(SP)`-`24(SP)`) 用来存储当前帧指针 `BP` (这是一个实际存在的寄存器)的值，以支持栈的展开和方便调试
- 1+3 个字节(`12(SP)`-`16(SP)`) 是预留出的给第二个返回值 (`bool`) 的空间，除了类型本身的 1 个字节，在 `amd64` 平台上还额外需要 3 个字节来做对齐
- 4 个字节(`8(SP)`-`12(SP)`) 预留给第一个返回值 (`int32`)
- 4 个字节(`4(SP)`-`8(SP)`) 是预留给传给被调用函数的参数 `b (int32)`
- 4 个字节(`0(SP)`-`4(SP)`) 预留给传入参数 `a (int32)`

最后，跟着栈的增长，`LEAQ` 指令计算出帧指针的新地址，并将其存储到 `BP` 寄存器中。

```Assembly
0x001d MOVQ     $137438953482, AX
0x0027 MOVQ     AX, (SP)
```

调用方将被调用方需要的参数作为一个 **Q**uad word(8 字节值)推到了刚刚增长的栈的栈顶。

尽管指令里出现的 `137438953482` 这个值看起来像是随机的垃圾值，实际上这个值对应的就是 `10` 和 `32` 这两个 4 字节值，它们两被连接成了一个 8 字节值。

```
$ echo 'obase=2;137438953482' | bc
10000000000000000000000000000000001010
\_____/\_____________________________/
   32                             10
```

```Assembly
0x002b CALL     "".add(SB)
```

我们使用相对于 static-base 指针的偏移量，来对 `add` 函数进行 `CALL` 调用: 这种调用实际上相当于直接跳到一个指定的地址。

注意 `CALL` 指令还会将函数的返回地址(8 字节值)也推到栈顶；所以每次我们在 `add` 函数中引用 `SP` 寄存器的时候还需要额外偏移 8 个字节！
例如，`"".a` 现在不是 `0(SP)` 了，而是在 `8(SP)` 位置。

```Assembly
0x0030 MOVQ     16(SP), BP
0x0035 ADDQ     $24, SP
0x0039 RET
```

最后，我们:

1. 将帧指针(frame-pointer)下降一个栈帧(stack-frame)的大小(就是“向下”一级)
2. 将栈收缩 24 个字节，回收之前分配的栈空间
3. 请求 Go 汇编器插入子过程返回相关的指令

## 关于协程, 栈及栈分裂

现在还不是能够深入 goroutine 内部实现的合适时间点(*这部分会在之后讲解*)，不过随着我们一遍遍 dump 出程序的汇编代码，栈管理相关的指令会越来越熟悉。
这样我们就可以快速地看出代码的模式，并且可以理解这些代码一般情况下在做什么，为什么要做这些事情。

### 栈

由于 Go 程序中的 goroutine 数目是不可确定的，并且实际场景可能会有百万级别的 goroutine，runtime 必须使用保守的思路来给 goroutine 分配空间以避免吃掉所有的可用内存。

也由于此，每个新的 goroutine 会被 runtime 分配初始为 2KB 大小的栈空间(Go 的栈在底层实际上是分配在堆空间上的)。

随着一个 goroutine 进行自己的工作，可能会超出最初分配的栈空间限制(就是栈溢出的意思)。
为了防止这种情况发生，runtime 确保 goroutine 在超出栈范围时，会创建一个相当于原来两倍大小的新栈，并将原来栈的上下文拷贝到新栈上。
这个过程被称为 *栈分裂*(stack-split)，这样使得 goroutine 栈能够动态调整大小。

### 栈分裂

为了使栈分裂正常工作，编译器会在每一个函数的开头和结束位置插入指令来防止 goroutine 爆栈。
像我们本章早些看到的一样，为了避免不必要的开销，一定不会爆栈的函数会被标记上 `NOSPLIT` 来提示编译器不要在这些函数的开头和结束部分插入这些检查指令。

我们来看看之前的 main 函数，这次不再省略栈分裂的前导指令:

```Assembly
0x0000 TEXT	"".main(SB), $24-0
  ;; stack-split prologue
  0x0000 MOVQ	(TLS), CX
  0x0009 CMPQ	SP, 16(CX)
  0x000d JLS	58

  0x000f SUBQ	$24, SP
  0x0013 MOVQ	BP, 16(SP)
  0x0018 LEAQ	16(SP), BP
  ;; ...omitted FUNCDATA stuff...
  0x001d MOVQ	$137438953482, AX
  0x0027 MOVQ	AX, (SP)
  ;; ...omitted PCDATA stuff...
  0x002b CALL	"".add(SB)
  0x0030 MOVQ	16(SP), BP
  0x0035 ADDQ	$24, SP
  0x0039 RET

  ;; stack-split epilogue
  0x003a NOP
  ;; ...omitted PCDATA stuff...
  0x003a CALL	runtime.morestack_noctxt(SB)
  0x003f JMP	0
```

可以看到，栈分裂(stack-split)前导码被分成 prologue 和 epilogue 两个部分:

- prologue 会检查当前 goroutine 是否已经用完了所有的空间，然后如果确实用完了的话，会直接跳转到后部。
- epilogue 会触发栈增长(stack-growth)，然后再跳回到前部。

这样就形成了一个反馈循环，使我们的栈在没有达到饥饿的 goroutine 要求之前不断地进行空间扩张。

**Prologue**
```Assembly
0x0000 MOVQ	(TLS), CX   ;; store current *g in CX
0x0009 CMPQ	SP, 16(CX)  ;; compare SP and g.stackguard0
0x000d JLS	58	    ;; jumps to 0x3a if SP <= g.stackguard0
```

`TLS` 是一个由 runtime 维护的虚拟寄存器，保存了指向当前 `g` 的指针，这个 `g` 的数据结构会跟踪 goroutine 运行时的所有状态值。

看一看 runtime 源代码中对于 `g` 的定义:
```Go
type g struct {
	stack       stack   // 16 bytes
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	stackguard0 uintptr
	stackguard1 uintptr

	// ...omitted dozens of fields...
}
```
我们可以看到 `16(CX)` 对应的是 `g.stackguard0`，是 runtime 维护的一个阈值，该值会被拿来与栈指针(stack-pointer)进行比较以判断一个 goroutine 是否马上要用完当前的栈空间。

因此 prologue 只要检查当前的 `SP` 的值是否小于或等于 `stackguard0` 的阈值就行了，如果是的话，就跳到 epilogue 部分去。

**Epilogue**
```Assembly
0x003a NOP
0x003a CALL	runtime.morestack_noctxt(SB)
0x003f JMP	0
```

epilogue 部分的代码就很直来直去了: 它直接调用 runtime 的函数，对应的函数会将栈进行扩张，然后再跳回到函数的第一条指令去(就是指 prologue部分)。

在 `CALL` 之前出现的 `NOP` 这个指令使 prologue 部分不会直接跳到 `CALL` 指令位置。在一些平台上，这样能够，直接跳到 `CALL` 可能会有一些麻烦的问题；所以在调用位置插一个 noop 的指令并在跳转时跳到这个 `NOP` 位置是一种最佳实践。

### 缺失的细节

本章的内容只是冰山一角。
栈的调整涉及的技术还有很多精妙的细节，这里暂时还没有提到。整个流程是一个非常复杂的流程，需要单独的一个章节来进行阐释。

之后我们会再回来讨论这些细节。

## 总结

对 Go 的汇编器的介绍应该已经为你提供了开始学习的足够的材料。

随着本书剩余部分对 Go 内部原理越来越深入的探究，Go 的汇编会是我们最为依仗的工具，用来帮助我们理解现象背后的那些不总是那么明显的实质。

如果你有问题或者建议，不要犹豫，开一个蛓有 `chapter1:` 前缀的 issue 即可！

## 链接

- [[Official] A Quick Guide to Go's Assembler](https://golang.org/doc/asm)
- [[Official] Go Compiler Directives](https://golang.org/cmd/compile/#hdr-Compiler_Directives)
- [[Official] The design of the Go Assembler](https://www.youtube.com/watch?v=KINIAgRpkDA)
- [[Official] Contiguous stacks Design Document](https://docs.google.com/document/d/1wAaf1rYoM4S4gtnPh0zOlGzWtrZFQ5suE8qr2sD8uWQ/pub)
- [[Official] The `_StackMin` constant](https://github.com/golang/go/blob/ea8d7a370d66550d587414cc0cab650f35400f94/src/runtime/stack.go#L70-L71)
- [A Foray Into Go Assembly Programming](https://blog.sgmansfield.com/2017/04/a-foray-into-go-assembly-programming/)
- [Dropping Down Go Functions in Assembly](https://www.youtube.com/watch?v=9jpnFmJr2PE)
- [What is the purpose of the EBP frame pointer register?](https://stackoverflow.com/questions/579262/what-is-the-purpose-of-the-ebp-frame-pointer-register)
- [Stack frame layout on x86-64](https://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64)
- [How Stacks are Handled in Go](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
- [Why stack grows down](https://gist.github.com/cpq/8598782)
