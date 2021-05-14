---
title: C++ Macro,  FUCK YOU!
date: 2019-06-04 21:10:26
tags:
    - C++
---

# 前言
这几天一直在搞Javafx和Graalvm。主要是Graalvm在Windows上编译native image问题太tm多了。主要问题都出在JNI的部分，现在已经发现有数组访问异常，浮点数传递不正确的bug。要是有人想尝试一下native image在windows上的效果的话，最好先从不涉及jni的库开始。  
那么，这篇的主题其实跟graalvm没什么关系，开篇只是发个牢骚。当然了，要是没有Graalvm的这些问题，我也不至于把JavaFx的C++实现全给看一遍，也就不会有这篇博客了。

# 宏(Macro)
虽然我第一次写C++的程序是在2012年，但是其实我的C++是写的很烂的。所以在很长的一段时间里，宏在我的认知里无非是做一些编译期的判断，像是实现DEBUG和RELEASE分支，或是防止头文件被重复引用这样的小事情。唯一跟代码相关的，也就是会把常量定义在宏里。

因此Graalvm和JavaFx的一些宏操作彻底闪瞎了我的狗眼。先从Graalvm的说起吧。

## Code Generator( Macro Version! )

代码生成器这种东西各位应该或多或少都接触过。这个东西在Java里可以说是各大框架的杀手锏，你给个接口，他替你实现。在没有宏定义的Java里，框架用各种代码生成工具来实现代码生成，而在C/C++的世界里，代码生成器又可以是什么样子的呢

先来看一段Graalvm的官方demo

```C
#include <polyglot.h>

#include <stdlib.h>
#include <stdio.h>

struct Point {
    double x;
    double y;
};

POLYGLOT_DECLARE_STRUCT(Point)

void *allocNativePoint() {
    struct Point *ret = malloc(sizeof(*ret));
    return polyglot_from_Point(ret);
}

void *allocNativePointArray(int length) {
    struct Point *ret = calloc(length, sizeof(*ret));
    return polyglot_from_Point_array(ret, length);
}

void freeNativePoint(struct Point *p) {
    free(p);
}

void printPoint(struct Point *p) {
    printf("Point<%f,%f>\n", p->x, p->y);
}
```

实现了什么功能不重要，重要的是其中的`POLYGLOT_DECLARE_STRUCT`和`polyglot_from_Point`以及` polyglot_from_Point_array`这三个东西。

首先我们可以确定，`Point`这个东西是我们自己定义的，所以显然这两个带Point的函数也不可能是由`polyglot.h`提供的。那么为什么在我们也没有定义这两个函数的情况下这段程序可以正常编译呢？这就要靠神奇的(*FUCKING UNREADABLE*)宏来实现了。

按照国际惯例(幸好你们还有个国际惯例)宏是全部大写的，所以`POLYGLOT_DECLARE_STRUCT`就一定是个宏了。于是我在`polyglot.h`的最底下找到了关于这个的定义。

```C
#define POLYGLOT_DECLARE_STRUCT(type) __POLYGLOT_DECLARE_GENERIC_TYPE(struct type, type)

#define __POLYGLOT_DECLARE_GENERIC_TYPE(typedecl, typename)                                                                                          \
  __POLYGLOT_DECLARE_GENERIC_ARRAY(typedecl, typename)                                                                                               \
                                                                                                                                                     \
  __attribute__((always_inline)) static inline typedecl *polyglot_as_##typename(void *p) {                                                           \
    void *ret = polyglot_as_typed(p, polyglot_##typename##_typeid());                                                                                \
    return (typedecl *)ret;                                                                                                                          \
  }                                                                                                                                                  \
                                                                                                                                                     \
  __attribute__((always_inline)) static inline void *polyglot_from_##typename(typedecl * s) {                                                        \
    return polyglot_from_typed(s, polyglot_##typename##_typeid());                                                                                   \
  }

  #define __POLYGLOT_DECLARE_GENERIC_ARRAY(typedecl, typename)                                                                                         \
  __attribute__((always_inline)) static inline polyglot_typeid polyglot_##typename##_typeid() {                                                      \
    static typedecl __polyglot_typeid_##typename[0];                                                                                                 \
    return __polyglot_as_typeid(__polyglot_typeid_##typename);                                                                                       \
  }                                                                                                                                                  \
                                                                                                                                                     \
  __attribute__((always_inline)) static inline typedecl *polyglot_as_##typename##_array(void *p) {                                                   \
    void *ret = polyglot_as_typed(p, polyglot_array_typeid(polyglot_##typename##_typeid(), 0));                                                      \
    return (typedecl *)ret;                                                                                                                          \
  }                                                                                                                                                  \
                                                                                                                                                     \
  __attribute__((always_inline)) static inline void *polyglot_from_##typename##_array(typedecl *arr, uint64_t len) {                                 \
    return polyglot_from_typed(arr, polyglot_array_typeid(polyglot_##typename##_typeid(), len));                                                     \
  }
```

最后发现`POLYGLOT_DECLARE_STRUCT(Point)`的`Point`就是宏里的`typename`。这就破案了，这段宏为我们定义了`polyglot_from_##typename`和`polyglot_from_##typename##_array`这两个方法，而其中的`typename`在编译期就被`Point`替换掉了（我看你是在为难我IDE）。

