---
title: java注解的学习和了解ButterKnife原理
---

## 注解的接口定义

简单的定义
```
package com.happy.fei.lib;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface FruitName {
    String value() default "苹果";
}
```
Target标识是写在哪的注解
ElementType.TYPE 定义在类上的
ElementType.FIELD 定义在成员变量上
ElementType.METHOD 定义在方法上
ElementType.PARAMETER 定义在参数上的
ElementType.CONSTRUCTOR 定义在构造函数上的
ElementType.LOCAL_VARIABLE 局部变量
ElementType.ANNOTATION_TYPE 定义在注解上,例如Target就是被他标识的
ElementType.PACKAGE 定义在包上

Retention 是保留的意思,意味着这个注解保留多久
RetentionPolicy.SOURCE 在源代码里保留,编译成class就没了
RetentionPolicy.CLASS 在class里保留,jvm加载后就没了
RetentionPolicy.RUNTIME 一直保留,jvm加载后也可以读到

定义个类,然后在成员变量上使用注解,这个注解的值就是banana
```
public class Fruit {
    @FruitName(value = "banana")
    private String name;
    @FruitColor
    private String color;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    @Override
    public String toString() {
        return "name " + name + " color " + color;
    }
}
```
虽然注解了,但是你new一个对象出来后,直接取name的值还是空的,需要处理下.
下面就是处理了
1. 读取被注解的成员变量
2. 读取注解的值
3. 通过Method方法调用对象的setXXX方法.把注解的值传进去.
这样对于的name值就变成了banana了

```
     private static void testFruit(Object object) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {

        //取对象的类
        Class<?> clazz = object.getClass();
        //取对象的成员变量
        Field[] fields = clazz.getDeclaredFields();
        //遍历对象的所有成员变量
        for (Field field : fields){
            //是否被FruitColor注解了
            if (field.isAnnotationPresent(FruitColor.class)){
                //注解了就拿注解的值
                FruitColor fruitColor = field.getAnnotation(FruitColor.class);
                //找到该类的setXXX方法
                Method setMethod = clazz.getDeclaredMethod("set"+upperCase(field.getName()),String.class);
                //调用setXXX方法,把值放进去
                setMethod.invoke(object,fruitColor.fruitColor().toString());
                System.out.println("color " + fruitColor.fruitColor().toString());
            }
            if (field.isAnnotationPresent(FruitName.class)){
                FruitName name = field.getAnnotation(FruitName.class);
                Method setMethod = clazz.getDeclaredMethod("set"+upperCase(field.getName()),String.class);
                setMethod.invoke(object,name.value().toString());
                System.out.println("name " + name.value());

            }
        }
        //打印类和对象成员参数的内容
        System.out.println("class =  " + object.getClass().getSimpleName() + " " + object.toString());
    }
```
测试下
```
        Fruit fruit = new Fruit();
        Orange orange = new Orange();
        testFruit(fruit);
       testFruit(orange);
```
##ButterKnife
项目中注解用的最多的应该是ButterKnife了.
ButterKnife用的RetentionPolicy.CLASS
安卓中 Java源码 —> Class文件 —> Dex文件 —> apk .CLASS是给开发编译过程的人使用的 其实SOURCE就可以了,为毛要CLASS.
另外ButterKnife已经不使用APT而使用annotationProcessor来进行编译

ButterKnife的大体流程
1. 使用BindView注解成员对象
1. 编译项目的时候,ButterKnife在app\build\generated\source\apt下生成XXX_ViewBind.java文件
1. 每个有注解的View,Activity等等都需要调用ButterKnife.bind(this)
1. bind就是根据View的类名去找对应生成的类
1. 找到对应的类后,通过反射调用构造函数,构造函数的参数就是注解成员变量所属的类对象和View对象
1. 构造函数里调用了 findViewByID来通过id找到view,并赋值

ButterKnife 编译的时候生成XXXX_ViewBinding.java XXXX_ViewBinding(T target, View source) 为它的构造函数 T是我们注解对象类的子类所以target就是我们将要操作的对象,source是view对象,在函数里
```
target.view_divider = Utils.findRequiredView(source, R.id.view_divider, "field 'view_divider'");
```
帮我们把对象找到了 这个也解释了为什么注解的对象不能是私有函数 而如何才会调用这个构造函数来实现对象的绑定呢?

关键代码就在下面,通过我添加的注释就可以看到找文件的过程
```
  public static Unbinder bind(@NonNull Object target, @NonNull Dialog source) {
    View sourceView = source.getWindow().getDecorView();
    return createBinding(target, sourceView);
  }

 private static Unbinder createBinding(@NonNull Object target, @NonNull View source) {
    Class<?> targetClass = target.getClass();
    if (debug) Log.d(TAG, "Looking up binding for " + targetClass.getName());
    //通过target来找到对应生成类的构造函数
    Constructor<? extends Unbinder> constructor = findBindingConstructorForClass(targetClass);

    if (constructor == null) {
      return Unbinder.EMPTY;
    }

    //noinspection TryWithIdenticalCatches Resolves to API 19+ only type.
    try {
      // 如果找到了就调用构造函数,构造函数就参数就是target,source
      return constructor.newInstance(target, source);
    } catch (IllegalAccessException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InstantiationException e) {
      throw new RuntimeException("Unable to invoke " + constructor, e);
    } catch (InvocationTargetException e) {
      Throwable cause = e.getCause();
      if (cause instanceof RuntimeException) {
        throw (RuntimeException) cause;
      }
      if (cause instanceof Error) {
        throw (Error) cause;
      }
      throw new RuntimeException("Unable to create binding instance.", cause);
    }
  }

private static Constructor<? extends Unbinder> findBindingConstructorForClass(Class<?> cls) {
    //BINDINGS是个Map表,用来缓存之前已经找到过的构造方法
    Constructor<? extends Unbinder> bindingCtor = BINDINGS.get(cls);
    if (bindingCtor != null) {
      if (debug) Log.d(TAG, "HIT: Cached in binding map.");
      return bindingCtor;
    }
    //android和java开头的是系统的类,直接跳过
    String clsName = cls.getName();
    if (clsName.startsWith("android.") || clsName.startsWith("java.")) {
      if (debug) Log.d(TAG, "MISS: Reached framework class. Abandoning search.");
      return null;
    }

    try {
    //找类的类名就是target的类+_ViewBinding
      Class<?> bindingClass = Class.forName(clsName + "_ViewBinding");
      //noinspection unchecked
      //确认下构造函数对不对
      bindingCtor = (Constructor<? extends Unbinder>)
      bindingClass.getConstructor(cls, View.class);
      if (debug) Log.d(TAG, "HIT: Loaded binding class and constructor.");
    } catch (ClassNotFoundException e) {
      if (debug) Log.d(TAG, "Not found. Trying superclass " + cls.getSuperclass().getName());
      bindingCtor = findBindingConstructorForClass(cls.getSuperclass());
    } catch (NoSuchMethodException e) {
      throw new RuntimeException("Unable to find binding constructor for " + clsName, e);
    }
    //完了放进去,再把构造对象丢出去
    BINDINGS.put(cls, bindingCtor);
    return bindingCtor;
  }
```
