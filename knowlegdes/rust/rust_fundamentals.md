# 变量

## 变量命名

变量命名时需要遵循Rust命名规范：[https://beatai.org/rust-course/practice/naming](https://beatai.org/rust-course/practice/naming)

## 变量绑定

```
let a = "hello world";
```

我们将这种定义变量的行为称为**变量绑定**，对应rust的**所有权机制**。

这是将内存对象绑定给一个变量，他的所有权自然顺承给该变量。（原主人会失去对该变量的所有权）

```
let mut x = 10；
x = 100；
```

rust变量在默认情况下是不可变的，可以通过**mut关键字**让变量变为可变。

```
let _x = 10;
```

如果一个变量被定义但不使用，需要在变量名前加一个**下划线_** ，否则rust编译器会给警告，因为这可能是一个潜在错误。

## 变量解构

```
let (a, mut b) = (true, false);
```

let支持从一个相对复杂的变量中，匹配出该变量的一部分内容。

## 解构式赋值

可以在赋值语句的左式中使用元组、切片和结构体模式了。因为还未学前置知识，先不给实例。

  

## 常量

```
//对于相同类型，使用_分割来增强可读性
const MAX_POINTS: u32 = 100_000;
```

常量名称要求字母全大写，必须用const关键字定义，必须声明类型。

最好将程序的硬编码数值都声明为常量。

## 变量遮蔽

```
let x = 10;
{
    let x = x * 10;
    println!("x:{}",x);
}
println!("x:{}",x);
```

rust允许声明同名变量，后声明的变量**在其作用域内会遮蔽掉前面声明的同名变量**。

这两个x是完全不同的新变量，因为是不可变变量，内存再分配了一次。如果是用mut修饰的可变变量，就可以直接修改内存地址上的值，不用再分配内存，性能要更好。

这样就不用绞尽脑汁的想一些变量名了🤔。

# 基本类型

## 常见基本类型

- **数值类型**：
- 有符号整数：`i8`， `i16`, `i32`, `i64`, `i128`, `isize`
- 无符号整数：`u8`， `u16`, `u32`, `u64`, `u128`, `usize`
- 浮点数：`f32`（单精度）, `f64`（双精度，默认）

**注意**： `isize` 和 `usize` 的大小取决于目标平台的指针大小（32 位平台为 4 字节，64 位平台为 8 字节）

- **字符串切片**：`&str`
- **布尔类型**：`true` 和 `false`
- **字符类型**：`char` 表示单个 Unicode 字符（存储为 4 字节），如 `'A'`, `'中'`, `'😻'`
- **单元类型**：`()`，其唯一可能的值也是 `()`

## 类型推导与标注

rust是一个静态类型语言，在编译阶段就必须知道所有变量类型。

rust编译器足够聪明，可以自动根据上下文推导变量类型，但有时推导会失效。

你可以像python一样显式声明变量的类型。

## 数值类型

### 整数类型

|   |   |   |
|---|---|---|
|长度|有符号类型|无符号类型|
|8 位|`i8`|`u8`|
|16 位|`i16`|`u16`|
|32 位|`i32`|`u32`|
|64 位|`i64`|`u64`|
|128 位|`i128`|`u128`|
|视架构而定|`isize`|`usize`|

`isize` 和 `usize` 类型取决于程序运行的计算机 CPU 类型： 若 CPU 是 32 位的，则这两个类型是 32 位的，同理，若 CPU 是 64 位，那么它们则是 64 位。

rust整形默认使用 **i32** 类型，他往往是性能最好的。

**整型字面量**可以用下表的形式书写：

|   |   |
|---|---|
|数字字面量|示例|
|十进制|`98_222`|
|十六进制|`0xff`|
|八进制|`0o77`|
|二进制|`0b1111_0000`|
|字节 (仅限于 `u8`<br><br>)|`b'A'`|

### 整形溢出

在debug模式下，会进行溢出检查，使程序在编译阶段panic。

当在--release模式下，rust会按照补码循环溢出的规则处理。

要显式处理可能的溢出，可以使用标准库针对原始数字类型提供的这些方法：

- 使用 `wrapping_*` 方法在所有模式下都按照补码循环溢出规则处理，例如 `wrapping_add`
- 如果使用 `checked_*` 方法时发生溢出，则返回 `None` 值
- 使用 `overflowing_*` 方法返回该值和一个指示是否存在溢出的布尔值
- 使用 `saturating_*` 方法，可以限定计算后的结果不超过目标类型的最大值或低于最小值

### 浮点类型

```
let x = 3.0_f64; //f64
let y : f32 = 3.0;

// 打印x的值，并控制小数位为2位
println!("{:.2}", x);
```

带有小数点的数字，有两种基本类型：`f32` 和 `f64`， `f64` 精度更高，是默认的浮点数类型。

浮点数根据 `IEEE-754` 标准实现。`f32` 类型是单精度浮点型，`f64` 为双精度。

**浮点数陷阱**

1. 浮点数在数值上是不确定的

2. rust浮点数没有实现std::cmp::Eq特征，意味着有时你不能比较浮点数大小

为了避免上面说的两个陷阱，你需要遵守以下准则：

- 避免在浮点数上测试相等性
- 当结果在数学上可能存在未定义时，需要格外的小心

### Nan

```
let x = (-42.0_f32).sqrt();
if x.is_nan(){ //进行防御性声明
    println!("未定义数学行为");
}
```

在数学上未定义结果，会产生一个特殊的结果 Nan（not a number）

**所有跟** `**NaN**` **交互的操作，都会返回一个** `**NaN**`，而且 `NaN` 不能用来比较

### 数字计算

rust所有运算符：

[https://beatai.org/rust-course/appendix/operators#%E8%BF%90%E7%AE%97%E7%AC%A6](https://beatai.org/rust-course/appendix/operators#%E8%BF%90%E7%AE%97%E7%AC%A6)

### 位运算

|   |   |
|---|---|
|运算符|说明|
|& 位与|相同位置均为1时则为1，否则为0|
|\| 位或|相同位置只要有1时则为1，否则为0|
|^ 异或|相同位置不相同则为1，相同则为0|
|! 位非|把位中的0和1相互取反，即0置为1，1置为0|
|<< 左移|所有位向左移动指定位数，右位补0|
|>> 右移|所有位向右移动指定位数，带符号移动（正数补0，负数补1）|

对于**移位**运算，Rust 会检查它是否超出该整型的位数范围，如果超出，则会报错 overflow。

### 序列

```
//生成序列 1到4
for i in 1..5 { 
    println("{}",i};
}

//生成序列 1到5
for i in 1..=5 { 
    println("{}",i};
}

//生成序列 a到z
for i in 'a'..='z' { 
    println("{}",i};
}
```

序列只允许使用**数字或字符类型**，生成连续数字，常用在循环中

### as类型转换

```
let a: i32 = 10;
let b: u16 = 100;
if a < (b as 132) {
    println!("a is  less than b")
}
```

将范围较大的数值转换为数值较小的数值容易出错。

```
let mut values:[i32;2] = [1,2];
let p1: *mut i32 = values.as_mut_ptr();
//将指针转换为地址
let first_address = p1 as usize;
//将地址转换为指针
let second = first_address + 4;
let p2 = second_address as *mut i32;
```

转换不具有传递性，`e as U1 as U2` 是合法的，也不能说明 `e as U2` 是合法的。

### 数值类型总结

Rust 拥有相当多的数值类型.

类型转换必须是显式的.

Rust 的数值上可以使用方法

  

## 字符、布尔、单元类型

### 字符类型

```
let c = 'z';
let z = 'ℤ';
let g = '国';
let heart_eyed_cat = '😻';
```

Rust 的字符不仅仅是 `ASCII`，所有的 `Unicode` 值都可以作为 Rust 字符。

`char`类型表示一个 `Unicode` 标量值（Unicode Scalar Value），它占用固定的 4 字节（32 位） 内存（与 UTF-32 编码单元一致）

Rust 的字符只能用 `''` 来表示， `""` 是留给字符串的

### 布尔类型

```
let t = true;
let f: bool = false;
```

Rust 中的布尔类型有两个可能的值：`true` 和 `false`，布尔值占用内存的大小为 `1` 个字节

### 单元类型

单元类型就是 `()`

fn main( )就返回一个单元类型

可以作为一个值用来占位，但是完全**不占用**任何内存，内存占用为0字节

  

## String类型

```
let s = “hello”； //s是&str类型 
```

`s` 是被硬编码进程序里的字符串值，使用字符串字面值进行赋值。

- **字符串字面值是不可变的**，因为被硬编码到程序代码中
- 并非所有字符串的值都能在编写代码时得知

```
let s = String::from("hello");
s.push_str(", world!");
println!("{}",s);
```

Rust 为我们提供动态字符串类型: `String`，该类型被分配到堆上，因此可以动态伸缩。

这里只简单说明一下，因为rust的字符串类型较复杂，我门先不过多讨论。

  

## 语句与表达式

```
//10是表达式
//let a = 10；是语句
let a = 10； 

fn add_with_extra(x: i32, y: i32) -> i32 {
    let x = x + 1; // 语句
    let y = y + 5; // 语句
    x + y // 表达式，作为函数返回值
}

//if语句块是表达式
//类似三元运算符的写法
let y =  if a / 10 == 1{
    “1”;
}else{
    “2”;
}   
```

对于 Rust 语言而言，**这种基于语句（statement）和表达式（expression）的方式是非常重要的，你需要能明确的区分这两个概念**。

**表达式不能包含分号**。这一点非常重要，一旦你在表达式后加上分号，它就会变成一条语句，再也**不会**返回一个值

表达式如果不返回任何值，会隐式地返回一个 [()](https://beatai.org/rust-course/basic/base-type/char-bool#%E5%8D%95%E5%85%83%E7%B1%BB%E5%9E%8B)

## 函数

### 函数的组成

```
fn add(a: i32, b: i32) -> i32 {
    a + b //这里可以写return a + b
}
```

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1784599884653-efded943-b58a-4f6e-9ca5-7f542986b648.png)

  

### 函数的要点

命名规范 https://beatai.org/rust-course/practice/naming

- 函数名和变量名使用[蛇形命名法(snake case)](https://beatai.org/rust-course/practice/naming)，例如 `fn add_two() {}`
- 函数的位置可以随便放，Rust 不关心我们在哪里定义了函数，只要有定义即可
- 每个函数参数都需要标注类型
- 返回值一定是表达式，无返回值默认是（）

  

### 发散函数

当使用 -> ！作为函数返回类型，意味着这是一个永不返回的发散函数。

这个函数的作用往往是让程序主动崩溃。

# 所有权和借用

## 所有权

### 不同的内存管理机制

编程语言为了与计算机内存交互，产生了三种流派：

- **垃圾回收机制(GC)**，在程序运行时不断寻找不再使用的内存，典型代表：Java、Go
- **手动管理内存的分配和释放**, 在程序中，通过函数调用的方式来申请和释放内存，典型代表：C++
- **通过所有权来管理内存**，编译器在编译时会根据一系列规则进行检查，rust选择了这种方式

### 堆栈性能

栈：数据先进后出，所有进栈的数据必须知道其占用的内存大小。

堆：数据大小可以任意，通过指针并在指向的地址分配足够的空间。接着指针会被推入栈中，通过栈中的**指针**，来获取数据在堆上的实际内存位置，进而访问该数据。

栈的性能往往比堆更好，在堆上分配内存往往要进行一些函数调用（或系统调用）

堆上的数据缺乏组织性，不规范的申请和释放空间会导致内存泄漏，二次释放等严重bug，

rust则使用所有权机制跟踪这些数据何时分配和释放，为其提供安全保障。

### 所有权原则

**谨记以下规则**：

- Rust 中每一个值都被一个变量所拥有，该变量被称为值的所有者
- 一个值同时只能被一个变量所拥有，或者说一个值只能拥有一个所有者
- 当所有者（变量）离开作用域范围时，这个值将被丢弃(drop)

  

### 所有权移动

```
let s1 = String::from("hello world!");
let s2 = s1; //所有权交接，s1失效
```

`**s1**` **被赋予** `**s2**` **后，Rust 认为** `**s1**` **不再有效。**此时再访问s1就会报错，且当s1离开作用域时，不会drop任何东西，这避免了double free错误。

```
let s1 = "hello world!"; //s1没有对 “hello world” 的所有权
let s2 = s1;  //仅对引用进行拷贝
```

`x` 只是引用了存储在二进制可执行文件( binary )中的字符串 `"hello, world"`，并没有持有所有权。

### 深拷贝与浅拷贝

```
let s1 = String::from("hello");
let s2 = s1.clone();
```

rust永远不会自动深拷贝，他会极大影响性能。

如果代码性能无关紧要，例如初始化程序时或者在某段时间只会执行寥寥数次时，你可以使用 `clone` 来简化编程。

```
let x = 5；
let y = x；
```

浅拷贝只发生在栈上，因此性能很高

这种浅拷贝，往往发生在拥有 `Copy` 特征且存储在栈上的类型中，拷贝过程不会发生所有权的交换。

**任何基本类型的组合可以** `**Copy**` **，不需要分配内存或某种形式资源的类型是可以** `**Copy**` **的：**

- 所有整数类型，比如 `u32`
- 布尔类型，`bool`，它的值是 `true` 和 `false`
- 所有浮点数类型，比如 `f64`
- 字符类型，`char`
- 元组，当且仅当其包含的类型也都是 `Copy` 的时候。比如，`(i32, i32)` 是 `Copy` 的，但 `(i32, String)` 就不是
- 不可变引用 `&T` ，例如[转移所有权](https://beatai.org/rust-course/basic/ownership/ownership#%E8%BD%AC%E7%A7%BB%E6%89%80%E6%9C%89%E6%9D%83)中的最后一个例子，**但是注意：可变引用** `**&mut T**` **是不可以 Copy的**


### 函数传值与返回

```
fn main() {
    let s = String::from("hello");  // s 进入作用域

    takes_ownership(s);             // s 的值移动到函数里 ...
                                    // ... 所以到这里不再有效

    let x = 5;                      // x 进入作用域

    makes_copy(x);                  // x 应该移动函数里，
                                    // 但 i32 是 Copy 的，所以在后面可继续使用 x

} // 这里, x 先移出了作用域，然后是 s。但因为 s 的值已被移走，
  // 所以不会有特殊操作

fn takes_ownership(some_string: String) { // some_string 进入作用域
    println!("{}", some_string);
} // 这里，some_string 移出作用域并调用 `drop` 方法。占用的内存被释放

fn makes_copy(some_integer: i32) { // some_integer 进入作用域
    println!("{}", some_integer);
} // 这里，some_integer 移出作用域。不会有特殊操作
```

```
fn main() {
    let s1 = gives_ownership();         // gives_ownership 将返回值
                                        // 移给 s1

    let s2 = String::from("hello");     // s2 进入作用域

    let s3 = takes_and_gives_back(s2);  // s2 被移动到
                                        // takes_and_gives_back 中,
                                        // 它也将返回值移给 s3
} // 这里, s3 移出作用域并被丢弃。s2 也移出作用域，但已被移走，
  // 所以什么也不会发生。s1 移出作用域并被丢弃

fn gives_ownership() -> String {             // gives_ownership 将返回值移动给
                                             // 调用它的函数

    let some_string = String::from("hello"); // some_string 进入作用域.

    some_string                              // 返回 some_string 并移出给调用的函数
}

// takes_and_gives_back 将传入字符串并返回该值
fn takes_and_gives_back(a_string: String) -> String { // a_string 进入作用域

    a_string  // 返回 a_string 并移出给调用的函数
}
```

在进行函数调用传参时，非Copy类型的所有权会move给函数参数。

在函数返回时，同样会将返回值的所有权转移给调用函数的变量。

  

## 引用与借用

Rust 通过 `借用(Borrowing)` 简化值传递的过程。**获取变量的引用，称之为借用(borrowing)**。

### 引用与解引用

```
let x = 5;
let y = &x; 

println!("{}",*y); 
```

`y` 是 `x` 的一个引用 `&x` ，使用`*y` 来解出引用所指向的值。

### 不可变引用

```
let s1 = String::from("hello");
let len = calculate_length(s1);

fn calculate_length(s: &String){
    s.len()
}
```

将s1的引用作为参数传递给函数，只允许使用值，不发生所有权的移动。

s是一个指向s1的引用，不具有值的所有权。所以在离开作用域不会丢弃该值，且对该值进行修改。

你可一在同一引用作用域下拥有多个不可变引用。

### 可变引用

```
let mut s = String::from("hello");

fn change(some_string : &mut String){
    some_string.push_str(", world!");
}
```

首先s必须是可变变量，在声明引用时使用 &mut 类型，这样声明的引用称为可变引用。

这种限制的好处就是使 Rust 在编译期就避免数据竞争，数据竞争可由以下行为造成：

- 两个或更多的指针同时访问同一数据
- 至少有一个指针被用来写入数据
- 没有同步数据访问的机制

### 引用原则

**引用作用域**

和变量不同，引用作用域持续到最低级花括号内，最后一次使用该引用的行。

**可变引用同时在同一个作用域内只能存在一个**

```
let mut s = String::from("hello");
{
    let r1 = &mut s;
}// r1 在这里离开了作用域，所以我们完全可以创建一个新的引用
let r2 = &mut s;
```

这种限制的好处就是使 Rust 在编译期就避免数据竞争，数据竞争可由以下行为造成：

- **两个或更多的指针同时访问同一数据**
- **至少有一个指针被用来写入数据**
- **没有同步数据访问的机制**

  

**可变引用与不可变引用不能同时存在**

```
let mut s = String::from("hello");

let r1 = &s; // 没问题
let r2 = &s; // 没问题
let r3 = &mut s; // 错误

println!("{}, {}, and {}", r1, r2, r3);
```

  

### NLL

对于这种编译器优化行为，Rust 专门起了一个名字 —— **Non-Lexical Lifetimes(NLL)**，专门用于找到某个引用在作用域(`}`)结束前就不再被使用的代码位置

### 悬垂引用

```
fn dangle() -> &String { // dangle 返回一个字符串的引用

    let s = String::from("hello"); // s 是一个新字符串

    &s // 返回字符串 s 的引用
} 
// 这里 s 离开作用域并被丢弃。其内存被释放。
// 危险！
```

解决方法是直接传递值的所有权

Rust 编译器可以确保引用永远也不会变成悬垂状态

  

  

# 复合类型

## 字符串

### 字符串切片

```
let s = String::from("hello");
let len = s.len();

//部分切片
let slice1 = &s[0..2];
//完整切片
let slice2 = &s[..];
let slice3 = &s[0..len];
```

字符串切片的类型标识是 **&str** ，范围是**左开右闭区间**。

切片的索引必须落在字符之间的边界位置，也就是 **UTF-8 字符**的边界。

字符串字面量是不可变的，因为 `&str` 是一个不可变引用。

- 字符串切片是个相当危险的操作，当进行字符串切片时，确保你明确索引范围

### rust字符串

**Rust 中的字符是 Unicode 类型，因此每个字符占据 4 个字节内存空间，但是在字符串中不一样，字符串是 UTF-8 编码，也就是字符串中的字符所占的字节数是变化的(1 - 4)**

**当 Rust 用户提到字符串时，往往指的就是** `**String**` **类型和** `**&str**` **字符串切片类型，这两个类型都是 UTF-8 编码。**

str来来自语言本身，常在字符串切片中以&str出现，而String则来自于标准库。他们之间可以相互转换。

### String与&str相互转换

- 将&str转换为String，通过一些库函数即可。
- 将String转换为&str，可以通过对String取引用和字符串切片

  

### 不存允许字符串索引

字符串的底层的数据存储格式实际上是[ `u8` ]，一个字节数组。不同字符在UTF-8编码中占用字节大小是不确定的。比如英文字母占用1字节，汉字占用4字节，这导致难以使用索引来访问字符串。

还有一个原因导致了 Rust 不允许去索引字符串：因为索引操作，我们总是期望它的性能表现是 O(1)，然而对于 `String` 类型来说，无法保证这一点，因为 Rust 可能需要从 0 开始去遍历字符串来定位合法的字符。

### 操作字符串

#### 追加：push

```
let mut s = String::from("hello");
s.push(", world!");
```

在字符串末尾追加字符串
**该方法是直接操作原来的字符串**，原变量（母串）必须是可变变量

#### 插入：insert

```
let mut s = String::from("hello");
s.insert(5,',');
s.insert_str( 6，"rust");
```

使用insert()方法在指定位置插入单个字符
使用insert_str()方法在指定位置插入字符串
索引从 0 开始计数，如果越界则会发生错误
**该方法是直接操作原来的字符串**，原变量必须是可变变量

#### 替换：replace

将字符串中某个字符串替换为其他字符串，replace()方法一共有三种。

**replace():**

```
let string_replace = String::from("I like rust. Learning rust is my favorite!");
let new_string_replace = string_replace.replace("rust", "RUST");
```

接受两个参数，将字符串中第一个参数替换为第二个参数。
**该方法会返回一个新的字符串。**

**replacen():**

```
 let string_replace = "I like rust. Learning rust is my favorite!";
    let new_string_replacen = string_replace.replacen("rust", "RUST", 1);
```

接受三个参数，将字符串中第一个参数替换为第二个参数，第三个参数是替换的个数。
**该方法会返回一个新的字符串。**

**replace_range():**

```
let mut string_replace_range = String::from("I like rust!");
string_replace_range.replace_range(7..8, "R");
```

将一定范围（的一个参数）的子串替换为第二个参数。
**该方法是直接操作原来的字符串**，原变量必须是可变变量

#### 删除：delete

**pop()：**

```
let mut string_pop = String::from("rust 你好");
let p1 = string_pop.pop(); //返回 ‘好’
```

删除并返回字符串最后一个字符。
其返回值是一个 `Option` 类型，如果字符串为空，则返回 `None`。
**该方法是直接操作原来的字符串**，原变量必须是可变变量

**remove():**

```
let mut string_remove = String::from("测试remove方法");
// 删除第一个汉字 “测”
string_remove.remove(0); 
```

删除指定位置的字符

`remove()` 方法是按照**字节**来处理字符串的，如果参数所给的位置不是合法的字符边界，则会发生错误。
**该方法是直接操作原来的字符串**，原变量必须是可变变量

**truncate()：**

```
let mut string_truncate = String::from("");
string_truncate.truncate(3); 
```

删除字符串中从指定位置开始到结尾的全部字符
**方法是直接操作原来的字符串**，原变量必须是可变变量

**clear():**

```
let mut string_clear = String::from("string clear");
string_clear.clear();
```
清空字符串的所有字符
**该方法是直接操作原来的字符串**，原变量必须是可变变量


#### 连接：Concatenate

```rust
let string_append = String::from("hello ");
let string_rust = String::from("rust");
let result = string_append + &string_rust;
let mut result = result + "!";
result += "!!!"
println!("{}",result);
```
使用 + 或 += 可以连接字符串，**第一个字符串的所有权会被转移给新变量**，相当于对最左侧的字符串变量调用add方法。
右侧的字符串可以有多个，且必须是字符串切片类型。


```rust
let s1 = String::from("hello");
let s2 = String::from(", rust!");
let s = format!("{} {}!", s1, s2});
println("{}",s);
```
相当于格式化输入输出


### 字符串转义

```rust
 // 通过 \ + 字符的十六进制表示，转义输出一个字符
    let byte_escape = "I'm writing \x52\x75\x73\x74!";
    println!("What are you doing\x3F (\\x3F means ?) {}", byte_escape);

    // \u 可以输出一个 unicode 字符
    let unicode_codepoint = "\u{211D}";
    let character_name = "\"DOUBLE-STRUCK CAPITAL R\"";

    println!(
        "Unicode character {} (U+211D) is called {}",
        unicode_codepoint, character_name
    );

    // 换行了也会保持之前的字符串格式
    // 使用\忽略换行符
    let long_string = "String literals
                        can span multiple lines.
                        The linebreak and indentation here ->\
                        <- can be escaped too!";
    println!("{}", long_string);
    
    
    println!("{}", "hello \\x52\\x75\\x73\\x74");
    let raw_str = r"Escapes don't work here: \x3F \u{211D}";
    println!("{}", raw_str);

    // 如果字符串包含双引号，可以在开头和结尾加 #
    let quotes = r#"And then I said: "There is no escape!""#;
    println!("{}", quotes);

    // 如果字符串中包含 # 号，可以在开头和结尾加多个 # 号，最多加255个，只需保证与字符串中连续 # 号的个数不超过开头和结尾的 # 号的个数即可
    let longer_delimiter = r###"A string with "# in it. And even "##!"###;
    println!("{}", longer_delimiter);

```
和其他语言一样使用\进行转义，使用\ \保持字符串原样。


### 操作UTF-8字符串
```rust
for c in "中国人".chars(){
	println!("{}", c);
}
```
使用chars()方法，可以通过Unicode字符的方式遍历字符串。


```rust
for b in "中国人".bytes(){
	println!("{}", b);
}
```
返回字节数组


使用标准库不能直接在UTF-8字符串中获得子串，可以尝试使用utf8_slice获取UTF-8子串。



## 元组

 ```rust
 let tup = (i32, f64, u8) = (500, 6.4, 1);
 ```
 创建一个元组，元组长度固定，其中元素顺序和类型也是固定的。

```rust
let tup1 = (500, 6.4, 1);
let (x, y, z) = tup1; //没有转移tup1的所有权

let tup2 = (500, 6.4, 1);
let x = tup2.0;
let y = tup2.1;
let z = tup2.2;
```
可以使用模式匹配来或 . 来获取元组的值
元组索引从0开始

元组可以用来作为函数返回值，再用模式匹配获取多个值。


## 结构体
### 结构体语法
```rust
struct User{
	active : bool,
	username : String,
	email: String,
	sign_in_count: u64,
}
```
一个结构体由几部分组成：
- 通过关键字 `struct` 定义
- 一个清晰明确的结构体 `名称`
- 几个有名字的结构体 `字段`

```rust
let mut user1 = User{
	email: String::from("someone@example.com"),
	username: String::from("someome_username"),
	active: true,
	sign_in_count: 1,
}

user1.username = String::from("xxx");
```
必须要将结构体实例声明为可变的，才能修改其中的字段
- 初始化实例时，**每个字段**都需要进行初始化
- 初始化时的字段顺序**不需要**和结构体定义时的顺序一致
- 使用 . 访问结构体成员


```rust
fn build_user(email: String, username: String)-> User{
	User {
		email: email,
		username: username, //同名变量赋值允许简为一个“username”
		active: true,
		sign_in_count: 1,
	}
}
```
可以用编写构造函数的方式简化结构体的创建

```rust
let user2 = User{
	email: String::from("xxx@example"),
	..user1,
}
```
使用结构体user1创建一个结构体user2，但email成员单独初始化
结构体更新语法：`..` 语法表明凡是我们没有显式声明的字段，全部从 `user1` 中自动获取。需要注意的是 `..user1` 必须在结构体的尾部使用。

注意：user1的username成员的所有权被转移到user2的成员中来，而其他成员（activate， sign_in_count）具有Copy特征，只做了复制。user1的email在本次赋值中没有使用，自然没有所有权的转移。


### 结构体内存排布
![[Pasted image 20260726090246.png]]


### 元组结构体
```rust
struct Point(i32, i32, i32);
let origin = Point(0, 0, 0);
```
不关心结构体成员的名称，又想让结构体有整体名称时使用。


### 单元结构体
```rust
struct MyUnitStruct；
```
单元结构体不包含任何字段，但十分有用。
单元结构体的核心价值在于：**用零运行时成本，在类型系统中表达“概念”、“行为”或“标记”**。它是 Rust 实现零成本抽象（Zero-Cost Abstraction）的重要基石之一。

因为我们还没学一些rust语法，先不列举单元结构体使用实例，等之后再补充。


### 结构体数据所有权
结构体成员是有各自的所有权的。
结构体成员最好让其拥有自身所有权，我们在前期避免使用引用类型，如&str。
如果想让结构体成员是一个引用类型，就需要引入“生命周期”的概念，情况就会变得复杂。


### 打印结构体

```rust
#[driver(Debug)]
struct Rectangle {
	width: u32,
	height: u32,
}

fn main() {
	let rect1 = Rectangle {
		width: 30,
		height: 50,
	};
	
	println!("rect1 is {:?}", rect1);
	println!("rect1 is {:#?}", rect1); //以结构体创建的格式输出
}
```
结构体是不能直接格式化输出的，你需要为其实现 Display 特征。
或者先添加 Debug 特征，再指定输出格式。


```rust
#[derive(Debug)]
struct Rectangle {
    width: u32,
    height: u32,
}

fn main() {
    let scale = 2;
    let rect1 = Rectangle {
        width: dbg!(30 * scale),
        height: 50,
    };

    dbg!(&rect1);
}
```

```shell
$ cargo run
[src/main.rs:10] 30 * scale = 60
[src/main.rs:14] &rect1 = Rectangle {
    width: 60,
    height: 50,
}
```
使用 dbg! 宏可以打印调试信息。
它会拿走表达式的所有权，然后打印出相应的文件名、行号等 debug 信息，表达式的求值结果。
**除此之外，它最终还会把表达式值的所有权返回！**


## 枚举类型
```rust
#[driver(Debug)]
enum PokerSuit {
	Clubs,
	Spades,
	Diamonds,
	Hearts,
}

fn main() {
	//使用::访问枚举类型的某个成员
	let heart = PokerSuit::Hearts; 
	let diamond = PokerSuit::Diamonds;
	
	print_suit(heart);
	print_suit(diamond);
}

fn print_suit(card: PokerSuit) {
	println!("{:?}",card);
}
```
枚举(enumeration)允许你通过列举可能的成员来定义一个**枚举类型**。
以枚举类型作为参数的函数，可以接受枚举范围内不同类型的传值

```rust
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
    ChangeColor(i32, i32, i32),
}

fn main() {
    let m1 = Message::Quit;
    let m2 = Message::Move{x:1,y:1};
    let m3 = Message::ChangeColor(255,255,0);
}
```
枚举值的类型可以和枚举成员相关联，以此简化代码
**任何类型的数据都可以放入枚举成员中**


### Option表达空值
