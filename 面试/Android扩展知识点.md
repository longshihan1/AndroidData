# Apk 包体优化
## Apk 组成结构
| 文件/文件夹 | 作用/功能
|--|--
| res | 包含所有没有被编译到.arsc里面的资源文件
| lib | 引用库的文件夹
| assets | assets文件夹相比于res文件夹，还有可能放字体文件、预置数据和web页面等,通过AssetManager访问
| META_INF | 存放的是签名信息，用来保证apk包的完整性和系统的安全。在生成一个APK的时候，会对所有的打包文件做一个校验计算，并把结果放在该目录下面
| classes.dex | 包含编译后的应用程序源码转化成的dex字节码。APK里面，可能会存在多个dex文件
| resources.arsc | 一些资源和标识符被编译和写入这个文件
| Androidmanifest.xml | 编译时，应用程序的AndroidManifest.xml被转化成二进制格式

## 整体优化
- 分离应用的独立模块，以插件的形式加载
- 解压APK，重新用 7zip 进行压缩
- 用 apksigner 签名工具 替代 java提供的jarsigner签名工具

## 资源优化 
- 可以只用一套资源图片，一般采用xhdpi下的资源图片
- 通过扫描文件的MD5值，找出名字不同，内容相同的图片并删除
- 通过 Lint 工具扫描工程资源，移除无用资源
- 通过 Gradle 参数配置 shrinkResources=true
- 对 png 图片压缩
- 图片资源考虑采用 WebP 格式
- 避免使用帧动画，可使用 Lottie 动画库
- 优先考虑能否用 shape 代码、.9图、svg矢量图、VectorDrawable类来替换传统的图片

## 代码优化
- 启用混淆以移除无用代码
- 剔除 R 文件
- 用注解替代枚举

## .arsc文件优化 
- 移除未使用的备用资源来优化.arsc文件
```groovy
android {
    defaultConfig {
        ...
        resConfigs "zh", "zh_CN", "zh_HK", "en"
    }
}
```

## lib目录优化
- 只提供对主流架构的支持，比如 arm，对于 mips 和 x86 架构可以考虑不提供支持
```groovy
android {
    defaultConfig {
        ...
        ndk {
            abiFilters  "armeabi-v7a"
        }
    }
}
```

# Hook
## 基本流程
1、根据需求确定 要hook的对象  
2、寻找要hook的对象的持有者，拿到要hook的对象  
3、定义“要hook的对象”的代理类，并且创建该类的对象  
4、使用上一步创建出来的对象，替换掉要hook的对象

## 使用示例
```java
/**
* hook的核心代码
* 这个方法的唯一目的：用自己的点击事件，替换掉 View 原来的点击事件
*
* @param view hook的范围仅限于这个view
*/
@SuppressLint({"DiscouragedPrivateApi", "PrivateApi"})
public static void hook(Context context, final View view) {//
    try {
        // 反射执行View类的getListenerInfo()方法，拿到v的mListenerInfo对象，这个对象就是点击事件的持有者
        Method method = View.class.getDeclaredMethod("getListenerInfo");
        method.setAccessible(true);//由于getListenerInfo()方法并不是public的，所以要加这个代码来保证访问权限
        Object mListenerInfo = method.invoke(view);//这里拿到的就是mListenerInfo对象，也就是点击事件的持有者

        // 要从这里面拿到当前的点击事件对象
        Class<?> listenerInfoClz = Class.forName("android.view.View$ListenerInfo");// 这是内部类的表示方法
        Field field = listenerInfoClz.getDeclaredField("mOnClickListener");
        final View.OnClickListener onClickListenerInstance = (View.OnClickListener) field.get(mListenerInfo);//取得真实的mOnClickListener对象

        // 2. 创建我们自己的点击事件代理类
        //   方式1：自己创建代理类
        //   ProxyOnClickListener proxyOnClickListener = new ProxyOnClickListener(onClickListenerInstance);
        //   方式2：由于View.OnClickListener是一个接口，所以可以直接用动态代理模式
        // Proxy.newProxyInstance的3个参数依次分别是：
        // 本地的类加载器;
        // 代理类的对象所继承的接口（用Class数组表示，支持多个接口）
        // 代理类的实际逻辑，封装在new出来的InvocationHandler内
        Object proxyOnClickListener = Proxy.newProxyInstance(context.getClass().getClassLoader(), new Class[]{View.OnClickListener.class}, new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                Log.d("HookSetOnClickListener", "点击事件被hook到了");//加入自己的逻辑
                return method.invoke(onClickListenerInstance, args);//执行被代理的对象的逻辑
            }
        });
        // 3. 用我们自己的点击事件代理类，设置到"持有者"中
        field.set(mListenerInfo, proxyOnClickListener);
    } catch (Exception e) {
        e.printStackTrace();
    }
}

// 自定义代理类
static class ProxyOnClickListener implements View.OnClickListener {
    View.OnClickListener oriLis;

    public ProxyOnClickListener(View.OnClickListener oriLis) {
        this.oriLis = oriLis;
    }

    @Override
    public void onClick(View v) {
        Log.d("HookSetOnClickListener", "点击事件被hook到了");
        if (oriLis != null) {
            oriLis.onClick(v);
        }
    }
}
```

