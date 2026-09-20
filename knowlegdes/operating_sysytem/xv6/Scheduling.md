# 调度的实现

## 上下文切换

从一个线程切换到另一个线程需要保存旧线程的CPU寄存器，并恢复新线程先前保存的寄存器；栈指针和程序计数器被保存和恢复的事实意味着CPU将切换栈和执行中的代码。



**这张图展示了进程调度时，内核线程与调度线程之间的关系：**

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787190631458-c6b5119f-d854-4139-9057-72fbf5ba5e57.png)

调度开始时，shell进程会经过系统调用/陷阱陷入内核态，保存上下文信息到他的内核栈 `kstack shell` 。之后内核调用 `swtch` 函数，将当前执行流从 `kstack shell` 切换到 `kstack scheduler`，调度器开始运行。经由调度策略决定调用cat进程，再次使用swtch将执行流切换到`cat kstack`，内核恢复 `cat` 进程之前保存的用户态上下文（图中标注 “restore”），然后返回用户空间，`cat` 进程继续执行。



**什么是内核线程：**

内核线程是操作系统内核自身创建并管理的执行单元，它不依附于任何用户进程，直接在内核态运行，负责处理系统级任务，如内存管理、I/O调度、进程调度等。

内核线程与普通用户进程的关键区别在于：

**没有用户空间**：它不拥有独立的虚拟地址空间，其 mm 字段为 NULL，只能访问内核空间的内存。

**由内核直接调度**：它由操作系统内核的调度器统一管理，调度策略与用户进程相同，但优先级通常更高。

**执行内核函数**：它本质上是“被委托执行特定内核任务”的轻量级进程，例如周期性同步内存页、管理文件系统日志、处理延迟任务等。



## xv6调度过程

从一个实际进程开始的底层调用过程

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787887384392-b68cf9ef-ace1-4540-95e4-58bf69a271d4.png)

首先进程会调用`yield`，主动放弃cpu。

内核需要确保进程只会持有本进程的锁，并已经释放其他资源的锁，且进程状态已经由`running`更改为`runable`。



![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787887771307-1518d649-9a15-4da1-a025-161ca0f65b80.png)

紧接着调用`sched`，他会对当前进程状态进行安全检查。检查无误后调用`swtch`，将当前进程上下文保存到`p->context`中，从 cpu->scheduler 中恢复寄存器。这实际上是将执行流切换到了**该 CPU 对应的调度器线程的内核栈上。**



![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787888525290-c1455e65-aa5f-4a38-8caa-ec6e37611deb.png)

进下来cpu会执行调度器代码：遍历进程列表，寻找可以执行的进程，设置该进程状态为`RUNNING`，设置`c->proc`指向该进程。再次调用 `swtch`，这次是从调度器栈切换到新选中进程的内核栈。



**思考：yield何时会调用？**

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787892320128-655c44bd-50c2-4f60-9a45-80e8b982f039.png)

不难想象yield()会在定时器中断时调用，这里是trap.c/usertrap()函数



**锁的特殊传递过程：**
	yield()会获取进程锁`p->lock`，又修改了进程状态为`RUNNABLE`，再调用`swtch`。但swtch执行过程中进程必须持有锁，否则该进程可能会被其他核执行。等待调度器完成所有必要的工作之后，就会释放锁`p->lock`



**当调度器选择一个从未运行过的新进程（刚被 fork 出来）时，流程略有不同：**

它不会从`sched`返回（因为它从未调用过 sched）,它会从一个名为`forkret`的特殊函数开始执行。

forkret 的唯一目的是释放 p->lock（因为调度器在切换给它时持有了这个锁），然后进入正常的用户空间陷阱返回流程 (usertrapret)。



## 调度过程中的协程

以上的调度过程中，cpu会在`scheduler()`和`sched()`两个线程中来回切换。他们执行顺序和上下文是可预测的，它们共享同一个 CPU，但各自拥有独立的栈和寄存器上下文。**协程是一种非抢占式的多任务处理组件，它们自愿地让出控制权。**



