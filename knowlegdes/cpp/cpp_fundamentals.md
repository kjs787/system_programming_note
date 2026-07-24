安装g++：sudo apt install g++

# 与c语言的区别

1. 头文件没有*.h的后缀。c++标准库由模板实现，模板必须看到完整的实现
2. 引入命名空间的概念
3. 输入输出方式不同，cout是一个流对象
4. <<在cpp中有输出流运算符的概念

  

## 命名空间

### 创建命名空间

cpp使用命名空间来避免命名冲突问题。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774145840766-7f87ea45-1a63-4a1b-96d5-453efb3dcdf9.png)

命名空间可以定义**变量，函数和命名空间（可以嵌套使用）**

在命名空间中定义的实体先不要缩进

通过（**作用域限定符**） **空间名::实体** 进行访问

### using编译指令

**using namespace 空间名；**

先定义命名空间，再使用该命令，之后访问空间中的实体不必加前缀。

注意：因为不知道命名空间中的实体，可能在后续代码中重复定义，导致命名冲突**（初学时不建议使用）**。

### using声明机制

**using 空间名::实体；**

引入单个实体，推荐使用，稍加注意可避免命名冲突。

### 匿名的命名空间

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774148515677-5200afa2-f274-4d27-af92-f1e1673229a6.png)

通过 **::实体名** 进行访问

一个.c/.cpp/.cc文件就是一个模块

**只能在本模块内部使用**：

- 局部变量
- static修饰的变量/函数
- 匿名命名空间中的实体

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774148576292-754dd46e-eac0-431e-8e13-b16a7f2f597a.png)

**全局变量**可以跨模块调用，请用extern先声明。

可以多用static声明，减少不必要的问题。

  

### 命名空间可以定义多次

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774149785831-f3cfb352-78aa-44fe-8a4a-1f2cfbbdc536.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774149795562-a6d0728a-6a42-48e8-beb2-67ddbc52ccc3.png)

在同一个模块，不同位置可以出现多次。在不同模块中，也可以出现。

但是同一个实体不能被多次定义。

相当于一个容器，每次定义都是往命名空间中放东西。

### 屏闭现象

函数调用变量时，如果多处定义了同名变量，遵循就近原则，按**形参》数据成员》命名空间》匿名命名空间》全局变量》外部变量** 的顺序进行屏蔽。

## const关键字

经关键字修饰的实质是改变其在内存中的形态

修饰**类型 char/int/string/...**

修饰**指针**

### 修饰常量

被const修饰就视为**常量**，**必须初始化，不能再修改**。

当然宏定义也能定义常量

**宏定义与const常量的区别**

**发生时机不同**

1. 宏定义在预处理时
2. const常量在编译时

**类型检查**

1. 宏定义无类型检查，只是简单的文本替换（编译阶段不报错，运行报错，难发现）
2. const是有类型检的，更安全（我们以k开头定义常量名）

### 修饰指针

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774168129568-aa81e5ae-3d84-4887-aaaf-2d6f7734c4d0.png)

  

## new/delete表达式的使用

c语言使用malloc和free

cpp使用new和delete

new的特点是申请空间后进行初始化

delete可以释放堆空间

**malloc的底层实现**

  

### 内存泄漏检测工具

valgrind下载：sudo apt install valgrind

由于命令比较长，起别名，放入vim ~/.bashrc的末尾，执行source .bashrc:

alias memcheck='valgrind --tool=memcheck --leak-check=full'

设置完成后，可用该命令进行检测

memcheck 可执行文件名

valgrind --tool=memcheck --leak-check=full ./可执行文件名

查看内存情况如下：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774177497504-bcf66e2d-cfeb-4d7c-8331-de753cc3d662.png)

### 正确回收操作

int* p = new int[10];

delete [] p;

如果用new申请数组，申请和回收时都要加上中括号。

一旦构造函数调用了堆空间中内存，就一定要重写析构函数将其释放

### new表达式工作步骤

1. 调用**operator new**标准库函数申请未类型的空间

重载：void* operator new(size_t);

2. 在该空间中调用构造函数初始化对象
3. 返回一个相应类型的**指针**

### delete表达式工作步骤

1. 显示调用析构函数（调用析构函数不一定销毁对象，它可能在销毁对象成员申请的堆空间）
2. 再调用operator delete回收本对象所在的空间

  

如果将重载的new和delete放在类之外定义，它的影响范围会变为**全局**，而非之对应某个类的空间分配。

delete一定不能写在析构函数中，会无限调用析构函数。

  

