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
空值的错误访问显然会引发bug，但保留空值是有必要的。rust使用Option枚举类型表达空值，并保证一个值不是 `Option<T>` 类型，你就 **可以** 安全的认定它的值不为空。

```rust
enum Option<T> {
	some(T),
	None,
}

let absent_number = Option<i32> = None;
```
`Option` 枚举包含两个成员，一个成员表示含有值：`Some(T)`, 另一个表示没有值。
some是任意类型的，而None需要指定类型，目的是告诉编译器some<T> 是什么类型。

```rust
let x: i8 = 5;
let y: Option<i8> = Some(5);

let sum = x + y;
```

```shell
error[E0277]: the trait bound `i8: std::ops::Add<std::option::Option<i8>>` is
not satisfied
 -->
  |
5 |     let sum = x + y;
  |                 ^ no implementation for `i8 + std::option::Option<i8>`
  |
```
这种情况显然会报错，i8 和 Option<i8> 不是相同的类型，不能直接相加。
在对 `Option<T>` 进行 `T` 的运算之前必须将其转换为 `T`


## 数组

```rust
let a = [1, 2, 3, 4, 5];
//声明数组类型
let b: [i32; 5] = [1, 2, 3, 4, 5];
//创建五个重复元素3
let c = [3; 5]; 
```
数组的三要素：
- 长度固定
- 元素必须有相同的类型
- 依次线性排列

rust中的基本类型array数组，长度不可变。**数组 `array` 是存储在栈上**，性能也会非常优秀。
rust中的Vector是可变长数组。**动态数组 `Vector` 是存储在堆上**，因此长度可以动态改变。


```rust
 let a = [9, 8, 7, 6, 5];

    let first = a[0]; // 获取a数组第一个元素
    let second = a[1]; // 获取第二个元素
```
下标访问数组元素，索引从0开始
越界访问会崩溃退出，这体现rust的安全特征，在大多系统编程语言中，不会进行越界检查。

```rust
let array: [String; 8] = std::array::from_fn(|_i| String::from("rust is good!"));

println!("{:#?}", array);
```
对于复杂类型的重复定义，使用对应的函数。基本类型之所以能`[3; 5]`这样写，是因为其具有`Copy`类型

```rust
let a: [i32; 5] = [1, 2, 3, 4, 5];

let slice: &[i32] = &a[1..3];

assert_eq!(slice, &[2, 3]);
```
上面的数组切片 `slice` 的类型是`&[i32]`，与之对比，数组的类型是`[i32;5]`，简单总结下切片的特点：

- 切片的长度可以与数组不同，并不是固定的，而是取决于你使用时指定的起始和结束位置
- 创建切片的代价非常小，因为切片只是针对底层数组的一个引用
- 切片类型 [T] 拥有不固定的大小，而切片引用类型 &[T] 则具有固定的大小，因为 Rust 很多时候都需要固定大小数据类型，因此 &[T] 更有用，`&str` 字符串切片也同理



# 流程控制

## if判断
```rust
let condition = 1;
let number = if condition == 1 {
        5
    } else if condition == 0 {
        6
    } else {
		7
    };
```
**`if` 语句块是表达式**，其返回值可以给变量赋值
`if 用来赋值时，其各分支的返回值类型应该相同
使用`else if`处理多重条件


## for循环
```rust
for i in 1..=5 {
	println!("{}", i);
}
```
```txt
for 元素 in 集合 {
	逻辑处理
} 
```
集合往往使用引用类型，除非它具有`Copy`特征

```rust
for item in &mut collection {
  // ...
}
```
如果想在循环中，**修改该元素**，可以使用 `mut` 关键字

|使用方法|等价使用方式|所有权|
|---|---|---|
|`for item in collection`|`for item in IntoIterator::into_iter(collection)`|转移所有权|
|`for item in &collection`|`for item in collection.iter()`|不可变借用|
|`for item in &mut collection`|`for item in collection.iter_mut()`|可变借用|


```rust
let a = [4, 3, 2, 1];
for (i, v) in a.iter().enumerate() {
	println!("第{}个元素是{}", i + 1, v);
}
```
使用迭代器的方式获取数组索引，这是推荐的数组遍历方法。
如果是下标遍历，会面临下标访问越界的风险，还需要耗费性能做越界检查。

```rust
for _ in 0..=10 {
	continue;
}
```
只使用循环，不使用变量来控制流程

由于 `for` 循环无需任何条件限制，也不需要通过索引来访问，因此是最安全也是最常用的。


## while循环

```rust
while 条件{
	//逻辑
}
```
没什么可说的，和其他语言一样

```rust
loop {
	if n > 5 {
		break n
	}
	n+=1
}
```
loop无条件循环循环需要你手动break退出
- **break 可以单独使用，也可以带一个返回值**，有些类似 `return`
- **loop 是一个表达式**，因此可以返回一个值



# 模式匹配
模式匹配：将模式与 `target` 进行匹配，即为模式匹配
## match匹配
```rust
enum Direction {
	East,
	West,
	North,
	South,
}

fn main() {
	let dire = Direction::South;
	match dire {
		Direction::East => println!("East");
		Direction::North | Direction::South => {
			println!("South or North");
		},
		_ => println!("West"),   //其他情况
	}
}
```
```rust
match target {
    模式1 => 表达式1,
    模式2 => {
        语句1;
        语句2;
        表达式2
    },
    _ => ( ),  //返回单元类型， _ 是通配符
	other => ( )    //可以用other实现相同效果
}
```
有以下几点值得注意：
- `match` 的匹配必须要穷举出所有可能，因此这里用 `_` 来代表未列出的所有可能性
- `match` 的每一个分支都必须是一个表达式，且所有分支的表达式最终返回值的类型必须相同
- **X | Y**，类似逻辑运算符 `或`，代表该分支可以匹配 `X` 也可以匹配 `Y`，只要满足一个即可
- `match` 后的target表达式返回值可以是任意类型，只要能跟后面的分支中的模式匹配起来即可

因为`match`本身也是一个表达式所以可以用match进行赋值
```rust
#[derive(Debug)]
enum UsState {
    Alabama,
    Alaska,
    // --snip--
}

enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter(UsState), // 25美分硬币
}
```
```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state); //取出了绑定的值state
            25
        },
    }
}
```
可以通过模式绑定在模式匹配的过程中取出绑定的值。


## if let 匹配
```rust
if let some(3) = v {
	println!("three");
}
```
`if let`匹配是当你只需要匹配一个条件，而忽略其他情况时使用。

## matches!宏
```rust
let foo = 'f';
assert!(matches!(foo, 'A'..='Z' | 'a'..='z'));
```
matches!宏将表达式与模式进行匹配，返回匹配的结果。

小心变量遮蔽，因为`match`和`if let`都算是新的代码块，会产生变量遮蔽。请尽量不要在代码块中起同名变量。



## 匹配Option<T>
```rust
fn plus_one(x: Option<i32>) -> Option<i32> {
	match x {
		None => None,
		Some(i) => Some(i+1),
	}
}

let five = Some(5);
let six = plus_one(five);
let none = plus_one(None);
```
对Option<T>进行模式匹配


## 模式匹配经典场景

```rust
// Vec是动态数组
let mut stack = Vec::new();

// 向数组尾部插入元素
stack.push(1);
stack.push(2);
stack.push(3);

// stack.pop从数组尾部弹出元素
while let Some(top) = stack.pop() {
    println!("{}", top);
}
```
`while let`条件匹配，只要匹配成立，就一直循环


```rust
let v = vec!['a', 'b', 'c'];

for (index, value) in v.iter().enumerate() {
    println!("{} is at index {}", value, index);
}
```
for循环的条件是一种模式匹配

```rust
fn print_coordinates(&(x, y): &(i32, i32)) {
    println!("Current location: ({}, {})", x, y);
}

fn main() {
    let point = (3, 5);
    print_coordinates(&point);
}
```
函数传值时的模式匹配

```rust
use std::str::FromStr;

fn get_count_item(s: &str) -> (u64, &str) {
    let mut it = s.split(' ');
    let (Some(count_str), Some(item)) = (it.next(), it.next()) else {
        panic!("Can't segment count item pair: '{s}'");
    };
    let Ok(count) = u64::from_str(count_str) else {
        panic!("Can't parse integer: '{count_str}'");
    };
    // error: `else` clause of `let...else` does not diverge
    // let Ok(count) = u64::from_str(count_str) else { 0 };
    (count, item)
}

fn main() {
    assert_eq!(get_count_item("3 chairs"), (3, "chairs"));
}
```
`let-else`的模式匹配，它显然扩充了匹配后变量作用域
else常用来做错误处理


## 全模式列表
常用模式匹配语法：https://beatai.org/rust-course/basic/match-pattern/all-patterns



# 方法 Method

## 定义方法
```rust
//定义数据成员
struct Rectangle {
	width: u32,
	height: u32,
}
//定义方法
impl Rectangle {
	fn area(&self) -> u32 {
		self.width * self.height
	}
}

fn main() {
	let rect1 = Rectangle { width: 30, height: 50};
	println!("this area =  {}", rect1.area());
}
```
rust和其他面向对象语言定义对象的区别是，rust对象的`数据`和`方法`是分开定义的
- 使用`struct`定义数据成员
- 使用`impl`定义成员方法

需要注意的是，`self` 依然有所有权的概念：

- `self` 表示 `Rectangle` 的所有权转移到该方法中，这种形式用的较少
- `&self` 表示该方法对 `Rectangle` 的不可变借用
- `&mut self` 表示可变借用


## 模块Mod
```rust
mod my {
	pub struct Rectangle {
		width: u32,
		pub heightL u32,
	}

	impl Rectangle {
		pub fn new(width: u32, height: u32) -> Self {
			Rectangle {width, height}
		}

		pub fn width(&self) -> u32 {
			return self.width;
		} 

		pub fn height(&self) -> u32 {
			return self.height;
		}
	}
}
```
在模块外部访问结构体，只能访问 `pub` 关键字修饰的数据和方法。
mod的引入更能体现封装特性



## 关联函数
```rust
struct Rectangle {
	width: u32,
	height: u32,
}

impl Rectangle {
	//定义关联函数
	fn new(w: u32, h: u32) -> Rectangle {
		Rectangle {width: w, height: h}
	}
}

let sq = Rectangle::new(3,3);
```
与cpp中的构造函数相似，是rust结构体的构造器，方法名为`new`
特点是不传入&self作为参数，返回结构体类型


一个结构体可以定义多个impl块，做方法分类用。


## 为枚举实现方法
```rust
#![allow(unused)]
enum  Message {
	Quit,
	Move {x: i32, y: i32},
	Write(String),
	ChangeColor(i32, i32, i32),
}

impl Message {
	fn main(&self) {
		//定义方法体
	}
}

fn main() {
	let m = Message::Write(String::from("hello"));
	m.call();
}
```
可以为枚举类型定义方法



# 泛型和特征

## 泛型Generics

### 函数中使用泛型
```rust
fn add<T: std::ops::Add<output = T>>(a: T, b: T) -> T {
	a + b
}
```
```rust
fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];

    for item in list.iter() {
        if item > largest {
            largest = item;
        }
    }

    largest
}

