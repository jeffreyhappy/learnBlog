动手写个简单版的ButterKnife加深理解
1. 新建个java lib module
2. 添加auto-service依赖,auto-service用来配置annotationProcessor
```
compile  'com.google.auto.service:auto-service:1.0-rc4'
```
添加javapoet依赖,javapoet用来方便的生成java文件
```
compile 'com.squareup:javapoet:1.11.1'
```
3. 自定义一个AbstractProcessor.重写process方法




### AbstractProcessor
AbstractProcessor这是一个抽象注解处理类,对于大部分具体注解处理来说是一个方便的超类.
```
	process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv)
```
这个方法是个抽象方法,用来具体处理注解
先要重写init方法
```
init(ProcessingEnvironment processingEnv)
```
ProcessingEnvironment会提供各种方便的类
```
//操作元素的工具类
Elements	        getElementUtils()
//操作文件的工具类
Filer	            getFiler()
//当前的位置,应该是运行路径?
Locale	            getLocale()
//用来报告信息,例如错误信息
Messager	        getMessager()
//传递给annotationProcess的各种选项
Map<String,String>	getOptions()
//返回生成的class或者java对应的java版本
SourceVersion	    getSourceVersion()
//操作类的工具类
Types	            getTypeUtils()
```


## 流程
##### 创建个processor的java library.我叫他ap
1. 新建个注解
```
    @Retention(RetentionPolicy.CLASS)
    @Target(ElementType.FIELD)
    public @interface BindView {
        int value() default -1;
    }
```
2. 实现AbstractProcessor
```
@AutoService(Processor.class)
public class MyProcessor extends AbstractProcessor
```

3. 重写process
```
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        Set<? extends Element> bindViewElements = roundEnv.getElementsAnnotatedWith(BindView.class);
        for (Element element : bindViewElements){
            //1.获取包名
            PackageElement packageElement = mElementUtils.getPackageOf(element);
            String pkName = packageElement.getQualifiedName().toString();
            note(String.format("package = %s", pkName));

            //2.获取包装类类型
            TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
            String enclosingName = enclosingElement.getQualifiedName().toString();
            note(String.format("enclosindClass = %s", enclosingElement));


            //因为BindView只作用于filed，所以这里可直接进行强转
            VariableElement bindViewElement = (VariableElement) element;
            //3.获取注解的成员变量名
            String bindViewFiledName = bindViewElement.getSimpleName().toString();
            //3.获取注解的成员变量类型
            String bindViewFiledClassType = bindViewElement.asType().toString();

            //4.获取注解元数据
            BindView bindView = element.getAnnotation(BindView.class);
            int id = bindView.value();
            note(String.format("%s %s = %d", bindViewFiledClassType, bindViewFiledName, id));

            //4.生成文件
            try {
                createFile(enclosingElement, bindViewFiledClassType, bindViewFiledName, id);
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
            return true;

        }
        return false;
    }

    private void createFile(TypeElement enclosingElement, String bindViewFiledClassType, String bindViewFiledName, int id) throws ClassNotFoundException {
        String pkName = mElementUtils.getPackageOf(enclosingElement).getQualifiedName().toString();
        String parentClassName = mElementUtils.getTypeElement(enclosingElement.getQualifiedName()).getSimpleName().toString();

        //文件的名字就是绑定对象的父对象类名+"_bindView"
        String className = parentClassName+"_bindView";

        //构造函数两个参数,一个就是activity,一个就是activity的contentView
        ClassName parentFullClassName = ClassName.get(enclosingElement);
        MethodSpec construct = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                .addParameter(parentFullClassName,"target",Modifier.FINAL)
                .addParameter(VIEW,"contentView")
                .addStatement("target.$N = contentView.findViewById(" + id + ")",bindViewFiledName)
                .build();



        TypeSpec helloWorld = TypeSpec.classBuilder(className)
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)

                .addMethod(construct)
                .build();

        //文件的包名就是父对象的包名
        JavaFile javaFile = JavaFile.builder(pkName, helloWorld)
                .build();

        try {
            //mFiler是在init的时候初始化的
            //mFiler = processingEnv.getFiler();

            javaFile.writeTo(mFiler);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
```