## 骚气满满的EventBus
上一个还是很容易理解的，毕竟只是使用了`##var`这么个特性罢了。不知道的时候觉得很神奇，知道了之后也就是这么一回事。而接下来这个是让我着实惊讶了许久。
先看一段代码
```C++
/*
 * Class:     com_sun_glass_ui_win_WinWindow
 * Method:    _setView
 * Signature: (JJ)Z
 */
JNIEXPORT jboolean JNICALL Java_com_sun_glass_ui_win_WinWindow__1setView
    (JNIEnv * env, jobject jThis, jlong ptr, jobject view)
{
    ENTER_MAIN_THREAD()
    {
        GlassWindow *pWindow = GlassWindow::FromHandle(hWnd);

        if (activeTouchWindow == hWnd) {
            activeTouchWindow = 0;
        }
        pWindow->ResetMouseTracking(hWnd);
        pWindow->SetGlassView(view);
        // The condition below may be restricted to WS_POPUP windows
        if (::IsWindowVisible(hWnd)) {
            pWindow->NotifyViewSize(hWnd);
        }
    }
    GlassView * view;
    LEAVE_MAIN_THREAD_WITH_hWnd;

    ARG(view) = view == NULL ? NULL : (GlassView*)env->GetLongField(view, javaIDs.View.ptr);

    PERFORM();
    return JNI_TRUE;
}
```
做过安卓的朋友肯定很熟悉这个套路，修改UI的事情只能在UI线程里做，而对于大部分程序来说UI线程就是主线程，所以这里的`ENTER_MAIN_THREAD`看着也很正常。

但是问题来了，这个乍一看很正常的程序，仔细想想怎么都不对劲啊。参数里传进来了一个`view`，为什么下面还能再声明一个？这个看着很像是函数体的`ENTER_MAIN_THREAD()`里面的那个`hWnd`是哪里跑出来的？最底下的`ARG`和`PERFORM`这两个又是干嘛的？

当我第一次意识到这些问题的时候，我甚至开始怀疑这是不是C++的新特性，C++式fp难道是长这个样子的。

在深入♂了解了之后，我才发现，都是假的。这种漂亮的代码都是假的，是特技，这个世界上根本就没有这种代码。所以现在让我们来看看它是怎么实现的。
首先是`ENTER_MAIN_THREAD`，既然有国际惯例在，那这个东西想必是个宏了

```C++
#define ENTER_MAIN_THREAD() \
    class _MyAction : public Action {    \
        public: \
                virtual void Do()

```

这tm就是它的定义。你可能会说，这tm括号都没闭合呢！是啊，所以它还有下半个，`LEAVE_MAIN_THREAD_WITH_hWnd`
```C++
#define LEAVE_MAIN_THREAD_WITH_hWnd  \
    HWND hWnd;  \
     } _action;  \
    ARG(hWnd) = (HWND)ptr;
```
万万没想到，这几天otto上当了，我也上当了。所以这部分真实情况是长这个样子的
```C++
class _MyAction:public Action {
    public:
        virtual void Do()
         {
            GlassWindow *pWindow = GlassWindow::FromHandle(hWnd);

            if (activeTouchWindow == hWnd) {
                activeTouchWindow = 0;
            }
            pWindow->ResetMouseTracking(hWnd);
            pWindow->SetGlassView(view);
            // The condition below may be restricted to WS_POPUP windows
            if (::IsWindowVisible(hWnd)) {
                pWindow->NotifyViewSize(hWnd);
        }
    }
    HWND hWnd;
    GlassView * view;
} _action;
ARG(hWnd) = (HWND)ptr;
ARG(view) = view == NULL ? NULL : (GlassView*);
 PERFORM();
```
行吧，你还真是`Runnable`转世到了C++上啊。借着一手宏定义给我整成了lambda的样子。

到了这里，就只有`ARG`和`PERFORM()`还没有解释清楚了。让我们来看看它们两个的定义。
```C++
#define ARG(var) _action.var

#define PERFORM() GlassApplication::ExecAction(&_action)
```
于是，最底下这部分就变成了
```C++
_action.hWnd = (HWND)ptr;
_action.view =  view == NULL ? NULL : (GlassView*);
GlassApplication::ExecAction(&_action);
```
顺便一提，`GlassApplication::ExecAction`是这样的
```C++
void GlassApplication::ExecAction(Action *action)
{
    if (!pInstance) {
        return;
    }
    ::SendMessage(pInstance->GetHWND(), WM_DO_ACTION, (WPARAM)action, (LPARAM)0);
}
```
搞过Win32编程的朋友一定很熟悉这个东西，没搞过的话理解成在EventBus上发送消息就行了。

所以到了最后，这个看着像是线程切换的操作只是把`_action`发送给了主线程而已。高，真的是高。


# 小结
宏这个东西真是写者爽翻天，看者流眼泪。这么一想，好像刚开始写C++的时候就有用宏会影响代码可读性这么一说。只是这几年接触C++的少，没见过什么市面，把宏想的太简单了。如今见过这种骚操作之后，我想说，*加入光荣的进化吧*