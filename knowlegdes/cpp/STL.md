# 标准模板库

**STL六大基本组件**：

1. 容器：用来存储数据，称为数据结构。

序列式容器：vector，list，deque

关联式容器：set，map

无序关联式容器：unordered_set，unordered_map

2. 迭代器 类似于指针，具有指针的功能，用来连接数据结构与算法
3. 算法 操作数据的方法
4. 适配器 STL算法实现不针对具体容器，某些算法不支持具体容器，需要适配器进行转接
5. 函数对象 做定制化操作
6. 空间配置器 空间申请与释放

## 序列式容器

如vector，list，deque,他们是序列式容器，具有相似的构造方式

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777454143929-cf79cf9d-f846-4625-9c20-588cb3953d19.png)

### **初始化和遍历方式：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777453400629-35874182-02cc-4819-8587-e69b1360b141.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777453428394-d1bdfa89-8af9-43b2-8442-392e48368c94.png)

vector，list，deque初始化和遍历方式也相似，但list不支持下标访问，也不能用下标进行遍历。

**迭代器iterator**有类似指针的功能，但不能说他是一个指针。某些容器将 类类型 重定义为 iterator 并重载了箭头或下标访问运算符，使其具有指针的特性。（但你不需要知道容器的内部实现，仍能将想要的类型提取出来，称为**类型萃取**）

### **修改操作：**

都可以使用push_back()和pop_back()在尾部增删元素。

list和deque则可以使用push_front()和pop_front()在头部增删元素

（vector的内存布局决定了，其在头部增删元素开销巨大，不支持此操作）

```
list<int> lt = {1,2,3,4,5};

lt.push_back(10);
lt.pop_back();

lt.push_front(122);
lt.pop_front();
```

### 清空操作

使用clear进行清空操作，但容器容量不会变。

```
vector<int> v = {1,2,3};
v.clear();
v.shrink_to_fit();
```

### 更多操作

很多东西大概记住就行，细节忘了去查：

使用**reverse()**翻转容器元素

对 **sort(comp())**重写排序方法

使用 sort + unique + erase 进行连续去重

**unique**的作用是将**把不重复的元素移到前面，把重复的元素移到后面**，并返回第一个重复元素的迭代器

a.erase(std::unique(a.begin(), a.end()), a.end());

使用**shrink_to_fit**( )来减小容量

使用list_A.**merge**(list_B)将两个已经按升序排序的列表归并成有序列表

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777794210319-827fe817-84f1-4fb3-9e14-a3f170c46e7d.png)

使用list_A.**splice**(list_A::iterator it,list<T> &B,...list_B::iterator)将列表B的部分插入列表A的指定位置中（操作同一个列表时，**不能交叉pos和截取范围**）

### **迭代器失效：**

1. 当使用迭代器iterator指向vector的数据存储空间时，如果发生了**扩容**，迭代器就会失效，此时如果使用未更新的迭代器，就会发生段错误。
2. 当vector中it指向的元素被删除时，后面的元素和尾指针会向前挪动，此时it应该自动指向下一个元素，如果it自增，就会产生迭代器失效
3. 如果对vector进行迭代器遍历时，进行了push操作，发生扩容后，如果扩容后的区域在it1前面，it就永远找不到尾迭代器，一直循环下去，发生迭代器失效

**解决迭代器失效的方法“**

- it = v.begin() 每次使用迭代器之前，进行重新置位

### vector源码解读

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777460687076-ad80ea6f-3f9b-4de1-870f-eada57e8bef1.png)

这能解释为什么sizeof(vector<int>)会返回24，vector内部有3个指针类型成员，分别是_M_start，_M_finish，_M_end_of_storage

### **vector动态扩容的过程：**

当动态数组的**size() == capacity()**时就会动态扩容，那么先申请 2 * capacity的空间，再将原来空间上的元素拷贝到新空间，再在新空间中存储新元素。

在微软编译器VC++中，每次进行扩容时，capacity会变为原来的1.5倍，但在gcc中会变为两倍。

