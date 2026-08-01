  
# Print a page table (easy)

kernel/vm.c

重点关注页表递归的方式，通过标志位判断中间级和最低级页表页。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781402053057-52f1d12c-ab42-4a6a-86eb-3705c6f036f8.png)

  

这种写vmprint接口函数再调用vmlevel_print的方式，简化不必要的参数。

%p用来打印16进制地址的

```
int vmlevel_print(pagetable_t pagetable, int level){

    // 遍历一层页表
    for (int i = 0; i < 512; i++)
    {
      pte_t pte = pagetable[i];
      if ((pte & PTE_V) && (pte & (PTE_R | PTE_W | PTE_X)) == 0)
      {
        uint64 padress = PTE2PA(pte);

        // 打印条目信息
        for (int i = 1; i <= level; i++)
        {
          printf("..");
          if (i < level)
            printf(" ");
        }
        printf("%d: pte %p pa %p\n", i, pte, padress);

        // 向低一层页表递归
        vmlevel_print((pagetable_t)padress, level + 1);
      }
      else if (pte & PTE_V)
      {
        // 页表的最后一级，PTE_V == 1, 但PTE_R | PTE_W | PTE_X不全为0
        uint64 padress = PTE2PA(pte);
        for (int i = 1; i <= level; i++)
        {
          printf("..");
          if (i < level)
            printf(" ");
        }
        printf("%d: pte %p pa %p\n", i, pte, padress);
      }
    }
    return 0;
}

// 打印页表信息
int vmprint(pagetable_t pagetable)
{
  printf("page table %p\n", pagetable);
  vmlevel_print(pagetable, 1);
  return 0;
}
```

  

kernel/defs.h

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781354485774-9ac884db-bbdd-4f7d-ba66-9e8c09c87c7f.png)

kernel/exec.c

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781354460491-d5916950-5322-49fc-b051-59fd7271f4f0.png)


# A kernel page table per process (hard)

kernel/proc.h

为struct proc添加新的项

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781403238528-4a2e1c33-b7a6-4648-bba8-cc3e99f38449.png)

  

kernel/vm.c

用来给用户的内核页表映射的函数，其实就是kvmmap多加了一个参数。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781604758009-339916aa-d38f-4d42-98c0-b515671d620d.png)

  

kernel/vm.c

为进程新建内核页表副本添加映射

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781604783669-a024741d-a81b-4291-8292-19af88a49d84.png)

  

kernel/proc.c

修改allocproc函数

主要做了两个工作：  
1 . 调用proc_kvminit函数，为内核页表申请空间和初始化基本映射

2. 参考procinit函数，为进程内核页表创建内核栈kstack，他只用维护自己进程的内核栈即可

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781604889885-75d566b1-2ee7-4a88-ab16-a2764730de64.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781604906117-d779890a-6f68-4cd8-aabd-66513232f1f1.png)

  

更改schduler函数

让他写SATP寄存器时，存储的物理地址是进程内核页表的地址。

如果没有进程就绪，就还是保持内核页表地址。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781605123615-9e971127-0410-44ee-9583-0bf5b9a76afc.png)

修改freeproc函数，让其释放进程时，会释放进程内核栈和页表内存（叶子物理内存不释放）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781605273886-3ea39212-d1bd-4892-8050-08b9c07c6fc0.png)

借用freewalk的实现方式，判断叶子和页目录表。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781605412211-7777da49-6219-4478-9111-2bc80e4033f8.png)

vm.c

修改kvmpa函数，让其将虚拟地址翻译回物理地址时，**用进程自己的内核栈**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781605515516-d5addd6e-63ec-4a19-9353-f4484a99c856.png)

测试，可以运行

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781435483686-d9f5d4eb-ab99-4fce-8be8-2f9647441cb5.png)

还有对defs.h的更改，就不细说了

  

  

# Simplify `copyin`/`copyinstr`（hard）

vm.c

需要定义该函数来将**用户空间的映射添加到进程的内核页表**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781617140814-b0d2f59a-117c-4031-8b64-d6642f241816.png)

**riscv.h xv6宏的作用**

# xv6 RISC-V 页表相关宏完整作用解析

## 一、页基础尺寸与对齐宏

### 1. `PGSIZE 4096`

- 单页内存字节大小：4KB（2¹²），RISC-V Sv39 标准页面尺寸。
- 所有物理内存、虚拟内存分配、页表映射的最小单位。

### 2. `PGSHIFT 12`

- 页内偏移占用的二进制位数：低 12 位是页内偏移，剩余高位用于三级页表索引。
- 2^PGSHIFT = PGSIZE。

### 3. `PGROUNDUP(sz)`

```
#define PGROUNDUP(sz)  (((sz)+PGSIZE-1) & ~(PGSIZE-1))
```

- 功能：将数值**向上对齐到整页边界**。
- 逻辑：先加 `PGSIZE-1` 进位，再把低 12 位清零；若本身已是页对齐，结果不变。
- 用途：内存扩容、拷贝、分配时，补齐到完整页数，保证按页操作。

### 4. `PGROUNDDOWN(a)`

```
#define PGROUNDDOWN(a) (((a) & ~(PGSIZE-1)))
```

