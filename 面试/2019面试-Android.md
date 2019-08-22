
# Android部分

1. Activity 的生命周期

      ![](https://img-blog.csdn.net/20180911102354506?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NhcnNzY29meQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

   1. Activity正常启动：onCreate()->onStart()->onResume()           // 上图中主流程的上半部分
   2. Acivity正常退出：onPause()->onStop()->onDestory()              // 上图中主流程的下半部分

   3. Activity A 启动另一个Activity B（A还未被Destroy），A的流程:  onPause()->onStop()             // 上图中主流程的下部分

   ​        再返回时，A的流程：onRestart()->onStart()->onResume()                           // 上图中右侧枝

   4. Activity按Back 退出： onPause()->onStop()->onDestory()     // 上图中主流程的下半部分，退出时finish了or被系统清理掉了

       再进入：onCreate()->onStart()->onResume()                          // 上图中主流程的上半部分

   5. Activity按Home 退出： onPause()->onStop()                          // 上图中主流程的下半部分，退出后没有被系统清理掉

       再进入：onRestart()->onStart()->onResume()                        // 上图中右侧枝

   6. 在当前Activity前显示系统的提示框or自定义提示信息（该提示信息显示时原来的Acitity可见），再Back返回：

      onPause()->onResume()                                                            // 上图中右侧枝

   简单总结：

   > Activity有如下特点：

   A、在OnCreate创建，在OnDestroy销毁；

   B、在onStart()之后可见，onStop之后不可见；

   C、在OnResume之后获得焦点，在OnPause之后失去焦点。

   D、在不可见时，如果系统资源紧张，会自动将Activity Destroy，再次拉起Activity时，需要重建则走OnCreate，否则走OnRestart ->OnStart。

   E、因此，在开发中可以将一些必要资源释放放在onDestory中进行，而只需要在重建时更新设置的处理放在OnCreate中进行，需要每次切换页面显示时更新的处理放在OnResume，当然要注意他们之间的配合，不然会出错哦。

2. Android 的 4 大启动模式，注意 `onNewIntent()` 的调用；

   1. standard:默认启动模式，每次激活Activity时都会创建Activity，并放入任务栈中。

   2. singleTop:如果在任务的栈顶正好存在该Activity的实例， 就重用该实例，否者就会创建新的实例并放入栈顶(即使栈中已经存在该Activity实例，只要不在栈顶，都会创建实例)。

   3. singleTask:如果在栈中已经有该Activity的实例，就重用该实例(会调用实例的onNewIntent())。重用时，会让该实例回到栈顶，因此在它上面的实例将会被移除栈。如果栈中不存在该实例，将会创建新的实例放入栈中。 (因为通过startActivity的方式启动一个Activity实例时，如果栈中不存在该Activity的实例时，会经过完成的生命周期，也会调用Activity的setIntent(Intent intent)方法，但是如果已经存在时就不会经过完整的生命周期了，而是调用onNewIntent()方法，而这个方法在Activity的源码中是没有做任何操作的，既没有改变这个实例中的mIntent的值)
   4. singleInstance:在一个新栈中创建该Activity实例，并让多个应用共享改栈中的该Activity实例。一旦改模式的Activity的实例存在于某个栈中，任何应用再激活改Activity时都会重用该栈中的实例，其效果相当于多个应用程序共享一个应用，不管谁激活该Activity都会进入同一个应用中。

3. 组件化架构思路，如何从一个老项目一步一步实现组件化，主要问实现思路，考察应试者的架构能力和思考能力。 这一块内容真的很多，你需要考虑的问题很多，哪一步做什么，顺序很重要。

4. MVC、MCP、MVVP 的区别和各种使用场景，如何选择适合自己的开发架构？

   MVC  Model-View-Controller
       ListView身上体现的 MVC 思想
       Model数据模型   数据集合 Arraylist
       View  显示          Listview
       Controller 控制    adapter

   MVP  Modle-View-Presenter
       主要是为了让应用程序分层，提高测试效率。
       主要目标是让显示逻辑和业务逻辑分离
       将各视图中的业务逻辑从中分离出来，放到相应的类中，
       MVP中Presenter充当视图和业务逻辑的缓冲层

   MVVM MVVM可以算是MVP的升级版，其中的VM是ViewModel的缩写，ViewModel可以理解成是View的数据模型和Presenter的合体，ViewModel和View之间的交互通过Data Binding双向绑定完成，而Data Binding可以实现双向的交互，这就使得视图和控制层之间的耦合程度进一步降低，关注点分离更为彻底，同时减轻了Activity的压力。（双向绑定：View的变动，自动反映在 ViewModel中）

   ![6116263-31680737f5c8a594.png](https://upload-images.jianshu.io/upload_images/6116263-31680737f5c8a594.png)

   MVVM与MVP唯一的区别是View和Model进行双向绑定，MVP中的View更新需要通过Presenter，而MVVM则不需要。此时，ViewModel角色需要做的只是业务逻辑的处理，以及修改修改View或者Model的状态。MVVM有点像ListView、Adapter、数据集的关系，这个Adapter就是ViewModel角色。

5. Router 原理，如何实现组件间通信，组件化平级调用数据方式。

   Arouter在通过Route将路由PATH和class存入map中，在调用是从path中获取class,直接使用Intent调用。

   组件通信通过接口，通过path路径去调用存在的class（接口类）

6. 系统打包流程

   ![](https://images2017.cnblogs.com/blog/357738/201708/357738-20170811144825570-687368085.png)

   1. 打包资源文件，生成R.java文件

   打包资源的工具是aapt（The Android Asset Packaing Tool)

   在这个过程中，项目中的AndroidManifest.xml文件和布局文件XML都会编译，然后生成相应的R.java，另外AndroidManifest.xml会被aapt编译成二进制。

   存放在APP的res目录下的资源，该类资源在APP打包前大多会被编译，变成二进制文件，并会为每个该类文件赋予一个resource id。对于该类资源的访问，应用层代码则是通过resource id进行访问的。Android应用在编译过程中aapt工具会对资源文件进行编译，并生成一个resource.arsc文件，resource.arsc文件相当于一个文件索引表，记录了很多跟资源相关的信息。

   2. 处理aidl文件，生成相应的Java文件

   这一过程中使用到的工具是aidl（Android Interface Definition Language），即Android接口描述语言。

   aidl工具解析接口定义文件然后生成相应的Java代码接口供程序调用。如果在项目没有使用到aidl文件，则可以跳过这一步。

   3. 编译项目源代码，生成class文件

   项目中所有的Java代码，包括*R.java*和*.aidl*文件，都会变Java编译器（javac）编译成*.class*文件，生成的class文件位于工程中的bin/classes目录下。

   4. 转换所有的class文件，生成classes.dex文件

   dx工具生成可供Android系统Dalvik虚拟机执行的classes.dex文件,任何第三方的*libraries*和*.class*文件都会被转换成*.dex*文件。dx工具的主要工作是将Java字节码转成成Dalvik字节码、压缩常量池、消除冗余信息等。

   5. 打包生成APK文件

   所有没有编译的资源，如images、assets目录下资源（该类文件是一些原始文件，APP打包时并不会对其进行编译，而是直接打包到APP中，对于这一类资源文件的访问，应用层代码需要通过文件名对其进行访问）；编译过的资源和*.dex*文件都会被apkbuilder工具打包到最终的*.apk*文件中。

   打包的工具apkbuilder位于 android-sdk/tools目录下。apkbuilder为一个脚本文件，实际调用的是（sdk\tools\lib）文件中的com.android.sdklib.build.ApkbuilderMain类。

   6. 对APK文件进行签名

   一旦APK文件生成，它必须被签名才能被安装在设备上。

   在开发过程中，主要用到的就是两种签名的keystore。一种是用于调试的debug.keystore，它主要用于调试，在Eclipse或者Android Studio中直接run以后跑在手机上的就是使用的debug.keystore。

   另一种就是用于发布正式版本的keystore。

   7. 对签名后的APK文件进行对齐处理

   如果你发布的apk是正式版的话，就必须对APK进行对齐处理，用到的工具是zipalign（sdk\build-tools\25.0.0\zipalign.exe）

   对齐的主要过程是将APK包中所有的资源文件距离文件起始偏移为4字节整数倍，这样通过内存映射访问apk文件时的速度会更快。对齐的作用就是减少运行时内存的使用。

7. [APP 启动流程](https://blog.csdn.net/pgg_cold/article/details/79491791)

   1. 三个进程：
      Launcher进程：整个App启动流程的起点，负责接收用户点击屏幕事件，它其实就是一个Activity，里面实现了点击事件，长按事件，触摸等事件，可以这么理解，把Launcher想象成一个总的Activity，屏幕上各种App的Icon就是这个Activity的button，当点击Icon时，会从Launcher跳转到其他页面；

      SystemServer进程：这个进程在整个的Android进程中是非常重要的一个，地位和Zygote等同，它是属于Application Framework层的，Android中的所有服务，例如AMS, WindowsManager, PackageManagerService等等都是由这个SystemServer fork出来的。所以它的地位可见一斑。

      App进程：你要启动的App所运行的进程；

   2. 六个大类：
      ActivityManagerService：（AMS）AMS是Android中最核心的服务之一，主要负责系统中四大组件的启动、切换、调度及应用进程的管理和调度等工作，其职责与操作系统中的进程管理和调度模块相类似，因此它在Android中非常重要，它本身也是一个Binder的实现类。

      Instrumentation：监控应用程序和系统的交互；

      ActivityThread：应用的入口类，通过调用main方法，开启消息循环队列。ActivityThread所在的线程被称为主线程；

      ApplicationThread：ApplicationThread提供Binder通讯接口，AMS则通过代理调用此App进程的本地方法

      ActivityManagerProxy：AMS服务在当前进程的代理类，负责与AMS通信。

      ApplicationThreadProxy：ApplicationThread在AMS服务中的代理类，负责与ApplicationThread通信。

      可以说，启动的流程就是通过这六个大类在这三个进程之间不断通信的过程；

      我先简单的梳理一下app的启动的步骤：

      （1）启动的起点发生在Launcher活动中，启动一个app说简单点就是启动一个Activity，那么我们说过所有组件的启动，切换，调度都由AMS来负责的，所以第一步就是Launcher响应了用户的点击事件，然后通知AMS

      （2）AMS得到Launcher的通知，就需要响应这个通知，主要就是新建一个Task去准备启动Activity，并且告诉Launcher你可以休息了（Paused）；

      （3）Launcher得到AMS让自己“休息”的消息，那么就直接挂起，并告诉AMS我已经Paused了；

      （4）AMS知道了Launcher已经挂起之后，就可以放心的为新的Activity准备启动工作了，首先，APP肯定需要一个新的进程去进行运行，所以需要创建一个新进程，这个过程是需要Zygote参与的，AMS通过Socket去和Zygote协商，如果需要创建进程，那么就会fork自身，创建一个线程，新的进程会导入ActivityThread类，这就是每一个应用程序都有一个ActivityThread与之对应的原因；

      （5）进程创建好了，通过调用上述的ActivityThread的main方法，这是应用程序的入口，在这里开启消息循环队列，这也是主线程默认绑定Looper的原因；

      （6）这时候，App还没有启动完，要永远记住，四大组建的启动都需要AMS去启动，将上述的应用进程信息注册到AMS中，AMS再在堆栈顶部取得要启动的Activity，通过一系列链式调用去完成App启动；

   ![](https://img-blog.csdn.net/20180309004634352?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcGdnX2NvbGQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

8. 如何做启动优化？ 冷启动什么的肯定是基础，后续应该还有的是懒加载，丢线程池同步处理，需要注意这里可能会有的坑是，丢线程池如何知道全部完成。

   [影响速度原因](https://blog.csdn.net/yuanguozhengjust/article/details/80052066)：

    高耗时任务：数据库初始化、某些第三方框架初始化、大文件读取、MultiDex加载等，导致CPU阻塞

   复杂的View层级：使用的嵌套Layout过多，层级加深，导致View在渲染过程中，递归加深，占用CPU资源，影响Measure、Layout等方法的速度

   类过于复杂：Java对象的创建也是需要一定时间的，如果一个类中结构特别复杂，new一个对象将消耗较高的资源，特别是一些单例的初始化，需要特别注意其中的结构

   主题及Activity配置：有一些App是带有Splash页的，有的则直接进入主界面，由于主题切换，可能会导致白屏，或者点了Icon，过一会儿才出现主界面

   [优化](https://www.jianshu.com/p/9772bfa87839)：

   MultiDex,优化方法数,少用一些不必要的框架,慎用Kotlin,Dex懒加载,插件化或H5/React Native方案

   1.利用提前展示出来的Window，快速展示出来一个界面，给用户快速反馈的体验
   2.避免在启动时做密集沉重的初始化（Heavy app initialization）
   3.定位问题：避免I/O操作、反序列化、网络操作、布局嵌套等

9. 事件分发机制。 事件分发已经不是直接让你讲了，会给你具体的场景，比如 A 嵌套 B ，B 嵌套 C，从 C 中心按下，一下滑出到 A，事件分发的过程，这里面肯定会有 ACTION_CANCEL 的相关调用时机。

​        

1. 如何检测卡顿，卡顿原理是什么，怎么判断是页面响应卡顿还是逻辑处理造成的卡顿？

2. 生产者模式和消费者模式的区别？

3. 单例模式双重加锁，为什么要这样做。

4. Handler 机制原理，IdleHandler 什么时候调用。
    提供一个android没有的声明周期回调时机（比如activitythread的onresume回调给act的onresume,可以优先绘制），[比如](https://blog.csdn.net/tencent_bugly/article/details/78395717)
    可以结合HandlerThread, 用于单线程消息通知器

5. LeakCanary 原理，为什么检测内存泄漏需要两次？
    一次是leak自己检测，通过ondestory查看可用性，进行清除弱引用，发现之后通过生成dump文件检测位置

6. BlockCanary 原理。
   android是消息驱动了，换句话说，所有的UI操作都是sendmessage到messagequeue里去执行的。oncreate onresume 等，这里面的方法都在looper里面执行的。
   因此如果应用发生卡顿，一定是在 dispatchMessage 中执行了耗时操作
   LooperMonitor第二次调用时，会判断第二次与第一次的时间间隔是否会超过阀值

7. ViewGroup 绘制顺序
    从log可以看到,当Viewgroup嵌套一个View的时候执行的顺序如下 
   =>>>>>View的onMeasure方法 =>>>>> ViewGroup的onMeasure方法 
   =>>>>>ViewGroup的onSizeChanged方法 =>>>>>View的onSizeChanged方法 
   =>>>>>View的onlayout方法确定自己在ViewGroup中的位置=>>>>>ViewGroup的onlayout方法确定自己的位置 
   =>>>>>View的ondraw方法绘制View

8. Android 有哪些存储数据的方式。

9. SharedPrefrence 源码和问题点；

    SP:优点：用法简单，

   ​     缺点：不支持跨进程，数据在启动时填充内存

10. 讲讲 Android 的四大组件；

11. 属性动画、补间动画、帧动画的区别和使用场景；
    ObjectAnimator
    TranslateAnimation,ScaleAnimation,RotateAnimation,AlphaAnimation


12. 自定义 ViewGroup 如何实现 FlowLayout？如何实现 FlowLayout 调换顺序？

13. 自定义 View 如何实现打桌球效果；

14. 自定义 View 如何实现拉弓效果，贝瑟尔曲线原理实现？

15. APK 瘦身是怎么做的，只用 armabi-v7a 没有什么问题么？ APK 瘦身这个基本是 100% 被面试问到，可能是我简历上提到的原因。
   可以

16. ListView 和 RecyclerView 区别？RecyclerView 有几层缓存，如何让两个 RecyclerView 共用一个缓存？

17. 如何判断一个 APP 在前台还是后台？

18. 如何做应用保活？全家桶原理？

19. 讲讲你所做过的性能优化。

20. Retrofit 在 OkHttp 上做了哪些封装？动态代理和静态代理的区别，是怎么实现的。

21. 讲讲轨迹视频的音视频合成原理；

22. AIDL 相关
   AIDL和Messenger的区别：
   Messenger不适用大量并发的请求：Messenger以串行的方式来处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个的去处理。
   Messenger主要是为了传递消息：对于需要跨进程调用服务端的方法，这种情景不适用Messenger。
   Messenger的底层实现是AIDL，系统为我们做了封装从而方便上层的调用。
   AIDL适用于大量并发的请求，以及涉及到服务端端方法调用的情况


23. Binder 机制，讲讲 Linux 上的 IPC 通信，Binder 有什么优势，Android 上有哪些多进程通信机制?
    采用C/S的通信模式。而在linux通信机制中，目前只有socket支持C/S的通信模式，但socket有其劣势，具体参看第二条。
   有更好的传输性能。对比于Linux的通信机制，
      socket：是一个通用接口，导致其传输效率低，开销大；
      管道和消息队列：因为采用存储转发方式，所以至少需要拷贝2次数据，效率低；
      共享内存：虽然在传输时没有拷贝数据，但其控制机制复杂（比如跨进程通信时，需获取对方进程的pid，得多种机制协同操作）。
   安全性更高。Linux的IPC机制在本身的实现中，并没有安全措施，得依赖上层协议来进行安全控制。而Binder机制的UID/PID是由Binder机制本身在内核空间添加身份标识，安全性高；并且Binder可以建立私有通道，这是linux的通信机制所无法实现的（Linux访问的接入点是开放的）。
   另外一个优点是对用户来说，通过binder屏蔽了client的调用server的隔阂，client端函数的名字、参数和返回值和server的方法一模一样，对用户来说犹如就在本地（也可以做得不一样），这样的体验或许其他ipc方式也可以实现，但binder出生那天就是为此而生。

24. RxJava 的线程切换原理。

25. OkHttp 和 Volloy 区别；

26. Glide 缓存原理，如何设计一个大图加载框架。
   将这个id连同着signature、width、height等等10个参数一起传入到EngineKeyFactory的buildKey()方法当中，从而构建出了一个EngineKey对象，这个EngineKey也就是Glide中的缓存Key了

27. LRUCache 原理；
   LinkedHashMap