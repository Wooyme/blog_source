---
title: See what's comming in Kotlin 1.3-M1 译文
date: 2018-08-11 19:41:08
tags: Kotlin
---
>原文在:
https://blog.jetbrains.com/kotlin/2018/07/see-whats-coming-in-kotlin-1-3-m1/   


前言
====
今天，在一长串的关于Kotlin 1.2.x的更新之后，是时候看看Kotlin 1.3会带来什么。我们很高兴的宣布Kotlin 1.3的尝鲜体验版Kotlin 1.3-M1正式发布。  
Kotlin 1.3相比之前版本有许多的进步，其中包括 *完全体的协程*，*实验版的无符号类型*(Java至今没有的东西),*inline class* 以及其他更多特性。  
这里要感谢为我们的新版本贡献代码的社区群众:Raluca Sauciuc, Toshiaki Kameyama, Leonardo Lopes, Jake Wharton, Jeff Wright, Lucas Smaira, Mon_chi, Nico Mandery, Oskar Drozda.  
>完整的ChangeLog  
https://github.com/JetBrains/kotlin/blob/1.3-M1/ChangeLog.md

稳定版的协程(1.3之前一直是实验版)
=====
终于，在1.3中协程不再是实验性的。无论是语法糖还是标准库都将趋于稳定并且保持向后兼容。自从1.1版本加入协程之后，协程这一特性一直保持着显著的提高。
>几个重要的特性  
KT-16908 Support callable references to suspending functions  
KT-18559 Serializability of all coroutine-related classes

现在，我们简化了协程的核心API，并且去掉了`experimental`包。同时，我们还在协程的跨平台性中做了许多工作，包括对基于Kotlin/Native（Kotlin的LLVM版本)的IOS支持

### 转战新协程
就像我们之前说的一样，所有与协程有关的函数都都已经丢掉了`experimental`的包名。同时，`buildSequence`和`buildIterator`函数也放到了他们在`kotlin.sequences`包中常驻的地方。  
在语言的层面上，我们仍然使用`suspend`关键字来支持协程并且所有的规则几乎与实验版中的规则一致。  
我们简化了稳定版中的`Continuation<T>`接口。现在它只保留了 `resumeWith(result: SuccessOrFailure<T>)`这一个成员函数。原先的`resume(value: T)`和`resumeWithException(exception: Throwable)`现在以扩展的形式出现。这个改动只影响了那些少数自己定义协程构造器，那些将回调函数包装成挂起函数(suspending functions)的代码大多数不会发生改变。比如说，为类`CompletableFuture<T>`定义挂起函数`await()`还是会像之前一样。
```Kotlin
suspend fun <T> CompletableFuture<T>.await(): T = suspendCoroutine { cont ->
    whenComplete { value, exception ->
        when {
            exception != null -> cont.resumeWithException(exception)
            else -> cont.resume(value)
        }
    }
}
```
稳定版的协程采用了不同的二进制接口，它们并不能与实验版的协程二进制兼容。为了确保代码能够平稳的转移，我们将在1.3中增加一个兼容层，并且实验版中的类都将保留在标准库中。在Kotlin/JVM中使用Kotlin 1.1-1.2的已经编译好的代码都能在Kotlin 1.3中运行。  
但是Kotlin 1.3中不提供任何调用1.2(原文中写了1.3，应该是写错了)版本中编译好的实验性协程的支持。如果想要在1.3稳定版协程中使用旧版本协程的库，你需要在1.3版本下重新编译它们。这只是一个暂时的问题，我们会尽快处理。(JetBrains团队的尽快，往往是真的很快)  
我们还将提供`kotlinx.coroutines`库的`x.x.x-eap13`版本。  
IDE将提示你转移到新的协程上去。我们会在1.3正式版发布前进一步扩大协程的使用范围。

一些新特性
=====
更重要的特性会出现在实验性部分，这里只提一些为大家带来便利的小特性

### 捕获when中的参数
这段懒得翻译了，就看一下代码吧。已经非常明了了
```Kotlin
fun Request.getBody() =
    when (val response = executeRequest()) {
        is Success -> response.body
        is HttpError -> throw HttpException(response.status)
    }
```
### 伴生接口中的@JvmStatic和@JvmField
```Kotlin
interface Service {
    companion object {
        @JvmField
        val ID = 0xF0EF

        @JvmStatic
        fun create(): Service = ...
    }
}
```
Kotlin中的接口现在可以把静态成员暴露给Java(捞你Java一手)。

### 可嵌套的注解声明
在Kotlin 1.3之前，注解类不能拥有类体(bodies)。1.3版本放宽了这一限制，现在我们允许注解类拥有嵌套类，接口和伴生对象
```Kotlin
annotation class Outer(val param: Inner) {
    annotation class Inner(val value: String)
}

annotation class Annotation {
    companion object {
        @JvmField
        val timestamp = System.currentTimeMillis()
    }
}

```
### 函数类型可以有更多参数
现在一个函数类型，可以拥有超过22个参数了！(Kotlin的一个梗，程序员的暴力美学)。我们现在将上限提高到了JVM的极限——255。如果你想知道我们是怎么做到在不定义额外233个类的情况下实现这个功能的话，请看这里
>https://github.com/Kotlin/KEEP/blob/master/proposals/functional-types-with-big-arity-on-jvm.md

