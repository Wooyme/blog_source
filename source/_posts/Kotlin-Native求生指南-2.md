---
title: Kotlin Native求生指南(2)
date: 2018-12-22 14:36:17
tags:
    - Kotlin
    - Native
    - C
---

# 上一篇地址
> https://wooyme.github.io/2018/12/22/Kotlin-Native%E6%B1%82%E7%94%9F%E6%8C%87%E5%8D%97-1/

# 前言
在**Kotlin Native求生指南1**中我们已经提到了KN的内存模型和GC。那么在本篇中，我们将一起了解一下KN的*Cinterop*。

# Cinterop
由于Kotlin Native内置的库实在是太少了，我们不得不大量依赖C的库，于是*Cinterop*就诞生了。*Cinterop*的作用就是把C的库翻译成可供Kotlin使用的*klib*。需要注意的是，现在Cinterop只支持C语言的库。不过实际上只要提供的头文件是C语言的即可，具体实现还是可以使用C++。

## Gradle
要使用Cinterop需要修改一下Gradle。
```Groovy
kotlin {
    targets {
        fromPreset(presets.linuxX64, 'linux')
        configure([linux]) {
            compilations.main.outputKinds('EXECUTABLE')
            compilations.main.entryPoint = 'main'
            //我添加的代码
            compilations.main{
                cinterops {
                    //包名称，需要与.def文件名对应,也可以增加一个参数修改def文件位置，一般不需要
                    libui {
                        packageName 'libui'
                    }
                }
            }
        }
    }
}
```
修改好gradle之后，应该就可以看到`interop/cinteropLibuiLinux`命令
## def
.def文件需要放在`src/nativeInterop/cinterop`里，并且要与gradle中的包名相同。比如这里就应该是`libui.def`。现在让我们来看看.def文件中应该是什么样的
```bash
#如果涉及到多个文件，都用空格隔开
#设置头文件位置
headers=/usr/include/ui.h
#包名
package=libui
#静态库文件名
staticLibraries = libui.a
#静态库路径
libraryPaths = /home/wooyme/Projects/libui/build/out
```
这样的.def文件是针对静态库使用的，如果要用动态链接库则需要改成下面这样
```bash
#如果涉及到多个文件，都用空格隔开
#设置头文件位置
headers=/usr/include/ui.h
#包名
package=libui
//编译选项，供编译器clang使用
compilerOpts.linux= -I/usr/include/
//链接选项，供链接器ldd使用
linkerOpts.linux = -L/usr/lib/x86_64-linux-gnu -lui
```
动态链接库的使用要比静态库更加复杂一点，其实就是开发C/C++经常会用到的这些参数，在linux上，我们可以使用**pkg-config**来获得这些参数,以libui为例
```bash
pkg-config --cflags --libs libui
```
然后**pkg-config**就会打印相应参数，非常的方便。  
一切都配置完之后，就只需要运行一下`interop/cinteropLibuiLinux`就可以了。

## 使用Interop
完成了上面的操作之后，就可以看到IDEA的`External libraries`里多了`xxx-cinterop-libui.klib`，里面就是从C语言转换过来的Kotlin Native Library。你可能会去试图打开里面的内容，看看都转换出了什么东西，但事实是你只能看到一堆被注释标记着的，莫名其妙的代码。  
**不要试图去看Cinterop转换后的knm文件**，如果想了解库中提供了哪些函数的话，正确的做法是去看原始的C语言头文件。  

下面我翻一下 Kotlin Native 在 github 上的 INTEROP.md 
>原文  
>https://github.com/JetBrains/kotlin-native/blob/master/INTEROP.md

### 基础类型
所有支持的C类型都会被转换成Kolin 类型

- 有符号、无符号整形和浮点型都会被转换到Kotlin上，并且保持相同的长度
- 指针和数组会被转换成CPointer<T>?
- 枚举类型可以根据def文件配置转换成Kotlin的枚举类或是整形
- 结构体会被转换成对应的类
- typedef会被转换成typealias

这里还有一段很难翻译，它引入了一个左值(`lvalue`),大意是Cinterop会给这些转换过来的类型加一个`${type}Var`,然后可以通过`${type}Var.value`调用这个类型本身的值，就像C++的Reference

### 指针类型
CPointer<T>的T必须是上述左值之一，比如说`struct S*`对应`CPointer<S>`,`int8_t*`对应`CPointer<int8_tVar>`,`char**`对应`CPointer<CPointerVar<ByteVar>`  
C语言的空指针，对应Kotlin的null, `CPointer<T>`可以使用所有kotlin的空安全操作`？:`,`?.`,`!!`等。比如
```Kotlin
val path = getenv("PATH")?.toKString() ?: ""
```
由于数组也被转换成`CPointer<T>`,所以`CPointer<T>`也支持`[]`操作,比如
```Kotlin
fun shift(ptr: CPointer<BytePtr>, length: Int) {
    for (index in 0 .. length - 2) {
        ptr[index] = ptr[index + 1]
    }
}
```
`CPointer<T>`的`.pointed`属性返回T,比如说`CPointer<ByteVar>`就返回`ByteVar`，而`ByteVar`就是就是`Byte`的左值，然后可以通过`ByteVar.value`得到这个`Byte`。左值又可以通过`.ptr`得到对应的`CPointer<T>`  
`void*`对应`COpaquePointer`,这是所有其他指针类型的父类，所以如果一个C函数的参数是`void*`，Kotlin中可以给他传任何`CPointer`  
指针类型转换可以使用`.reinterop<T>`,比如
```Kotlin
val intPtr = bytePtr.reinterpret<IntVar>()
//或
val intPtr: CPointer<IntVar> = bytePtr.reinterpret()
```
这个跟C语言里的强制转换是一样不安全的。  
同样的，`CPointer<T>`也可以通过`.toLong()`和`.toCPointer<T>`与`Long`互相转换,当然这也是不安全的。
### 内存分配
内存可以通过使用`NativePlacement`接口分配,如
```Kotlin
val byteVar = placement.alloc<ByteVar>()
//或
val bytePtr = placement.allocArray<ByteVar>(5)
```
最常用的是`nativeHeap`,它就和`malloc`与`free`一样
```Kotlin
val buffer = nativeHeap.allocArray<ByteVar>(size)
//use buffer.....
nativeHeap.free(buffer)
```
除此之外还可以使用`memScope`，我们在前一篇已经写过，这里就不再赘述。

