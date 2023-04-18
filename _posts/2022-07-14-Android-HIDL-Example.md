---
title: Android HAL 与 HIDL 开发实例
author: Mista
date: 2022-07-14 00:55:00 +0800
categories: [Android build]
tags: [HAL & HIDL]
---

# 关于 Android HAL 与 HIDL

Android 8.0之后，Google 为了解决 Android 版本碎片化问题，推出了 treble 架构，核心思想就是 vendor 分区和 system 分区的隔离。把厂商的修改限制在 vendor 分区，system 分区由 Google 把控。system 访问vendor 分区的话，需要通过 HIDL 的形式来访问。 `/dev/binder`拓展多出了两个域，即`/dev/hwbinder`和`/dev/vndbinder`，

* `/dev/hwbinder` 主要用于 HIDL 接口的通信，
* `/dev/vndbinder` 则是用于 vendor 进程之间的 AIDL 通信。

作为 OEM/ODM 厂商，需要了解 Android 硬件的开发和集成流程，把自己硬件添加到 ROM 中，比如集成杜比音效。

> 关于 ROM：国内的定制系统开发者，经常会陷入自己的产品究竟是应该称为 OS (ColorOS) 还是 UI (MIUI) 的争论，为了避免此类争论和表示谦虚，会自称为 ROM。应该就是 Android /system分区。

## [HAL 硬件抽象层](https://source.android.com/docs/core/architecture/hal)

HAL，Hardware Abstraction Layer，即硬件抽象层。

从碎片化角度看，系统设计者希望底层硬件按照类型整齐划一，而不是半导体厂商各自的接口标准不一；从商业角度看，OEMs 自家的硬件驱动是不愿意开源出去被人研究，所以要求 OS 可以无视底层实现，只需商定统一的交互协议。

对于 Android 而言，这层抽象就是HAL，简而言之，Android HAL 就是定义了 `.h` 接口，并由 OEMs 实现接口为动态链接库 `.so`，并使用商定的方式去加载和调用。

现在已经是 Android 13，但早在 Android 8 之后就弃用了 HAL，不过由于碎片化的原因，目前还有 IoT 等设备仍在使用传统的 HAL 模式。

## [HIDL - HAL 接口定义语言](https://source.android.com/docs/core/architecture/hidl)

HAL 是最初的硬件抽象方案，在 Android 8 之后废弃并被 HIDL 取代。

HIDL，HAL Interface Definition Language，和AIDL类似，是用来描述硬件接口的语言。HIDL 设计的初衷是更新 frameworks 时避免重新编译 HAL，HAL 可以由厂商单独编译并定义在 vendor 分区中单独更新，以及进行版本管理。

# HIDL 开发实例

