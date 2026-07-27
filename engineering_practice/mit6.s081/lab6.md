# Implement copy-on write (hard)

riscv.h

先声明cow标志位

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782134385742-41f28afa-7aa3-40d5-a603-a23c16e1dbdc.png)

  

kernel/vm.c

对uvmcopy的更改不去申请实际物理空间，仅对pte性质做更改，让其不可读并标记为cow页面。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782292141414-535830f4-dab4-4597-b902-57629f2e3911.png)

  

kalloc.c

创建引用结构体，该结构体中包括 **自旋锁spinlock**和**引用计数数组**

因为引用计数是多个进程共享的资源，应该用锁保护起来。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782292826498-35692d66-f7e8-4eed-acc2-640b37d74f33.png)

  

需要在kinit中初始化锁

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782292962366-a44511b0-b704-455b-bcd8-c87b801128d4.png)

需要修改一下freerange，内核启动时会回收所有物理地址资源，不能让引用计数减为负数

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782293312187-eaa494bb-0e91-4f10-b865-8bbc30218d16.png)

kalloc为申请每一块物理内存空间，并将其引用计数设置为1

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782293007300-800b73e7-597e-48a9-8753-ac0efcd2a04c.png)

  

分别是增加引用计数和获取某块物理内存的引用计数的函数，

都应该声明到到defs.h中

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782293412264-1679ecc2-3bd3-4a44-a575-9f3ddb2b88ab.png)

  

vm.c

fork会调用该函数进行内存分配

我们不需要申请物理内存，仅让在进程用户页表映射到父进程物理内存就行。

如果该内存是可写的，我们将其设置为只读和cow标志位

最后增加该内存引用计数

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782293576029-987265fa-f4a7-4294-86e3-7a08b0a1ed2b.png)

  

trap.c的处理cow逻辑

我之前犯了很严重的错误，认为cow和lazy allocation一样，需要在（(which_dev = devintr() )!= 0）做判断，但其实不是这样，cow引起的**异常**和lazy allocation引起的**设备中断**不是一类。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782292749064-7f3d7268-a95f-49bf-a21e-d4de9b04608b.png)

先判断是否为cow页

对引用计数为一的内存，我们仅更改其可写，取消cow标志即可。

使用memmove将数据拷贝到申请的物理内存页上，注意到物理内存地址好像都用char *表示（被调用函数的参数类型是void *,在函数里将其转换为uint64）

在调用mmapage前，先取消pte的PTE_V标志位，防止报重复映射的错误。

映射时pte要加上可读PTE_W和取消写时复制PTE_RSW标志位

因为指向另一块物理页，将原物理页引用计数减一

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782292765933-31369f8a-d840-47e1-a54f-7829f815d239.png)

  

vm.c

copyout同样有将数据写入到其他物理地址的动作，和usertrap一样，要更改成cow的逻辑。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782298355330-df32e097-fe7e-4530-ad6d-baf92192b01a.png)

  

题解做的更好，把很长的逻辑封装成函数，便于调试和调用。

测试通过：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782298454980-16b5cee5-54dd-4446-b295-bb13ec2fa856.png)