fn main() {
    let number_list = vec![34, 50, 25, 100, 65];

    let result = largest(&number_list);
    println!("The largest number is {}", result);

    let char_list = vec!['y', 'm', 'a', 'q'];

    let result = largest(&char_list);
    println!("The largest char is {}", result);
```
泛型`T`可以是任意类型，但不是所有类型都适用函数中的逻辑，需要对类型加以限制。
使用`Trait`特征 对T进行限制 （上例：std::cmp::PartialOrd>）


```rust
use std::fmt::Display;

fn create_and_print<T>() where T: From<i32> + Display {
	let a: T = 100.into();
	println!("a is {}", a);
}

fn main() {
	create_and_print::<i64>();
}
```
有时，编译器不能确定泛型的类型，需要手动指定



### 结构体使用泛型
```rust
struct Point<T, U> {
	x: T,
	y: U,
	z: U,
}

let p = Point{x: 1, y: 1.5, z: 3.14};
```
结构体使用泛型要先声明，同种泛型的类型必须相同。



### 枚举使用泛型
```rust
enum Option<T> {
    Some(T),
    None,
}

enum Result<T, E> {
	Ok(T),
	Err(E),
}
```
Option是老朋友，用来确定一个变量是否有值
Result表明返回的结果，正常返回Ok(T)，错误返回Err(T)



### 方法中使用泛型
```rust
struct Point<T> {
	x: T,
	y: T,
}

impl<T> Point<T> {
	fn get_x(&self) -> &T {
		&self.x
	}
}

let p = Point{x: 1, y: 2};
println!("x is {}", p.x());
```
这里的Point<T>是一个完整的结构体类型而非泛型声明。
这几种泛型的定义是不一样的，互不冲突


```rust
impl Point<f32> {
	fn distance(&self) -> f32 {
		(self.x.powi(2) + self.y.powi(2)).sqrt()
	}
}
```
可以为某一具体的泛型实现方法



### const泛型

```rust
fn dispaly_array<T: std::fmt::Debug, const N: usize>(arr: [T; N]) {
	println!("{:?}", arr);
}

fn main() {
	let arr: [i32; 3] = [1, 2, 3];
	display_arrray(arr);
}
```
const泛型是针对值的泛型，便于rust操作数组长度。


```rust
fn something<T>(val: T)
where
    Assert<{ core::mem::size_of::<T>() < 768 }>: IsTrue,
    //       ^-----------------------------^ 这里是一个 const 表达式，换成其它的 const 表达式也可以
{
    //
}

fn main() {
    something([0u8; 0]); // ok
    something([0u8; 512]); // ok
    something([0u8; 1024]); // 编译错误，数组长度是1024字节，超过了768字节的参数长度限制
}
```
伪代码，使用const泛型表达式可以限制函数参数占用内存大小。




### const fn常量函数

```rust
const fn add(a: usize, b: usize) -> usize {
    a + b
}

const RESULT: usize = add(5, 10);

fn main() {
    println!("The result is: {}", RESULT);
}
```
`const fn常量函数`在编译期就会被执行，并将结果内嵌到代码中，减少运行时性能消耗。
- const fn 是有一定限制的，不可将随机数生成器写成 `const fn`
- 不建议使 `数组长度 (arr.len())` 和 `Enum判别式` 依赖于浮点计算，const fn 可能得到不同结果

```rust
struct Buffer<const N: usize> {
    data: [u8; N],
}

const fn compute_buffer_size(factor: usize) -> usize {
    factor * 1024
}

fn main() {
    const SIZE: usize = compute_buffer_size(4);
    let buffer = Buffer::<SIZE> {
        data: [0; SIZE],
    };
    println!("Buffer size: {} bytes", buffer.data.len());
}
```
将 `const fn` 与 `const 泛型` 结合，可以实现更加灵活和高效的代码设计。
例如，创建一个固定大小的缓冲区结构，其中缓冲区大小由编译期计算确定。


### 泛型开销
`泛型`是零成本抽象，没有运行时开销。
相对的，rust损失了编译速度并增大了最终生成文件的大小。



## Trait特征

### 特征的定义与实现
```rust
pub trait Summary {
	//只进行方法声明
	fn summrize(&self) -> String;
}

pub struct Post {
	pub title: String,
	pub author: String.
	pub content: String;
}

//为 `Post` 类型实现 `Summary` 特征
impl Summary for Post {
	fn summarize(&self) -> String {
		format!("文章{}，作者是{}", self.title, self.author);
	}
}
```
特征定义了**一组可以被共享的行为，只要实现了特征，你就能使用这组行为**。
特征与其实现是分开的，特征内仅对行为进行声明，在外对特定类型实现该行为。
（有种实现多态的感觉）

### 孤儿规则
**如果你想要为类型** `A` **实现特征** `T`**，那么** `A` **或者** `T` **至少有一个是在当前作用域中定义的！**
确保不同人写的代码不会相互破坏


### 默认实现
```rust
pub trait Summary {
	fn summarize(&self) -> String {
		String::from("hello world!");
	}
}
```
如果你不对类型重写该行为，就会执行默认实现
默认实现允许**调用相同特征中的其他方法**，哪怕这些方法没有默认实现。


### 特征作为函数参数
```rust
pub fn notify(item: &impl Summary) {
	println!("The great new`s summary is {}", item.summarize());
}
```
特征作为函数参数是很强大的功能
所有实现了Summary特征的类型（可以是不同类型）都可以作为该函数的参数，在函数内可以调用Summary特征的方法。



### 特征约束
```rust
pub fn notify<T: Summary>(item1: &T, item2, &T) {
}
```
泛型类型 `T` 说明了 `item1` 和 `item2` 必须拥有同样的类型，同时 `T: Summary` 说明了 `T` 必须实现 `Summary` 特征。

```rust
pub fn notify<T: Summary + Display>(item: &T) {}
```
多重约束，要求泛型T即实现了Summary特征还实现了Dispaly特征。


```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
	where T: Display + Clone,
		U: Clone + Debug
{}
```
当特征约束很多时，使用where进行一些形式上的改进


### 利用特征约束有条件地实现方法或特征
```rust
use std::fmt::Display;

struct Pair<T> {
	x: T,
	y: T,
}

impl<T> Pair<T> {
	fn new(x: T, y: T) -> self {
		self {
			x,
			y,
		}
	}
}

impl<T: Display + PartialOrd> Pair<T> {
	fn cmp_display(&self) {
		if self.x >= self.y {
			println!("The largest number is x = {}", self.x);
		}else {
			println!("The largest number is y = {}", self.y);		
		}
	}
}
```
对所有Pair<T>类型都定义new方法
对具有 Display + PartialOrd 特征的pair<T>类型定义cmp_display方法


```rust
impl<T: Display> ToString for T {

}
```
对具有Display特征的类型实现特征



### 函数返回Impl Trait
```rust
fn reurn_summarizable() -> impl Summary {
	Weibo {
		username: String::from("sunface"),
		content: String::from("xxx"),
	}
}
```
返回一个具有Summary特征的一个对象。
这样做的好处是，如果返回的对象类型很复杂，你不需要写出他的所有类型。
比如返回迭代器，所有迭代器都实现了`Iterator`特征。

缺点是只能返回一个具体类型。



### 通过driver派生特征

使用如 `#[derive(Debug)]`进行标记，被标记的类型自动实现对应特征，继承相应功能。
`driver`派生的特征是 Rust 默认给我们提供的特征，可以自己手动重写该实现。



### 调用方法时引入特征
```rust
use std::convert::TryInto;

fn main() {
  let a: i32 = 10;
  let b: u16 = 100;

  let b_ = b.try_into()
            .unwrap();

  if a < b_ {
    println!("Ten is less than one hundred.");
  }
}
```
在代码中调用了try_into()方法，他来自std::convert::TryInto特征，**那么你需要将该特征引入当前的作用域中**
当然你可以把最常用的标准库中的特征通过 `std::prelude` 模块提前引入到当前作用域中



## 特征对象

### 定义特征对象
```rust
trait Draw {
	fn draw(&self) -> String;
}

impl Draw for u8 {
	fn draw(&self) -> String {
		format!("u8: {}", *self)
	}
}

fn draw1(x: Box<dyn Draw>) {
	x.draw();
}

fn draw2(x: &dyn Draw) {
	x.draw();
}

fn main() {
	let y = 8u8;
	draw1(Box::new(x));
	draw2(&x);
}
```
使用`dyn`关键字声明一个特征对象。
可以通过 `&` 引用或者 `Box<T>` 智能指针的方式来创建特征对象。
- `draw1` 函数的参数是 `Box<dyn Draw>` 形式的特征对象，该特征对象是通过 `Box::new(x)` 的方式创建的
- `draw2` 函数的参数是 `&dyn Draw` 形式的特征对象，该特征对象是通过 `&x` 的方式创建的
- `dyn` 关键字只用在特征对象的类型声明上，在创建时无需使用 `dyn`

`dyn` 不能单独作为特征对象的定义，编译器在编译期不知道该类型的大小，rust要求所有变量，函数参数，返回值大小确定。而 `&dyn` 和 `Box<dyn>` 在编译期都是已知大小，所以可以用作特征对象的定义。



```rust
pub struct Screen {
	pub components: Vec<Box<dyn Draw>>,
}
```
定义一个`Screen`结构体，存储一个动态数组，数组中的成员类型为实现了`Draw`特征的类型


```rust
impl Screen {
	pub fn run(&self) {
		for component in self.components.iter() {
			component.draw();
		}
	}
}
```
为Screen定义run方法，遍历Vec数组


```rust
pub struct Screen<T: Draw> {
	pub components: Vec<T>,
}

impl<T> Screen<T>
	where T: Draw {
	pub fn run(&self) {
		for component in self.components.iter() {
			component.draw();
		}
	}
}
```
使用泛型实现，相较于特征对象有个缺点，数组中所有元素都必须是一个类型。

**鸭子类型(duck typing)**简单来说，就是只关心值长啥样，而不关心它实际是什么。在特征对象中，表现为我们只关心某种类型是否实现了某个特征。


### 动态分发和静态分发
编译器会为每一个泛型参数对应的具体类型生成一份代码，这种方式是**静态分发(static dispatch)**，因为是在编译期完成的，对于运行期性能完全没有任何影响。


**动态分发(dynamic dispatch)**，在这种情况下，直到运行时，才能确定需要调用什么方法。之前代码中的关键字 `dyn` 正是在强调这一“动态”的特点。

![[Pasted image 20260729110441.png]]


对动态分发的解释：
- 特征对象引用由两个指针组成，**ptr指针指向实现特征的类型的具体实例；vptr指针指向vtable虚表**，虚表中存储该类型对特征方法的实现。特征对象可以直接从虚表中找到方法并调用。
- 不同类型的虚表显然是不同的
- 特征对象大小不固定，但特征对象引用大小是固定的，很常用


### Self与self
```rust
trait Draw {
	fn draw(&self) -> Self;
}
```
self指向当前实例对象，Self指代特征或方法类型



### 特征对象安全

只有满足对象安全的特征才能拥有特征对象
满足两个条件：
- 方法的返回类型不能是 `Self`
- 方法没有任何泛型参数

这体现了特征对象的抽象性，特征对象忽略了原本对象的类型，只关心是否实现了特征。



### 关联类型
```rust
pub trait Iterator {
	type Item;
	fn next(&mut self) -> Option<Self::Item>;
}

impl Iterator for Counter {
	type Item = u32;
	fn next(&mut self) -> Option<self::Item> {
		//snip
	}
}
```
关联类型是在特征中声明的类型，在其他类型进行实现，可以在特征中使用`self::关联类型`进行调用。
他比泛型声明要来的简洁，声明一次多处调用。


### 调用同名的方法
```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}

struct Human;

impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}

impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}

impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
```

调用类型上的方法：
```rust
let person  = Human;
person.fly();
```

调用特征上的方法：
```rust
let person = Human;
Pilot::fly(&person);
Wizard::fly(&person);
```
正因为有self参数，可以明确调用类型。如果是没有self参数的关联函数呢？就要使用完全限定语法。



### 完全限定语法
```rust
trait Animal {
	fn baby_name() -> String;
}

struct Dog;

impl Dog {
	fn baby_name() -> String {
		String::from("Spot")
	}
}

impl Animal for Dog {
	fn baby_name() -> String {
		String::from("puppy")
	}
}

fn main() {
	println!("The dog`s name is {}", <Dog as Animal>::baby_name());
}
```
其定义为：
```rust
<Type as Trait>::function(receiver_if_method, next_arg, ...);
```
大多数情况，rust编译器能够自动推断出调用目标函数，只有当重名函数过多，编译器推断失败时，才会用该语法



### 特征定义中的特征约束
```rust
use std::fmt::Display;