**协程模式的设计优势:**

**简化调度逻辑**：调度器无需处理复杂的“启动新进程”逻辑（除了首次），绝大多数切换都是“恢复已有进程”，代码结构清晰。

**保证状态一致性**：由于切换点是固定的，可以精确控制锁的持有与释放时机，避免在中间状态被其他 CPU 干扰。

**高效上下文切换**：`swtch` 只保存/恢复被调用者保存的寄存器，开销极小，且切换路径短，有利于性能。



# wake和sleep在xv6中的应用

## mycpu()和myproc()

**mycpu()：**

kernel/proc.c：

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787900875441-aa805cb6-c33c-4d7c-8ad4-49d053ed9527.png)

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787900858799-648bd116-c25d-44d6-829a-24c37d03a763.png)

kernel/proc.h

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787900906871-badb4576-ac2c-456b-b2e8-d03d66dfea5f.png)

每个cpu核都会有一个`hartid`被存储在tp寄存器中，内核维护一个cpu结构体数组，通过hartid进行检索。mycpu()通过检索，返回指向该核的`struct cpu`的指针。

`cpuid`和`mycpu`的返回值很脆弱：如果定时器中断并导致线程让步（yield），然后移动到另一个CPU，以前返回的值将不再正确。为了避免这个问题，xv6要求调用者禁用中断，并且只有在使用完返回的`struct cpu`后才重新启用。



**myproc()：**

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787901491295-b2f1d69a-2677-43f0-8a57-ce44bf47505d.png)

myproc()会返回当前cpu上运行的进程`struct proc`指针。

在调用mycpu的前后需要关开中断，即使开中断后发生了调度，因为r_tp()已经完成了，myproc的返回值也可以正常使用。



## sleep()和wakeup()

在没有共享资源时，让进程一直在处理机上**轮询（自旋）**是十分浪费cpu资源的。我们希望在没有资源时让进程下处理机，等产生了资源再唤醒进程。使用sleep()和wakeup()，来实现这一流程。

如果处理不好，就会引发**死锁**或者**错过唤醒（Lost Wakeup）**

```c
struct semaphore {
    struct spinlock lock;
    int count;
};

void V(struct semaphore* s) {
    acquire(&s->lock);
    s->count += 1;
    wakeup(s);
    release(&s->lock);
}

void P(struct semaphore* s) {
    acquire(&s->lock);

    while (s->count == 0)
        sleep(s, &s->lock);  // !pay attention，这里的更改避免了错过唤醒
    s->count -= 1;
    release(&s->lock);
}
```

sleep的实现很重要，如果**先睡觉再释放锁会产生死锁。如果先释放锁再睡觉会产生错过唤醒**。必须让睡觉和释放锁原子化。



**sleep()和wakeup源码：**

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787904520254-cfd541f3-6034-4529-af15-ca84e4949c52.png)

sleep需要保证**原子化释放锁并休眠**，他需要传入**等待通道**`chan`(即等待的资源，这里是一个任意地址指针)，还需要传入一个锁`lk`。

首先判断传入的`lk`是否是`p->lock`。如果是，释放该lk，获取p->lock锁；如果不是，则需保证已经存在p->lock，不需要重复加锁了。

之后设置等待通道，更改进程状态，调用`sched`下处理器。当`wakeup`重新将该进程唤醒后，清空等待通道。释放锁，再次获取lk。这保证了持有的锁`lk`调用前后状态一致。



![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787907474029-0de3452c-45af-4231-9ad5-de75ec4a6d5b.png)

wakeup会遍历如果进程为`SLEEPING`状态且他的等待资源已经存在，就改变进程状态为`RUNNABLE`，接着该进程就可以在scheduler中被调度了。调度完成后，他就会在sleep中返回。



## pipe

