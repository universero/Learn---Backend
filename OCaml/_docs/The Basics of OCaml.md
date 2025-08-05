___
## The OCaml Toplevel

OCaml的_顶层_ 就像计算器或命令行界面。它类似于 Java 的 JShell，或者交互式 Python 解释器。顶层方便您尝试小段代码，而无需费力启动 OCaml 编译器。但不要过度依赖它，因为创建、编译和测试大型程序需要更强大的工具。一些其他语言将顶层称为 _REPL_ ，它代表读取-求值-打印-循环：它读取程序员的输入，对其进行求值，打印结果，然后重复此过程。

在终端窗口中，输入 `utop` 启动顶层。按 Control-D 退出顶层。您也可以输入 `#quit;;` 并按回车键。请注意，您必须在此处输入 `#` ：它是对您已看到的 `#` 提示符的补充。

### Types and values

你可以在 OCaml 顶层输入表达式。用双分号 `;;` 结束表达式，然后按回车键。OCaml 会计算该表达式，并显示结果值及其类型
```ocaml
42
// - : int = 42
```
你可以使用如下方式将值与名称绑定
```ocaml
let x = 42
val x : int = 42
```
### Functions

使用如下语法定义函数
```ocaml
let increment x = x + 1
// val increment : int -> int = <fun>
```
`int -> int` 是值的类型。这类函数以 `int` 为输入，输出也为 `int` 。可以将箭头 `->` 视为一种视觉隐喻，表示将一个值转换为另一个值——这正是函数的作用。

`<fun>` itself is not a value. It just indicates an unprintable function value.  
`<fun>` 本身不是一个值。它只是表示一个不可打印的函数值。

使用如下方法调用函数
```ocaml
increment 0
increment(21)
increment(increment 5)
```
但在 OCaml 中，通常的词汇是“应用”函数而不是“调用”它。(apply instead of call)

请注意，OCaml 对于是否使用括号以及是否使用空格非常灵活。初学 OCaml 的挑战之一可能是搞清楚何时需要使用括号。因此，如果您遇到语法错误，一种策略是尝试添加一些括号。不过，首选的样式通常是在不需要括号时省略它们。因此， `increment 21` 比 `increment(21)` 更好。
### Loading code in the toplevel

除了允许你定义函数之外，顶层还可以接受非 OCaml 代码的  _指令_ ，指示顶层本身执行某项操作。所有指令都以 `#` 字符开头。也许最常见的指令是 `#use` ，它会将文件中的所有代码加载到顶层，就像您将该文件中的代码输入到顶层一样。
如使用如下命令导入mycode.ml文件
```ocaml
# #use "mycode.ml";;
```
### Workflow in the toplevel

使用顶层的最佳工作流程是
- 编辑文件中的代码。
- 使用 `#use` 在顶层加载代码。
- 交互式地测试代码。
- 退出顶层。 **警告：** 不要跳过此步骤。
假设你想修复代码中的一个错误。你可能不想退出顶层，编辑文件，然后在同一个顶层会话中重新执行 `#use` 指令。但要克制住这种想法。在同一会话中，从之前的 `#use` 指令加载的“过时代码”可能会导致意想不到的情况发生——至少在你刚开始学习这门语言的时候是会出乎意料的。所以，在重新使用文件之前，务必退出顶层。

## Compiling OCaml Programs

使用如下命令编译源文件并生成指定的OCaml字节码, 此外还会生成.cmi和.cmo, 我们暂时不用关系这两文件
```ocaml
ocamlc -o hello.byte hello.ml
```
`./hello.byte`即可执行OCaml程序

与 C 或 Java 不同，OCaml 程序不需要有一个名为 `main` 函数用于启动程序。通常的做法是将文件中的最后一个定义作为主函数，启动所有需要进行的计算。

在大型项目中，我们不想运行编译器或手动清理。相反，我们希望使用 _构建系统_ 来自动查找和链接库。OCaml 有一个名为 ocamlbuild 的旧构建系统，以及一个名为 Dune 的较新的构建系统。类似的系统包括 C 语言用`make` ，以及Java用Gradle、Maven 和 Ant

### Dune

在名为dune的文件中输入如下内容, 这声明了一个可执行文件, 主文件是hello.ml
```dune
(executable
 (name hello))
```
还要创建一个dune-project文件, 输入如下内容, 这告诉了Dune使用dune 3.4版本
```dune
(lang dune 3.4)
```
通常，源代码树的每个子目录中都会有一个 `dune` 文件，但只有根目录中一个 `dune-project` 文件。
```shell
dune build hello.exe
```
dune在所有平台上都使用exe扩展名, dune构建的是原生可执行文件而不是字节码执行文件

