# Ferryman Android页面路由框架
主要解决项目初具规模后，页面跳转，传参，页面路由等功能代码十分冗余且难以管理的问题。  
主要功能：

1. Android 端页面路由，与 web 页面路由统一，非常便捷的由 web 跳转 activity 页面并携带参数
2. 使用自动生成的函数进行 Activity 跳转代码，将页面所需数据作为了函数参数。
3. Activity 返回监听功能，不再需要重写 `onActivityResult` 方法，还能自动装箱返回数据并返回。

支持Kotlin, 支持在 Library 中使用以及模块化场景。  
全库没有一个反射，纯依靠 APT 实现。  
使用简洁直观的代码处理页面跳转：
```java
Ferryman.from(MainActivity.this)
        .gotoNameInputActivity(name)
        .onResultWithData(new OnDataResultListener<NameInputActivityResult>() {
            @Override
            public void fullResult(@NonNull NameInputActivityResult data) {
                name = data.getName();
                tvName.setText(data.getName());
            }

            @Override
            public void emptyResult() {
            }

        });
```  
以及使用URL跳转
```java
RouterDriver.startActivity(this,"activity://phoneNumber?name=Lee&country=China");
```
以及兼容来自 Web 与其他应用的 Deeplink ( manifest里信息需自行填写 )
```java
Intent intent = new Intent();
intent.setAction("android.intent.action.VIEW");
Uri url = Uri.parse("activity://phoneNumber?name=Lee&country=China");
intent.setData(url);
startActivity(intent);
```
## Dependency

    compile 'com.jude:ferryman-core:1.4.6'
    annotationProcessor 'com.jude:ferryman-compiler:1.4.6'

## Usage

### 1. 页面路由
使用 `@Page` 注解标记Activity。  
默认将使用 Activity 包名作为 URL。  
可以填入页面 URL，进行 URL 路由。一个 Activity 可以有多个地址。一个地址只能对应一个 Activity。  
```java
@Page("activity://two")
public class ActivityTwo extends Activity{
}

//API 方式启动页面
Ferryman.from(ctx).gotoActivityTwo());

//Url 方式启动页面
RouterDriver.startActivity(this,"activity://two");
```
然后就可以使用上面2种优雅的 Activity 跳转方法了。

### 2. 页面参数
使用 `@Params` 注解标记参数。  
在 Activity 中可以直接使用 `Ferryman.inject(this);` 对参数数据拆箱并注入 Activity。    
```java
@Page("activity://phoneNumber")
public class NumberInputActivity extends AppCompatActivity {

    @Params String name;
    @Params String mCountry;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Ferryman.inject(this);
     }
}

//API 方式
Ferryman.from(MainActivity.this).gotoNumberInputActivity("Lee","China");

//Url方式 注意url的参数要经过Encode
RouterDriver.startActivity(this,"activity://phoneNumber?name=Lee&country=China");

```
#### Notice
+ 如果是在 Kotlin 中使用，参数还需要加上 `@JvmField` 注解。  
+ **注解参数支持 DeepLink**. 可以直接自己构造 DeepLink url进行跳转。api,router,deeplink 三合一
+ 可以在参数上随意增加注解，会自动应用到生成的API中，比如`@Nullable`,`@NotNull`,`@IdRes` 或者其他任何支持 `PARAMETER` 的自定义注解。

##### 2.1 页面参数进阶规则
```java
    @Params
    String name;  // 将以 name 为参数名创建 API
    
    // 指定参数名
    @Params("key")
    String name;  // 将以 key 为参数名创建 API
    
    // 可忽略
    @Params(ignore = true)
    String name;  // 将会生成2个API, 一个包含name, 一个不包含
    
    // 参数分组
    // 将生成2个API, 一个包含 name, count, 一个包含 age, color, count. 
    // 拥有同一个分组标识的参数，会被分在同一组，每一组参数会创建一个API
    @Params(group = "A")
    String name; 
    @Params(group = "B")
    String age;
    @Params(group = "B")
    String color;
    @Params(group = {"A","B"})
    int count;
    
    // 参数功能拓展 
    // 可以使用 name.has() 来判断此参数是否被设置值，
    // 可用于在 url 来源及 可空参数 场景下无法区分未设参数与传递默认值的问题 
    @Params
    Param<String> name; 
    
```

### 3. 页面返回数据
使用 `@Results` 注解标记返回数据。  
使用 `Ferryman.save(this);` 将参数装箱并保存进 Activity。  
```java
@Page
public class NameInputActivity extends AppCompatActivity {

    @Results String name;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Ferryman.inject(this);
    }

    public void submit(){
        name = etName.getText().toString();
        Ferryman.save(this);
        finish();
    }
}

//API 方式
Ferryman.from(MainActivity.this)
        .gotoNameInputActivity()
        .onResultWithData(new OnDataResultListener<NameInputActivityResult>() {
            @Override
            public void fullResult(@NonNull NameInputActivityResult data) {
                name = data.getName();
                tvName.setText(data.getName());
            }

            @Override
            public void emptyResult() {

            }

        });
```
#### Notice
+ 如果是在 Kotlin 中使用，参数还需要加上 `@JvmField` 注解。
+ 可以在参数上随意增加注解，会自动应用到生成的API中，比如`@Nullable`,`@NotNull`,`@IdRes` 或者其他任何支持 `PARAMETER` 的自定义注解。

