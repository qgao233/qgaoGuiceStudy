# 框架模型(mental modal)

* buzzword：梗
* jargon：行话，术语

本章主要介绍`Guice`的实现模型，让我们更容易去思考这个框架的工作原理。

## 1 Guice就是个map

和`spring`中的`ioc`容器很像，`Guice`就是个`map`，其中的每一个`entry`包含2个部分：

* Guice key: map的key，存的是Guice定义的一个Key类
* Provider: map的value，存的是Guice定义的一个Provider类，里面定义了如何返回我们想要的bean实例

### 1.1 Guice key

上一章[快速上手](../chapter2/index.md)中的`Greeter`类在构造方法中声明的2个依赖，在`Guice map`中的`key`是这样表示的：

* @Message String：map中为 `Key<String>`
* @Count int：map中为 `Key<Integer>`

`java`中表示一个类型的`key`最简单的表达形式为：

```
// 标识一个String类实例的依赖
Key<String> databaseKey = Key.get(String.class);
```

但是程序总会有相同类型的依赖：

```
final class MultilingualGreeter {
  private String englishGreeting;
  private String spanishGreeting;

  MultilingualGreeter(String englishGreeting, String spanishGreeting) {
    this.englishGreeting = englishGreeting;
    this.spanishGreeting = spanishGreeting;
  }
}
```

因此`Guice`使用[绑定注解(bind annotations)]()来区分相同类型的依赖，使之可以区分开是不同的依赖：

```
@Inject
MultilingualGreeter(@English String englishGreeting, @Spanish String spanishGreeting) {
    this.englishGreeting = englishGreeting;
    this.spanishGreeting = spanishGreeting;
}
```

用了**绑定注解**后的`key`可以这样取：

```
Key<String> englishGreetingKey = Key.get(String.class, English.class);
Key<String> spanishGreetingKey = Key.get(String.class, Spanish.class);
```

最后，当程序调用`Greeter`的构造方法想获得两个参数的依赖时，实际上`Guice`内部帮我们做了这样的事：

```
String english = injector.getInstance(Key.get(String.class, English.class));
String spanish = injector.getInstance(Key.get(String.class, Spanish.class));
MultilingualGreeter greeter = new MultilingualGreeter(english, spanish);
```

总结：`Guice Key`就是一个用来标明依赖，可以选择使用绑定注解搭配使用的一个类。


### 1.2 Guice Providers

这个value用来存bean的，

`Provider`是个接口，声明了一个`get`方法:

interface Provider<T> {
  /** 提供T类的实例 **/
  T get();
}

每个实例了该接口的类都可以自定义你想要返回的类实例，可以使用：

* 直接new
* 用其它方法构造，比如反射
* 或者已经存进缓存中，直接反序列化等。

大多数程序不会选择直接实现该接口，而是去继承AbstractModule，通过configure来配置Guice的injector，injector内部会通过我们配置过的bean，自动去实现Provider。

比如上一章[快速上手](../chapter2/index.md)中，

```
class DemoModule extends AbstractModule {
  @Provides
  @Count
  static Integer provideCount() {
    return 3;
  }

  @Provides
  @Message
  static String provideMessage() {
    return "hello world";
  }
}
```

* Provider<String>：接口的get中调用provideMessage方法，返回"hello world"
* Provider<Integer>：接口的get中调用provideCount方法，返回3

## 2 使用 Guice

要想使用Guice你得知道2个东西：

1. Configuration：我们的程序需要往`Guice map`里塞东西
2. Injection：我们的程序叫Guice从它的map里创建和获取我们想要的bean实例

### 2.1 Configuration

怎么给Guice的配置塞东西，继而填充到Guice map里面？

用Guice modules (或者[Just-In-Time bindings]())，通常有2种方式：

1. 使用像`@Provides`这样的注解
2. 全用`Guice`的`Domain Specific Language` (DSL)

概念上(Conceptually)，Guice的API提供了一些简单的方法来操作(manipulate)Guice map，并且直截了当，下面是java 8的语法写的一些例子，简单明了：

Guice DSL syntax（我们程序中的写法）|Mental model（Guice内部做的事）
-|-
bind(key).toInstance(value)|map.put(key, () -> value)(instance binding)
bind(key).toProvider(provider)|map.put(key, provider)(provider binding)
bind(key).to(anotherKey)|map.put(key, map.get(anotherKey))(linked binding)
@Provides Foo provideFoo() {...}|map.put(Key.get(Foo.class), module::provideFoo)(provider method binding)

### 2.2 Injection

依赖注入中最重要的事情就是，你只需要声明你需要这些依赖，自然会由容器返回给你想要的，而不是脱离容器的使用，自己从外面其它方式获得你想要的bean。

像下面这样：

```
class Foo {
  private Database database;

  @Inject
  Foo(Database database) {  // We need a database, from somewhere
    this.database = database;
  }
}
@Provides
Database provideDatabase(
    // We need the @DatabasePath String before we can construct a Database
    @DatabasePath String databasePath) {
  return new Database(databasePath);
}
```

## 3 依赖树状图

>When injecting a thing that has dependencies of its own, Guice recursively injects the dependencies.

当注入A时，发现A还依赖B，然后去注入B，结果B又依赖C，接着就去注入C，这样dfs递归地向上遍历(up through)去注入会形成依赖的一条线的树状图。

为了使用Injector，Guice会去检查这个依赖树，如果出问题了会抛CreationException。