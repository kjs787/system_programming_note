# 函数式编程

## 闭包Closure

闭包是**一种匿名函数，它可以赋值给变量也可以作为参数传递给其它函数，不同于函数的是，它允许捕获调用者作用域中的值**

````rust
fn main() {
	let x = 1;
    
    //入参y, 捕获了作用域中的x的值
    //sum就是闭包函数本身，而非其返回值
    let sum = |y| x + y; 
    assert_eq!(3, sum(2));
}
````

闭包的形式定义：

```rust
|param1, param2,...| {
    语句1;
    语句2;
    返回表达式
}
//如果只有一个返回表达式：
|param1| 返回表达式
```

闭包的类型标注：

如果不对闭包显式标注类型，编译器就会自动推导。**当编译器推导出一种类型后，它就会一直使用该类型**。

```rust
fn  add_one_v1   (x: u32) -> u32 { x + 1 }
//以下的闭包和上面的函数实现功能是相同的
let add_one_v2 = |x: u32| -> u32 { x + 1 };
let add_one_v3 = |x|             { x + 1 };
let add_one_v4 = |x|               x + 1  ;
```



### 结构体中的闭包

```rust
struct Cacher<T, E> 
where 
	T : fn(E) -> E,	//这是一种对闭包类型的约束
{
	query: T,
    value: Option<E>,
}

impl<T,E> Cacher<T,E>
where
	T: fn(E) -> E,
{
	fn new(query: T) -> Cacher<T> {
		Cacher{
            query,
            value: None,
        }
    }
    
    fn value(&mut self, arg: E) -> E {
		match self.value {
		   Some(v) => v,
            None => {
            	let v = (self.query)(arg);
                 self.value = Some(v);
                 v
            }
        }
    }
}
```

有关闭包的知识比较繁杂，等用到后再进行学习。=



## 迭代器Iterator

迭代器允许我们迭代一个连续的集合，例如数组、动态数组 `Vec`、`HashMap` 等，在此过程中，只需关心集合中的元素如何处理，而无需关心如何开始、如何结束、按照什么样的索引去访问等问题。

在Rust中，**迭代器初始化是惰性的**，创建迭代器不会有任何性能损耗，其中的元素也不会被消耗。

我们使用迭代器遍历数组：

```rust
fn main() {
	let arr = [1,2,3];
    //arr不是迭代器，但它实现了IntoIterator特征，rust自动在for循环中将其转换为迭代器
    for v in arr {
        println!("{}",v);
    }
}
```



### Iterator特征

`Iterator` 就是迭代器特征，只有实现了它才能称为迭代器，才能调用 `next`。

`next`方法可以**消耗性的**获取迭代器中的元素，返回的是 `Option` 类型。

```rust
pub trait Iterator {
    type Item;

    fn next(&mut self) -> Option<Self::Item>;

    // 省略其余有默认实现的方法
}
```



**手动遍历迭代器实现for循环**：

手动迭代必须将迭代器声明为 `mut` 可变，因为调用 `next` 会改变迭代器其中的状态数据

```rust
let values = vec![1,2,3];
{
	let result = match Iterator::into_iter(values) {
		mut iter => loop{
            match iter{
		   	   Some(x) => { x }, 
                None => break,
            }
        }
    };
    result
}
```



### IntoIterator特征

如果一个类型实现了`IntoIterator`特征，它就可以通过调用`into_iter()`将其转换为迭代器。好在迭代器自身也实现了该特征，调用`into_iter()`它会返回它本身。

**`into_iter, iter, iter_mut`方法的区别**：

- `into_iter` 会夺走所有权
- `iter` 是借用
- `iter_mut` 是可变借用

对一个集合类型调用上述方法，返回的是**单个**实现了 `Iterator` 特征的迭代器对象，而非一个迭代器的集合。链是调用时，每个适配器返回一个新的迭代器，包裹住前一个，形成嵌套链。

示例给的很清楚：

```rust
fn main() {
    let values = vec![1, 2, 3];

    for v in values.into_iter() {
        println!("{}", v)
    }

    // 下面的代码将报错，因为 values 的所有权在上面 `for` 循环中已经被转移走
    // println!("{:?}",values);

    let values = vec![1, 2, 3];
    let _values_iter = values.iter();

    // 不会报错，因为 values_iter 只是借用了 values 中的元素
    // 调用 next 方法返回的类型是 Some(&T)
    println!("{:?}", values);

    let mut values = vec![1, 2, 3];
    // 对 values 中的元素进行可变借用
    let mut values_iter_mut = values.iter_mut();

    // 取出第一个元素，并修改为0
    // 调用 next 方法返回的类型是 Some(&mut T)
    if let Some(v) = values_iter_mut.next() {
        *v = 0;
    }

    // 输出[0, 2, 3]
    println!("{:?}", values);
```



### 消费者与适配器

 消费者是迭代器上的方法，它会消费掉迭代器中的元素，然后返回其类型的值。消费者都依赖`next()`实现，这也是为什么迭代器要实现 `Iterator` 特征，而该特征必须要实现 `next` 方法的原因。

**消费者适配器**：

只要迭代器上的某个方法 `A` 在其内部调用了 `next` 方法，那么 `A` 就被称为**消费者适配器**，所以方法 `A` 的调用也会消耗掉迭代器上的元素。但**next本身不是消费者适配器**。

**迭代器适配器**：

迭代器适配器会返回一个新的迭代器。迭代器适配器是惰性的，意味着你**需要一个消费者适配器来收尾，最终将迭代器转换成一个具体的值**：

输入迭代器输出迭代器，这是链式调用的关键。

```rust
let v1: Vec<i32> = vec![1,2,3];
//闭包可以作为适配器参数
//collect是消费者适配器
let v2: Vec<_> = v1.iter().map(|x| x + 1).collect();
assert_eq!(v2, vec![2,3,4]);
```

**collect**：

该方法就是一个消费者适配器，使用它可以将一个迭代器中的元素收集到指定类型中。`collect`在消费时必须**显式指定类型**。

```rust
use std::collections::HashMap;
fn main() {
    let names = ["sunface", "sunfei"];
    let ages = [18, 18];
    let folks: HashMap<_, _> = names.into_iter().zip(ages.into_iter()).collect();

    println!("{:?}",folks);
}
```



### 实现Iterator特征

为了实现Iterator特征，我们需要手动实现`next`方法，其他方法都有默认实现。**只要实现了Iterator特征，该类型变量就是一个迭代器**。

```rust
struct Counter {
	counter: u32,
}

impl Iterator for Counter {
	type Item = u32;
    fn next(&mut self) -> Option<Self::Item> {
		if self.count < 5 {
			self.count += 1;
        	Some(self.count)
        }else { None }
    }
}
```



