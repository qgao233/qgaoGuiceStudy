# Built-in Bindings

本节介绍内置building。

但除了`Provider`这种内置building之外，其它的内置building应该避免使用。

除了`显式绑定`和`即时绑定`外(Alongside explicit and [just-in-time bindings](https://github.com/google/guice/wiki/JustInTimeBindings))，注入器还会自动包含其他绑定。

只有`Injector`才能创建这些绑定，如果自己尝试绑定它们将会出错。

## 1 Loggers

Guice自带了一个`java.util.logging.Logger`的内置绑定，目的是保存一些样板文件，用户可以直接使用。绑定会自动将`logger`的名称设置为要注入`logger`的类的名称。

```
@Singleton
public class ConsoleTransactionLog implements TransactionLog {

  private final Logger logger;

  @Inject
  public ConsoleTransactionLog(Logger logger) {
    this.logger = logger;
  }

  public void logConnectException(UnreachableException e) {
    /* the message is logged to the "ConsoleTransacitonLog" logger */
    logger.warning("Connect exception failed, " + e.getMessage());
  }
```

## 2 The Injector

在框架代码中，有时直到运行时你才知道你需要的类型。在这种罕见的情况下，您应该注入注入器。注入注入器的代码不会对其依赖项进行自我文档化(self-document)，因此应该少用这种方法。

## 3 Providers

For every type Guice knows about, it can also inject a Provider of that type. [Injecting Providers](https://github.com/google/guice/wiki/InjectingProviders) describes this in detail.

## 4 TypeLiterals

If you're injecting parameterized types, you can inject a TypeLiteral<T> to reflectively tell you the element type.

## 5 The Stage

Guice supports a stage `enum` to differentiate between development and production runs.

## 6 MembersInjectors

在绑定到提供程序或编写扩展时，您可能希望Guice将依赖项注入到您自己构造的对象中。为此，添加一个`MembersInjector<T>`的依赖(其中`T`是你的对象的类型)，然后调用`MembersInjector.injectmembers(myNewObject)`。当调用injectMembers时，Guice将对myNewObject执行字段和方法注入([field and method injection](https://github.com/google/guice/wiki/Injections))。