### 传递指针
虽然C语言的指针对应的是`CPointer<T>`，但是C语言的函数中的指针参数对应的是`CValuesRef<T>`。当我们传入的参数是`CPointer<T>`时，一切都很正常，但是除了`CPointer<T>`，我们还可以传别的东西。设计`CValuesRef<T>`就是为了能够让我们在向函数传递数组的时候不需要显式分配一块内存，Kotlin为我们提供了这些方法。

- ${type}Array.toCValues(), type是Kotlin的基本类型
- Array<CPointer< T >?>.toCValues(), List<CPointer< T >?>.toCValues()
- cValuesOf(vararg elements: ${type}), type是基本类型或者指针
  
比如可以这么写
```C
//C语言
void foo(int* elements, int count);
...
int elements[] = {1, 2, 3};
foo(elements, 3);
```
```Kotlin
//Kotlin
foo(cValuesOf(1, 2, 3), 3)
```
### 关于字符串
不同于其他的指针，`const char*`被转换成Kotlin的String。除此之外，还有其他的一些工具可以让Kotlin的String与C语言的`const char*`进行转换

- fun CPointer< ByteVar >.toKString(): String
- val String.cstr: CValuesRef< ByteVar >
  要得到`.cstr`的指针，需要给`cstr`分配内存,比如
  ```Kotlin
  //官方这里是这么写的，但是不知道为什么我这里不行，我只能在memScope里调用.cstr.ptr
  val cString = kotlinString.cstr.getPointer(nativeHeap)
  //我的版本
  memScope{
      val cString = kotlinString.cstr.ptr
  }
  ```
在所有情况下，C语言的string都可以使用UTF-8编码  
如果不想要使用`const char*`到`String`的自动转换，可以在def文件里设置
```
noStringConversion = LoadCursorA LoadCursorW
```
调用LoadCursorA，LoadCursorW就变成了这样
```Kotlin
memScoped {
    LoadCursorA(null, "cursor.bmp".cstr.ptr)   // for ASCII version
    LoadCursorW(null, "cursor.bmp".wcstr.ptr)  // for Unicode version
}
```
### 传递、接收结构体
*这个其实不是很重要，因为大部分成熟一点的C的库都不会直接把结构体本身作为参数或是返回值,所以我就直接跳过了，关于这一段的原文也不长*

### 回调函数
要把Kotlin的函数变成C语言的函数指针需要使用`staticCFunction(::kotlinFunction)`,`staticCFunction`也接受**lambda**作为参数，顺便一提，`staticCFunction`依然继承了Kotlin的暴力美学，就像最初的40+参数的lambda一样。这里要注意的是，这个lambda必须是*静态*的，也就是不能用闭包，不能用*class*内的值，而且现在`staticCFunction`有个bug，不能直接使用`object`内的方法
```Kotlin
object A{
    fun foo(){}
}
//这样不行
staticCFunction(A::foo)
//这样OK
staticCFunction{
    A.foo()
}
```
如果callback没有运行在主线程里，那么就需要在callback开头加上`kotlin.native.initRuntimeIfNeeded()`,初始化Kotlin Native环境，这一点和Kotlin Native的多线程模型有关系。
#### 向callback传递数据
由于callback不能使用闭包这类的操作，传递参数就很重要了。大多数的C API都允许用户向callback传递一些指针，但是Kotlin的类并不能直接传递给C，所以就需要一些操作把类转换成指针。
```Kotlin
val stableRef = StableRef.create(kotlinReference)
val voidPtr = stableRef.asCPointer()
```
把指针转换成Kotlin类
```Kotlin
val stableRef = voidPtr.asStableRef<KotlinClass>()
val kotlinReference = stableRef.get()
```
这样两者的转换就完成了，要注意的是,创建的`stableRef`需要用`.dispose()`手动释放，以防止内存泄露。

### 可移植性
大意就是提供了一个`.convert<T>()`方法，能把基本类型转换成C函数需要的类型，跟`.toShort()`,`toUInt()`等方法有相同的作用。

# 小结
我实在是没想到一个*Cinterop*可以写这么长，本来还想在这一篇里把KN的*多线程模型*给写了，这么看来还是再写第三篇好了。这里稍微说一下我的*Cinterop*使用体验，总的来说就是功能实现的很完善了，但是用户体验真的很烂。其实最大的问题就是Kotlin是一门OOP的语言，但是C语言不是。其实很多C语言的库都有OOP的影子，无论是命名方式还是那些函数与Kotlin扩展函数别无二致的第一个参数，如果KN的*Cinterop*能够更加智能一点的话应该会有更好的体验。  
这一点，我也在Kotlin Native的github上提了issue，官方表示会考虑通过在def中增加一些选项的方式来修改*Cinterop*的行为
> https://github.com/JetBrains/kotlin-native/issues/2486

Kotlin Native，路途还很漫长