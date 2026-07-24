面向对象的精髓是：**消息传递**

# 类与对象

## 面向对象的特征

抽象、封装、继承、多态

**抽象：**

现实世界：通过抽象思维开始提出一类对象**共同的特征（属性），共同的行为**。

程序世界：编程从类开始，到对象，对应**数据成员，成员函数。**

## 类的定义

```
class 类名{
//成员函数
public:
    void pay();
//数据成员
private:
    int _money;

};
```

类中有 **public/protected/private** 进行修饰。

暂时不关心 protected

在花括号中叫**类之内**，花括号之外叫**类之外**

不希望外部访问的，使用private修饰

希望外部调用的，使用public修饰

请将public成员放在类前面，是该类向外提供的接口

将private放在后面，不希望被看到

只要是类中的数据成员，名字都要有_ 开头

使用 **类名 对象名** 创建一个对象

面向对象编程，**没有函数，只有对象，只是对象在做各种各样的行为。**

  

## 类的声明

```
class computer{
public:
    void print_length();
private:
    int _length
};

 void computer::print_length(){
    std::cout << _length << std::endl;     
}
```

你可以只在类中进行函数声明，在类外进行函数实现。

在类中只要有一个成员函数没有被实现，就只能叫类的声明，不是类的实现。

只要有在类中声明但未在类外实现的函数，编译就会报错。

**class与struct的区别：**

在cpp中，struct被扩展了，在不需要设置访问权限（struct中默认public）的情况下，大胆使用struct，他和类用法几乎一样。

## 对象的创建

```
class computer{
public:
//默认构造函数
computer(){
}

//cpp11之后
//告诉编译器生成一个默认无参构造函数
Point() = default;

//有参构造函数
computer(float price, const char* brand){
    _price = price;
    strcpy(_brand,brand);
}

private:
float _price;
char _brand[20];
}


int main(){
    computer a; //调用默认构造函数
    computer b(30.0,"mind hacks"); //调用有参构造函数
    return 0;
}
```

cpp中，**对象创建时**会调用一个特殊的成员函数，即**构造函数。**

构造函数作用就是**初始化成员变量**。

仔细观察他其实是一种**赋值操作**

构造函数会遵循以下三条原则：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774426910057-86b215b9-ab67-4920-815b-f1bbcf64c1c6.png)

注意：有时没重写无参构造函数，还能正常使用。此时它是一个**函数声明**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1775961247519-64c13750-bdf5-46cd-a14b-a5d821ef0120.png)

## 构造列表

首先cpp初始化可以通过：**变量名(值)** 的形式进行，他和 **变量名 = 值** 没有区别

```
class computer{
computer(){
}

computer(int cnt,int price,char * brand)
: _price(price)
, _cnt = 100
{
    strcpy(_brand,brand);

}


private:
    int _cnt;
    int _price;
    char _brand[20]; //不能直接使用 _brand(brand) ,这变相改变数组名（指针常量）指向的地址
};
```

通过构造列表，实现真正意义上的**类成员初始化**。但是**初始化顺序仅和成员声明顺序有关**，和构造列表中的顺序无关。

变量定义时尽量进行初始化，特别是指针类型。

## 对象的销毁

对象在销毁时，会调用**析构函数**

作用是清理类对象成员申请的资源（堆空间）

析构函数默认情况下，系统会自动提供一个

形式：

没有返回值，即使void也没有

没有参数

函数名与类名相同，在类名之前需要加上一个 ~

**~类名(){}**

析构函数只有一个

**析构函数的调用**

1. 当栈中定义对象被销毁时，他会自动调用。
2. 全局对象，静态对象，局部静态对象在main函数退出时，会自动调用
3. 堆对象，在执行delete时，会自动调用。

  

## 对象的复制（拷贝）

### 拷贝构造函数调用的时机

1. 用一个已经存在的对象初始化另一个对象。
2. 当函数参数是对象，**实参初始化形参**。
3. 当函数返回值是对象，执行return语句

去除编译优化：g++ file.cc -fno-elide-constructors

调用**复制构造函数**，默认情况下，系统会给一个

### 自定义拷贝构造函数