c11标准只是一个文档，不同厂家有不同实现，只需要有固定功能就行（接口和实现分离，体现解耦的思想）

**insert扩容原理：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777598252634-a520064a-c0d7-47c7-9b55-14ea0077c52e.png)

### deque源码解读

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777460706908-9703c269-4192-48a4-88d2-220f3b417bdc.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777461916876-76f557f9-8c87-4100-b78a-ae41f9ded561.png)

deque使用**中控器Map数组**来管理数据，由多个小片段组成，其**逻辑上是连续的但物理上分散**。

当元素过多后，会重新申请小片段，当空间不够用会扩容中控器数组大小。

# 关联式容器

像map，set属于关联式容器。查找复杂度都较高

## set集合

- 关联容器，集合
- 保持**唯一的key值**，会自动进行**去重、排序**
- 底层实现是**红黑树**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776498291938-ef2fdde3-a9bd-4720-880e-e3bc982842b9.png)

## 关联式容器的使用

**inputit = input iterator 迭代器**，可以理解为广义指针

一个类可以像**指针**一样使用，说明类中重载了，箭头(**->**)，解引用(*****)，自增自减运算符(**前置++，后置++，前置--，后置--，偏移+-n，+=，-=** )

## set的基本使用

**初始化：**

通过**大括号**或**数组赋值**进行初始化

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776499196120-ec43589e-3afb-471b-8fe8-b27f71d60354.png)

**遍历：**

可通过**迭代器+brgin() end()**或**auto增强for循环**进行遍历

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776499248794-0546f75b-341b-4af6-8fd8-144a43009b14.png)

**查找：**

通过 **.count(key)** 方法进行查找，返回 size_t 类型的 0 或 1

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776500078356-0f75b44d-f16f-4108-a484-e9d030f26a15.png)

通过 **.find(key)** 方法进行查找，返回 迭代器类型，查找失败返回类似nullptr

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776500068974-b64b385f-4998-4760-bb08-407260fc4bf5.png)

**插入：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776501961351-e7daaf38-4419-4ffe-ad40-7128eed7561c.png)

**借用insert对pair进行补充：**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776500754668-6e924419-589c-4d28-bdde-5ec5edc3cb4a.png)![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776500770158-bbdc600a-1f49-4d86-94b5-2e2e7abe35b7.png)

通过pair类型，可以**获得俩个返回值**，且能用 **.first 和 .second** 进行访问

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776500689858-6965c087-704d-4f4f-b9dc-8e27945b3fd8.png)

insert会返回一个**pair<iterator,bool>**类型的值

若成功，返回iterator和**true**

若失败，返回iterator和**false**

**这样设计让insert变得更安全**

因为红黑树的特性**set不支持修改操作**

## multiset

和set基本相同，但**可以有重复元素**，默认按升序排列

**查找元素：**

lower_hound(t) 返回第一个大于等于 t 的元素迭代器

upper_bound(t) 返回第一个大于 t 的元素迭代器

可以使用equal_range( t ) 返回pair类型的一对迭代器

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777604883431-d46c6055-bf01-44af-b8a4-a7a6c65f06a9.png)

还有一点区别是，multiset的insert()操作会返回一个迭代器，而set会返pair，因为multiset不会出现插不进去的情况

## map键值对字典

- 关联容器，字典
- 保持**唯一key值，(可重复)value值**，会自动进行**按key值排序 ，**存储的是**pair类型**
- 底层实现是**红黑树**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776502198693-482f20f9-9f79-4866-b44e-8e8d787c603c.png)

## map的基本使用

map和set使用几乎没区别，但他多带有一个值

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776503671743-8dc6f664-99e1-408f-8f4a-0a9a2b3c4b9d.png)

**map的下标访问：**

map的下标访问有**查找，赋值（插入），修改**的功能。

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1776504073395-f31d93cc-270a-4208-bda0-06474e2831dd.png)

## multimap

可以有相同的k值，默认按升序排列。

因为·有相同的key值，所以不支持下标访问。