以下参考[AOSP相关实现](https://cs.android.com/android/platform/superproject/+/master:hardware/interfaces/audio/)，从零创建自己的HIDL服务，

## 1 添加HAL目录

在 Android HAL（Hardware Abstraction Layer）中，HAL库的源代码通常存储在 `/hardware` 目录下。如果要添加新的HAL库，则需要在该目录下创建一个新的目录，并将库的源代码放在其中。以下是添加新HAL库的一般步骤：

1. 在 `/hardware/interfaces` 目录下创建一个新目录，用于存储新的HAL库的源代码。
2. 在新的目录中添加 HAL 库的源代码。具体来说，你需要添加 `.h` 头文件和 `.cpp` 源文件，其中包括实现 HAL 接口的函数。
3. 在 `Android.bp` 文件中添加库的定义。在 `/hardware/interfaces` 目录下，每个 HAL 库都有一个相应的 `Android.bp` 文件，用于定义库的编译选项和依赖项。要添加新的库，请编辑相应的 `Android.bp` 文件，并添加库的定义。
4. 在 `Android.mk` 文件中添加库的定义。`Android.mk` 文件是一个传统的 Makefile，用于定义编译选项和依赖项。要添加新的库，请编辑相应的 `Android.mk` 文件，并添加库的定义。
5. 运行 `make` 命令来编译新的 HAL 库。编译完成后，库文件将存储在 `/system/lib/hw` 目录下，以 `.so` 为后名。

代码目录结构，HAL 层只是定义接口标准，具体的实现在 `/vender` 由 OEMs 负责，

```shell
/hardware/interfaces/audio$ tree
├── current.txt
├── effect
│   └── 1.0
│       ├── Android.bp
│       └── IEffect.hal
└── update-makefiles.sh
......
/vender/../audio$ tree
├── my-product.mk
├── effect
│   └── 1.0
│       ├── default
│       │   ├── Android.bp
│       │   ├── android.hardware.audio.effect@1.0-service.rc
│       │   ├── Effect.cpp
│       │   └── Effect.h
│       └── test
│           ├── Android.bp
│           └── TVServerTest.cpp
└── sepolicy... // 一般杜比等三方会提供所有文件（sepolicy、manifest.xml等）到vendor，需平台自行添加到自家指定仓库
```

以音效 IEffect 为例，在 platform/hardware/interfaces 目录下添加 audioeffect，

```shell
mkdir -phardware/interfaces/audio/effect/1.0
```

## 2 定义 HAL 接口

定义 hal 接口，IEffect.hal，

```java
package android.hardware.audio.effect@1.0;

import android.hardware.audio.common.V6_0.EffectUuid;
import android.hardware.audio.common.V6_0.AudioEffectConfig;

interface IEffect {
    // The unique identifier for this effect.
    EffectUuid getUuid();

    // Configure the effect with the given configuration.
    bool configure(const AudioEffectConfig& config);

    // Enable or disable the effect.
    bool enable(bool enable);

    // Get the current state of the effect.
    bool isEnabled();

    // Process the given input data and write the output to the output buffer.
    void process(const int16_t* input, size_t inputSize, int16_t* output, size_t outputSize);
}
```

在上面的接口中，我们定义了一个 IEffect 接口，用于描述音效效果器的基本功能。它包括

- getUuid() 获取效果器的唯一标识符，
- configure() 配置效果器，
- enable() 启用或禁用效果器，
- isEnabled() 获取效果器的状态，
- process() 对输入数据进行处理并将输出写入输出缓冲区。

接下来，我们需要实现这个接口。

在 HAL 库的源代码中，我们需要创建一个名为 IEffect 的类，并实现接口中定义的方法。

## 3 根据 HAL 文件自动生成 CPP

HIDL 文件可以通过 HIDL 工具链来生成对应的 C++ 代码。

在 Android 源代码中，HIDL 工具链通常已经预先安装在编译环境中，可以使用以下命令来生成 C++ 代码，

```shell
hidl-gen -Lc++-impl -o <output-directory> <input-file>.hal
```

其中，<output-directory> 表示输出目录的路径，<input-file> 表示 HIDL 文件的名称。

这个命令生成 <interface-name>.cpp 和 <interface-name>.h ，分别对应 HIDL 接口的实现和头文件。

举例 IEffect.hal 来说，可以在 Android 源代码 hardware/interface/audio/effect 目录下执行以下命令：

```shell
hidl-gen -Lc++-impl -o . IEffect.hal
```

这将会在当前目录下生成 IEffect.cpp 和 IEffect.h 两个文件，就可以在 HAL 模块的源代码中引用这些文件并开始编写 HAL 接口的实现了。IEffect.cpp 实现如下，

```c++
#include "IEffect.h"
#include "Effect.h"

namespace android {
namespace hardware {
namespace audioeffect {
namespace V6_0 {
namespace implementation {

IEffect::IEffect() {
    mEffect = new Effect();
}

IEffect::~IEffect() {
    delete mEffect;
}

EffectUuid IEffect::getUuid() {
    return mEffect->getUuid();
}

bool IEffect::configure(const AudioEffectConfig& config) {
    return mEffect->configure(config);
}

bool IEffect::enable(bool enable) {
    return mEffect->enable(enable);
}

bool IEffect::isEnabled() {
    return mEffect->isEnabled();
}

void IEffect::process(const int16_t* input, size_t inputSize, int16_t* output, size_t outputSize) {
    mEffect->process(input, inputSize, output, outputSize);
}

}  // namespace implementation
}  // namespace V6_0
}  // namespace audioeffect
}  // namespace hardware
}  // namespace android
```

上面的代码实现了 IEffect 类，并将其绑定到了我们自己实现的 Effect 类上。Effect 类是一个真正实现音效处理逻辑的类，它可以在这里定义。在 IEffect 类的构造函数中，我们创建了一个新的 Effect 实例，并在 IEffect 类的其他方法中调用它的方法。

## 4 开机自启服务

要将 IEffect 设置为开机自启服务，需要在 Android 系统的启动流程中添加相应的代码。

以下是一些基本步骤：

### 4.1 实现一个 IServiceManager 的服务，以便能够管理和启动其他服务。

例如，在 IEffect 的 HAL 实现中，可以添加以下代码：

```C++
#include <android/hidl/manager/1.2/IServiceManager.h>
using android::hardware::defaultServiceManager;

int main() {
    // 获取服务管理器
    sp<IServiceManager> manager = defaultServiceManager();
    
    // 注册 IEffect 实现
    IEffectImpl* effect = new IEffectImpl();
    manager->addService(IEffect::descriptor, effect);
    
    // 等待进程退出
    android::hardware::joinRpcThreadpool();
    return 0;
}
```

### 4.2 将 HAL 模块的可执行文件添加到 Android 系统启动脚本中。

在default目录下创建android.hardware.audio.effect@1.0-service.rc文件：

```xml
service android.hardware.audio.effect /vendor/bin/hw/android.hardware.audio.effect@1.0-service
    class hal
    user root
    group root
```

其中，

- /vender/bin/hw 表示 HAL 模块的可执行文件路径，
- class hal 表示该服务是一个 HAL 模块的系统服务，
- user root 和 group root 表示该服务以 root 用户和组的身份运行。

## 5 生成 Android.bp

同上，`hidl-gen -Landroidbp-impl` 生成 Android.bp，

```shell
hidl-gen -Landroidbp-impl -o . IEffect.hal
```

下面是 `hidl-gen` 工具生成的 `android.hardware.audio.effect@1.0-service` 模块的 Android.bp 示例：

```
// Android.bp

// 定义模块名
// 生成的库文件名为 android.hardware.audio.effect@1.0-service.so
// 生成的 HIDL 接口头文件名为 IEffect.hal
// 生成的 HIDL 接口实现文件名为 IEffect.cpp
cc_binary {
    name: "android.hardware.audio.effect@1.0-service",
    init_rc: ["android.hardware.audio.effect@1.0-service.rc"],
    relative_install_path: "hw",
    vendor: true,
    srcs: [
        "IEffect.cpp",
    ],
    shared_libs: [
        "libhidlbase",
        "libhidltransport",
        "libutils",
        "android.hardware.audio.effect@1.0",
    ],
}
```

rc 文件会被安装到 /vendor/etc/init/ 目录下，系统启动时会自动加载这个目录下的所有 rc 文件

## 6 自动生成 HAL 接口的 Android.bp

[hardware/interfaces/update-makefiles.sh](https://cs.android.com/android/platform/superproject/+/master:hardware/interfaces/update-makefiles.sh) 生成 HAL 接口的 Android.bp，

```
// This file is autogenerated by hidl-gen -Landroidbp.

hidl_interface {
    name: "android.hardware.audio.effect@1.0",
    root: "android.hardware",
    product_specific: true,
    srcs: [
        "IEffect.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],
    gen_java: true,
    gen_java_constants: false,
}
```

`hidl_interface` 模块的 root 属性作为默认路径，和 `hidl_package_root` 的作用一样，是告诉 hidl-gen 生成的文件的根目录在哪里。如果你没有指定 hidl_package_root，使用 root 路径。

## 7 更新current.txt hash

每次更改 HAL 接口定义时，都需要更新哈希值，并将其添加到 current.txt 文件中，以便系统服务管理器可以正确加载与该接口对应的 HAL 模块。

要将 IEffect 添加到 current.txt 文件中，除了添加 android.hardware.audio.effect@1.0::IEffect 外，还应该更新该接口对应的哈希值。

运行 `hidl-gen` 命令以获取 IEffect 接口的哈希值：

```shell
hidl-gen -Lhash android.hardware.audio.effect@1.0::IEffect
```

将输出的哈希值复制到 current.txt 文件中 android.hardware.audio.effect@1.0::IEffect 行的开头。确保使用制表符分隔哈希值和接口名称，如下所示：

```
[哈希值]    android.hardware.audio.effect@1.0::IEffect
```

## 8 编译和部署 HAL 模块

```shell
mmm hardware/interfaces/audio/effect/1.0
```

编译完成后，将在 `out/target/product/[设备名称]/vendor/lib/hw` 目录下生成 `android.hardware.audio.effect@1.0-service.so` 动态库文件。这是 IEffect 接口的 HAL 实现模块。

可以使用 `adb push` 命令将 HAL 模块的可执行文件和其他必要的文件复制到设备上。

## 9 更新 manifest.xml

要更新 IEffect 的 manifest.xml 文件，以便系统服务管理器可以加载和管理该 HAL 接口模块，您可以按照以下步骤操作：

1. 进入 HAL 实现模块的目录，例如 `out/target/product/[设备名称]/vendor/lib/hw`。

2. 找到名为 `android.hardware.audio.effect@1.0-[设备名称].so` 的动态库文件，并确定其路径。

3. 进入 Android 源代码根目录，找到名为 `android.hardware.audio.effect@1.0-[设备名称].xml` 的默认 HAL 描述文件。

   如果该文件不存在，则可以将 `android.hardware.audio.effect@1.0.xml` 文件复制到当前目录并重命名为 `android.hardware.audio.effect@1.0-[设备名称].xml`。

4. 打开 `android.hardware.audio.effect@1.0-[设备名称].xml` 文件，并按照以下示例格式，更新 <hal> 标记的 path 属性，以指向 IEffect 接口的实现模块路径：

```xml
<hal format="hidl">
    <name>IEffect</name>
    <transport>hwbinder</transport>
    <version>1.0</version>
    <interface>
        <name>android.hardware.audio.effect@1.0::IEffect</name>
        <instance>default</instance>
    </interface>
</hal>
```

请将 `[设备名称]` 替换为要部署 HAL 模块的设备名称。

现在，IEffect 的 manifest.xml 文件已经更新。请注意，需要将更新后的 manifest.xml 文件 push 到您的 Android 设备上，以便系统服务管理器可以正确加载和管理 IEffect 接口的 HAL 模块。

## 10 添加到 MakeFile 文件

系统构建时，

将 IEffect 的 manifest.xml 文件复制到 Android 系统镜像的 vendor/hw/ 目录下，

```makefile
PRODUCT_COPY_FILES += \\
    hardware/interfaces/android.hardware.audio.effect@1.0.xml:vendor/hw/android.hardware.audio.effect@1.0.xml
```

将 android.hardware.audio.effect@1.0-service 添加到 Android 系统镜像中，

```makefile
PRODUCT_PACKAGES += \\
    android.hardware.audio.effect@1.0-service
```

- `PRODUCT_PACKAGES` 变量用于将某个**软件包**添加到 Android 系统镜像中。当您在 PRODUCT_PACKAGES 变量中指定一个软件包时，**Android 系统构建过程将包含该软件包，并在设备启动时加载和运行它**。通常，这些软件包是系统服务、库、应用程序等。例如，如果将 `android.hardware.audio.effect@1.0 HAL` 接口添加到 PRODUCT_PACKAGES 中，则系统服务管理器会在启动时加载和管理该 HAL 接口模块。
- `PRODUCT_COPY_FILES` 变量用于将**文件**复制到 Android 系统镜像中。在 Android 系统启动时，这些文件将位于设备的文件系统中。您可以使用 PRODUCT_COPY_FILES 变量**将文件从源代码树复制到 Android 系统镜像中的特定位置**，例如库文件、HAL 接口模块、应用程序二进制文件等。

总之，PRODUCT_COPY_FILES 变量只是复制文件到 Android 系统镜像，但 PRODUCT_PACKAGES 变量会在Android 系统构建过程将包含该软件包，并在设备启动时加载和运行它。

## 11 添加 SELINUX 规则

为了确保 Android 系统的安全性，Android 通过 SELinux（Security-Enhanced Linux）机制实现了强制访问控制（MAC）模型，对于 Android 设备上的各种进程、服务和文件等资源进行访问控制。因此，当我们在 Android 系统中添加新的 HAL 接口模块时，需要为其添加相应的 SELinux 规则，以确保该模块在系统中运行时不会破坏系统的安全性。

以下是添加 IEffect HAL 接口模块的 SELinux 规则的示例：

```
# ieffect.te

# Define the IEffect HAL service type
type ieffect_service,                 # SELinux上下文
     hal_service_type;

# Define the IEffect HAL interface type
type ieffect,                         # SELinux上下文
     interface_type;

# Define the SELinux domain for the IEffect HAL service
domain ieffect_service domain_type;

# Define the SELinux domain for the IEffect HAL implementation
domain ieffect_impl domain_type;

# Allow the IEffect HAL service to communicate with the IEffect HAL implementation
allow ieffect_service ieffect:interface { find getattr setattr };
allow ieffect_service ieffect_impl:process { sigchld };
allow ieffect_service ieffect_impl:unix_stream_socket { connectto };

# Allow the IEffect HAL implementation to access its own domain
allow ieffect_impl ieffect_impl:process { fork getattr open read write };
allow ieffect_impl self:process { sigchld };
```

以上是针对 IEffect HAL 接口模块的 SELinux 规则的示例，其中：

- `ieffect_service` 表示 IEffect HAL 服务的 SELinux 上下文；
- `ieffect` 表示 IEffect HAL 接口的 SELinux 上下文；
- `ieffect_service` 和 `ieffect_impl` 表示 IEffect HAL 服务和实现的 SELinux 域；
- `hal_service_type` 和 `interface_type` 是 Android HAL 框架中预定义的 SELinux 类型，用于表示 HAL 服务和接口的类型；
- `find、getattr、setattr、process、sigchld、unix_stream_socket、connectto、open、read、write` 是 SELinux 策略中常用的权限规则，用于控制不同 SELinux 上下文之间的访问和通信。

`IEffect.te` 是 SELinux 策略文件，定义了 IEffect HAL 接口需要的权限。在 IEffect.te 中，我们定义了一些 SELinux 上下文（context），如 ieffect_hwservice_context，该上下文定义了 HAL 服务在 SELinux 中所需的安全上下文。

`file_contexts、hwservice.te、hwservice_contexts` 是用于在 Android 系统中配置 SELinux 的文件。

* file_contexts 文件中列出了 **Android 文件系统中每个文件**的安全上下文，
* hwservice.te 和 hwservice_contexts 文件则定义了 **HAL 服务**在 SELinux 中所需的安全上下文和访问权限。

当我们将 IEffect.te 编译成二进制格式后，就可以将其与 file_contexts、hwservice.te 和 hwservice_contexts 文件一起打包进 Android 系统的 sepolicy 文件中。这样，在 Android 系统启动时，SELinux 策略会自动加载并应用到系统中。

## 12 测试

目前已经定义好了 IEffect 接口并生成了对应的 C++ 客户端和服务端代码，以下编写一个客户端程序，以调用 IEffect 服务。

### 12.1 编写客户端调用 IEffect 服务

在客户端代码中，引入 IEffect.h 头文件，并使用 `IEffect::getService()` 方法来获取一个指向 IEffect 接口的代理对象。

```c++
#include <android/hardware/sample/ieffect/1.0/IEffect.h>
using android::hardware::sample::ieffect::V1_0::IEffect;
using android::hardware::sample::ieffect::V1_0::IEffectCallback;

sp<IEffect> effect = IEffect::getService();
```

调用 IEffect 接口的方法，通过代理对象与服务端通信。

```c++
int32_t status = effect->setParameter(0, 0);
if (status != 0) {
    ALOGE("Failed to set parameter: %d", status);
    return status;
}

sp<IEffectCallback> callback = new MyEffectCallback();
status = effect->command(0, callback);
if (status != 0) {
    ALOGE("Failed to send command: %d", status);
    return status;
}
```

这里的 `setParameter()` 和 `command()` 是 IEffect 接口中的方法。

* `setParameter()` 方法将两个整数作为输入参数，并将结果作为输出参数返回。
* `command()` 方法需要一个整数和一个 IEffectCallback 接口的实例作为输入，并将执行结果通过 IEffectCallback 接口回传给客户端。

在完成对 IEffect 服务的调用后，需要调用effect->unlinkToDeath()方法解除与该服务的连接。

```c++
effect->unlinkToDeath(deathRecipient);
```

 `unlinkToDeath()` 方法用于取消注册服务的死亡通知，参数 deathRecipient 是实现了 android::hardware::hidl_death_recipient 接口的对象，用于处理服务死亡时的回调。

这就是调用HIDL服务的一般流程。当然，在实际开发中还有很多需要注意的细节和问题需要处理。

### 12.2 重新启动 Android 系统并验证服务是否已成功启动

构建 Android 系统镜像并烧录后，可以使用 `adb logcat` 命令查看系统日志，或使用其他工具来检查系统服务的运行状态。

```shell
$ ps -A | grep audio.effect
root    ...    android.hardware.audio.effect@1.0-service
```



## reference

[HAL 硬件抽象层](https://source.android.com/docs/core/architecture/hal)

[HIDL - HAL 接口定义语言](https://source.android.com/docs/core/architecture/hidl)

[Android系统开发入门-11.添加hidl服务](http://qiushao.net/2020/01/07/Android%E7%B3%BB%E7%BB%9F%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8/11-%E6%B7%BB%E5%8A%A0hidl%E6%9C%8D%E5%8A%A1/)
