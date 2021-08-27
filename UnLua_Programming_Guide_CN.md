[TOC]

# Overview 概述
UnLua is a feature-rich and high optimized scripting solution for UE4. Developes can develop game logic using Lua with it, and it's possible to iterate game logic much more faster with Lua's hotload feature. This document will introduce the major features and basic programming pattern of UnLua.

UnLua 是一个有丰富特性的高性能 UE4 脚本解决方案。开发者可以使用 Lua 在 UE4 中做开发，并且使得热更新在 UE4 项目中变为可能。本文档将介绍 UnLua 一些重要的特性和编程模式。

---

# Lua&Engine Binding Lua 与 引擎的绑定
UnLua provides a simple way to bind Lua and engine, including static binding and dynamic binding:

Unlua 使用便利的方式在 Lua 和引擎之间做绑定，包括静态绑定和动态绑定：

## Static Binding 静态绑定

#### C++ C++
Your UCLASS only need to implement **IUnLuaInterface**, and return a Lua file path in **GetModuleName_Implementation()**.

你的 UCLASS 只需要实现 **IUnLuaInterface** 接口并且在 **GetModuleName_Implementation()** 函数中返回一个 Lua 文件的路径即可。

![CPP_UNLUA_INTERFACE](./Images/cpp_unlua_interface.png)

#### Blueprints 蓝图
Your blueprint only need to implement **UnLuaInterface**, and return a Lua file path in **GetModuleName()**.

你的蓝图只需要实现 **UnLuaInterface** 接口并且在 **GetModuleName()** 函数中返回一个 Lua 文件路径即可。

![BP_UNLUA_INTERFACE](./Images/bp_unlua_interface.png)
![BP_GETMODULENAME](./Images/bp_getmodulename.png)

## Dynamic Binding
Dynamic binding is suitable for runtime spawned Actors and Objects.

运行时生成的 Actor 和 Object 适用动态绑定。

#### Actor Actor
```
local Proj = World:SpawnActor(ProjClass, Transform, ESpawnActorCollisionHandlingMethod.AlwaysSpawn, self, self.Instigator, "Weapon.BP_DefaultProjectile_C")
```
**“Weapon.BP_DefaultProjectile_C”** is a Lua file path.

**“Weapon.BP_DefaultProjectile_C”** 是一个 Lua 文件路径。


#### Object Object
```
local ProxyObj = NewObject(ObjClass, nil, nil, "Objects.ProxyObject")
```
**“Objects.ProxyObject”** is a Lua file path.

**““Objects.ProxyObject””** 是一个 Lua 文件路径。

## Lua File Path Lua 文件路径
Both static and dynamic binding need a Lua file path. It's a **relative** path to project's **'Content/Script'**.

静态绑定和动态绑定都需要提供 Lua 文件路径。该路径是工程下 **'Content/Script'** 的**相对**路径。

---

# Lua Calls Engine Lua 层调用 Engine
UnLua provides two ways to access engine from Lua side: 1. dynamically export using reflection system; 2. statically export classes, member variables, member functions, global functions and enums outside the reflection system.

UnLua 提供了两种从 Lua 层到引擎层调用的方法：
1. 使用反射系统动态导出；
2. 静态的导出类、成员变量、成员函数，以及没被纳入反射系统的全局函数和枚举。

## Dynamically Export using Reflection System 使用反射系统动态导出
Dynamically export using reflection system makes codes clean, intuitive, and it can eliminate huge glue codes.

使用反射系统动态导出可以使代码看起来更整洁和只管，并且避免了很多胶水代码。


### Access UCLASS 访问 UCLASS
```
local Widget = UWidgetBlueprintLibrary.Create(self, UClass.Load("/Game/Core/UI/UMG_Main"))
```
**UWidgetBlueprintLibrary** is a UCLASS. Class' name in Lua must be **PrefixCPP** + **ClassName** + **[``_C``]**, for example: **AActor** (Native Class), **ABP_PlayerCharacter_C**（Blueprint Class）

