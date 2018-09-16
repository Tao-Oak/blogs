<font face="Times New Roman">
# <center>ReactNative设计与实现及Android端源码分析</center>

跨平台是软件开发中的一个重要概念，本质上，跨平台开发就是为了**增加代码复用，减少开发者对各平台差异适配的工作量，降低开发成本**。在提高业务专注度的同时，提供比web更好的体验。通俗了讲就是：*省钱、偷懒*。ReactNative是Facebook推出的一套针对移动端APP开发的跨平台解决方案。

本文首先从**代码复用**的视角，向大家介绍一些跨平台开发的背景知识，并将RN与其他跨平台解决方案做一些简单的对比；其次是从实践的角度，将RN与Native(Android, iOS)开发做一些简单对比；随后向大家介绍RN的整体架构以及它的线程模型；最后是对Android端部分源码的分析。

## 一、背景

### 1、Android与iOS做代码共享
现假设我们要开发一个处理音视频的APP，需同时支持Android和iOS两端，它能对音视屏文件进行编解码、格式转换等操作。此时，我们有: (1) C/C++编写的核心功能库FFmpeg，它负责具体的编解码、格式转换等操作；(2) Android与iOS的平台SDK以及它们各自的UI库。为了能够使用FFmpeg提供的音视屏处理能力，我们首先需要做一层**跨语言函数调用**的封装。

iOS端的同学比较幸福，因为OC作为C语言的扩展，可以直接调用C/C++函数。Android端稍为麻烦一点，Java语言并非C或C++语言的扩展，它不能直接调用C/C++定义的函数。为了能够调用C/C++函数，Java需要借助JNI。JNI即Java Native Interface，它是Java虚拟机(JVM)提供的一套能够使运行在JVM上的Java代码调用native程序、以及被native程序调用的编程框架。在Android平台上开发native功能，还需要借助NDK(Native Development Kit)，它主要提供对native部分代码的编译的支持。注意，此处的native是相对于Java语言来说的，即C、C++或汇编程序；需要与ReactNative中的native区分开来。至此，我们完成了跨语言函数调用的封装，如下图所示：

