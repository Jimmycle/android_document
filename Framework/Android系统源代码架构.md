# Android系统源代码架构

开发者访问[SDK 平台工具版本说明  |  Android Studio  |  Android Developers](https://developer.android.google.cn/tools/releases/platform-tools?hl=zh-cn)(androiddevtools) 上面已经有了所有你要的资源。

如果只是想查看源码，[XRefAndroid - Support AOSP 15.0 AndroidXRef & OpenHarmony 5.0](https://xrefandroid.com/) (androidxref) 一个不错的资源，需科学上网。

#### 整体结构

Android 7.0 的根目录结构说明如下表所示

```
|– Makefile （全局Makefile文件，用来定义编译规则）
|– abi （应用程序二进制接口）
|– art （ART运行环境）
|– bionic （bionic C库）
|– bootable （启动引导相关代码）
|– build （存放系统编译规则及generic等基础开发包配置）
|– cts （Android兼容性测试套件标准）
|– dalvik （dalvik JAVA虚拟机）
|– developers （开发者目录）
|– development （应用程序开发相关）
|– device （设备相关配置）
|– docs （参考文档目录）
|– external （android使用的一些开源的模组）
|– frameworks （核心框架——java及C++语言）
|– hardware （部分厂家开源的硬解适配层HAL代码）
|– kernel （内核）
|– libcore （核心库相关文件）
|– libnativehelper （动态库，实现 JNI 库的基础）
|– ndk （NDK相关代码，帮助开发人员在应用程序中嵌入C/C++代码）
|– out （编译完成后的代码输出与此目录）
|– packages （应用程序包）
|– pdk （Plug Development Kit 的缩写，本地开发套件）
|– prebuilts （x86和arm架构下预编译的一些资源）
|– sdk （sdk及模拟器）
|– system （底层文件系统库、应用及组件——C语言）
|– tools （工具文件）
|– toolchain（工具链文件）
|– vendor （厂商定制代码）
```

#### 应用层部分

应用层位于整个Android系统的最上层，开发者开发的应用程序以及系统内置的应用程序都位于应用层。源码根目录中的**packages**目录对应着系统应用层。

```
|– apps （核心应用程序）
|– experimental （第三方应用程序）
|– inputmethods （输入法目录）
|– providers （内容提供者目录）
|– screensavers （屏幕保护）
|– services （通信服务）
|– wallpapers （墙纸）
```

从目录结构可以发现，packages目录存放着系统核心应用程序、第三方的应用程序和输入法等等，这些应用都是运行在系统应用层的，因此packages目录对应着系统的应用层。

#### 应用框架层部分

应用框架层是系统的核心部分，一方面向上提供接口给应用层调用，另一方面向下与C/C++程序库以及硬件抽象层等进行衔接。 应用框架层的主要实现代码在/frameworks/base和/frameworks/av目录下，

其中/frameworks/base目录结构如下：

```
|– api （定义API） 
|– core （核心库） 
|– docs （文档） 
|– include （头文件） 
|– libs （库） 
|– media （多媒体相关库） 
|– nfc-extras （NFC相关） 
|– opengl 2D/3D （图形API） 
|– sax （XML解析器） 
|– telephony （电话通讯管理）
|– tests （测试相关） 
|– test-runner （测试工具相关） 
|– tools （工具） 
|– wifi （wifi无线网络） 
|– cmds （重要命令：am、app_proce等） 
|– data （字体和声音等数据文件） 
|– graphics （图形图像相关） 
|– keystore （和数据签名证书相关） 
|– location （地理位置相关库） 
|– native （本地库） 
|– obex （蓝牙传输） 
|– packages （设置、TTS、VPN程序） 
|– services （系统服务）
```