Dune 会创建一个目录 `_build` 并在其中编译我们的程序。这是构建系统相对于直接运行编译器的一个好处：它们不会用一堆生成的文件污染你的源目录，而是干净地创建在一个单独的目录中 `_build` 内部有许多由 Dune 创建的文件。我们的可执行文件被埋藏在下面的几层
但我们只需要使用`dune exec ./hello.exe`就可以运行, 通过dune clean就能清楚编译产生的内容
`dune init project name`可以初始化dune项目
`dune build --watch`可以热更新, 每次保存文件都自动重新编译并运行

## Expressions

正如命令式语言的核心是命令一样, OCaml语法的核心是表达式.
函数式语言中计算的主要任务是将表达式求值, 值是指没有剩余需要执行计算的表达式, 因此所有值都是表达式, 但不是所有表达式都是值

表达式无法求值存在两种情况: 表达式的计算引发了异常, 表达式计算永不终止.
### 原始类型和值

原始类型是内置的最基本的类型：整数、浮点数、字符、字符串和布尔值。它们与其他编程语言中的原始类型类似，易于识别。
- 类型 `int` ：整数。OCaml 整数的写法与常规写法相同： `1` 、 `2` 等。常用运算符有： `+` 、 `-` 、 `*` 、 `/` 和 `mod` 。后两个运算符是整数除法和模数
	- 在现代平台上，OCaml 整数使用 64 位机器字实现，相当于 64 位处理器上寄存器的大小。但 OCaml 实现会“窃取”其中一位，从而将其转换为 63 位表示。该位在运行时用于区分整数和指针。对于需要真正 64 位整数的应用程序，标准库中有一个 `Int64` 模块。对于需要任意精度整数的应用程序，有一个单独的 `Zarith` 库。但在大多数情况下，内置的 `int` 类型足以满足需求，并提供最佳性能。
- 类型 `float` ：浮点数。OCaml 浮点数是 IEEE 754 定义的双精度浮点数。语法上，它们必须始终包含一个点，例如 `3.14` 、 `3.0` 甚至 `3.` 。最后一个是 `float` ；如果你写成 `3` ，它实际上是 `int` 
	- OCaml 刻意不支持运算符重载，浮点数的算术运算在后面加一个点。例如，浮点乘法写成 `*.` 而不是 `*`
	- OCaml 不会自动在 `int` 和 `float` 之间转换。如果需要转换，可以使用两个内置函数： `int_of_float` 和 `float_of_int` 。
	- 与任何语言一样，浮点数的表示形式都是近似的。这可能会导致舍入误差
- 类型 `bool` ：布尔值。布尔值写作 `true` 和 `false` 。常用的短路合取运算符 `&&` 和析取运算符 `||` 均可用。
- 类型 `char` ：字符。字符用单引号括起来，例如 `'a'` 、 `'b'` 和 `'c'` 。它们以字节（即 8 位整数）表示，符合 ISO 8859-1“Latin-1”编码。该范围内的前半部分字符是标准 ASCII 字符。您可以使用 `char_of_int` 和 `int_of_char` 在字符和整数之间进行转换。
- 类型 `string` ：字符串。字符串是字符序列。它们用双引号括起来，例如 `"abc"` 。字符串连接运算符为 `^`
	- 面向对象语言通常提供可重写的方法将对象转换为字符串，例如 Java 中的 `toString()` 或 Python 中的 `__str__()` 。但大多数 OCaml 值并非对象，因此需要其他方法将其转换为字符串。对于三种原始类型，内置函数有： `string_of_int` 、 `string_of_float` 、 `string_of_bool` 。奇怪的是，没有 `string_of_char` ，但可以使用库函数 `String.make` 来实现相同的目的。
	- 同样，对于相同的三种原始类型，如果可能的话，有内置函数可以从字符串转换： `int_of_string` ， `float_of_string` 和 `bool_of_string` 。
### 更多操作符

OCaml 中有两个相等运算符： `=` 和 `==` ，以及对应的不等运算符 `<>` 和 `!=` 。运算符 `=` 和 `<>` 检查结构相等，而 `==` 和 `!=` 检查物理相等。在我们学习完 OCaml 的命令式特性之前，它们之间的区别很难解释。如果您现在感兴趣，请参阅 `Stdlib.(==)` 的文档。