insert返回结果是迭代器

# 无序关联式容器

无序关联式容器底层都使用哈希，让查找的时间复杂度将为 O(1)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777619577168-a5d2e3e0-7881-42e0-a8de-a21163284e8f.png)

遍历出来的值一定是随机的

## 对于自定义类型的特化

对于自定义类型，要求重写模板参数**hash<T>和equal_to<T>**

1. 可以以**仿函数（重载operator()的类）**的形式进行实现

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777621722401-0c833fb3-1f44-4511-8139-aff8667dd56d.png)

2. 可以在标准命名空间中，需要对模板进行**自定义类型**的全特化（内置类型已经给写好了）

greater<>和less<>也可以以该方法进行特化

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777798165905-6a596eb5-a9da-4077-bd7e-a9fe7f854eb3.png)

3.对**运算符进行重载**

std::less的源码如下

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777859424970-d7a19e0f-94cd-4188-94b3-f4dc966d156d.png)

在程序中对operator<进行重载的效果也是一样

**为什么使用仿函数:**

- STL模板参数必须是一个类
- 伪函数可以存储状态（他毕竟是类）
- 调用速度比函数指针要快，编译器inline内联的

## 容器的选取

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777623578138-5f748b48-a153-4c3c-a7a6-6f8f15a1c22d.png)

# 容器适配器

## 优先队列priority_queue

优先级队列底层使用**大根堆（大顶堆）**

排序原理：  
![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777624731423-602f3ef5-8acc-45bb-9b80-09ac0c2ac167.png)

**priority_queue的定义：**

priority_queue模板是<数据类型，虚列式容器，模板类>

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777624844220-0cfb3e25-4de5-4ed0-bd85-e47cc290781e.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777625352484-fbac19c7-7287-4a80-8790-f90bf44acc93.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777625342693-6804ce72-955c-4334-839b-344f74caf9cd.png)

自定义类型重载如下：

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777799586288-87b66a57-a6be-4554-b352-beca7c050f03.png)

# 迭代器

将容器和算法分开写，实现**低耦合**，用迭代器将其连接。

迭代器类似于指针，使得算法独立于容器，独立于类型

## 五种迭代器类型

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777861927629-624114be-2ced-4789-94d2-4c57b6d15901.png)

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777861994860-3cc29883-d5e9-4799-a773-db8f22376dbb.png)

## **输入与输出流迭代器**

- 使用流迭代器必须引入头文件<iterator>
- 输入/输出流将**数据存储到缓冲区**中，可以看成一个**容器**。
- 使用流迭代器的算法必须可以和流一起工作

### 输出流迭代器

```
//构造函数
ostream_iterator( ostream_type& stream, const CharT* delim );
ostream_iterator( ostream_type& stream ); 

/*
构造函数源码
ostream_iterator(ostream_type& __s, const _CharT* __c) 
    : _M_stream(&__s), _M_string(__c)  {}
  ostream_iterator<_Tp>& operator=(const _Tp& __value) { 
    *_M_stream << __value;
    if (_M_string) *_M_stream << _M_string;
    return *this;
  }
*/
    
//算法copy
template< class InputIt, class OutputIt >
 OutputIt copy( InputIt first, InputIt last, OutputIt d_first );

//示例 一种使用输出流迭代器遍历容器的方式
    vector<int> v = {1,2,3,4,5};
    ostream_iterator oit(cout,"\n");
    copy(v.begin(),v.end(),oit);
```

### copy源码

```
//copy层层调用后的结果如下
inline _OutputIter __copy(_InputIter __first, _InputIter __last, _OutputIter __result,
                          input_iterator_tag, _Distance*){
  for ( ; __first != __last; ++__result, ++__first)
    *__result = *__first;
  return __result;
}

//调用cpoy后，对应数据成员如下
first = Container.begin();
last = Container.end();
result = ostream_iterator;

/*
当执行 *__result = *__first;

输入流迭代器内部重载赋值运输符
ostream_iterator& operator=( const T& value ){
    *out_stream << value;
    if(delim != 0)
         *out_stream << delim;
    return *this;
}

stream - 此迭代器所访问的输出流(取得地址 &cout) 
delim - 在每次输出后插入流的空终止字符串 
这样数据就输入到输出流中了
*/
```