当栈对象申请了堆空间时，默认拷贝函数会进行一次**浅拷贝（值传递和地址拷贝）**，但当对象销毁调用析构函数时，free空间的操作进行了两次，出现**double free**的错误

此时需要自定义拷贝构造函数：

固定形式：**类名(const 类名 & rhs)**  
![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774440761391-e42ed73e-50af-4c0c-9ebc-2a3cf903faf3.png)

先申请空间，再进行拷贝（深拷贝）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774442186927-adde432b-77c2-4c6f-acb0-a7637f8e5e49.png)

### 是否可以去掉引用符号&？

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774442815037-607cbc69-30b3-4141-bb27-636ec26339d8.png)

  

### 可以去掉const吗？

首先，不能被取地址的称为 **右值**，可以被取地址的称为 **左值 。**左值已经被放入了内存区域，右值称为临时对象（拷贝的可能是临时对象）

**常引用可以绑定到右值，非const左值引用无法绑定右值**

const引用是一个**万能引用**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774865356730-206891cf-4bfc-492a-afe2-3b535a08b6b3.png)

## this指针

为什么调用类的成员函数能精准定义到某一个对象的数据？

  

1. 每一个成员函数都拥有一个隐函的this指针
2. this指针作为成员函数的第一个参数
3. this指针**代表当前的对象**，且不能更改指向
4. 编译器在编译阶段自动加上

## 赋值运算符的特点和实现

operator是c++中用于**重载运算符**或**定义类型转换运算符**的关键字

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774834996736-609196f4-64c9-4486-a33c-9b299e131b9e.png)

当自定义类型进行赋值操作时，会调用赋值运算符语句，此时重载了 = 的功能使之能在类型中使用。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774834855578-8dac3cb0-3e95-4ca7-8c3d-546c69028b06.png)

如果pt2不存在，就会调用**拷贝构造函数**

但pt2已存在，就会调用**赋值运算符函数**

这两个函数是**有复制控制语义的函数**

  

赋值构造函数应该满足四个功能：

1. 自复制的判断
2. 回收左操作数的空间
3. 进行深拷贝
4. 返回对象本身

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774859153969-a8749d26-b0e2-4c8a-aa83-4f7a4a4d1bf9.png)

如果不重写赋值运算函数会造成double free错误

  

## 三合成原则

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774859971793-a1161637-c424-4f96-aa08-6da316b687dd.png)

## 特殊的数据成员初始化

1. 引用成员
2. const成员
3. 对象成员
4. 静态成员

除了静态成员外都要放到构造函数的初始化表达式中，**静态成员需要在类之外进行初始化**。

### 引用成员

引用成员必须被初始化，且只能在**初始化列表**中被初始化。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774596168508-0a67fc30-515e-4c6d-ab7c-a33aca64870d.png)

  

### const成员

const成员必须被初始化，且只能在**初始化列表**中被初始化。

（cpp11之后，可以在定义时初始化）

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774596443441-6c64158a-d027-4d67-b76f-db9c79d1bc69.png)

  

### 对象成员

对于类中的对象成员来说，即使不显式调用构造函数进行初始化，也会调用默认构造函数进行初始化。

下图为显式调用

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774597723208-88af0d1a-fee2-43c8-995c-86e548210e90.png)

  

### 静态成员

静态成员被存储在：**全局静态区** ，不属于对象布局的一部分，被**整个类的所有对象共享**

静态成员在类之外初始化。

## 特殊的成员函数

### 静态成员函数

有static修饰的成员函数

**不能是 构造 / 析构 / 赋值运算符 函数**

**特点：**  
没有this指针的存在

**不能访问非静态的数据成员（如果非要访问非静态数据成员，可以通过函数传参进行）**

静态成员函数可以仅通过**“类名调用”** 使用，**不需要对象的存在**。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774598607918-2bf9c810-2759-47bb-ad70-94e7b622daf5.png)

### const成员函数

在**函数参数列表之后，函数体之前**，加上const关键字：

void print() const {}

**特点：**  
1.该成员函数，不能修改数据成员，只能读取数据成员。

2.const成员函数影响的是this指针的形态：  
**const** 类名 * const this

该指针被双重修饰了，**即不能修改指向，又不能修改指向的值**。