表达式 `assert e` 求值 `e` 。如果结果为 `true` ，则不发生任何其他操作，整个表达式求值结果为一个称为 unit 的特殊值。该 unit 值写作 `()` ，其类型为 `unit` 。但如果结果为 `false` ，则会引发异常。

如果 `e1` 计算结果为 `true` ，则表达式 `if e1 then e2 else e3` 计算结果为 `e2` ，否则计算结果为 `e3` 。我们称 `e1` 为 `if` 表达式的guard。

与您可能在命令式语言中使用过的 `if-then-else` 语句不同，OCaml 中的 `if-then-else` 表达式与任何其他表达式一样；它们可以放在表达式允许的任何位置。这使得它们类似于您可能在其他语言中使用过的三元运算符 `? :` 。

每当我们要写“ `e` has type `t` ”时，我们都会写成 `e : t` 。冒号的发音是“has type”。这种冒号的用法与顶层在计算你输入的表达式后的响应方式一致

`let` 还有另一种用途，即作为表达式：
let x = 42 in x + 1
这里我们将一个值绑定到名称 `x` ，然后在另一个表达式 `x+1` 中使用该绑定。我们将这种对 `let` 的使用称为 let 表达式。由于它是一个表达式，所以它的计算结果为一个值。这与定义不同，定义本身不会计算任何值。如果你尝试用 let 定义代替需要表达式的位置，你会发现这一点：

### Scope

`Let` 绑定仅在其所在的代码块中有效。这正是您在几乎所有现代编程语言中都习惯的做法

名称无关性: 和函数一样, 自变量叫什么无所谓
```ocaml
# let x = 42;;
val x : int = 42
# let x = 22;;
val x : int = 22
```
这里不是修改了x的值, 而是得到两个不同的表达式
### 类型注解

OCaml可以推断类型, 但有时用 x : t 指定类型还是有用的, 不正确的指定会编译错误

类型注解并非像 C 或 Java 中那样进行类型转换。它们并不表示从一种类型到另一种类型的转换。而是表示检查表达式是否确实具有给定的类型。
## Function

方法和函数不是一回事。方法是对象的组件，它隐式地拥有一个接收者，通常使用类似 `this` 或 `self` 的关键字来访问。OCaml 函数不是方法：它们不是对象的组件，也没有接收者。
### 函数定义

非递归函数定义: `let f x = ...`
递归函数定义: `let rec f x = ...`
阶乘函数
```ocaml
let rec fact n = if n = 0 then 1 else n * fact (n - 1)
```
如果想定义类型可以:
```ocaml
let rec pow (x : int) (y : int) : int = ...
```
可以使用`and`关键字定义相互递归函数
```ocaml
let rec f x1 ... xn = e1
and g y1 ... yn = e2
```
函数类型的语法是
```ocaml
t -> u
t1 -> t2 -> u
t1 -> ... -> tn -> u
```
`t` 和 `u` 是表示类型的元变量。类型 `t -> u` 表示一个函数的类型，它接受类型 `t` 的输入，并返回类型 `u` 的输出。我们可以将 `t1 -> t2 -> u` 视为一个函数的类型，它接受两个输入，第一个输入的类型为 `t1` ，第二个输入的类型为 `t2` ，并返回类型 `u` 的输出。接受 `n` 个参数的函数也是如此。
### 匿名函数

匿名函数也称为 Lambda 表达式，这个术语源自 Lambda 演算。Lambda 演算是一种计算的数学模型，其含义与图灵机是一种计算模型相同。在 Lambda 演算中， `fun x -> e` 可以写成  λx.e。  λ表示匿名函数。
### 函数应用

```ocaml
e0 e1 e2 ... en
```
第一个表达式 `e0` 是函数，它应用于参数 `e1` 到 `en`
### 管道

由于 `e1 |> e2` 只是 `e2 e1` 的另一种写法，我们无需说明 `|>` 的语义：它与函数应用完全相同。这两个程序是语法不同但语义等价的表达式的另一个例子。
### 多态函数

identity function是简单返回其输入的函数, 如 `let id x = x`
类型是`val id : 'a -> 'a = <fun>`
`'a` 是类型变量：它代表未知类型，就像常规变量代表未知值一样。类型变量始终以单引号开头。常用的类型变量包括 `'a` 、 `'b` 和 `'c` ，OCaml 程序员通常用希腊语发音：alpha、beta 和 gamma。 这类函数可以用于任意类型
可以手动指定类型
```ocaml
let id_int (x : int) : int = x
let id_int' : int -> int = id
```
### Labeled and Optional Arguments