##### 3.1 返回数据进阶规则
```java
    // 返回值功能拓展 
    // 使用 name.set(T t) 来设置值，可以在页面未设置返回值时，直接回调`emptyResult`而不是`fullResult`加上默认值。
    @Results
    Result<String> name; 
```

### 4. 页面数据注入抽取
参数及返回数据可以定义在非 Activity 类里，比如定义在 Presenter 中，只要此类**与 Activity 建立关联**。建立关联有2种方式：  
##### 4.1 通过 `@BindActivity` 注解直接关联。
```java
@BindActivity(ShopActivity.class)
public class ShopPresenter {
    @Params
    public String id;
}
```
##### 4.2 通过`@ActivityRelation` 使用正则匹配对类名进行匹配来批量关联。  
接口与方法名无所谓，只要加上了 `@ActivityRelation` 即可。  
对所有定义类与Activity的类名进行正则匹配，如果 `activityNameRegular` 匹配出的字符串与 `objectNameRegular` 匹配出的字符串相等，即为关联。
```java
public interface OneActivityRelations {

    // 示例：将 XxxxActivity 与 XxxxPresenter 关联起来。
    @ActivityRelation(activityNameRegular = "\\b\\w+(?=Activity)", objectNameRegular = "\\b\\w+(?=Presenter)")
    void presenter();

}
```
然后数据的拆箱装箱。  
```java
// 拆箱注入数据
Ferryman.injectFrom(activity).to(this);
// 装箱保存数据
Ferryman.saveFrom(this).to(activity);
```
##### 4.3 最佳实践
直接在BaseActivity进行操作, 省去在每个activity与数据类进行手动注入抽取。
```java
public abstract class BaseActivity{
    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Ferryman.inject(this);
        // Ferryman.injectFrom(this).to(presenter);
    }

    @Override
    public void finish() {
        Ferryman.save(this);
        // Ferryman.saveFrom(getPresenter()).to(this);
        super.finish();
    }
}
```

### 5. 自定义路由
允许自己处理未被绑定 Activity 的 url。返回 null 则表示不能处理这个 url。  
```java
FerrymanSetting.addRouterFactory(new Router.Factory() {
    @Override
    public Router createRouter(String url) {
        return null;
    }
});
```
如果可以处理这个 url，则返回处理 Router  
```java
public interface Router {
    Intent start(@NonNull Context context, @NonNull String url);
}
```
### 6. 自定义数据传递序列化
默认提供 Gson 实现的对象序列化，可以添加自定义序列化方式。返回 null 表示不能处理这个类型。  
```java
FerrymanSetting.addConverterFactory(Converter.Factory factory);
```

### 7. URL 拦截器
提供注册拦截器，对需要跳转的url进行一些批处理。
```java
// url 方式的跳转
FerrymanSetting.addUrlInterceptors(RouterInterceptor interceptor);
// api 方式的跳转
FerrymanSetting.addAPIInterceptors(RouterInterceptor interceptor);
```

## Library中使用
Ferryman 可以被使用在 Library 中，Library 中如上正常使用(需要添加 `annotationProcessor`)。
但在 app 与 Library 中都需要添加额外添加一个`ferryman-modular`插件来合并库中自带的路由。
```grovvy
buildscript {
    dependencies {
        classpath 'com.jude:ferryman-modular:1.4.6'
    }
}

apply plugin: 'com.jude.ferryman-modular'

```

### 页面管理
提供页面管理功能，单独一套 API .
```groovy
    compile 'com.jude:ferryman-record:1.4.6'
```

```java
// 初始化，在 Application 中设置
PageManager.init(Context ctx);

// 添加/删除 Application 前后台切换监听器
PageManager.addApplicationStateListener(ApplicationStateListener listener);
PageManager.removeApplicationStateListener(ApplicationStateListener listener);

// 取栈顶 Activity 
PageManager.getTopActivity();

// 取最上层指定类的 Activity 
PageManager.getTopActivity(Class<? extends Activity> activityClass)
PageManager.getTopActivity(String activityName)

// 取最上层指定类的 Activity 深度
PageManager.getDeep(Class<? extends Activity> activityClass)
PageManager.getDeep(String activityName)

// 关闭栈顶 Activity 直到展示指定类的 Activity
PageManager.closeToLastActivity(Class<? extends Activity> activityClass)
PageManager.closeToLastActivity(String activityName)

// 关闭所有 Activity
PageManager.clearAllStack();

// 返回 Activity 栈
PageManager.printPageStack();
```

License
-------

    Copyright 2017 Jude

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

