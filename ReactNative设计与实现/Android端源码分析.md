<font face="Times New Roman">
## 四、Android端源码分析

在本章中，我首先会介绍RN在Android端的初始化流程，紧接着介绍JSBundle的加载流程以及NativeModules的初始化过程，最后会介绍三端通信。

### 1、Android端的初始化流程

通过前面的介绍，我们已经了解到，在Android上，RN工程其本身就是一个Android工程，与JS相关的一切都发生在```ReactRootView```内部。```ReactRootView```继承自```FrameLayout```，也就是说```ReactRootView```是一个View，我们可以像操作Android UI框架里的其他View一样来操作```ReactRootView```。通过[Create React Native App](https://github.com/react-community/create-react-native-app)创建的RN工程，会将```ReactRootView```设置为工程主Activity的ContentView，详见第二章的第二小节：React-Native与原生开发的对比。

在这里，我们的主Activty比较像一个容器，一个承载RN JS部分的容器，在一个工程中，我们可以有多个独立的RN容器，这些RN容器可以拥有独立的初始化流程，加载不同的JSBundle，处理不同的逻辑；此外，Fragment也可以作为RN容器，这样我们就可以在同一个页面加载多个RN容器了。这有点像WebView，我们可以初始化不同的WebView实例，加载不同的HTML页面，用以处理不同的业务；同时，我们也可以在同一个页面加载多个WebView。或许我们可以简单地认为RN容器是一种性能更优的WebView，毕竟，它们是如此的相似。

RN的初始化流程需要遵从Android的初始化流程，当我们路由到RN的宿主Activity时(这里我们以Activity为例，以Fragment为容器的初始化流程与以Activity为容器的初始化流程基本一致)，RN开始初始化，如下图所示：

![15.png](https://upload-images.jianshu.io/upload_images/1042695-68a96de38f304640.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图给出了RN初始化过程中的关键方法调用，当```ReactRootView```中的```startReactApplication```方法被调后，流程会分为左右两部分。如果ReactContext不存在，左侧分支的流程会被触发，它会创建一个新的线程，然后在这个新的线程中创建ReactContext；无论ReactContext不存在与否，左侧分支的流程都会被触发，它们只是一些简单的Wrapper方法。创建ReactContext是一个比较耗时的过程，在这个过程中，整个程序都会被阻塞，这就是为什么在启动一个RN程序时，尤其是复杂的RN程序，会有一小段的白屏时间。因此，为了缩短启动时白屏时间，就需要优化创建ReactContext的流程，随后我们会详细介绍这一流程。

当ReactContext创建成功后，会将```ReactRootView```添加到```UIManager```上，```UIManager```主要负责管理和控制程序的外观与整体效果。当```ReactRootView```被添加到```UIManager```上后，会返回一个int型的rootTag，用以标识当前的```ReactRootView```实例，当程序结束时，我们需要通过这个rootTag来确定需要被销毁的```ReactRootView```。随后，```ReactRootView```的```invokeJSEntryPoint```的方法会被调用，在这个方法的内部，会通过跨语言的调用，来启动JS的入口文件。

在JS侧，我们通过AppRegistry来注册入口文件。以上文中提到的NavigatorTest工程为例，在工程根目录下的index.js文件中，有如下代码：

```javascript
import { AppRegistry } from 'react-native';
import App from './App';

AppRegistry.registerComponent('NavigatorTest', () => App);
```

当```invokeJSEntryPoint```被调用后，App中的JS代码会被执行，至此，JS开始处理业务逻辑，渲染组件。注意，我们在注册入口文件时，需要给我们的组件取一个名字，上述例子中的入口的名字是'NavigatorTest'，这个名字需要与主Activity中```getMainComponentName ```方法的返回值一致，如下所示：

```java
public class MainActivity extends ReactActivity {
    /**
     * Returns the name of the main component registered from JavaScript.
     * This is used to schedule rendering of the component.
     */
    @Override
    protected String getMainComponentName() {
        return "NavigatorTest";
    }
}

```

接下来，我们将讨论在创建ReactContext的过程中都发生了些什么。

### 2、createReactContext方法都干了些什么？

在创建ReactContext的过程中，主要干了三件事：

1. 初始化NativeModuleRegistery
2. 初始化CatalystInstance
3. 加载JSBundle

CatalystInstance应该是整个RN中最核心的的一个类了，它有两个实现，一个在Java侧，一个在C++侧，这两个实现通过JNI来关联。下面让我们来看一下在Java侧构建CatalystInstance实例的代码：

```java
CatalystInstanceImpl.Builder catalystInstanceBuilder = 
	new CatalystInstanceImpl.Builder()
	.setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
	.setJSExecutor(jsExecutor)
	.setRegistry(nativeModuleRegistry)
	.setJSBundleLoader(jsBundleLoader)
	.setNativeModuleCallExceptionHandler(exceptionHandler);
catalystInstance = catalystInstanceBuilder.build()
```
CatalystInstance需要的第一个参数是```ReactQueueConfigurationSpec```，上文中提到的mqt\_js和mqt\_native\_modules两个线程会在调用```ReactQueueConfigurationSpec.createDefault()```时被创建。同时，我们可以通过CatalystInstance提供的```getReactQueueConfiguration```方法来获得这两个线程的引用。

CatalystInstance需要的第二个参数是```jsExecutor```，用来指定执行JS代码JS引擎。上文中我们有提到，当我们debug一个RN程序，JS代码是运行在PC的Chrome中的，此时使用的是Chrome的V8引擎；而在非debug模式下，JS代码运行在内嵌的JavaScriptCore上。

第三个需要的参数是```nativeModuleRegistry```。在第二章的第四小节：《如何开发一个NativeModule》中，我们通过```ReactNativeHost```的```getPackages```方法来注册我们需要的NativeModule。在调用```CatalystInstanceImpl.Builder```之前，会通过```processPackages ```来处理所有的NativeModule，并生成```NativeModuleRegistry```，如下所示：

```java
NativeModuleRegistry nativeModuleRegistry = 
		processPackages(reactContext, mPackages, false);
```

我们可以将```NativeModuleRegistry```当作一个两层的方法表，第一层是Module，第二层是Module下的常量以及被```@ReactMethod```修饰的方法，如下图所示：

![16.png](https://upload-images.jianshu.io/upload_images/1042695-ae35d1fb22005f9f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Java与JavaScript侧都会维持这样的一个方法表，JavaScript调用Java的方法就是通过这两个方法表之间映射关系来确定具体的Java方法的。

第四个参数是```jsBundleLoader ```。[Metro](https://facebook.github.io/metro/en/)是RN默认的JS打包器，从其文档的[Bundling](https://facebook.github.io/metro/docs/en/bundling)这一节可以知道，JSBundle可以被打成多种不同的格式，每种格式都需要不同的加载器(loader)来加载，默认情况下，JSBundle会被打成纯文本格式。同时，JSBundle也可以被存储在不同的位置，默认情况下，JSBundle被放在assets目录下，需通过assetLoader加载；而通过CodePush等热更新技术下载的JSBundle，会被存储在应用对应的私有目录下，需通过fileLoader加载。

最后一个是```exceptionHandler```，所有JavaScript调Java方法时的异常都会被它捕获到，我们可以通过注册```exceptionHandler```来处理这些异常。

在CatalystInstance构造函数的内部，会调用```initializeBridge```这一Java Native方法来触发C++侧的CatalystInstance的初始化。CatalystInstance中定义了几组的Java Native方法，它们分别用于初始化、加载JSBundle和Java调用JS，详细的方法声明如下所示：

```java
// 初始化相关
private native static HybridData initHybrid ();
private  native void jniExtendNativeModules (
		Collection<JavaModuleWrapper> javaModules,
		Collection<ModuleHolder> cxxModules
);
private native void initializeBridge (
		ReactCallback callback,
		JavaScriptExecutor jsExecutor,
		MessageQueueThread jsQueue,
		MessageQueueThread moduleQueue,
		Collection<JavaModuleWrapper> javaModules,
		Collection<ModuleHolder> cxxModules
);

// 加载JSBundle相关
private native void jniSetSourceURL (String sourceURL);
private native void jniRegisterSegment (int segmentId, String path);
private native void jniLoadScriptFromAssets (
		AssetManager assetManager,
		String assetURL,
		boolean loadSynchronously
);
private native void jniLoadScriptFromFile (
		String fileName,
		String sourceURL,
		boolean loadSynchronously
);

// Java调用JS代码相关
private native void jniCallJSFunction (
		String module,
		String method,
		NativeArray arguments
);
private native void jniCallJSCallback (
		int callbackID,
		NativeArray arguments
);

// 其他
private native void jniHandleMemoryPressure (int level);
public native void setGlobalVariable (String propName, String jsonValue);
private native void getJavaScriptContext ();
```

### 3、NativeModule的初始化过程

上一节中我们提到，在创建ReactContext的过程中，会通过```processPackages```方法来处理所有的NativeModule，并生成```NativeModuleRegistry```，并作为CatalystInstance初始化的参数之一传递给CatalystInstance。在CatalystInstance的构造函数中，会调用```initializeBridge```这一java native方法，并将```NativeModuleRegistry```传递到C++侧。相对于JavaScript，NativeModule分为两部分：JavaModule和CxxModule，分别为JavaScript提供调用Java与C++方法的能力。初始化流程如下图所示：
![17.png](https://upload-images.jianshu.io/upload_images/1042695-5ad16d6f52ce63c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在Java侧调用```initializeBridge```这一java native方法后，CatalystInstanceImpl.cpp中的```initializeBridge```会被触发，随后是Instance.cpp中的```initializeBridge```，在这里，会通过```jsQueue->runOnQueueSync```将剩下的初始化流程放到mqt\_js线程中处理。jsQueue由java侧通过```initializeBridge```方法传递到C++侧，通过它可以操作mqt\_js线程的消息队列，向消息队列中插入新的可执行体。

此时，在mqt\_js上，会初始化新的JavaScriptCore实例即```new JSCExecutor()```，在JSCExecutor的构造函数内部，会调用```installGlobalProxy```方法，通过该方法，JSC在JavaScript的global对象上定义了一个名为```nativeModuleProxy```的属性，当在JavaScript侧访问这个属性时，会触发JSCExecutor中的```getNativeModule```这一回调方法，该回调方法的返回值即```nativeModuleProxy```属性对应的值，下面会详细介绍这个回调方法。至此NativeModule在native侧初始化已经完成。

上文中有提到，我们可以将```NativeModuleRegistry```认为是一个方法表，Java与JavaScript侧都会维持这样的一个方法表，下面将介绍NativeModule在JavaScript侧的初始化流程，如下图所示：
![18.png](https://upload-images.jianshu.io/upload_images/1042695-6d9626d20e7fae63.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当CatalystInstance实例创建成功后，会调用它的```runJSBundle```方法，这个方法会去加载JSBundle并开始执行JavaScript，JSBundle的加载流程会在下一节中介绍，这里不再赘述。当JavaScript开始执行，并通过```require('NativeModules')```加载NativeModule.js文件时，NativeModule.js中的全局指令会被执行。首先，在JavaScript的global对象上定义了一个名为```__fbGenNativeModule```的方法属性，并指向```genModule```方法；随后，会访问global对象上的```nativeModuleProxy```，当访问```nativeModuleProxy```属性时，JSCExecutor中的```getNativeModule```回调方法会被触发，该方法最终会调用JSCNativeModules.cpp 中的```createModule```方法，其实现如下：

```c++
folly::Optional<Object> JSCNativeModules::createModule(const std::string& name, JSContextRef context) {
  ReactMarker::logTaggedMarker(ReactMarker::NATIVE_MODULE_SETUP_START, name.c_str());

  if (!m_genNativeModuleJS) {
    auto global = Object::getGlobalObject(context);
    m_genNativeModuleJS = global.getProperty("__fbGenNativeModule").asObject();
    m_genNativeModuleJS->makeProtected();
  }

  auto result = m_moduleRegistry->getConfig(name);
  if (!result.hasValue()) {
    return nullptr;
  }

  Value moduleInfo = m_genNativeModuleJS->callAsFunction({
    Value::fromDynamic(context, result->config),
    Value::makeNumber(context, result->index)
  });
  CHECK(!moduleInfo.isNull()) << "Module returned from genNativeModule is null";

  folly::Optional<Object> module(moduleInfo.asObject().getProperty("module").asObject());

  ReactMarker::logTaggedMarker(ReactMarker::NATIVE_MODULE_SETUP_STOP, name.c_str());

  return module;
}
```
在上述实现中，首先会访问JavaScript global对象上的```__fbGenNativeModule```，并赋值给```m_genNativeModuleJS```；随后通过```m_moduleRegistry->getConfig(name)```来获得某一个NativeModule的定义，这里的```m_moduleRegistry```既是通过```initializeBridge```方法传递到C++侧```NativeModuleRegistry```；最后，通过```m_genNativeModuleJS->callAsFunction```来调用JavaScript侧```genModule```方法来完成具体的初始化操作。当所有NativeModule初始化完成后，会作为```getNativeModule```方法的返回值传递到JavaScript侧，并赋给NativeModules对象，如下所示：

```javascript
// NativeModules.js
function genModule(...) {
  ...
}
global.__fbGenNativeModule = genModule;
...
NativeModules = global.nativeModuleProxy
...
module.exports = NativeModules;
```

### 4、JSBundle的加载过程

当CatalystInstance实例创建成功后，会调用它的```runJSBundle```来加载JSBundle，其流程如下图所示：
![19.png](https://upload-images.jianshu.io/upload_images/1042695-1da697cfa2ed10e8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过`runJSBundle`方法，最终会调到`jniLoadScriptFromAssets`或`jniLoadScriptFromFile`方法，这是两个JNI方法，调用它们会触发其在C++侧的实现，即CatalystinstanceImpl.cpp中对应的`jniLoadScriptFromAssets`和`jniLoadScriptFromFile`方法，在这两个方法内部，会去读取JSbundle文件，并将其转换为内存中的字符串。读取JSbundle文件是一个耗时的过程，JSbundle文件越大，耗时越长。因为加载JSbundle是创建ReactContext的过程中的一个步骤，它或许是最耗时的一个步骤，优化JSbundle的加载时间，可以显著减少APP启动时的白屏时间。一个常见的优化方案是将JSBundle拆分成多个bundle文件，在APP启动过程中只加载基础功能部分的bundle文件，其他bundle文件按照按需加载的方式以此加载。这也是web上最常用的优化方法之一。

上文中提到JSBundle有多种格式，默认情况下，JSBundle是纯文本格式的，因此最终会调到Instance.cpp中的```loadScriptFromString```方法，随后是NativeToJsBridge.cpp中的```loadApplication```方法，在该方法内部会通过```runOnExecutorQueue```将执行权限传递到mqt\_js线程。在mqt\_js线程上，JSC会加载JavaScript代码并开始执行。

至此，创建ReactContext的过程已结束，随后，`ReactRootView`被添加到`UIManager`上，并调用`invokeJSEntryPoint`方法并启动业务的JS入口文件，渲染开始。我们可以通过注册`ReactInstanceEventListener`来获得ReactContext创建成功的事件，如下所示：

```java
mInstanceManager.addReactInstanceEventListener(new ReactInstanceManager.ReactInstanceEventListener() {
  @Override
  public void onReactContextInitialized(ReactContext reactContext) {
    // do something
    mInstanceManager.removeReactInstanceEventListener(this);
  }
});
```

### 5、三端通信

RN Android会涉及到三种语言Java、JavaScript和C++，这三种语言之间的互相调用是RN需要支持的基础功能，如下图所示：
![20.png](https://upload-images.jianshu.io/upload_images/1042695-ac5191fdfa1964db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Java与C++可通过JNI实现双方的相互调用；JavaScript与C++可通过JavaScriptCore提供的API实现相互调用；Java与JavaScript之间没有直接相互调用的机制，因此它们之间的相互需要通过C++的桥接来完成，这即是上文中提到的C++ Bridge。下面将详细介绍它们之间的相互调用过程。

#### (1) Java调用C/C++方法
java可以通过定义native方法的方式来调用C/C++方法，如上文中列举出的CatalystInstance中的native方法，它们的声明在java侧，实现却在C/C++侧。有两种定义java native方法的方式：

1. 通过函数名的映射
2. 通过JNI接口提供的`RegisterNatives`方法

##### 通过函数名的映射
下属代码定义了三个java native方法，其中`getNDKString`是native方法与普通java方法间的重载，`convertToInt`是native方法之间的重载。native方法之间的重载对C++侧方法的申明有影响：

```java
package com.example.caotao.ndkexample;
public class TestNDK {
    static { System.loadLibrary("MyLibrary"); }

    public String getNDKString (int i) { return ""; }
    public native String getNDKString();

    public native int convertToInt (double d);
    public native int convertToInt (double d, String type);
}
```
下面是上述java native方法对于的C++方法的申明：

```c++
JNIEXPORT jstring JNICALL Java_com_example_caotao_ndkexample_TestNDK_getNDKString(JNIEnv *, jobject);
JNIEXPORT jint JNICALL Java_com_example_caotao_ndkexample_TestNDK_convertToInt__D(JNIEnv *, jobject, jdouble);
JNIEXPORT jint JNICALL Java_com_example_caotao_ndkexample_TestNDK_convertToInt__DLjava_lang_String_2(JNIEnv *, jobject, jdouble, jstring);
```

通过函数名映射时，所有与java native方法对应的C/C++函数都需通过`JNIEXPORT`对外暴露，其次是对应的C/C++函数必须按照指定的方式进行命名，函数名较长，书写麻烦。更多关于函数名映射的细节，请参考[jni文档](https://docs.oracle.com/javase/7/docs/technotes/guides/jni/spec/design.html)。

##### 通过RegisterNatives方法

相对于通过函数名映射方式的缺点，通过`RegisterNatives`方法定义native方法是，我们可以随心所欲的对C/C++函数进行命名，同时这些C/C++方法也不需要对外暴露。

`RegisterNatives`需与`JNI_OnLoad`方法配合使用，当在Java侧通过调用`System.loadLibrary("LibraryName")`动态加载加载C/C++库时，如果对应的C/C++库定义了`JNI_OnLoad`方法，`JNI_OnLoad`方法会被自动触发，我们可以在`JNI_OnLoad`内部调用`RegisterNatives`来向Java侧注册native方法。



React-Native通过这种方式来向Java注册native方法的，其在ReactAndroid/src/main/jni/react/jni/OnLoad.cpp中定义了`JNI_OnLoad`方法，如下所示：

```c
extern "C" JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved) {
  return initialize(vm, [] {
    ...
    CatalystInstanceImpl::registerNatives();
    ...
  });
}
```

下面是CatalystInstanceImpl.cpp中的`registerNatives`方法，与Java侧CatalystInstance中的java native方法一致。

```c++
void CatalystInstanceImpl::registerNatives() {
  registerHybrid({
    makeNativeMethod("initHybrid", CatalystInstanceImpl::initHybrid),
    makeNativeMethod("initializeBridge", CatalystInstanceImpl::initializeBridge),
    makeNativeMethod("jniExtendNativeModules", CatalystInstanceImpl::extendNativeModules),
    makeNativeMethod("jniSetSourceURL", CatalystInstanceImpl::jniSetSourceURL),
    makeNativeMethod("jniRegisterSegment", CatalystInstanceImpl::jniRegisterSegment),
    makeNativeMethod("jniLoadScriptFromAssets", CatalystInstanceImpl::jniLoadScriptFromAssets),
    makeNativeMethod("jniLoadScriptFromFile", CatalystInstanceImpl::jniLoadScriptFromFile),
    makeNativeMethod("jniCallJSFunction", CatalystInstanceImpl::jniCallJSFunction),
    makeNativeMethod("jniCallJSCallback", CatalystInstanceImpl::jniCallJSCallback),
    makeNativeMethod("setGlobalVariable", CatalystInstanceImpl::setGlobalVariable),
    makeNativeMethod("getJavaScriptContext", CatalystInstanceImpl::getJavaScriptContext),
    makeNativeMethod("jniHandleMemoryPressure", CatalystInstanceImpl::handleMemoryPressure),
  });

  JNativeRunnable::registerNatives();
}
```

关于`RegisterNatives`方法，StackOverflow有一个很好的回答：[What does the registerNatives() method do?](https://stackoverflow.com/questions/1010645/what-does-the-registernatives-method-do)

需要注意的是，React-Native中对jni做了一层封装，对应的实现在ReactAndroid/src/main/jni/first-party/fb/jni/fbjni.cpp，头文件在ReactAndroid/src/main/jni/first-party/fb/include/fb/fbjni中。

#### (2) C/C++调用Java方法

C/C++需要通过JNI提供的API来调用Java方法，分为调用实例(instance)方法和调用静态(static)方法两大类。C/C++调用Java方法的基本原则是：(1) 根据类、方法名、方法签名照到对应的方法id； (2) 根据方法的返回值类型，调用对应的Call方法，并传入参数。如下所示：

调用实例(instance)方法：

```c
jmethodID GetMethodID (JNIEnv *env, jclass clazz, const char *name, const char *sig);

Call<type>Method(JNIEnv *env, jobject obj, jmethodID methodID, ...);
// e.g. CallVoidMethod()	CallObjectMethod()

```

调用静态(static)方法：

```c
jmethodID GetStaticMethodID (JNIEnv *env, jclass clazz, const char *name, const char *sig);

CallStatic<type>Method(JNIEnv *env, jclass clazz, jmethodID methodID, ...);
// e.g. CallStaticVoidMethod()	CallStaticObjectMethod()
```

#### (3) JavaScript调用C++方法(JavaScriptCore)

在介绍NativeModule的初始化过程时，当访问JavaScript global对象上的`nativeModuleProxy`属性时，就会触发JSCExecutor.cpp中的`getNativeModule`方法，这就是一个JavaScript调用C++方法的例子。

在C++侧，我们可以利用JavaScriptCore提供的API在JavaScript的global对象上定义属性或方法。通过C++定义在global对象上的属性，在C++侧会有一个与之对应的回调函数，用于计算该属性的值。在JavaScript侧，通过访问或调用global对象上的这些属性和方法，就可以达到调用C++函数的目的。

在JSCHelpers.cpp中定义了两个方法，它们的分别用来在JavaScript的global对象定义属性和方法

```c++
void installGlobalProxy( JSGlobalContextRef ctx, const char* name, JSObjectGetPropertyCallback callback);

void installGlobalFunction(JSGlobalContextRef ctx, const char *name, JSObjectCallAsFunctionWithCallback callback);
```

RN中通过JavaScriptCore API定义的全局方法有：`nativeFlushQueueImmediate`、`nativeCallSyncHook`、`nativeLoggingHook`和`nativePerformanceNow`，全局属性有：`nativeModuleProxy`和`nativeExtensions`。


#### (4) C++调用JavaScript方法(JavaScriptCore)

C++通过访问JavaScript global对象上的方法和属性来调用JavaScript方法，其中比较重要的两个是`__fbGenNativeModules`和`__fbBatchedBridge`，它们的定义如下:

```javascript
// NativeModule.js
global.__fbGenNativeModules = genModule(config: ?ModuleConfig, moduleID: number)

// BatchedBridge.js
global.__fbBatchedBridge = {
  callFunctionReturnFlushedQueue : Function,
  invokeCallbackAndReturnFlushedQueue : Function,
  flushedQueue : Function,
  callFunctionReturnResultAndFlushedQueue : Function
}
```

`__fbGenNativeModules`在之前的章节中已经介绍过了，在接下来介绍Java与JavaScript互调的章节中，将会用到`__fbBatchedBridge`中定义的四个方法，这里就不再赘述。

#### (5) JavaScript调用Java方法

JavaScript调用Java主要通过NativeModule来完成，例如`NativeModule.ToastModule.show`就会调到ToastModule.java中的show方法。

JavaScript对Java调用，不是调用后就直接传递到native侧并触发对应的Java方法。在通过NativeModule.js中的`genModule`初始化JS侧的NativeModule时，将所有对native方法的调用都做了一层封装，如下所示：

```javascript
function(...args: Array<any>) {
  ...
  BatchedBridge.enqueueNativeCall(moduleID, methodID, args, onFail, onSuccess)
}
// BatchedBridge继承自MessageQueue
```
当在JS侧调用某个NativeModule的方法，就会调用到MessageQueue.js中的`enqueueNativeCall`方法，并将对native侧的调用缓存在一个调用队列中。每5ms，就会通过global对象上的`nativeFlushQueueImmediate`方法，批量传送到native侧，我们在之前章节中提到的RN中UI的异步更新就体现在这里。`nativeFlushQueueImmediate`是在RN初始化过程中通过JSC API定义在JavaScript global对象上的一个C++方法。JavaScript调用Java方法的流程如下图所示：

![21.png](https://upload-images.jianshu.io/upload_images/1042695-e020917a57f5074a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
当JavaModuleWrapper.cpp中的`invoke`方法时，会将控制权由mqt\_js线程传递到mqt\_native\_modules线程，并触发JavaModuleWrapper.java中的`invoke`方法，在Java侧，会通过反射的方式找到对应的方法并调用。

#### (6) Java调用JavaScript方法

与JavaScript通过NativeModule来调用Java方法类似，Java通过JavaScriptModule来调用JavaScript方法。不同的是，RN提供了扩展NativeModule的功能，却没有提供扩展JavaScriptModule的功能。相对JavaScript调用Java，Java调用JavaScript的场景要少得多，也比较固定，大多集中在RN的初始化阶段，如上文中提到的`invokeJSEntryPoint`就是其中之一。

JavaScriptModule在Java侧全部以接口(interface)的形式呈现，并在接口中定义好Java需要调用的方法。Java侧不负责这些方法的实现，JavaScript侧负责。`AppRegistry`是最常见的一个JavaScriptModule，在JS侧，我们使用它来注册业务的入口文件，即：

```javascript
// index.js or app.js
AppRegistry.registerComponent('NavigatorTest', () => App);
```

让我们来看一下`AppRegistry`在Java侧的定义：

```java
public interface AppRegistry extends JavaScriptModule {
  void runApplication(String appKey, WritableMap appParameters);
  void unmountApplicationComponentAtRootTag(int rootNodeTag);
  void startHeadlessTask(int taskId, String taskKey, WritableMap data);
}
```

在`invokeJSEntryPoint`方法内部，会调用`AppRegistry`的`runApplication`方法来启动业务的入口文件，其中`appKey`就是我们在`registerComponent`是指定的key，在我们的例子中，即"NavigatorTest"。

因为Java侧没有`runApplication`方法的实现，当Java侧调用`runApplication`时，会通过Java反射机制中的`Proxy`与`InvocationHandler`，将对该方法的调用代理到CatalystInstance的`callFunction`方法上。相关实现细节在[这里](https://github.com/facebook/react-native/blob/v0.55.4/ReactAndroid/src/main/java/com/facebook/react/bridge/JavaScriptModuleRegistry.java)可以找到，代码量很少，也很容易理解。关于Java反射，这里也不做介绍，感兴趣的同学可以自行查阅相关资料。

`callFunction`最终会调用`jniCallJSFunction`这一Java native方法，将对应的JavaScriptModule名、方法名以及参数传递到C++侧，整个调用流程如下图所示：
![22.png](https://upload-images.jianshu.io/upload_images/1042695-64e136a756dc2642.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当方法调用到NativeToJsBridge.cpp文件中的`callFunction`方法时，通过`runOnExecutorQueue`将Java对JavaScript方法的调用传递到了mqt\_js线程。在JSCExecutor.cpp的`callFunction`方法内部，会通过JSC API拿到定义在JavaScript global对象上的`__fbBatchedBridge`属性，这一global属性我们在前面的章节中介绍过，它是RN初始化过程中由JS侧定义的，详见
[BatchedBridge.js](https://github.com/facebook/react-native/blob/v0.55.4/Libraries/BatchedBridge/BatchedBridge.js)。

`__fbBatchedBridge`本身是一个对象，它的内部定义了四个函数，C++侧利用这四个函数，将Java对JavaScript的调用传递到JS侧，核心代码如下所示：

```c++
return m_callFunctionReturnFlushedQueueJS->callAsFunction({
  Value(m_context, String::createExpectingAscii(m_context, moduleId)),
  Value(m_context, String::createExpectingAscii(m_context, methodId)),
  Value::fromDynamic(m_context, std::move(arguments))
});
```

上述代码使用的是`callFunctionReturnFlushedQueue`，通过它，C++将需要被调用的JavaScript模块名、方法名以及参数传递到了JS侧。在JS侧，会根据这些信息找到对应的JS方法并执行。另一个方法，`invokeCallbackAndReturnFlushedQueue`，专用于处理Java侧对JavaScript侧回调，与`jniCallJSCallback`相对应。对于`callFunctionReturnResultAndFlushedQueue`方法，JSCExecutor.cpp只有对它的定义(RN v0.55)，却没有对它的调用，故不做讨论。

很容易注意到，这些方法的名字中都包含"FlushedQueue"，那么FlushedQueue指什么呢？在*JavaScript调用Java方法* 这一小节中，我们提到，JavaScript对Java方法的调用会被缓存在一个队列中，每隔5ms向Java侧传递一次，这个队列就是FlushedQueue。每次Java调用JavaScript方法时，FlushedQueue都会以返回值的形式传递到native侧，并得到相应的处理。这是native侧主动从JavaScript侧获取JavaScript对native侧的调用的场景之一，这一行为在Android与iOS上是一致的。此外，native侧还可以通过`__fbBatchedBridge`的`flushedQueue`方法，直接获得FlushedQueue，而不需要调用JavaScript侧的其他方法。

通过前面的讨论我们可以知道，Java对JavaScript的调用，最终会通过`jniCallJSFunction`方法来完成(不考虑回调的场景，即`jniCallJSCallback`)。那么我们可不可以绕开JavaScriptModule，直接利用这个方法来调用JavaScript中的任意函数呢？答案是，我们可以绕开JavaScriptModule直接使用`jniCallJSFunction`方法，但是不可以通过它随意调用JavaScript侧定义的函数。让我们先来看一看`callFunctionReturnFlushedQueue`函数在JavaScript侧的实现：

```javascript
// MessageQueue.js
callFunctionReturnFlushedQueue(module: string, method: string, args: any[]) {
  this.__guard(() => {
    this.__callFunction(module, method, args);
  });

  return this.flushedQueue();
}

__callFunction(module: string, method: string, args: any[]): any {
  ...
  const moduleMethods = this.getCallableModule(module);
  const result = moduleMethods[method].apply(moduleMethods, args);
  return result;
}
```

由上述代码可知，JavaScript侧会通过native侧传递过来的模块名、方法名等参数，从CallableModule中找到找到对应的JavaScript函数并执行。也就是说，通过`jniCallJSFunction`，我们只能调用CallableModule中的JavaScript函数。在JavaScript侧，我们可以通过BatchedBridge提供的`registerCallableModule`或`registerLazyCallableModule`来注册CallableModule。常见的CallableModule有：AppRegistry，ReactFabric和HMRClient。
</font>