函数的类型和名称通常能让你很好地了解其参数应该是什么。但是，对于具有多个参数（尤其是相同类型的参数）的函数，给它们加上标签会很有用。例如，你可能会猜测函数 `String.sub` 返回给定字符串的子字符串（而且你猜对了）。你可以输入 `String.sub` 来查找它的类型：
```ocaml
String.sub;;
- : string -> int -> int -> string = <fun>
```
OCaml 支持函数的带标签参数。你可以使用以下语法声明此类函数：
```ocaml
let f ~name1:arg1 ~name2:arg2 = arg1 + arg2;;
let f ~name1 ~name2 = name1 + name2;;
// 可以按任意顺序传参调用
f ~name2:3 ~name1:4
```
### 偏应用

一个多个参数的函数, 传入一个参数后, 得到的是一个部分应用了的函数, 具有更少的参数
```ocaml
let add x y = x + y
val add : int -> int -> int = <fun>

let addx x = fun y -> x + y
val addx : int -> int -> int = <fun>

let add5 = addx 5
val add5 : int -> int = <fun>
add5 2
- : int = 7
```
addx和add都能实现相同的效果
### 函数结合性

**Every OCaml function takes exactly one argument.  
每个 OCaml 函数都只接受一个参数。**
```ocaml
let f =
  fun x1 ->
    (fun x2 ->
       (...
          (fun xn -> e)...))
```
尽管您将 `f` 视为接受 `n` 个参数的函数，但实际上它是一个接受 1 个参数并返回一个函数的函数。
t1 -> t2 -> t3 -> t4意味着t1 -> (t2 -> (t3 -> t4))
函数类型是右结合的：函数类型周围有隐式括号，从右到左。这里的直觉是，一个函数接受一个参数，并返回一个接受其余参数的新函数。
另一方面，函数应用是左结合的：函数应用周围有隐式括号，从左到右。
e1 e2 e3 e4 意味着 ((e1 e2) e3) e4
### 运算符作为函数

加法运算符 `+` 的类型为 `int -> int -> int` 。它通常写成中缀运算符，例如 `3 + 4` 。通过在其周围加上括号，我们可以使其成为前缀运算符：
```ocaml
( + )
- : int -> int -> int = <fun>
( + ) 3 4;;
- : int = 7
```
通常情况下，空格是不必要的。我们可以写成 `(+)` 或 `( + )` ，但最好还是包含空格。注意乘法，必须写成 `( * )` ，因为 `(*)` 会被解析为注释的开头。

也可以自己定义中缀运算符
```ocaml
let ( ^^ ) x y = max x y
(*)现在 `2 ^^ 3` 的计算结果为 `3` 。
```
### 尾递归

在递归函数的末尾, 递归调用, 但还剩余计算要进行, 那么当前的函数栈帧就不能释放, 从而可能会在过深的递归中出现栈溢出. 但对尾递归进行优化, 使用累加器变量, 就可以实现不需要返回当前栈帧, 从而不断复用当前栈帧, 避免溢出, 如:
```ocaml
// 会溢出
let rec count n =
  if n = 0 then 0 else 1 + count (n - 1)

// 不会溢出
let rec count_aux n acc =
  if n = 0 then acc else count_aux (n - 1) (acc + 1)
let count_tr n = count_aux n 0
```
## Documentation

OCaml 提供了一个名为 OCamldoc 的工具，其工作原理与 Java 的 Javadoc 工具非常相似：它从源代码中提取特殊格式的注释并将其呈现为 HTML，使程序员可以轻松阅读文档。

```ocaml
(** [sum lst] is the sum of the elements of [lst]. *)
let rec sum lst = ...
```
双星号使得该注释被识别为 OCamldoc 注释。注释部分周围的方括号表示这些部分应该在 HTML 中呈现为 `typewriter font` 而不是常规字体。
与 Javadoc 类似，OCamldoc 支持_文档标签_ ，例如 `@author` ， `@deprecated` 、 `@param` 、 `@return` 等。