# Proguard
Proguard 具有以下三个功能：
- 压缩（Shrink）: 检测和删除没有使用的类，字段，方法和特性
- 优化（Optimize） : 分析和优化Java字节码
- 混淆（Obfuscate）: 使用简短的无意义的名称，对类，字段和方法进行重命名

## 公共模板
```
#############################################
#
# 对于一些基本指令的添加
#
#############################################
# 代码混淆压缩比，在 0~7 之间，默认为 5，一般不做修改
-optimizationpasses 5

# 混合时不使用大小写混合，混合后的类名为小写
-dontusemixedcaseclassnames

# 指定不去忽略非公共库的类
-dontskipnonpubliclibraryclasses

# 这句话能够使我们的项目混淆后产生映射文件
# 包含有类名->混淆后类名的映射关系
-verbose

# 指定不去忽略非公共库的类成员
-dontskipnonpubliclibraryclassmembers

# 不做预校验，preverify 是 proguard 的四个步骤之一，Android 不需要 preverify，去掉这一步能够加快混淆速度。
-dontpreverify

# 保留 Annotation 不混淆
-keepattributes *Annotation*,InnerClasses

# 避免混淆泛型
-keepattributes Signature

# 抛出异常时保留代码行号
-keepattributes SourceFile,LineNumberTable

# 指定混淆是采用的算法，后面的参数是一个过滤器
# 这个过滤器是谷歌推荐的算法，一般不做更改
-optimizations !code/simplification/cast,!field/*,!class/merging/*


#############################################
#
# Android开发中一些需要保留的公共部分
#
#############################################

# 保留我们使用的四大组件，自定义的 Application 等等这些类不被混淆
# 因为这些子类都有可能被外部调用
-keep public class * extends android.app.Activity
-keep public class * extends android.app.Appliction
-keep public class * extends android.app.Service
-keep public class * extends android.content.BroadcastReceiver
-keep public class * extends android.content.ContentProvider
-keep public class * extends android.app.backup.BackupAgentHelper
-keep public class * extends android.preference.Preference
-keep public class * extends android.view.View
-keep public class com.android.vending.licensing.ILicensingService


# 保留 support 下的所有类及其内部类
-keep class android.support.** { *; }

# 保留继承的
-keep public class * extends android.support.v4.**
-keep public class * extends android.support.v7.**
-keep public class * extends android.support.annotation.**

# 保留 R 下面的资源
-keep class **.R$* { *; }

# 保留本地 native 方法不被混淆
-keepclasseswithmembernames class * {
    native <methods>;
}

# 保留在 Activity 中的方法参数是view的方法，
# 这样以来我们在 layout 中写的 onClick 就不会被影响
-keepclassmembers class * extends android.app.Activity {
    public void *(android.view.View);
}

# 保留枚举类不被混淆
-keepclassmembers enum * {
    public static **[] values();
    public static ** valueOf(java.lang.String);
}

# 保留我们自定义控件（继承自 View）不被混淆
-keep public class * extends android.view.View {
    *** get*();
    void set*(***);
    public <init>(android.content.Context);
    public <init>(android.content.Context, android.util.AttributeSet);
    public <init>(android.content.Context, android.util.AttributeSet, int);
}

# 保留 Parcelable 序列化类不被混淆
-keep class * implements android.os.Parcelable {
    public static final android.os.Parcelable$Creator *;
}

# 保留 Serializable 序列化的类不被混淆
-keepnames class * implements java.io.Serializable
-keepclassmembers class * implements java.io.Serializable {
    static final long serialVersionUID;
    private static final java.io.ObjectStreamField[] serialPersistentFields;
    !static !transient <fields>;
    !private <fields>;
    !private <methods>;
    private void writeObject(java.io.ObjectOutputStream);
    private void readObject(java.io.ObjectInputStream);
    java.lang.Object writeReplace();
    java.lang.Object readResolve();
}

# 对于带有回调函数的 onXXEvent、**On*Listener 的，不能被混淆
-keepclassmembers class * {
    void *(**On*Event);
    void *(**On*Listener);
}

# webView 处理，项目中没有使用到 webView 忽略即可
-keepclassmembers class fqcn.of.javascript.interface.for.webview {
    public *;
}
-keepclassmembers class * extends android.webkit.webViewClient {
    public void *(android.webkit.WebView, java.lang.String, android.graphics.Bitmap);
    public boolean *(android.webkit.WebView, java.lang.String);
}
-keepclassmembers class * extends android.webkit.webViewClient {
    public void *(android.webkit.webView, java.lang.String);
}

# js
-keepattributes JavascriptInterface
-keep class android.webkit.JavascriptInterface { *; }
-keepclassmembers class * {
    @android.webkit.JavascriptInterface <methods>;
}

# @Keep
-keep,allowobfuscation @interface android.support.annotation.Keep
-keep @android.support.annotation.Keep class *
-keepclassmembers class * {
    @android.support.annotation.Keep *;
}
```

