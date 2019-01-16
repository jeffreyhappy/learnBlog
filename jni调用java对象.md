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



## 第二天

任务:在c++中开启线程,然后不停的调用java方法

#### 使用std::thread相关的方法来创建线程
c++的测试代码
```
#include "stdafx.h"
#include <iostream>
#include <thread>
#include <stdio.h>

using namespace std;


void foo()
{
	int count = 0;
	while (true)
	{
		printf("foo %d\n",count);
		count++;
		this_thread::sleep_for(chrono::milliseconds(100));
		if (count == 20) {
			break;
		}
	}
}

void bar(int x)
{
	int count = 0;
	while (true)
	{
		printf("bar %d\n", count);
		count++;
		this_thread::sleep_for(chrono::milliseconds(100));
		if (count == 20) {
			break;
		}
	}
}



void testJoin() {
	printf("start\n");

	//创建完了之后,就直接运行了
	thread first(foo);
	thread second(bar, 0);

	//等待first运行结束
	first.join();
	//等待second运行结束
	second.join();
	printf("end\n");
	getchar();
}

void testDetach() {
	printf("start\n");

	//创建完了之后,就直接运行了
	thread first(foo);
	thread second(bar, 0);

	//不等待结果
	first.detach();
	//不等待结果
	second.detach();
	printf("end\n");
	getchar();

}



void testDonothing() {
	printf("start\n");

	//创建完了之后,就直接运行了
	thread first(foo);
	thread second(bar, 0);

	//默认就是detach的,在安卓中还是需要detach,不然寻址错误
	printf("end\n");
	getchar();

}


int main()
{
	testDonothing();
	getchar();
    return 0;
}


```
了解c++创建线程后,可以搞jni了

在java中新建个私有方法,然后每次更新tv,对于c++来说,java的方法隔离没用,它都可以调用
```
private void updateText(final String text){
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                tv.setText(text);
            }
        });
    }
```

新建个jni方法叫startThread
里面创建了个新的线程,执行foo方法,特别要注意的是JNIEnv *env,jobject instance这两个对象在方法结束后就销毁了,如果直接传递到foo里,会报指针错误
```
//全局对象,给子线程使用的
JavaVM* javaVM;
jobject mainActivityObj;
//给线程退出用的
bool taskRun = false;


//jni装载的时候调用
JNIEXPORT jint JNICALL JNI_OnLoad(JavaVM* vm, void* reserved){
    javaVM = vm;
    __android_log_print(ANDROID_LOG_INFO, "native-lib", "JNI_OnLoad");
    return JNI_VERSION_1_6;

}


void foo()
{


    JNIEnv *env;
	//这个javaVM是全局对象,看上面的onLoad
    jint res = javaVM->GetEnv((void**)&env, JNI_VERSION_1_6);
    if (res != JNI_OK) {
        res = javaVM->AttachCurrentThread( &env, NULL);
        if (JNI_OK != res) {
            __android_log_print(ANDROID_LOG_INFO, "native-lib", "Failed to AttachCurrentThread, ErrorCode = %d", res);
            return ;
        }
    }


    int count = 0;
    while (taskRun)
    {

        std::stringstream ss ;
        ss << "foo "<<count ;
        std::string msg = ss.str();
        const char *charMsg   = msg.c_str();


        jclass clz = env->GetObjectClass(mainActivityObj);
        jmethodID  methodId = env->GetMethodID(clz,"updateText","(Ljava/lang/String;)V");
        jstring javaMsg = env->NewStringUTF( charMsg);
        env->CallVoidMethod(mainActivityObj,methodId,javaMsg);
        __android_log_print(ANDROID_LOG_INFO, "native-lib", "foo %d",count);
        count++;
		//睡眠1秒
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
	//删除全局引用
	env->DeleteGlobalRef(mainActivityObj);
	//线程使用完了要销毁,不然会报错
    javaVM->DetachCurrentThread();
}


/**
* 开始线程
**/
extern "C" JNIEXPORT void JNICALL
Java_www_lixiangfei_top_hello_1jni_MainActivity_startThread(
        JNIEnv *env,
        jobject instance/* this */) {
    
    __android_log_print(ANDROID_LOG_INFO, "native-lib", "start thread start");
    //测试下能否直接调用updateText方法
	jclass clz = env->GetObjectClass(instance);
    jmethodID  methodId = env->GetMethodID(clz,"updateText","(Ljava/lang/String;)V");
    jstring javaMsg = env->NewStringUTF( "update start");
    env->CallVoidMethod(instance,methodId,javaMsg);
    //根据instance创建下全局对象
	mainActivityObj = env->NewGlobalRef(instance);
	//创建完了,线程就开始跑了
    std::thread first(foo);
	//设置线程可以异步运行,不用等待结束
    first.detach();
	
    __android_log_print(ANDROID_LOG_INFO, "native-lib", "start thread done");
}


/**
* 结束线程
**/
extern "C" JNIEXPORT void JNICALL
Java_www_lixiangfei_top_hello_1jni_MainActivity_stopThread(
        JNIEnv *env,
        jobject instance/* this */) {
    taskRun = false;
}

```
通过java测试通过
```
findViewById(R.id.start_thread).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                startThread();
            }
        });

findViewById(R.id.stop_thread).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                stopThread();
            }
        });
```

最后,感觉c++这边类型转换比较操蛋,还是java的+好用.
### c++的拼接
```
void testFormat() {
	//string 和 string拼接很方便
	string s1 = " hello ";
	string s2 = " world ";
	string result = s1 + s2;
	//打印要的类型是const char*
	printf((result+"\n").c_str());

	//string 和int 拼接使用std::to_string() 这个的头是<string>
	int i1 = 1;
	string result2 = s1 + std::to_string(i1) + "\n";
	printf(result2.c_str());
}

```







