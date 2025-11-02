# UEC++ `DynamicCastSharedPtr` 与 RTTI

## 概述

Unreal Engine C++ (UEC++) 默认禁用 C++ 标准库的运行时类型信息 (RTTI) 功能，以优化性能、减少内存占用并确保跨平台兼容性。然而，UEC++ 的智能指针 `TSharedPtr` 仍然提供了 `DynamicCastSharedPtr` 函数，能够实现安全的运行时类型检查。这并非依赖于 C++ 原生 RTTI，而是通过 Unreal Engine 自身实现的一套轻量级、可选的类型信息系统。

## `DynamicCastShared-Ptr` 的工作原理

`DynamicCastSharedPtr` 的行为取决于所处理的类类型：

1.  **对于 `UObject` 派生类**：
    *   `UObject` 及其派生类是 UE 中最核心的类型，它们通过 Unreal Header Tool (UHT) 自动生成一套强大的反射系统。
    *   `Cast<T>()` 函数是用于 `UObject` 的主要类型转换机制，它利用 UHT 生成的元数据进行高效的运行时类型检查。
    *   `DynamicCastSharedPtr` 通常不直接用于 `UObject`，因为 `TSharedPtr` 主要用于非 `UObject` 的普通 C++ 对象。

2.  **对于非 `UObject` 的普通 C++ 类**：
    *   `TSharedPtr` 主要用于管理不继承自 `UObject` 的普通 C++ 对象。
    *   为了让 `DynamicCastSharedPtr` 对这些类生效，开发者需要**手动将这些类注册到 UE 的类型系统中**。这通过一对宏实现：`DECLARE_TYPE_HIERARCHY` 和 `IMPLEMENT_TYPE_HIERARCHY`。

### 内部机制：`DECLARE_TYPE_HIERARCHY` 和 `IMPLEMENT_TYPE_HIERARCHY`

当你为非 `UObject` 类使用这些宏时，它们会在你的类中注入必要的代码，从而构建一个“穷人版”的 RTTI：

*   **`DECLARE_TYPE_HIERARCHY(TypeName, ParentTypeName)`** (在头文件中使用):
    *   声明一个静态的 `TypeId`，作为该类的唯一标识符。
    *   声明一个静态函数 `GetStaticTypeId()`，返回该类的静态 `TypeId`。
    *   **关键是，它声明了一个虚函数 `GetDynamicTypeId()`**。这个虚函数在运行时被调用，返回对象的真实（动态）类型 ID。
    *   要求类必须至少有一个虚函数（通常是虚析构函数），以确保 `GetDynamicTypeId()` 能够被正确地多态调用。

*   **`IMPLEMENT_TYPE_HIERARCHY(TypeName, ParentTypeName)`** (在 .cpp 文件中实现):
    *   定义并初始化 `DECLARE_TYPE_HIERARCHY` 中声明的静态成员和函数。
    *   记录父类的 `TypeId`，从而在内部建立起一个继承链，供运行时查询。

### `DynamicCastShared-Ptr` 的执行流程

当调用 `DynamicCastSharedPtr<TargetType>(SourcePtr)` 时，其内部大致执行以下步骤：

1.  **获取运行时类型 ID**：通过 `SourcePtr->GetDynamicTypeId()` 虚函数调用，获取源对象在运行时的真实类型 ID。
2.  **获取目标类型 ID**：获取 `TargetType::GetStaticTypeId()`，即目标类型的静态类型 ID。
3.  **检查继承关系**：UE 的类型系统利用这两个 ID，沿着 `IMPLEMENT_TYPE_HIERARCHY` 建立的父类信息链，检查源对象的运行时类型是否是目标类型本身，或者目标类型的派生类。
4.  **返回结果**：
    *   如果检查通过，转换成功，返回一个新的、指向同一对象的 `TSharedPtr<TargetType>`。
    *   如果检查失败，返回一个空的 `TSharedPtr`。

### 示例

假设有如下类结构，并希望使用 `DynamicCastSharedPtr`：

**Animal.h**
```cpp
#pragma once

#include "CoreMinimal.h" // 包含 UE 核心模块

class FAnimal
{
public:
    virtual ~FAnimal() {} // 必须有虚函数以支持多态

    // 声明类型层次结构，FAnimal 是基类，没有父类
    DECLARE_TYPE_HIERARCHY(FAnimal, nullptr)
};

class FDog : public FAnimal
{
public:
    void Bark() { /* ... */ }

    // 声明类型层次结构，FDog 的父类是 FAnimal
    DECLARE_TYPE_HIERARCHY(FDog, FAnimal)
};
```

**Animal.cpp**
```cpp
#include "Animal.h"

// 实现类型层次结构
IMPLEMENT_TYPE_HIERARCHY(FAnimal, nullptr)
IMPLEMENT_TYPE_HIERARCHY(FDog, FAnimal)
```

**使用示例**
```cpp
#include "Animal.h" // 包含上述头文件

// ... 在某个函数中 ...
TSharedPtr<FDog> MyDog = MakeShared<FDog>();
TSharedPtr<FAnimal> MyAnimal = MyDog; // 隐式向上转型

// 使用 DynamicCastSharedPtr 进行安全的向下转型
TSharedPtr<FDog> MyDogDynamicCasted = DynamicCastSharedPtr<FDog>(MyAnimal);

if (MyDogDynamicCasted.IsValid())
{
    // 转换成功，MyAnimal 确实指向一个 FDog 对象
    MyDogDynamicCasted->Bark();
}
else
{
    // 转换失败，MyAnimal 可能指向的不是 FDog 或其子类的对象
}
```

通过这种方式，UEC++ 在禁用标准 RTTI 的前提下，为 `TSharedPtr` 管理的普通 C++ 对象提供了灵活且可控的运行时类型检查能力。