**UWidgetBlueprintLibrary** 是一个 UCLASS。Lua 层类名必须是以下模式：**PrefixCPP** + **ClassName** + **[``_C``]**，例如：**AActor**（本地类），**ABP_PlayerCharacter_C**（蓝图类）

### Access UFUNCTION 访问 UFUNCTION
```
Widget:AddToViewport(0)
```
**AddToViewport** is a UFUNCTION of **UUserWidget**. **0** is the function parameter. If a parameter of UFUNCTION (tagged with **'BlueprintCallable'** or **'Exec'**) has a default value, it can be ignored in Lua codes:

**AddToViewport** 是 **UUserWidget** 的一个 UFUNCTION。**0** 为函数参数。如果一个 UFUNCTION（带有 **'BlueprintCallable'** 或 **'Exec'** 标签的） 有默认参数，那么它可以在 Lua 代码中被省略：

```
Widget:AddToViewport()
```

#### Output Values Handling 输出值处理 
*（这里没有用 return 应该是 Lua 中函数可以有多个返回值，而 C++ 中如果要实现多个返回值一般是给某个函数定义指针或引用类型参数（C# 中的 out），所以这里用 output 更广义一些。再说 C++ 中想要实现多个返回值，还有一种方式，是定义一个包裹了你想返回的所有类型的 struct/class，此种方式不常用，因为一旦定义了一个新的 struct/class，势必要使用时 include 它，并且还涉及到这个 struct/class 的拷贝构造、赋值构造、赋值操作符、析构操作等等一堆问题，在使用是稍不注意就会引入新的问题。）*

Output Values includes **non-const reference parameters** and **return parameter**. Both of them distinguish **primitive types (bool, integer, number, string)** and **non-primitive types (userdata)**.

输出值包括**非常量引用参数**和**返回参数**。他们都区分是**原始类型（bool, integer, number, string）**还是**非原始类型（userdata）**。

##### Non-Const Reference Parameters 非常量引用参数
###### Primitive Types 原始类型
![OUT_PRIMITIVE_TYPES](./Images/out_primitive_types.png)

Lua codes：
```
local Level, Health, Name = self:GetPlayerBaseInfo()
```


###### Non-Primitive Types 非原始类型
![OUT_NON_PRIMITIVE_TYPES](./Images/out_non_primitive_types.png)

There are two ways to call it in Lua:

在 Lua 中有两种方式调用它：

```
local HitResult = FHitResult()
self:GetHitResult(HitResult)
```
Or
```
local HitResult = self:GetHitResult()
```
The first one is similiar to C++, it's much more efficient than the second one when called multiple times such as in a loop.

第一种方式与 C++ 类似，而且如果当前逻辑在循环中，他比第二种性能更高。

*为什么性能要更高？应该是这样：*
``` lua
-- 如下，case1 比 case2 节省了 n - 1 次构造函数的开销。

-- case 1:

local HitResult = FHitResult()
for i = 1, n do
    self:GetHitResult(HitResult);
    -- use HitResult
end
-------------------------------------

-- case 2:
for i = i, n do
    local HitResult = self:GetHitResult()
    -- use HitResult
end
-------------------------------------
```


##### Return Parameter 返回参数
###### Primitive Types 原始类型
![RET_PRIMITIVE_TYPES](./Images/return_primitive_types.png)

Lua codes：
```
local MeleeDamage = self:GetMeleeDamage()
```

###### Non-Primitive Types 非原始类型
![RET_NON_PRIMITIVE_TYPES](./Images/return_non_primitive_types.png)

There are three ways to call it in Lua:

在 Lua 中有三种方式调用它：

```
local Location = self:GetCurrentLocation()
```
Or
```
local Location = FVector()
self:GetCurrentLocation(Location)
```
Or
```
local Location = FVector()
local LocationCopy = self:GetCurrentLocation(Location)
```
The first one is most intuitive, however the latter two are much more efficient than the first one when called multiple times such as in a loop. The last one is equivalent to:

