# libxposed API 完整参考文档 (中文版)

- [概述](#概述)
  - [快速入门](#快速入门)
  - [入口注册](#入口注册)
  - [模块配置](#模块配置)
  - [作用域 (Scope)](#作用域-scope)
  - [Hook 模型](#hook-模型)
  - [Invoker 系统](#invoker-系统)
  - [模块生命周期回调](#模块生命周期回调)
  - [错误处理](#错误处理)
- [包 io.github.libxposed.api](#包-io-github-libxposed-api)
  - [接口 XposedInterface](#接口-xposedinterface)
    - [嵌套接口摘要](#xposedinterface-嵌套接口摘要)
    - [字段常量](#xposedinterface-字段常量)
    - [方法摘要](#xposedinterface-方法摘要)
  - [接口 XposedInterface.Chain](#接口-xposedinterfacechain)
    - [方法摘要](#xposedinterfacechain-方法摘要)
  - [接口 XposedInterface.CtorInvoker](#接口-xposedinterfacectorinvoker)
    - [方法摘要](#xposedinterfacectorinvoker-方法摘要)
  - [枚举 XposedInterface.ExceptionMode](#枚举-xposedinterfaceexceptionmode)
    - [常量](#xposedinterfaceexceptionmode-常量)
  - [接口 XposedInterface.HookBuilder](#接口-xposedinterfacehookbuilder)
    - [方法摘要](#xposedinterfacehookbuilder-方法摘要)
  - [接口 XposedInterface.Hooker](#接口-xposedinterfacehooker)
    - [方法摘要](#xposedinterfacehooker-方法摘要)
  - [接口 XposedInterface.HookHandle](#接口-xposedinterfacehookhandle)
    - [方法摘要](#xposedinterfacehookhandle-方法摘要)
  - [接口 XposedInterface.Invoker](#接口-xposedinterfaceinvoker)
    - [嵌套接口摘要](#xposedinterfaceinvoker-嵌套接口摘要)
    - [方法摘要](#xposedinterfaceinvoker-方法摘要)
  - [接口 XposedInterface.Invoker.Type](#接口-xposedinterfaceinvokertype)
    - [实现类](#xposedinterfaceinvokertype-实现类)
  - [类 XposedInterfaceWrapper](#类-xposedinterfacewrapper)
    - [构造函数](#xposedinterfacewrapper-构造函数)
    - [方法摘要](#xposedinterfacewrapper-方法摘要)
  - [类 XposedModule](#类-xposedmodule)
    - [构造函数](#xposedmodule-构造函数)
  - [接口 XposedModuleInterface](#接口-xposedmoduleinterface)
    - [方法摘要](#xposedmoduleinterface-方法摘要)
    - [嵌套接口摘要](#xposedmoduleinterface-嵌套接口摘要)
- [包 io.github.libxposed.api.error](#包-io-github-libxposed-apierror)
  - [类 XposedFrameworkError](#类-xposedframeworkerror)
    - [构造函数](#xposedframeworkerror-构造函数)
  - [类 HookFailedError](#类-hookfailederror)
    - [构造函数](#hookfailederror-构造函数)

---

## 概述

现代 Xposed 模块 API。本包提供了使用现代 Xposed 框架开发 Xposed 模块的公共 API，它用一个重新设计的、类型安全的接口取代了旧版 XposedBridge API。

### 快速入门

模块入口类应继承 [`XposedModule`](#类-xposedmodule)。框架会自动调用 `attachFramework()` 方法；在 `onModuleLoaded()` 被调用之前，模块**不**应执行初始化工作。

### 入口注册

- **Java 入口**: 在 `META-INF/xposed/java_init.list` 中列出（每行一个完整的类名）。
- **Native 入口**: 使用 `META-INF/xposed/native_init.list`。

将这些文件放在 `src/main/resources/META-INF/xposed/` 下，Gradle 会自动将它们打包到 APK 中。

### 模块配置

模块元数据通过标准的 Android 资源（`android:label` 作为模块名称，`android:description` 作为描述）和 `META-INF/xposed/module.prop`（Java `Properties` 格式）指定。必需的属性：

- `minApiVersion` – 所需的最低 Xposed API 版本
- `targetApiVersion` – 目标的 Xposed API 版本

可选属性：

- `staticScope` (boolean) – 模块作用域是否固定，用户不应在作用域列表之外的应用上启用模块。
- `exceptionMode` (string) [`protective`|`passthrough`] – 默认为 `protective`，参见 [`ExceptionMode`](#枚举-xposedinterfaceexceptionmode)。

### 作用域 (Scope)

模块作用域由 `META-INF/xposed/scope.list` 中的包名列表定义。框架会将模块注入到这些包所声明的所有常规进程中。注入后，该进程中加载的每个包都会触发回调。因此，模块可能会收到超出原始作用域包的回调，所以它们应始终根据进程名和包名进行过滤。

对于在 `android:sharedUserId="android.uid.system"` 包中声明了 `android:process="system"` 的组件，它们运行在 system server 中。对于那些所有组件都运行在 system server 中的包，将包名本身添加到作用域是无效的。相反，system server 用特殊的虚拟包名 `system` 表示，应明确使用它。

注意：`android` 包仍然是一个有效的作用域目标，因为它的一些组件声明了 `android:process=":ui"`，因此不运行在 system server 中。作用域为 `android` 的模块仍然可以接收到 `android` 包加载的事件，即使 `android` 包本身没有代码。相比之下，`com.android.providers.settings` 不是一个有效的作用域目标，模块应使用 `system` 作用域，然后等待 `com.android.providers.settings` 包加载事件。

### Hook 模型

API 使用**拦截器链**模型（类似于 OkHttp 的拦截器）。模块实现 [`Hooker`](#接口-xposedinterfacehooker) 接口及其 `intercept(Chain)` 方法。通过 `hook(Executable)` 返回的构建器来执行 Hook：

```java
HookHandle handle = hook(method)
    .setPriority(PRIORITY_DEFAULT)
    .setExceptionMode(ExceptionMode.PROTECTIVE)
    .intercept(chain -> {
        // 预处理
        Object result = chain.proceed();
        // 后处理
        return result;
    });
```

### Invoker 系统

要绕过访问检查调用原始（或被 Hook 的）方法，可以通过 `getInvoker(Method)` 或 `getInvoker(Constructor)` 获取 [`Invoker`](#接口-xposedinterfaceinvoker)。调用器类型控制执行 Hook 链的哪一部分（参见 [`Invoker.Type`](#接口-xposedinterfaceinvokertype)）。

### 模块生命周期回调

在 [`XposedModule`](#类-xposedmodule) 中重写以下回调：

- [`onModuleLoaded()`](#xposedmoduleinterface-方法摘要) – 当模块被加载到目标进程时调用一次。
- [`onPackageLoaded()`](#xposedmoduleinterface-方法摘要) – 当默认类加载器准备就绪时调用，在 `AppComponentFactory` 实例化之前（API 29+）。
- [`onPackageReady()`](#xposedmoduleinterface-方法摘要) – 在应用的类加载器创建之后调用。
- [`onSystemServerStarting()`](#xposedmoduleinterface-方法摘要) – 在 system server 启动时调用一次。此回调取代了第一个包加载阶段。

### 错误处理

框架级别的错误通过 [`XposedFrameworkError`](#类-xposedframeworkerror) 的子类报告。特别是 [`HookFailedError`](#类-hookfailederror) 表示致命的 Hook 失败，应报告给框架维护者，而不是由模块处理。

---

## 包 `io.github.libxposed.api`

这是现代 Xposed 模块 API 的核心包。

### 接口 `XposedInterface`

Xposed 框架提供给模块操作应用进程的核心接口。

#### `XposedInterface` 嵌套接口摘要

| 接口 | 描述 |
| :--- | :--- |
| `XposedInterface.Chain` | 方法或构造函数的**拦截器链**。 |
| `XposedInterface.CtorInvoker<T>` | **构造函数的调用器**，用于绕过访问检查创建实例。 |
| `XposedInterface.ExceptionMode` | **Hook 的异常处理模式**枚举。 |
| `XposedInterface.HookBuilder` | **Hook 的配置构建器**，用于设置优先级和异常模式。 |
| `XposedInterface.Hooker` | **Hooker 接口**，模块需要实现它来定义拦截逻辑。 |
| `XposedInterface.HookHandle` | **Hook 的句柄**，用于取消一个已经激活的 Hook。 |
| `XposedInterface.Invoker<T, U>` | **方法或构造函数的通用调用器**，用于绕过访问检查进行调用。 |

#### `XposedInterface` 字段常量

| 常量 | 类型 | 值 | 描述 |
| :--- | :--- | :--- | :--- |
| `API_101` | `int` | 101 | 第一个 API 版本。<br>**行为变化（所有模块）**：模块不能注入到 zygote；它们只在作用域进程内加载。<br>**行为变化（目标 API 101 或更高）**：这是第一个 API 版本。 |
| `LIB_API` | `int` | 101 | **此库**的 API 版本。模块应使用 `getApiVersion()` 检查运行时版本。 |
| `PROP_CAP_SYSTEM` | `long` | 1 << 0 | 框架有能力 Hook `system_server` 和其他系统进程。 |
| `PROP_CAP_REMOTE` | `long` | 1 << 1 | 框架提供远程配置（`SharedPreferences`）和远程文件支持。 |
| `PROP_RT_API_PROTECTION` | `long` | 1 << 32 | 框架禁止通过反射或动态加载的代码访问 Xposed API。 |
| `PRIORITY_DEFAULT` | `int` | 50 | 默认 Hook 优先级。 |
| `PRIORITY_LOWEST` | `int` | 0 | 最低优先级，在拦截器链的**末尾**执行。 |
| `PRIORITY_HIGHEST` | `int` | 100 | 最高优先级，在拦截器链的**开头**执行。 |

#### `XposedInterface` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `int` | `getApiVersion()` | 获取运行时的 Xposed API 版本。框架实现**不**应重写此方法。 |
| `String` | `getFrameworkName()` | 获取当前 Xposed 框架实现的名称。 |
| `String` | `getFrameworkVersion()` | 获取当前 Xposed 框架实现的版本。 |
| `long` | `getFrameworkVersionCode()` | 获取当前 Xposed 框架实现的版本号。 |
| `long` | `getFrameworkProperties()` | 获取 Xposed 框架的属性。前缀为 `PROP_RT_` 的属性可能在启动间发生变化。 |
| `HookBuilder` | `hook(Executable origin)` | **Hook 一个普通方法或构造函数**。 |
| `HookBuilder` | `hookClassInitializer(Class<?> origin)` | **Hook 一个类的静态初始化器 (`<clinit>`)**。<br>静态初始化器被视为一个无参数的 `static void` 方法。传递给 Hooker 的 `Chain` 中：`getExecutable()` 返回一个代表静态初始化器的合成 `Method`；`getThisObject()` 始终返回 `null`；`getArgs()` 返回空列表；`proceed()` 返回 `null`。<br>注意：如果类已经被初始化，该 Hook 永远不会被调用。 |
| `boolean` | `deoptimize(Executable executable)` | **去优化一个方法/构造函数**，用于解决因内联导致 Hook 不生效的问题。<br>通过去优化方法，运行时会回退到不内联地调用所有被调用者。例如，当一个短方法 B 被方法 A 调用时，Hook B 后回调没有被触发，这可能意味着 A 将 B 内联到了其方法体中。为了强制 A 调用被 Hook 的 B，你可以去优化 A，然后 Hook 就会生效。<br>通常你需要找到所有被 Hook 方法的调用者，这很难实现（但你可以使用 DexKit 搜索所有调用者）。如果你确定去优化的调用者就是你所需要的，才使用此方法。否则，最好更改 Hook 点或手动去优化整个应用（简单地重新安装应用而不卸载）。 |
| `Invoker<?, Method>` | `getInvoker(Method method)` | 获取一个**方法调用器**。调用器将绕过访问检查。默认类型是 `FULL`。 |
| `CtorInvoker<T>` | `getInvoker(Constructor<T> constructor)` | 获取一个**构造函数的调用器**。调用器将绕过访问检查。默认类型是 `FULL`。 |
| `void` | `log(int priority, String tag, String msg)` | 向 Xposed 日志中写入一条消息。 |
| `void` | `log(int priority, String tag, String msg, Throwable tr)` | 向 Xposed 日志中写入一条带异常信息的消息。 |
| `ApplicationInfo` | `getModuleApplicationInfo()` | 获取模块自身的 `ApplicationInfo`。 |
| `SharedPreferences` | `getRemotePreferences(String group)` | 获取存储在 Xposed 框架中的远程配置。注意：在 Hook 的应用中是**只读**的。如果框架是嵌入式的，会抛出 `UnsupportedOperationException`。 |
| `String[]` | `listRemoteFiles()` | 列出模块在框架中共享数据目录下的所有文件。如果框架是嵌入式的，会抛出 `UnsupportedOperationException`。 |
| `ParcelFileDescriptor` | `openRemoteFile(String name)` | **只读**方式打开模块在框架中共享数据目录下的一个文件。文件名不能包含路径分隔符、`.` 或 `..`。如果文件不存在或路径被禁止，抛出 `FileNotFoundException`；如果框架是嵌入式的，抛出 `UnsupportedOperationException`。 |

---

### 接口 `XposedInterface.Chain`

拦截器链，代表一次方法或构造函数的调用。`Chain` 对象不能在线程间共享，也不能在 `Hooker.intercept(Chain)` 结束后重用。

#### `XposedInterface.Chain` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `Executable` | `getExecutable()` | 获取当前被 Hook 的方法或构造函数。 |
| `Object` | `getThisObject()` | 获取调用的 `this` 对象。对于静态方法，返回 `null`。 |
| `List<Object>` | `getArgs()` | 获取调用参数的不可变列表。 |
| `Object` | `getArg(int index)` | 获取指定索引位置的参数。如果索引越界，抛出 `IndexOutOfBoundsException`；如果类型转换失败，抛出 `ClassCastException`。 |
| `Object` | `proceed()` | **继续执行拦截器链**，使用相同的 `this` 对象和参数。返回下一个拦截器或原始可执行对象的结果。对于 void 方法和构造函数，始终返回 `null`。 |
| `Object` | `proceed(Object[] args)` | **继续执行拦截器链**，使用相同的 `this` 对象和新的参数。 |
| `Object` | `proceedWith(Object thisObject)` | **继续执行拦截器链**，使用新的 `this` 对象和相同的参数。静态方法拦截器不应调用此方法。 |
| `Object` | `proceedWith(Object thisObject, Object[] args)` | **继续执行拦截器链**，使用新的 `this` 对象和新的参数。静态方法拦截器不应调用此方法。 |

---

### 接口 `XposedInterface.CtorInvoker<T>`

构造函数专用的调用器，继承自 `XposedInterface.Invoker`。

#### `XposedInterface.CtorInvoker` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `T` | `newInstance(Object... args)` | 通过由调用器类型决定的 Hook 链**创建一个新实例**。 |
| `U` | `newInstanceSpecial(Class<U> subClass, Object... args)` | 为给定的子类创建实例，但使用父类的构造器进行初始化。这会跳过子类构造器的初始化，可能导致对象处于无效状态（子类字段未初始化）。当你需要自己初始化子类中的某些字段时，此方法很有用。 |

---

### 枚举 `XposedInterface.ExceptionMode`

Hook 的异常处理模式。默认模式为 `DEFAULT`。

#### `XposedInterface.ExceptionMode` 常量

- **`DEFAULT`**: 遵循 `module.prop` 中的全局配置。如果未指定，默认是 `PROTECTIVE`。
- **`PROTECTIVE`** (推荐): Hooker 抛出的任何异常都会被**捕获并记录日志**，然后调用会**像没有这个 Hook 一样继续执行**。这可以防止因 Hook 代码错误导致应用崩溃。
  - 如果异常在 `proceed()` 之前抛出，框架会跳过该 Hook 继续链；如果异常在 `proceed()` 之后抛出，框架会返回 `proceed` 的结果/异常。
  - 由 `proceed()` 抛出的异常总是会被传播。
- **`PASSTHROUGH`**: Hooker 抛出的异常会**原样传播给调用者**。这主要用于调试目的。

---

### 接口 `XposedInterface.HookBuilder`

用于配置和最终安装 Hook 的构建器。

#### `XposedInterface.HookBuilder` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `HookBuilder` | `setPriority(int priority)` | 设置当前 Hook 的优先级。优先级越高，越早执行。默认优先级为 `PRIORITY_DEFAULT`。 |
| `HookBuilder` | `setExceptionMode(ExceptionMode mode)` | 设置当前 Hook 的异常处理模式。默认模式为 `DEFAULT`。 |
| `HookHandle` | `intercept(Hooker hooker)` | 设置 `Hooker` 实例并**最终构建和安装这个 Hook**。如果 origin 是框架内部或 `Constructor.newInstance()`，或者 hooker 无效，抛出 `IllegalArgumentException`；如果 Hook 因框架内部错误失败，抛出 `HookFailedError`。 |

---

### 接口 `XposedInterface.Hooker`

Hook 器，模块需实现此接口来提供具体的拦截逻辑。

#### `XposedInterface.Hooker` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `Object` | `intercept(Chain chain)` | **拦截一个方法或构造函数的调用**。在此方法中，通常应先进行预处理，然后调用 `chain.proceed()`，最后进行后处理并返回结果。如果 Hooker 不希望改变结果，应调用 `chain.proceed()` 并返回其结果。对于 void 方法和构造函数，返回值被框架忽略。抛出任何异常都会根据异常模式传播。 |

---

### 接口 `XposedInterface.HookHandle`

一个已安装 Hook 的句柄。

#### `XposedInterface.HookHandle` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `Executable` | `getExecutable()` | 获取此 Hook 关联的方法或构造函数。 |
| `void` | `unhook()` | **取消这个 Hook**。此方法是幂等的，可以安全地多次调用。 |

---

### 接口 `XposedInterface.Invoker<T, U>`

方法或构造函数的通用调用器，所有调用都会绕过 Java 的访问权限检查。

#### `XposedInterface.Invoker` 嵌套接口摘要

- `XposedInterface.Invoker.Type`: 调用器的类型，它决定了会执行 Hook 链的哪一部分。

#### `XposedInterface.Invoker` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `T` | `setType(Type type)` | 设置调用器的**类型**，从而控制执行哪部分 Hook 链。 |
| `Object` | `invoke(Object thisObject, Object... args)` | 通过由调用器类型决定的 Hook 链**调用方法/构造函数**。对于非静态调用，`thisObject` 是 `this` 指针，否则为 `null`。返回调用结果，void 方法和构造函数返回 `null`。 |
| `Object` | `invokeSpecial(Object thisObject, Object... args)` | 执行**非虚方法调用**（类似于 JNI 的 `CallNonVirtual`），直接调用指定类中的方法，无视子类的覆写。这在需要调用 `super.xxx()` 时很有用。 |

---

### 接口 `XposedInterface.Invoker.Type`

调用器类型的标记接口。此接口是密封的（sealed），只允许以下两个实现。

#### `XposedInterface.Invoker.Type` 实现类

- **`Type.Origin`** (record): 用于**调用原始的方法/构造函数，跳过所有 Hook**。
  - 常量 `XposedInterface.Invoker.Type.ORIGIN` 是其一个便利实例。
  - 提供标准的 `toString()`、`hashCode()` 和 `equals()` 方法。

- **`Type.Chain`** (record): 用于从 Hook 链的**中间开始执行**，它会跳过所有优先级高于给定 `maxPriority` 值的 Hook。
  - 常量 `XposedInterface.Invoker.Type.Chain.FULL` 是一个便利实例，代表**执行完整的 Hook 链**（传入 `PRIORITY_HIGHEST` 作为 `maxPriority`）。
  - 包含一个 `int maxPriority()` 方法，返回最大优先级阈值。
  - 提供标准的 `toString()`、`hashCode()` 和 `equals()` 方法。

---

### 类 `XposedInterfaceWrapper`

`XposedInterface` 的包装类，用于屏蔽框架的实现细节。模块通常不直接使用它。

#### `XposedInterfaceWrapper` 构造函数

| 构造函数 | 描述 |
| :--- | :--- |
| `XposedInterfaceWrapper()` | 创建一个未附加框架的包装器实例。 |

#### `XposedInterfaceWrapper` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `void` | `attachFramework(XposedInterface base)` | 将框架接口附加到模块。**模块永远不应调用此方法**。 |
| `int` | `getApiVersion()` | 获取运行时 API 版本。 |
| `String` | `getFrameworkName()` | 获取框架名称。 |
| `String` | `getFrameworkVersion()` | 获取框架版本。 |
| `long` | `getFrameworkVersionCode()` | 获取框架版本号。 |
| `long` | `getFrameworkProperties()` | 获取框架属性。 |
| `HookBuilder` | `hook(Executable origin)` | Hook 方法/构造函数。 |
| `HookBuilder` | `hookClassInitializer(Class<?> origin)` | Hook 类的静态初始化器。 |
| `boolean` | `deoptimize(Executable executable)` | 去优化方法/构造函数。 |
| `Invoker<?, Method>` | `getInvoker(Method method)` | 获取方法调用器。 |
| `CtorInvoker<T>` | `getInvoker(Constructor<T> constructor)` | 获取构造函数调用器。 |
| `void` | `log(int priority, String tag, String msg)` | 写入日志。 |
| `void` | `log(int priority, String tag, String msg, Throwable tr)` | 写入日志（带异常）。 |
| `ApplicationInfo` | `getModuleApplicationInfo()` | 获取模块应用信息。 |
| `SharedPreferences` | `getRemotePreferences(String name)` | 获取远程配置。 |
| `String[]` | `listRemoteFiles()` | 列出远程文件。 |
| `ParcelFileDescriptor` | `openRemoteFile(String name)` | 打开远程文件（只读）。 |

- **直接已知子类**: `XposedModule`

---

### 类 `XposedModule`

**所有 Xposed 模块入口类都应继承的基类**。每个进程会且只会实例化一次。

- **继承关系**: `java.lang.Object` → `XposedInterfaceWrapper` → `XposedModule`
- **实现的接口**: `XposedInterface`, `XposedModuleInterface`

模块不应在 `onModuleLoaded()` 被调用之前执行初始化工作。

#### `XposedModule` 构造函数

| 构造函数 | 描述 |
| :--- | :--- |
| `XposedModule()` | 创建一个模块实例。 |

---

### 接口 `XposedModuleInterface`

模块初始化的主接口。`XposedModule` 实现了它。

#### `XposedModuleInterface` 方法摘要

| 返回值 | 方法名 | 描述 |
| :--- | :--- | :--- |
| `void` | `onModuleLoaded(ModuleLoadedParam param)` | **当模块被加载到目标进程时调用**。此回调在进程的生命周期中**保证只被调用一次**。回调抛出的所有 `RuntimeException` 会被捕获并记录。 |
| `void` | `onPackageLoaded(PackageLoadedParam param)` | **当一个有代码的包被加载到进程时调用**。此时默认的 `ClassLoader` 已经就绪，但在 `AppComponentFactory` 实例化之前（API 29+）。<br>每个包名在进程内只会调用一次。注意一个进程可能加载多个包（如 `sharedUserId` 和 `createPackageContext`）。在 system server 中，第一个回调被 `onSystemServerStarting` 替代，因此 `param.isFirstPackage()` 永远不会为 `true`。 |
| `void` | `onPackageReady(PackageReadyParam param)` | **当包的 `ClassLoader` 完全准备好后调用**。此时 `AppComponentFactory` 已经实例化了类加载器，准备创建 `Application`。<br>类似于 `onPackageLoaded`，每个包名只会调用一次。在 system server 中，第一个回调被 `onSystemServerStarting` 替代。 |
| `void` | `onSystemServerStarting(SystemServerStartingParam param)` | **当 system_server 准备启动关键服务时调用**。这个回调会替换 system_server 中的第一个包加载阶段（即 `onPackageLoaded` 和 `onPackageReady` 的第一个回调）。 |

#### `XposedModuleInterface` 嵌套接口摘要

| 接口 | 描述 |
| :--- | :--- |
| `XposedModuleInterface.ModuleLoadedParam` | 包装了模块被加载时的进程信息。 |
| `XposedModuleInterface.PackageLoadedParam` | 包装了被加载包的信息。 |
| `XposedModuleInterface.PackageReadyParam` | 包装了包就绪时的信息，继承自 `PackageLoadedParam`。 |
| `XposedModuleInterface.SystemServerStartingParam` | 包装了 system_server 启动时的信息。 |

**`ModuleLoadedParam` 方法**:
- `boolean isSystemServer()` – 当前进程是否为 system_server
- `String getProcessName()` – 获取进程名称

**`PackageLoadedParam` 方法**:
- `String getPackageName()` – 获取包名
- `ApplicationInfo getApplicationInfo()` – 获取 ApplicationInfo
- `boolean isFirstPackage()` – 是否为进程中第一个/主包
- `ClassLoader getDefaultClassLoader()` – 获取默认类加载器（API 29+）

**`PackageReadyParam` 方法**（继承 `PackageLoadedParam`）:
- `ClassLoader getClassLoader()` – 获取包类加载器（可能与默认不同）
- `AppComponentFactory getAppComponentFactory()` – 获取 AppComponentFactory（API 28+）

**`SystemServerStartingParam` 方法**:
- `ClassLoader getClassLoader()` – 获取 system_server 类加载器

---

## 包 `io.github.libxposed.api.error`

此包包含了框架在遇到错误时会抛出的异常类。

### 类 `XposedFrameworkError`

表示 Xposed 框架的功能出现问题。这是一个 `Error`，而不是 `Exception`。

- **继承关系**: `java.lang.Object` → `Throwable` → `Error` → `XposedFrameworkError`
- **直接已知子类**: `HookFailedError`

#### `XposedFrameworkError` 构造函数

| 构造函数 | 描述 |
| :--- | :--- |
| `XposedFrameworkError(String message)` | 带消息构造。 |
| `XposedFrameworkError(String message, Throwable cause)` | 带消息和原因构造。 |
| `XposedFrameworkError(Throwable cause)` | 带原因构造。 |

### 类 `HookFailedError`

表示一次 Hook 操作因为框架内部错误而失败。

- **继承关系**: `...` → `Error` → `XposedFrameworkError` → `HookFailedError`
- **设计说明**: 此类继承自 `Error`，表明它是致命的框架 Bug。**模块开发者不应尝试捕获此错误**，而应向框架维护者报告问题。

#### `HookFailedError` 构造函数

| 构造函数 | 描述 |
| :--- | :--- |
| `HookFailedError(String message)` | 带消息构造。 |
| `HookFailedError(String message, Throwable cause)` | 带消息和原因构造。 |
| `HookFailedError(Throwable cause)` | 带原因构造。 |

---