trait OutlinePrint: Display {
    fn outline_print(&self) {
        let output = self.to_string();
        let len = output.len();
        println!("{}", "*".repeat(len + 4));
        println!("*{}*", " ".repeat(len + 2));
        println!("* {} *", output);
        println!("*{}*", " ".repeat(len + 2));
        println!("{}", "*".repeat(len + 4));
    }
}
```
在特征定义时，进行约束。上例当为某一类型实现OutlinePrint特征时，必须先为类型实现Display特征。因为OutlinePrint特征调用了Display特征的to_string()。



### newtype
为了绕过孤儿规则引入`newtype`，就是为一个**元组结构体**创建新类型。该元组结构体封装有一个字段，该字段就是希望实现特征的具体类型。
没有很明白newtype的意义，等之后再进行补充。




# 集合类型

## 动态数组 Vector

### 创建动态数组
```rust
//显示声明
let v: Vec<i32> = Vec::new();

//自动推导
let mut v = Vec::new();
v.push(1);
```
使用push可以想尾部添加一个元素
可以使用 `Vec::with_capacity(capacity)` 创建已知大小的动态数组，提升性能。


```rust
let v  = vec![1,2,3];
```
可以使用宏来创建动态数组，能够在创建同时初始化数组元素
当Vector被删除时，数组中的元素也会被删除



### 获取数组元素
```rust
let v = vec![1, 2, 3, 4, 5];