### 惰性迭代器链式处理示例

迭代器是 Rust 的 **零成本抽象**（zero-cost abstractions）之一，意味着抽象并不会引入运行时开销，接下来的代码在执行过程中没有申请额外的内存空间：

```rust
let v = vec![1u64, 2, 3, 4, 5, 6];
let val = v.iter()	//生成不可变借用的迭代器
    .enumerate() //给流中的每个元素绑定一个从 0 开始的索引
    .filter(|&(idx, _)| idx % 2 == 0) //筛选器。保留索引为偶数的元素。
    .map(|(_, val)| val) //转换器。把元组“剥开”，只留下数值本身。
    .fold(0u64, |sum, acm| sum + acm); //消费者，累加求和

println!("{}", val);
```

# 智能指针

## `Box<T>`堆对象分配

`Box<T>` 是 Rust 中最常见的智能指针，`Box<T>` 允许你将一个值分配到堆上，然后在栈上保留一个智能指针指向堆上的数据。

`Box` 背后是调用 `jemalloc` 来做内存管理，所以堆上的空间无需我们的手动管理。



### rust堆栈性能

在 Rust 中，`main` 线程的栈大小是 `8MB`，普通线程是 `2MB`，在函数调用时会在其中创建一个临时栈空间，调用结束后 Rust 会让这个栈空间里的对象自动进入 `Drop` 流程，最后栈顶指针自动移动到上一个调用栈顶，无需程序员手动干预，因而栈内存申请和释放是非常高效的。

对于堆栈性能的比较：

- 小型数据，在栈上的分配性能和读取性能都要比堆上高
- 中型数据，栈上分配性能高，但是读取性能和堆上并无区别，因为无法利用寄存器或 CPU 高速缓存，最终还是要经过一次内存寻址
- 大型数据，只建议在堆上分配和使用



### 使用`Box<T>`

以下是该智能指针常见使用场景：

1. 使用`Box<T>`将数据存储在堆上

   ```rust
   fn main() {
   	let a = Box::new(3);
       println!("a = {}", a);	//Box<T>实现了Deref特征
   } 
   //Box<T>实现了Drop特征
   //a持有的智能指针会在作用域结束自动释放。
   ```

2. 避免栈上的数据拷贝

   ```rust
   fn main() {
       //在堆上申请了长度为1000的数组
       let arr = Box::new([0;1000]);
       
       //arr将堆上数据的所有权转交给arr1
       //只拷贝了一份栈上的智能指针
       let arr1 = arr;
       println!("{:?}", arr1.len());
   }
   ```

3. 将动态类型大小转化为Sized固定大小类型

   rust不知道List的大小，认为其是一个**动态大小类型 DST**，会报错。

   我们用`Box<T>`指向它，将DST转化为Sized类型

   ```rust
   enum List {
   	Cons(i32, Box<List>),  
       Nil,
   }
   ```

4. 特征对象

   将不同类型的特征对象放入同一个数组中，因为特征是DST类型

   ```rust
   trait Draw {
   	fn draw(&self);
   }
   
   struct Button {
       id: u32,
   }
   
   impl Draw for Button {
   	fn draw(&self) {
   		//...
       }
   }
   
   struct Select {
       id: u32,
   }
   
   impl Draw for Select{
   	fn draw(&self) {
   		//...
       }
   }
   
   fn main() {
   	let elems: Vec<Box<dyn Draw>> = vec![Box::new(Botton{id:1}), Box::new(Select {id: 2})];
       
       for e in elems {
   		e.draw();
       }
   }
   ```

   

### Box::leak

`Box::leak`可以强制消费掉`Box`并让内存泄漏，可以利用这个特性，让一个运行中申请的值生命周期变为`'static`。

**如果你需要一个在运行期初始化的值，但是可以全局有效**，那么就可以使用 `Box::leak`

```rust
fn main() {
   let s = gen_static_str();
   println!("{}", s);
}

fn gen_static_str() -> &'static str{
    let mut s = String::new();
    s.push_str("hello, world");

    Box::leak(s.into_boxed_str())
}
```



## Deref解引用

智能指针实现了`Dref`特征，让其可以像指针一样解引用出里面的值。且rust为其做了特殊处理，使其自动匹配需要的类型，不用多次解引用。



### 智能指针的解引用

我们尝试定义一个自己的`Box<T>`，并为其实现`Dref`特征：

```rust
struct MyBox<T>(T);

impl<T> MyBox<T> {
	fn main(x: T) -> MyBox<T> {
		MyBox(x);
    }
}


use std::ops::Deref;
//实现Dref特征
impl<T> Deref for MyBox<T> {
	type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```



当我们对智能指针进行解引用时，Rust首先调用 `deref` 方法返回值的常规引用，然后通过 `*` 对常规引用进行解引用，最终获取到目标值。这样不会发生所有权的转移。

```rust
*(y.deref())	//这种替换只会发生一次
```



### 连续的隐式转换

`Deref` 可以支持连续的隐式转换，直到找到适合的形式为止，且这种行为在编译期完成的，完全没有性能损耗。以降低可读性和编译性能为代价，换来了代码的简洁和强大。

```rust
fn main() {
    let s = MyBox::new(String::from("hello, world"));
    //方法调用会自动解引用，Deref会进行连续的隐式转换
    //String -> &String -> &str
    let s2: String = s.to_string();	
}
```



### 引用归一化

Rust 会在解引用时自动把智能指针和 `&&&&v` 做引用归一化操作，转换成 `&v` 形式，最终再对 `&v` 进行解引用，源码说的很明白：

```rust
impl<T: ?Sized> Deref for &T {
    type Target = T;

    fn deref(&self) -> &T {
        *self
    }
}
```



### 三种 Deref 转换

- 当 `T: Deref<Target=U>`，可以将 `&T` 转换成 `&U`，也就是我们之前看到的例子
- 当 `T: DerefMut<Target=U>`，可以将 `&mut T` 转换成 `&mut U`
- 当 `T: Deref<Target=U>`，可以将 `&mut T` 转换成 `&U`



## Drop释放资源

类似于c++的析构函数，他会在变量离开作用域时自动插入代码，释放资源。

### 实现Drop特征

如果为某一个结构体实现`Drop`特征，会在释放资源之前先执行一遍`drop`代码块中的内容。

```rust
struct Foo;

impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!");
    }
}
```



### 手动回收资源

当我们想提前释放锁、文件描述符或管道时，就需要手动释放资源。

调用`drop()`函数，他会在释放资源的同时取走所有权。

