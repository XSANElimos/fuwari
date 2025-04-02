---
title: 跨DLL的C++单例模式
published: 2025-04-01
description: '在大型C++项目中，通常会将不同的功能模块分割成多个DLL。每个DLL在编译时都会有自己的全局变量和静态变量，若需要在多个DLL之间共享某些数据，或者确保某个资源（如单例对象）在多个DLL中只有一个实例时，如何实现跨DLL的单例模式便成为一个常见的挑战。'
image: ''
tags: ['C++']
category: '编程'
draft: false 
lang: ''
---

# 可用项目示例

[GitHub - xhawk18/singleton-cpp: cpp singleton works across dll/exe boundaries.](https://github.com/xhawk18/singleton-cpp)

```cpp
#include <mutex>
#include <memory>
#include <cstdlib>
#include <typeindex>
#include "ApiExport.h"

namespace BLXCORE
{

BX_API void GetSharedInstance(std::type_index const& typeIndex, void* (*getStaticInstance)(), void*& instance);

template<typename T>
class Singleton
{
public:
    Singleton() = default;
    static T& GetInstance()
    {
        static void* instance = nullptr;
        if (instance == nullptr) { GetSharedInstance(typeid(T), &GetStaticInstance, instance); }
        return *reinterpret_cast<T*>(instance);
    }
    // 获取单例的 shared_ptr，但不会自动释放
    static std::shared_ptr<T> GetInstancePtr()
    {
        static void* instance = nullptr;
        if (instance == nullptr) { GetSharedInstance(typeid(T), &GetStaticInstance, instance); }

        // 使用自定义删除器的 shared_ptr，避免自动释放
        return std::shared_ptr<T>(reinterpret_cast<T*>(instance), [](T*) {});
    }
    Singleton(Singleton const&)            = delete;
    Singleton& operator=(Singleton const&) = delete;

private:
    static void* GetStaticInstance()
    {
        static T t;
        return reinterpret_cast<void*>(&reinterpret_cast<char&>(t));
    }
};

```

```cpp
#include "pch.h"
#include "Include/Singleton.hpp"

#include <typeinfo>
#include <typeindex>
#include <unordered_map>

namespace
{
struct SingleTonHolder
{
    void*                       object_;
    std::shared_ptr<std::mutex> mutex_;
};
}   // namespace

// Global mutex
static std::mutex& getSingleTonMutex()
{
    // s_singleTonMutex is not 100% safety for multithread
    // but if there's any singleton object used before thread, it's safe enough.
    static std::mutex s_singleTonMutex;
    return s_singleTonMutex;
}

static SingleTonHolder* getSingleTonType(std::type_index const& typeIndex)
{
    static std::unordered_map<std::type_index, SingleTonHolder> s_singleObjects;

    // Check the old value
    std::unordered_map<std::type_index, SingleTonHolder>::iterator itr = s_singleObjects.find(typeIndex);
    if (itr != s_singleObjects.end()) { return &itr->second; }

    // Create new one if no old value
    std::pair<std::type_index, SingleTonHolder> singleHolder(typeIndex, SingleTonHolder());
    itr                              = s_singleObjects.insert(singleHolder).first;
    SingleTonHolder& singleTonHolder = itr->second;
    singleTonHolder.object_          = NULL;
    singleTonHolder.mutex_           = std::shared_ptr<std::mutex>(new std::mutex());

    return &singleTonHolder;
}

void BLXCORE::GetSharedInstance(std::type_index const& typeIndex, void* (*getStaticInstance)(), void*& instance)
{
    SingleTonHolder* singleTonHolder = NULL;
    {
        // Locks and get the global mutex
        std::lock_guard<std::mutex> myLock(getSingleTonMutex());
        if (instance != NULL) { return; }

        singleTonHolder = getSingleTonType(typeIndex);
    }

    // Create single instance
    {
        // Locks class T and make sure to call construction only once
        std::lock_guard<std::mutex> myLock(*singleTonHolder->mutex_);
        if (singleTonHolder->object_ == NULL)
        {
            // construct the instance with static funciton
            singleTonHolder->object_ = (*getStaticInstance)();
        }
    }

    // Save single instance object
    {
        std::lock_guard<std::mutex> myLock(getSingleTonMutex());
        instance = singleTonHolder->object_;
    }
}

```

## 1. 背景



在跨DLL的场景下，单例模式的实现需要特别注意内存模型、链接、符号的可见性等问题。本文将深入探讨如何在C++中实现一个跨DLL的单例模式。

## 2. 单例模式概述

单例模式（Singleton Pattern）确保一个类只有一个实例，并提供一个全局访问点来获取该实例。基本的单例模式实现如下：

### 基本实现（线程安全）

```cpp
class Singleton {
public:
    static Singleton& getInstance() {
        static Singleton instance; // C++11以后的静态局部变量
        return instance;
    }

private:
    Singleton() {}  // 私有构造函数
    ~Singleton() {}

    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
};
```

此实现确保了在多线程环境下，`getInstance()`​返回的实例是唯一的。C++11及以后，静态局部变量的生命周期由编译器管理，因此也能确保线程安全。

## 3. 跨DLL单例的挑战

### 3.1 跨DLL实例的问题

在跨DLL环境下，直接采用基本的单例模式会面临以下问题：

1. **多重实例问题**：如果每个DLL都定义了单例的静态局部变量，那么每个DLL都会创建各自的单例实例，导致实例不唯一。
2. **内存隔离**：不同DLL加载到不同的内存地址空间，因此如果DLL内有静态数据，另一个DLL可能无法访问到它。
3. **符号可见性**：在不同的DLL中，符号和符号表的管理方式不同，导致某些符号在某些DLL中不可见。

### 3.2 跨DLL共享单例的解决方案

要在跨DLL的环境下实现共享单例，可以考虑以下几种策略：

#### 3.2.1 使用全局指针

将单例的实例指针定义为DLL外部可见的全局变量，并在需要时通过该指针访问单例。

```cpp
// Singleton.h
#ifdef MY_DLL_EXPORTS
#define MY_DLL_API __declspec(dllexport)
#else
#define MY_DLL_API __declspec(dllimport)
#endif

class Singleton {
public:
    static Singleton* getInstance();
private:
    Singleton() {}
    static Singleton* instance;
};

// Singleton.cpp
#include "Singleton.h"

Singleton* Singleton::instance = nullptr;

Singleton* Singleton::getInstance() {
    if (instance == nullptr) {
        instance = new Singleton();
    }
    return instance;
}
```

#### 3.2.2 使用 `extern`​ 关键字共享单例实例

在一个DLL中定义一个静态实例，并使用`extern`​在其他DLL中共享该实例。通过`extern`​声明，多个DLL可以访问同一个实例。

```cpp
// Singleton.h
#ifdef MY_DLL_EXPORTS
#define MY_DLL_API __declspec(dllexport)
#else
#define MY_DLL_API __declspec(dllimport)
#endif

class Singleton {
public:
    static Singleton* getInstance();
private:
    Singleton() {}
    static Singleton* instance;
};

// Singleton.cpp
#include "Singleton.h"

Singleton* Singleton::instance = nullptr;

Singleton* Singleton::getInstance() {
    if (instance == nullptr) {
        instance = new Singleton();
    }
    return instance;
}
```

```cpp
// OtherDll.cpp
#include "Singleton.h"

extern Singleton* g_singleton;
```

#### 3.2.3 使用线程安全的单例实现

跨DLL时，若单例实例化时存在竞争条件，可以考虑使用线程安全的方式来创建单例。最常用的做法是通过`std::call_once`​来确保线程安全。

```cpp
#include <mutex>

class Singleton {
public:
    static Singleton* getInstance() {
        std::call_once(initFlag, []() {
            instance.reset(new Singleton());
        });
        return instance.get();
    }

private:
    Singleton() {}
    static std::unique_ptr<Singleton> instance;
    static std::once_flag initFlag;
};

std::unique_ptr<Singleton> Singleton::instance;
std::once_flag Singleton::initFlag;
```

在跨DLL的环境下，`std::once_flag`​可以确保单例只会被初始化一次，避免了多次初始化的问题。

#### 3.2.4 使用静态初始化和销毁

为了确保在程序退出时正确清理单例，可以使用静态局部变量的析构函数。利用C++的静态初始化和销毁机制，程序结束时单例对象会自动销毁。

```cpp
class Singleton {
public:
    static Singleton* getInstance() {
        static Singleton instance;
        return &instance;
    }
private:
    Singleton() {}
    ~Singleton() {}
};
```

这种方式依赖于C++的静态局部变量的初始化和销毁顺序来确保单例对象在程序生命周期内只会初始化一次。

### 3.3 解决DLL导入导出的问题

跨DLL使用单例时，通常会遇到不同编译单元之间的符号导入导出问题。为了处理这些问题，可以使用`__declspec(dllexport)`​和`__declspec(dllimport)`​来显式地控制符号的导入和导出。

```cpp
#ifdef MY_DLL_EXPORTS
#define MY_DLL_API __declspec(dllexport)
#else
#define MY_DLL_API __declspec(dllimport)
#endif
```

## 4. 常见问题与解决方案

### 4.1 为什么不能直接使用静态局部变量来实现跨DLL单例？

静态局部变量的生命周期在其定义的作用域内，因此如果每个DLL都定义了一个单例类，并使用静态局部变量，那么每个DLL都会创建一个独立的实例。这导致在多个DLL之间无法共享单例实例。

### 4.2 如何保证跨DLL的单例对象销毁顺序？

由于不同DLL之间的析构顺序不可控，跨DLL单例对象的销毁顺序可能会导致访问已销毁的单例对象。为避免此问题，最好使用智能指针或者手动管理单例对象的销毁顺序。

## 5. 总结

实现跨DLL的C++单例模式需要考虑DLL之间的内存隔离、符号可见性、线程安全等问题。通过全局指针、`extern`​共享、线程安全的初始化等方法，可以在多个DLL之间实现共享单例对象。特别需要注意的是，要管理好DLL加载和卸载时的资源释放，避免出现析构顺序问题。