let third: &i32 = &v[2];
println!("第三个元素是 {}", third);

match v.get(2) {
    Some(third) => println!("第三个元素是 {third}"),   //这是一种推荐的格式化输出写法
    None => println!("去你的第三个元素，根本没有！"),
}
```
第一种是下标访问，&v[2]以引用的形式获取数组元素
第二种是使用 .get(pos)方法，它返回Option<T>因此需要使用match进行模式匹配
下标访问会比 .get()方法访问速度快，而 .get()会进行越界检查，返回Option<T>，更安全



### 同时借用多个数组元素

```rust
let mut v = vec![1, 2, 3, 4, 5];

let first = &v[0];  //不可变借用

v.push(6);    //可变借用

println!("The first element is: {first}");   //调用了前面的不可变借用
```
这种情况会报错，因为你在可变借用的后面调用了前面的不可变借用。对于动态数组，数组长度增大可能大于预留长度，此时rust会新开一块更大的内存，将数组数据拷贝过去。之前的引用就指向一块无效内存。


### 遍历数组
```rust
let x = vec![1,2,3];
for in &v {
	println!("{i}");
}

//可以在遍历的过程修改数组
for i in &mut v {
	*i += 10;
}
```


### 存储不同类型的元素
```rust
#[derive(Debug)]
enum IpAddr {
	V4(String),
	V6(String)
}

fn main() {
	let v = vec![
		IpAddr::V4("127.0.0.1".to_string()),
		IpAdrr::V6("::1".to_string())
	];

	for ip in v {
		show_addr(ip);
	}
}

fn show_addr(ip: IpAddr) {
	println!("{:?}", ip);
}
```
通过**枚举类型**让动态数组存储不同类型。


```rust
trait IpAddr {
    fn display(&self);
}

struct V4(String);
impl IpAddr for V4 {
    fn display(&self) {
        println!("ipv4: {:?}",self.0)
    }
}
struct V6(String);
impl IpAddr for V6 {
    fn display(&self) {
        println!("ipv6: {:?}",self.0)
    }
}

fn main() {
    let v: Vec<Box<dyn IpAddr>> = vec![
        Box::new(V4("127.0.0.1".to_string())),
        Box::new(V6("::1".to_string())),
    ];

    for ip in v {
        ip.display();
    }
}
```
通过**特征对象**来实现
特征对象的形式用的比较多，因为特征对象更加灵活。



### Vector常用方法

```rust
let v = vec![0;3];
let v_from = Vec::from([0, 0, 0]); 
```
一些初始化方法


```rust
//申请一个预估容量的动态数组
let mut v = Vec::with_capacity(10);
//附加数据
v.extern([1,2,3]);
//调整容量
v.reverse(100);
//释放剩余容量
v.shrink_to_fit();
```
对Vector容量进行操作


```rust
let mut v =  vec![1, 2];
assert!(!v.is_empty());         // 检查 v 是否为空

v.insert(2, 3);                 // 在指定索引插入数据，索引值不能大于 v 的长度， v: [1, 2, 3] 
assert_eq!(v.remove(1), 2);     // 移除指定位置的元素并返回, v: [1, 3]
assert_eq!(v.pop(), Some(3));   // 删除并返回 v 尾部的元素，v: [1]
assert_eq!(v.pop(), Some(1));   // v: []
assert_eq!(v.pop(), None);      // 记得 pop 方法返回的是 Option 枚举值
v.clear();                      // 清空 v, v: []

let mut v1 = [11, 22].to_vec(); // append 操作会导致 v1 清空数据，增加可变声明
v.append(&mut v1);              // 将 v1 中的所有元素附加到 v 中, v1: []
v.truncate(1);                  // 截断到指定长度，多余的元素被删除, v: [11]
v.retain(|x| *x > 10);          // 保留满足条件的元素，即删除不满足条件的元素

let mut v = vec![11, 22, 33, 44, 55];
// 删除指定范围的元素，同时获取被删除元素的迭代器, v: [11, 55], m: [22, 33, 44]
let mut m: Vec<_> = v.drain(1..=3).collect();    

let v2 = m.split_off(1);        // 指定索引处切分成两个 vec, m: [22], v2: [33, 44]
```
其他一些常用方法


```rust
let v = vec![11,22,33,44,55];
let slice = &v[1..=3];
```
获取数组切片



### 对Vector的排序

```rust
//整数排序
let mut vec = vec![1, 5, 10, 2, 15];    
vec.sort_unstable();    
assert_eq!(vec, vec![1, 2, 5, 10, 15]);

//浮点数排序
let mut vec = vec![1.0, 5.6, 10.3, 2.0, 15f32];    
vec.sort_unstable_by(|a, b| a.partial_cmp(b).unwrap());    
assert_eq!(vec, vec![1.0, 2.0, 5.6, 10.3, 15f32]);

```
稳定的排序 `sort` 和 `sort_by`，以及非稳定排序 `sort_unstable` 和 `sort_unstable_by`。
区别是稳定排序对相等元素不会重新排序，非稳定排序不保证这一点。
`非稳定` 排序的算法的速度会优于 `稳定` 排序算法，同时，`稳定` 排序还会额外分配原数组一半的空间。

浮点数因为存在`NAN`值，不能实现权数值`Ord`特性。如果你能保证数组没有NAN，可以使用`partial_cmp`自定义比较排序。


```rust
#[derive(Debug)]
struct Person {
    name: String,
    age: u32,
}

impl Person {
    fn new(name: String, age: u32) -> Person {
        Person { name, age }
    }
}