```rust
fn main() {
    let foo = Foo;
    drop(foo);
    // 以下代码会报错：借用了所有权被转移的值
    // println!("Running!:{:?}", foo);
}
```



### 互斥的 Copy 和 Drop

我们无法为一个类型同时实现 `Copy` 和 `Drop` 特征。因为实现了 `Copy` 特征的类型会被编译器隐式的复制，因此非常难以预测析构函数执行的时间和频率。因此这些实现了 `Copy` 的类型无法拥有析构函数。



## Rc 与 Arc

当一个值需要被多个对象使用时，rust的所有权机制就会很棘手。通过引用计数的方式，rust允许一个数据资源在同一时刻拥有多个所有者。这种实现机制就是 `Rc` 和 `Arc`，**前者适用于单线程，后者适用于多线程**。



### 引用计数`Rc<T>`

`Rc` 正是**引用计数(reference counting)**的英文缩写。当我们**希望在堆上分配一个对象供程序的多个部分使用且无法确定哪个部分最后一个结束时，就可以使用 `Rc` 成为数据值的所有者**，

智能指针 `Rc<T>` 在创建和克隆时，会将引用计数加 1；当智能指针被释放锁，引用计数减 1；引用计数归零时，就代表该数据不再被使用，因此可以被清理释放。

`Rc<T>` 是指向底层数据的不可变的引用，因此你无法通过它来修改数据，需要配合后面章节的内部可变性 `RefCell` 或互斥锁 `Mutex`。

```rust
use std::rc::Rc;
fn main() {
    let a = Rc::new(String::from("hello, world"));
    let b = Rc::clone(&a);	//clone是浅拷贝，仅拷贝栈上的智能指针

    assert_eq!(2, Rc::strong_count(&a));	//strong_count会返回当前引用计数
    assert_eq!(Rc::strong_count(&a), Rc::strong_count(&b))
}
```



### Arc

`Arc` 是 `Atomic Rc` 的缩写，顾名思义：原子化的 `Rc<T>` 智能指针。`Arc`实现了`Send`和`Sync`特征，保证数据在多线程环境下安全的传播。它的用法和`Rc`相同，特点是用性能损耗换来线程安全。

```rust
use std::sync::Arc;
use std::thread;

fn main() {
    let s = Arc::new(String::from("多线程漫游者"));
    for _ in 0..10 {
        let s = Arc::clone(&s);
        let handle = thread::spawn(move || {
           println!("{}", s)
        });
    }
}
```



## Cell 和 RefCell

 Rust 提供了 `Cell` 和 `RefCell` 用于内部可变性，简而言之，可以在拥有不可变引用的同时修改目标数据。

### Cell

`Cell<T>` 适用于 `T` 实现 `Copy` 的情况：

- `c.get` 用来取值，`c.set` 用来设置新值。

```rust
use std::cell::Cell;
fn main() {
    let c = Cell::new("kskbl");
    let one = c.get();
    c.set("zdjd");
    let two = c.get();
    println!("{},{}", one, two);
}
```



###  RefCell

和Cell类似但不限定Copy特征，且能正常通过编译，但会在运行时panic。

**`RefCell`用在你确信代码是正确的，而编译器发生误判的情况**。这通常是个问题，但也可以借用该机制修改复杂的借用关系。

```rust
use std::cell:RefCell;

fn main() {
	let s = RefCell::new(String::form("hello world"));
    let s1 = s.borrow();
    let s2 = s.borrow_mut();
    println!("{}, {}", s1, s2);
}
```



### Cell和RefCell的性能比较

- `Cell` 只适用于 `Copy` 类型，用于提供值，而 `RefCell` 用于提供引用

- `Cell` 不会 `panic`，而 `RefCell` 会

  当非要使用内部可变性时，首选 `Cell`，只有你的类型没有实现 `Copy` 时，才去选择 `RefCell`



### Rc+RefCell的组合使用

这是一种常见组合，前者实现一个数据拥有多个所有者，后者可以实现数据的可变性。这种写法性能很高，对内存和`cpu`的损耗非常低：

```rust
use std::cell::RefCell;
use std::rc::Rc;
fn main() {
	let s = Rc::new(RefCell::new("hello"));
    
    let s1 = s.clone();
    let s2 = s.clone();
    
    s2.borrow_mut().push_str(" world");
    
    println!("{:?}\n{:?}\n{:?}"s, s1, s2);
}

//输出结果:
//hello world
//hello world
//hello world
```



# 多线程与并发编程

## 使用多线程

### 创建线程

使用`thread::spawn` 可以创建线程，线程中的代码是一个**闭包**；使用`handle.join()`让当前线程阻塞，等待子线程程运行结束；使用`thread::sleep`让当前线程休眠指定时间。

因为不确定线程执行顺序，使用 `move` 将变量的所有权从一个线程转移到另外一个线程。

```rust
use std::thread;
use std::time::Duration;

fn main(){
    let v = 100; 
    thread::spawn(move ||{
   		for i in 0..10 {
         	println!("hi number {} from the spawned thread!", v);   
            thread::sleep(Duration::from_millis(1));		
        }
    });
    
    handle.join().unwrap();
}
```



### 线程屏障

使用 `Barrier` 让多个线程都执行到某个点后，才继续一起往后执行

```rust
use std::sync::{Arc, Barrier};
use std::thread;

fn main(){
	let mut handles = Vec::with_capacity(6);
    let barrier = Arc::new(Barrier::new(6)); //创建barrier
    handles.push(thread::spawn(move|| {
        println!("before wait");
        b.wait(); //直到所有线程运行到这里，再继续执行
        println!("after wait");
    }));
    
    for handle in handles {
		handle.join().unwrap();
    }
}
```



### 线程局部变量

使用 `thread_local` 宏可以初始化线程局部变量，然后在线程内部使用该变量的 `with` 方法获取变量值。每个新的线程访问它时，都会使用它的初始值作为开始，**各个线程中的 `FOO` 值彼此互不干扰**。

```rust
use std::cell:RefCell;
use std::thread;

//FOO是使用 static 声明为生命周期为 'static 的静态变量。
thread_local!(static FOO: RefCell<u32> = RefCell::new(1));

let t = thread::spawn(move|| {
	FOO.woth(|f| {
       assert_eq!(*f.borrow(), 1);
        *f.borrow_mut() = 3;	//在子线程内部修改了FOO的值
    });
});

t.join().unwrap();

//尽管子线程做了修改，但主线程的局部值不变
FOO.with(|f| {
	assert_eq!(*f.borrow(), 2);
});
```

可以在结构体中使用

```rust
use std::cell::RefCell;

struct Foo;
impl Foo {
    thread_local! {
        static FOO: RefCell<usize> = RefCell::new(0);
    }
}

fn main() {
    Foo::FOO.with(|x| println!("{:?}", x));
}
```