### 输入流迭代器

```
//在构造stream_iterator时显式调用了_M_read()
istream_iterator(istream_type& __s) 
:_M_stream(&__s) {
    _M_read(); 
}
//校验流的状态并为_M_value进行赋值
void _M_read() {
    _M_ok = (_M_stream && *_M_stream) ? true : false;
    if (_M_ok) {
      *_M_stream >> _M_value; //这里会造成一次阻塞 
      _M_ok = *_M_stream ? true : false;
    }
  }
//前置++操作会调用一次_M_read()
istream_iterator& operator++() { 
    _M_read(); 
    return *this;
  }
//解引用会返回_M_value
 reference operator*() const { return _M_value; }


//示例：将输入信息存储到number变长数组中
//back_insert_iterator<vector<int>>(number) 与 back_inserter(number)完全等价
vector<int> number;
istream_iterator<int> isi(std::cin); 
copy(isi,istream_iterator<int>(),
    std::back_insert_iterator<vector<int>>(number))  //只有识别到Eof才会跳出输入循环
```

```
__first = istream_iterator<int> isi
__last = istream_iterator<int>()
__result = back_insert_iterator<vector<int>>(number)

最后一定会执行copy代码
inline _OutputIter __copy(_InputIter __first, _InputIter __last, _OutputIter __result,
                          input_iterator_tag, _Distance*)
{
  for ( ; __first != __last; ++__result, ++__first)
    *__result = *__first;
  return __result;
}


//back_insert_iterator重载了赋值运算符，调用了push_back()
back_insert_iterator<_Container> &
  operator=(const typename _Container::value_type& __value) { 
    container->push_back(__value);
    return *this;
}

//重载了前置++,返回back_insert_iterator对象本身
back_insert_iterator<_Container>& operator++() { return *this; }
//重载了前置++,返回back_insert_iterator对象本身
back_insert_iterator<_Container>& operator*() { return *this; }
```

### 插入迭代器

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777883356016-74f4035e-162b-4187-a96f-d0ef3c33af04.png)

std::copy + 迭代器适配器 是 for 循环 + push_back 的「**声明式封装**」，底层本质一样，但代码更简洁、可复用性更强，是 C++ STL 风格的推荐写法。

## 反向迭代器

```
vector<int> number = {1,4,2,4,1,2,4,5};
vector<int>::reverse_iterator rit; 
for(rit = number.rbegin(); rit != number.rend(); ++rit){ //++在源码中重载成了--
    cout << *rit << " ";
}cout << endl;
```

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777884211159-b1ebec97-cb54-4d24-8a57-070b2eba3c6d.png)

# 算法简介

## 算法类型

算法库<algorithm>中的算法类型

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777885613303-8e49155b-c4bb-43ba-ab02-553dfd5d0b55.png)

## 函数类型

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1777889118946-d4b33208-f9e0-478b-8496-f46df908d6e1.png)

## 修改式算法

### for_each

将迭代器 __first ->__last 进行遍历，并将迭代器指向的元素放入一元函数func中

```
void func(int &value){
    cout << value << " ";
}

//遍历number.begin(),number.end()并将迭代器指向的值传给func()
vector<int> number = {1，2，3，4，5};
for_each(number.begin(),number.end(),func); //func必须是一元函数

//lambda表达式 == 匿名函数
for_each(number.begin(),number.end(),
    [](int &value){
        cout << value << " ";
    });

//for_each源码
template <class _InputIter, class _Function>
_Function for_each(_InputIter __first, _InputIter __last, _Function __f) {
  __STL_REQUIRES(_InputIter, _InputIterator);
  for ( ; __first != __last; ++__first)
    __f(*__first);
  return __f;
}
```

### re_moveif