fn main() {
    let mut people = vec![
        Person::new("Zoe".to_string(), 25),
        Person::new("Al".to_string(), 60),
        Person::new("John".to_string(), 1),
    ];
    // 定义一个按照年龄倒序排序的对比函数
    people.sort_unstable_by(|a, b| b.age.cmp(&a.age));

    println!("{:?}", people);
}
```
对结构体以年龄降序排序
实现 `Ord` 需要我们实现 `Ord`、`Eq`、`PartialEq`、`PartialOrd` 这些属性。需要确保你的结构体中所有的属性均实现了 `Ord` 相关特性，否则会发生编译错误。他会根据属性顺序进行依次比较。



## HashMap
### 创建HashMap
```rust
use std::collections::HashMap;

let mut my_gems = HashMap::new();
my_gems.insert("红宝石", 1);
my_gems.insert("蓝宝石", 2);
my_gems.insert("石头", 3);
```
HashMap就是其他语言的字典，存储`KV键值对`，可以使用`new`来创建
需要先引入HashMap，因为 `HashMap` 并没有包含在 Rust 的 `prelude` 中
可以使用HashMap::with_capacity(capacity)来创建预知大小的hashMap，提升性能


```rust
fn main() {
	use std::collections::HashMap;
	let teams_list = vec![
		("china".to_string(), 100),
		("US".to_string(), 200),
		("jp".to_string(), 10)
	];
	
	let teams_map: HashMap<_,_> = teams_list.into_iter().collect();
	println!("{:?}", teams_map);
}
```
使用迭代器+collect的方式创建`HashMap`，into_iter方法把列表转化为迭代器，接着通过collect进行收集。
因为`collect`方法在内部实际上支持生成多种类型的目标集合，因此我们需要通过类型标注 `HashMap<_,_>` 来告诉编译器匹配的具体类型。


### 所有权转移
```rust
use std::collections:HashMap;

fn main() {
	let name = String::new("name");
	let age = 18;
	let mut boys = HashMap::new();
	//name的所有权被转移了
	boys.insert(name, age);
	//再调用就会报错
	println!("name is {}", name); 
}
```
`HashMap` 的所有权规则与其它 Rust 类型没有区别：

- 若类型实现 `Copy` 特征，该类型会被复制进 `HashMap`，因此无所谓所有权
- 若没实现 `Copy` 特征，所有权将被转移给 `HashMap` 中


```rust
	let name = String::new("name");
	let age = 18;
	let mut boys = HashMap::new();
	//将引用类型传递给HashMap
	boys.insert(&name, age);
	println!("name is {}", name); 
```
传递引用类型时，请确保该引用的生命周期至少和HashMap一样长。



### 查询HashMap
```rust
use std::collections::HashMap;

let mut scores = HashMap::new();

scores.insert(String::from("Blue"), 10);
scores.insert(String::from("Yellow"), 50);

let team_name = String::from("Blue");
let  score: Option<&i32> = scores.get(&team_name);
```

```rust
let score: i32 = scores.get(&team_name).copied().unwrap_or(0);
```
上面有几点需要注意：

- `get` 方法返回一个 `Option<&i32>` 类型：当查询不到时，会返回一个 `None`，查询到时返回 `Some(&i32)`
- `&i32` 是对 `HashMap` 中值的借用，如果不使用借用，可能会发生所有权的转移
- `get` 方法的 `key` 参数必须是一个引用



### 更新HashMap的值
```rust
fn main() {
	use::std::collections::HashMap;
	
	let mut scores = HashMap::new();
	scores.insert("Blue", 10);
	//覆盖已有的值
	let old = socres.insert("Blue", 20);
	//查询是否有值，没有就插入
	let v = scores.entry("Yellow").or_insert(50);
}
```
一些更新值的情况



```rust
use::std::collections::HashMap();
let text = “hello world  hello great world”；
let mut map = HashMap::new();
//根据空格分割字符串
for word in text.split_whitespace() {
	let count = map.entry(word).or_insert(0);
	*count += 1;
}
```
一种词频统计的例子
有两点值得注意：

- `or_insert` 返回了 `&mut v` 引用(值的引用)，因此可以通过该可变引用直接修改 `map` 中对应的值
- 使用 `count` 引用时，需要先进行解引用 `*count`，否则会出现类型不匹配



### 哈希函数
若性能测试显示当前标准库默认的哈希函数不能满足你的性能需求，就需要去 [`crates.io`](https://crates.io) 上寻找其它的哈希函数实现，使用方法很简单：

```rust
use std::hash::BuildHasherDefault;
use std::collections::HashMap;
// 引入第三方的哈希函数
use twox_hash::XxHash64;

// 指定HashMap使用第三方的哈希函数XxHash64
let mut hash: HashMap<_, _, BuildHasherDefault<XxHash64>> = Default::default();
hash.insert(42, "the answer");
assert_eq!(hash.get(&42), Some(&"the answer"));
```

目前`HashMap`使用的哈希函数为`SipHash`，它的性能不是很高，但是安全性很高。`SipHash` 在中等大小的 `Key` 上，性能相当不错，但是对于小型的 `Key` （例如整数）或者大型 `Key` （例如字符串）来说，性能还是不够好。若你需要极致性能，例如实现算法，可以考虑这个库：ahash




# 生命周期

## 借用检查
```rust
{
    let r;

    {
        let x = 5;
        r = &x;
    }

    println!("r: {}", r);
}
```
x的生命周期比r小，r却引用了x，这个代码会使r变为悬垂指针，而出现错误。
rust会使用`借用检查器`，比较生命周期的长度，来保证借用正确。
被引用的量生命周期应该更长



## 生命周期标注语法
```rust
fn longest<`a>(x: &`a str, y: &`a str) -> &`a str {
	if x.len() > y.len() {
		x
	}else{
		y
	}
}
```
该函数返回字符串切片中较长的那个，因为函数可能返回 x 也可能返回 y，编译器无法推断出返回的生命周期长度，就需要我们显示的进行生命周期标注。
- 和泛型一样，使用生命周期参数，需要先声明 `<'a>`
- `x`、`y` 和返回值**至少**活得和 `'a` 一样久（因为返回值要么是 `x`，要么是 `y`）
- 标注不会改变实际生命周期，我们只是想让编译通过


