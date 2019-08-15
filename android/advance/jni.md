# NDK 开发

## 一、需要的技术栈：

 - JNI 文档
 
	Java Native Interface doc
	
	https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/functions.html
	
	JNI Tips：
	
	https://developer.android.com/training/articles/perf-jni
	
	JNI Tips 胡凯翻译版本：
	
	http://hukai.me/android-training-course-in-chinese/performance/perf-jni/index.html
	
- C/C++

  C/C++ Primer Pro 	
  
- ndk 相关脚本

  javah, javap,ndk-build,ndk-stash...
  
- CMake
  
  CMake 使用 http://www.hahack.com/codes/cmake/
    

## JNI开发流程主要分为以下6步

- 1、编写声明了native方法的Java类
- 2、将Java源代码编译成class字节码文件
- 3、用javah命令生成.h头文件
- 4、用本地代码(C/C++)实现.h头文件中的函数
- 5、将本地代码编译成动态库（windows：*.dll，linux/unix：*.so，mac ：*.jnilib）
- 6、拷贝动态库至本地库搜索目录下，并运行Java程序


## 关键概念及方法说明
JavaVM:一个进程只能有一个
JNIEnv ：线程间不能共享，如果线程使用必须要先关联：AttachCurrentThread

JNI 特殊方法
JNI_OnLoad
JNI_OnUnload

## so加载流程
阅读 System.loadLibrary源码实现了解到：

- 1. System.loadLibrary会优先查找apk中的so目录，再查找系统目录
apk  ： /data/app/${package-name}/lib/arm/
system：/vendor/lib     /system/lib；64位：/vendor/lib64   /system/lib64
- 2. 不能使用不同的ClassLoader加载同一个动态库
- 3. System.loadLibrary加载过程中会调用目标库的JNI_OnLoad方法，我们可以在动态库中加一个JNI_OnLoad方法用于动态注册
- 4. 如果加了JNI_OnLoad方法，其的返回值为JNI_VERSION_1_2 ，JNI_VERSION_1_4， JNI_VERSION_1_6其一。我们一般使用JNI_VERSION_1_4即可
- 5. Android动态库的加载与Linux一致使用dlopen系列函数，通过动态库的句柄和函数名称来调用动态库的函数

## 断点调试：
LLDB：支持断点调试c/c++源码。


### 阅读开源源码的收获，以及项目中实际问题的解决，开发的一些好用的组件
- 1.变量全局缓存池(jclass,jmethodID,jFieldID),解决频繁查询为空的问题。
- 2.JNI log,用于打印日志的组件，使用简单，打印日志更详细。一键关闭日志。
- 3.auto_lock 线程锁，使用了c++对象构造与析构的原理，来实现自动加锁解锁的线程锁。使用简单，解决死锁问题
- 4.convert_tools 转换工具，用于Java中对象与C/C++对象互相转换。
- 5.LRUCache ，泛型编程，使用简单。
- 6.scope_jenv ，安全的获取env的包装器，可以在多线程中使用。使用简单


## JNI异常处理：

JNI调用java代码出现异常后，不要调用异常处理意外的代码，否则必定崩溃。本地代码异常是一个挂起的异常。需要处理后才能接着调用。
通过调用ExceptionCheck或者ExceptionOccurred捕获到异常，然后使用ExceptionClear清除掉异常。

万恶的UnsatisfiedLinkError
- 库文件不存在或者不能被app访问到。使用adb shell ls -l 
- 依赖的函数或者库没有找到，soload
- lib库不同目录下的SO文件参差不齐
- lib库目录下的SO不符合相应的CPU架构
- java代码混淆导致

No implementation found for native xxx
1.库文件没有得到加载。检查日志输出中关于库文件加载的信息
2.由于名称或者签名错误，方法不能匹配成功。使用javah 自动生成头文件

Android常见错误UnsatisfiedLinkError的解决方案
https://www.zhupite.com/android/unsatisfiedlinkerror.html

ROM/Framework问题,PackageManager 安装过程出了问题
ReLinker
https://medium.com/keepsafe-engineering/the-perils-of-loading-native-libraries-on-android-befa49dce2db
https://github.com/KeepSafe/ReLinker



## 增强so的安全性方案探究：

一、提高静态分析的难度（目的：静态增强后，单纯从静态分析的层面已经不可行）
	1.使用RegisterNatives动态注册native方法，隐藏入口函数
	2.混淆Java层native入口方法。
	3.so内部使用的所有字符串都是用base64编码，使用的时候去解码。提供全局的字符串缓存，解码只需要解一次，之后都使用缓存数据

二、提高动态调试的难度
	1.自己附加进程，防止调试。使用ptrace(PTRACE_TRACEME, 0, 0, 0)
	2.签名校验，本地服务端双管齐下，直接从apk，不要经过PackageManager获取签名
	3.借助系统api判断应用调试状态和调试属性
	4.轮询检查android_server调试端口信息和进程信息，防护IDA
	5.轮询检查自身status中的TracerPid字段值，防止被其他进程附加调试
	
	
	
## 	脚本及使用：

生成JNIHelper Native 方法对应的C代码的头文件

如用到Android中的对象需要指定classpath

javah -classpath $ANDROID_PLATFORMS/android-22/android.jar:. com.wuba.bangjob.secret.JNIHelper

查看Test.class 中所有方法的签名等信息
javap -s -p Test.class ：


构建so
ndk-build NDK_PROJECT_PATH=`pwd` NDK_APPLICATION_MK=`pwd`/Application.mk

ndk-stack 查看so奔溃的堆栈信息

通过logcat直接输出

so_path :是包含so的文件夹,例如 
<java> </java>