可以通过引用的方式使用

```rust
use std::cell::RefCell;
use std::thread::LocalKey;

thread_local! {
    static FOO: RefCell<usize> = RefCell::new(0);
}
struct Bar {
    foo: &'static LocalKey<RefCell<usize>>,
}
impl Bar {
    fn constructor() -> Self {
        Self {
            foo: &FOO,
        }
    }
}
```

**第三方库 thread-local：**
	允许每个线程持有值的独立拷贝，十分好用。



### 初识锁与条件变量

条件变量(Condition Variables)必须与锁`mutex`配合使用，用来同步线程。

```rust
use std::thread;
use std::sync::{Arc, Mutex, Condvar};

fn main(){
    let pair = Arc::new((Mutex::new(false), Condvar::new()));
    let pair2 = pair.clone(); //Arc的clone是浅拷贝，指向同一个堆空间
    
    thread::spawn(move|| {
        let (lock, cvar) = &*pair2;
        let mut started = lock.lock().unwrap();
       	*started = true; //子线程获取锁，并将其修改为true
        println!("changing started!");
        cvar.notify_one(); //唤醒一个正在等待的线程
    });
    
 	//主线程
    let (lock.cvar) = &*pair;
    let mut started = lock.lock().unwrap();
    while !*started {
        //cvar.wait()会释放当前持有的锁，让其他进程可以访问该锁
        //让线程在该信号量上阻塞，等待唤醒
        started = cvar.wait(started).unwrap();
    }
    
}
```

**通过RAII + Drop的方式进行解锁：**

​	值得注意的是，`Mutex::lock()`会返回`MutexGuard<'_, T>`，它实现了 `Drop` trait。当 `MutexGuard` 被销毁时，`Drop` 实现会自动调用底层解锁操作。

当然可以直接调用`drop()`进行解锁。



### 让函数只被调用一次

如初始化变量的函数，只让一个线程调用一次就好，其他的线程会忽略该函数。

```rust
use std::thread;
use std::sync::Once;

static mut VAL: usize = 0;
static INIT: Once = Once::new();

fn main() {
    let handle1 = thread::spawn(move || {
        //在多线程条件下，只会调用一次。
        INIT.call_once(|| {
            unsafe {
                VAL = 1;
            }
        });
    });

    handle1.join().unwrap();
    println!("{}", unsafe { VAL });
}
```



### 使用thread::Builder构建线程

`std::thread::Builder`是rust标准库中的线程配置器，采用了一种叫做**构建器模式 (Builder Pattern)**的设计，允许你通过链式调用的方式清晰地配置线程。

特点是可以为线程命名和自定义栈大小：

```rust
use std::thread;

fn main(){
	//设置线程名和栈大小
    let builder = thread::Builder::new().name("thread_name".into()).stack_size(128 * 1024);
    
     //线程执行闭包中的代码
     //返回类型为io::Result<JoinHandle<T>>，允许更优雅的错误处理
    let handle = builder.spawn(||{
     	println!("hello world!");
        42	//返回值
    }).unwrap();
    
      //获取线程返回值
    let result = handle.join().unwrap();
}
```



### 作用域线程 scope thread

`std::thread::scope` 提供了一种“强制同步屏障”。编译器保证：**只要在这个作用域里创建的子线程，它们一定在作用域结束前全部执行完毕。**可以直接**借用（borrow）**父线程的局部变量，而无需 `move` 或 `Arc`，解决变量生命周期问题。

```rust
use std::thread;

fn main() {
	let a = vec![1,2,3];
    let b = vec![4,5,6];
    let (sum_a, sum_b) = thread::scope(|s|{
        let h1 = s.spawn(||{a.iter().sum::<i32>()});
        let h2 = s.spawn(||{b.iter().sum::<i32>()});
        (h1.join().unwrap(), h2.join().unwrap())
    });
}
```



### thread_local! 宏

为每个线程单独维护一个变量副本，互不影响，不需要加锁。

1. **日志和追踪系统**：你需要为每个请求或每个线程生成一个唯一的追踪 ID（Trace ID），在日志输出时附带这个 ID，这样即使多个线程同时打印日志，你也能分清哪些日志属于同一个处理流程。
2. **线程专属的缓存/缓冲区**：避免多个线程频繁申请和释放小内存，每个线程维护一个自己的小缓存池，提升性能。
3. **替代全局可变状态**：在某些单线程模型或异步运行时中，用来安全地存储当前上下文的运行时信息，而不需要使用全局锁

```rust
use std::thread;

// 定义了一个名为 THREAD_ID 的线程本地变量，类型为 RefCell<usize>
// 每个线程都会拥有自己独立的 RefCell<usize>，且初始值为 0
thraed_local! {
    static THREAD_ID = RefCell::new(0);
}

fn main（） {
    //通过闭包访问和修改
    THREAD_ID.with(|id| {
        *id.mut_borrow() = 1;
        println!("id_value: {}", *id.borrow()); //1
    });
    
    //子线程
    thread::spawn(||{
        THREAD_ID.with(|id| *id.mut_borrow() = 2;);
        println!("id_value: {}", *id.borrow()); //2
    });
}
```





## 线程间传递消息

rust引入消息通道`channel`的概念，发送者通过channel将消息发送给接收者。



### 多发送者单接收者

标准库提供了通道`std::sync::mpsc`，其中`mpsc`是**multiple producer, single consumer**的缩写，代表了该通道支持多个发送者，但是只支持唯一的接收者。

消息的发送仍遵循rust的所有权机制，若实现`Copy`特性就发送值的拷贝，否则发送信息的所有权。

```rust	
use std::sync::mpsc;
use std::thread;

fn main(){
	//创建消息通道，返回元组
    let (tx, rx) = mpsc::channel();
    
	thread::spawn(move ||{
        //发送消息
        tx.send(1).unwrap();
    });
    
    //接收信息，recv()会阻塞该线程，直到读取到值或通道关闭
    println!("receive : {}", rx.recv().unwrap());
}
```

`try_recv()`提供非阻塞的IO方式，如果没读取到信息或管道关闭时，他会返回错误。

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        tx.send(1).unwrap();
    });
	//子线程还没来得及发消息，主线程就读取了，完全有可能。
    println!("receive {:?}", rx.try_recv());
}
```



### for循环接收

```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let vals = vec![
            String::from("hi"),
            String::from("from"),
            String::from("the"),
            String::from("thread"),
        ];

        for val in vals {
            tx.send(val).unwrap();
            thread::sleep(Duration::from_secs(1));
        }
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```



### 使用多发送者

因为线程会拿走`tx`的所有权，需要使用`clone`对发送者进行拷贝，再分发给发送线程。

我们不清楚线程的执行顺序，所以输入通道的数据是无序的。但通道就是队列，遵循`FIFO`规则，所以数据输出顺序和输入顺序会保持一致。

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();
    let tx1 = tx.clone();
    thread::spawn(move || {
        tx.send(String::from("hi from raw tx")).unwrap();
    });

    thread::spawn(move || {
        tx1.send(String::from("hi from cloned tx")).unwrap();
    });

    for received in rx {
        println!("Got: {}", received);
    }
}
```



