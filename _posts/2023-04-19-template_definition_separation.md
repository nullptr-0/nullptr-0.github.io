---
layout: post
title: "C++模板的声明和实现的分离"
date:   2023-04-19
tags: [C++,Generic Programming,Compilation And Linkage]
comments: true
author: nullptr-0
---

> 使用内联函数在头文件中实现模板是目前使用它们的唯一通用方式。

对于一个模板，它的声明和实现不能分开置于头文件和源文件(即 .cpp)中。要解释这一点，我们需要先大致了解C++源代码是如何编译和链接的。

### C++源代码的编译和链接

1. 只有源文件会被编译，而**头文件不会被直接编译**。

   (1) 在进行编译之前，C++编译器会对源代码进行预处理，其中就包括将源文件中的 `#include<XXX.h>` 替换为这个头文件的具体内容。

   (2) 在编译时，只有经过预处理的源文件会被编译为目标文件，头文件会被忽略掉。

2. **编译隔离**

   (1) C++编译器会做"编译隔离"，这意味着它会分开编译每一个源文件。当编译一个源文件时，编译器不会关注其它源文件。

   (2) 当一个源文件中有无法解析的外部符号——例如从其它源文件导入的函数或类——时，编译器并不关心它们的具体内容，它只会把这些符号标记下来并继续编译。

   (3) 在完成编译后，链接器会将所有目标文件链接成一个可执行文件。所以链接器知道整个可执行文件的全貌。**每一个目标文件中都有一个符号表**，其中记录了这个目标文件中实现的符号。所以当链接器遇到一个无法解析的外部符号时，它会尝试在其它目标文件的符号表中查找这个符号。如果在任何目标文件的符号表中都无法找到这个符号，链接器就会抛出一个"Undefined reference to XXX"的错误，其中"XXX"代表这个符号。

   (4) 概括起来，**编译隔离**意味着**编译器只关注当前正在编译的文件**，而**链接器才关注整体**。

3. 小结

   - 预处理器将源文件中的`#include<XXX.h>`替换为头文件的内容。
   - 编译器逐个对源文件分别进行编译。
   - 链接器在符号表中查找符号并将所有目标文件链接成可执行文件。

### 编译器对模板的处理

模板只是源代码的样式。编译器并不会在目标文件中生成相应的机器码。编译器只会为这个模板的实例生成机器码。
<br />
例如，对于如下没有显式实例化的模板:

```c++
// example.cpp
template <typename T>
class temp
{
    // other code
}
```

当将 `example.cpp` 编译成目标文件后，目标文件里实际上并**没有**关于 `temp` 类的**任何信息**，因为编译器不会为类模板的原型生成机器码。
<br />
只有将类模板实例化后，编译器才会为这些实例生成机器码:

```c++
// example.cpp
template <typename T>
class temp
{
    // other code
}

// inst.cpp
temp<int> inst;
```

编译器会为 `inst` 生成机器码，进而链接器就可以在 `inst.cpp` 对应的目标文件中找到 `inst` 的信息。

### 编译器对模板实例的处理

```c++
// example.cpp
template <typename T>
class temp
{
    public:
    	std::vector<T> storage;
    	void func(T input) {} // implementation
}

// inst.cpp
temp<int> inst;
```

当编译器处理语句 `temp<int> inst;`时，它会先在内部生成类模板 `temp` 的一个实例，其中 `T` 使用实例化的数据型别 `int` 替换:

```c++
temp<int> test
{
    public:
    	std::vector<int> storage;
    	void func(int input) {}
}
```

然后编译器会编译这个实例，生成相应的机器码，并将它的信息插入到符号表中，并进行其它操作等等。这样之后，链接器就可以在 `inst.cpp` 对应的目标文件中找到 `test` 。

### 分离声明和实现到头文件和源文件中的问题

假设你在头文件中声明了一个类模板，而在源文件中实现它:

```c++
// temp.h
template <typename T>
class temp
{
    public:
    	std::vector<T> storage;
    	void func(T input); // Only declaration, no implmentation 
}

// temp.cpp
template <typename T>
void temp::func(T input) 
{
    // implementation
}
```

并接着在 `main.cpp` 包含 `temp.h` 并实例化 `temp`:

```c++
#include "temp.h"
int main()
{
    temp<int> test;
    return 0;
}
```

这种情况下，编译仍然能成功，但链接器会抛出"Undefined reference to temp\<int\>"的错误，因为链接器无法确定"temp\<int>\"是什么。
<br />
真是无语 `temp` 类的实现不在 `temp.h` 中，所以即使在 `main.cpp` 中包含了这个头文件，也只是导入了 `temp` 类的接口，而非这个类的全部信息。所以编译器会认为 `temp<int>` 是一个外部符号，而将它留给链接器处理。而由于编译器不会为 `temp.cpp` 中的内容生成符号表，链接器将无法找到 `temp<int>` 的信息，所以它会抛出未定义引用的错误。

### 分离声明和实现的方法

1. 将模板的声明和实现合并到一个头文件中:

   ```c++
   // temp.h
   template <typename T>
   class temp
   {
       public:
       	std::vector<T> storage;
       	void func(T input); // Only declaration, no implmentation 
   }
   
   // Also in temp.h
   template <typename T>
   void temp::func(T input) 
   {
       // implementation
   }
   ```

   并接着在 `main.cpp`中:

   ```c++
   #include "temp.h"
   int main()
   {
       temp<int> test;
       return 0;
   }
   ```

   这时 `main.cpp` 仍然包含 `temp.h` ，但现在 `temp` 的声明和实现都被导入进来了。所以编译器会在编译时将它视为内部（局部）符号，并生成一个使用 `int` 型别的 `temp` 实例，并在 `main.cpp` 对应的目标文件中生成包含 `temp<int>` 的机器码和符号表。之后链接器就可以在这个目标文件中找到相应的内容了。

2. 将模板的声明和实现分离到两个头文件中:

   ```c++
   // temp.h
   template <typename T>
   class temp
   {
       public:
       	std::vector<T> storage;
       	void func(T input); // Only declaration, no implmentation 
   }
   
   // temp_impl.h
   template <typename T>
   void temp::func(T input) 
   {
       // implementation
   }
   ```

   并在 `main.cpp`中分别包含它们:

   ```c++
   #include "temp.h"
   #include "temp_impl.h"
   int main()
   {
       temp<int> test;
       return 0;
   }
   ```

   与上一种方法类似地， `temp` 的声明和实现都被导入到 `main.cpp` 中了。

3. 分离模板的声明和实现到头文件和源文件中，并在源文件中显式地实例化所有需要使用的模板实例:

   ```c++
   // temp.h
   template <typename T>
   class temp
   {
       public:
       	std::vector<T> storage;
       	void func(T input); // Only declaration, no implmentation 
   }
   
   // temp.cpp
   template <typename T>
   void temp::func(T input) 
   {
       // implementation
   }
   // explicitly instantiate template instances
   template class temp<int>; 
   template class temp<float>;
   ```

   如前所述，编译器不会编译模板而只编译它的实例。所以显式地实例化它也是一种可行的方法。在上面的示例中，当编译器处理最后两行时，它会将这两个实例编译到目标文件中，这样链接器就可以找到它们了。
<br />
   **需要注意的是：** 这种方法需要一次性实例化每个需要用到的实例。例如，如果只实例化了 `temp<int>` 和 `temp<float>` ，后续将无法使用如 `temp<string>` 的其它型别，只有那些先前实例化的型别是可以使用的。所以如果不知道需要用的什么型别，就必须采取前两种方法了。这就是为什么说在头文件中实现模板是*使用模板的**唯一**通用方式*。
