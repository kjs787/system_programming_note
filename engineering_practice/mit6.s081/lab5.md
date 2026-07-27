  

# Eliminate allocation from sbrk() (easy)

修改sysproc.c/sys_sbrk

# ![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782024401179-1995f8ad-7d70-4168-ba5c-e2aa88c76d8e.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782024386652-85b4a61a-3d07-478e-aee4-4f0a21de460b.png)

  

  

# Lazy allocation (moderate)

对kernel/trap.c更改

参考vm.c/uvmalloc，申请一页物理页为出错的虚拟页映射。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782027260178-3d9029c2-6acb-4ce3-ab13-d86928c2ca1e.png)

  

对kernel/vm.c更改，允许懒分配空间

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782026253115-d65c863e-f885-4193-8629-aeb0746889bf.png)

  

没有出现任何的错误？

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782027399533-c32ea0a8-36e6-4d63-b03a-e455d7ee9b94.png)

  

  

# Lazytests and Usertests (moderate)

在做这个实验时要想着，所有的申请的堆空间都可能不存在。

walk可能检索不到正确的pte；访问pte的PTE_V可能是0，这些都可能导致内核直接退出，不给你懒分配的机会。

所以有些本应该panic的地方要更改成continue

  

1. 向sbrk传递负数的情况

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782039565458-15e0ffe3-54ab-4d99-9626-6913b97584b6.png)

  

2. 如果页错误出现地址比懒分配地址更高，杀死进程

限制在堆区

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782039624656-a5ce8320-6433-4309-a895-77f069863434.png)

  

3. fork的正确拷贝

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782031462818-4074d8ae-2f9c-44e9-818f-8e38f804c009.png)

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782031596621-6a898f72-b839-48d5-831e-f5bc9c692dc7.png)

4. 处理这种情形：进程从`sbrk()`向系统调用（如`read`或`write`）传递有效地址，但尚未分配该地址的内存。

如果是系统调用，就是在进行r_scause() == 13 || r_scause() ==15 判断之前，就被系统调用的r_scause()截获了。如果再传递未分配的地址，就不会进行懒分配了，系统调用就失败了。于是需要追踪什么时候，系统调用接收到了该地址，再显式为其分配内存。

可以看到他调用了一遍argaddr()获取分配的地址

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782032155177-7099d673-a4b9-4dc6-ba08-218e67c85223.png)

我们为其分配物理地址和映射关系

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782039806965-c2ce6acb-d531-4e67-bb51-0af55ba39a21.png)

  

  

  

5. 正确处理内存不足：如果在页面错误处理程序中执行`kalloc()`失败，则终止当前进程。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782029036418-3adcfa25-5267-4536-b15a-b51f34c30666.png)

  

6. 处理用户栈下面的无效页面上发生的错误。

将分配地址限制在堆区

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782039619197-2589a69a-5c1b-43c2-8ac4-310316ea87c2.png)

  

如你所见lazytests可以全部通过。

但usertests第一个就挂了，可能参考答案有问题。

但我今天是注意力涣散了，可能等暑假继续这个项目。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782041151713-72933b8d-45a8-42a2-a6f6-e969b2948c6a.png)