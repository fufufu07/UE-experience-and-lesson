# Puerts 用法总结与易错点

> 本文档基于 puerts_unreal_demo 示例项目整理，涵盖 Puerts 在 Unreal Engine 中的核心用法和常见问题。

---

## 目录

1. [基本概念](#一基本概念)
2. [启动虚拟机](#二启动虚拟机)
3. [TypeScript 与 UE 交互](#三typescript-与-ue-交互)
4. [容器操作](#四容器操作)
5. [Delegate（委托）处理](#五delegate委托处理)
6. [蓝图 Mixin](#六蓝图-mixin)
7. [UI Widget 操作](#七ui-widget-操作)
8. [异步操作](#八异步操作)
9. [C++ 静态绑定](#九c-静态绑定)
10. [继承 UE 类](#十继承-ue-类已废弃)
11. [蓝图加载最佳实践](#十一蓝图加载最佳实践)
12. [易错点汇总](#十二易错点汇总)
13. [项目结构建议](#十三项目结构建议)

---

## 一、基本概念

**Puerts** 是一个在 Unreal Engine 中嵌入 JavaScript/TypeScript 虚拟机的插件，让开发者可以用 TypeScript 编写游戏逻辑，并与 C++/蓝图进行双向调用。

### 核心组件

| 组件 | 说明 |
|------|------|
| **FJsEnv** | JavaScript 虚拟机实例，可类比一个 Node.js 进程 |
| **V8** | 默认的 JavaScript 引擎后端 |
| **QuickJS** | 轻量级 JavaScript 引擎后端（可选） |
| **Node.js** | 支持 Node.js API 的后端（可选） |

### 工作原理

```
┌─────────────────┐      ┌─────────────────┐
│   TypeScript    │◄────►│   Unreal C++    │
│    (.ts/.js)    │      │   (UCLASS等)    │
└────────┬────────┘      └────────┬────────┘
         │                        │
         ▼                        ▼
┌─────────────────────────────────────────┐
│              Puerts (FJsEnv)            │
│         JavaScript 虚拟机 (V8)          │
└─────────────────────────────────────────┘
```

---

## 二、启动虚拟机

### 基本启动方式

**C++ 端 (TsGameInstance.cpp)**

```cpp
#include "TsGameInstance.h"

void UTsGameInstance::OnStart()
{
    Super::OnStart();
    
    // 创建虚拟机
    // 参数1: 模块加载器，指定 JavaScript 文件目录
    // 参数2: 日志器
    // 参数3: 调试端口（用于 VSCode 调试）
    GameScript = MakeShared<puerts::FJsEnv>(
        std::make_unique<puerts::DefaultJSModuleLoader>(TEXT("JavaScript")), 
        std::make_shared<puerts::FDefaultLogger>(), 
        8080
    );
    
    // 可选：等待调试器连接
    // GameScript->WaitDebugger();
    
    // 传递参数给 TypeScript
    TArray<TPair<FString, UObject*>> Arguments;
    Arguments.Add(TPair<FString, UObject*>(TEXT("GameInstance"), this));
    
    // 启动入口脚本
    GameScript->Start("QuickStart", Arguments);
}

void UTsGameInstance::Shutdown()
{
    Super::Shutdown();
    // 必须释放虚拟机
    GameScript.Reset();
}
```

### 头文件定义

```cpp
// TsGameInstance.h
#pragma once

#include "CoreMinimal.h"
#include "Engine/GameInstance.h"
#include "JsEnv.h"
#include "TsGameInstance.generated.h"

UCLASS()
class UTsGameInstance : public UGameInstance
{
    GENERATED_BODY()
    
public:
    TSharedPtr<puerts::FJsEnv> GameScript;
    
    virtual void OnStart() override;
    virtual void Shutdown() override;
};
```

### ⚠️ 易错点

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 虚拟机内存泄漏 | 忘记调用 `Reset()` | 在 `Shutdown()` 中必须调用 `GameScript.Reset()` |
| 调试端口冲突 | 多个虚拟机使用相同端口 | 每个虚拟机使用不同端口 |
| 脚本找不到 | 路径错误 | 确保 `DefaultJSModuleLoader` 路径正确指向 `Content/JavaScript` |

---

## 三、TypeScript 与 UE 交互

### 3.1 获取启动参数

```typescript
import * as UE from 'ue'
import {argv} from 'puerts';

// 获取 C++ 端传入的 GameInstance 对象
let gameInstance = (argv.getByName("GameInstance") as UE.GameInstance);

// 获取 World
let world = gameInstance.GetWorld();
```

### 3.2 创建和操作 UE 对象

```typescript
// 创建 UObject
let obj = new UE.MainObject();

// 访问属性
console.log("修改前:", obj.MyString);
obj.MyString = "Hello Puerts";
console.log("修改后:", obj.MyString);

// 调用方法
let sum = obj.Add(100, 300);
console.log("计算结果:", sum);

// 传递复杂类型
obj.Bar(new UE.Vector(1, 2, 3));
```

### 3.3 引用类型参数 (ref/out)

当 C++ 函数有 `引用参数` 或 `out 参数` 时，需要使用 `$ref`、`$unref`、`$set`：

```typescript
import {$ref, $unref, $set} from 'puerts';

// 创建引用包装
let vectorRef = $ref(new UE.Vector(1, 2, 3));

// 传递引用给函数（函数内部可以修改值）
obj.Bar2(vectorRef);

// 获取引用包装的实际值
let modifiedVector = $unref(vectorRef);
console.log("修改后的向量:", modifiedVector.ToString());

// 设置引用包装的值
$set(vectorRef, new UE.Vector(10, 20, 30));
```

### 3.4 静态方法调用

```typescript
// 调用蓝图函数库的静态方法
let name = UE.JSBlueprintFunctionLibrary.GetName();
let greeting = UE.JSBlueprintFunctionLibrary.Hello(name);
console.log(greeting);
```

### 3.5 枚举使用

```typescript
// 使用 UE 枚举
obj.EnumTest(UE.EToTest.V1);
obj.EnumTest(UE.EToTest.V13);

// 蓝图枚举
console.log(UE.Game.StarterContent.TestEnum.TestEnum.Blue);
console.log(UE.Game.StarterContent.TestEnum.TestEnum.Red);
```

### 3.6 默认参数

```typescript
// C++ 函数有默认参数时可以省略
obj.DefaultTest();
obj.DefaultTest("hello john");
obj.DefaultTest("hello john", 1024);
obj.DefaultTest("hello john", 1024, new UE.Vector(7, 8, 9));
```

### ⚠️ 易错点

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 引用参数不生效 | 未使用 `$ref` 包装 | 必须使用 `$ref`/`$unref`/`$set` |
| 类型断言失败 | 类型不匹配 | 确保 `as` 转换的类型正确 |
| 找不到方法 | 没有 UFUNCTION 标记 | C++ 方法必须有反射标记或使用静态绑定 |

---

## 四、容器操作

### 4.1 创建容器

```typescript
import * as UE from 'ue'

// 创建不同类型的 TArray
let boolArray = UE.NewArray(UE.BuiltinBool);
let intArray = UE.NewArray(UE.BuiltinInt);
let stringArray = UE.NewArray(UE.BuiltinString);
let actorArray = UE.NewArray(UE.Actor);
let vectorArray = UE.NewArray(UE.Vector);

// 创建 TSet
let stringSet = UE.NewSet(UE.BuiltinString);

// 创建 TMap
let strToIntMap = UE.NewMap(UE.BuiltinString, UE.BuiltinInt);
```

### 4.2 TArray 操作

```typescript
// 添加元素
intArray.Add(100);
intArray.Add(200);
intArray.Add(300);

// 获取元素数量
console.log("数量:", intArray.Num());

// 获取元素（注意：使用 Get 方法，不是下标）
console.log("第一个元素:", intArray.Get(0));

// 设置元素
intArray.Set(0, 999);

// 遍历数组
for (let i = 0; i < intArray.Num(); i++) {
    console.log(`[${i}]: ${intArray.Get(i)}`);
}

// 检查索引有效性
if (intArray.IsValidIndex(5)) {
    console.log(intArray.Get(5));
}

// Vector 数组
vectorArray.Add(new UE.Vector(7, 8, 9));
console.log(vectorArray.Get(0).ToString());
```

### 4.3 定长数组操作

```typescript
// C++ 中定义的定长数组 int32 MyFixSizeArray[100]
console.log("数组大小:", obj.MyFixSizeArray.Num()); // 100

// 获取元素
console.log("元素[32]:", obj.MyFixSizeArray.Get(32));

// 设置元素
obj.MyFixSizeArray.Set(33, 1000);
```

### 4.4 TSet 操作

```typescript
// 添加元素
stringSet.Add("John");
stringSet.Add("Che");

// 获取数量
console.log("数量:", stringSet.Num());

// 检查是否包含
console.log("包含 John:", stringSet.Contains("John"));  // true
console.log("包含 Hello:", stringSet.Contains("Hello")); // false
```

### 4.5 TMap 操作

```typescript
// 添加键值对
strToIntMap.Add("John", 1);
strToIntMap.Add("Hello", 2);

// 获取值
console.log("John 的值:", strToIntMap.Get("John"));   // 1
console.log("不存在的键:", strToIntMap.Get("None")); // undefined

// 修改值
strToIntMap.Add("John", 100);
console.log("修改后:", strToIntMap.Get("John")); // 100
```

### 4.6 ArrayBuffer 操作

```typescript
// 获取 ArrayBuffer
let ab = obj.ArrayBuffer;

// 转换为 Uint8Array 操作
let u8a = new Uint8Array(ab);
for (let i = 0; i < u8a.length; i++) {
    console.log(`[${i}]: ${u8a[i]}`);
}

// 传递 ArrayBuffer 给 C++
obj.ArrayBufferTest(ab);
obj.ArrayBufferTest(new Uint8Array([1, 2, 3, 4, 5]).buffer);
```

### ⚠️ 易错点

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 索引越界返回 undefined | TArray 不会抛异常 | 先调用 `IsValidIndex()` 检查 |
| 使用下标访问失败 | 必须用方法访问 | 使用 `Get(i)` 而非 `[i]` |
| FName 空值问题 | FName 空值为 "None" | 注意区分空字符串和 "None" |

---

## 五、Delegate（委托）处理

### 5.1 多播委托 (Multicast Delegate)

```typescript
// 定义回调函数
function onNotify1(value: number) {
    console.log("回调1收到:", value);
}

function onNotify2(value: number) {
    console.log("回调2收到:", value);
    // 只触发一次后移除
    actor.NotifyWithInt.Remove(onNotify2);
}

// 添加回调
actor.NotifyWithInt.Add(onNotify1);
actor.NotifyWithInt.Add(onNotify2);

// 广播（触发所有回调）
actor.NotifyWithInt.Broadcast(888);

// 移除回调
actor.NotifyWithInt.Remove(onNotify1);
```

### 5.2 单播委托 (Unicast Delegate)

```typescript
// 检查是否已绑定
console.log("是否绑定:", actor.NotifyWithRefString.IsBound());

// 绑定回调
actor.NotifyWithRefString.Bind((strRef) => {
    // 读取引用参数
    console.log("收到:", $unref(strRef));
    
    // 修改引用参数（作为返回值）
    $set(strRef, "处理完成");
});

// 执行委托
let strRef = $ref("原始消息");
actor.NotifyWithRefString.Execute(strRef);
console.log("返回值:", $unref(strRef));

// 解绑
actor.NotifyWithRefString.Unbind();
```

### 5.3 带返回值的委托

```typescript
// 绑定带返回值的回调
actor.NotifyWithStringRet.Bind((inputStr) => {
    return "处理后: " + inputStr;
});

// 执行并获取返回值
let result = actor.NotifyWithStringRet.Execute("hello world");
console.log("返回结果:", result);
```

### 5.4 传递 JS 函数作为 Delegate 参数

```typescript
import {toManualReleaseDelegate, releaseManualReleaseDelegate} from 'puerts';

// 定义函数
function isJohn(name: string): boolean {
    return name === "John";
}

// 包装为委托并传递给 C++
obj.PassJsFunctionAsDelegate(toManualReleaseDelegate(isJohn));

// ⚠️ 重要：使用完毕后必须释放！
releaseManualReleaseDelegate(isJohn);
```

### 5.5 未处理的 Promise 拒绝

```typescript
import {on} from 'puerts';

// 监听未处理的 Promise 拒绝
on('unhandledRejection', function(reason: any) {
    console.error('未处理的 Promise 拒绝:', reason);
});

// 这个拒绝会被上面的处理器捕获
new Promise(() => {
    throw new Error('something went wrong');
});
```

### ⚠️ 易错点

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 委托内存泄漏 | 忘记释放手动委托 | 使用后调用 `releaseManualReleaseDelegate()` |
| 混淆 API | 多播和单播 API 不同 | 多播用 `Add/Remove/Broadcast`，单播用 `Bind/Execute/Unbind` |
| 引用参数无法修改 | 未使用 `$set` | 使用 `$set(ref, value)` 修改引用参数 |

---

## 六、蓝图 Mixin

Mixin 允许用 TypeScript 扩展或覆盖蓝图/C++ 类的方法，是实现热更新和脚本化的重要功能。

### 6.1 基本用法

```typescript
import * as UE from 'ue'
import {argv, blueprint} from 'puerts';

// 1. 加载蓝图类
let ucls = UE.Class.Load('/Game/StarterContent/MixinTest.MixinTest_C');

// ⚠️ 关键：必须保持 ucls 的引用，否则蓝图会被 UE GC 回收
const MixinTest = blueprint.tojs<typeof UE.Game.StarterContent.MixinTest.MixinTest_C>(ucls);

// 2. 声明接口，让 TypeScript 能调用原类的方法
interface Loggable extends UE.Game.StarterContent.MixinTest.MixinTest_C {};

// 3. 定义 Mixin 类
class Loggable {
    // 覆盖蓝图中的方法
    Log(msg: string): void {
        console.log(this.GetName(), msg);
        // 调用自定义方法
        console.log(`1 + 3 = ${this.TsAdd(1, 3)}`);
    }

    // 新增纯 TypeScript 方法
    TsAdd(x: number, y: number): number {
        console.log(`TS Add(${x}, ${y})`);
        return x + y;
    }
}

// 4. 执行 Mixin
const MixinTestWithMixin = blueprint.mixin(MixinTest, Loggable);

// 5. 使用 Mixin 后的类创建对象
let gameInstance = argv.getByName("GameInstance") as UE.GameInstance;
let actor = UE.GameplayStatics.BeginDeferredActorSpawnFromClass(
    gameInstance, 
    MixinTestWithMixin.StaticClass(), 
    undefined,
    UE.ESpawnActorCollisionHandlingMethod.Undefined,
    undefined
) as Loggable;
UE.GameplayStatics.FinishSpawningActor(actor, undefined);

// 调用 Mixin 后的方法
actor.Log("来自 TypeScript 的消息");
```

### 6.2 原生 C++ 类 Mixin

可以覆盖 `BlueprintNativeEvent` 和 `BlueprintImplementableEvent` 方法：

```typescript
// 定义 Mixin 类
class Calc {
    // 覆盖 BlueprintNativeEvent
    Mult(x: number, y: number): number {
        console.log(`TS Mult(${x}, ${y})`);
        return x * y;
    }

    // 覆盖 BlueprintImplementableEvent
    Div(x: number, y: number): number {
        console.log(`TS Div(${x}, ${y})`);
        return x / y;
    }
}

// 声明接口
interface Calc extends UE.MainObject {};

// 全局 Mixin（影响所有实例）
blueprint.mixin(UE.MainObject, Calc);

// 现在所有 MainObject 实例的 Mult/Div 都会走 TS 实现
let obj = new UE.MainObject();
obj.Mult(1, 2);  // 输出: TS Mult(1, 2)
obj.Div(4, 5);   // 输出: TS Div(4, 5)
```

### 6.3 使用 super 关键字

当需要在 Mixin 中调用父类方法时：

```typescript
// 加载基类
let ucls_base = UE.Class.Load('/Game/StarterContent/MixinSuperTestBase.MixinSuperTestBase_C');
const MixinSuperTestBase = blueprint.tojs<typeof UE.Game.StarterContent.MixinSuperTestBase.MixinSuperTestBase_C>(ucls_base);

// 创建占位类（关键步骤）
interface MixinSuperTestBasePlaceHold extends UE.Game.StarterContent.MixinSuperTestBase.MixinSuperTestBase_C {};
class MixinSuperTestBasePlaceHold {}
Object.setPrototypeOf(MixinSuperTestBasePlaceHold.prototype, MixinSuperTestBase.prototype);

// 加载子类
let ucls_child = UE.Class.Load('/Game/StarterContent/MixinSuperTestDerived.MixinSuperTestDerived_C');
const MixinSuperTestDerived = blueprint.tojs<typeof UE.Game.StarterContent.MixinSuperTestDerived.MixinSuperTestDerived_C>(ucls_child);

// 定义 Mixin 类，继承占位类以支持 super
interface DerivedClassMixin extends UE.Game.StarterContent.MixinSuperTestDerived.MixinSuperTestDerived_C {};
class DerivedClassMixin extends MixinSuperTestBasePlaceHold {
    Foo(): void {
        console.log("TS Mixin Foo 执行");
        super.Foo();  // 调用父类方法
    }
}

// 执行 Mixin
const MixinSuperTestDerivedWithMixin = blueprint.mixin(MixinSuperTestDerived, DerivedClassMixin);
```

### 6.4 Mixin 配置选项

```typescript
const MixedClass = blueprint.mixin(OriginalClass, MixinClass, {
    // 如果为 true，生成新类而不是修改原类
    inherit: true,
    
    // 如果 Mixin 类包含纯脚本字段，设为 true 让 UE 管理生命周期
    objectTakeByNative: true
});
```

### ⚠️ 易错点

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 蓝图被 GC | 只保持了转换后的类引用 | 同时保持 `ucls` 和转换后类的引用 |
| 方法签名不兼容 | 覆盖方法参数/返回值类型不匹配 | 确保满足协变逆变规则 |
| Mixin 字段丢失 | 纯脚本字段被 GC | 使用 `objectTakeByNative: true` 或保持对象引用 |
| super 调用失败 | 未正确设置原型链 | 使用占位类并调用 `Object.setPrototypeOf` |

---

## 七、UI Widget 操作

### 7.1 加载和创建 Widget

```typescript
import * as UE from 'ue'
import {argv} from 'puerts'

// 获取 World
let world = (argv.getByName("GameInstance") as UE.GameInstance).GetWorld() as UE.World;

// 加载 Widget 蓝图类
let widgetClass = UE.Class.Load("/Game/StarterContent/TestWidgetBlueprint.TestWidgetBlueprint_C");

// 创建 Widget 实例
let widget = UE.UMGManager.CreateWidget(world, widgetClass) as UE.Game.StarterContent.TestWidgetBlueprint.TestWidgetBlueprint_C;

// 添加到视口
widget.AddToViewport(0);
```

### 7.2 绑定 UI 事件

```typescript
// 绑定按钮点击事件
widget.Button1.OnClicked.Add(() => {
    console.log("按钮被点击!");
    
    // 获取输入框内容
    let inputText = widget.TextBox.GetText();
    console.log("输入内容:", inputText);
});

// 绑定其他事件
widget.Button1.OnHovered.Add(() => {
    console.log("鼠标悬停");
});

widget.Button1.OnUnhovered.Add(() => {
    console.log("鼠标离开");
});
```

### 7.3 修改 UI 属性

```typescript
// 设置文本
widget.TextBlock.SetText("Hello World");

// 设置可见性
widget.Button1.SetVisibility(UE.ESlateVisibility.Visible);

// 设置颜色
widget.Image1.SetColorAndOpacity(new UE.LinearColor(1, 0, 0, 1));
```

---

## 八、异步操作

### 8.1 延迟等待

```typescript
import * as UE from 'ue'
import * as AsyncUtils from './AsyncUtils'
import {argv} from 'puerts';

let world = (argv.getByName("GameInstance") as UE.GameInstance).GetWorld();

async function delayTest() {
    console.log("开始延迟 3 秒...");
    
    // 创建延迟状态对象
    let latentActionState = new UE.LatentActionState();
    
    // 调用 UE 的延迟函数
    UE.KismetSystemLibrary.Delay(world, 3, latentActionState.GetLatentActionInfo());
    
    // 等待延迟完成
    await AsyncUtils.WaitLatentActionState(latentActionState);
    
    console.log("延迟结束!");
}

delayTest().catch((error) => console.error("错误:", error));
```

### 8.2 异步加载资源

```typescript
async function loadTest() {
    console.log("开始异步加载...");
    
    // 异步加载蓝图类
    let cls = await AsyncUtils.AsyncLoad("/Game/StarterContent/TestBlueprint.TestBlueprint_C");
    
    console.log("加载完成:", cls.GetName());
    
    // 使用加载的类
    let actor = UE.GameplayStatics.BeginDeferredActorSpawnFromClass(
        gameInstance, 
        cls, 
        undefined
    );
    UE.GameplayStatics.FinishSpawningActor(actor, undefined);
}

loadTest();
```

### 8.3 异步工具函数实现

```typescript
// AsyncUtils.ts
import * as UE from 'ue'

/**
 * 等待 LatentAction 完成
 */
export function WaitLatentActionState(state: UE.LatentActionState): Promise<void> {
    return new Promise<void>((resolve, reject) => {
        state.LatentActionCallback.Bind(() => {
            // 解绑回调防止内存泄漏
            state.LatentActionCallback.Unbind();
            resolve();
        });
    });
}

/**
 * 异步加载资源
 */
export function AsyncLoad(path: string): Promise<UE.Class> {
    return new Promise<UE.Class>((resolve, reject) => {
        let asyncLoadObj = new UE.AsyncLoadState();
        
        asyncLoadObj.LoadedCallback.Bind((cls: UE.Class) => {
            // 解绑回调
            asyncLoadObj.LoadedCallback.Unbind();
            
            if (cls) {
                resolve(cls);
            } else {
                reject(`加载失败: ${path}`);
            }
        });
        
        // 开始加载
        asyncLoadObj.StartLoad(path);
    });
}
```

### 8.4 错误处理

```typescript
async function safeAsyncOperation() {
    try {
        await someAsyncFunction();
    } catch (error) {
        console.error("操作失败:", error);
    }
}

// 或使用 .catch()
someAsyncFunction()
    .then(result => console.log("成功:", result))
    .catch(error => console.error("失败:", error));
```

---

## 九、C++ 静态绑定

用于绑定没有 UFUNCTION 反射标记的普通 C++ 类和函数。

### 9.1 C++ 端：定义类

```cpp
// TestClass.h
#pragma once

#include <string>

class BaseClass
{
public:
    int A;
    int B;
    void Foo(int p);
};

class TestClass : public BaseClass
{
public:
    int32_t X;
    int32_t Y;

    TestClass();
    TestClass(int32_t InX, int32_t InY);

    // 静态方法
    static int32_t Add(int32_t a, int32_t b);
    static void Overload();
    static void Overload(int32_t a);
    static void Overload(int32_t a, int32_t b);
    static void Overload(std::string a, int32_t b);

    // 实例方法
    int32_t OverloadMethod();
    int32_t OverloadMethod(int32_t a);
    TestClass* GetSelf();

    // 引用参数
    int Ref(int32_t& a);
    void StrRef(std::string& str);
    int Ptr(int32_t* a);
    const char* CStr(const char* str);

    // 静态变量
    static int StaticInt;
    static const float Ten;
};
```

### 9.2 C++ 端：注册绑定

```cpp
// TestClassWrap.cpp
#include "TestClass.h"
#include "Binding.hpp"

// 声明要使用的 C++ 类型
UsingCppType(BaseClass);
UsingCppType(TestClass);

// 自动注册结构体
struct AutoRegisterForTestClass
{
    AutoRegisterForTestClass()
    {
        // 注册基类
        puerts::DefineClass<BaseClass>()
            .Method("Foo", MakeFunction(&BaseClass::Foo))
            .Register();

        // 注册 TestClass
        puerts::DefineClass<TestClass>()
            .Extends<BaseClass>()  // 继承关系
            
            // 构造函数（支持多个重载）
            .Constructor(CombineConstructors(
                MakeConstructor(TestClass, int32_t, int32_t),
                MakeConstructor(TestClass)
            ))
            
            // 属性
            .Property("X", MakeProperty(&TestClass::X))
            .Property("Y", MakeProperty(&TestClass::Y))
            
            // 静态变量
            .Variable("StaticInt", MakeVariable(&TestClass::StaticInt))
            .Variable("Ten", MakeReadonlyVariable(&TestClass::Ten))
            
            // 静态方法
            .Function("Add", MakeFunction(&TestClass::Add))
            
            // 重载的静态方法
            .Function("Overload", CombineOverloads(
                MakeOverload(void(*)(), &TestClass::Overload),
                MakeOverload(void(*)(int32_t), &TestClass::Overload),
                MakeOverload(void(*)(int32_t, int32_t), &TestClass::Overload),
                MakeOverload(void(*)(std::string, int32_t), &TestClass::Overload)
            ))
            
            // 实例方法
            .Method("GetSelf", MakeFunction(&TestClass::GetSelf))
            .Method("Ref", MakeFunction(&TestClass::Ref))
            .Method("StrRef", MakeFunction(&TestClass::StrRef))
            
            // 重载的实例方法
            .Method("OverloadMethod", CombineOverloads(
                MakeOverload(int32_t(TestClass::*)(), &TestClass::OverloadMethod),
                MakeOverload(int32_t(TestClass::*)(int32_t), &TestClass::OverloadMethod)
            ))
            
            .Register();
    }
};

// 全局变量实现自动注册
AutoRegisterForTestClass _AutoRegisterForTestClass__;
```

### 9.3 TypeScript 端：使用绑定的类

```typescript
import {$ref, $unref} from 'puerts';
import * as cpp from 'cpp'  // 静态绑定的类在 cpp 模块

let TestClass = cpp.TestClass;

// 调用静态方法
console.log(TestClass.Add(12, 34));  // 46

// 重载方法调用
TestClass.Overload();
TestClass.Overload(1);
TestClass.Overload(1, 2);
TestClass.Overload("hello", 2);

// 构造对象
let obj = new TestClass();
obj = new TestClass(8, 9);

// 调用实例方法
obj.OverloadMethod();
obj.OverloadMethod(1024);

// 访问属性
console.log(obj.X, obj.Y);
obj.X = 96;
obj.Y = 97;

// 访问静态变量
console.log(TestClass.StaticInt);
TestClass.StaticInt = 789;

// 引用参数
let r = $ref(999);
let ret = obj.Ref(r);
console.log("引用结果:", $unref(r), "返回值:", ret);

// 字符串引用
let sr = $ref("原始消息");
obj.StrRef(sr);
console.log("字符串结果:", $unref(sr));
```

### 9.4 std::function 支持

```cpp
// C++ 端
class AdvanceTestClass
{
public:
    void StdFunctionTest(std::function<int(int, int)> callback)
    {
        int result = callback(10, 20);
        UE_LOG(LogTemp, Warning, TEXT("Result: %d"), result);
    }
};

// 注册
puerts::DefineClass<AdvanceTestClass>()
    .Constructor<int>()
    .Method("StdFunctionTest", MakeFunction(&AdvanceTestClass::StdFunctionTest))
    .Register();
```

```typescript
// TypeScript 端
let obj = new cpp.AdvanceTestClass(100);

obj.StdFunctionTest((x: number, y: number) => {
    console.log('x =', x, 'y =', y);
    return x + y;
});
```

---

## 十、继承 UE 类（已废弃）

> ⚠️ **注意：`makeUClass` 已废弃，功能有限（不支持 RPC 等），推荐使用 Mixin 或 TS 继承 UE 类功能。**

```typescript
import * as UE from 'ue'
import {argv, makeUClass} from 'puerts';

// 定义继承 UE 类的 TypeScript 类
class MyActor extends UE.Actor {
    tickCount: number;

    // ⚠️ 构造函数必须大写开头 Constructor
    Constructor() {
        this.PrimaryActorTick.bCanEverTick = true;
        this.tickCount = 0;
    }

    // 覆盖生命周期方法
    ReceiveBeginPlay(): void {
        console.log("BeginPlay 被调用");
    }

    ReceiveTick(DeltaSeconds: number): void {
        if (this.tickCount % 100 === 0) {
            console.log("Tick:", this.tickCount, "Delta:", DeltaSeconds);
        }
        this.tickCount++;
    }

    // 自定义方法
    CustomMethod(): void {
        console.log("自定义方法");
    }
}

// 生成 UClass
let cls = makeUClass(MyActor);

// 生成 Actor
let gameInstance = argv.getByName("GameInstance") as UE.GameInstance;
let actor = UE.GameplayStatics.BeginDeferredActorSpawnFromClass(
    gameInstance, 
    cls, 
    undefined
) as UE.Actor;
UE.GameplayStatics.FinishSpawningActor(actor, undefined);
```

---

## 十一、蓝图加载最佳实践

### 11.1 方式一：使用 UE.Class.Load

```typescript
// 直接加载蓝图类
let bpClass = UE.Class.Load('/Game/StarterContent/TestBlueprint.TestBlueprint_C');

// 使用类生成对象
let bpActor = UE.GameplayStatics.BeginDeferredActorSpawnFromClass(
    gameInstance, 
    bpClass, 
    undefined
) as UE.Game.StarterContent.TestBlueprint.TestBlueprint_C;
UE.GameplayStatics.FinishSpawningActor(bpActor, undefined);

// 调用蓝图方法
bpActor.Foo(false, 8000, 9000);
```

### 11.2 方式二：使用 blueprint.load（推荐）

```typescript
import {blueprint} from 'puerts';

// 加载蓝图（会缓存类引用）
blueprint.load(UE.Game.StarterContent.TestBlueprint.TestBlueprint_C);

// 创建别名方便使用
const TestBlueprint_C = UE.Game.StarterContent.TestBlueprint.TestBlueprint_C;

// 使用
let bpActor = UE.GameplayStatics.BeginDeferredActorSpawnFromClass(
    gameInstance, 
    TestBlueprint_C.StaticClass(), 
    undefined
) as UE.Game.StarterContent.TestBlueprint.TestBlueprint_C;
UE.GameplayStatics.FinishSpawningActor(bpActor, undefined);

// ⚠️ 使用完毕后释放内存
blueprint.unload(TestBlueprint_C);
```

### 11.3 蓝图结构体加载

```typescript
// 加载蓝图结构体
blueprint.load(UE.Game.StarterContent.TestStruct.TestStruct);
const TestStruct = UE.Game.StarterContent.TestStruct.TestStruct;

// 创建结构体实例
let testStruct = new TestStruct();
testStruct.age = 10;
testStruct.speed = 5;

// 传递给方法
bpActor.Bar(testStruct);

// 使用完毕后释放
blueprint.unload(TestStruct);
```

### 11.4 蓝图枚举使用

```typescript
// 蓝图枚举可以直接访问
console.log(UE.Game.StarterContent.TestEnum.TestEnum.Blue);
console.log(UE.Game.StarterContent.TestEnum.TestEnum.Red);
console.log(UE.Game.StarterContent.TestEnum.TestEnum.Green);
```

---

## 十二、易错点汇总

### 生命周期问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 蓝图被 UE GC 回收 | 只保持了 JS 侧类引用，C++ 侧 UClass 无引用 | 同时保持 `ucls` 和 `blueprint.tojs()` 返回值的引用 |
| 委托回调内存泄漏 | 使用 `toManualReleaseDelegate` 后未释放 | 必须调用 `releaseManualReleaseDelegate()` |
| 虚拟机内存泄漏 | 未在 Shutdown 中清理 | 调用 `GameScript.Reset()` |
| Widget 无法销毁 | 事件绑定持有引用 | 手动解绑事件后再销毁 |

### 类型问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 引用参数不生效 | 未使用 `$ref` 包装 | 使用 `$ref`/`$unref`/`$set` |
| 容器越界返回 undefined | TArray 不抛异常 | 先调用 `IsValidIndex()` 检查 |
| int64/uint64 精度丢失 | JavaScript Number 只有 53 位精度 | 使用 `BigInt` 类型 |
| float 精度问题 | 双精度转单精度 | 接受一定误差或使用 double |

### 调用问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 找不到扩展方法 | 扩展方法未注册 | 添加到 wrapper 列表或使用 `puer.$extension` |
| Mixin 方法不生效 | 签名不兼容或原型链问题 | 确保签名匹配，正确设置原型链 |
| makeUClass 功能受限 | 已废弃的 API | 改用 Mixin 或新的 TS 继承功能 |
| 静态绑定类找不到 | 未使用 `UsingCppType` | 确保在绑定文件中声明类型 |

### 调试问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 断点不生效 | 脚本执行太快 | 使用 `debugger;` 语句主动触发断点 |
| 调试器连接失败 | 端口被占用或未等待 | 使用不同端口，调用 `WaitDebugger()` |
| 热更新不生效 | 未开启相关功能 | 检查 Puerts 配置 |

### 常见报错

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `Access Invalid Object` | 访问已被 GC 的对象 | 确保对象引用被正确持有 |
| `can not find delegate bridge` | 委托类型未生成包装 | 重新生成包装代码 |
| `invalid arguments to XXX` | 参数类型不匹配 | 检查参数类型，使用正确的转换 |

---

## 十三、项目结构建议

```
项目根目录/
├── Source/                      # C++ 源代码
│   └── YourProject/
│       ├── TsGameInstance.cpp   # 虚拟机启动入口
│       ├── TsGameInstance.h
│       ├── MainObject.cpp       # 演示用 UObject
│       ├── MainObject.h
│       └── ...
│
├── TypeScript/                  # TypeScript 源文件
│   ├── QuickStart.ts           # 入口脚本
│   ├── AsyncUtils.ts           # 异步工具
│   ├── UsingMixin.ts           # Mixin 示例
│   ├── UsingWidget.ts          # UI 示例
│   └── ...
│
├── Content/
│   └── JavaScript/              # 编译后的 JS 文件（tsc 输出）
│       ├── QuickStart.js
│       ├── AsyncUtils.js
│       └── ...
│
├── Typing/                      # 类型声明文件
│   ├── ue/                     # UE API 类型
│   │   ├── index.d.ts
│   │   └── ue.d.ts
│   ├── cpp/                    # C++ 静态绑定类型
│   │   └── index.d.ts
│   └── puerts/                 # Puerts API 类型
│       └── index.d.ts
│
├── Plugins/
│   └── Puerts/                 # Puerts 插件
│
├── tsconfig.json               # TypeScript 配置
└── package.json                # npm 配置
```

### tsconfig.json 示例

```json
{
  "compilerOptions": {
    "target": "esnext",
    "module": "commonjs",
    "experimentalDecorators": true,
    "jsx": "react",
    "sourceMap": true,
    "typeRoots": [
      "Typing",
      "./node_modules/@types"
    ],
    "outDir": "Content/JavaScript"
  },
  "include": [
    "TypeScript/**/*"
  ]
}
```

---

## 参考资源

- [Puerts 官方仓库](https://github.com/Tencent/puerts)
- [Puerts 官方文档](https://github.com/Tencent/puerts/tree/master/doc)
- [Puerts Unreal Demo](https://github.com/chexiongsheng/puerts_unreal_demo)

---

*文档生成时间: 2025年12月30日*
