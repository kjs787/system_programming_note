  

# RISC-V assembly (easy)

## 了解RV汇编代码

请参考这个博客：[https://blog.csdn.net/weixin_43083491/article/details/146122711?spm=1001.2014.3001.5502](https://blog.csdn.net/weixin_43083491/article/details/146122711?spm=1001.2014.3001.5502)

建议查阅这个网站：[https://ai-embedded.com/risc-v/riscv-isa-manual/](https://ai-embedded.com/risc-v/riscv-isa-manual/)

汇编代码分析网站：[https://godbolt.org/](https://godbolt.org/)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781953223471-b4ef66cc-5ce0-454a-8fc6-418c52e1d090.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781953244100-05ba88b3-9816-4cd2-90c2-7a84f820fb3f.png)

answers-traps.txt

1. a2寄存器保存13，a0寄存器保存函数参数x
2. 因为编译器优化，没出现f,g函数跳转指令，在24行他直接给结果算出来了，保存到a2寄存器
3. printf函数位于34行，halr 1536(ra)
4. ra被写入0x30，每条指令左侧的序号就是指令所在的虚拟地址。rv指令固定4字节宽
5. 打印结果是0xe110 World，如果是大端存储，i的01串应该倒过来。
6. 结果是x=3 y=2040347648，因为printf的第三个参数没有设置，他随机打印出一个寄存器的值

s0指向栈的底部，sp指向栈顶

# Backtrace(moderate)

  

kernel/riscv.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781957543496-b076cb76-b3ad-4ec0-9f74-8fce5bcae965.png)

  

kernel/defs.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781957612311-99912d5e-70bf-45c8-8762-c65b82d92a21.png)

  

kernel/sysproc.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781957986959-350a6f59-cd44-43be-bba1-c86855265e49.png)

  

kernel/printf.c

重点是我们怎么看待fp，他是栈底指针，在寄存器中以双字存储，所以我们从r_fp读出来也是uint64类型。

我们实际使用并进行移位操作时，应该将其强转成uint64 *类型，移动时注意每次移动8字节。

已经知道内核会为每个进程分配一页大小栈空间，且栈向下伸展，只要比较fp到栈底小（未达到最底层栈）就继续循环。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781961329499-fb4f4a0e-1db7-4d75-b8c6-3ed6d3aefbd1.png)

printf.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781961677162-cc32109f-d8cd-4106-ac57-f3f2ea4c04b2.png)

  

  

# Alarm(Hard)

## test0

MakeFile

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781962103493-8850de7e-0753-434d-a58b-556e554ea419.png)

  

user/user.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782002946543-1034d1a6-0466-42b1-950f-aa20c6a69784.png)

  

user/usys.pl

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781962419831-e13b1f42-0eea-4a5f-932b-143293f44e63.png)

  

kernel/syscall.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781962697350-31e71577-3ab9-4900-bf14-8b134c5c63af.png)

  

kernel/syscall.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781962940058-83188378-c843-4762-8810-e2bb698d3ab5.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781962951640-2147c0dd-ca46-421a-8f7c-90a25eb14151.png)

  

sysfile.c

接收传入的参数

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782008116623-bf274d47-e310-4686-9082-869fc732fc62.png)

  

defs.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782004032319-c2549462-92d9-48ed-b29b-a74efec55155.png)

  

kernel/proc.h

新增三个字段

我将这三个字段都改成 uint类型，默认他们大于0 （用户可能传入一个负数时间间隔或异常地址，但那明显是错误的，我们在sysfile.c/SYS_sigalarm中进行了判断）

当ticks滴答数 == alarm_interval报警间隔时，调用一遍处理函数fn

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782005385162-cd263fd1-8118-45a6-a0f3-6e9562123545.png)

  

proc.c/alloc

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782004009005-28cf98d8-8b71-4158-b360-c126ac7470d3.png)

  

kernel/proc.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782004397025-26e3b725-22aa-4932-856d-91c893962627.png)

  

/kernel/trap.c/usertrap

我们更改了时钟中断处理代码，当符合条件，进程陷阱帧帧中的epc就会被替换为我们指定的报警函数。

他会在usertrapreturn中被调用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782007862992-debd427a-7e14-490d-9594-c324d713ef52.png)

test0测试通过

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782008006758-422080da-db1d-407a-980a-8b4cdaee4ccc.png)

  

  

## test1/2

kernel/proc.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782013553026-f5899429-2553-4be8-ade1-0df57d1a26b3.png)

proc.c/allocproc

**用kalloc申请一片物理页存储警报帧**，并返回其虚拟地址，因为内核恒等映射，内核虚拟地址 == 物理地址，所以可以直接访问。

别忘了在freeproc中释放该物理页。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782013589134-60043bec-1e10-4efd-9606-dead6be8aa37.png)

freeproc

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782013633419-7b7c472e-3c20-42af-8946-b65564bff205.png)

  

trap.c/usertrap

重点是用memove进行完整的物理地址拷贝，存储和保存所有寄存器状态。

ret_flag作为是否继续进行报警处理的信号。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782014172842-b2818632-6af3-405f-8ae5-b9dac1134492.png)

proc.c/sigreturn

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782014210784-717ae8d5-90f9-4af4-92ff-bacd00903610.png)

测试通过

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1782013495824-aa437bf9-fd4d-4b31-a33a-98d3c41a0bc6.png)

  

对页表还是不熟！