- 功能：将地址**向下对齐到当前页的页首地址**。
- 逻辑：直接抹除低 12 位偏移，舍弃不足一页的部分。
- 用途：取一个虚拟 / 物理地址所在页的起始地址。

---

## 二、页表项权限标志宏（PTE 标志位）

### 5. `PTE_V (1L << 0)` Valid

- 有效位：置 1 代表该页表项存在合法物理页映射；为 0 则该条目无效，访问触发缺页异常。
- 所有合法映射必须开启此位。

### 6. `PTE_R (1L << 1)` Read

- 可读权限：置 1 允许 CPU 读取该页内存。

### 7. `PTE_W (1L << 2)` Write

- 可写权限：置 1 允许 CPU 写入该页；仅置 R 不置 W 则页面只读。

### 8. `PTE_X (1L << 3)` Execute

- 可执行权限：置 1 允许 CPU 从该页取指运行代码；数据页通常关闭 X。

### 9. `PTE_U (1L << 4)` User

- 用户态访问位：

- 置 1：用户态、内核态都可访问（用户空间页面）；
- 置 0：仅内核态可访问（内核栈、内核代码、全局内核数据）；

- 用户程序访问无 PTE_U 的页面会触发保护故障。

---

## 三、物理地址 ↔ 页表项 转换宏

### 10. `PA2PTE(pa)`

```
#define PA2PTE(pa) ((((uint64)pa) >> 12) << 10)
```

- 功能：**物理地址 → 填入 PTE 的物理页号**
- 逻辑：

1. `>>12`：去掉物理地址低 12 位页内偏移，得到物理页编号 PPN；
2. `<<10`：将 PPN 左移 10 位，腾出低 10 位存放 PTE 权限标志（PTE_V/R/W/X_U 共 5 位，剩余预留）。

- 用途：构建页表项时，把物理页地址存入 PTE 高位。

### 11. `PTE2PA(pte)`

```
#define PTE2PA(pte) (((pte) >> 10) << 12)
```

- 功能：**从页表项 PTE 提取完整物理地址**
- 逻辑：

1. `>>10`：抹除低 10 位权限标志，取出 PPN；
2. `<<12`：恢复 12 位页内偏移空位，得到页首物理地址。

- 用途：地址翻译时，从 PTE 拿到映射的物理页基地址。

### 12. `PTE_FLAGS(pte)`

```
#define PTE_FLAGS(pte) ((pte) & 0x3FF)
```

- 功能：提取 PTE 低 10 位全部权限标志（0x3FF = 二进制 10 个 1）。
- 用途：单独判断页面的可读 / 可写 / 用户 / 有效等权限，剥离物理页号部分。

---

## 四、虚拟地址三级页表索引解析宏（Sv39）

Sv39 虚拟地址拆分：`[9位L2][9位L1][9位L0][12位偏移]`，共 3 级页表

### 13. `PXMASK 0x1FF`

- 页表索引掩码：9 位二进制全 1，取值范围 0~511，对应每级页表 512 个 PTE 项。

### 14. `PXSHIFT(Level)`

```
#define PXSHIFT(Level) (PGSHIFT+(9*(level)))
```

- 功能：计算对应页表级别的索引在虚拟地址中的**起始偏移位数**

- Level=0（最低级页表 L0）：12 位偏移之后 → shift=12
- Level=1（L1）：shift=12+9=21
- Level=2（最高级页表 L2）：shift=12+18=30

### 15. `PX(Level, va)`

```
#define PX(Level, va) (((uint64)(va) >> PXSHIFT(Level)) & PXMASK)
```

- 功能：从虚拟地址`va`中，取出指定 Level 对应的 9 位页表索引。
- 用途：`walk()`遍历页表时，逐级算出每一级页表的下标，查找对应 PTE。

---

## 五、虚拟地址边界宏

### 16. `MAXVA`

```
#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))
```

- Sv39 总虚拟地址有效位数：9+9+9+12=39 位；
- 减 1 后代表**合法虚拟地址上限**（最高可用虚拟地址）；
- 设计目的：避开第 39 位符号扩展问题，内核 / 用户空间地址均小于 MAXVA，高于 MAXVA 为非法地址。

---

## 六、类型别名 typedef（配套补充）

1. `typedef uint64 pte_t;` 页表项类型，64 位无符号整数，存储 PPN + 权限标志。
2. `typedef uint64 *pagetable_t;` 页表根指针，指向一级页表数组（数组内每个元素是`pte_t`），单级页表固定 512 个 PTE。

  

对fork的更改

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781616269322-39808c95-8a6e-473c-96e1-4cc02e46346b.png)

  

对exec的更改

exec有个值得注意的点：**内核栈不做更改**，exec依然使用原有的内核栈。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781617420941-4cc5c6ec-5d21-41c1-bd00-93109fb82651.png)

  

对growproc的更改

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781617342280-17eae051-3aba-48d3-a75b-1db93db455c1.png)

  

userinit

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781619210876-5a3f5ee2-527e-4811-9f88-c6dd7ac26fb3.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1781619356651-8ed9d59d-f8fe-4faf-a348-ff29c508213b.png)

  

  

# GDB调试

make CPUS=1 qemu-gdb

新开窗口：gdb-multiarch kernel/kernel

连接qemu调试端口 target remote localhost:26000