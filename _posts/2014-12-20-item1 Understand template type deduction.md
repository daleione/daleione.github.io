---
layout: page
title: 第1条 理解函数类型推导
---

衡量一个复杂系统的设计如何，有个简单的办法——看看能否在不必了解它的工作方式的情况下，就能顺利流畅地得到满意的效果（以至大部分时候甚至会忽略了它的存在）。以这个标准看，C++的模版类型推导无疑是很成功的。有了模板类型推导后，如果不是一些必须显式指定的场合，程序员几乎不会再刻板地手动填充类型参数，即是明证。

在 C++11 中，auto 的类型推导仍然沿袭了模板类型推导的机制。虽然二者大部分情况下是可类比的，但有时 auto 的推导看起来更晦涩一些。这就需要我们比原先更深入地理解类型推导的具体过程。

------------------------

[GL] EMC 一上来第一条不再是以往那样的具体实践准则，而更是像是概念和理论性的铺垫内容。


------------------------

## 基本推导的分情况讨论

举个例子：

```c
template<typename T>
void f( ParamType param);
...
f( expr); 
```

这个例子里， expr 被用来推导出 T 和 ParamType。注意， T 和 ParamType 经常是不一样的，因为 ParamType 可以加上 cvr (const-volatile-ref) 修饰符，比如下面这种情况：


```c

template<typename T>
void f( const T& param );  
...
int x = 0;
23 f( x);

```

当类型推导时，T 被推导为 int， 而 ParamType 则会变成 const int&。

通常我们认为 T 会被推导为传进来参数的类型（比如上面的例子），但并不总是这样，实际上 T 的实际类型不仅取决于 expr， 而且也跟 ParamType 的形式有关系。具体分为一下三种情况：

1.  ParamType 是一个指针或引用，却不是普适引用（universal reference）。
2.  ParamType 是一个普适引用。
3.  ParamType 既不是指针也不是引用（通常是传值 pass-by-value）。

----------------------------

下面分别对三种情况讨论。

### 1) ParamType 是一个指针或引用，却不是普适引用（universal reference）。

在这种情况下，所有的推导属于符合直觉的正常情况——推导出的 T 是去掉对应的应用和指针后的类型。
如 

```c

// 引用的情况
template<typename T>
void f( T& param);
...
int x = 27;
const int& rx = x;
...
f(x);  // T 为 int
f(rx); // T 为 const int


// 常引用的情况
template<typename T>
void f( const T& param);
...
int x = 27;
const int& rx = x;
...
f(x);  // T 为 int
f(rx); // T 为 int


// 指针的情况
template<typename T> 
void f(T* param);   
int x = 27;          
const int *px = &x;  
 
f(&x);                   // T 为 int
f(px);                   // T 为 const int


```

所有这些都是程序员已经熟知的典型情况。


### 2) ParamType 是一个普适引用。

典型的普适引用如下所示：

```c
template<typename T>
void f(T&& param);
```

当 expr (用于推导的表达式) 为左值时， **T 和 ParamType (此例中的 T&&) 被推导为左值引用**。这是一个不寻常的特例，特殊在两点：

1. 这是模板类型推导中唯一一个 T 会被推导为引用的情况。
2. 虽然 ParamType 被声明为右值引用 (T&&)，在这里却会被推导为左值引用。

```c
（接上例）
int x = 27;              
const int cx = x;        
const int& rx = x;    
  
f(x);                    // x 是左值, T 为 int& , 参数类型也是 int&
f(cx);                   // cx 是左值, T 为 const int& , 参数类型也是 const int&
f(rx);                   // rx 是左值, T 为 const int&, 参数类型也是 const int&

f(27);                   // 27 是右值, T 为 int , 参数类型是 int&&
```

第 24 条解释了为什么会这样，这里我们只需记住，形如上例的普适引用推导规则与典型的推导规则不同，对左值和右值会推导出不同的结果，就可以了。


### 3)  ParamType 既不是指针也不是引用（通常是传值 pass-by-value）

典型的传值如下所示：

```c
template<typename T>
void f(T param);
```

传值意味着新的对象被创建。由于传进来的对象被用于构造新的对象，其 cvr 属性 (const, volatile, ref) 在传递过程中被抛弃了。

这是最单纯的一种推导，不过还是举个稍微复杂点的例子说明一下。

```c
template<typename T> 
void f( T param);         // 传值
const char* const ptr = "Fun with pointers";      // 指针及其内容均为 const

f(ptr);                  // 此时 T 为 const char*
```

上例中，由于指针 ptr 被复制给新的指针 param，指针本身的 const 丢了，而指向的类型在这个过程中则保持不变。


## 数组作为参数的推导

为了保持与 C 的兼容性，数组在大部分情况下是可以退化成指针的。

```c
const char name[] = "J. P. Briggs";  
const char * ptrToName = name;       
```

那么当参与模板参数推导的时候会发生什么？

```c
template<typename T>
void f(T param);

f(name);              // 会推导成什么呢？
```

首先我们知道函数的参数是没有数组类型这个说法的，即使像下面这样写

```c
void myFunc(int param[]);
```

实际上效果也跟下面这样写是等价的：

```c
void myFunc(int* param);
```

正如我们知道的那样，数组在通过参数传进函数时，退化成了指针。
由此可知，在上面的模板例子中，当推导 T 时，模板参数被推导为了 const char*。

**然而**，虽然函数不能把参数声明为数组类型（总是会退化成指针），但却可以声明为**数组的引用**！

如果把上面的模板改为这样：

```c
template<typename T>
void f(T& param);     // 传引用
```

再以同样的方式调用，


```c
f(name);    
```

在这种情况下，T 就会被推导为数组的类型！这个数组类型包括了数组的长度，所以在上面的例子里，T 会被推导为 const char [13]，f 的参数类型是 const char (&) [13]（数组的引用）。写到这儿，Scott Meyers 不由得感叹，这语法，实在是有点矬啊。

利用这个推导数组类型的能力，可以写一个模版函数，用来返回数组的尺寸：

```c
template<typename T, std::size_t N>                
constexpr std::size_t arraySize( T (&)[N]) noexcept  
{                                                  
    return N;                                         
}                                                   
```

注意 constexpr 把函数的返回值变为编译期常量，这样就可以用在另一个数组的定义里。

```c
int keyVals[] = { 1, 3, 7, 9, 11, 22, 35 };
int mappedVals[arraySize(keyVals)];
```

## 函数作为参数的推导

当把函数作为参数传递时，函数会和数组一样退化为指针。例子很寻常，就不举了。


## 要点归纳

- 函数类型推导时，如果参数声明为引用（引用传递），那么模板参数 T 为对应的非引用类型
- 传普适引用时，参数如果是左值，会作为一个特例对待
- 传值时， cvr 修饰符（在传递过程中）都会被忽略
- 传数组或函数时，会退化成指针，除非参数声明为引用类型