### 同步和异步通道

我们之前使用的`mpsc::channel`是**异步通道**，发送端不会受到接收端的影响。

同步通道**发送消息是阻塞的，只有在消息被接收后才解除阻塞**

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    //这里输入的值n为信道可以缓存的消息数
    //发送端可以无阻向信道发送n条消息，之后变为同步收发
    let (tx, rx)= mpsc::sync_channel(0); 

    let handle = thread::spawn(move || {
        println!("发送之前");
        tx.send(1).unwrap();	//必须等待接收端接收数据，才能继续执行
        println!("发送之后");
    });

    println!("receive {}", rx.recv().unwrap());
    handle.join().unwrap();
}
```



### 传递多种类型的数据

Rust 会按照枚举中占用内存最大的那个成员进行内存对齐，因此会造成内存上的浪费。

```rust
use std::sync::mpsc::{self, Receiver, Sender};
enum Fruit {
    Apple(u8),
    Orange(String)
}

fn main() {
    let (rx, tx) : (Sender<Fruit>, Receiver<Fruit>) = mpsc::channel();
    tx.send(Fruit::Orange("sweet".to_string().unwrap()));
    tx.send(Fruit::Apple(2)).unwrap();
    
    for _ in 0..2 {
        match rx.recv().unwrap(){
            Fruit::Apple(count) => println!("The count of apple: {}", count),
            Fruit::Orange(flavor) => println!("The flavor is {}", flavor),
        }
    }
}
```



### 一些易错点

因为`Drop`特性，**所有发送者被`drop`或者所有接收者被`drop`后，通道会自动关闭**。

有时需要手动`drop`掉`tx` ，让通道关闭，防止读线程一直阻塞。



## 线程同步

在多线程编程中，同步性极其的重要，Rust 中有多种方式可以实现同步性：**消息传递和共享内存**。

### 互斥锁Mutex

`Mutex::new(0)`创建锁，`m.lock()`获得锁并返回一个**智能指针`MutexGuard<T>`：**

- 它实现了`Deref`特征，会被自动解引用后获得一个引用类型，该引用指向`Mutex`内部的数据
- 它还实现了`Drop`特征，在超出作用域后，自动释放锁，以便其它线程能继续获取锁

使用`Arc<T>`克隆这些锁，让子线程可以安全的`move`掉锁的所有权。使用`Arc<T>`而不使用`Rc<T>`的原因是**`Arc<T>`是多线程安全的**。

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
	let counter = Arc::new(Mutex::new(0));
    let mut handles = vec![];
    
    for _ in 0..10 {
		let counter = Arc::clone(&counter);
         let handle = thread::spawn(move||{
             let mut num = counter.lock().unwrap();
             *num += 1;
        });  
        handles.push(handle);
    }

    for handle in handles {
		handle.join().unwrap();
    }
    
    println!("Rusult: {}", *counter.lock().unwrap());
}
```

`Rc<T>/RefCell<T>`用于单线程内部可变性， `Arc<T>/Mutex<T>`用于多线程内部可变性。



### 死锁

死锁产生的原因多种多样，往往和不规则的加锁顺序，一个线程持有多个锁或设计逻辑上的漏洞有关，在此不再赘述。

`try_lock()`会尝试获取锁，如果获取失败，则返回错误，不会被阻塞。



### 读写锁RwLock

读写锁`RwLock`允许多个线程读数据，但同一时间只能有一个线程写数据。

```rust
use std::sync::RwLock

fn main() {
	let lock = RwLock::new(5);
    
    //同一时间，多个读数据
    {
		let r1 = lock.read().unwrap();
        let r2 = lock.read().unwrap();
        assert_eq!(*r1, 5);
        assert_eq!(*r2, 5);
    } //读锁在此处被drop
    
    //同一时间只允许一个写
    {
        let mut w = lock.write().unwrap();
        *w += 1;
        assert_eq!(*w, 6);
    } //写锁在此处被drop
}
```

还可以使用`try_read()`和`try_write()`进行非阻塞读写，所以读写锁在项目中会安全的多。



**Mutex和RwLock的性能比较：**

无论是在简单性和性能上`Mutex`都会更好，如果不知道用什么，就使用`Mutex`。

使用`RwLock`要确保满足以下两个条件：**并发读，且需要对读到的资源进行"长时间"的操作**。



### 信号量Semaphore