**函数的返回值如果是一个引用类型**，那么它的生命周期只会来源于：
- 函数参数的生命周期
- 函数体中某个新建引用的生命周期
- 不用对未参与的参数标注生命周期

如果是返回函数体中某个新建引用的生命周期，会出现垂悬引用，解决方法是返回所有权

生命周期语法用来将函数的多个引用参数和返回值的作用域关联到一起，一旦关联到一起后，Rust 就拥有充分的信息来确保我们的操作是内存安全的。



## 结构体生命周期
```rust
struct ImportantExcerpt<`a> {
	part: &`a str,
}

fn main() {
	let novel = String::from("Call me Ishmael. Some years ago..." );
	let first_sentence = novel.split('.').next().expect("Could not find a '.'");
	let i = ImportantExcerpt {
		part: first_sentence,
	};
}
```
该生命周期标注说明，**结构体 `ImportantExcerpt` 所引用的字符串 `str` 生命周期需要大于等于该结构体的生命周期**。



## 生命周期消除
```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();

    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }

    &s[..]
}
```
该函数不用标注生命周期仍能通过编译，是因为**编译器明确知道**，该返回的引用来自于函数参数，使用了`生命周期消除规则`
**函数或者方法中，参数的生命周期被称为 `输入生命周期`，返回值的生命周期被称为 `输出生命周期`**



###三条消除规则
1. **每一个引用参数都会获得独自的生命周期**
2. **若只有一个输入生命周期（函数参数中只有一个引用类型），那么该生命周期会被赋给所有的输出生命周期**，也就是所有返回值的生命周期都等于该输入生命周期
3. **若存在多个输入生命周期，且其中一个是 `&self` 或 `&mut self`，则 `&self` 的生命周期被赋给所有的输出生命周期**

只要满足以上一个规则，就不用显示标注生命周期。编译器可以套用以上规则 （1->2/3规则）来自动进行生命周期标注。



## 方法中的生命周期
```rust
struct ImportantExcerpt<`a> {
	part: &`a str,
}

impl<`a> ImportantExcerpt<`a> {
	fn level(&self) -> i32 {
		3
	}
} 
```
- `impl` 中必须使用结构体的完整名称，包括 `<'a>`，因为_生命周期标注也是结构体类型的一部分_
- 方法签名中，往往不需要标注生命周期，得益于生命周期消除的第一和第三规则


```rust
impl<`a> ImportantExcerpt<`a> {
	fn fuinction<`b>(&`a self, val: &`b str) -> &`b str{
		self.part
	}
}
```
按照第三规则，返回值生命周期应该是`a和&self一样，但我们显示标注了返回值生命周期是`b。此时编译器会报错，因为不知道`a和`b谁的生命周期大。



```rust
impl<'a: 'b, 'b> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```
```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'b str
    where
        'a: 'b,
    {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```
我们可以通过加约束`a: b`来告诉编译器，a的约束更大，因为返回对&self成员的引用。


## 静态生命周期
```rust
let s: &'static str = "我没啥优点，就是活得久，嘿嘿";
```
静态生命周期活的和程序一样久，用来解决很复杂的生命周期问题。但当程序员都不清楚该引用的生命周期时，这无疑是很危险的。




# 错误处理

## panic与不可恢复错误

### panic的触发方式
```rust
fn main() {
	let ret = error_check();  
	if ret == ERROR {
		panic!("error");
	}
}
```
可以使用 `panic!` 宏自动调用
当进行一些严重错误（如数组越界访问）时会被动触发

使用`RUST_BACKTRACE=1 cargo run` 或 `$env:RUST_BACKTRACE=1 ; cargo run`，可查看函数的调用栈（最近调用的在最上层）。


### panic的两种终止方式
```rust
[profile.release]
panic = 'abort'
```
可以通过修改Cargo.toml文件来更改终止方式
默认的方式就是 `栈展开`，Rust 会回溯栈上数据和函数调用，因此也意味着更多的善后工作，好处是可以给出充分的报错信息和栈调用信息。
便于事后的问题复盘。`直接终止`，不清理数据就直接退出程序，善后工作交与操作系统来负责。


### 使用panic的时机
```rust
use std::net::IpAddr;
let home: IpAddr = "127.0.0.1".parse().unwrap().expect("Transation ERROR");
```
`unwrap()`的作用很明确，parse()会返回Result<T, E>类型的值，如果是OK(T)就正常执行，如果是Err(E)就panic，不进行任何错误处理，调用except会打印错误信息。
这适合明确知道该处一般不会报错或先对此代码打标记，之后再写错误处理代码的情况。

当某处错误是可以预期的且可以处理，不会导致整个程序受影响，此时应该运用错误处理而非`panic`
当启动时某个流程发生了错误，对后续代码的运行造成了影响，那么就应该使用 `panic`，而不是处理错误后继续运行


### panic原理
这里先不过多说明，之后再补充。



## 可恢复错误 Result
```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt");
	
	//返回类型是std::result::Result<std::fs::File, std::io::Error>
	let f = match f {
        Ok(file) => file,
        Err(error) => match error.kind() {
	        ErrorKind::NotFound => match File::create("hello.txt") {
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => panic!("Problem opening the file: {:?}", other_error),
        },
	};
}
```
这是一种常规错误处理写法，返回Result<T, E>类型，对其进行模式匹配，对Err(E)这一项进行错误处理。
- 如果是文件不存在错误 `ErrorKind::NotFound`，就创建文件，这里创建文件`File::create` 也是返回 `Result`，因此继续用 `match` 对其结果进行处理：创建成功，将新的文件句柄赋值给 `f`，如果失败，则 `panic`
- 剩下的错误，一律 `panic`




## 传播错误与 ? 的使用
```rust
fn open_file() -> Result<File, Box<dyn std::error::Error>> {
    let mut f = File::open("hello.txt")?;
    Ok(f)
}
```
一般来说，更底层调用的函数会将错误传递给上层函数，**错误类型的抽象程度也会上升**。他要求上层函数返回值类型是Result<T,E>或 Option<T>。
`?` 是一个宏，和 `match` 的功能相似，判断返回类型是否出错，不出错可以继续运行，出错提前让上层函数返回，且他会**自动进行错误类型转换**。


```rust
use std::fs::File;
use std::io;
use std::io::Read;

fn read_username_from_file() -> Result<String, io::Error> {
    let mut s = String::new();

    File::open("hello.txt")?.read_to_string(&mut s)?;

    Ok(s)
}
```
`?`可以进行链式调用，进一步简化代码。



```rust
use std::fs::File;

fn main() {
    let f = File::open("hello.txt")?;
}
```
```rust
use std::error::Error;
use std::fs::File;

fn main() -> Result<(), Box<dyn Error>> {
    let f = File::open("hello.txt")?;

    Ok(())
}
```
第一段代码是一个常见错误，`?` 要求 `Result<T, E>` 形式的返回值，而 `main` 函数的返回是 `()`，因此无法满足。
但main函数可以有多种返回值，因为实现了 std::process::Termination特征，目前为止该特征还没进入稳定版 Rust 中




# 包和模块

## 包

Rust 为我们提供了强大的包管理工具：

- **项目(Package)**：可以用来构建、测试和分享包
- **工作空间(WorkSpace)**：对于大型项目，可以进一步将多个包联合在一起，组织成工作空间
- **包(Crate)**：一个由多个模块组成的树形结构，可以作为三方库进行分发，也可以生成可执行文件进行运行
- **模块(Module)**：可以一个文件多个模块，也可以一个文件一个模块，模块可以被认为是真实项目中的代码组织单元

### 包 Crate
对于 Rust 而言，**包是一个独立的可编译单元**，它编译后会生成一个可执行文件或者一个库。一个包会将相关联的功能打包在一起，使得该功能可以很方便的在多个项目中分享。
同一个包中不能有同名的类型，但是在不同包中就可以。

### 项目 Package
```console
$ cargo new my-project
     Created binary (application) `my-project` package
$ ls my-project
Cargo.toml
src
$ ls my-project/src
main.rs
```
创建一个二进制`Package`

`Package` 就是一个项目，因此它包含有独立的 `Cargo.toml` 文件，以及因为功能性被组织在一起的一个或多个包。
一个 `Package` 只能包含**一个**库(library)类型的包，但是可以包含**多个**二进制可执行类型的包。

**`src/main.rs` 是二进制包的根文件，该二进制包的包名跟所属 `Package` 相同，在这里都是 `my-project`**，所有的代码执行都从该文件中的 `fn main()` 函数开始。


```console
$ cargo new my-lib --lib
     Created library `my-lib` package
$ ls my-lib
Cargo.toml
src
$ ls my-lib/src
lib.rs
```
创建一个库类型`Package`
类型的 `Package` 只能作为三方库被其它项目引用，而不能独立运行
如果一个 `Package` 包含有 `src/lib.rs`，意味它包含有一个库类型的同名包 `my-lib`，该包的根文件是 `src/lib.rs`。

包和`Package`项目是容易混淆的。牢记 `Package` 是一个项目工程，而包只是一个编译单元，`src/main.rs` 和 `src/lib.rs` 都是编译单元，因此它们都是包。



### 典型Package结构
```rust
.
├── Cargo.toml
├── Cargo.lock
├── src
│   ├── main.rs
│   ├── lib.rs
│   └── bin
│       └── main1.rs
│       └── main2.rs
├── tests
│   └── some_integration_tests.rs
├── benches
│   └── simple_bench.rs
└── examples
    └── simple_example.rs
```
- 唯一库包：`src/lib.rs`
- 默认二进制包：`src/main.rs`，编译后生成的可执行文件与 `Package` 同名
- 其余二进制包：`src/bin/main1.rs` 和 `src/bin/main2.rs`，它们会分别生成一个文件同名的二进制可执行文件
- 集成测试文件：`tests` 目录下
- 基准性能测试 `benchmark` 文件：`benches` 目录下
- 项目示例：`examples` 目录下

这种目录结构基本上是 Rust 的标准目录结构，在 `GitHub` 的大多数项目上，你都将看到它的身影。

理解了包的概念，我们再来看看构成包的基本单元：模块。


## 模块 Module
###模块嵌套
```rust
// 餐厅前厅，用于吃饭
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}

        fn seat_at_table() {}
    }

    mod serving {
        fn take_order() {}

        fn serve_order() {}

        fn take_payment() {}
    }
}
```
```console
crate
 └── front_of_house
     ├── hosting
     │   ├── add_to_waitlist
     │   └── seat_at_table
     └── serving
         ├── take_order
         ├── serve_order
         └── take_payment
```
图一的代码创建了三个模块，图二是他的模块树。
使用模块，将功能相近的代码组织在一起，然后通过一个模块名称来说明这些代码为何被组织在一起。
模块之间是有嵌套关系的，子模块可以自由访问父模块，反之不行。

- 使用 `mod` 关键字来创建新模块，后面紧跟着模块名称
- 模块可以嵌套，这里嵌套的原因是招待客人和服务都发生在前厅，因此我们的代码模拟了真实场景
- 模块中可以定义各种 Rust 类型，例如函数、结构体、枚举、特征等
- 所有模块均定义在同一个文件中


### 引用模块
```rust
mod front_of_house {
    mod hosting {
        fn add_to_waitlist() {}
    }
}

pub fn eat_at_restaurant() {
    // 绝对路径
    crate::front_of_house::hosting::add_to_waitlist();

    // 相对路径，crate模块名引用
    front_of_house::hosting::add_to_waitlist();
}
```
有`绝对路径`和`相对路径`引用两种主要方式，各有优缺点。
其中相对引用又有`self`、`super`和 `crate` 或者模块名引用三种形式
选哪种方法主要是看模块树结构和后续项目结构变更情况



```rust
fn serve_order() {}

// 厨房模块
mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        super::serve_order();
    }

    fn cook_order() {}
}
```
```rust
fn serve_order() {
    self::back_of_house::cook_order()
}

mod back_of_house {
    fn fix_incorrect_order() {
        cook_order();
        crate::serve_order();
    }

    pub fn cook_order() {}
}
```
补充`super`和`self`引用，了解一下



### 代码可见性

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

/*--- snip ----*/
```
使用 `pub` 关键字标注代码可见性。rust默认所有模块，函数，结构体等，对外都不可见。所以要手动表明代码可见性。