##  常用的自定义混淆规则
```xml
# 通配符*，匹配任意长度字符，但不含包名分隔符(.)
# 通配符**，匹配任意长度字符，并且包含包名分隔符(.)

# 不混淆某个类
-keep public class com.jasonwu.demo.Test { *; }

# 不混淆某个包所有的类
-keep class com.jasonwu.demo.test.** { *; }

# 不混淆某个类的子类
-keep public class * com.jasonwu.demo.Test { *; }

# 不混淆所有类名中包含了 ``model`` 的类及其成员
-keep public class **.*model*.** {*;}

# 不混淆某个接口的实现
-keep class * implements com.jasonwu.demo.TestInterface { *; }

# 不混淆某个类的构造方法
-keepclassmembers class com.jasonwu.demo.Test { 
  public <init>(); 
}

# 不混淆某个类的特定的方法
-keepclassmembers class com.jasonwu.demo.Test { 
  public void test(java.lang.String); 
}
```


## aar中增加独立的混淆配置
``build.gralde``
```gradle
android {
    ···
    defaultConfig {
        ···
        consumerProguardFile 'proguard-rules.pro'
    }
    ···
}
```

## 检查混淆和追踪异常
开启Proguard功能，则每次构建时 ProGuard 都会输出下列文件：

- dump.txt  
说明 APK 中所有类文件的内部结构。

- mapping.txt  
提供原始与混淆过的类、方法和字段名称之间的转换。

- seeds.txt  
列出未进行混淆的类和成员。

- usage.txt  
列出从 APK 移除的代码。

这些文件保存在/build/outputs/mapping/release/ 中。我们可以查看seeds.txt里面是否是我们需要保留的，以及usage.txt里查看是否有误删除的代码。mapping.txt文件很重要，由于我们的部分代码是经过重命名的，如果该部分出现bug，对应的异常堆栈信息里的类或成员也是经过重命名的，难以定位问题。我们可以用 retrace 脚本（在 Windows 上为 retrace.bat；在 Mac/Linux 上为 retrace.sh）。它位于/tools/proguard/ 目录中。该脚本利用 mapping.txt文件和你的异常堆栈文件生成没有经过混淆的异常堆栈文件,这样就可以看清是哪里出问题了。使用 retrace 工具的语法如下：

```shell
retrace.bat|retrace.sh [-verbose] mapping.txt [<stacktrace_file>]
```

