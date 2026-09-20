# 多线程与并发编程

## 使用多线程

使用`thread::spawn` 可以创建线程，线程中的代码是一个**闭包**；使用`handle.join()`让当前线程阻塞，等待子线程程运行结束；使用`thread::sleep`让当前线程休眠指定时间。

因为不确定线程执行顺序，使用 `move` 来将变量的所有权从一个线程转移到另外一个线程。

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