`tokio`中提供的`Semaphore`实现: [`tokio::sync::Semaphore`](https://github.com/tokio-rs/tokio/blob/master/tokio/src/sync/semaphore.rs)

信号量的本质是一个受保护的整数计数器，通过原子操作（P/V 操作）来协调多个线程的执行，事先线程间的互斥与同步。

```rust
use std::sync::Arc;
use tokio::sync::Semaphore;

#[tokio::main]
async fn main() {
    let semaphore = Arc::new(Semaphore::new(3));
    let mut join_handles = Vec::new();

    for _ in 0..5 {
        let permit = semaphore.clone().acquire_owned().await.unwrap();
        join_handles.push(tokio::spawn(async move {
            //
            // 在这里执行任务...
            //
            drop(permit);
        }));
    }

    for handle in join_handles {
        handle.await.unwrap();
    }
}
```



## Atomic 原子类型与内存顺序

原子操作是通过指令提供的支持，因此它的性能相比锁和消息传递会好很多。相比较于锁而言，原子类型不需要开发者处理加锁和释放锁的问题，同时支持修改，读取等操作，还具备较高的并发性能，几乎所有的语言都支持原子类型。

### 使用Atomic做全局变量

和`Mutex`一样`Atomic`的值具有内部可变性，无需声明其为`mut`

```rust
use std::ops::Sub;
use std::sync::stomic::{AtomicU64, Ordering};
use std::thread::{self, JoinHandle};

const N_TIMES: u64 = 100000;
const N_THREADS: usize = 10;

//使用原子类型来创建全局变量
static R: AtomicU64 = AtomicU64::new(0);

//创建线程并返回句柄
fn add_n_times(n: u64) -> JoinHandle<()> {
	thread::spawn(move ||{
        for _ in 0..n {
			R.fetch_add(1, Ordering::Relaxed);
        }
    })
}

fn main() {
 	let mut threads = Vec::with_capacity(N_THREADS);
    for _ in 0..N_THREADS {
		threads.push(add_n_times(N_TIMES));
    }
    
    for thread in threads {
		thread.join().unwrap();
    }
    
    //和期望的结果进行比较，判断有没有脏数据
    //通过Ordering::Relax来限定内存顺序
    assert_eq!(N_TIMES * N_THREADS as u64, R.load(Ordering::Relaxed));
}
```



### 内存顺序

内存顺序是指 CPU 在访问内存时的顺序，会受到一下因素的影响：

- 代码中的先后顺序
- 编译器优化导致在编译阶段发生改变(内存重排序 reordering)
- 运行阶段因 CPU 的缓存机制导致顺序被打乱



### 补充：缓存一致性

**缓存一致性（Cache Coherence）是指在多核或多处理器系统中，保证各核心私有缓存与主存之间、以及各缓存副本之间数据一致的硬件机制**。该机制是为了解决多核并发访问同一内存地址时因**私有缓存副本不同步**可能导致的数据不一致的问题。

![image-20260920161842267](C:\Users\86567\AppData\Roaming\Typora\typora-user-images\image-20260920161842267.png)

### 限定内存顺序的 5 个规则

rust 提供了`Ordering::Relaxed`用于限定内存顺序了，事实上，该枚举有 5 个成员:

- **Relaxed**， 这是最宽松的规则，它对编译器和 CPU 不做任何限制，可以乱序
- **Release 释放**，设定内存屏障(Memory barrier)，保证它之前的操作永远在它之前，但是它后面的操作可能被重排到它前面
- **Acquire 获取**, 设定内存屏障，保证在它之后的访问永远在它之后，但是它之前的操作却有可能被重排到它后面，往往和`Release`在不同线程中联合使用
- **AcqRel**, 是 *Acquire* 和 *Release* 的结合，同时拥有它们俩提供的保证。比如你要对一个 `atomic` 自增 1，同时希望该操作之前和之后的读取或写入操作不会被重新排序
- **SeqCst 顺序一致性**， `SeqCst`就像是`AcqRel`的加强版，它不管原子操作是属于读取还是写入的操作，只要某个线程有用到`SeqCst`的原子操作，线程中该`SeqCst`操作前的数据操作绝对不会被重新排在该`SeqCst`操作之后，且该`SeqCst`操作后的数据操作也绝对不会被重新排在`SeqCst`操作前。

以`Release`和`Acquire`为例，构建内存屏障：
```rust
use std::thread::{self, JoinHandle};
use std::sync::atomic::{Ordering, AtomicBool};

static mut DATA: u64 = 0;
static READY: AtomicBool = AtomicBool::new(false);

//保证生产者操作的内存序
fn producer() -> JoinHandle<()> {
	thread::spawn(move || {
        unsafe {
			DATA = 100;
        }
        //在其之前的代码顺序只会在其之前
        READY.store(true, Ordering::Release);
    })
}

//保证消费者的内存序
fn consumer() -> JoinHandle<()> {
	thread::spawn(move || {
        //保证之后的代码顺序只会在其之后
        while !READY.load(Ordering::Acquire) {}
        assert_eq!(100, unsafe { DATA });
    })
}
```

**内存顺序的选择**

1. 不知道怎么选择时，优先使用`SeqCst`，虽然会稍微减慢速度，但是慢一点也比出现错误好
2. 多线程只计数`fetch_add`而不使用该值触发其他逻辑分支的简单使用场景，可以使用`Relaxed`参考 [Which std::sync::atomic::Ordering to use?](https://stackoverflow.com/questions/30407121/which-stdsyncatomicordering-to-use)



### 多线程使用Atomic

在多线程环境中要使用`Atomic`需要配合`Arc`：

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};
use std::{hint, thread};

fn main() {
    let spinlock = Arc::new(AtomicUsize::new(1));

    let spinlock_clone = Arc::clone(&spinlock);
    let thread = thread::spawn(move|| {
        spinlock_clone.store(0, Ordering::SeqCst);
    });

    // 等待其它线程释放锁
    while spinlock.load(Ordering::SeqCst) != 0 {
        //这是关键的性能优化提示。
        //每次循环迭代调用它，向 CPU 发出信号："我正在自旋等待"
        hint::spin_loop();
    }

    if let Err(panic) = thread.join() {
        println!("Thread had an error: {:?}", panic);
    }
}
```



## 基于Send和Sync的线程安全

`Send`和`Sync`是 Rust 安全并发的重中之重，但是实际上它们只是标记特征(marker trait，该特征未定义任何行为，因此非常适合用于标记)：

- 实现`Send`的类型可以在线程间安全的传递其所有权
- 实现`Sync`的类型可以在线程间安全的共享(通过引用)

一个类型要在线程间安全的共享的前提是，指向它的引用必须能在线程间传递。由上可知，**若类型 T 的引用`&T`是`Send`，则`T`是`Sync`**。

绝大部分类型都实现了`Send`和`Sync`，常见的未实现的有：裸指针、`Cell`、`RefCell`、`Rc` 等，为裸指针实现Send和Sync特征：

```rust
use std::thread;
use std::sync::Arc;
use std::sync::Mutex;

#[derive(Debug)]
struct MyBox(*const u8);
//为裸指针实现Send和Sync特征
unsafe impl Send for MyBox {}
unsafe impl Sync for MyBox {}

fn main() {
    let b = &MyBox(5 as *const u8);
    let v = Arc::new(Mutex::new(b));
    let t = thread::spawn(move || {
        let _v1 =  v.lock().unwrap();
    });

    t.join().unwrap();
}
```



## 子进程和IPC

### 创建子进程

使用`std::process::Command`灵活的创建子进程，根据配置不同返回不同的结果。

```rust
use std::process::{Command, Stdio};
use std::io::Write;

fn main() {
	//1.捕获子进程输出，底层自动帮你创建了管道（pipes），在内部完成了子进程的等待（waitpid）。
    let output = Command::new("echo").args("hello world!").output().ecpect("Command output error!");
	
    //2.手动与子进程交互
    //spawn可以返回
	let child = Command::new("echo").stdin(Stdio::piped()).stdout(Stdio.piped()).spawn().unwrap();
    {
        //提取句柄,离开作用域会自动drop掉stdin，关闭管道
     	let stdin = child.stdin.as_mut().unwrap();
   		stdin.write_all(b"hello\n").unwrap();
    } 
	// 读取输出
	let output = child.stdout.as_mut().unwrap();
    //等待子进程结束，防止出现僵尸进程。
    let _status = child.wait().unwrap();
    
    
    //3.配置环境变量
    let output = Command::new("printenv")
    .arg("APP_ENV")
    .env("APP_ENV", "production") // 只给子进程添加 APP_ENV=production
    .current_dir("/tmp")          // 改变子进程的工作目录
    .output()
    .expect("Failed to execute");

	let env_val = String::from_utf8_lossy(&output.stdout);
	println!("子进程看到的 APP_ENV: {}", env_val.trim());
}
```

因为只有写管道关闭读管道才会解除阻塞。所以在读数据之前确保所有的stdout都可以被drop掉，否则会出现死锁。



### 对执行结果进行错误处理

```rust
pub fn run_command_with_result(program: &str, args: &[&str]) -> io::Result<String> {
    //tring::from_utf8有可能会出错
    //如果错误就返回io::Error类型的错误
    let output =  Command::new(program).args(args).stdout(Stdio::piped()).output()?;
    match String::from_utf8(output.stdout) {
        Ok(str) => Ok(str),
        Err(e) => Err(io::Error::new(std::io::ErrorKind::InvalidData, e)),
        
    }
}

//下面的写法也行
fn run_command_with_result() -> Result<String, io::Error> {
    let output = Command::new("non_existent_program") // 故意写错或者是存在的程序
        .output()?; // '?' 操作符：如果 spawn 失败，直接返回 Err；如果成功，继续

    if output.status.success() {
        Ok(String::from_utf8_lossy(&output.stdout).to_string())
    } else {
        // 如果程序运行了但报错了（比如 ls 了一个不存在的文件）
        Err(io::Error::new(io::ErrorKind::Other, "Command failed"))
    }
}
```



# rust内联汇编

## 内联汇编的基本组成

Rust 内联汇编用的是 `core::arch::asm!`，它的整体结构可以拆成四部分：**模板字符串 + 操作数 + clobber + options**。

```rust
let x: u64 = 3;
let y: u64;

unsafe {
    core::arch::asm!(
        "add {0}, 5",	//模板字符串，就是插入的汇编代码，不同架构汇编指令是不同的
        inout(reg) x => y,	//带入位置参数0，inout是操作数，reg是指定的寄存器，
    );
}
// y == 8
```



可以为位置参数起名字，在之后进行赋值：

```rust

unsafe {
    core::arch::asm!(
        "add {val}, 5",
        val = inout(reg) x => y,
    );
}
// y == 8
```



可使用的**操作数**：

```rust
in(reg) expr           // 输入：把 Rust 表达式放进一个通用寄存器
out(reg) var           // 输出：汇编结束后，从这个寄存器写回 var
out(reg) _             // 丢弃输出：寄存器被写了，但 Rust 不关心结果
inout(reg) var         // 输入输出：同一个寄存器既是输入也是输出
inout(reg) in_var => out_var  // 输入和输出可以是不同变量
inlateout(reg) var     // 晚写输出：保证所有输入读完之后才写输出
```

可以指定具体寄存器：
```rust
in("a0") value
out("ra") ret
inlateout("sp") sp
```

### 

clobber 用来告诉编译器：**这段汇编还破坏了哪些状态。**
options 用来告诉编译器：**这段汇编有什么特殊行为。**

**常用option：**

指定option可以让编译器更好的优化代码

| option            | 含义                      |
| :---------------- | :------------------------ |
| `preserves_flags` | 汇编没有修改条件标志      |
| `nostack`         | 没有往栈上 push / 修改 sp |
| `nomem`           | 不读写内存                |
| `readonly`        | 只读内存，不写            |
| `pure`            | 无副作用，输出只依赖输入  |
| `noreturn`        | 不会返回，比如直接跳转走  |
| `att_syntax`      | x86 上使用 AT&T 语法      |



## niked_asm!宏和裸函数

`naked_asm!` 是 Rust 1.88 稳定下来的宏，专门配合 `#[unsafe(naked)]` 裸函数使用。它的核心作用就是：**让函数体完全由你写的汇编指令组成，编译器不插入任何 prologue / epilogue。**

关键限制：**`naked_asm!` 里不能用 `in(reg) old` 这种操作数绑定。**函数参数只能通过 ABI 约定的寄存器位置来访问。RISC-V 64 的 calling convention 里，第一个参数在 `a0/x10`，第二个在 `a1/x11`，所以用 `a0、a1` 占位即可。

在裸函数中，编译器不会为函数添加额外指令，你需要手动管理上下文。

```rust
#[unsafe(naked)]
pub unsafe fn switch_context(old: &mut TaskContext, new: &TaskContext) {
    naked_asm!(
        "sd ra, 0(a0)",
        "sd sp, 8(a0)",
        "ld ra, 0(a1)",
        "ld sp, 8(a1)",
        "mv a0, x0",
        "mv a1, x0",
        "ret",
    );
}
```

**注意事项:**

- 必须标记 `unsafe`，因为跳过了编译器的 ABI 安全网。

- 函数签名只是给 Rust 调用方看的，汇编里不自动处理参数传递。

- 必须自己写 `ret`（或等价跳转），否则函数不会返回。

- 建议写清晰的 SAFETY 注释，说明遵循的 ABI、寄存器约定、被破坏的寄存器等。

  

# 异步编程

## async/await简单入门

引入`futures`包:

```tex
[dependencies]
futures = "0.3"
```



创建异步函数：
```rust
//异步函数会返回Future
async fn do_something() {
	prineln!("lets goooo!");
}                          
```



返回的Futures不能直接运行。需要执行器`executor`配合使用：
```rust
use futures::executor::block_on;

async fn hello_world() {
	println!("hello world!");
}

fn main() {
	let future = hello_world(); //获取Future
    block_on(future);	//执行`Future`并等待运行完成,block_on会阻塞当前线程
}
```



**使用`.await`同步异步函数：**

​	`.await`不会只是当前线程，而是异步等待`Future A`完成，等待过程中，还可以执行其他Future，最终实现并发处理。他是异步编程的重要组成部分，让一个线程内实现多任务。

```rust
use futures::executor::block_on;
async fn hello_world() {
	hello_cat().await;
    println!("hello world");
}

async fn hello_cat() {
	 println!("hello world");
}

fn main() {
    let future = hello_world();
    block_on(future);
}
```



## Future特征

### Future的实现原理

`	Future` 的定义：它是一个能产出值的异步计算。以下是简化版Future的代码：

```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}
```

​	若在当前 轮询`poll` 中， `Future` 可以被完成，则会返回 `Poll::Ready(result)` ，反之则返回 `Poll::Pending`， 并且安排一个 `wake` 函数：当未来 `Future` 准备好进一步执行时， 该函数会被调用，然后管理该 `Future` 的执行器(例如上一章节中的`block_on`函数)会再次调用 `poll` 方法，此时 `Future` 就可以继续执行了。

这种接到通知再执行的方式，比对所有Future定期遍历更加高效。



**如何同时运行多个 `Future`：**

```rust
trait SimpleFuture {
    type Output;
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}

/// 一个SimpleFuture，它会并发地运行两个Future直到它们完成
///
/// 之所以可以并发，是因为两个Future的轮询可以交替进行，一个阻塞，另一个就可以立刻执行，反之亦然
pub struct Join<FutureA, FutureB> {
    // 结构体的每个字段都包含一个Future，可以运行直到完成.
    // 等到Future完成后，字段会被设置为 `None`. 这样Future完成后，就不会再被轮询
    a: Option<FutureA>,
    b: Option<FutureB>,
}

impl<FutureA, FutureB> SimpleFuture for Join<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        // 尝试去完成一个 Future `a`
        if let Some(a) = &mut self.a {
            if let Poll::Ready(()) = a.poll(wake) {
                self.a.take();	//Future运行完成，设置为None
            }
        }

        // 尝试去完成一个 Future `b`
        if let Some(b) = &mut self.b {
            if let Poll::Ready(()) = b.poll(wake) {
                self.b.take();
            }
        }

        if self.a.is_none() && self.b.is_none() {
            // 两个 Future都已完成 - 我们可以成功地返回了
            Poll::Ready(())
        } else {
            // 至少还有一个 Future 没有完成任务，因此返回 `Poll::Pending`.
            // 当该 Future 再次准备好时，通过调用`wake()`函数来继续执行
            Poll::Pending
        }
    }
}
```



**使用 `Future` 特征去表达异步控制流（简化版本）：**

```rust
/// 一个SimpleFuture, 它使用顺序的方式，一个接一个地运行两个Future
//
// 注意: 由于本例子用于演示，因此功能简单，`AndThenFut` 会假设两个 Future 在创建时就可用了.
// 而真实的`Andthen`允许根据第一个`Future`的输出来创建第二个`Future`，因此复杂的多。
pub struct AndThenFut<FutureA, FutureB> {
    first: Option<FutureA>,
    second: FutureB,
}

impl<FutureA, FutureB> SimpleFuture for AndThenFut<FutureA, FutureB>
where
    FutureA: SimpleFuture<Output = ()>,
    FutureB: SimpleFuture<Output = ()>,
{
    type Output = ();
    fn poll(&mut self, wake: fn()) -> Poll<Self::Output> {
        if let Some(first) = &mut self.first {
            match first.poll(wake) {
                // 我们已经完成了第一个 Future， 可以将它移除， 然后准备开始运行第二个
                Poll::Ready(()) => self.first.take(),
                // 第一个 Future 还不能完成
                Poll::Pending => return Poll::Pending,
            };
        }

        // 运行到这里，说明第一个Future已经完成，尝试去完成第二个
        self.second.poll(wake)
    }
}
```



### pin特性

```rust
trait Future {
    type Output;
    fn poll(
        // 首先值得注意的地方是，`self`的类型从`&mut self`变成了`Pin<&mut Self>`:
        self: Pin<&mut Self>,
        // 其次将`wake: fn()` 修改为 `cx: &mut Context<'_>`:
        cx: &mut Context<'_>,
    ) -> Poll<Self::Output>;
}
```

`pin`会创建一个无法被移动的`Future`，他有固定的内存地址，意味着可以通过存储指针进行访问。

其次，从 `wake: fn()` 变成了 `&mut Context<'_>` 。意味着 `wake` 函数可以携带数据了，他让Future的管理变得更加简单。



