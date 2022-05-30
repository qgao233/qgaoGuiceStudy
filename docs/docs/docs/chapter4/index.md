# bean的重用(scope)

默认，`Guice`是返回的`bean`采用原型模式，但可以在以下不同级别的生命周期通过`scope`配置bean的`reuse`模式：

* application
* session
* request

## 1 内置的(built-in) scopes

### 1.1 单例(singleton)

可以使用内置的`@Singleton`注解让程序运行中，始终只能得到一个实例。

对于该注解，Guice支持2种包下的：

* java标准库：javax.inject.Singleton
* Guice提供的：com.google.inject.Singleton

不过使用java标准库的更好，因为该包下的注解同样被其它像`Dagger`一样的**注入框架**支持。

### 1.2 RequestScoped

Guice提供的servlet扩展：[servlet extension](https://github.com/google/guice/wiki/Servlets)支持一些额外的有关web的注解，如`@RequestScoped`

## 2 使用 scope

### 2.1 注解

Guice使用注解来标明scope，只需要把有关scope的注解：

* 写到接口的实现类上

除功能性外(As well as being functional)，该注释还用作文档。例如，@singleton表示该类旨在为线程安全。

```
@Singleton
public class InMemoryTransactionLog implements TransactionLog {
  /* everything here should be threadsafe! */
}
```

* 或者写在被@Provides注解过的方法上

```
@Provides 
@Singleton
TransactionLog provideTransactionLog() {
    ...
}
```

### 2.2 bind

```
bind(TransactionLog.class).to(InMemoryTransactionLog.class).in(Singleton.class);
```

* `bind()`：指定`Guice map`中的`Provider`
* `to()`：指定`bind()`中引用的实现类
* `in()`：(这句话建议读原文)接受either a scoping annotation like RequestScoped.class and also Scope instances like ServletScopes.REQUEST:
  ```
  in(RequestScoped.class);
  in(ServletScopes.REQUEST);
  ```

### 2.3 优先级

`bind`方法总是最优先的，如果使用注解写的`scope`不是想要的，可以直接使用`bind`方法绑定为`Scopes.NO_SCOPE`。

### 2.4 接口与实现类

如果1个类同时实现了2个接口，如下：

```
public class Applebees implements Bar,Grill {}
```

然后在bind时写成如下代码：

```
bind(Bar.class).to(Applebees.class).in(Singleton.class);
bind(Grill.class).to(Applebees.class).in(Singleton.class);
```

这样写会生成两个单例的实例，一个`Bar`的，一个`Grill`的，但2个实例其实都是`Applebees`，因为`Guice`绑定的`bean`是源类，即`bind()`里面的参数，而不是`to()`里面的参数。

如果只想创建一个实例，把`@Singleton`写在实现类`Applebees`上，或者重新`bind`：

```
bind(Applebees.class).in(Singleton.class);
```

这样上面的两个绑定的就没必要了，建议删掉。

## 3 饿汉 singleton

默认情况下，`Guice`使用懒汉加载单例，但也支持饿汉(`Eager`)单例：

```
bind(TransactionLog.class).to(InMemoryTransactionLog.class).asEagerSingleton();
```

使用枚举变量Stage来区分使用策略：

方法\Stage|PRODUCTION|DEVELOPMENT
-|-|-
.asEagerSingleton()|eager|eager
.in(Singleton.class)|eager|lazy
.in(Scopes.SINGLETON)|eager|lazy
@Singleton|eager*|lazy

## 4 线程安全

被以下注解注解过的类必须是线程安全的：

* `@Singleton`
* `@SessionScoped`

而`@RequestScoped`注解过的类不必追求线程安全。

通常被`@Singleton`和`@SessionScoped`注解的类中如果依赖`@RequestScoped`注解过的类是错误的。

## 5 测试

如果想测试在使用了`scope`的`Guice`环境下的方法，却不关心具体的`scope`，可以使用`Scopes.NO_SCOPE`（没有`scope`就意味着使用默认的，即每次请求都是新的实例）

```
import com.google.inject.Scopes;

@RunWith(Junit4.class)
final class FooTest {
  static class TestModule extends AbstractModule {
    @Override
    protected void configure() {
      bindScope(BatchScoped.class, Scopes.NO_SCOPE);
    }
  }
}
```