### **只能生成栈对象，不能生成堆对象：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774773427667-a336092c-7fad-4d6f-9f37-8ae445ea918a.png)

将构造函数 void * operator new(size_t sz); 私有化

将析构函数 void operator delete(void * p); 私有化

### **只能生成堆对象，不能生成栈对象：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774773467789-389d149f-248a-4fc3-96b7-25b1bc7f47b2.png)

将析构函数 ~类名(){} 私有化

再写一个成员函数专门释放堆空间

成员函数需要调用 delete this; 而不是直接调用析构函数

  

## 引用

### 定义引用

对变量起别名（此时&叫引用符号）

**类型名 &reference = varable;**

引用是个别名，不能独立存在，必须绑定在一个已经存在的变量。

操作reference就是操作varable本身，他们的地址是一样的。

### 引用可以用作函数的参数

**参数传递的方式：**

1. 值传递
2. 地址传递
3. 引用传递

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774179415409-c61980c8-e7f0-4b4b-96be-206815cb7d09.png)

这种情况根本不会影响a，b的值

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774179751173-708d9c97-437d-45b3-9fff-509482e01ea1.png)

引用的出现是为了代替指针（**但底层还是指针**），避免指针操作的出错，此时a, b的值真正被交换了。

引用一旦初始化就不能更改，它是一个**受限的指针**。

优点：  
1. 没有复制的开销，提高函数执行效率

### 引用作为函数的返回值

返回值形式：

1. 类型的变量（int/struct）在return语句中，直接返回一个变量，进行的是**复制。**
2. 指针类型，做了一个地址传递，效率高。
3. 引用类型，type &/struct type & 代表变量本身，无开销

**注意：**

**不要返回一个局部变量的引用（返回变量的生命周期一定大于函数）**--> 可以通过返回一个**static静态变量**来解决问题

**不要轻易返回一个堆空间的引用，除非有妥善的内存回收机制**

返回不存在的变量是很危险的

**内存泄漏的危害：**

服务器一直运行的，空间越来越少，直到没有足够的空间，直至出现**崩溃**。

程序崩溃后，泄漏的内存会被操作系统回收。

**内存管理对程序员十分重要。**

  

## C++强制转换

c风格强制转换：**type a = (type) expression;**

缺点：安全性差，转换之后合法性？？？

