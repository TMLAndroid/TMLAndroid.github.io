---
title: 完全掌握Android Data Binding(MVVM)-入门篇
tags: Architecture
description: MVVM架构的基本使用
categories: Android
---
## MVVM风流史
-  出生时间不详，Android Studio 1.3时需要通过依赖库添加`com.android.databinding `
- 于Android Studio 2.0被内置，直接配置
```gradle
android {
    ....
    dataBinding {
        enabled = true
    }
}
```
- 于Android Studio2.1 Preview 3开始支持双向绑定，从此人生走入正轨
- 于2017年下半年 伴随着 [Android Architecture Components](https://developer.android.google.cn/topic/libraries/architecture/index.html) 1.0 stable发布，人生从此进入了巅峰

>PS: MVVM 是一种思想，数据的双向绑定,并不是Android特有的，其他的比如AngularJS和IOS的MVVM
最初我还以为是我们大Google的专利[尴尬脸]😌

## 一个优秀的男人需要的气质
###  数据驱动
在我们平常的开发，当数据变化需要更新UI的时候，需要先获取UI控件的引用（findViewById）,然后去更新UI，获取控件的属性也是需要通过UI控件的引用。在MVVM中，这些都是通过数据驱动来自动完成的，数据变化后会自动更新UI，UI的改变也能自动反馈到数据层，数据成为主导因素。这样MVVM层在业务逻辑处理中只要关心数据，不需要直接和UI打交道，在业务处理过程中简单方便很多。
###  低耦合度
MVVM模式中，数据是独立于UI的。

数据和业务逻辑处于一个独立的ViewModel中，ViewModel只需要关注数据和业务逻辑，不需要和UI或者控件打交道。UI想怎么处理数据都由UI自己决定，ViewModel不涉及任何和UI相关的事，也不持有UI控件的引用。即便是控件改变了（比如：TextView换成EditText），ViewModel也几乎不需要更改任何代码。它非常完美的解耦了View层和ViewModel
### 更新UI
在MVVM中，数据发生变化后，我们在工作线程直接修改（在数据是线程安全的情况下）ViewModel的数据即可，不用再考虑要切到主线程更新UI了，这些事情相关框架都帮我们做了。
### 可复用性
一个ViewModel可以复用到多个View中。同样的一份数据，可以提供给不同的UI去做展示。对于版本迭代中频繁的UI改动，更新或新增一套View即可。如果想在UI上做A/B Testing，那MVVM是你不二选择。

>PS: 本部分优势参考[这篇文章](https://tech.meituan.com/android_mvvm.html)

## 如何成为一个优秀的男人
###  如何把数据和View进行绑定
把大象装入箱子需要三部

- 添加一个POJO类
```java
    public class User {
        private final String firstName;
        private final String lastName;
        private String displayName;
        private int age;
     
        public User(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }
     
        public User(String firstName, String lastName, int age) {
            this(firstName, lastName);
            this.age = age;
        }
     
        public int getAge() {
            return age;
        }
     
        public String getFirstName() {
            return firstName;
        }
     
        public String getLastName() {
            return lastName;
        }
    }
```
- 定义与使用Variable
   
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">
<data>
    <import type="android.graphics.drawable.Drawable"/>
    <variable name="user"  type="com.example.User"/>
    <variable name="image" type="Drawable"/>
    <variable name="note"  type="String"/>
<variable
    name="viewModel"
    type="com.demo.tangminglong.mvvmprojecttest.viewmodel.MainViewModel"/>
 </data>
    ...
<Button
            android:padding="10dp"
            android:layout_gravity="center_horizontal"
            android:gravity="center"
            android:layout_width="200dp"
            android:layout_height="50dp"
            android:text="@{user.firstName}"
         />
 </layou>

```

一个根据layout.xml将自动生成一个以Binding结尾的类文件，例如`main_activity.xml `将生成MainActivityBinding,生成的代码位置如下图所示
![847D2E67-524A-451D-97D6-2CE117B4B674.png](http://upload-images.jianshu.io/upload_images/1263922-61de051a521349e7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 绑定Variable

	Binding类根据layout.xml定义的变量（variable）,将会自动生成相应的set方法，同时我们通常的setContentView(int)也被替换，如下代码

```java
ActivityMainBinding binding = DataBindingUtil.setContentView(this,R.layout.activity_main);
        mainViewModel = new MainViewModel(this,this);
        binding.setViewModel(mainViewModel);
```

### 如何更新数据
一个POJO值改变并不能够自动更新UI，dataBing的强大之处在于值改变能够自动同步更新UI(单向绑定)，实现的方式有如下三种

- **Observable Objects** 
用一个类来实现[Observable](https://developer.android.google.cn/reference/android/databinding/Observable.html)为了方便，Android 原生提供了已经封装好的一个类 - BaseObservable，并且实现了监听器的注册机制。
我们可以直接继承BaseObservable

```java
private static class User extends BaseObservable {
   private String firstName;
   private String lastName;
   @Bindable
   public String getFirstName() {
       return this.firstName;
   }
   @Bindable
   public String getLastName() {
       return this.lastName;
   }
   public void setFirstName(String firstName) {
       this.firstName = firstName;
       notifyPropertyChanged(BR.firstName);
   }
   public void setLastName(String lastName) {
       this.lastName = lastName;
       notifyPropertyChanged(BR.lastName);
   }
}
```

BR 是编译阶段生成的一个类，功能与 R.java 类似，用 @Bindable 标记过 getter 方法会在 BR 中生成一个 entry，当我们

通过代码可以看出，当数据发生变化时还是需要手动发出通知。 通过调用notifyPropertyChanged(BR.firstName)来通知系统 BR.firstName 这个 entry 的数据已经发生变化，需要更新 UI。

- **ObservableFields**
 还有一种更细粒度的绑定方式，可以具体到成员变量，这种方式无需继承 BaseObservable，一个简单的 POJO 就可以实现。
 
```java
private static class User {
   public final ObservableField<String> firstName =
       new ObservableField<>();
   public final ObservableField<String> lastName =
       new ObservableField<>();
   public final ObservableInt age = new ObservableInt();
}
```

系统为我们提供了所有的 primitive type 所对应的 Observable类，例如 ObservableInt、ObservableFloat、ObservableBoolean 等等，还有一个 ObservableField 对应着 reference type。
 
 - **Observable Collections**
对应ObservableFields的集合，用法和ObservableFields差不多

>PS: 双向绑定表达式对应`@={setting.cacheEnable}`,单向绑定对应`@{setting.cacheEnable}`,具体可参照[GitHub的Demo](https://github.com/TMLAndroid/MVVMDemo)

### 事件处理（Event Handling）
Data Binding 可以通过表达式处理View的事件（eg.onClick）,事件处理有两种方式处理，一个是**方法引用（Method Reference）**,一个是**监听绑定（Listener Bingings）**
- **方法引用（Method Reference）**
方法引用的方式`android:onClcik`对应方法Vuew#onClick，因为属性是在编译期进行，所以一旦有错误会立即报错，引用的方式如下

```java
public class MainViewModel {
    private Context context;

    public MainViewModel(Context context) {
        this.context = context;
    }

    public void onClickOne(View view){
        context.startActivity(new Intent(context, SingleMainActivity.class));
    }

    public void onClickTwo(View view){
        context.startActivity(new Intent(context, DoubleMainActivity.class));
    }
}
```


```xml
<Button
            android:padding="10dp"
            android:layout_gravity="center_horizontal"
            android:gravity="center"
            android:layout_width="200dp"
            android:layout_height="50dp"
            android:text="@string/singleBundle"
            android:onClick="@{viewModel::onClickOne}"/>
        <Button
            android:padding="10dp"
            android:layout_gravity="center_horizontal"
            android:gravity="center"
            android:layout_width="200dp"
            android:layout_height="50dp"
            android:text="@string/doubleBundle"
            android:onClick="@{viewModel.onClickTwo}"/>
```

调用方式为`variable.method`或者`variable::method`
注意：方法里面需到参数view
- **监听绑定（Listener Bingings）**
Listener Bingings的优点是可以使用复杂的参数，比如下面接口

```java
public interface ItemEventHandler {
    void clickTitle(View view,Item item);
}
```

布局文件如下

```xml
<TextView
    android:id="@+id/textView"
    android:layout_width="0dp"
    android:layout_height="match_parent"
    android:layout_weight="1"
    android:onClick="@{(view)->handler.clickTitle(view,item)}"
    />
```

两者的区别主要是Method Reference会在绑定数据时创建好View的onClickListener，而Listener Bindings在事件发生时才会创建。
> PS :Method Reference在xml不会自动提示

###  RecyclerView(列表)

 ```java
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
//or
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
  ```

###  自定义属性

```xml
<de.hdodenhof.circleimageview.CircleImageView
                    android:id="@+id/image_owner"
                    android:layout_width="65dp"
                    android:layout_height="65dp"
                    app:imageUrl="@{viewModel.ownerAvatarUrl}"/>
```

这里面有自定义属性imageUrl，这并不是定义在自定义CircleImageView里面，而是通过代码动态添加自定义属性，关键代码如下

```java
 @BindingAdapter({"imageUrl"})
    public static void loadImage(ImageView view, String imageUrl) {
        Picasso.with(view.getContext())
                .load(imageUrl)
                .placeholder(R.drawable.placeholder)
                .into(view);
    }
```

属性imageUrl传入地址就可以实现自动加载图片到ImageView
 自定义属性Attribute默认是找setAttribute方法，对xml传入的值进行设置，eg android:text 对应 setText(String)，与属性相关的还有`@BindingMethods`可以重命名
 
>PS: 我们在编写xml时候有时候调用import的变量Android Studio不能自动提示，原因是对于自定义属性Android Studio对值设置不支持自动提示功能

###  其他
其他一些需要注意的，比如[语法表达](https://developer.android.google.cn/topic/libraries/data-binding/index.html#expression_language)，布局的[inclued](https://developer.android.google.cn/topic/libraries/data-binding/index.html#includes)，[ViewStubs](https://developer.android.google.cn/topic/libraries/data-binding/index.html#viewstubs) 以及[converters](https://developer.android.google.cn/topic/libraries/data-binding/index.html#converters)等更多使用，[参照官方文档](https://developer.android.google.cn/topic/libraries/data-binding/index.html[https://developer.android.google.cn/topic/libraries/data-binding/index.html](https://developer.android.google.cn/topic/libraries/data-binding/index.html)
)
## 填坑
1. 如果出现这样的错误信息`cannot resolve symbol 'ActivityMainBinding'`以为这数据绑定自动生成没有创建成功，试试下面几个方法
	1. 确认Module添加了`dataBinding.enabled = true`  
    2. 确认XML的根布局是`<layout>`   
    3.  检查layout的名称是否正确拼写 eg.activity_main.xml 对应ActivityMainBinding.java
	4.  Run File => Invalidate Caches / Restart to clear the cahes. 
    5.  Run Project => Clean and Project => Re-Build to regenerate the classfile.
    6.  重启Android Studio. 
2. 如果出现错误信息像`**.**.databinding does not exist`很可能是因为在数据绑定到XML时候有错误，检查那些错误（比如忘记import a java class）
3. `dentifiers must have user defined types from the XML file. View is missing it`这通常是因为你忘记import statement或者变量拼写错误  

## 特殊通道👉
如果想*深入了解*，可以参看我的GitHub [MVVMDemo](https://github.com/TMLAndroid/MVVMDemo)
## 参考
[http://xuyushi.github.io/2016/03/05/Android%20MVVM/](http://xuyushi.github.io/2016/03/05/Android%20MVVM/)  
[https://tech.meituan.com/android_mvvm.html](http://xuyushi.github.io/2016/03/05/Android%20MVVM/)  
[https://guides.codepath.com/android/Applying-Data-Binding-for-Views](https://guides.codepath.com/android/Applying-Data-Binding-for-Views)  
[https://developer.android.google.cn/topic/libraries/data-binding/index.html](https://developer.android.google.cn/topic/libraries/data-binding/index.html)  
[http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0603/2992.html](https://developer.android.google.cn/topic/libraries/data-binding/index.html)