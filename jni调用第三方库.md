## 静态库和共享库

从使用方法上分库大体上可以分为两类：静态库和共享库。在windows中静态库是以 .lib 为后缀的文件，共享库是以.dll 为后缀的文件。在linux中静态库是以 .a 为后缀的文件，共享库是以 .so为后缀的文件。

当程序与静态库连接时，库中目标文件所含的所有将被程序使用的函数的机器码被copy到最终的可执行文件中。这就会导致最终生成的可执行代码量相对变多，相当于编译器将代码补充完整了，这样运行起来相对就快些。不过会有个缺点: 占用磁盘和内存空间. 静态库会被添加到和它连接的每个程序中,而且这些程序运行时, 都会被加载到内存中. 无形中又多消耗了更多的内存空间.

与共享库连接的可执行文件只包含它需要的函数的引用表，而不是所有的函数代码，只有在程序执行时,那些需要的函数代码才被拷贝到内存中。这样就使可执行文 件比较小,节省磁盘空间，更进一步，操作系统使用虚拟内存，使得一份共享库驻留在内存中被多个程序使用，也同时节约了内存。不过由于运行时要去链接库会花 费一定的时间，执行速度相对会慢一些，总的来说静态库是牺牲了空间效率，换取了时间效率，共享库是牺牲了时间效率换取了空间效率，没有好与坏的区别，只看 具体需要了。'


CMake的文档可以看 https://cmake.org/cmake/help/latest/command/target_include_directories.html#command:target_include_directories

看googleDemo中
```
 sourceSets {
        main {
            // let gradle pack the shared library into apk
            jniLibs.srcDirs = ['../distribution/gperf/lib']
        }
    }
```
想到了动态库是需要打包进入apk中的,demo中使用了静态库和共享库,像百度地图等第三方都是给.so文件,所以一步一步来

### 引用共享库
1 把hello-libs的库拷贝到我们项目的app目录下
2 在app的build.gradle中加入,跟引入百度地图的so一样的
```
sourceSets {
        main {
            // let gradle pack the shared library into apk
            jniLibs.srcDirs = ['../distribution/gperf/lib']
        }
    }
```
3 修改CMakelist.txt,把下面的内容加进去
3.1 定义lib_gperf类型
3.2 定义lib_gperf库的位置
3.3 链接lib_gperf头文件
3.4 将lib_gperf链接到native-lib里
```

# configure import libs
# 定义一个变量叫distribution_DIR ,值是后面的
set(distribution_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../distribution)

# shared lib will also be tucked into APK and sent to target
# refer to app/build.gradle, jniLibs section for that purpose.
# ${ANDROID_ABI} is handy for our purpose here. Probably this ${ANDROID_ABI} is
# the most valuable thing of this sample, the rest are pretty much normal cmake
# 添加一个叫lib_gperf的共享库SHARED,  来源是导入接口库 interface library
add_library(lib_gperf SHARED IMPORTED)

# 给lib_gperf 设置属性IMPORTED_LOCATION,属性是位置
set_target_properties(lib_gperf PROPERTIES IMPORTED_LOCATION
        ${distribution_DIR}/gperf/lib/${ANDROID_ABI}/libgperf.so)
		
.....
原来的内容
.....

# 设置include,貌似就是头文件
target_include_directories(
        native-lib
        PRIVATE
        ${distribution_DIR}/gperf/include)


target_link_libraries( # Specifies the target library.
                       native-lib
                       lib_gperf
                       # Links the target library to the log library
                       # included in the NDK.
                        log )		


```
4 这样就关联起来了,剩下的就是正常使用了,写一个C++方法测试下
```
#include <gperf.h>

extern "C" JNIEXPORT jlong JNICALL
Java_www_lixiangfei_top_hello_1jni_MainActivity_testSo(
        JNIEnv *env,
        jobject instance/* this */) {
    uint64_t  time = GetTicks();
    return time;
}

```

### 引用静态库
1. 静态库就是.a之前一起把文件拷贝过来了
2. 修改CMakeLists.txt ,照着上面的流程来一套
```
# 添加一个叫lib_gmath的静态库STATIC,  来源是导入接口库 interface library
add_library(lib_gmath STATIC IMPORTED)
# 给lib_gmath 设置属性IMPORTED_LOCATION,属性是位置
set_target_properties(lib_gmath PROPERTIES IMPORTED_LOCATION
        ${distribution_DIR}/gmath/lib/${ANDROID_ABI}/libgmath.a)
		# 设置include,貌似就是头文件
target_include_directories(
        native-lib
        PRIVATE
        ${distribution_DIR}/gperf/include
        ${distribution_DIR}/gmath/include)
		
target_link_libraries( # Specifies the target library.
                       native-lib
                       lib_gperf
                       lib_gmath
                       # Links the target library to the log library
                       # included in the NDK.
                        log )
```
4. 可以测试了
```
#include <gmath.h>
extern "C" JNIEXPORT int JNICALL
Java_www_lixiangfei_top_hello_1jni_MainActivity_testStatic(
        JNIEnv *env,
        jobject instance/* this */) {
    int   result = gpower(2);
    return result;
}
```
测试通过~~~


NDK的文档:https://developer.android.google.cn/ndk/guides/audio/