将符合一元断言的元素移到最后，返回第一个满足条件的指针，可以配合erase将容器的元素删除。

```
//一元断言
bool func(int num){
    return num > 4;
}
//使用实例    
vector<int> number = {1,2,3,4,5,6,7,8};
number.erase(std::re_moveif(number.begin(),number.end(),func),number.end())

    
//remove_if实现
template<class ForwardIt, class UnaryPredicate>
ForwardIt remove_if(ForwardIt first, ForwardIt last, UnaryPredicate p)
{
    first = std::find_if(first, last, p); //先调用了一步find_if()
    if (first != last)
        for(ForwardIt i = first; ++i != last; )
            if (!p(*i))
                *first++ = std::move(*i);
    return first;
}

//find_if实现
template<class InputIt, class UnaryPredicate>
InputIt find_if(InputIt first, InputIt last, UnaryPredicate p)
{
    for (; first != last; ++first) {
        if (p(*first)) { 
            return first;
        }
    }
    return last;
}
```

## bind

### bind的使用

**绑定函数参数并返回函数对象**，用auto接收

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1778054800165-434ed2d1-92fa-41ac-a925-0ddef9811ffb.png)

bind可以绑定到n元函数，该函数可以是**自由函数，全局函数**，也可以绑定到**成员函数**，甚至**数据成员**上

```
#include<functional>

//可以绑定普通函数
int add(int a, int b, int c)
{ return a + b + c;}

auto f = std::bind(&add,1,2,3);
cout << f() << endl;  // cout: 6

//可以绑定类的成员函数
class Line{
public:
    int getLength(int x, int y){
        return sqrt(x*x + y*y);
    }

    int data = 200;
};

Line l1; //如果想调用类的非静态成员函数，第一个变量是 this指针
auto f1 = std::bind(& Line::getLength, &l1, 1, 2);
cout << f1() << endl;

//可以绑定到数据成员
auto f2 = syd::bind(&Line::data,&l1); //绑定一个this指针
cout << f2 << endl;  //cout: 200 返回一个 int() 类型函数


//可以绑定到函数指针
int func(int y){
    return y;
}

typedef int (pFunc*)(int); //指向一个参数为int，返回类型为int的函数指针类型
                   
pFunc f3 = &func2; //该指针可以像函数一样被调用, 此时可以说注册了一个回调函数

//... ....回调的核心，函数在恰当时机（运行时）被调用
cout << f3(100) << endl;
```

**bind1st/bind2st函数**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1778053394082-4ac32dda-bce0-4c55-a5d2-d1a6b5eda1e5.png)

传入二元函数对象和参数变量，就会将参数变量绑定到该函数的第一/二个参数，此时**该二元函数就近似为一个一元函数对象**，供算法调用。

使用：remove_if (number.begin() , number.end(), bind1st(std::less<int>(), num));

### **占位符**

占位符占用的是**形参**的对应位置，获取**实参**对应位置

```
void func(int x1, int x2, int x3, const int & x4){
    cout << x1 << endl;
    cout << x2 << endl;
    cout << x3 << endl;
    cout << x4 << endl;
}

number = 100;
auto f = bind(&func, 10, std::placeholders::_2,std::placeholders::_3, number);
f(1,2,3,4,5,6,7);
//10 2 3 100 100
```

### **引用包装器**

```
int number = 100;
auto f = bind(&func, 10, std::placeholders::_2,std::placeholders::_3, std::cref(number));
number = 700;
f(); //此时会输出700而非100，因为指定了引用传递，而非值传递。
     //侧面表明了bind绑定的函数作用时机是在运行时
```

## function

我们可以推导出被bind绑定后**返回的函数类型**。

function可以接收函数类型，function其实是一种函数的容器

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1778058410411-b95a6a66-a590-4d6d-9dcb-d40a64449786.png)

```
int add(int a, int b, int c)
{ return a+b+c; }

function<int()> f =  bind(&add,1,2,3); //bind的返回值是 int() 类型的函数对象
cout << f() << endl;

```

### function和bind实现多态

## 成员函数绑定器