# 架构
## MVC
![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCwLhGyLdicyLzgUDKFTZVt1OgU6iaSx2IUwnygzmQzW7Renaa8hmQ62cQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 Android 中，三者的关系如下：

![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCicNvEVMO9vDgukUR29Z1DCacZJwmmH1EEb7gUOZmDxolWexP01O8jfg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

由于在 Android 中 xml 布局的功能性太弱，所以 Activity 承担了绝大部分的工作，所以在 Android 中 mvc 更像：

![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCOq89MLQX4UM3dgBTQfU72desHb1XbOWRQZINnXOCCdZCuicUiaTHhtEg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

总结：
- 具有一定的分层，model 解耦，controller 和 view 并没有解耦
- controller 和 view 在 Android 中无法做到彻底分离，Controller 变得臃肿不堪
- 易于理解、开发速度快、可维护性高

## MVP
![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCLVgibsuVQFguBI8FBdZibLNfpvbpd6njkdGWdyR2UL6TzMOhKHFqLC0Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过引入接口 BaseView，让相应的视图组件如 Activity，Fragment去实现 BaseView，把业务逻辑放在 presenter 层中，弱化 Model 只有跟 view 相关的操作都由 View 层去完成。

总结：
- 彻底解决了 MVC 中 View 和 Controller 傻傻分不清楚的问题
- 但是随着业务逻辑的增加，一个页面可能会非常复杂，UI的改变是非常多，会有非常多的case，这样就会造成View的接口会很庞大
- 更容易单元测试

## MVVM
![](https://mmbiz.qpic.cn/mmbiz_png/zKFJDM5V3Wy5xbLTp6JMMdouZiavFxyYCMygIDD6xo5djkq6Y3jZo53sT2A4kKNaz8JEVRwmUnTmcAwJm0pZVWg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在 MVP 中 View 和 Presenter 要相互持有，方便调用对方，而在 MVP 中 View 和 ViewModel 通过 Binding进行关联，他们之前的关联处理通过 DataBinding 完成。

总结：
- 很好的解决了 MVC 和 MVP 的问题
- 视图状态较多，ViewModel 的构建和维护的成本都会比较高
- 但是由于数据和视图的双向绑定，导致出现问题时不太好定位来源

# Jetpack
## 架构
![](https://developer.android.google.cn/topic/libraries/architecture/images/final-architecture.png)

## 使用示例
``build.gradle``
```groovy
android {
    ···
    dataBinding {
        enabled = true
    }
}
dependencies {
    ···
    implementation "androidx.fragment:fragment-ktx:$rootProject.fragmentVersion"
    implementation "androidx.lifecycle:lifecycle-extensions:$rootProject.lifecycleVersion"
    implementation "androidx.lifecycle:lifecycle-livedata-ktx:$rootProject.lifecycleVersion"
    implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:$rootProject.lifecycleVersion"
}
```

``fragment_plant_detail.xml``
```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="viewModel"
            type="com.google.samples.apps.sunflower.viewmodels.PlantDetailViewModel" />
    </data>

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            ···
            android:text="@{viewModel.plant.name}"/>

    </androidx.constraintlayout.widget.ConstraintLayout>
</layout>
```


``PlantDetailFragment.kt``
```kotlin
class PlantDetailFragment : Fragment() {

    private val args: PlantDetailFragmentArgs by navArgs()
    private lateinit var shareText: String

    private val plantDetailViewModel: PlantDetailViewModel by viewModels {
        InjectorUtils.providePlantDetailViewModelFactory(requireActivity(), args.plantId)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        val binding = DataBindingUtil.inflate<FragmentPlantDetailBinding>(
                inflater, R.layout.fragment_plant_detail, container, false).apply {
            viewModel = plantDetailViewModel
            lifecycleOwner = this@PlantDetailFragment
        }

        plantDetailViewModel.plant.observe(this) { plant ->
            // 更新相关 UI
        }

        return binding.root
    }
}
```

``Plant.kt``
```kotlin
data class Plant (
    val name: String
)
```

``PlantDetailViewModel.kt``
```kotlin
class PlantDetailViewModel(
    plantRepository: PlantRepository,
    private val plantId: String
) : ViewModel() {

    val plant: LiveData<Plant>

    override fun onCleared() {
        super.onCleared()
        viewModelScope.cancel()
    }

    init {
        plant = plantRepository.getPlant(plantId)
    }
}
```

``PlantDetailViewModelFactory.kt``
```kotlin
class PlantDetailViewModelFactory(
    private val plantRepository: PlantRepository,
    private val plantId: String
) : ViewModelProvider.NewInstanceFactory() {

    @Suppress("UNCHECKED_CAST")
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        return PlantDetailViewModel(plantRepository, plantId) as T
    }
}
```

``InjectorUtils.kt``
```kotlin
object InjectorUtils {
    private fun getPlantRepository(context: Context): PlantRepository {
        ···
    }

    fun providePlantDetailViewModelFactory(
        context: Context,
        plantId: String
    ): PlantDetailViewModelFactory {
        return PlantDetailViewModelFactory(getPlantRepository(context), plantId)
    }
}
```

# 设计模式
| 模式 & 描述 | 包括
|--|--
| **创建型模式**<br>提供了一种在创建对象的同时隐藏创建逻辑的方式。| 工厂模式（Factory Pattern）<br>抽象工厂模式（Abstract Factory Pattern）<br>单例模式（Singleton Pattern）<br>建造者模式（Builder Pattern）<br>原型模式（Prototype Pattern）
| **结构型模式**<br>关注类和对象的组合。| 适配器模式（Adapter Pattern）<br>桥接模式（Bridge Pattern）<br>过滤器模式（Filter、Criteria Pattern）<br>组合模式（Composite Pattern）<br>装饰器模式（Decorator Pattern）<br>外观模式（Facade Pattern）<br>享元模式（Flyweight Pattern）<br>代理模式（Proxy Pattern）
| **行为型模式**<br>特别关注对象之间的通信。| 责任链模式（Chain of Responsibility Pattern）<br>命令模式（Command Pattern）<br>解释器模式（Interpreter Pattern）<br>迭代器模式（Iterator Pattern）<br>中介者模式（Mediator Pattern）<br>备忘录模式（Memento Pattern）<br>观察者模式（Observer Pattern）<br>状态模式（State Pattern）<br>空对象模式（Null Object Pattern）<br>策略模式（Strategy Pattern）<br>模板模式（Template Pattern）<br>访问者模式（Visitor Pattern）

## 工厂模式
```java
Bitmap bitmap = BitmapFactory.decodeResource(getResources(), R.drawable.demo);
```

``BitmapFactory.java``
```java
public static Bitmap decodeResource(Resources res, int id, Options opts) {
    validate(opts);
    Bitmap bm = null;
    InputStream is = null; 
    
    try {
        final TypedValue value = new TypedValue();
        is = res.openRawResource(id, value);

        bm = decodeResourceStream(res, value, is, null, opts);
    } 
    ···
    return bm;
}
```

## 单例模式
``InputMethodManager.java``
```java
/**
* Retrieve the global InputMethodManager instance, creating it if it
* doesn't already exist.
* @hide
*/
public static InputMethodManager getInstance() {
    synchronized (InputMethodManager.class) {
        if (sInstance == null) {
            try {
                sInstance = new InputMethodManager(Looper.getMainLooper());
            } catch (ServiceNotFoundException e) {
                throw new IllegalStateException(e);
            }
        }
        return sInstance;
    }
}
```

## 建造者模式
```java
AlertDialog.Builder builder = new AlertDialog.Builder(this)
        .setTitle("Title")
        .setMessage("Message");
AlertDialog dialog = builder.create();
dialog.show();
```

``AlertDialog.java``
```java
public class AlertDialog extends Dialog implements DialogInterface {
    ···

    public static class Builder {
        private final AlertController.AlertParams P;
        ···

        public Builder(Context context) {
            this(context, resolveDialogTheme(context, ResourceId.ID_NULL));
        }
        ···

        public Builder setTitle(CharSequence title) {
            P.mTitle = title;
            return this;
        }
        ···

        public Builder setMessage(CharSequence message) {
            P.mMessage = message;
            return this;
        }
        ···

        public AlertDialog create() {
            // Context has already been wrapped with the appropriate theme.
            final AlertDialog dialog = new AlertDialog(P.mContext, 0, false);
            P.apply(dialog.mAlert);
            ···
            return dialog;
        }
        ···
    }
}
```

## 原型模式
```java
ArrayList<T> newArrayList = (ArrayList<T>) arrayList.clone();
```

``ArrayList.java``
```java
/**
 * Returns a shallow copy of this <tt>ArrayList</tt> instance.  (The
 * elements themselves are not copied.)
 *
 * @return a clone of this <tt>ArrayList</tt> instance
 */
public Object clone() {
    try {
        ArrayList<?> v = (ArrayList<?>) super.clone();
        v.elementData = Arrays.copyOf(elementData, size);
        v.modCount = 0;
        return v;
    } catch (CloneNotSupportedException e) {
        // this shouldn't happen, since we are Cloneable
        throw new InternalError(e);
    }
}
```

## 适配器模式
```java
RecyclerView recyclerView = findViewById(R.id.recycler_view);
recyclerView.setAdapter(new MyAdapter());

private class MyAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        ···
    }

    ···
}
```

``RecyclerView.java``
```java
···
private void setAdapterInternal(@Nullable Adapter adapter, boolean compatibleWithPrevious,
        boolean removeAndRecycleViews) {
    if (mAdapter != null) {
        mAdapter.unregisterAdapterDataObserver(mObserver);
        mAdapter.onDetachedFromRecyclerView(this);
    }
    ···
    mAdapterHelper.reset();
    final Adapter oldAdapter = mAdapter;
    mAdapter = adapter;
    if (adapter != null) {
        adapter.registerAdapterDataObserver(mObserver);
        adapter.onAttachedToRecyclerView(this);
    }
    if (mLayout != null) {
        mLayout.onAdapterChanged(oldAdapter, mAdapter);
    }
    mRecycler.onAdapterChanged(oldAdapter, mAdapter, compatibleWithPrevious);
    mState.mStructureChanged = true;
}

···
public final class Recycler {
    @Nullable
    ViewHolder tryGetViewHolderForPositionByDeadline(int position,
            boolean dryRun, long deadlineNs) {
        ···
        ViewHolder holder = null;
        ···
        if (holder == null) {
            ···
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
            ···
        }
        ···
        return holder;
    }
}

···
public abstract static class Adapter<VH extends ViewHolder> {
    ···
    @NonNull
    public abstract VH onCreateViewHolder(@NonNull ViewGroup parent, int viewType);

    @NonNull
    public final VH createViewHolder(@NonNull ViewGroup parent, int viewType) {
        try {
            TraceCompat.beginSection(TRACE_CREATE_VIEW_TAG);
            final VH holder = onCreateViewHolder(parent, viewType);
            ···
            holder.mItemViewType = viewType;
            return holder;
        } finally {
            TraceCompat.endSection();
        }
    }
    ···
}
```

## 观察者模式
```java
MyAdapter adapter = new MyAdapter();
recyclerView.setAdapter(adapter);
adapter.notifyDataSetChanged();
```

``RecyclerView.java``
```java
···
private final RecyclerViewDataObserver mObserver = new RecyclerViewDataObserver();

···
private void setAdapterInternal(@Nullable Adapter adapter, boolean compatibleWithPrevious,
        boolean removeAndRecycleViews) {
    ···
    mAdapter = adapter;
    if (adapter != null) {
        adapter.registerAdapterDataObserver(mObserver);
        adapter.onAttachedToRecyclerView(this);
    }
    ···
}

···
public abstract static class Adapter<VH extends ViewHolder> {
    private final AdapterDataObservable mObservable = new AdapterDataObservable();
    ···
    public void registerAdapterDataObserver(@NonNull AdapterDataObserver observer) {
        mObservable.registerObserver(observer);
    }

    ···
    public final void notifyDataSetChanged() {
        mObservable.notifyChanged();
    }
}

static class AdapterDataObservable extends Observable<AdapterDataObserver> {
    ···
    public void notifyChanged() {
        for (int i = mObservers.size() - 1; i >= 0; i--) {
            mObservers.get(i).onChanged();
        }
    }
    ···
}

private class RecyclerViewDataObserver extends AdapterDataObserver {
    ···
    @Override
    public void onChanged() {
        assertNotInLayoutOrScroll(null);
        mState.mStructureChanged = true;

        processDataSetCompletelyChanged(true);
        if (!mAdapterHelper.hasPendingUpdates()) {
            requestLayout();
        }
    }
    ···
}
```