建议的风格
`index` 的文档指出该函数会引发异常，如下所示 以及该异常是什么以及引发该异常的条件。（我们 将在下一章中更详细地介绍异常。） `random_int` 指定函数的参数必须满足某个条件。
在之前的课程中，你了解了 _前提_ 条件和 _后置条件_ 。前提条件是指在某段代码执行之前必须为真的条件；后置条件是指在某段代码执行之后必须为真的条件。
上述 `random_int` 文档中的“Requires”子句是一种前置条件。它表明 `random_int` 函数的客户端有责任保证 `bound` 的值的某些属性。同样，该文档的第一句话也是一种后置条件。它保证了函数返回值的某些属性。
```ocaml
(** [lowercase_ascii c] is the lowercase ASCII equivalent of
    character [c]. *)

(** [index s c] is the index of the first occurrence of
    character [c] in string [s].  Raises: [Not_found]
    if [c] does not occur in [s]. *)

(** [random_int bound] is a random integer between 0 (inclusive)
    and [bound] (exclusive).  Requires: [bound] is greater than 0
    and less than 2^30. *)
```
## Printing

OCaml 内置了一些打印函数，用于打印一些内置的原始类型： `print_char` 、 `print_string` 、 `print_int` 和 `print_float` 。此外，还有一个 `print_endline` 函数，它类似于 `print_string` ，但会输出换行符。

### Unit
```ocaml
print_endline
- : string -> unit = <fun>
print_string
- : string -> unit = <fun>
```
它们都接受一个字符串作为输入，并返回一个 `unit` 类型的值，这是我们之前从未见过的。这种类型的值只有一个，写成 `()` 也读作“unit”。所以 `unit` 类似于 `bool` ，只是 `unit` 类型的值比 `bool` 类型的值少一个。当你需要接受一个参数或返回一个值，但没有实际需要传递或返回的值时，可以使用 Unit。它相当于 Java 中的 `void` ，类似于 Python 中的 `None` 通常用于编写或使用具有副作用的代码。打印就是一个副作用的例子：它会改变世界，并且无法撤销。
### Semicolon
```ocaml
print_endline "Camels";
print_endline "are";
print_endline "bae"

let () = print_endline "Camels" in
let () = print_endline "are" in
print_endline "bae"

let _ = print_endline "Camels" in
let _ = print_endline "are" in
print_endline "bae"
```
上述方法均等价, 但一般都用第一个
### Ignore

如果 `e1` 类型不是 `unit` ，则 `e1; e2` 会发出警告，因为您正在丢弃一个可能有用的值。如果您确实希望如此，可以调用内置函数 `ignore : 'a -> unit` 将任何值转换为 `()` ：
### Printf

和其他语言类似, 用于格式化输出
```ocaml
let print_stat name num =
  Printf.printf "%s: %F\n%!" name num
```
`printf` 的另一个非常有用的变体是 `sprintf` ，它以字符串形式收集输出而不是打印它：
```ocaml
let string_of_stat name num =
  Printf.sprintf "%s: %F" name num
```
### Debugging

### Defenses against Bugs

- **The first defense against bugs is to make them impossible.**
- **The second defense against bugs is to use tools that find them.**
- **The third defense against bugs is to make them immediately visible.**
- **The fourth defense against bugs is extensive testing.**
当所有这些防御措施都失败后，程序员就被迫采取调试措施。
### How to Debug

- **将错误提炼成一个小的测试用例。** 调试是一项艰苦的工作，但测试用例越小，你就越有可能将注意力集中在隐藏错误的代码上。因此，花在提炼上的时间可以节省下来，因为你不必重新阅读大量的代码。在得到一个小的测试用例之前，不要继续调试！
- **采用科学方法。** 针对 bug 发生的原因，提出一个假设。你甚至可以像在化学实验室一样，把这个假设写在笔记本上，在脑海中理清思路，并记录你已经考虑过的假设。接下来，设计一个实验来验证或否定这个假设。运行实验并记录结果。根据你所了解的情况，重新制定你的假设。继续进行，直到你理性、科学地确定 bug 的原因。
- **修复错误。** 修复可能只是简单的拼写错误，也可能揭示了导致你进行重大修改的设计缺陷。考虑一下是否需要将修复应用到代码库的其他位置——例如，这是一个复制粘贴错误吗？如果是，你是否需要重构代码？
- **将小测试用例永久添加到你的测试套件中。** 你不会 希望这个 bug 再次潜入你的代码库。所以要跟踪这个小 将其作为单元测试的一部分。这样，任何时候你 做出未来的改变，你将自动防范同样的 错误。反复运行从以前的错误中提取的测试是 _回归测试_ 。

插入打印语句
```ocaml
let inc x = x + 1

let inc x =
  let () = print_int x in
  x + 1
```

函数跟踪
```ocaml
# let rec fib x = if x <= 1 then 1 else fib (x - 1) + fib (x - 2);;
# #trace fib;;
要停止跟踪，请使用 `#untrace` 指令。
```
## Exercises

此处不赘述, 都比较简单