![1.png](https://upload-images.jianshu.io/upload_images/1042695-de10aa647bc06219.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接下来就是业务流程与交互界面的开发了，这部分由Android与iOS端的同学分别完成。至此，我们开发出了我们音的视频处理APP，整个项目的结构大致如下图所示：

![2.png](https://upload-images.jianshu.io/upload_images/1042695-5ce9dc48c599c8db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上面的方案它很有效，也能很好地满足我们的需求，但是它还不够完美。从代码复用的角度来看，在上述方案中，除核心库FFmpeg外，Android与iOS几乎没有复用任何代码。

我们应该知道，核心库FFmpeg它提供了基础的编解码、格式转换等功能，但它却不支持我们特定的业务流程。为此，我们需要在FFmpeg提供的基础功能上，进一步开发我们特定的业务流程。现假设有这样一个需求：有十个wav格式的视频文件，需要将第1、3、5、7、9个视频转换为mp4格式，第2、4、6、8、10个视频转换为rmvb格式。按照上述方案，Android端同学用Java实现了这一流程，iOS端同学需要用OC再实现一遍。

通过对流程本身的分析我们可以发现：**流程它是与平台、与语言无关的**。那么，针对同一流程，我们为什么要用不同的语言来实现多次呢？关于这样做的缺点，大家应该都很清楚，网上也有很多相关的讨论，在此就不再赘述了。

针对上述项目结构的缺点，我们可以做如下调整：(1) 将特定业务流程的实现逻辑下沉到C/C++层，(2) Android与iOS层通过跨语言的封装来调用这些业务流程。此时整个项目的结构如下图所示：

![3.png](https://upload-images.jianshu.io/upload_images/1042695-c158a29ad839545d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

第二种方案相对于第一种，通过将业务流程下沉到C/C++层，实现了较大程度的代码复用。这样已经很好了，但能不能更好呢？此时，Android与iOS的同学依然需要分别去实现各自端的UI，我们能不能复用UI部分的实现呢？

让我们先来简单地分析一下UI系统的特定。首先，UI系统需要有组件库，每个组件都向用户传达特定的信息，并接收特定的用户操作；不同的组件传递不同的信息，接受不同的操作。例如按钮组件，它向用户传达点击的信息，并接受用户的点击操作。所谓HTML标签的语义化，表达的就是这个意思。简单扩展一下HTML标签语义化的概念，我们可以提出**组件的语义化**这一概念。组件的语义是与平台无关的，它是程序开发者与程序用户之间所达成的一种共识。

Android与iOS有各自的组件库，它们采用不同的语言实现，有各自特定的内部机制；但是，因为需要向用户传递同样的信息，它们绝大部分的组件都是可以一一对应的，例如Android的```Button```和iOS的```UIButton```。

其次是布局，布局即是指定各个组件在屏幕上的位置的过程，它确定了组件与屏幕边框、组件与组件之间的相对位置。例如在屏幕的左上角放置一个按钮，按钮距离屏幕顶部边框10个像素单位，距离屏幕左侧边框15个像素单位。显然，这也是一个与平台、与语言无关的过程。

既然Android与iOS的组件可以认为是一一对应的，且布局是一个与平台、与语言无关的过程，那么我们可不可以将布局的过程抽离出来，作为一个控制中心，我们通过这个控制中心发出布局相关的指令，然后由一个中间转换系统，根据当前的宿主环境，将这些指令转换为平台相关的组件与布局语法呢？这样，我们就可以跨平台复用UI与逻辑了。如下图所示：

![4.png](https://upload-images.jianshu.io/upload_images/1042695-a64a5ba47f1ee59e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在上图中，__*UI逻辑*__与__*统一的组件与布局*__两部分组成了控制中心，我们通过它指定需要在屏幕上显示的组件以及组件在屏幕上显示的位置，同时还负责处理用户与组件之间的交互。控制中心可以使用任一语言来实现。__*中间转换系统*__一般会涉及到跨语言调用和对平台相关组件以及组件的属性和方法的封装。最底层的__*UI系统*__负责具体的渲染操作。

React-Native就是这么做的。它的控制中心由JavaScript语言实现，然后通过NativeModule这一中间转换系统，将布局指令翻译成平台相关的组件与布局语法。简而言之，RN就是通过JavaScript来控制Native的渲染，请注意，RN的JS层代码是控制Native的渲染而不是JS本身负责渲染。

### 2、如何在Android与iOS上支持JavaScript

我们都知道，Android可以使用Java和Kotlin，iOS可以使用Objective-C和Swift，来开发应用程序；但它们都支持直接使用JavaScript来编写应用程序。为了在应用程序中支持JavaScript语言，目前主要有两种实现方式：(1) 利用系统的浏览器组件，即Android上的```WebView```和iOS上的```UIWebView```，(2) 编译并集成一个全功能的JavaScript引擎。

目前，利用第一种方式来支持JavaScript的应用主要有Adobe PhoneGap和Apache Cordova，它们是另一种思路的跨平台实现方案；利用第二种方式来支持JavaScript的应用是React-Native。下面我们将这两种跨平台方案做一个简单的对比。

#### (1) React-Native  vs. Cordova, PhoneGap

||React-Native|Cordova, PhoneGap|
|--------------|----------------|-------|
|运行环境|JavaScriptCore，无兼容性问题 |运行在系统浏览器上，存在浏览器兼容问题|
|渲染|Native渲染，接近原生渲染效果|借助浏览器渲染，渲染速度较慢|
|Native扩展的实现|通过NativeModule扩展；支持Map、Array等复杂参数|Android端只支持原始数据类型的参数；iOS无相关公有API，需使用UIWebView的私有API|
|通信|利用JSC作为中间适配层桥接，JS端与原生端的双向通信交互|原生侧需通过callback从JS API获取返回值|

上表从**JavaScript的运行环境**、**UI的渲染**、**Native扩展的实现**以及**JS与Native侧的通信**这四个方面对React-Native与Cordova, PhoneGap进行了对比。

Cordova, PhoneGap利用了系统的浏览器组件，其好处是很容易实现，但是它不灵活且效率低下。Android上的```WebView```提供了一个叫```addJavascriptInterface```的方法，它可以将Java类插入到JavaScript的上下文中，但是它只支持传递原始数据类型(primitive data type)，这限制了API的设计。而且它不稳定，由[issue #12987](https://issuetracker.google.com/issues/36923426])可知，它在Android 2.3的模拟器和一些真实设备上可能会导致崩溃。在iOS上情况更糟糕，```UIWebView```没有公有API来支持从JavaScript到Objective-C的直接交互，你必须使用私有API来实现与 ```addJavascriptInterface```相同的功能。利用系统浏览器组件的另一个缺点是开发者们不得不使用回调的方式来获取JavaScript API的返回值，这对于游戏来说是复杂且低效的。

React-Native中的JavaScript代码是直接运行在JavaScript引擎上的，故不存在系统组件的兼容性问题。对于UI的渲染，JavaScript代码负责指定需要渲染的组件以及各组件之间的相对位置，而实际的渲染工作由Native侧完成，能达到接近原生的渲染效果。RN的Native扩展通过它的NativeModule来实现，可以支持Map、Array等复杂类型参数的传递。RN中与Native相关的组件、包括官方提供的，都是通过NativeModule来实现的。最后，RN中利用C++实现了一个桥，借助这个桥，实现了JS端与原生端的双向通信交互，它在Android上的表现是Java与JS代码的相互调用，在iOS上则是Objective-C与JS代码的相互调用。关于这个C++ Bridge和NativeModule，稍后会有详细的介绍。

#### (2) JavaScript引擎的选择

目前主流的JavaScript引擎有JavaScriptCore，SpiderMonkey，V8和Rhino，它们的主要应用以及在Android与iOS上的兼容性如下表所示：

![7.png](https://upload-images.jianshu.io/upload_images/1042695-874a2743b6140d2a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从兼容性的角度来看，Rhino由于不兼容iOS，而V8也只兼容越狱后的iOS设备，故先将它两排除。JavaScriptCore和SpiderMonkey在各方面基本相同，React-Native使用JavaScriptCore作为它的内置JavaScript引擎。

### 3、三端代码共享

三端是指Android，iOS与Web这三端。在Web端，我们利用JS来调用DOM API，以此来控制浏览器对界面的渲染，在这里，JS本身也不负责渲染，具体的渲染工作由浏览器完成。这与上文中描述的RN的渲染行为是一致的。对于这三端的代码复用，其整体结构大致如下图所示：
![5.png](https://upload-images.jianshu.io/upload_images/1042695-4edb34e7e8bb33bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 4、小结

通过上面的讨论，我们了解到了RN出现的背景以及它所解决的问题。同时也了解到了RN的基本工作原理：即通过JavaScript代码来控制Native侧的渲染。接下来我们将通过一些具体的例子，来看看RN与Native的异同点。

## 二、实践

### 1、一个React-Native工程的目录结构

我们按照React-Native官网的[Getting Started](https://facebook.github.io/react-native/docs/getting-started.html)的指引，借助[Create React Native App](https://github.com/react-community/create-react-native-app)创建了一个名为NavigatorTest的RN工程。然后借助tree命令，分析了这个RN工程的目录结构，如下图所示(略去了部分不相关文件)：

![8.png](https://upload-images.jianshu.io/upload_images/1042695-a472385830621fc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

由上图可知，一个RN工程中包含一个完整的Android和一个完整的iOS工程，它们分别位于android和ios目录下。当我们使用```react-native run-android```命令时，它最终会启动位于android目录下的Android工程，并根据manifest加载主Activity。这里，主Activity的定义如下：

```java
public class MainActivity extends ReactActivity
```
其中```ReactActivity```由RN提供，它继承自Android的Activity。

### 2、React-Native与原生开发的对比

首先让我们来看两段Android代码：

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  setContentView(R.layout.activity_main);
  ...
}
```
上述代码是Android Activity中onCreate方法的常见写法，我们通过```setContentView```方法指定Activity的UI。

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
  super.onCreate(savedInstanceState);
  
  mReactRootView = new ReactRootView(this);
  mReactInstanceManager = initReactInstanceManager();
  mReactRootView.startReactapplication(mReactInstanceManager, "demo", null);
  
  setContentView(mReactRootView)
}
```
这段代码是RN的ReactActivity的onCreate方法的写法。可以看到，它同样是通过```setContentView```方法来指定Activity的UI。不同的是，第一段代码指定的是一个layout文件，第二段代码指定的是```ReactRootView```的一个实例。

```ReactRootView```继承```SizeMonitoringFrameLayout```，而```SizeMonitoringFrameLayout```继承Android的```FrameLayout```。所有与JS的交互都发生在```ReactRootView```内。

### 3、在原生APP中集成React-Native

关于如何在一个已有的Native APP中继承RN，RN官方文档的[Integration with Existing Apps](https://facebook.github.io/react-native/docs/integration-with-existing-apps)这一小节有详细的介绍，简而言之，可分为以下五步：

1. 设置好React-Native依赖项和目录结构，将已有工程根目录下的文件全部拷贝到android文件夹下
2. 用JavaScript开发React-Native组件
3. 新建一个activity，并将ReactRootView设为它的ContentView
4. 在工程根目录下通过npm start启动React-Native服务，随后运行Android工程
5. 验证APP的React-Native部分是否按预期工作

感兴趣的同学可以按照官方教程去实验一下，这里我就不再赘述了。这里需要注意一点：我们需要在app/build.gradle的defaultConfig中添加 ```ndk { abiFilters “armeabi-v7a”, “x86” }```，否则可能会报下面的错误:

```
java.lang.UnsatisfiedLinkError:
	dlopen failed: "libgnustl_shared.so" is 32-bit instead of 64-bit
```

### 4、如何开发一个NativeModule

前面我们已经提到，RN借助NativeModule来做Native扩展，且官方提供的与Native相关的功能与组件，都是通过这一机制来实现的。其实，我们借助NativeModule来扩展的Native功能和组件与RN官方提供的无本质上的差别。

为了实现一个Native功能或组件，我们需要在Android与iOS上分别开发，官方文档的[native-modules-android](https://facebook.github.io/react-native/docs/native-modules-android)和[native-modules-ios](https://facebook.github.io/react-native/docs/native-modules-ios)这两小节分别有详细的介绍。在第一章《背景》部分，我们提到了**中间转换系统**的概念，添加新的NativeModule其实就是扩展我们的中间转换系统，使得它能够转换更多的JS指令。

一个NativeModule具有以下三个要素：

1. Module的名字，由getName方法返回；JS侧直接通过这个名字来调用该Module提供的功能
2. Module暴露给JS侧的常量：由getConstants返回
3. Module暴露给JS侧的方法：由@ReactNative注解修饰的方法

下面我们以Android端的Toast为例，来展示如何开发一个NativeModule。首先，我们需要定义一个NativeModule，代码如下：

```java
public class ToastModule extends ReactContextBaseJavaModule {
  private static final String DURATION_SHORT_KEY = "SHORT";
  private static final String DURATION_LONG_KEY = "LONG";

  public ToastModule(ReactApplicationContext reactContext) {
    super(reactContext);
  }
  
  @Override
  public String getName() { return "ToastExample"; }
  
  @Override
  public Map<String, Object> getConstants() {
    final Map<String, Object> constants = new HashMap<>();
    constants.put(DURATION_SHORT_KEY, Toast.LENGTH_SHORT);
    constants.put(DURATION_LONG_KEY, Toast.LENGTH_LONG);
    return constants;
  }
  
  @ReactMethod
  public void show(String message, int duration) {
    Toast.makeText(getReactApplicationContext(), message, duration).show();
  }
}
```
上述代码中，我们通过```getName()```方法定义了一个名为```ToastExample```的NativeModule；并通过```getConstants()```方法对JS侧暴露了两个常量，用来控制Toast的现实时间；最后我们通过```@ReactMethod```注解向JS侧暴露了一个```show```方法，JS侧将通过调用这个```show```方法来展示Toast。

第二，定义一个ReactPackage，并向其注册ToastModule：

```java
public class CustomToastPackage implements ReactPackage {
  @Override
  public List<ViewManager> createViewManagers(ReactApplicationContext reactContext) {
    return Collections.emptyList();
  }
  
  @Override
  public List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    List<NativeModule> modules = new ArrayList<>();
    modules.add(new ToastModule(reactContext));
    return modules;
  }
}
```
上述代码定义了一个名为CustomToastPackage的ReactPackage，通过ReactPackage，我们可以将多个功能相近的NativeModule聚合在一起。在```createNativeModules```方法中，我们定义了一个NativeModule的List，并将上文中定义的ToastModule添加到这个List中，最后返回这个List。

第三，注册ReactPackage，这需要通过ReactNativeHost的getPackages方法来完成。以上文中提到的NavigatorTest工程为例，我们需要在android/app/src/main/java/com/navigatortest/MainApplication.java内的getPackages()方法中注册CustomToastPackage。

```java
public class MainApplication extends Application implements ReactApplication {
  
  private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
    ...
    
    protected List<ReactPackage> getPackages() {
      return Arrays.<ReactPackage>asList(
        new MainReactPackage(),
        new CustomToastPackage()); // <-- Add this line with your package name.
      }
    };
  }
  
  @Override
  public ReactNativeHost getReactNativeHost() {
    return mReactNativeHost;
  }
  ...
}

```

最后，在JS侧使用ToastModule：

```javascript
// ToastExample.js
import { NativeModules } from 'react-native';
module.exports = NativeModules.ToastExample;

// Other.js
import ToastExample from './ToastExample';
ToastExample.show('Awesome', ToastExample.SHORT);
```
上述代码中，我们通过```NativeModules.ToastExample```来获得上文中定义的ToastModule，请注意，ToastExample与ToastModule中的```getName()```方法的返回值一致。在JS侧的常见做法是先将NativeModule做一层封装，如上述代码中的ToastExample.js，然后在其他JS文件中引用这个封装，如上述代码中的Other.js。

### 5、小结

这一章我们将RN与Native做了一些简单的对比，并详细介绍了如何开发一个NativeModule。下面我们将着重讨论RN的整体架构。

## 三、整体架构

### 1、React-Native的整体架构

一个RN工程可以分为三大部分，JavaScript域、Native域以及负责两个域之间通信的C++ Bridge。如下图所示：
![9.png](https://upload-images.jianshu.io/upload_images/1042695-1ffecce3c5369251.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中，JavaScript域为单线程，使用的编程语言是JavaScript，其中JavaScript代码运行在JavaScriptCore上。**JavaScript域主要负责实现APP的业务逻辑、指定需要渲染的组件以及组件的布局。**

Native域是一个多线程的环境，它有负责UI渲染的主UI线程，以及其他后台任务线程。值得注意的是，负责运行JavaScript代码的线程是Native域的多条后台任务线程中的一条。熟悉JavaScript事件循环的同学应该都很清楚，JavaScript本身没有线程，线程由宿主环境提供。**Native域的主要作用是提供宿主环境，并负责UI渲染与交互。**Native域的编程语言因平台而异，Android主要是Java和Kotlin，iOS主要是Objective-C和Swift。

C++ Bridge主要负责JavaScript域与Native域的通信，而通信则是指JS与Java、Objective-C等语言之间的相互调用。上文提到的NativeModule，就是依赖C++ Bridge来实现的。

### 2、React-Native性能

性能是我们考量一个框架或一套技术方案的重要指标之一。下面我们将分别分别讨论RN三大部分的性能问题。
![10.png](https://upload-images.jianshu.io/upload_images/1042695-c72d5e20b4f13344.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

相对于Java、Objective-C等编成语言，一直以来JavaScript给我们的都是执行慢的印象，其实随着JavaScript语言的发展与JavaScript引擎的不断优化，现代JavaScript的执行速度已经非常快了，性能问题一般不会出在JavaScript域。Android与iOS对其各自的性能都有保证，性能问题一般也不会出现在Native域。RN的性能瓶颈往往会出现在C++ Bridge上。

我们知道，在JavaScript域和Native域中运行的是不同的语言，因此在不同域中定义的变量是不能相互访问的，所有跨语言的通信都需要通过C++ Bridge来完成。事实上，这与客户端与服务器间通过web通信的方式类似 -- 数据必须序列化后才能通过。数据的序列化与反序列化是非常耗时的。同时，这里有一件非常酷的事 -- 当你在Chrome中调试RN的JS代码时，JavaScript域和Native域实际是运行在不同的计算机设备中的(你的PC和你的移动设备)，它们之间通过WebSocket来桥接。

因此，**为了构建一个高性能的React-Native APP，我们必须将桥上传递的数据量保持在最低限度**。

### 3、UI的异步更新

有一个很明显的性能陷阱：跨JS与Native域进行UI的同步更新。如下图所示：
![12.png](https://upload-images.jianshu.io/upload_images/1042695-ac3691ec199687b6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们在JS域通过```Button.setColor```来改变一个button的颜色，然后由Native侧来完成实际的改变button颜色的操作。当Native在执行时，JS线程会被阻塞，等待Native侧执行完成后，JS侧继续执行。这是一个性能陷阱。我们真的需要这份同步吗？

让我们来看看RN是怎么解决这个问题的，事实上，RN本身并没去解决这个问题。React.js在Web上解决了一个类似的问题，RN直接复用了它的解决方案。Web与RN有着类似的结构，我们有JavaScript脚本，它运行在JavaScript线程上，同时我们有DOM，DOM是浏览器Native结构的一部分，每次通过DOM API来更新DOM时，都是一次跨JS与Native域的UI的同步更新。React为了解决这个问题，提出了虚拟DOM的概念，结合智能的diff算法，**将我们在JS侧对组件的更改批量、异步地发送到Native侧** -- 从而最小化了需要通过Bridge传递的数据量。

因此，RN中的UI更新是异步完成的。如下图所示：
![13.png](https://upload-images.jianshu.io/upload_images/1042695-dbc5358f02dc2c60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

JS侧的更新指令被批量、异步地发送到Native侧，在Native执行实际的更新操作时，JS线程不会被阻塞。这里需要注意一点，当我们在JS侧调用```Button.setColor```来更改button的颜色，紧接着调用```Button.getColor```来获取button的最新颜色，我们可能获取不到button的最新的颜色。此时Native侧的更新操作可能还没有完成，对于这种场景，我们需要特别注意。

### 4、React-Native中的线程(Android)

RN的Android端主要有三个线程，负责UI的的main\_ui线程，负责执行JavaScript代码的mqt\_js线程，以及与NativeModule相关的mqt\_native\_modules线程，每个线程都有与其绑定的消息队列。

```ReactQueueConfigurationImpl.java```中定义并创建了mqt\_js和mqt\_native\_modules线程，main\_ui线程由Android本身创建。我们可以通过```ReactContext```来获取这些线程的引用，代码如下：


```java
CatalystInstance catalystInstance = reactContext.getCatalystInstance();
if (catalystInstance == null) {
  return;
}
ReactQueueConfiguration queueCfg =
		catalystInstance.getReactQueueConfiguration();
if (queueCfg == null) {
  return;
}
MessageQueueThreadImpl jsQueueThread = 
		(MessageQueueThreadImpl) queueCfg.getJSQueueThread();
MessageQueueThreadImpl nativeModulesQueueThread = 
		(MessageQueueThreadImpl) queueCfg.getNativeModulesQueueThread();
```

三个线程之间的交互主要借助Android的Handler来完成。如下图所示：

![11.png](https://upload-images.jianshu.io/upload_images/1042695-514b7f064474d26f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

mqt\_js线程中运行的是JavaScript代码，它位于JavaScript域；而main\_ui与mqt\_native\_modules线程中运行的是Native代码，它们位于Native域。Native侧(main\_ui和mqt\_native\_modules线程)通过调用```jniCallJSFunction```和```jniCallJSCallback```方法来执行JS侧的代码，从而实现与mqt\_js线程交互；mqt\_js线程通过调用NativeModule的暴露给JS侧的方法与mqt\_native\_modules线程交互；而main\_ui与mqt\_native\_modules可通过Handler直接交互。

关于JavaScript域与Native域代码互调的细节，我们将在第四章《源码分析》中详细介绍。

### 5、Android APK的结构

下图是基于React-Native 0.53.3打出的Android APK结构：
![14.png](https://upload-images.jianshu.io/upload_images/1042695-a0c791434da8b88c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中，所有与RN相关的动态链接文件，都给出了其在源码中的位置，一共有八个；同时，有三个文件名被加粗。第一个是assets下面的index.android.bundle，我们编写的JavaScript代码以及这些代码所需要的依赖都会被打包到这个文件中。在APP的启动过程中，首先会加载这个bundle文件，并交由JavaScriptCore来执行。与业务相关的逻辑都在这个bundle中，如果我们能通过某种机制将这个bundle文件替换为新的bundle文件，下次APP启动时，就会加载这个新的bundle文件，这样就达到更新业务逻辑的目的。这就是RN的热更新，CodePush等热更新方案就是通过这一机制来实现的。注意：热更新智能更新JS部分的改动，而不能更新Native部分的改动，所有Native部分的改动都需要通过正常的发版机制来完成。

第二个文件是libjsc.so，上文中我们有提到，RN中编译并继承了一个全功能的JavaScript引擎：JavaScriptCore。libjsc.so就是JavaScriptCore的动态链接文件，它比较大，有4M左右，且每一个由RN工程打包出来的apk都包含一个独立的libjsc.so。

第三个文件是libreactnativejni.so，它是所有与RN相关的so文件的入口文件。

### 6、小结

本章介绍了RN的整体架构，随后介绍的RN的性能以及UI异步更新的特性，同时还介绍了RN Android端的线程模型以及它们之间的交互，最后简单分析了RN Android APK的结构。下面我们将开始Android端的源码分析。

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

### 4、JSBundle的加载过程

### 5、三端通信

</font>