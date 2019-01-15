### jni中调用java对象

### 首先创建一个java类
有构造函数,有私有方法,有静态公共方法,有公共发放
```
package www.lixiangfei.top.hello_jni;

import android.os.Build;
import android.util.Log;

public class JniHandler {

    public JniHandler(){
        Log.d("JniHandler","www.lixiangfei.top.hello_jni.JniHandler create");
    }


    /*
     * Print out status to logcat
     */
    private void updateStatus(String msg) {
        if (msg.toLowerCase().contains("error")) {
            Log.e("JniHandler", "Native Err: " + msg);
        } else {
            Log.i("JniHandler", "Native Msg: " + msg);
        }
    }

    /*
     * Return OS build version: a static function
     */
    static public String getBuildVersion() {
        return Build.VERSION.RELEASE;
    }

    /*
     * Return Java memory info
     */
    public long getRuntimeMemorySize() {
        return Runtime.getRuntime().freeMemory();
    }
}

```

c++代码
```
extern "C" JNIEXPORT void JNICALL
Java_www_lixiangfei_top_hello_1jni_MainActivity_JniHandler(
        JNIEnv *env,
        jobject /* this */) {
    //找到JniHandler这个类
    jclass jniHandlerClass = env->FindClass("www/lixiangfei/top/hello_jni/JniHandler");
    //找到默认构造函数
    jmethodID mid = (*env).GetMethodID(jniHandlerClass, "<init>", "()V");
    //通过默认构造函数,创建对象
    jobject jniHandlerObject = env->NewObject(jniHandlerClass, mid);
    //找到类的getRuntimeMemorySize方法,()是空的参数,J是返回类型为Long
    jmethodID memorySizeMethod = (*env).GetMethodID(jniHandlerClass, "getRuntimeMemorySize", "()J");
    //调用对象的对应方法,获取到返回值
    jlong jmemroy = env->CallLongMethod(jniHandlerObject, memorySizeMethod);
    //打印返回值
    __android_log_print(ANDROID_LOG_INFO, "native-lib", "jniHandlerClass memorySizeMethod %d",
                        jmemroy);
    //释放对象
    env->DeleteLocalRef(jniHandlerObject);


    //获取类的静态方法
    jmethodID jniVersionMethod = env->GetStaticMethodID(jniHandlerClass,"getBuildVersion","()Ljava/lang/String;");
    //调用静态方法
    jstring  strVersion = (jstring)env->CallStaticObjectMethod(jniHandlerClass,jniVersionMethod);
    //将jstring转为char*
    const char *charBuildVersion   = env->GetStringUTFChars(strVersion, 0);
    //打印
    __android_log_print(ANDROID_LOG_INFO, "native-lib", "jniHandlerClass getBuildVersion %s",charBuildVersion);

}
```
使用的话
```
//声明下方法
public native void JniHandler();

//然后就可以点击调用了
........
findViewById(R.id.jni_handler).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                JniHandler();
            }
        });
.....        
```
