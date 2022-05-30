# @Provides Methods

被@Provides注解的方法（静态方法或实例方法）必须写在继承了AbstractModule的类里面，而且这个方法的返回值就是将被放进Guice map里的Provider中被绑定的bean(`bind()`)，至于具体的实现类是啥(`to()`)，就看方法里是怎么写的了。

```
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    ...
  }

  @Provides
  static TransactionLog provideTransactionLog() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setJdbcUrl("jdbc:mysql://localhost/pizza");
    transactionLog.setThreadPoolSize(30);
    return transactionLog;
  }
}
```

## 1 配合Binding Annotation使用

如果`@ providers`注解的方法有`@PayPal`或`@Named(“Checkout”)`这样的绑定注释，`Guice`就会绑定带注释的类型。依赖项可以作为参数传递给方法。在调用方法之前，注入器会对每一个变量执行绑定。

```
@Provides 
@PayPal
CreditCardProcessor providePayPalCreditCardProcessor(@Named("PayPal API key") String apiKey) {
    PayPalCreditCardProcessor processor = new PayPalCreditCardProcessor();
    processor.setApiKey(apiKey);
    return processor;
}
```

## 2 异常

`Guice`不允许从`Providers`中抛异常，被`@Provides`注解的方法抛出的异常会被包含进`ProvisionException`，如果你真的想抛自定义的异常，去看一下[ThrowingProviders extension](https://github.com/google/guice/wiki/ThrowingProviders)中的`@CheckedProvides`