**std::mem_fn**

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1778224752213-d3f86102-9850-4b6f-a857-5fb20e480159.png)

```
//函数指针和成员函数指针
int (*Func) (int, int); 
int (Test::*pFunc) (int ,int); 

//对成员函数指针进行赋值
//要取地址的原因是，c++中成员函数名不能代表函数地址，c++允许进行函数重载
//c++又兼容c，普通函数的函数名仍然代表函数地址
pFunc = &Test::add; 
```

```
vector<Number> v = {Number(1),Number(13),Number(14),Number(11),Number(2)};
//调用Number类中的print函数打印数据
std::for_each(v.begin(),v.end(),std::mem_fn(&Number::print));
```

# 空间配置器allocator

将空间申请与对象构建分开

## 常用成员函数

```
//大多时候无需构造，std::allocator本身无状态
//allocator的成员函数都是静态成员函数，直接用作用域限定符进行访问
template<class T>
static std::allocator<T> _allo;


//申请n长度的空间，返回对应的迭代器（指针）
iterator it =  _allo.allocate(n);
//定位new，在it所指的未初始化存储中构造 T 类型对象
_allo.construct(it,T());
//调用对象的析构函数,释放资源
_allo.destroy(it，it + n);
//归还申请的内存，使之供操作系统调用，从it开始释放n个
 _allo.dealallocate(it, n);
```

## 空间配置器源码解读

