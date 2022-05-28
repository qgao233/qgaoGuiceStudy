# Guice 起因

将各个模块连接起来是比较枯燥(tedious)的，用一个使用信用卡支付账单的例子来讲解为何开发`Guice`框架。

## 1 传统的写法

### 1.1 BillingService

在支付账单的service里，**直接使用new的方式**去获取两个工具类来达到真正地**支付**的操作：

* PaypalCreditCardProcessor
* DatabaseTransactionLog

```
public class RealBillingService implements BillingService {
  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = new PaypalCreditCardProcessor();
    TransactionLog transactionLog = new DatabaseTransactionLog();

    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

### 1.2 Factories

感觉直接new两个工具类不太好，于是决定用工厂模式替代直接new的方式，并达到解耦(`decouple`)**service和工具类**的工作。

下面是根据工厂模式的模板(boilerplate)代码写的：

```
public class CreditCardProcessorFactory {

  private static CreditCardProcessor instance;

  public static void setInstance(CreditCardProcessor processor) {
    instance = processor;
  }

  public static CreditCardProcessor getInstance() {
    if (instance == null) {
      return new SquareCreditCardProcessor();
    }

    return instance;
  }
}
```

更新后的service：

```
CreditCardProcessor processor = CreditCardProcessorFactory.getInstance();
TransactionLog transactionLog = TransactionLogFactory.getInstance();
```

### 1.3 unit test

在测试启动关闭时将两个工具类设值或`Null`，但是这个代码很蠢，因为一个真正的业务`factory`居然保存了两个用来测试的工具类。

所以我们需要很谨慎地来设置`factory`里的值，一旦在测试时`teardown`方法执行失败，那么`factory`里的两个工具类就会影响到其它依赖该`factory`的程序。

而且最大的问题是`service`通过`factory`依赖的两个工具类是隐藏在代码里面的，对`client`是完全透明的，一旦我们忘记`service`里需要依赖这两个工具类，从而不在`client`里给`factory`设值，程序就运行崩溃了(`attempted`)。

```
public class RealBillingServiceTest extends TestCase {

  private final PizzaOrder order = new PizzaOrder(100);
  private final CreditCard creditCard = new CreditCard("1234", 11, 2010);

  private final InMemoryTransactionLog transactionLog = new InMemoryTransactionLog();
  private final FakeCreditCardProcessor processor = new FakeCreditCardProcessor();

  @Override public void setUp() {
    TransactionLogFactory.setInstance(transactionLog);
    CreditCardProcessorFactory.setInstance(processor);
  }

  @Override public void tearDown() {
    TransactionLogFactory.setInstance(null);
    CreditCardProcessorFactory.setInstance(null);
  }

  public void testSuccessfulCharge() {
    RealBillingService billingService = new RealBillingService();
    Receipt receipt = billingService.chargeOrder(order, creditCard);

    assertTrue(receipt.hasSuccessfulCharge());
    assertEquals(100, receipt.getAmountOfCharge());
    assertEquals(creditCard, processor.getCardOfOnlyCharge());
    assertEquals(100, processor.getAmountOfOnlyCharge());
    assertTrue(transactionLog.wasSuccessLogged());
  }
}
```

## 2 依赖注入

这个设计模式的核心是依赖倒转原则(`separate behaviour from dependency resolution`)。

依赖倒转原则认为，`service`不应该负责去寻找到两个具体的工具类，而应该是外部找到注入给`service`的。

并且注入的方式应该是**通过构造方法**的方式注入，这样该类需要什么参数，一目了然。

>The dependency is exposed in the API signature.

更改`service`后的代码：

```
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  public RealBillingService(CreditCardProcessor processor,TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

## 3 Guice里的依赖注入

`Guice`框架底层已经把实现搞定，我们需要做的事就是将我们的接口对应到它们的实现。

### 3.1 AbstractModule

用上面的例子举例，首先得自定义一个`Module`类去继承`Guice`提供的`AbstractModule`。

```
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(CreditCardProcessor.class).to(PaypalCreditCardProcessor.class);
    bind(BillingService.class).to(RealBillingService.class);
  }
}
```

### 3.2 @Inject

将注解`@Inject`放到`service`的构造方法上，`Guice`会去它的容器找到构造方法中的参数对应的`bean`，并设置给构造方法的参数。

```
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  @Inject
  public RealBillingService(CreditCardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

### 3.3 createInjector

用该方法就可以初始化`Guice`里的容器：

```
public static void main(String[] args) {
    Injector injector = Guice.createInjector(new BillingModule());
    BillingService billingService = injector.getInstance(BillingService.class);
    ...
}
```

并且使用`injector`可以从任意类获取到放进`Guice`中的`bean`。