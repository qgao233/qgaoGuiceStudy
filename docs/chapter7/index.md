# Guice AOP

Aspect Oriented Programming

使用Guice拦截方法。

为了补充依赖注入，Guice支持方法拦截。此特性使您能够编写每次调用匹配方法时执行的代码。

它适用于横切关注点(“方面”)，例如事务、安全性和日志。因为拦截器将问题划分为方面而不是对象，所以它们的使用被称为**面向切面编程**(AOP)。

大多数开发人员不会直接编写方法拦截器；但是他们可能会看到在像 [Warp Persist](http://www.wideplay.com/guicewebextensions2)这样的集成库中的使用时，确实需要选择匹配方法，创建拦截器并将其全部配置在`Module`中。

`Matcher`是一个简单的接口，它可以接受或拒绝一个值。对于`Guice AOP`，需要两个匹配器:

* 一个定义参与的类，
* 另一个定义这些类的方法。

为了简单起见，有一个工厂类来满足常见的场景。

每当调用匹配的方法时，都会执行`MethodInterceptors`。

* 他们有机会检查调用:方法、参数和接收实例。
* 它们可以执行横切逻辑，然后委托给底层方法。
* 它们可以检查返回值或异常并返回。

由于拦截器可以应用于许多方法，并将接收许多调用，它们的实现应该是有效的和不受干扰的。

## 1 Example: Forbidding method calls on weekends

为了说明方法拦截器如何使用Guice，我们将禁止在周末调用披萨账单系统。送餐员只在周一到周五工作，所以我们会防止在送不出去的时候有人点披萨!这个示例在结构上类似于使用AOP进行授权。

为了将选择方法标记为仅限工作日，我们定义了一个注解：

```
@Retention(RetentionPolicy.RUNTIME) 
@Target(ElementType.METHOD)
@interface NotOnWeekends {}
```

把它应用到需要被拦截的方法上：

```
public class RealBillingService implements BillingService {

  @NotOnWeekends
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    ...
  }
}
```

接下来，我们通过实现`org.aopalliance.intercept.MethodInterceptor`接口来定义拦截器。当需要通过底层方法进行调用时，可以调用`invoke.proceed()`

```
public class WeekendBlocker implements MethodInterceptor {
  public Object invoke(MethodInvocation invocation) throws Throwable {
    Calendar today = new GregorianCalendar();
    if (today.getDisplayName(DAY_OF_WEEK, LONG, ENGLISH).startsWith("S")) {
      throw new IllegalStateException(
          invocation.getMethod().getName() + " not allowed on weekends!");
    }
    return invocation.proceed();
  }
}
```

最后，我们配置一切。那就是我们为要拦截的类和方法创建匹配器的地方。

在本例中，我们匹配任何类，但只匹配带有`@NotOnWeekends`注释的方法：

```
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    bindInterceptor(Matchers.any(), Matchers.annotatedWith(NotOnWeekends.class),
        new WeekendBlocker());
  }
}
```

把这些配置好后，然后等到周六时，如果有人下单，可以看到该方法被拦截，订单被拒绝：

```
Exception in thread "main" java.lang.IllegalStateException: chargeOrder not allowed on weekends!
  at com.publicobject.pizza.WeekendBlocker.invoke(WeekendBlocker.java:65)
  at com.google.inject.internal.InterceptorStackCallback.intercept(...)
  at com.publicobject.pizza.RealBillingService$$EnhancerByGuice$$49ed77ce.chargeOrder(<generated>)
  at com.publicobject.pizza.WeekendExample.main(WeekendExample.java:47)
```

## 2 Limitations

在底层，方法拦截是通过在运行时生成字节码来实现的。

Guice通过重写方法动态创建一个子类，该子类应用拦截器。如果您所处的平台不支持字节码生成(比如Android)，那么应该使用不支持AOP的Guice。

这种方法对可以拦截的类和方法施加了限制：

* 类必须是`public`或`package-private`。
* 类必须是非`final`的方法必须是`public`，
* `package-private`或`protected`方法必须是非`final`的。
* 实例必须由`Guice`通过带有`@inject`注解的或无参数的构造函数创建。
* 不可能对不是由`Guice`构造的实例使用方法拦截。

## 3 Injecting Interceptors

如果你需要将依赖项注入拦截器，可以使用`requestInjection` API。

```
public class NotOnWeekendsModule extends AbstractModule {
  protected void configure() {
    WeekendBlocker weekendBlocker = new WeekendBlocker();
    requestInjection(weekendBlocker);
    bindInterceptor(Matchers.any(), Matchers.annotatedWith(NotOnWeekends.class),
       weekendBlocker);
  }
}
```