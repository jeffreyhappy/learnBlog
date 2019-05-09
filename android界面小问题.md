## statusbar透明

activity中
```
 if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
                getWindow().addFlags(WindowManager.LayoutParams.FLAG_DRAWS_SYSTEM_BAR_BACKGROUNDS);
                getWindow().clearFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_STATUS);
                getWindow().addFlags(WindowManager.LayoutParams.FLAG_TRANSLUCENT_NAVIGATION);
                getWindow().setStatusBarColor(Color.TRANSPARENT);
}
```
xml中的根节点
```
    android:fitsSystemWindows="true"
```


#room的使用

gradle依赖
```
    implementation "android.arch.persistence.room:runtime:1.1.1"
    annotationProcessor "android.arch.persistence.room:compiler:1.1.1"
    //输出表结构
    defaultConfig {
        ....
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = ["room.schemaLocation": "$projectDir/schemas".toString()]
            }
        }


    }
```
创建
```
    static AppDatabase db;
    db = Room.databaseBuilder(context, AppDatabase.class, "fire")
                    .allowMainThreadQueries()
                    .build();
```
dao
```
@Database(entities = {FireEntity.class}, version = 1)
public abstract class AppDatabase extends RoomDatabase {
    public abstract FirePlanDao firePlanDao();
}
```

entity
```
@Entity(tableName = "fire_newstype")
public class FireEntity {
    @PrimaryKey
    int n_id;
    String news_title;
    String news_items;
    int  type_id;
    String wavfile;
    String mp3file;
    int top_id;
}
```


获取屏幕的宽高密度
```

 public static void printInfo(Context context){
        DisplayMetrics displayMetrics =  context.getResources().getDisplayMetrics();
        float density = displayMetrics.density;
        float densityDpi = displayMetrics.densityDpi;
        float heightPixels = displayMetrics.heightPixels;
        float widthPixels = displayMetrics.widthPixels;
        Log.d("DisplayHelper","density " + density + " densityDpi " + densityDpi  + "  heightPixels " + heightPixels + " widthPixels " + widthPixels);
    }

```

Dpi（dots per inch像素密度）

指每英寸中的像素数。如160dpi指手机水平或垂直方向上每英寸距离有160个像素点。假定设备分辨率为320*240，屏幕长2英寸宽1.5英寸，dpi=320/2=240/1.5=160

Density（密度）
可以理解为像素密度，手机屏幕dpi与标准mdpi(160dpi)的比值，比如160dpi的手机，density=1，440dpi的手机，density=2.75



#edittext选中未选中效果
style.xml
```
    <style name="MyEditText" parent="Theme.AppCompat.Light">
        <item name="colorControlNormal">@android:color/holo_red_dark</item>
        <item name="colorControlActivated">@android:color/holo_blue_bright</item>
    </style>
```
xml
```
  <EditText
        android:id="@+id/et_1"
        android:theme="@style/MyEditText"
        android:layout_width="100dp"
        android:layout_height="40dp" />
```
