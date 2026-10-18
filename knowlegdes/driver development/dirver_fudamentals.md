# 什么是驱动？

驱动是**让硬件动起来**的工具，是沟通底层硬件和上层软件（操作系统）的桥梁。

无操作系统时的驱动就是裸机程序，在单片机开发中常见。有操作系统的驱动就要基于操作系统给出的框架进行编程。

linux将驱动分为**字符设备，网络设备，块设备**三类。其中字符设备指的是只能以串行顺序进行访问的设备，块设备则能以任意类型进行访问，网络设备则面向数据包的接收和发送。

# 驱动基础



## 驱动程序的组成框架

0 头文件 必须包含`<linux/module,h>`和`<linux/init.h>`这两个头文件

1 **模块加载函数**（驱动入口函数） 当加载驱动模块时，内核会执行模块加载函数，完成模块加载函数中的初始化工作。 

2 **模块卸载函数**（驱动出口函数） 当卸载驱动模块时，内核会执行模块卸载函数，完成模块卸载函数中的退出工作。

 3 **模块许可证声明** 许可证声明描述了内核模块的许可权限，如果不声明模块许可，模块在加载的时候，会收 到“内核被污染（`kerneltainted`）”的警告。可接受的内核模块声明许可有“GPL”“GPLv2” 等。 

4 模块参数（可选择） 内核模块参数是模块被加载的时候可以传递给它的值。 

5 模块导出符号（可选择） 内核模块可以导出的符号，导出符号以后其他内核模块可以使用本模块中的变量或函数。 

6 模块作者信息等说明（可选择） 其中模块参数，模块导出符号，模块作者信息是可选的部分。其余部分是必须有的



## 编译并运行驱动程序

### 将驱动程序编译成内核模块

linux提供了两种编译驱动程序的方法：**直接编译进linux内核/编译成linux内核模块**（.ko文件）。

为何需要内核模块：将过多代码编译进内核会导致内核体积变大，但有些驱动很少被使用。将驱动编译成内核模块后，在需要的时候动态加载该模块，不用的时候卸载，这种方法可以减少内核大小，加快启动速度。



我们编写一个`MakeFile`文件，自动进行驱动的内核模块的编译：

```makefile
export ARCH=arm64。
export CROSS_COMPILE=aarch64-linux-gnu
obj-m += helloworld.o
#内核源码所在虚拟机ubuntu的实际路径（绝对路径），并且保证该路径下的内核源码已经编译通过
KDIR :=/home/topeet/Linux/linux_sdk/kernel
PWD?=$(shell pwd)
all:
	make -C $(KDIR) M=$(PWD) modules
clean:
	make -C $(KDIR) M=$(PWD) clean
```



需要在终端设置一些**环境变量**才能进行编译，这些变量存在与linux的顶层`MakeFile`中，写死就不用重复设置了：

```shell
export ARCH=arm64
export CROSS_COMPILE=/home/topeet/Linux/rk356x_linux/prebuilts/gcc/linux-x86/aarch64/gcc-linaro-6.3.1-2017.05-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-


```

![image-20261015144521473](C:\Users\86567\AppData\Roaming\Typora\typora-user-images\image-20261015144521473.png)

输入make，编译成功后，会生成一个**.ko文件**（helloworld.ko）



### 加载驱动程序命令

```shell
#模块加载操作
insmod mod_name.ko 	#直接加载内核模块
modprobe mod_name.ko #将模块所依赖的前置模块一起进行加载，比insmod更高级

#模块卸载
rmmod mod_name.ko

#查看模块信息
lsmod #查看已经加载的模块，cat /proc/modules也可以
modinfo mod_name.ko #查看内核模块信息
```



## make menuconfig图形化配置界面

### 操作图形化配置界面

首先设置平台架构：`export ARCH=arm`

在内核源码顶层文件夹中，输入`make menuconfig`，打开图形化配置界面。使用上下左右键移动，空格键选中。使用 h 查看帮助信息，使用 / 进行搜索



图形化界面一般有一下几类条目：

```text
<> / <*> / <M> 不编译/编译进内核/以模块的方式编译进内核
[] / [*] 不编译/编译进内核
() 接收一个字符串，进行系统的配置
	进入下一级目录
```



### 配置文件的作用

`Kconfig`文件：Kconfig位于arch/xxx/Kconfig目录下，是图形化配置界面的源文件，配置界面中的选项由Kconfig文件的内容决定。

`config`文件和 `.config`文件：都是linux内核配置文件。config位于arch/xxx/configs目录下，是linux系统默认配置文件。`.config`文件位于linux内核源码顶层目录中，**编译内核镜像时会使用`.config`中的配置进行编译。**当使用图形化配置界面时，会更新.config文件，如果不存在，会通过config文件生成一份默认配置的.config文件进行配置。



### Kconfig语法

```text
menu "menu title" 	#menu和endmenu成对出现，是菜单的开始和结束标志

config helloworld #创建一个选项
	bool"helloworld support" #bool/tristate/string是选项类型；bool有两个值y/n，tristate有三个值y/m/n
	default y #默认状态  
	help  
		hello world #帮助信息
comment"The option is for ..."  #选项注释
		
config A
	depends on B #依赖关系，选了B才能选A
config A
	select on B #反向依赖关系，选了选A自动选B

Source"init/Kconfig" #引入init目录下的Kconfig文件
	
endmenu
```



## 将驱动编译进内核