cpp风格强制转换：**static_cast**/**const_cast**/**dynamic_cast**/**reinterpret_cast**

格式：Type a = static_cast<Type>(待转换的量)

### static_cast

常见于指针转换，把void*转换成其它类型的指针

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774254209902-90b297d0-ac97-419f-a517-316b4b4dbe99.png)

### const_cast

去除常量属性

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774253904374-5e4ae774-8b1a-4648-b701-5f864c626d3b.png)

const_cast的滥用可能会导致，**有两个相同地址但不同值的量**出现，const常量的值也没有被改变。

一种说法是，该值被存储于寄存器中，没有被写入地址对应的内存。

无论如何，不要写过于迷惑的写法。

  

### dynamic

只在多态使用，基类与派生类之间的转换。

### reinterpret_cast

不轻易使用，任意类型之间的转换。

## 函数重载

**函数重载的概念：**

函数名相同，参数列表不同。

参数列表：函数参数的**类型，个数，顺序**不同。

C语言没有函数重载

**在cpp中，函数通过返回值和参数列表来区分**，函数名不再有意义。

g++ -c 文件名 //仅编译，生成目标文件

nm 目标文件.o //查看符号表，可以看到真实函数名

  

### 两种不同函数调用方式

cpp使用**函数名改编**实现：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774255276480-4e396314-0347-4da5-8c55-973425db87f3.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774255251559-34cc3498-aee8-4896-8c88-b7e0081665fe.png)

注意：即便cpp有函数重载，使用不当也可能发生**“重载歧义”**

函数参数可以设置默认值，这样可以减少函数重载的数量。默认值参数必须**从右往左进行设置**。

  

  

c语言解决该问题是**用数字命名新接口**：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774255417196-24c8db0a-136a-425f-bdb7-89ee63994b2d.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774255393849-ab8fb2bd-c76c-4c5f-845b-599243d4d39b.png)

  

### c与cpp的混合编程

c语言的函数库已经广为使用，因此cpp必须适应c的调用方式。

我们希望让cpp编译器可以编译c源码。

**extern "C" {}**

"C"中必须是大写的C

包含在该括号中的部分会按c语言的方式进行调用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774260716213-0b19b597-78fe-4b4c-9235-25cf6be8aa8c.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774260738418-3248bbbe-d26d-457f-a79f-2d9538b6aa2b.png)

  

**.h 头文件写法**

既然是否被编译取决于是否含有 extern "C" {}，我们用条件编译撰写头文件，使之支持cpp

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774261086530-19fdf38e-b04b-47ba-beb6-80f4e2ea1e9a.png)

如果是cpp进行编译，只需要先声明 __cplusplus 宏

  

## inline函数

c语言中，如果一些函数比较短，且频繁被调用（如Error_Check函数），为了消除函数调用开销的影响，一般会采用 **宏函数：带参数的宏定义**（发生时机在预处理时，进行简单的字符串替换，容易发生错误）

cpp中也有类似实现，即**inline函数**（时机发生在编译时）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774262528159-750005dc-6888-4056-9c32-bd627e6eacc7.png)

有时只能使用宏函数，达到某些效果

inline函数的定义只能在头文件中，不能在*.cc/*.cpp

inline函数是在编译时进行替换，根本**不会进行连接操作**。

## cpp内存布局

这一块是面试重难点，在写代码时需要明确知道所写代码存储在什么地方。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774400582255-79a3ca2e-9798-4e1b-b885-5938f0569178.png)

  

指针指向的字符串，被存储在文字常量区，不能修改，定义时要加上const

由字符数组存放的字符串被存储在栈区，它不是从文字常量区拷贝而来

函数外声明的 const int 常量 也存储在文字常量区

**栈区由操作系统自动控制，堆区由程序员控制**

像是函数中声明的普通变量，存储在栈上

有static修饰的变量存储在全局静态区

局部静态对象在第一次调用函数时创建，后续再调用该函数，直接使用该静态变量

函数外声明的普通变量，存储在全局静态区

函数本身，存储在程序代码区

如malloc, new之类的分配动态内存的函数，存储在堆区

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774401746913-8cd87c61-578c-405c-bbd3-c40c852fba39.png)

对于数组来说：

数组名 = 首元素地址

& 数组名 = 整个数组 ，对其进行偏移操作会偏移整个数组的长度（类似于指向数组的指针）

## cpp风格字符串

std::string是cpp标准库中提供的一个自定义**类类型**（string构造出的是个对象）

c语言中字符串是 char * 或 char str[] 实现的

string底层有一个空间配置器，会自动完成空间分配和释放：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774751881582-ec910c58-2205-49bb-871e-4e719452ca37.png)

**请多多使用cpp风格字符串std::string!**

### basic_string

basic_string通过构造函数实现string对像的初始化

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774747203220-e6f62046-2c41-44af-b0bb-feedf0905336.png)

### 字符串操作

**cpp和c字符串相互转换：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774748533726-42d12f5b-3956-49d9-8c44-a507ce2c3d0b.png)

使用c_str()让cpp字符串转换为c字符串

**字符串的截取与拼接:**

可以使用 非成员运算符：**+** 对字符串进行拼接操作

可以使用append()函数，进行更灵活的拼接（append会切实改变调用的字符串）

可以使用strsub()函数，对字符串进行左闭右开的截取

具体使用请查阅**cpp参考文档**

**字符串可以像字符数组一样，支持遍历操作**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774751306044-443b644a-bd04-445b-9c45-11906decec4b.png)

**将string当成容器使用auto进行遍历**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774751457261-4ea7ae5f-9977-40d3-815b-cba5372418bb.png)

这种写法叫**增强 for 循环**，auto自动推导

& 是引用符号，不可缺少

：是分隔符，左边是元素，右边是数组

**在尾部增删字符：**  
使用 push_back() 在尾部增加一个字符

使用 pop_back() 在尾部删除一个字符

## cpp输入输出流

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774868954577-2f1bfc45-b29a-4784-a3df-faba15676aa0.png)

  

### 编程语境下的输入输出

从程序（进程）角度看：

数据从其他地方送到程序（内存）叫输入，数据从程序（内存）传送到其他地方叫输出。

### 标准输入输出流

**流的基本单位是字节**

分为三类： 标准输入输出 ， 文件IO，网络IO

c++标准库头文件位置： /usr/include/c++

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774870364087-d5409c95-5cb6-4220-b6a4-2d84448ed6b9.png)

  

### 流的四种状态

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774871708323-980f693e-a75b-4582-b108-0272e4a9ac8e.png)

当输入出现错误，流的状态就会被改变，在程序运行过程中要**时刻关注流的状态**。

**重置流的状态：**

使用cin.clear()函数就能**重置流的状态**，但是不能清空缓冲区。

使用cin.ignore(1024,'\n')清空读缓冲区（最多丢弃1024个字符，直到碰到回车）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774874321980-1d5ee164-80ff-46b2-8c44-0c6b31c134e3.png)

  

## 缓冲区

1. **全缓冲区**：只有缓冲区满时，执行刷新操作
2. **行缓冲区**：碰到换行符，进行刷新
3. **非缓冲区**：不带缓冲区，有多少数据就刷新多数

比如：

cout的缓冲区默认1024个字节，是行缓冲。它在碰到换行符就会清空缓冲区，或者当缓冲区满时清空。

## CPP文件IO

### 文件输入流ifstream

被包含在头文件**<fstream>**中

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775041150555-4cf7d756-7298-4fcf-bf42-1a10cadba29e.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775039422660-10fbe4a0-19ad-4a70-8d28-eb7a546a22fc.png)

上文使用有参构造初始化std::ifstream对象，当然也可以用**open()**方法，但它没有返回值，要结合**.good()**检测流的状态

  

#### 如果想动态调整读取一行字符串的长度

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775039742766-5c7bc2c5-36cf-4ffd-bae9-126aeeb94fe8.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775040037360-5c55382d-508c-4cf5-8bf9-180ef00461a6.png)

可以使用cpp标准库中的std::getline，传入文件流和std::string，其中std::string会动态调整字符串长度，不需要程序员关心。

（**获得的这一行不包括行尾的'\n'**）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775041053808-331eb9b7-3835-4240-838b-544427450b63.png)

可以使用**字符串变长数组**存储文件流中读取的字符串，行号就是数组下标。

使用vector内置行为 shrink_to_fit()，可以将多余容量释放。

### 文件输出流ofstream

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775045075890-9cd8315c-8ae6-4035-a0a1-fb08187f3d5d.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775045087123-88e785b0-0428-4364-8da5-880ae0e5a06f.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775042912711-728b531a-f838-43af-8be1-40b0fbb4a975.png)

和**ifstream**使用相似，默认情况会清空文件流内容再在**文件开头写入（覆盖式写入）**，可以将**第二个参数mode**改成 **std::ios::out | std::ios::app ，这样就能在文件末尾写入。**

**命令：tail 文件名 -F** //可以观察文件末尾写入情况，用作查看日志文件

  

#### 定位文件游标

#### ![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775043315315-9a1826be-85c6-4948-8ed2-b4f8aea05a24.png)

通过获取游标位置和对游标进行偏移操作，指定写入位置。

但在**std::ios::app**模式下，仍然会在**文件末尾写入**。

读文件流，要在**std::ios::beg**开始

### 文件输入输出流fstream

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775045175119-dd5dde1d-d2ff-48ac-adda-0eeffd267b5f.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775045186307-72b9a728-d8f7-4068-a73e-d990529397f2.png)

创建的文件流对象，既可以读，也可以写，需要保证创建流时文件存在。

  

## 字符串IO

  

  

## 友元

将**函数/类**设定成**类的友元**，该类的所有**成员函数，成员变量**就能被自由访问。

```
class B;//类的前项声明，可以之后再实现类，但这样可以让A先看见

class A{
friend print(); //函数可以被声明为友元
friend class B; //类可以被声明为友元
friend B::getInstance(const A &lsh , const A &rsh); //需要对其他类的成员函数进行限定
private:
    int _num;
    std::string str;
};

void print(){
A example;
cout << example._num << endl; //可以访问私有成员了！
}
    
```

函数分为**成员函数和非成员函数**（全局函数，自由函数，普通函数）

**显式调用构造函数**时会创建对象，创建对象时会调用构造函数

为了解决类的位置问题，会使用**类的前项声明**

如果要将其他类的成员函数设为友元，需要**限定( 类名:: ）**。

**重载函数**，需要重新手动设定友元

友元是**单向的，不存在传递性，不能被继承**

**友元的声明不受修饰符限制**

友元的实现必须在类的外面，在类中实现的必须是成员函数

尽量少的使用友元，友元破坏了cpp面向对象中**封装**的特点！

## 运算符重载

  

```
class Complex{};

Complex opertaor+(const Complex &lhs,const Complex &rhs){
    
}

Complex c = a + b; // + 已经被重载了
```

形式：Type operator+运算符 (参数){}

### 运算符重载的规则

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775636793723-8f8d8913-35fa-4e54-9ebe-0bb5aca12d9e.png)

**短路求值特性**：如果逻辑运算符能提前确定结果，后面的表达式就不再执行。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775637075810-24d786c6-0d2e-45f1-9649-68c520d736c7.png)

不能重载的运算符是

**. 成员访问运算符**

**.* 成员指针运算符**

**?. 三目运算符**

**:: 作用域限定符**

**sizeof**

### 运算符重载的三种形式

1. 以普通函数进行重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775698305589-d1533ac0-8fee-4484-97e9-49ec52763481.png)

2. 以成员函数进行重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775698669716-b855423f-1144-4971-b2ec-f7a6c550b028.png)

3. 以友元的形式进行重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775698576316-091b2277-bd3f-42ab-8025-dee3585ff790.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775698648908-f885105d-4b5c-4b11-8e37-d69afae64924.png)

某些重载方式会破坏运算符的形式，要选一个最合理的一种，重载运算符要**符合运算符本身的逻辑**

### 特殊运算符的重载

- **复合赋值运算符**+=重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775699726515-d41ecdfd-223e-42bf-b94f-b95d83e46b6f.png)

- 返回值是**类的引用**可以少调用一次拷贝构造函数
- 对象本身会发生改变的，以成员函数方式进行重载

  

- **前置++**进行重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775803820569-649da3ed-1b7c-4bf5-a0f1-91a5da0d0537.png)

- **后置++**进行重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775804173963-39e23645-b480-43a5-be3c-eee0e5321a5f.png)

int 是占位符，无意义，让函数可以进行重载。

返回类型必须是Complex对象，而非引用，因为生命周期问题。

为什么前置++比后置++运行速度更快？

后置++为了满足先传值后自增的原则，**进行了两次拷贝构造**

前置++可以被取地址**（左值）**，返回对象本身

后置++不能被取地址**（右值）**，返回一个临时对象

  

- **输出流运算符 <<** 重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775809355657-a53f8db8-720e-4601-a753-d279b09af4d7.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775809325263-9fdf9d7c-e9b8-41cf-8668-792efee1e9ea.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775809389613-232c799e-70c7-4041-a194-21b78db13c5a.png)

不能以**成员函数**的方式进行重载。因为<<是二元运算符，但成员函数会**默认加上第三个参数this指针**。你可以删除一个参数const Complex & lhs，但会破坏<<的形式。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775810001583-51c15503-f765-4a31-9c13-5a91146fd4cf.png)

因为流的拷贝构造函数被取消了（**流对象不能被复制，也不能被赋值**），所以重载时必须加上引用符号 &，因为需要修改流对象，所以不能加const。

- **输入流运算符**>>重载

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775814409903-7b1b0e37-5d40-4f2c-b9fd-99634b745d2c.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775814457091-8380c944-bebc-4d8b-a2f8-622be97d0049.png)

输入检测

  

### 重载函数运算符

重载函数调用运算符的类创建的对象被称为**函数对象**（对象像函数一样被使用）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775815838797-ae32ebbf-5812-48ff-99f6-d0a4f4959a08.png)

函数指针类型：可以指向**某种类型**的函数  
**返回值 (*函数名)(参数列表)；**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775870984414-9c5845eb-2cff-4c61-8e97-4a0ed8dcd369.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775871706871-9449056b-3917-49e9-a72f-266313bb6e43.png)

以上两种写法并无区别

### 重载下标访问运算符

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775875392577-d515f1f3-a225-4b00-9fba-b8e53622ce30.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775875411331-2e93efc3-fb08-4336-942c-d2cf8c96b5fd.png)

  

### 重载箭头和解引用运算符

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775963026707-956d5c28-0f35-48fc-9b51-8ef659f491b8.png)

箭头

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775963294083-e29015b1-ee1b-43c9-95a8-56de105b4197.png)

解引用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775964241557-5609ae14-6824-46e1-9f37-10d2c253792f.png)

对象可以使用箭头去返回指向某一类型的指针；

对对象解引用可以返回成员指向的某一对象；

### 一些重载的建议

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775964564342-da472778-e1bd-4114-b43a-e0c9a09ef59b.png)

## 隐式转换

### **自定义类型转换为其他类型**

需要构造函数做支撑

例如将 Complex类型隐式转换成 Point

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775966027378-41e33aec-8c7f-473a-93fe-be5a22d6d5d2.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775966013925-a5771eda-5ffc-410f-b4f1-28320aa7ce03.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775966045070-83b51fa1-d0b3-4c77-94f5-5ffa2d637b47.png)

### **其他类型转换为自定义类型**

不能直接转换，需要依靠**类型转换函数**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776153540641-fca219ba-fe44-4680-aaf9-21f676fc63eb.png)

  

有些特殊情况类型转换函数会被调用，而存在两种不同的类型转换函数就会报错（不知道用那个转换）

## 写时拷贝cow

当进行读操作是只需使用浅拷贝，进行写操作时进行深拷贝。

  

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776308614923-b9dc95c2-0c74-439a-89c2-553d87b930b0.png)

### 引用计数

写时复制是通过**浅拷贝+引用计数（refcount）**的方式

引用计数需要存放在**堆区**，且最好**紧跟在数据之前**，保证其不会受到数据偏移的影响

**引用计数的实现：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776322063754-9beee641-40fe-42e2-b0b3-aff9055ff395.png)

基本结构是**引用计数+字符串**，每次操作指针总是**指向真正存放数据的位置**。

一旦发生浅拷贝，就将引用计数加一，发生一些字符串覆盖或删除，就将引用计数减一。总之，一直在堆空间中维持这一个区域，直到没有任何指针指向他，再将其删除。

写时复制代码

```
#include<iostream>
#include<string.h>
using std::cout;
using std::endl;

class String{    
public:
    //构造函数
    String()
    :_pstr(new char[5]() + 4)
    {
        cout << "String()" << endl;
        *(int *)(_pstr-4) = 1;
    };

    String(const char *str)
    : _pstr(new char[strlen(str) + 5] + 4) 
    {
        cout << "String(const char* str)" << endl;
        strcpy(_pstr, str);
        *(int *)(_pstr-4) = 1;
    };

    //拷贝构造函数
    String(const String & str)
    : _pstr(str._pstr)
    {
        cout << "String(const String & str)" << endl;
        ++*(int *)(_pstr - 4);
    };
    ~String()
    {
        cout << "~String()" << endl;
        if (_pstr)
        {
            --*(int *)(_pstr - 4);
            if (*(int *)(_pstr - 4) == 0)
            {
                delete[] (_pstr - 4);
                _pstr = nullptr;
            }
        }
    };
    const char* c_str()const{
        return _pstr;
    }
    int getRefCount(){
        return *(int*)(_pstr-4);
    }
    size_t size()
    {
        return strlen(_pstr);
    }

    class CharPro
    {
    public:
        CharPro(String &c, int index)
        : _index(index), 
         _cp(c)
        {
        }

        CharPro* getThis(){
            return this;
        }

        CharPro &operator=(const CharPro &rhs)
        {
            cout << "CharPro &operator=(const CharPro &rhs) is called!" << endl;
            if (this->_cp._pstr[_index] == rhs._cp._pstr[rhs._index])
            {
                return *this;
            }
            else
            {
                char *tmp = new char[rhs._cp.size() + 5] + 4;
                strcpy(tmp, rhs._cp._pstr);
                --*(int *)(_cp._pstr - 4);
                if (*(int *)(_cp._pstr - 4) == 0)
                {
                    delete[] (_cp._pstr - 4);
                    _cp._pstr = nullptr;
                }
                _cp._pstr = tmp;
            }
        }
         friend std::ostream &operator<<(std::ostream &os, String::CharPro &c);
    private:
        String &_cp;
        int _index;
    };

    CharPro& operator[](int index)
    {
        cout << "char &operator[](size_t index) is called!" << endl;
        if (index < size())
        {
            _mychar = CharPro(*this, index).getThis();
            return *_mychar;
        }
        else
        {
            cout << "Segement ERROR!" << endl;
            exit(-1);
            return *_mychar;
        }
    }

    String &operator=(const String &rhs)
    {
        if (this != &rhs)
        {
            --*(int *)(_pstr - 4);
            if (*(int *)(_pstr - 4) == 0)
            {
                delete[] (_pstr - 4);
            }
            _pstr = rhs._pstr;
            ++*(int *)(_pstr - 4);
        }
        return *this;
    }

    friend std::ostream &operator<<(std::ostream &os, const String &str);

private:
    char *_pstr;
    CharPro* _mychar;
};

std::ostream &operator<<(std::ostream &os, const String &str){
    if(str.c_str()){
        os << str.c_str();
    }
    return os;
}

std::ostream &operator<<(std::ostream &os, String::CharPro &c){
    int index = c._index;
    // printf("%c\n", c._cp.c_str()[index]); 正常运行
    // printf("%c\n", c._cp.c_str()[c._index]); 段错误
    os << c._cp.c_str()[index];
    return os;
}

void test()
{
    String s1("hello");
    #if 0
    String s2 = s1;
    printf("s1_adr = %p\ns1_refCount = %d\n",s1.c_str(),s1.getRefCount());
    printf("s2_adr = %p\ns2_refCount = %d\n", s2.c_str(),s2.getRefCount());
    String s3("world");
    printf("s3_adr = %p\ns3_refCount = %d\n", s3.c_str(), s3.getRefCount());
    s3 = s1;
    printf("s3_adr = %p\ns3_refCount = %d\n", s3.c_str(), s3.getRefCount());
    #endif

    cout << s1[2] << endl;
    //s1[1] = '0';
    //cout << s1[1] << endl;
}

int main()
{
    test();
    return 0;
}
```

注意：**不确定的求值顺序**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776396593775-2c9a421f-6cdb-4fc1-bc15-e39edb0afe01.png)

  

## 短字符串优化sso

比如sprintf("str");

当字符串str大小大于15个字节，将其存储在堆上（返回指针），小于则存储在栈上。

  

## 移动语义

可以使用右值引用，传递右值： **String && ref** = String("helloi");

（**右值引用是左值**，可取地址，作为函数参数，但也**可能是右值**，作为函数返回值）

移动语义的函数优先于拷贝语义的函数，且移动语义的函数中尽量保持无拷贝语义的操作

重写类的 （移动）拷贝构造函数和（移动）赋值运算符函数 ，将其参数列表改为右值引用：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776839406824-560a5731-de13-4ca0-98f8-d3f397bc9eba.png)

对堆区数据仅作浅拷贝，修改右值对象的指针为nullptr，让其销毁时不释放堆空间。

**std::move(对象) 可以把左值转换为右值**，实际做了一步强制转换

- c风格字符串，是个右值（存放在**文字常量区**）
- 临时对象会**短暂**存放于栈上
- 加const修饰的全局变量存放在文字常量区
- 函数内加const修饰的变量存放在栈上

## 资源管理---RALL

由c++之父提出，**利用栈对象生命周期管理程序资源**（包括内存，文件句柄，锁等）

使用RALL，在资源获取时构建对象，对象生存期间，资源一直有效。对象析构资源释放。

关键：保证**资源获取和释放顺序严格相反**

简单解决c语言资源释放问题

```
#include<iostream>
#include<stdio.h>
using std::string;

class FileSave{
public:
    FileSave(const string &name, const string &mode)
    : _file(fopen(name.c_str(),mode.c_str()))
    {
        std::cout << "File has been loaded successfully!" << std::endl;
    }

    ~FileSave(){
        if(_file){
            int ret = fclose(_file);
            if(!ret){
                std::cout << "File has been closeed successfully!" << std::endl;
            }
        }
    }

    void write(const string & str){
        if(_file){
            int ret = fwrite(str.c_str(),1,str.size(),_file);
            if(ret){
                std::cout << "File has been writed successfully!" << std::endl;
            }
        }
    }
private:
    FILE *_file = nullptr; 
};

int main(){
    FileSave file("test.txt","a+");
    file.write("hello world!\n");

    return 0;
}
```

  

### **RALL的特点**

- 在构造函数中初始化资源
- 在析构函数中释放资源
- 提供若干访问资源的方法
- 不允许复制与赋值（可以将拷贝构造函数和赋值运算符函数进行私有化、delete）

## 智能指针

### auto_ptr

auto_ptr可以寄存一个指针，但在复制时会发生所有权的交换（执行release），存在缺陷

  

### unique_ptr

**独享所有权**的智能指针（强引用），他提供了一个严格语义上的所有权

- 拥有所指向的对象
- 不能复制或赋值
- 保存指向对象的指针，当他本身被删除释放的时候，会用**删除器**删除它指向的对象
- 具有**移动语义**（ std : : move ( ) ），可以作为**容器元素**

**unique_ptr的Deleter删除器：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776944404385-587cfedc-4d55-44b8-bf56-2c12ff00ee91.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776944561503-8f59e433-2822-40c6-bb45-a745af3a3579.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776944495401-2553fb4e-4474-486b-86cb-fdc7cc4588c0.png)

只要符合一定规则，cpp的改写相当灵活

这个示例是针对文件指针改写删除器

删除器会在智能指针销毁时自动调用，**容易产生double free错误**，不要使用两个不同的unique_ptr对象托管同一片堆空间

### shared_ptr

使用**引用计数**的智能指针（弱引用），共享对象所有权

- 引入shared_count表示有多少个共享指针指向改内存块
- 释放指针内存块会将shared_count减一，减到0就释放改内存块
- 拷贝构造与赋值运算符只提供一般意义的复制，将shared_count加一
- 具有移动语义，可以所为容器的元素

缺点：循环引用问题，俩个智能指针一旦循环引用，即使程序结束，他们还会保有值为1的引用计数，堆空间就不会释放，产生内存泄漏。

**shared_ptr的删除器**：  
![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776946040696-239992c1-1674-4fd8-96c9-40870a5c51d7.png)

删除器对象可以作为shared_ptr构造函数的第二个参数

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776948000387-a6a66cb1-1d1b-4f2e-9db8-5dd1da7086e4.png)

如果想托管一个推空间，**请用智能指针相互赋值**，不要用裸指针进行赋值，会产生double free错误

在用函数返回值进行赋值的情况最容易出错，希望返回一个智能指针且不会进行多余构造，使用**enable_share_from_this类**的成员函数 **share_from_this(指针地址)** ，返回智能指针

需要继承std::enable_share_from_this类

### weak_ptr

一般与shared_ptr联合使用，托管一个堆空间

weak_ptr不会让引用计数增加，解决了循环引用的问题

使用 .expired() 方法，返回一个bool值，判断weak_ptr托管的资源是否还存在

  

  

## 模板

注意：这里仅简单提一下模板，目的是为STL服务。泛型编程知识冗杂，需要专门学习。

C++ 模板是**泛型编程**的核心工具，它允许你编写**与类型无关**的通用代码，编译器会在使用时根据传入的具体类型自动生成对应代码，避免重复编写逻辑相同、仅类型不同的代码。

```
template <模板参数列表>
函数返回类型 函数名（函数参数列表）
{
}

template<typename T1, typename T2> //新版一般这样写
template<class T1, class T2> //class和typename含义完全一样
```

### 函数模板

```
template<typename T>
T add(T x, T y)  //x，y的类型在编译阶段就推导完毕
{
    return x + y;
}
```

当存在普通函数的**函数形式与函数模板相似**时：

- 优先调用普通函数
- 普通函数可与函数模板进行重载

隐式实例化：靠编译器推导出来类型

显式实例化：主动声明类型 add<double>(x,y);

模板的**声明和实现**必须在同一文件中（如果一定要分开声明，需要在头文件中 include"模板.cc"）

### 类模板

```
template<typename T>
class Stack
{
private:
    T * _data;
}
```

**成员函数模板不能是虚函数**，发生时机就不对，虚函数在运行时确定，模板的类型推导在编译时确定。

虚表在编译阶段就要确定了，函数模板会使虚函数数量不确定

（类模板的成员函数可以是虚函数，区分开）

### 模板的特化

对一些复杂类型，编译器不能自动推导让其功能成立，此时需要模板的特化

**如果一个模板参数没有被函数参数使用，也没有显式指定，编译器就无法推导它 → 直接报错**

**模板的全特化：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777175039624-2d026414-4398-4cbc-9385-f6a107939934.png)

在函数前加 **template<>** 表示这是一个模板特化的函数

如果模板中所有推导类型都被指定了，叫做模板的全特化，反之叫偏特化

  

### 可变模板参数

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777175283300-de0de347-81dc-4c89-8316-f331d674a4f5.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777175377420-13c8ff87-3c1e-49e0-89df-a15ff1a7e57c.png)

int 就是类型参数，20 是非类型参数

且**模板参数可以是typename类型，都可以赋初始值（设置默认类型）**

```
template<typename ...T>  //模板参数包
void func(T ...t)  //函数参数包
{

}
```

**利用递归打印**

```
//必须为递归写出口(解包到最后会变成空包)
void show(){
    cout << endl;
}
//通过递归和解包的方式输出函数参数包中的内容
template<typename T ,typename...Args>  //...参数 -> 打包
void show(T t, Args ...args){	//参数... -> 解包
    cout << t << endl;
    show(args...);
    
}
```

### 模板的嵌套

模板之间可以互相嵌套，模板的参数可以是模板

```
//模板之间互相嵌套
template<typename T>
class example{

    template<typename K>
    class base{

        template<typename T,typename ...Args>
        K func(T t, Args ...args){
            cout << t << endl;
            return t;
        }
    };
};

//模板参数的嵌套
template<template<class T1> class T2, class T3, int num> 
```