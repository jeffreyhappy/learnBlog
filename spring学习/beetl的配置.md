config类
```
@Configuration
public class BeetlConfig {

    @Bean(initMethod = "init")
    public BeetlGroupUtilConfiguration beetlConfiguration() {
        BeetlGroupUtilConfiguration beetlGroupUtilConfiguration = new BeetlGroupUtilConfiguration();
        ResourcePatternResolver patternResolver = ResourcePatternUtils.getResourcePatternResolver(new DefaultResourceLoader());
        try {
            // WebAppResourceLoader 配置root路径是关键
            WebAppResourceLoader webAppResourceLoader =
                    new WebAppResourceLoader(patternResolver.getResource("classpath:/").getFile().getPath());//设置beetl根路径
            beetlGroupUtilConfiguration.setResourceLoader(webAppResourceLoader);
        } catch (IOException e) {
            e.printStackTrace();
        }
        //读取配置文件信息
        return beetlGroupUtilConfiguration;
    }

    @Bean
    public BeetlSpringViewResolver beetlViewResolver() {
        BeetlSpringViewResolver beetlSpringViewResolver = new BeetlSpringViewResolver();
        beetlSpringViewResolver.setPrefix("templates/");//设置beetl文件的路径为：resources/templates
//        beetlSpringViewResolver.setSuffix(".btl");//设置beetl的后缀设置为btl
        beetlSpringViewResolver.setSuffix(".html");//设置beetl的后缀设置为btl
        beetlSpringViewResolver.setContentType("text/html;charset=UTF-8");
        beetlSpringViewResolver.setOrder(0);
        beetlSpringViewResolver.setConfig(beetlConfiguration());
        return beetlSpringViewResolver;
    }
}
```

pom依赖
```
        <dependency>
            <groupId>com.ibeetl</groupId>
            <artifactId>beetl</artifactId>
            <version>2.9.3</version>
        </dependency>
```
配置就完成了，测试controller
```
@Controller
@RequestMapping("/hello")
public class HelloController {

    @RequestMapping("/hello_beetl")
    public ModelAndView  hello(){
        ModelAndView modelAndView = new ModelAndView();
        modelAndView.addObject("username","jeffrey");
        modelAndView.setViewName("hello");
        return modelAndView;
    }
}
```
测试的html在templates下
hello.html
```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<p>获取后台返回的数据--->${username}</p>
<p>获取项目的context-path-->${ctxPath}</p>
</body>
</html>
```
