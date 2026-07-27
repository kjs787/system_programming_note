# 做lab前的前置知识

## **调用系统调用**

syscallls函数指针数组（表），输入对应的系统调用号（存储在寄存器a7中)，就能检索对应的系统调用。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781080829545-e39ef8f2-0d19-4d25-b2cb-f4c78c125291.png)

**syscall.c :**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781080460169-0e13c95f-bb27-4c3b-b2c4-0adfccb626ec.png)

  

syscall就是决定执行什么系统调用 ( kernel )

```
void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;  // 从陷阱帧（trapframe）中提取(a7)系统调用号
  if(num > 0 && num < NELEM(syscalls) && syscalls[num]) {
    p->trapframe->a0 = syscalls[num](); // 正常调用，返回值放到a0中，-1就是错误返回
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

## **系统调用参数**

系统调用参数被存储在**寄存器**中，函数`artint`、`artaddr`和`artfd`从**陷阱框架trapframe**中检索第n个**系统调用参数**并以**整数、指针或文件描述符**的形式保存。他们都调用`argraw`来检索相应的保存的用户寄存器

```
static uint64
argraw(int n)
{
  struct proc *p = myproc();
  switch (n) {
  case 0:
    return p->trapframe->a0;
  case 1:
    return p->trapframe->a1;
  case 2:
    return p->trapframe->a2;
  case 3:
    return p->trapframe->a3;
  case 4:
    return p->trapframe->a4;
  case 5:
    return p->trapframe->a5;
  }
  panic("argraw");
  return -1;
}
```

就是通过这种方式读寄存器的

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781081404221-a887c4ae-cb59-456f-9666-8045b5a37ef0.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781081383736-d6298619-8b03-43f9-9e13-21acd1c88113.png)

  

## 安全的数据传递

内核实现了安全地将数据传输到用户提供的地址和从用户提供的地址传输数据的功能。`fetchstr`是一个例子（_**kernel/syscall.c**_:25），他的逐层调用如下:

```
// Fetch the nul-terminated string at addr from the current process.
// Returns length of string, not including nul, or -1 for error.
int
fetchstr(uint64 addr, char *buf, int max)
{
  struct proc *p = myproc();
  int err = copyinstr(p->pagetable, buf, addr, max);
  if(err < 0)
    return err;
  return strlen(buf);
}