### 使用Waker唤醒任务

我们创建一个简单的定时器`Future`：

```rust
//引入需要的包
use std::{
    future::Future,
    pin::Pin,
    sync::{Arc, Mutex},
    task::{Context, Poll, Waker},
    thread,
    time::Duration,
};
```

定义数据结构`TimeFuture`，因为他需要在新线程和Future定时器中共享，使用`Arc<Mutex<Y>>`共享状态

```rust
pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

/// 在Future和等待的线程间共享状态
struct SharedState {
    /// 定时(睡眠)是否结束
    completed: bool,

    /// 当睡眠结束后，线程可以用`waker`通知`TimerFuture`来唤醒任务
    waker: Option<Waker>,
}
```

实现Futrue特征：

```rust
impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        // 通过检查共享状态，来确定定时器是否已经完成
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            // 设置`waker`，这样新线程在睡眠(计时)结束后可以唤醒当前的任务，接着再次对`Future`进行`poll`操作,
            //
            // 下面的`clone`每次被`poll`时都会发生一次，实际上，应该是只`clone`一次更加合理。
            // 选择每次都`clone`的原因是： `TimerFuture`可以在执行器的不同任务间移动，如果只克隆一次，
            // 那么获取到的`waker`可能已经被篡改并指向了其它任务，最终导致执行器运行了错误的任务
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

再来创建一个 API 用于构建定时器和启动计时线程:

```rust
impl TimerFuture {
    /// 创建一个新的`TimerFuture`，在指定的时间结束后，该`Future`可以完成
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        // 创建新线程
        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            // 睡眠指定时间实现计时功能
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            // 通知执行器定时器已经完成，可以继续`poll`对应的`Future`了
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```

​	有了定时器，我们需要配合执行器让Future动起来。



### 执行器Executor

```rust
[dependencies]
futures = "0.3"
```

```rust
use {
    futures::{
        future::{BoxFuture, FutureExt},
        task::{waker_ref, ArcWake},
    },
    std::{
        future::Future,
        sync::mpsc::{sync_channel, Receiver, SyncSender},
        sync::{Arc, Mutex},
        task::{Context, Poll},
        time::Duration,
    },
    // 引入之前实现的定时器模块
    timer_future::TimerFuture,
};
```









## Pin和Unpin

### 深入理解Pin

pin是一个结构体，它包裹一个指针，能够保证该指针指向的数据不被移动。自引用数据结构的值是不能被移动的。有趣的是，被pin住的值都必须是`!Unpin`的。

```rust
pub struct Pin<P> {
    pointer: P,
}
```



使用下面的例子理解为什么被pin的指针可以保持不变。

```rust
#[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());
}
```

```text
//text1 -> a: test1, b: test1
//text2 -> a: test1, b: test2
```

![img](https://pica.zhimg.com/80/v2-eaeb33da283dc1063b862d2307821976_1440w.jpg)

由上面的例子可以看到，指针确实被转移了，但指针指向的地址不变，取出来的值就是固定地址的值。



## 异步运行时tokio

