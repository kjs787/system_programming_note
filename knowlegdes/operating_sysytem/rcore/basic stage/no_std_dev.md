

# no_std_dev

 在 `#![no_std]` 环境下，**只使用与平台无关的核心库（`core`）和可选的分配器库（`alloc`）**。



## 编写堆内存分配器

堆分配器原理和内存对齐:

```rust
//! # Bump Allocator (no_std)
//!
//! Implement the simplest heap memory allocator: a Bump Allocator (bump pointer allocator).
//!
//! ## How It Works
//!
//! A Bump Allocator maintains a pointer `next` to the "next available address".
//! On each allocation, it aligns `next` to the requested alignment, then advances by `size` bytes.
//! It does not support freeing individual objects (`dealloc` is a no-op).
//!
//! ```text
//! heap_start                              heap_end
//! |----[allocated]----[allocated]----| next |---[free]---|
//!                                        ^
//!                                    next allocation starts here
//! ```
//!
//! ## Task
//!
//! Implement `BumpAllocator`'s `GlobalAlloc::alloc` method:
//! 1. Align the current `next` up to `layout.align()`
//!    Hint: `align_up(addr, align) = (addr + align - 1) & !(align - 1)`
//! 2. Check if the aligned address plus `layout.size()` exceeds `heap_end`
//! 3. If it exceeds, return `null_mut()`; otherwise atomically update `next` with `compare_exchange`
//!
//! ## Key Concepts
//!
//! - `core::alloc::{GlobalAlloc, Layout}`
//! - Memory alignment calculation
//! - `AtomicUsize` and `compare_exchange` (CAS loop)
pub struct BumpAllocator {
    heap_start: usize,
    heap_end: usize,
    next: AtomicUsize,
}

unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // TODO: Implement bump allocation
        //
        // Steps:
        // 1. Load current next (use Ordering::SeqCst)
        // 2. Align next up to layout.align()
        //    Hint: align_up(addr, align) = (addr + align - 1) & !(align - 1)
        // 3. Compute allocation end = aligned + layout.size()
        // 4. If end > heap_end, return null_mut()
        // 5. Atomically update next to end using compare_exchange
        //    (if CAS fails, another thread raced — retry in a loop)
        // 6. Return the aligned address as a pointer
        loop {
            let current = self.next.load(Ordering::SeqCst);
            let align = layout.align();
            let size = layout.size();

            let aligned = (current + align - 1) & !(align - 1); 
            let end = aligned + size;
            if end > self.heap_end {
                return null_mut();
            }else {
                match self.next.compare_exchange(current, end,Ordering::SeqCst,Ordering::Acquire) {
                    Ok(_) => return aligned as *mut u8,
                    Err(_) => continue,
                };
                
            }
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // Bump allocator does not reclaim individual objects — leave empty
    }
}
```



## 编写链表组成的freeblock分配器

编写内存分配器时，使用双重指针遍历列表，替换如下危险代码：

```rust
// TODO: Step 1 — traverse free_list, find a suitable block (first-fit)
        //
        // Hints:
        // - Use prev_ptr and curr to traverse the list
        // - Check if curr address satisfies align, and (*curr).size >= size
        // - If found, remove it from the list (update prev's next or the free_list head)
        // - Return curr as *mut u8
        let mut curr = self.free_list_head();
        let mut prev_ptr = curr;
        while !curr.is_null() {
            let curr_addr = curr as usize; 
            if (curr_addr % (align - 1)) == 0 && (*curr).size >= size {
                // found a block
                (*prev_ptr).next = (*curr).next;
                return curr as *mut u8

            }
            
            prev_ptr = curr;
            curr =  (*curr).next;
        }

```



pre_ptr指向前一个节点的next指针，curr指向当前节点的指针。我们不需要为前一个节点额外申请空间，还避免了头节点遍历不到和解引用空指针的问题。

```rust
 let mut prev_ptr = &mut self.free_list_head();
        let mut curr = *prev_ptr;

        while !curr.is_null() {
            let curr_addr = curr as usize; 
            if (curr_addr % (align - 1)) == 0 && (*curr).size >= size {
                // found a block
                *prev_ptr = (*curr).next;
                return curr as *mut u8

            }
            
            prev_ptr = &mut (*curr).next;
            curr =  (*curr).next
        }
```



alloc在进行分配时，选了内存对齐后的大小`aligned`进行分配，所以在对释放内存块时，应该将(*new_freeblock_ptr).size = size;设置为其真实大小。

```rust
unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        let size = layout.size().max(core::mem::size_of::<FreeBlock>());

        // TODO: Insert the freed block at the head of free_list
        //
        // Steps:
        // 1. Cast ptr to *mut FreeBlock
        // 2. Write FreeBlock { size, next: current list head }
        // 3. Update free_list head to ptr
        let new_freeblock_ptr = ptr as *mut FreeBlock;

        // 获取分配的真实物理大小，因为内存对齐，其内存块的大小可能不为64
        let original_size = core::ptr::read(&(*new_freeblock_ptr).size);
    
        // 真实大小应该是 max(原始大小, 最小大小)
        let size = original_size.max(size);

        (*new_freeblock_ptr).size = size;
        (*new_freeblock_ptr).next = self.free_list_head();
        self.set_free_list_head(new_freeblock_ptr);
    }
```





## 编写syscall wrapper

### 这个 wrapper 做了什么？

- **ABI 描述层**：`SyscallABI` + `x86_64_abi/aarch64_abi/riscv64_abi` 负责描述每个架构的指令、寄存器、clobber 和系统调用号。
- **执行层**：`syscall3` 负责真正触发系统调用，内部用 `cfg` 分支选择不同架构的内联汇编。
- **封装层**：`sys_write/sys_read/sys_close/sys_exit` 是高级封装，调用者不需要关心系统调用号。
- **桥接层**：`current_abi()` 根据当前编译目标架构返回对应 ABI，让封装函数可以拿到正确的 `sys_write`、`sys_read` 等编号。

`syscall_wrapper` 模块让这些程序能够**脱离标准库的依赖**，直接通过内联汇编（`asm!`）向内核发起请求，它是跨平台的。



## 编写fd表

比较简单。考了vec和arc的知识点，可以注意一下take的使用，减少arc引用计数的方法：

```rust
pub struct FdTable {
    // TODO: Design the internal structure
    // Hint: use Vec<Option<Arc<dyn File>>>
    //       the index is the fd number, None means the fd is closed or unallocated
    ftable: Vec<Option<Arc<dyn File>>>,
}
```



```rust
    /// Close an fd. Returns true on success, false if the fd doesn't exist or is already closed.
    pub fn close(&mut self, fd: usize) -> bool {
        // TODO
        let item = self.ftable.get_mut(fd);
        match item {
            Some(slot) => {
                if slot.take().is_some() {
                    return true;
                }else{
                    return false;
                }
            },
            None => return false,
        }
    }

```



# os_concurrency

## atomic_counter

一些有关AtomicU64，实现无锁化和线程安全的方法使用。



## atomic_ordering

有关内存序与线程同步

//! ## 核心概念 //! - `Ordering::Relaxed`：不提供任何同步保证 //! - `Ordering::Acquire`：用于读操作，防止后续的读写操作被重排序到该操作之前 //! - `Ordering::Release`：用于写操作，防止前面的读写操作被重排序到该操作之后 //! - `Ordering::AcqRel`：同时具备 Acquire 和 Release 的语义 //! - `Ordering::SeqCst`：顺序一致性（全局统一排序）