3.const对象只能调用const成员函数（const可以修饰类类型），非const对象可以调用const成员函数

**原则：**

如果类只能提供一个版本的成员函数，就写const版本的。

如果定义的成员函数不会修改成员数据，**就必须定义为const**

**常见bug:**

在函数传参中，如果使用**const修饰的引用**（比如 const 类名 & 参数名），只能调用const版本的成员函数。

## 类数组

你可以像创建整形数组一样，使用类类型创建数组。

按正常数组一样初始化，对象会调用有参构造函数，未初始化的会调用默认构造函数。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774601292040-e84fa537-a4e1-4fb6-afe1-56529a70cb76.png)

## 类域

### 作用域和可见域

全局类：类被定义在全局中

内部类：类被定义在另一个类中。

局部类：类被定义在函数中

我们认为c语言中的一些函数，在**匿名命名空间**中。

因为cpp同名变量存在屏蔽现象，一个同名变量的**作用域和可见域**是不同的，可见域会更小。

当一个类的存在仅是为了服务另一个类，此时使用内部类

# 继承

## 继承的定义

cpp中对继承的解释：**从既有类中产生新类的过程**

既有类：**基类（父类）**

新类：**派生类（子类）**

```
class 子类
: public/protected/private 父类
{

};
```

### **继承的三个过程**

1. **吸收**基类的成员
2. **改变**基类的成员
3. **新增**成员

### 继承的局限性

1. 友元关系不能被继承
2. 构造函数，拷贝构造函数和析构函数不能被继承
3. operator new 和 operator delete， operator= 不能被继承

### 继承的访问权限

被protected修饰的成员是为他儿子所服务的

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776412365776-099f1e21-ac32-4f4e-ba20-e1aaba40ccee.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776480084205-43d3365b-53c8-45ab-9601-e20ad6b7771a.png)

对于继承方式和权限的关系，更像数学中的**取交集**

当继承的层次变多，不同继承方式对派生类的影响越明显。

默认继承方式是**私有继承**

从基类继承来的数据成员需要使用类似基类中构造函数形式进行初始化

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776415832254-76dda6fe-e30b-4f77-83a3-7b79cc0948b6.png)

## 派生类对象的构造

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776418443120-e92cc457-d4c9-4bd7-a150-b3b881a742bb.png)

**情况一**

**当派生类定义了有参构造函数，基类没定义，但需要初始化继承的基类成员**

此时调用派生类构造函数时会调用父类的无参构造函数

注意：**派生类构造函数先调用**，只是在中途需要调用父类的无参构造函数，返回结构看似是父类先调用的

**情况二**

**当派生类没定义有参构造函数，基类显式定义了有参构造函数**

要求基类必须显式定义默认构造函数

**情况三**

**当派生类和基类同时定义了有参构造函数**，但基类没定义无参构造函数

此时，派生类的初始化列表中**需要使用基类的有参构造函数**去初始化继承于基类的数据成员

**推荐不管基类有没有有参/无参构造函数都写一下，报错会很明显**

  

## 派生类对象的销毁

当派生类被销毁时，派生类的析构函数被自动调用，然后（在执行过程中）会自动**调用基类的析构函数**。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776421355949-f075d6f8-0f5f-4a97-843f-dccfd71eaae5.png)

## 多基派生

**一个子类继承于多个基类，每个基类都能选择继承方式，默认private**

构造函数的执行顺序于继承的先后顺序有关，与参数列表中的顺序无关（正序+子类）,析构函数同理（子类+倒序）.

### 访问冲突

- 当继承的**多个基类成员函数命名相同**，会出现访问冲突问题，此时需要在函数前加上 **基类名::** 的方式解决。
- 当继承的多个基类成员**有变量名和类型都相同的数据成员**，此时需要在前面加上**virtual（虚拟继承）**关键字，建立数个虚基指针。当进行一个普通继承时，**相同的量会变为一个**，多个**虚基指针（8字节）**都会被加上。如果不使用虚拟进程，子类的大小是复数个父类成员的和（会有多个相同数据成员）。

## 基类与派生类的相互转换

”**类型适应**“一个作用范围更大的事物，可以适应作用范围更小的事物。

”**向上转型**“子类向父类进行转换

