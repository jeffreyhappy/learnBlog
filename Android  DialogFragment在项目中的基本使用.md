---
title: Android  DialogFragment在项目中的基本使用
---

app打开dialogFragment后 statusbar颜色会不对。正好来复习下DialogFragment的使用，以巩固下基础


![问题.png](http://upload-images.jianshu.io/upload_images/2120696-376b6b61c5d9c668.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


测试了一下发现，我需要对话框宽度全屏，并且保持在底部。果然问题出现在全屏这里，如果view的宽高全是match_parent，对话框内容后的背景色就会消失
````
@Override
    public void onStart() {
        super.onStart();
        Dialog dialog = getDialog();
        if (dialog != null )
        {
           //如果宽高都为MATCH_PARENT,内容外的背景色就会失效，所以只设置宽全屏
            int width = ViewGroup.LayoutParams.MATCH_PARENT;
            int height = ViewGroup.LayoutParams.WRAP_CONTENT;
            dialog.getWindow().setLayout(width, height);//全屏
            dialog.getWindow().setGravity(Gravity.BOTTOM);//内容设置在底部
            //内容的背景色.系统的内容宽度是不全屏的，替换为自己的后宽度可以全屏
            dialog.getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        }
    }
````
测试DialogFragment类
````
/**
 * 内容居下，宽度填满屏幕的dialogFragment
 * Created by Li on 2017/8/3.
 */
public class TestDialog extends DialogFragment {

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        //DialogFragment.STYLE_NO_FRAME 没有边框，
        //R.style.dialogTheme 主要就是设置对话框内容区域外的背景色，
        setStyle(DialogFragment.STYLE_NO_FRAME,R.style.dialogTheme);
    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.dialog_test,container,false);
        return view;
    }

    @Override
    public void onStart() {
        super.onStart();
        Dialog dialog = getDialog();
        if (dialog != null )
        {
           //如果宽高都为MATCH_PARENT,内容外的背景色就会失效，所以只设置宽全屏
            int width = ViewGroup.LayoutParams.MATCH_PARENT;
            int height = ViewGroup.LayoutParams.WRAP_CONTENT;
            dialog.getWindow().setLayout(width, height);//全屏
            dialog.getWindow().setGravity(Gravity.BOTTOM);//内容设置在底部
            //内容的背景色.对于全屏很重要，系统的内容宽度是不全屏的，替换为自己的后宽度可以全屏
            dialog.getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        }
    }
}
````
再看下style
````
<style name="dialogTheme" parent="Theme.AppCompat.Dialog">
        <!-- 边框 -->
        <item name="android:windowFrame">@null</item >
        <!-- 测试了下也是内容背景色 会覆盖android:windowBackground-->
        <!--<item name="android:background">@android:color/holo_red_dark</item>-->
        <!--
            如果width和height都是MATCH_PARENT 对话框外的背景色就无效了
            int width = ViewGroup.LayoutParams.MATCH_PARENT;
            int height = ViewGroup.LayoutParams.WRAP_CONTENT;
            dialog.getWindow().setLayout(width, height);//全屏
            dialog.getWindow().setGravity(Gravity.BOTTOM);//内容设置在底部
            gravity 和 宽高设置实测无效！！！！！！还是需要代码来设置
        -->
        <item name="android:layout_width">match_parent</item>
        <item name="android:layout_height">wrap_content</item>
        <!-- 内容的背景色.对于全屏很重要，系统的内容宽度是不全屏的，替换为自己的后宽度可以全屏-->
        <!--相当于 dialog.getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));-->
        <item name="android:windowBackground">@android:color/transparent</item>
        <!-- 是否有背景色 -->
        <item name="android:backgroundDimEnabled">true</item>
        <!-- 灰色的百分比 0就是全透明了-->
        <item name="android:backgroundDimAmount">0.4</item>
    </style>
````
使用就很简单了
````
TestDialog dialog = new TestDialog();
dialog.show(getSupportFragmentManager(),"test");
````

再复习下最简单的AlertDialog
````
public class TestAlertDialog extends DialogFragment {

    public static TestAlertDialog newInstance(String title ,String msg) {
        TestAlertDialog frag = new TestAlertDialog();
        Bundle args = new Bundle();
        args.putString("title", title);
        args.putString("msg", msg);
        frag.setArguments(args);
        return frag;
    }

    @Override
    public Dialog onCreateDialog(Bundle savedInstanceState) {
        String title = getArguments().getString("title");
        String msg = getArguments().getString("msg");

        return new AlertDialog.Builder(getActivity())
                .setTitle(title)
                .setMessage(msg)
                .setPositiveButton(android.R.string.ok,
                        new DialogInterface.OnClickListener() {
                            public void onClick(DialogInterface dialog, int whichButton) {
                                Toast.makeText(getActivity(),"你点击了确认",Toast.LENGTH_LONG).show();
                            }
                        }
                )
                .setNegativeButton(android.R.string.cancel,
                        new DialogInterface.OnClickListener() {
                            public void onClick(DialogInterface dialog, int whichButton) {
                                Toast.makeText(getActivity(),"你点击了取消",Toast.LENGTH_LONG).show();
                            }
                        }
                )
                .create();
    }
}

````

![alertDialog.png](http://upload-images.jianshu.io/upload_images/2120696-1f0f1dda2fd099af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这里按钮的颜色是通过colorAccent控制的
![accentColor.png](http://upload-images.jianshu.io/upload_images/2120696-c33b94a2ed9b43cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
