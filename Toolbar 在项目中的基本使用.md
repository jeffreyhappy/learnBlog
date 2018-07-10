### 基本说明
  Toolbar已经出现很久了，Toolbar主要比ActionBar支持了更多常用特性。例如：导航按钮，品牌logo图片,标题,子标题等等。

  然而他是长这个样子的：

  ![Toolbar](http://upload-images.jianshu.io/upload_images/2120696-e9b2de753ee4fa86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  正常项目要求都是这样的
![1.png](http://upload-images.jianshu.io/upload_images/2120696-1f6b4d82e22e409a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
  不管怎么样，我们先来熟悉下Toolbar的使用。以下文字来源于android api官网
  简单说明：
>  您应使用支持库的 Toolbar 类来实现 Activity 的应用栏。使用支持库的工具栏有助于确保您的应用在最大范围的设备上保持一致的行为。例如，Toolbar 小部件能够在运行 Android 2.1（API 级别 7）或更高版本的设备上提供 Material Design 体验，但除非设备运行的是 Android 5.0（API 级别 21）或更高版本，否则原生操作栏不会支持 Material Design。
  那我们就根据项目来设置一下

  步骤简述：
1  在build.gradle加入如下依赖

```
  compile 'com.android.support:appcompat-v7:24.2.1'
```
2  app使用的主题要为无ActionBar主题

直接使用系统的（正常不这么干）
```
<application
    android:theme="@style/Theme.AppCompat.Light.NoActionBar"
    />
```
继承系统的，正常这么干  在styles文件夹下
```
<!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
        <item name="windowActionBar">false</item>  //很重要
        <item name="windowNoTitle">true</item>     //很重要
    </style>
```
然后manifests里这样写
```
<application
    android:theme="@style/AppTheme"
    />
```
3  在activiy的xml布局里加入Toolbar

```
 <android.support.v7.widget.Toolbar
   android:id="@+id/my_toolbar"
   android:layout_width="match_parent"
   android:layout_height="?attr/actionBarSize"
   android:background="?attr/colorPrimary"
   android:elevation="4dp"
   android:theme="@style/ThemeOverlay.AppCompat.ActionBar"
   app:popupTheme="@style/ThemeOverlay.AppCompat.Light"/>
```

4  然后就可以绑定了，在Activity的onCreate加入如下方法

```
import android.support.v7.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_my);
    Toolbar myToolbar = (Toolbar) findViewById(R.id.my_toolbar);//找到toolbar
    setSupportActionBar(myToolbar);//把toolbar和原来的actionbar关联起来

    //以后要对toolbar进行操作的时候需要使用getSupportActionBar()，例如：

    getSupportActionBar().setDisplayShowTitleEnabled(false);//不显示标题
    getSupportActionBar().setDisplayHomeAsUpEnabled(true);//显示左边的Home图标为返回按钮

    }
}
```







###正常项目使用
1  设置返回按钮
```
    myToolbar.setNavigationIcon(R.drawable.ic_left_arrow);  //这个ic_left_arrow就是自定义的返回按钮，
```

2  监听用户点击返回按钮，只需要复写Activity的onOptionsItemSelected()

```
  @Override
  public boolean onOptionsItemSelected(MenuItem item) {
      switch (item.getItemId()) {
          case android.R.id.home: //这个就是toolbar的返回按钮空间的id
              onBackPressed();
              break;
      }
      return true; //告诉系统我们自己处理了点击事件
  }
```
3  设置标题
  系统的标题是居左的，我们需要的是居中的,只需要添加一个居中子TextView

```
<android.support.v7.widget.Toolbar
  android:id="@+id/my_toolbar"
  android:layout_width="match_parent"
  android:layout_height="?attr/actionBarSize"
  android:background="?attr/colorPrimary"
  android:elevation="4dp"
  android:theme="@style/ThemeOverlay.AppCompat.ActionBar"
  app:popupTheme="@style/ThemeOverlay.AppCompat.Light">

  <TextView
    android:id="@+id/tv_title"
    style="@style/TextAppearance.AppCompat.Widget.ActionBar.Title"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:layout_gravity="center" />

</android.support.v7.widget.Toolbar>
```

然后只需要

```
TextView title = (TextView) findViewById(R.id.tv_title);
title.setText("标题");//直接设置标题就可以了
```

4  设置菜单
  设置菜单需要重写activity的onCreateOptionsMenu()方法

```
  <menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <item
      android:id="@+id/menu_text_1"
      android:title="菜单1"
      app:showAsAction="always" />
    <item
      android:id="@+id/menu_text_2"
      android:title="菜单2"
      app:showAsAction="always" />
</menu>
  ```

  R.menu.menu_test在res的menu文件夹下

  ```
  @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.menu_test, menu);
        return true;
    }
  ```

  这样有两个文字菜单按钮的toolbar就出现了，
  点击事件的响应也和返回按钮的响应一样

  ```
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()) {
            case android.R.id.home: //这个就是toolbar的返回按钮空间的id
                onBackPressed();
                break;
            case R.id.menu_text_2:
                //做一些事
                break;
            case R.id.menu_text_2:
                //做一些事
                break;
        }
        return true; //告诉系统我们自己处理了点击事件
    }
  ```

![1.png](http://upload-images.jianshu.io/upload_images/2120696-1f6b4d82e22e409a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  现在我们显示图片和文字混合的toolbar

  ```
  <menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context=".view.activity.NormalDetailActivity">


    <item
    android:id="@+id/menu_text_1"
    android:title="菜单1"
    app:showAsAction="always" />


    <item
        android:id="@+id/menu_img"
        android:title="img菜单"
        android:icon="@drawable/icon_share_blue"
        app:showAsAction="always" />
</menu>
  ```

  ![2.png](http://upload-images.jianshu.io/upload_images/2120696-7f3e80074b38e334.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

产品经理说这样还不行，我们需要有checkbox，用户要添加收藏


  ```
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    tools:context=".view.activity.NormalDetailActivity">
    <item
        android:id="@+id/menu_checkbox"
        android:title="checkbox"
        app:actionViewClass="android.support.v7.widget.AppCompatCheckBox"
        app:showAsAction="always" />

    <item
        android:id="@+id/menu_img"
        android:title="img菜单"
        android:icon="@drawable/icon_share_blue"
        app:showAsAction="always" />

</menu>
  ```

  这样checkbox就出来了

![3.png](http://upload-images.jianshu.io/upload_images/2120696-d556c9c274bc9fe2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![4.png](http://upload-images.jianshu.io/upload_images/2120696-ac7b77c7894327c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


  这样肯定不过关，然后我们用代码把checkbox的样式修改下

```
  @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        getMenuInflater().inflate(R.menu.menu_test, menu);

        //找到checkbox设置下样式
        AppCompatCheckBox checkbox = (AppCompatCheckBox) menu.findItem(R.id.menu_checkbox).getActionView();
        checkbox.setButtonDrawable(R.drawable.selector_favour);
        checkbox.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
            @Override
            public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                //响应消息
            }
        });
        return true;
    }
```

selector_favour如下

```
<?xml version="1.0" encoding="utf-8"?>
<selector xmlns:android="http://schemas.android.com/apk/res/android">
    <item android:drawable="@drawable/ic_favorite_selected" android:state_checked="true"/>
    <item android:drawable="@drawable/ic_favorite_normal" android:state_checked="false"/>
    <item android:drawable="@drawable/ic_favorite_normal" />
</selector>
```

这样大概符合各种需求了吧

![5.png](http://upload-images.jianshu.io/upload_images/2120696-2bbc7e74a7f64377.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![6.png](http://upload-images.jianshu.io/upload_images/2120696-1bd6ae22327e852c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


5 修改菜单文字颜色

  从上面看我们的菜单文字颜色是黑色的，可是图标都是绿的，肯定要统一颜色。

  修改toolbar的style中actionMenuTextColor属性
```
  <style name="ToolbarStyle" parent="ThemeOverlay.AppCompat.ActionBar">
      <item name="actionMenuTextColor">@color/blue</item>
  </style>
```

  然后修改toolbar的android:theme为我们刚刚定义的

```
   <android.support.v7.widget.Toolbar
     android:id="@+id/my_toolbar"
     android:layout_width="match_parent"
     android:layout_height="?attr/actionBarSize"
     android:background="?attr/colorPrimary"
     android:elevation="4dp"
     android:theme="@style/ToolbarStyle"
     app:popupTheme="@style/ThemeOverlay.AppCompat.Light"/>
```
这样样式就统一了
![7.png](http://upload-images.jianshu.io/upload_images/2120696-c1e441a32e1d643f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最后罗嗦一句，这个toolbar还是不好用，我现在用自己写的。^_^