1. 派生类可以适应(赋值给)基类
2. 对基类的引用可以绑定到派生类
3. 对基类的指针可以指向派生类

”**向下转型**“基类向派生类转换，语法不支持，但可以强制类型转换，在某些情况下是安全的（还是要看内存分配情况）。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776428405001-6ced6093-0597-492d-9a71-98f05e1570c2.png)

**在普通继承条件下：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776428564226-5a5216f0-215f-4373-a6ce-98b9f4cd3621.png)

派生类所占的空间大小一般大于基类，这样**引用/指针不会产生越界问题**，但进行赋值操作会产生**切片问题**，派生类的特征被丢弃了，**仅拷贝基类部分**，但语法是正确的。

## 派生对象间的相互赋值

1. 如果派生类已经定义了 拷贝运算符函数或赋值运算符函数，会直接调用。
2. 若派生类没定义，基类定义了拷贝运算符函数或赋值运算符函数，**派生类会执行缺省的赋值行为**，会调用基类的函数进行拷贝/赋值。
3. 若派生类已经定义了，基类的赋值函数不会自动调用，会发生缺省行为。要求在调用派生类的的拷贝运算符函数或赋值运算符函数时，要**显示调用基类的赋值函数（类名：：函数名）**

# 多态

同一种指令，针对不同对象，产生不同行为。

## 多态的类型

静态多态：函数重载，运算符重载，模板。**发生时机在编译时**

动态多态：虚函数。**发生时机在运行时**

## 虚函数

在成员函数前加上**virtual关键字，**声明一个虚函数。

派生类会将基类的虚函数继承下来。

  

派生类可以**重定义虚函数**，满足以下格式：

1. 函数名相同
2. 参数列表相同（参数个数，类型，顺序均相同）
3. 返回值类型相同
4. 函数体可以不同

只有虚函数函数体在运行时被确定，其他在编译时就确定了，包括参数

## 虚函数的实现

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776595372191-ef0ce272-651b-456c-85ec-3c4adb3901e0.png)

当定义虚函数时，在存储布局中多一个**虚函数指针（8字节 long *）**。

派生类会继承基类的虚函数指针，该指针指向类型自己的虚函数表，虚函数表存储虚函数的入口地址，继而找到程序代码区对应的位置。

当派生类重写虚函数时，虚表的入口地址就会被覆写。

**虚表的存在：**

对于单继承而言，虚表只有一个，存放在**只读段**中，供类中所有对象所共有。

- 将类的vfptr用（long*）取出来再进行偏移，可以获得数据成员的值，但此时就**丧失了面向对象的特性**

## 虚函数被激活的五个条件

1. 基类要定义虚函数
2. 派生类要重定义虚函数（重写、覆盖）
3. 创建派生类对象
4. 用基类 指针(引用) 指向(绑定) 派生类对象（不能是将派生类对象·强转成基类指针）
5. 用基类指针调用虚函数（此时基类指针**只能调用虚函数**）

这5个条件是**显示动态多态**所必需的

## 不能被设置成虚函数的函数

1. 普通函数，声明为虚函数无意义，编译器在编译时就绑定；=了普通函数
2. 静态函数，为全体对象共有没有动态绑定的必要
3. 内联函数，在编译时作用，替换函数调用，而虚函数在运行时作用。
4. 友元（友元本身是另一个类的成员函数的情况除外）
5. 构造函数。构造函数没执行的时候，类就没有被初始化，虚函数指针有没有还不知道。

当构造函数的初始化列表执行完毕后，对象就已经被初始化了

**虚函数和动态多态不等价**：

动态多态的体现必须有虚函数，虚函数存在不一定能体现动态多态。

## 纯虚函数和抽象类

```
class Point{
    virtual void print() = 0; //纯虚函数声明但未实现，Point是抽象类
                              //难以修改纯虚函数形式，不然所有派生类都要改
}

class Line
: public Point
{
    void print() override{ //对虚函数进行重写，Line不是抽象类
        cout << "print a line" << endl;
    }
}

//多个类继承Point并重写print(),体现抽象类的接口特性
class Circle: public Point{};

void test(Point * pt){
    pt->Point(); //体现多态特性
}
```

**声明了纯虚函数但未实现的类**被称为**抽象类**，抽象类不能创建对象。