第一种方式更直观，但是在循环中，后两种方式要比第一种方式性能更好 *（原因应该和上述一样）*。最后一种方式相当于：

```
local Location = FVector()
self:GetCurrentLocation(Location)
local LocationCopy = Location
```

#### Latent Function
Latent functions allow developers to develop asynchronous logic using synchronous coding style. A typical latent function is **Delay**:

Latent 函数允许开发者以同步的形式编写异步逻辑，一个典型的例子是 **Delay**：*（![参考文档](https://docs.unrealengine.com/4.27/zh-CN/ProgrammingAndScripting/GameplayArchitecture/Metadata/), ![参考官方问答](https://answers.unrealengine.com/questions/583811/custom-latent-function-for-actor-blueprint-in-c.html)）*

![LATENT_FUNCTION](./Images/latent_function.png)

You can call latent function in Lua coroutines:

你可以使用 Lua 的协程调用 latent 函数：

```
coroutine.resume(coroutine.create(function(GameMode, Duration) UKismetSystemLibrary.Delay(GameMode, Duration) end), self, 5.0)
```

#### Optimization 优化
UnLua optimizes UFUNCTION invoking in following two points:

 * Persistent parameters buffer
 * Optimized local function invoking
 * Optimized parameters passing
 * Optimized output values handling

UnLua 是基于以下几点来优化 FUNCTION 调用的：

- 持久性参数缓存；
- 优化 local 函数的调用；
- 优化参数传递；
- 优化输出值的处理。

### Access USTRUCT 访问 USTRUCT
```
local Position = FVector()
```
**FVector** is a USTRUCT.

**FVector** 是一个 USTRUCT。

### Access UPROPERTY 访问 UPROPERTY
```
local Position = FVector()
Position.X = 256.0
```
**X** is a UPROPERTY of **FVector**.

**X** 是 **FVector** 的一个 UPROPERTY。

#### Delegates 代理
* Bind 绑定
```
FloatTrack.InterpFunc:Bind(self, BP_PlayerCharacter_C.OnZoomInOutUpdate)
```
**InterpFunc** is a Delegate of FTimelineFloatTrack, **'Bind'** binds a callback (**BP_PlayerCharacter_C.OnZoomInOutUpdate**) for **InterpFunc**.

**InterpFunc** 是 FTimelineFloatTrack 的一个代理，函数 **'Bind'** 给 **InterpFunc** 绑定了一个回调函数(**BP_PlayerCharacter_C.OnZoomInOutUpdate**)。

* Unbind 解绑
```
FloatTrack.InterpFunc:Unbind()
```
**InterpFunc** is a Delegate of FTimelineFloatTrack, **'Unbind'** unbinds the callback for **InterpFunc**.

**InterpFunc** 是 FTimelineFloatTrack 的一个代理，函数 **'UnBind'** 将 **InterpFunc** 的回调函数解绑。

* Execute 触发
```
FloatTrack.InterpFunc:Execute(0.5)
```
**InterpFunc** is a Delegate of FTimelineFloatTrack, **'Execute'** calls the function on the object bound to **InterpFunc**.

**InterpFunc** 是 FTimelineFloatTrack 的一个代理，函数 **'Execute'** 调用了绑定在 **InterpFunc** 的回调。


#### Multicast Delegates 多播代理
* Add 注册
```
 self.ExitButton.OnClicked:Add(self, UMG_Main_C.OnClicked_ExitButton)
```
**OnClicked** is a MulticastDelegate of UButton, **'Add'** adds a callback (**UMG_Main_C.OnClicked_ExitButton**) for **OnClicked**.

**OnClicked** 是 UButton 的一个 MulticastDelegate，函数 **'Add'** 给 **OnClicked** 事件添加了一个回调(**UMG_Main_C.OnClicked_ExitButton**)。

* Remove 移除
```
 self.ExitButton.OnClicked:Remove(self, UMG_Main_C.OnClicked_ExitButton)
```
**OnClicked** is a MulticastDelegate of UButton, **'Remove'** removes a callback (**UMG_Main_C.OnClicked_ExitButton**) for **OnClicked**.

**OnClicked** 是 UButton 的一个 MulticastDelegate，函数 **'Remove'** 将回调(**UMG_Main_C.OnClicked_ExitButton**) 从 **OnClicked** 事件移除了出去。

* Clear 清除
```
 self.ExitButton.OnClicked:Clear()
```
**OnClicked** is a MulticastDelegate of UButton, **'Clear'** clears all callbacks for **OnClicked**.

**OnClicked** 是 UButton 的一个 MulticastDelegate，函数 **'Clear'** 清除了 **OnClicked** 事件的所有回调。

* Broadcast 触发（广播）
```
 self.ExitButton.OnClicked:Broadcast()
```
**OnClicked** is a MulticastDelegate of UButton, **'Broadcast'** calls all functions on objects bound to **OnClicked**.

**OnClicked** 是 UButton 的一个 MulticastDelegate，函数 **'Broadcast'** 调用了 **OnClicked** 事件的所有回调。

### Access UENUM 访问 UENUM
```
Weapon:K2_AttachToComponent(Point, nil, EAttachmentRule.SnapToTarget, EAttachmentRule.SnapToTarget, EAttachmentRule.SnapToTarget)
```
**EAttachmentRule** is a UENUM, **SnapToTarget** is an entry of **EAttachmentRule**.

**EAttachmentRule** 是一个 UENUM，**SnapToTarget** 是枚举 **EAttachmentRule** 的一个元素。

#### Customized Collision Enums 碰撞中的自定义枚举
![ENUM_COLLISION](./Images/enum_collision.png)

* EObjectTypeQuery
```
local ObjectTypes = TArray(EObjectTypeQuery)
ObjectTypes:Add(EObjectTypeQuery.Player)
ObjectTypes:Add(EObjectTypeQuery.Enemy)
ObjectTypes:Add(EObjectTypeQuery.Projectile)
local bHit = UKismetSystemLibrary.LineTraceSingleForObjects(self, Start, End, ObjectTypes, false, nil, EDrawDebugTrace.None, HitResult, true)
```
**EObjectTypeQuery.Player**, **EObjectTypeQuery.Enemy** and **EObjectTypeQuery.Projectile** are customized Object Channels.

**EObjectTypeQuery.Player**，**EObjectTypeQuery.Enemy** 和 **EObjectTypeQuery.Projectile** 是自定义的对象通道。

* ETraceTypeQuery
```
local bHit = UKismetSystemLibrary.LineTraceSingle(self, Start, End, ETraceTypeQuery.Weapon, false, nil, EDrawDebugTrace.None, HitResult, true)
```
**ETraceTypeQuery.Weapon** is a customized Trace Channel.

**ETraceTypeQuery.Weapon** 是自定义的跟踪通道。

### Manually Exported Libraries 手动导出库
For customization and performance considerations, UnLua manually exports several important classes in the engine, including the following (See the codes for details):

考虑到扩展性和性能，UnLua 手动导出了引擎中一些重要的类，以下列出（详细情况请查看代码）

#### Basic Classes
 * UObject
 * UClass
 * UWorld
 
#### Common Containers
 * TArray
 * TSet
 * TMap
 
##### Example
```
	local Indices = TArray(0)
	Indices:Add(1)
	Indices:Add(3)
	Indices:Remove(0)
	local NbIndices = Indices:Length()
```
```
	local Vertices = TArray(FVector)
	local Actors = TArray(AActor)
```

#### Math Libraries
 * FVector
 * FVector2D
 * FVector4
 * FQuat
 * FRotator
 * FTransform
 * FColor
 * FLinearColor
 * FIntPoint
 * FIntVector

## Statically Export 静态导出
UnLua provides a simple solution to export classes, member variables, member functions, global functions and enums outside the reflection system statically.

UnLua 提供了一个简单的解决方案，用以导出没有纳入静态反射系统的类、成员变量、成员函数和全局函数以及枚举。

### Classes 类
* Non-reflected classes 不需要反射的类
```
BEGIN_EXPORT_CLASS(ClassType, ...)
```
 Or
```
BEGIN_EXPORT_NAMED_CLASS(ClassName, ClassType, ...)
```
 **'...'** means type list of parameters in constructor.

**'...'** 意为构造函数的参数表。

* Reflected classes 需要反射的类
```
BEGIN_EXPORT_REFLECTED_CLASS(UObjectType)
```
 Or
```
BEGIN_EXPORT_REFLECTED_CLASS(NonUObjectType, ...)
```
 **'...'** means type list of parameters in constructor.

 **'...'** 意为构造函数的参数表。

#### Member Variables 成员变量
```
ADD_PROPERTY(Property)
```
Or (for bitfield bool property)
```
ADD_BITFIELD_BOOL_PROPERTY(Property)
```

#### Member Functions 成员函数
##### Non-static member functions 非静态成员函数
* Compact style 紧凑风格
```
ADD_FUNCTION(Function)
```
 Or
```
ADD_NAMED_FUNCTION(Name, Function)
```

* Complete style 完全风格
```
ADD_FUNCTION_EX(Name, RetType, Function, ...)
```
 Or
```
ADD_CONST_FUNCTION_EX(Name, RetType, Function, ...)
```
 **'...'** means parameter types.

**'...'** 意为参数类型表。

##### Static member functions 静态成员函数
```
ADD_STATIC_FUNCTION(Function)
```
Or
```
ADD_STATIC_FUNCTION_EX(Name, RetType, Function, ...)
```
**'...'** means parameter types.

**'...'** 意为参数类型表。

#### Example 例子
```
struct Vec3
{
	Vec3() : x(0), y(0), z(0) {}
	Vec3(float _x, float _y, float _z) : x(_x), y(_y), z(_z) {}

	void Set(const Vec3 &V) { *this = V; }
	Vec3& Get() { return *this; }
	void Get(Vec3 &V) const { V = *this; }

	bool operator==(const Vec3 &V) const { return x == V.x && y == V.y && z == V.z; }

	static Vec3 Cross(const Vec3 &A, const Vec3 &B) { return Vec3(A.y * B.z - A.z * B.y, A.z * B.x - A.x * B.z, A.x * B.y - A.y * B.x); }
	static Vec3 Multiply(const Vec3 &A, float B) { return Vec3(A.x * B, A.y * B, A.z * B); }
	static Vec3 Multiply(const Vec3 &A, const Vec3 &B) { return Vec3(A.x * B.x, A.y * B.y, A.z * B.z); }

	float x, y, z;
};

BEGIN_EXPORT_CLASS(Vec3, float, float, float)
	ADD_PROPERTY(x)
	ADD_PROPERTY(y)
	ADD_PROPERTY(z)
	ADD_FUNCTION(Set)
	ADD_NAMED_FUNCTION("Equals", operator==)
	ADD_FUNCTION_EX("Get", Vec3&, Get)
	ADD_CONST_FUNCTION_EX("GetCopy", void, Get, Vec3&)
	ADD_STATIC_FUNCTION(Cross)
	ADD_STATIC_FUNCTION_EX("MulScalar", Vec3, Multiply, const Vec3&, float)
	ADD_STATIC_FUNCTION_EX("MulVec", Vec3, Multiply, const Vec3&, const Vec3&)
END_EXPORT_CLASS()
IMPLEMENT_EXPORTED_CLASS(Vec3)
```

### Global Functions 全局函数
```
EXPORT_FUNCTION(RetType, Function, ...)
```
Or
```
EXPORT_FUNCTION_EX(Name, RetType, Function, ...)
```
**'...'** means parameter types.

**'...'** 意为参数类型表。

#### Example 例子
```
void GetEngineVersion(int32 &MajorVer, int32 &MinorVer, int32 &PatchVer)
{
	MajorVer = ENGINE_MAJOR_VERSION;
	MinorVer = ENGINE_MINOR_VERSION;
	PatchVer = ENGINE_PATCH_VERSION;
}

EXPORT_FUNCTION(void, GetEngineVersion, int32&, int32&, int32&)
```

### Enums 枚举
* Unscoped enumeration 无域的枚举（普通枚举）
```
enum EHand
{
	LeftHand,
	RightHand
};

BEGIN_EXPORT_ENUM(EHand)
	ADD_ENUM_VALUE(LeftHand)
	ADD_ENUM_VALUE(RightHand)
END_EXPORT_ENUM(EHand)
```

* Scoped enumeration 有域的枚举（枚举类）
```
enum class EEye
{
	LeftEye,
	RightEye
};

BEGIN_EXPORT_ENUM(EEye)
	ADD_SCOPED_ENUM_VALUE(LeftEye)
	ADD_SCOPED_ENUM_VALUE(RightEye)
END_EXPORT_ENUM(EEye)
```

## Optional 'UE4' Namespace 可配置的 “UE4” 命名空间
UnLua provides an option to add a namespace **'UE4'** to all classes and enums in the engine. You can find this option in **UnLua.Build.cs**.

UnLua 可以为所有类的枚举配置一个名为 **'UE4'** 的明明空间。在 **UnLua.Build.cs** 可以找到该配置。

![UE4_NAMESPACE](./Images/ue4_namespace.png)

If this option is enabled, your Lua codes should be:

如果这个配置生效，Lua 代码应是如下形式：

```
local Position = UE4.FVector()
```

---

# Engine Calls Lua 引擎层调用 Lua
UnLua provides a Blueprint-like solution to cross the C++/Script boundary. It allows C++/Blueprint codes to call functions that are defined in Lua codes. 

UnLua 提供了一个类似蓝图的打通 C++ 和脚本之间界线的方法。通过这个方法，可以使用 C++ 或蓝图调用定义在 Lua 中的函数。

## Override BlueprintEvent 覆盖蓝图事件
Your can override all **BlueprintEvent** in Lua codes. **BlueprintEvent** includes:

* UFUNCTION tagged with **'BlueprintImplementableEvent'**
* UFUNCTION tagged with **'BlueprintNativeEvent'**
* **All** Events/Functions Defined in Blueprints

你可以在 Lua 中覆盖所有 **BlueprintEvent**，**BlueprintEvent** 包括：

* 包含 **'BlueprintImplementableEvent'** 标签的 UFUNCTION；
* 包含 **'BlueprintNativeEvent'** 标签的 UFUNCTION；
* **所有** 在蓝图中定义的事件和函数。

### Example (BlueprintEvent without Return Value) 例子（没有返回值的蓝图事件）

![BP_IMP_EVENT](./Images/bp_imp_event.png)

You can override it in Lua:

你可以这样在 Lua 中覆盖它：

```
function BP_PlayerController_C:ReceiveBeginPlay()
  print("ReceiveBeginPlay in Lua!")
end
```

### Example (BlueprintEvent with Return Values) 例子（有返回值的蓝图事件）

![BP_IMP_EVENT_RET](./Images/bp_imp_event_return_values.png)

There are two ways to override it in Lua:

有两种方式可以在 Lua 中覆盖它：

```
function BP_PlayerCharacter_C:GetCharacterInfo(HP, Position, Name)
	Position.X = 128.0
	Position.Y = 128.0
	Position.Z = 0.0
	return 99, nil, "Marcus", true
end
```
Or
```
function BP_PlayerCharacter_C:GetCharacterInfo(HP, Position, Name)
	return 99, FVector(128.0, 128.0, 0.0), "Marcus", true
end
```
The first one is preferred.

推荐第一种方式。*（因为第二种方式多 new 了一个 FVector）*

## Override Animation Notifies 覆盖动画事件
AnimNotify:

![ANIM_NOTIFY](./Images/anim_notify.png)

Lua codes:
```
function ABP_PlayerCharacter_C:AnimNotify_NotifyPhysics()
	UBPI_Interfaces_C.ChangeToRagdoll(self.Pawn)
end
```
Lua function's name must be **'AnimNotify_'** + **NotifyName**.

Lua 函数名必须是 **'AnimNotify_'** + **NotifyName**。

## Override Input Events 覆盖输入事件
![ACTION_AXIS_INPUTS](./Images/action_axis_inputs.png)

#### Action Inputs 动作输入
```
function BP_PlayerController_C:Aim_Pressed()
	UBPI_Interfaces_C.UpdateAiming(self.Pawn, true)
end

function BP_PlayerController_C:Aim_Released()
	UBPI_Interfaces_C.UpdateAiming(self.Pawn, false)
end
```
Lua function's name must be **ActionName** + **'_Pressed' / '_Released'**.

Lua 函数名必须是 **ActionName** + **'_Pressed' / '_Released'**。

#### Axis Inputs 坐标轴输入
```
function BP_PlayerController_C:Turn(AxisValue)
	self:AddYawInput(AxisValue)
end

function BP_PlayerController_C:LookUp(AxisValue)
	self:AddPitchInput(AxisValue)
end
```
Lua function's name must be the same with **AxisName**.

Lua 函数名必须和 **AxisName** 相同。

#### Key Inputs 键盘输入
```
function BP_PlayerController_C:P_Pressed()
	print("P_Pressed")
end

function BP_PlayerController_C:P_Released()
	print("P_Released")
end
```
Lua function's name must be **KeyName** + **'_Pressed' / '_Released'**.

Lua 函数名必须是 **KeyName** + **'_Pressed' / '_Released'**。

#### Other Inputs 其它输入
You can also override **Touch/AxisKey/VectorAxis/Gesture Inputs** in Lua.

输入 **Touch/AxisKey/VectorAxis/Gesture Inputs** 在 Lua 中也可以被覆盖。

## Override Replication Notifies 覆盖复制通知
If you are developing dedicated/listenning server&amp;clients game, you can override replication notifies in Lua codes:

如果你使用了 UE4 的网络复制方案，你可以在 Lua 中覆盖赋值通知：

![REP_NOTIFY](./Images/rep_notify.png)

```
function BP_PlayerCharacter_C:OnRep_Health(...)
	print("call OnRep_Health in Lua")
end
```

## Call Overridden Functions 调用已经被覆盖的方法
If you override a 'BlueprintEvent', 'AnimNotify' or 'RepNotify' in Lua, you can still access original function implementation.

如果在 Lua 中覆盖了蓝图事件、动画事件和复制通知，依然可以访问被覆盖掉的方法。

```
function BP_PlayerController_C:ReceiveBeginPlay()
	local Widget = UWidgetBlueprintLibrary.Create(self, UClass.Load("/Game/Core/UI/UMG_Main"))
	Widget:AddToViewport()
	self.Overridden.ReceiveBeginPlay(self)
end
```

**self.*Overridden*.ReceiveBeginPlay(self)** will call Blueprint implemented 'ReceiveBeginPlay'.

**self.*Overridden*.ReceiveBeginPlay(self)** 调用了蓝图中的 “ReceiveBeginPlay” 实现。

## Call Lua Functions in C++ 在 C++ 中调用 Lua 函数
UnLua also provides two generic methods to call global Lua funtions and functions in global Lua table in C++ codes.

UnLua 提供了两种方式通过 C++ 来调用 Lua 的全局函数和存在于全局 table 中的函数。

* Global functions 全局函数
```
template <typename... T>
FLuaRetValues Call(lua_State *L, const char *FuncName, T&&... Args);
```

* Functions in global table 存在于全局 table 中的函数
```
template <typename... T>
FLuaRetValues CallTableFunc(lua_State *L, const char *TableName, const char *FuncName, T&&... Args);
```

---

# Others 其它

* Lua Template File Export 导出 Lua 模板

You can export Lua template file for blueprints:

![TEMPLATE](./Images/lua_template.png)

The template file:

模板文件：

![TEMPLATE_CODES](./Images/generated_lua_template.png)

