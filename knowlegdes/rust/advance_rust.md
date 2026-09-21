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