子类继承了抽象类，**不实现虚函数也是抽象类**，不能创建对象。

可以创建抽象类的**指针/引用**（指针固定占有8字节，但抽象类是不完整的）

- 特点：**抽象类是作为接口使用的，对扩展开放，对修改关闭**

**抽象类的第二种形式：**

用**protected修饰构造函数**的类也是抽象类，不能创建对象，但是其派生类能创建对象。

## 虚析构函数

用delete释放基类的指针（指向派生类类）时，只会调用基类的析构函数，派生类多的空间没有被释放，产生内存泄漏。

当基类的析构函数设置成虚函数后，派生类的析构函数自动变为虚函数，这是**唯一一个名字相同的虚函数**。

对于任何一个类而言，析构函数只有一个，编译器将析构函数名视为destructor

对于一个有虚函数的类，需要让他的基类设置为虚析构函数

## 重载、覆盖、隐藏

**重载**：在同一作用域中，函数名相同，参数列表不同（**参数个数、类型、顺序**），**仅靠返回值不构成函数重载**

**覆盖**（重定义，重写）：在基类和派生类中，**虚函数**的函数名，返回值和参数列表完全相同

**隐藏**：在基类和派生类中，派生类的函数隐藏了基类的同名函数（不一定是虚函数，普通成员函数、同名数据成员也会被隐藏）

  

## 多基派生的二义性

当派生类继承多个基类时，会将**多张虚表**继承下来，多个同名虚函数会有二义性，需要加 **基类名 : :** 区分

当派生类对多个同名虚函数进行重写时，多张虚表的入口地址都会改变

- 使用 **override** 判断一个函数是不是虚函数

为了解决这种二义性，我们引入了**虚拟继承**

  

## 虚继承

这个博客，详细讲解了cpp内存布局：[https://www.cnblogs.com/QG-whz/p/4909359.html](https://www.cnblogs.com/QG-whz/p/4909359.html)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776770331830-1ad6d995-b415-4ab9-a4ab-85e0c0988d40.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776769309086-4b066db2-dfba-4057-a8ff-7f1f7a07909a.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776770353011-d550cb80-9c73-43d6-b6c0-15ca7e5c9391.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776770367535-55159c51-2648-456e-a6b5-ae5dd3c5ac92.png)

  

  
# 设计模式

## 单例模式

**一个类只能生成一个对象**，且是唯一的对象。（由类到对象，称为类的实例化）

该对象不能是：栈对象，全局/静态对象

单例模式

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774746139383-6f477ddf-a5e3-4b74-a30c-c16e48eff559.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1774746157101-c2fd152d-a253-456b-b4d9-264fa629f6bc.png)

设计步骤：

1. 将**构造函数私有化**
2. 在类中定义一个**静态的**，指向本类的指针变量
3. 定义一个**返回值为类指针的静态成员函数**

  

不能使用new，创建堆对象，因为new创建对象时会调用构造函数。

不能使用delete，删除堆对象，因为delete删除对象时会调用析构函数

也需要将析构函数私有，其步骤和前者一样

**应用：**  
1. 用单例模式替换全局变量

2. 读取配置文件

3. 词典、网页库

## 单例模式的自动释放

单例模式可以不写destroy（）函数，因为进程结束前会自动回收资源。

这样仍然有内存泄漏风险，会被内存泄漏工具检测出来，形象代码效率。

我们希望单例模式可以自动释放堆空间！

### 使用内部类的方式

创建一个内部类作为单例模式的成员，其进行销毁时，执行析构函数会销毁单例模式的对象。要**用static修饰内部类成员变量**，这样该成员和单例模式对象就分离了，不会出现两者互相等待对方销毁而产生死锁现象。

内部类模式