```
//有些代码可能根本读不懂，此时可以将值代入算一算
//——————————————————————————————————————————————————————————————————————————————————
static void* allocate(size_t __n){
    if(__n > 128){
        malloc(__n);
    }else{
        //16个自由链表 + 内存池
        //平衡效率问题和解决小空间malloc的内存碎片问题
        _Obj** __my_free_list 
          = _S_free_list + _S_freelist_index(__n);
        _Obj* __result = *__my_free_list; //__result指向内存块链表
      if (__result == 0)
        __ret = _S_refill(_S_round_up(__n));
      else {
        *__my_free_list = __result -> _M_free_list_link; //自由链表
        __ret = __result;
      }
    }
     return __ret;
}

//——————————————————————————————————————————————————————————————————————————————————
  union _Obj {
        union _Obj* _M_free_list_link;
        char _M_client_data[1];    /* The client sees this. */
  };
private:
# if defined(__SUNPRO_CC) || defined(__GNUC__) || defined(__HP_aCC)
    static _Obj* __STL_VOLATILE _S_free_list[]; 

//——————————————————————————————————————————————————————————————————————————————————
 static  size_t _S_freelist_index(size_t __bytes) {
        return (((__bytes) + (size_t)_ALIGN-1)/(size_t)_ALIGN - 1);
  }

#if ! (defined(__SUNPRO_CC) || defined(__GNUC__))
    enum {_ALIGN = 8};
    enum {_MAX_BYTES = 128};
    enum {_NFREELISTS = 16}; // _MAX_BYTES/_ALIGN

//——————————————————————————————————————————————————————————————————————————————————
//对8的倍数向上取整
S_round_up(size_t __bytes) 
    { return (((__bytes) + (size_t) _ALIGN-1) & ~((size_t) _ALIGN - 1)); }

//——————————————————————————————————————————————————————————————————————————————————
//切割内存块
void* __default_alloc_template::_S_refill(size_t __n)
{
    int __nobjs = 20;
    char* __chunk = _S_chunk_alloc(__n, __nobjs);
    _Obj** __my_free_list;
    _Obj* __result;
    _Obj* __current_obj; //链表
    _Obj* __next_obj;
    int __i;

    if (1 == __nobjs) return(__chunk);
	
    __my_free_list = _S_free_list + _S_freelist_index(__n);

    /* Build free list in chunk */
      __result = (_Obj*)__chunk;
      *__my_free_list = __next_obj = (_Obj*)(__chunk + __n);
      for (__i = 1; ; __i++) {
        __current_obj = __next_obj;
        __next_obj = (_Obj*)((char*)__next_obj + __n);
        if (__nobjs - 1 == __i) {
            __current_obj -> _M_free_list_link = 0;
            break;
        } else {
            __current_obj -> _M_free_list_link = __next_obj;
        }
      }
    return(__result);
}

//——————————————————————————————————————————————————————————————————————————————————
 // Chunk allocation state.
  static char* _S_start_free;
  static char* _S_end_free;
  static size_t _S_heap_size;

template <bool __threads, int __inst>
char*
__default_alloc_template<__threads, __inst>::_S_chunk_alloc(size_t __size, 
                                                            int& __nobjs)
{
    char* __result;
    size_t __total_bytes = __size * __nobjs; //__nobjs = 20
    size_t __bytes_left = _S_end_free - _S_start_free; //如果结果为0，需要申请空间

    if (__bytes_left >= __total_bytes) {
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else if (__bytes_left >= __size) {
        __nobjs = (int)(__bytes_left/__size);
        __total_bytes = __size * __nobjs;
        __result = _S_start_free;
        _S_start_free += __total_bytes;
        return(__result);
    } else {
        size_t __bytes_to_get = 
	  2 * __total_bytes + _S_round_up(_S_heap_size >> 4);
        // Try to make use of the left-over piece.
        if (__bytes_left > 0) {
            _Obj** __my_free_list =
                        _S_free_list + _S_freelist_index(__bytes_left);

            ((_Obj*)_S_start_free) -> _M_free_list_link = *__my_free_list;
            *__my_free_list = (_Obj*)_S_start_free;
        }
        //如果内存池为空，也没有足够的堆空间，就会返回 0
        _S_start_free = (char*)malloc(__bytes_to_get); 
        if (0 == _S_start_free) { 
            size_t __i;
            _Obj** __my_free_list;
	    _Obj* __p;
            // Try to make do with what we have.  That can't
            // hurt.  We do not try smaller requests, since that tends
            // to result in disaster on multi-process machines.
            //如果申请不到，就往列表更大的内存区块借空间
            for (__i = __size;
                 __i <= (size_t) _MAX_BYTES;
                 __i += (size_t) _ALIGN) {
                __my_free_list = _S_free_list + _S_freelist_index(__i);
                __p = *__my_free_list;
                if (0 != __p) {
                    *__my_free_list = __p -> _M_free_list_link;
                    _S_start_free = (char*)__p;
                    _S_end_free = _S_start_free + __i;
                    return(_S_chunk_alloc(__size, __nobjs)); //递归调用_S_chunk_alloc
                    // Any leftover piece will eventually make it to the
                    // right free list.
                }
            }
	    _S_end_free = 0;	// In case of exception.
            _S_start_free = (char*)malloc_alloc::allocate(__bytes_to_get);
            // This should either throw an
            // exception or remedy the situation.  Thus we assume it
            // succeeded.
        }
        _S_heap_size += __bytes_to_get;
        _S_end_free = _S_start_free + __bytes_to_get;
        return(_S_chunk_alloc(__size, __nobjs));
    }
}
//——————————————————————————————————————————————————————————————————————————————————
```

![](https://cdn.nlark.com/yuque/0/2026/png/63601471/1778312966258-173c6a4e-39fb-48b0-a6f9-4b101a537613.png)

```
//——————————————————————————————————————————————————————————————————————————————————
 static void deallocate(void* __p, size_t __n)
  {
    if (__n > (size_t) _MAX_BYTES)
      malloc_alloc::deallocate(__p, __n);
    else {
      _Obj* __STL_VOLATILE*  __my_free_list
          = _S_free_list + _S_freelist_index(__n);
      _Obj* __q = (_Obj*)__p;

      // acquire lock
#       ifndef _NOTHREADS
      /*REFERENCED*/
      _Lock __lock_instance;
#       endif /* _NOTHREADS */
      __q -> _M_free_list_link = *__my_free_list;
      *__my_free_list = __q;
      // lock is released here
    }
  }

//——————————————————————————————————————————————————————————————————————————————————
```