在项目的app下的build.gradle添加注解处理依赖
```
  annotationProcessor project(path:':ap')
```
然后运行Build->make Project.就可以在app\build\generated\source\apt\debug\xxxx\xxxx_bindView了
生成的类如下,调用构造函数后就会给注解过的类赋值.剩下的工作就是找到这个构造函数 ,给他赋值
```
public final class MainActivity_bindView {
  public MainActivity_bindView(final MainActivity target, View contentView) {
    target.viewOne = contentView.findViewById(2131165317);
  }

}
```
##### 使用生成的类,给用注解的对象赋值
创建一个android lib的module.叫MyKnife
```
public class MyKnife {
    public static void bind(Activity activityCompat){
        //1 拼接生成的类名
        //2 反射调用构造函数
        String bindClassName  = activityCompat.getClass().getName() + "_bindView";
        try {
            Class bindClass = Class.forName(bindClassName);
            //调用对应类的构造函数
            Constructor constructor =  bindClass.getConstructor(activityCompat.getClass(),View.class);
            Object object = constructor.newInstance(activityCompat,activityCompat.getWindow().getDecorView());
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```
同样需要引用下
```
implementation project(path: ':myknife')
```
调用的时候
````
   @BindView(R.id.view_one)
    View viewOne;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        MyKnife.bind(this);
        //试试绑定成功了没有
        viewOne.setVisibility(View.GONE);
    }
````

之前的process只能处理一个注解,如果一个类里有多个注解,就需要修改下
保存一个类中的一个对象
```
private static class FindTargetView {
        private String name;
        private int    id;

        public FindTargetView(String name, int id) {
            this.name = name;
            this.id = id;
        }

        public String getName() {
            return name;
        }

        public int getId() {
            return id;
        }
    }
```

在取对象的时候,取出来放到列表里
```
@Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        //这个会把所有注解过的对象返回,会跨很多文件,所以需要HashMap来存储
        Set<? extends Element> bindViewElements = roundEnv.getElementsAnnotatedWith(BindView.class);
        //key是一个类对象,value是该类对象下被注解过的成员变量
        HashMap<TypeElement,ArrayList<FindTargetView>> findMap = new HashMap<>();
        for (Element element : bindViewElements){
            //1.获取包名
            PackageElement packageElement = mElementUtils.getPackageOf(element);
            String pkName = packageElement.getQualifiedName().toString();
            note(String.format("package = %s", pkName));

            //2.获取包装类类型
            TypeElement enclosingElement = (TypeElement) element.getEnclosingElement();
            String enclosingName = enclosingElement.getQualifiedName().toString();
            note(String.format("enclosindClass = %s", enclosingElement));


            //因为BindView只作用于filed，所以这里可直接进行强转
            VariableElement bindViewElement = (VariableElement) element;
            //3.获取注解的成员变量名
            String bindViewFiledName = bindViewElement.getSimpleName().toString();
            //3.获取注解的成员变量类型
            String bindViewFiledClassType = bindViewElement.asType().toString();

            //4.获取注解元数据
            BindView bindView = element.getAnnotation(BindView.class);
            int id = bindView.value();
            note(String.format("%s %s = %d", bindViewFiledClassType, bindViewFiledName, id));

            if (findMap.get(enclosingElement) == null){
                ArrayList<FindTargetView> findTargetViews = new ArrayList<>();
                findTargetViews.add(new FindTargetView(bindViewFiledName,id));
                findMap.put(enclosingElement,findTargetViews);
            }else {
                ArrayList<FindTargetView> findTargetViews = findMap.get(enclosingElement);
                findTargetViews.add(new FindTargetView(bindViewFiledName,id));
            }
        }

        if (findMap.size() > 0 ){
            Iterator<Map.Entry<TypeElement,ArrayList<FindTargetView>>> iterator = findMap.entrySet().iterator();
            while (iterator.hasNext()){
                Map.Entry<TypeElement,ArrayList<FindTargetView>> item = iterator.next();
                createFile(item.getKey(),item.getValue());

            }
        }
        return false;
    }
```
构建java文件,在构造函数中把列表中的对象都加入
```
   private void createFile(TypeElement enclosingElement, ArrayList<FindTargetView> targetViews){
        String pkName = mElementUtils.getPackageOf(enclosingElement).getQualifiedName().toString();
        String parentClassName = mElementUtils.getTypeElement(enclosingElement.getQualifiedName()).getSimpleName().toString();

        String className = parentClassName+"_bindView";

        ClassName parentFullClassName = ClassName.get(enclosingElement);
        MethodSpec.Builder  constructBuilder = MethodSpec.constructorBuilder()
                .addModifiers(Modifier.PUBLIC)
                .addParameter(parentFullClassName,"target",Modifier.FINAL)
                .addParameter(VIEW,"contentView");
        for (FindTargetView item : targetViews){
            constructBuilder
                    .addStatement("target.$N = contentView.findViewById(" + item.id + ")",item.name);
        }
        MethodSpec construct =    constructBuilder.build();


        TypeSpec bindView = TypeSpec.classBuilder(className)
                .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                .addMethod(construct)
                .build();

        JavaFile javaFile = JavaFile.builder(pkName, bindView)
                .build();

        try {
            javaFile.writeTo(mFiler);
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
```


如果需要调试:https://www.jianshu.com/p/80a14bc35000

新路历程:一开始找各种demo运行出各种问题,学习新技术的时候就该应该直接找用这个功能的项目,例如annotationProcessor就直接看ButterKnife.边看边抄
