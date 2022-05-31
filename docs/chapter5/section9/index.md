# Just-in-time Bindings

这个是被Guice自动进行绑定的一个操作。

* 显式绑定：声明在Module的binding
* 隐式绑定：或称为Just-in-time(JIT) Bindings，即没有声明在Module中的binding，当用到时，Guice将尝试创建需要用到的binding

## 1 @Inject Constructors

**可注入**的构造方法有如下两类：

* 被@Inject显式注解过的构造方法
* 或者虽然没有被@Inject显式注解过，但是无参构造方法，且类和构造方法必须都是public的，因为反射是有成本的。

如：

```
public final class Foo {
  // An @Inject annotated constructor.
  @Inject
  Foo(Bar bar) {
    ...
  }
}

public final class Bar {
  // A no-arg non private constructor.
  Bar() {}

  private static class Baz {
    // A private constructor to a private class is also usable by Guice, but
    // this is not recommended since it can be slow.
    private Baz() {}
  }
}
```

**不能被注入**的构造方法：

* 被有被@Inject显示注解的构造方法中还有参数
* 一个类中有多个构造方法被@Inject注解
* 构造方法被定义在普通内部类里：内部类对其外围类有一个隐式引用，不能注入。

如：

```
public final class Foo {
  // Not injectable because the construct takes an argument and there is no
  // @Inject annotation.
  Foo(Bar bar) {
    ...
  }
}

public final class Bar {
  // Not injectable because the constructor is private
  private Bar() {}

  class Baz {
    // Not injectable because Baz is not a static inner class
    Baz() {}
  }
}
```

应用程序可以通过在`Module`中调用`binder().requireAtInjectRequired()`来强制`Guice`只使用`@Inject`注解的构造函数。当选择注入时，`Guice`将只考虑`@Inject`注解的构造函数，如果没有，则报告`MISSING_CONSTRUCTOR`。

tip: 使用`Modules.requireAtInjectOnConstructorsModule()`来在构造函数需求中为`@Inject`选择。

## 2 @ImplementedBy

该注解是告诉一个类的默认实现类是啥，和[Linked Bindings](../section1/index.md)差不多。

```
@ImplementedBy(PayPalCreditCardProcessor.class)
public interface CreditCardProcessor {
  ChargeResult charge(String amount, CreditCard creditCard)
      throws UnreachableException;
}
```

和下面这个一样：

```
bind(CreditCardProcessor.class).to(PayPalCreditCardProcessor.class);
```

但绑定的type如果两种方法都用了，以`bind()`优先。

由于`@ImplementedBy`是编译时的依赖，因此会被运行时的`bind()`给覆盖。

## 3 @ProvidedBy

`@ProvidedBy`告诉`Injector`可以创建实例的`Provider`。

```
@ProvidedBy(DatabaseTransactionLogProvider.class)
public interface TransactionLog {
  void logConnectException(UnreachableException e);
  void logChargeResult(ChargeResult result);
}
```

等价于：

```
bind(TransactionLog.class).toProvider(DatabaseTransactionLogProvider.class);
```

和`@ImplementedBy`一样，上面两种情况如果都用了，以bind()为准。

## 4 Enforce Explicit Bindings

为了禁止隐式绑定，可以使用`requireExplicitBindings`：

```
final class ExplicitBindingModule extends AbstractModule {
  @Override
  protected void configure() {
    binder().requireExplicitBindings();
  }
}
```

使用了之后，就必须得将所有需要注入的binding写在Module里。