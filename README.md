and-load-aot
===============

- Load data ahead of time for Android
- 提前为页面加载数据



About
-----

要解决的问题：
- 在打开页面Activity时，一般的流程是这样的:
    - 1、通知AMS去创建新的Activity
    - 2、Activity创建完后初始化布局View 
    - 3、然后去网络中或者数据库中异步请求数据
    - 4、数据准备好后通知渲染到View上
    
- 上面的流程一般是串行的，即要等到Activity准备好后再去请求数据，而准备Activity的过程往往是耗时的过程（例如启动Activity涉及到跨进程、遍历创建View树都是耗时的过程），为什么不把这个过程改为并行的呢？甚至改为提前进行呢？

- 怎样优雅地把创建页面和请求数据并行进行，同时又不改变以前数据请求的调用方式呢？and-load-aot提供了一种思路



Usage
-----

详细见[ExampleActivity](https://github.com/heimashi/and-load-aot/blob/master/example/src/main/java/com/sw/aot/example/ExampleActivity.java)

- 在项目的build.gradle中添加api依赖库以及编译时的注解处理器
```groovy
    annotationProcessor 'com.sw.aot.load:load-aot-compiler:1.0.0'
    implementation 'com.sw.aot.load:load-aot-annotation:1.0.0'
    implementation 'com.sw.aot.load:load-aot-api:1.0.0'
```

- 添加注解处理器的配置信息AOT_INDEX，会在编译器生成该类ExampleAotIndex.java，该类里面含有加载方法的路由信息
```groovy
defaultConfig {
    javaCompileOptions {
        annotationProcessorOptions {
             arguments = [AOT_INDEX: 'com.sw.aot.example.ExampleAotIndex']
        }
    }
}
```
    
- 在加载数据的方法上加上@AOTLoad注解，注解的参数router代表着该方法的路由，后面会通过这个路由来调用该方法。    
```java
    @AOTLoad(router = "/Example/LoadMockData")
    public ResultData<String> loadMockData(){
        final ResultData<String> result = new ResultData<String>();
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(2000);
                    result.setCode(0);
                    result.setData("MOCK: LOAD DATA SUCCESS");
                    result.flush();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    result.setCode(-1);
                    result.flush();
                }
            }
        }).start();
        return result;
    }
```

- 编译Build一下项目，就会生成含有上面注解信息的类ExampleAotIndex.java，例如：
```java
/** This class is generated by AOTLoad, do not edit. */
public class ExampleAotIndex implements AotRouterInterface{

    /* mock load async data */
    public static String EXAMPLE_LOADMOCKDATA = "/Example/LoadMockData";

    private final HashMap<String, String> routerMethodMap = new HashMap<String, String>();
    private final HashMap<String, Class<?>> routerClassMap = new HashMap<String, Class<?>>();

    public ExampleAotIndex() {
         routerMethodMap.put(EXAMPLE_LOADMOCKDATA, "loadMockData");
         routerClassMap.put(EXAMPLE_LOADMOCKDATA, com.sw.aot.example.ExampleActivity.class );
    }

    @Override
    public HashMap<String, String> getMethodMap() {
        return routerMethodMap;
    }

    @Override
    public HashMap<String, Class<?>> getClassMap() {
        return routerClassMap;
    }

}
```


- 在应用启动后注入路由表ExampleAotIndex，一般在Application中：
```java
AotLoader.enableLog(true);//是否开始日志
AotLoader.addRouter(new ExampleAotIndex());//注入加载方法的路由表
```


- 现在就可以根据方法路由来提前执行加载任务的方法了，该框架将加载数据的方法抽象为生产和消费的task，是一个典型的生产者消费者模型。
当打开Activity前去生产加载数据的task，执行AotLoader.produce(methodRouter），会根据传人的方法路由名来定位到申明了该注解路由的方法，然后反射调用执行，AotLoader.produce
(methodRouter）的返回值是该任务的ID，将ID以参数的形式传给Activity
```java
public static void invoke(Context context){
    Intent intent = new Intent(context, ExampleActivity.class);
    intent.putExtra(START_AOT_LOAD_ID, AotLoader.produce(ExampleAotIndex.EXAMPLE_LOADMOCKDATA));
    context.startActivity(intent);
}
```

- Activity初始化完成并且View准备好以后，就可以根据传递过来的任务的ID来消费提前加载的数据了：
```java
    aotTaskId = getIntent().getStringExtra(START_AOT_LOAD_ID);
    if(AotLoader.isValidTask(aotTaskId)){
        AotLoader.consume(aotTaskId, listener);
    }
```

