# Constructor Bindings

有时候会碰到一种情况，就是你想绑定给某个类一个具体的实现类，但你无法做到，因为你无法将`@Inject`写到实现类的构造方法上，比如你想将第三方jar包中的类，或者一个类有多个构造方法都被`@Inject`注解，然后导入`Guice map`管理，[@Provides Methods](../section4/index.md)可以解决这个问题，

但是存在一个问题，即通过`Provider`的方法去`new`一个第三方的类，这样不能被切面`AOP`注入。

Guice有一个`toConstructor()`，其中可以通过反射来找到实现类的构造方法，如果找不到则处理一个异常即可：

```
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    try {
      bind(TransactionLog.class).toConstructor(
          DatabaseTransactionLog.class.getConstructor(DatabaseConnection.class));
    } catch (NoSuchMethodException e) {
      addError(e);
    }
  }
}
```

在此示例中，`DataBaseTransactionLog`必须具有一个构造函数，该构造函数采用单个`databaseconnection`参数。该构造函数不需要`@Inject`注释。`Guice`将调用该构造函数以满足绑定。

每个`ToConstructor()`绑定都是独立范围范围的。如果创建针对同一构造函数的多个单例绑定，则每个绑定都会产生其自己的实例。