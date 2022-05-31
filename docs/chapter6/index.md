# Guice 注入原理

依赖注入模式将行为与依赖解析分离开来。该模式建议传入依赖关系，而不是直接查找依赖关系或从工厂查找依赖关系。将依赖项设置到对象中的过程称为注入。

## 1 Constructor Injection

构造仪注入将实例化与注入相结合。要使用它，请用@Inject注释注释构造函数。该构造函数应接受类依赖性作为参数。然后，大多数构造函数将把参数分配给最终字段。

```
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processorProvider;
  private final TransactionLog transactionLogProvider;

  @Inject
  RealBillingService(CreditCardProcessor processorProvider,
      TransactionLog transactionLogProvider) {
    this.processorProvider = processorProvider;
    this.transactionLogProvider = transactionLogProvider;
  }
```

如果类没有带`@inject`注解的构造函数，Guice将使用一个公共的无参数构造函数(如果它存在的话)。

最好使用注解，它记录了该类型参与了依赖注入。

构造函数注入与单元测试配合得很好。如果您的类在一个构造函数中接受它的所有依赖项，您就不会意外地忘记设置依赖项。当引入一个新的依赖项时，所有的调用代码都会很方便地中断!修复编译错误，您就可以确信一切都已正确连接。

## 2 Method Injection

方法注入意味着被注入的类是可变的(mutable)，这通常应该避免。所以最好**选择构造函数注入**而不是方法注入。

Guice可以注入具有@Inject注释的方法。依赖采用参数的形式，注入器在调用方法之前解析这些参数。注入的方法可以有任意数量的参数，方法名不会影响注入。

```
public class PayPalCreditCardProcessor implements CreditCardProcessor {

  private static final String DEFAULT_API_KEY = "development-use-only";

  private String apiKey = DEFAULT_API_KEY;

  @Inject
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```

## 3 Field Injection

与方法注入相同，字段注入意味着被注入的类是可变的，这通常应该避免。所以最好选择构造函数注入，而不是字段注入。

Guice用@Inject注释注入字段。这是最简洁的注入，但最不可测试。

```
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  @Inject Connection connection;

  public TransactionLog get() {
    return new DatabaseTransactionLog(connection);
  }
}
```

Avoid using field injection with `final` fields, which has [weak semantics](http://java.sun.com/javase/6/docs/api/java/lang/reflect/Field.html#set(java.lang.Object,%20java.lang.Object)).

## 4 Optional Injections

tip: 首选[OptionalBinder](../chapter5/section10/index.md)而不是`@Inject(optional = true)`。

有时，在存在依赖项时使用依赖项，在不存在依赖项时回退到默认依赖项会很方便。

方法和字段注入可能是可选的，这将导致Guice在依赖项不可用时静默地忽略它们。要使用可选注入，请应用`@Inject(optional=true)`注解

```
public class PayPalCreditCardProcessor implements CreditCardProcessor {
  private static final String SANDBOX_API_KEY = "development-use-only";

  private String apiKey = SANDBOX_API_KEY;

  @Inject(optional=true)
  public void setApiKey(@Named("PayPal API key") String apiKey) {
    this.apiKey = apiKey;
  }
```

混合可选的注入和即时绑定可能会产生令人惊讶的结果。例如，即使没有显式绑定Date，也始终会注入以下字段。这是因为Date有一个公共的无参数构造函数，适合即时绑定。

```
@Inject(optional=true) Date launchDate;
```

## 5 On-demand Injection

方法和字段注入可用于初始化现有实例。你可以使用`Injector.injectMembers`API

```
public static void main(String[] args) {
    Injector injector = Guice.createInjector(...);

    CreditCardProcessor creditCardProcessor = new PayPalCreditCardProcessor();
    injector.injectMembers(creditCardProcessor);
```

## 6 Static Injections

在将应用程序从静态工厂迁移到Guice时，可以进行增量更改。

静态注入在这里是一个很有用的工具。通过获得对注入类型的访问，而无需自身被注入，对象可以部分参与依赖项注入。在模块中使用`requestStaticInjection()`来指定在创建注入器时要注入的类

```
@Override public void configure() {
    requestStaticInjection(ProcessorFactory.class);
    ...
}
```

Guice将注入具有@Inject注释的类的静态成员

```
class ProcessorFactory {
  @Inject static Provider<Processor> processorProvider;

  /**
   * @deprecated prefer to inject your processor instead.
   */
  @Deprecated
  public static Processor getInstance() {
    return processorProvider.get();
  }
}
```

静态成员不会在实例注入时注入。

不建议将此API用于一般用途，因为它存在许多与静态工厂相同的问题:测试起来很笨拙，使依赖关系不透明，并且依赖全局状态。

## 7 Automatic Injection

Guice会自动对以下类型的对象执行字段和方法注入:

* 在绑定语句中传递给toInstance()的实例
* 在绑定语句中传递给toProvider()的实例。

这些注入会作为创建注入器的一部分执行。

## 8 Injection Points

An injection point is a place in the code where Guice has been asked to inject a dependency.

Example injection points:

* parameters of an injectable constructor
* parameters of a @Provides method
* parameters of an @Inject annotated method
* fields annotated with @Inject