```

```
// Copy a null-terminated string from user to kernel.
// Copy bytes to dst from virtual address srcva in a given page table,
// until a '\0', or max.
// Return 0 on success, -1 on error.
int
copyinstr(pagetable_t pagetable, char *dst, uint64 srcva, uint64 max)
{
  uint64 n, va0, pa0;
  int got_null = 0;

  while(got_null == 0 && max > 0){
    va0 = PGROUNDDOWN(srcva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (srcva - va0);
    if(n > max)
      n = max;

    char *p = (char *) (pa0 + (srcva - va0));
    while(n > 0){
      if(*p == '\0'){
        *dst = '\0';
        got_null = 1;
        break;
      } else {
        *dst = *p;
      }
      --n;
      --max;
      p++;
      dst++;
    }

    srcva = va0 + PGSIZE;
  }
  if(got_null){
    return 0;
  } else {
    return -1;
  }
}
```

  

walkaddr调用walk遍历页表, 实现从虚拟地址va0找到真实地址pa0，`walkaddr`（_**kernel/vm.c**_:95）检查用户提供的虚拟地址是否为进程用户地址空间的一部分，因此程序不能欺骗内核读取其他内存。

```
// Look up a virtual address, return the physical address,
// or 0 if not mapped.
// Can only be used to look up user pages.
uint64
walkaddr(pagetable_t pagetable, uint64 va)
{
  pte_t *pte;
  uint64 pa;

  if(va >= MAXVA)
    return 0;

  pte = walk(pagetable, va, 0);
  if(pte == 0)
    return 0;
  if((*pte & PTE_V) == 0)
    return 0;
  if((*pte & PTE_U) == 0)
    return 0;
  pa = PTE2PA(*pte);
  return pa;
}


// Return the address of the PTE in page table pagetable
// that corresponds to virtual address va.  If alloc!=0,
// create any required page-table pages.
//
// The risc-v Sv39 scheme has three levels of page-table
// pages. A page-table page contains 512 64-bit PTEs.
// A 64-bit virtual address is split into five fields:
//   39..63 -- must be zero.
//   30..38 -- 9 bits of level-2 index.
//   21..29 -- 9 bits of level-1 index.
//   12..20 -- 9 bits of level-0 index.
//    0..11 -- 12 bits of byte offset within the page.
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
  if(va >= MAXVA)
    panic("walk");

  for(int level = 2; level > 0; level--) {
    pte_t *pte = &pagetable[PX(level, va)];
    if(*pte & PTE_V) {
      pagetable = (pagetable_t)PTE2PA(*pte);
    } else {
      if(!alloc || (pagetable = (pde_t*)kalloc()) == 0)
        return 0;
      memset(pagetable, 0, PGSIZE);
      *pte = PA2PTE(pagetable) | PTE_V;
    }
  }
  return &pagetable[PX(0, va)];
}
```

  

一个类似的函数`copyout`，将数据从内核复制到用户提供的地址。

```
// Copy from kernel to user.
// Copy len bytes from src to virtual address dstva in a given page table.
// Return 0 on success, -1 on error.
int
copyout(pagetable_t pagetable, uint64 dstva, char *src, uint64 len)
{
  uint64 n, va0, pa0;

  while(len > 0){
    va0 = PGROUNDDOWN(dstva);
    pa0 = walkaddr(pagetable, va0);
    if(pa0 == 0)
      return -1;
    n = PGSIZE - (dstva - va0);
    if(n > len)
      n = len;
    memmove((void *)(pa0 + (dstva - va0)), src, n);

    len -= n;
    src += n;
    dstva = va0 + PGSIZE;
  }
  return 0;
}
```

  

  

# trace

## 修复存根问题

user.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781090775134-7c697c7f-100e-42bd-9150-5c4e206949f2.png)

添加存根->usys.pl

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781090883993-3db94d53-ca76-4504-869a-571ebfa2bfa7.png)

kernel/syscall.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781090974749-db4821c4-bf1b-4a45-9139-a7fe9356d404.png)

## 实现系统调用

kernel/sysproc.c

我犯了一个错误，argaddr（） 第二个参数只能接收 uint64*类型的指针，他用来获取参数

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781095549472-56c7b622-d722-4a86-8e65-e96e73f01dcb.png)

kernel/defs.h 声明 trace 再在 kernel/proc.c实现

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781095476534-b308f1d4-fed2-40b0-b1b7-3cb59124e037.png)

  

kernel/proc.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781093022322-9f99400d-53b3-4c15-9760-4c452f214d77.png)

kernel/proc.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781093100138-ecc177ed-1375-43b2-853f-1697a6f99bcf.png)

kernel/proc.c

修改fork，让子进程能继承系统调用追踪信息

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781093340317-d9c14ab1-1771-4c39-a3fc-80849ebba341.png)

->放大

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781093393537-25fd3aa3-0316-4d59-b73e-315a85909fa4.png)

  

修改kerenl/syscall.c：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781096071600-8df5ea27-cebe-4cbd-a116-aee7153cc663.png)

更新系统调用指针数组（忘了跟新产生错误）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781159604231-b7a30b98-21b7-42b2-8ed8-8e4f2a974aeb.png)

建立系统调用名称数组

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781095899529-3af57091-9180-4fe3-8d32-6b283f0681cf.png)

打印跟踪信息

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781095924843-6d15d569-7247-4812-9a24-0d19ad3edaaa.png)

  

# sysinfo

## 对user的更改

user.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781163775611-c27e372a-dbd8-4fa2-8320-d26fbf3d5b25.png)

usys.pl添加存根

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781164150114-9f7b63c7-37f2-4f5b-84e5-b6a2d7f9b1d7.png)

  

## 对kernel的更改

kernel/syscall.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781164224300-2ed42966-136d-4540-9df2-06553f5e777a.png)

  

defs.h更新sysinfo结构体

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781169297167-f25fe273-c07b-4996-b10b-0baa92779abe.png)

  

kernel/sysfile.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781166903872-f37e0650-5cc4-4b33-a893-551570f3f689.png)

kernel/file.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781170378989-27078538-8ae8-4747-9786-fb1e185a9cbc.png)

在defs.h中更新

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781170470271-5f817cc9-151f-4610-9444-cc005d8dc072.png)

  

kernel/kalloc.c

目的是获取剩余 内存页 * 内存页容量，为sysinfo赋值

该函数应该在file.c中被调用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781168922822-96fc2c38-b790-427d-9223-4d82e4d6d41b.png)

为其在defs.h中更新了

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781170287387-29fc4ea8-dabf-4cae-9736-6b00ab3b6103.png)

  

kernel/proc.c的更改

通过遍历进程数组，获取已存在的进程数

应该在file.c中被调用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781170148485-4726622f-8ed8-4911-92fd-4bbdc3e8697a.png)

在defs.h中更新

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781170302896-51c57471-a4c0-4b26-bab7-1a8f958f321a.png)

  

**仔细看源码，找到解决问题的方式**

比如：

想找剩余内存页的数量，发现kalloc和kfree （kernel/kalloc.c中）他们都在使用freelist结构体，先创建一个 struct run r ,让他对kmem.freelist进行链表遍历，就能求出结果。

想找已使用进程数， 发现proc.c下的函数，都使用proc[nproc]这个数组确认进程状态，我们只要统计p.stat != UNUSED的进程数就行。