实验性特性
=====
就像协程已经证明了的一样，通过把EAP的重要特性设为实验性能帮助我们从社区中收集到可贵的反馈。我们将继续使用这个技术，让Kotlin的所有特性都经过实战的检验。Kotlin 1.3将带来三个激动人心的实验性特性。你需要明确选择使用这些特性，不然编译器会提示警告或者错误。

### 内联类
内联类能让你在不用真正创建一个类的情况下包装某个类型。
```Kotlin
inline class Name(internal val value: String)
```
当使用这样一个类的时候，编译器会内联它的内容，并且所有操作会直接作用在被包装的类本身。于是，就像下面这样
```Kotlin
val name: Name = Name("Mike")
fun capitalize(name: Name): Name = Name(name.value.capitalize())
```
编译结果会与下面的代码一样
```Kotlin
val name: String = "Mike"
fun capitalize(name: String): String = name.capitalize()
```
内联类与类型别名有些相似，但它们不是赋值兼容的。所以你不能把`String`赋值给`Name`,反之亦然。  
由于内联类实际并不存在，所以不能对它们使用`===`操作符。  
还有其他内联类产生包装器的地方，就像`Int`的装箱一样
```Kotlin
val key: Any = Name("Mike") // boxing to actual Name wrapper

val pair = Name("Mike") to 27 // Pair is a generic type, so Name is boxed here too
```
这个特性可以通过添加编译选项`-XXLanguage:+InlineClasses`来开启

### 无符号数字类型
内联类最明显的应用就是无符号类型。现在标准库已经加入了`UInt`，`ULong`，`UByte`和`UShort`。通过内联类，这些类型定义了自己的运算符，可以将存储的数值转化为无符号类型。
```Kotlin
val operand1 = 42
val operand2 = 1000 * 100_000

val signed: Int = operand1 * operand2
val unsigned: UInt = operand1.toUInt() * operand2.toUInt()
```
除了新的类型，我们还添加了一些新的语言特性来让它们变得特殊
- 允许在变长参数中使用无符号类型，这与其他内联类不用
```Kotlin
fun maxOf(vararg values: UInt): UInt { ... }
```

- 无符号类型的关键字
```Kotlin
val uintMask = 0xFFFF_FFFFu
val ulongUpperPartMask = 0xFFFF_FFFF_0000_0000uL
```

- 无符号常量
```Kotlin
const val MAX_SIZE = 32768u
//Koltin 1.3-M1暂不支持无符号常量的复杂表达式
const val MAX_SIZE_BITS = MAX_SIZE * 8u // Error in 1.3-M1
```

为了使用无符号类型，你需要选择启动它们
- either annotate the code element that uses unsigned types with the @UseExperimental(ExperimentalUnsignedTypes::class) annotation
- or specify the -Xuse-experimental=kotlin.ExperimentalUnsignedTypes compiler option.

新的标准库API
=======
现在让我们看看1.3中的新API

### SuccessOrFailure
内联类`SuccessOrFailure`是一个有效的判别函数执行成功或失败`Success T | Failure Throwable`的集合。它被用来捕获函数的执行结果无论成功或失败，以便于我们在之后的代码中处理它们。
```Kotlin
val files = listOf(File("a.txt"), File("b.txt"), File("42.txt"))
val contents: List<SuccessOrFailure<String>> = files.map { runCatching { readFileData(it) } }

println("map successful items:")
val upperCaseContents: List<SuccessOrFailure<String>> =
    contents.map { r -> r.map { it.toUpperCase() } }
upperCaseContents.printResults()

println()
println("map successful items catching error from transform operation:")
val intContents: List<SuccessOrFailure<Int>> =
    contents.map { r -> r.mapCatching { it.toInt() } }
intContents.printResults()
```
引入这个类最主要的原因是我们想要在新的协程接口中使用`resumeWith(result: SuccessOrFailure<T>)`而不是`resume(T)`和`resumeWithException(Throwable)`

### 多平台随机数生成器
没啥好说的，原本Kotlin/JVM的东西现在支持Kotlin的所有平台了

### Boolean类型的伴生对象
为Boolean加了个内容为空的伴生对象。今后可能有用，像各种类型比较，转换之类的地方。
### 基本类型伴生对象的常亮
- 为Byte, Short, Int, Long, Char几个类型加入了SIZE_BITS和SIZE_BYTES

- 为Char增加了MAX_VALUE('\u0000')和MIN_VALUE('\uFFFF')

总结
----
JetBrains的效率是真的挺高。2018年6月发布Kotlin 1.2.50，转眼1个月之后又发布了1.3-M1。1.3版本比较重要的就是协程不再带有实验性标志。同时最有趣的就是内联类了。
