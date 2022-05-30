# Binding Annotations

## 1 含义

说人话就是，有时候可能想要给一个接口注入一个实现类，但是这个接口有两个实现类，但这两个实现类都被放进了容器里，此时就不知道该这个接口注入哪一个实现类了。

在[Guice 框架模型(mental modal)](../../chapter3/index.md)中也提到过这个问题，只需要配合`@Qualifer`或者`@BindingAnnotation`(java元注解)使用即可，并且应该用在`Guice map`中的key为`Key`的对象里。

如下所示例子：

```
package example.pizza;

import static java.lang.annotation.ElementType.PARAMETER;
import static java.lang.annotation.ElementType.FIELD;
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import java.lang.annotation.Target;
import java.lang.annotation.Retention;
import javax.inject.Qualifier;

@Qualifier
@Target({ FIELD, PARAMETER, METHOD })
@Retention(RUNTIME)
public @interface PayPal {}

// Older code may still use Guice `BindingAnnotation` in place of the standard
// `@Qualifier` annotation. New code should use `@Qualifier` instead.
@BindingAnnotation
@Target({ FIELD, PARAMETER, METHOD })
@Retention(RUNTIME)
public @interface GoogleCheckout {}
```

这些代码可以像[Guice 快速上手](../../chapter2/index.md)写在单独一个java文件里，也可以直接写在使用这个注解的类里。

两种方式创建`bindings`，`bind`或`@Provide`：

```
final class CreditCardProcessorModule extends AbstractModule {
  @Override
  protected void configure() {
    // This uses the optional `annotatedWith` clause in the `bind()` statement
    bind(CreditCardProcessor.class)
        .annotatedWith(PayPal.class)
        .to(PayPalCreditCardProcessor.class);
  }

  // This uses binding annotation with a @Provides method
  @Provides
  @GoogleCheckout
  CreditCardProcessor provideCheckoutProcessor(
      CheckoutCreditCardProcessor processor) {
    return processor;
  }
}
```

然后像这样使用：

```
@Inject
public RealBillingService(@PayPal CreditCardProcessor processor,TransactionLog transactionLog) {
    ...
}
//或者
@Inject
public RealBillingService(@GoogleCheckout CreditCardProcessor processor,TransactionLog transactionLog) {
    ...
}
```

## 2 @Named

除了上面的方式，还可以使用`@Named`，它有一个String参数，通过指定具体的String，可以唯一标明一个实例：

```
final class CreditCardProcessorModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(CreditCardProcessor.class)
      .annotatedWith(Names.named("Checkout"))   //只要在依赖时，有注解@Named("Checkout")
      .to(CheckoutCreditCardProcessor.class);   //就给注入CheckoutCreditCardProcessor
  }
}
```

依赖注入时：

```
@Inject
public RealBillingService(@Named("Checkout") CreditCardProcessor processor,
    TransactionLog transactionLog) {
    ...
}
```

但是不推荐这种@Named方式(sparingly保守使用)，因为编译器无法check string，最好还是像第1小节一样自定义自己的注解来实现唯一标识。