我们分析下sleep和wakeup在管道上的应用

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787911531141-c2eec6a6-37a9-4ca0-be51-9ba55b3138dd.png)

数据缓冲区是环形的，而读写统计nread/nwrite是持续增加的。pi->nread == pi->nwrite 表示管道空，nwrite==nread+PIPESIZE 表示全部缓冲区。对缓冲区的索引必须使用`buf[nread%PIPESIZE]`。



![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787912303951-4161e528-d9eb-4ea1-8793-fcba218597da.png)

![img](https://cdn.nlark.com/yuque/0/2026/png/63601471/1787912314243-dc33b356-989e-4f1b-8c77-f1d531c6bc5c.png)





## kill，wait和exit

```c
// Wait for a child process to exit and return its pid.
// Return -1 if this process has no children.
int
wait(uint64 addr)
{
  struct proc *np;
  int havekids, pid;
  struct proc *p = myproc();

  // hold p->lock for the whole time to avoid lost
  // wakeups from a child's exit().
  acquire(&p->lock);

  for(;;){
    // Scan through table looking for exited children.
    havekids = 0;
    for(np = proc; np < &proc[NPROC]; np++){
      // this code uses np->parent without holding np->lock.
      // acquiring the lock first would cause a deadlock,
      // since np might be an ancestor, and we already hold p->lock.
      if(np->parent == p){
        // np->parent can't change between the check and the acquire()
        // because only the parent changes it, and we're the parent.
        acquire(&np->lock);
        havekids = 1;
        if(np->state == ZOMBIE){
          // Found one.
          pid = np->pid;
            //拷贝子进程退出状态。
          if(addr != 0 && copyout(p->pagetable, addr, (char *)&np->xstate,
                                  sizeof(np->xstate)) < 0) {
            release(&np->lock);
            release(&p->lock);
            return -1;
          }
          freeproc(np);
          release(&np->lock);
          release(&p->lock);
          return pid;
        }
        release(&np->lock);
      }
    }

    // No point waiting if we don't have any children.
    if(!havekids || p->killed){
      release(&p->lock);
      return -1;
    }
    
    // Wait for a child to exit.
    sleep(p, &p->lock);  //DOC: wait-sleep
  }
}
```

首先获得父进程结构体指针，对父进程加锁防止**唤醒丢失**。遍历进程列表，发现当前进程`np`为父进程的子进程时，对子进程加锁（**提前加锁会导致死锁**）。判断子进程状态，如果为`ZOMBIE`状态，将子进程退出状态`np->xstate`拷贝到`addr`中，再使用`proc`释放子进程资源，一切无误就对父子进程解锁，返回子进程id。

如果`havekids`标志位为0或父进程已经被标记销毁`p->killed`，就对父进程解锁返回-1。

子进程没有执行完毕，就让父进程`sleep`



```c
// Exit the current process.  Does not return.
// An exited process remains in the zombie state
// until its parent calls wait().
void
exit(int status)
{
  struct proc *p = myproc();

  if(p == initproc)
    panic("init exiting");

  // Close all open files.
  for(int fd = 0; fd < NOFILE; fd++){
    if(p->ofile[fd]){
      struct file *f = p->ofile[fd];
      fileclose(f);
      p->ofile[fd] = 0;
    }
  }

  begin_op();
  iput(p->cwd);
  end_op();
  p->cwd = 0;

  // we might re-parent a child to init. we can't be precise about
  // waking up init, since we can't acquire its lock once we've
  // acquired any other proc lock. so wake up init whether that's
  // necessary or not. init may miss this wakeup, but that seems
  // harmless.
  acquire(&initproc->lock);
  wakeup1(initproc);
  release(&initproc->lock);

  // grab a copy of p->parent, to ensure that we unlock the same
  // parent we locked. in case our parent gives us away to init while
  // we're waiting for the parent lock. we may then race with an
  // exiting parent, but the result will be a harmless spurious wakeup
  // to a dead or wrong process; proc structs are never re-allocated
  // as anything else.
  acquire(&p->lock);
  struct proc *original_parent = p->parent;
  release(&p->lock);
  
  // we need the parent's lock in order to wake it up from wait().
  // the parent-then-child rule says we have to lock it first.
  acquire(&original_parent->lock);

  acquire(&p->lock);

  // Give any children to init.
  reparent(p);

  // Parent might be sleeping in wait().
  wakeup1(original_parent);

  p->xstate = status;
  p->state = ZOMBIE;

  release(&original_parent->lock);

  // Jump into the scheduler, never to return.
  sched();
  panic("zombie exit");
}
```

exit()永远不会返回

先获取当前进程结构体指针，判断当前进程是否为初始化进程，如果是则崩溃退出。

关闭所有打开的文件，再通过`iput`释放工作目录。对初始化进程加锁，唤醒一遍初始化进程，因为我们不能在持有其他进程锁的情况下获取初始化锁（**父进程必须先于子进程持有锁**），这里必须先唤醒，即使他会出现**错过唤醒**。

先对本进程加锁，获取该进程的**父进程的拷贝**。对父进程和当前进程加锁，把当前进程的所有子进程转移给初始化进程。之后唤醒父进程一次，设置当前进程退出状态，设置进程状态为`ZOMBIE`。释放父进程锁，当前进程调用`sched`下处理机。



```c
// Pass p's abandoned children to init.
// Caller must hold p->lock.
void
reparent(struct proc *p)
{
  struct proc *pp;

  for(pp = proc; pp < &proc[NPROC]; pp++){
    // this code uses pp->parent without holding pp->lock.
    // acquiring the lock first could cause a deadlock
    // if pp or a child of pp were also in exit()
    // and about to try to lock p.
    if(pp->parent == p){
      // pp->parent can't change between the check and the acquire()
      // because only the parent changes it, and we're the parent.
      acquire(&pp->lock);
      pp->parent = initproc;
      // we should wake up init here, but that would require
      // initproc->lock, which would be a deadlock, since we hold
      // the lock on one of init's children (pp). this is why
      // exit() always wakes init (before acquiring any locks).
      release(&pp->lock);
    }
  }
}
```

reparent在遍历进程表，找到传入进程的所有子进程，将其的父进程改为初始化进程。



```c
// Kill the process with the given pid.
// The victim won't exit until it tries to return
// to user space (see usertrap() in trap.c).
int
kill(int pid)
{
  struct proc *p;

  for(p = proc; p < &proc[NPROC]; p++){
    acquire(&p->lock);
    if(p->pid == pid){
      p->killed = 1;
      if(p->state == SLEEPING){
        // Wake process from sleep().
        p->state = RUNNABLE;
      }
      release(&p->lock);
      return 0;
    }
    release(&p->lock);
  }
  return -1;
}
```

在进程表中找指定的pid，更改其标志位`p->killed = 1`，唤醒处于`SLEEPING`状态的进程。



# 复杂的调度缺陷

xv6使用轮询的调度策略，现代操作系统则更加复杂，如**进程优先级**。复杂的策略可能会导致意外的交互，例如**优先级反转（priority inversion）和航队（convoys）**。当低优先级进程和高优先级进程共享一个锁时，可能会发生优先级反转，当低优先级进程持有该锁时，可能会阻止高优先级进程前进。当许多高优先级进程正在等待一个获得共享锁的低优先级进程时，可能会形成一个长的等待进程航队；一旦航队形成，它可以持续很长时间。为了避免此类问题，在复杂的调度器中需要额外的机制。

睡眠条件受到某种在睡眠过程中原子级释放的锁的保护。在`wakeup`中扫描整个进程列表以查找具有匹配`chan`的进程效率低下。一个更好的解决方案是用一个数据结构替换`sleep`和`wakeup`中的`chan`，该数据结构包含在该结构上休眠的进程列表, 例如Linux的等待队列。