```
#include<iostream>
#include<stdlib.h>
#include<pthread.h>


class Singleton{
public:
    static Singleton *getInstance()
    {
        if (!_pInstance)
        {
            _pInstance = new Singleton();
            atexit(destroy);
        }
        return _pInstance;
    }

    static void destroy(){
        if(_pInstance){
            delete _pInstance;
            _pInstance = nullptr;
        }
    }
private:
    class AutoRlease
    {
    public:
        AutoRlease()
        {
            std::cout << "AutoRelase() is called!" << std::endl;
        }

        ~AutoRlease()
        {
            std::cout << "~AutoRelase() is called!" << std::endl;
            if (Singleton::_pInstance != nullptr)
            {
                delete _pInstance;
                Singleton::_pInstance = nullptr;
            }
        }
    };

private:
    Singleton(){
        std::cout << "Sigleton is called!" << std::endl;
    }
    ~Singleton(){
        std::cout << "~Sigleton is called!" << std::endl;
    }

    static Singleton *_pInstance;
    static AutoRlease _ar;
};

Singleton *Singleton::_pInstance = nullptr;
Singleton::AutoRlease Singleton::_ar;

int main(){
    Singleton* sl = Singleton::getInstance();
    return 0;
}
```

### 线程安全模式

delete绝对不能写道私有化的析构函数里面，会无限调用析构函数

使用了**atexit(void (*function)(void))**函数，其注册的函数在进程退出前会执行一次/数次

使用**pthread_once(**pthread_once_t ***once_control,void (*init_routine)(void))**函数保证线程安全，当once_control被写成固定值，init_routine()函数就会被执行固定次

线程安全模式

```
#include<iostream>
#include<stdlib.h>
#include<pthread.h>


class Singleton{
public:

    static Singleton * getInstance(){

        if(!_pInstance){
            pthread_once(&_once, init_routine);
        }
        return _pInstance;
    }

    static void destroy(){
        if(_pInstance){
            delete _pInstance;
            _pInstance = nullptr;
        }
    }

    static void init_routine(){
        _pInstance = new Singleton();
        atexit(destroy);
    }

private:
    Singleton(){
        std::cout << "Sigleton is called!" << std::endl;
    }
    ~Singleton(){
        std::cout << "~Sigleton is called!" << std::endl;
    }

    static pthread_once_t _once;
    static Singleton *_pInstance;
};

Singleton *Singleton::_pInstance = Singleton::getInstance();//饿汉模式
pthread_once_t Singleton::_once = PTHREAD_ONCE_INIT;

int main(){
    Singleton* sl = Singleton::getInstance();
    return 0;
}
```

## PIMPL

由内部类引申出的设计模式，通过**私有的成员指针**，将指针指向的的类的内部实现做隐藏。

**优点如下**：  
![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776157201824-5fab26cf-603d-4077-b7a8-5732377b98ec.png)

如果两个类之间相关性强，你又不想让别人去看到类的实现，就可以用PIMPL

PIMPL示例

```
#ifndef _LINE_H_
#define _LINE_H_

class Line{
public:
    Line(int, int, int, int);
    ~Line();

    void printline(){
    }    
    class Linepimpl;
private:
    Linepimpl *_pimpl;
};

#endif
```

  

```
#include <iostream>
#include "line.h"
using std::cout;
using std::endl;

class Point
{
public:
    Point(int x, int y)
        : _ix(x), _iy(y)
    {
    }
    ~Point()
    {
    }

    void ptintPoint() const
    {
        cout << "(" << _ix << "," << _iy;
    }

private:
    int _ix;
    int _iy;
};

class Line::Linepimpl
{
public:
    Linepimpl(int x1, int y1, int x2, int y2)
        : _p1(x1, y1), _p2(x2, y2)
    {
    }

    ~Linepimpl()
    {
    }

    void print() const
    {
        _p1.ptintPoint();
        cout << "---->";
        _p2.ptintPoint();
        cout << endl;
    }

private:
    Point _p1;
    Point _p2;
};

Line::Line(int x1, int y1, int x2, int y2)
: _pimpl(new Linepimpl(x1, y1, x2, y2))
{
    cout << "Line::Line(int x1, int y1, int x2, int y2) is called!" << endl;
}

Line::~Line()
{
    cout << "Line::~Line" << endl;
    if (_pimpl)
    {
        delete _pimpl;
        _pimpl = nullptr;
    }
}

void Line::printline()
{
    _pimpl->print();
}
```

  

```
#include"line.h"

int main(){
    // g++ testLine.cc line.h pointToLine.cc
    Line line(1, 2, 3, 4);
    line.printline();
    return 0;
}
```