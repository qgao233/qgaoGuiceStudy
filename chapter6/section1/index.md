# Injecting Providers

对于普通的依赖项注入，每个类型都得到它的每个依赖类型的一个实例。

RealBillingService获得一个CreditCardProcessor和一个TransactionLog。

有时需要依赖类型的多个实例。当需要这种灵活性时，Guice会绑定提供者。当调用get()方法时，提供程序生成一个值:

```
public interface Provider<T> {
  T get();
}
```

将`provider`的`type`参数化，以区分`Provider<TransactionLog>`和`Provider<CreditCardProcessor>`。无论在何处注入值，都可以为该值注入`provider`。

```
public class RealBillingService implements BillingService {
  private final Provider<CreditCardProcessor> processorProvider;
  private final Provider<TransactionLog> transactionLogProvider;

  @Inject
  public RealBillingService(Provider<CreditCardProcessor> processorProvider,
      Provider<TransactionLog> transactionLogProvider) {
    this.processorProvider = processorProvider;
    this.transactionLogProvider = transactionLogProvider;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    CreditCardProcessor processor = processorProvider.get();
    TransactionLog transactionLog = transactionLogProvider.get();

    /* use the processor and transaction log here */
  }
}
```

对于每一个绑定，不管有没有注解，注入器都为它的`Provider`提供了一个内置绑定。

## 1 Providers for multiple instances

当需要同一类型的多个实例时，请使用`Provider`。假设您的应用程序在比萨饼收费失败时保存了一个摘要条目和详细信息。通过`Provider`，您可以在需要的时候获得一个新条目：

```
public class LogFileTransactionLog implements TransactionLog {

  private final Provider<LogFileEntry> logFileProvider;

  @Inject
  public LogFileTransactionLog(Provider<LogFileEntry> logFileProvider) {
    this.logFileProvider = logFileProvider;
  }

  public void logChargeResult(ChargeResult result) {
    LogFileEntry summaryEntry = logFileProvider.get();
    summaryEntry.setText("Charge " + (result.wasSuccessful() ? "success" : "failure"));
    summaryEntry.save();

    if (!result.wasSuccessful()) {
      LogFileEntry detailEntry = logFileProvider.get();
      detailEntry.setText("Failure result: " + result);
      detailEntry.save();
    }
  }
```

## 2 Providers for lazy loading

如果您对某个类型有依赖，而该类型的生成成本特别高，那么您可以使用`Provider`来延迟该工作。当您不总是需要依赖项时，这尤其有用：

```
public class DatabaseTransactionLog implements TransactionLog {

  private final Provider<Connection> connectionProvider;

  @Inject
  public DatabaseTransactionLog(Provider<Connection> connectionProvider) {
    this.connectionProvider = connectionProvider;
  }

  public void logChargeResult(ChargeResult result) {
    /* only write failed charges to the database */
    if (!result.wasSuccessful()) {
      Connection connection = connectionProvider.get();
    }
  }
```

## 3 Providers for Mixing Scopes

直接注入范围较窄的对象通常会导致应用程序中出现意外行为。

在下面的示例中，假设您有一个独立的`ConsoleTransactionLog`，它依赖于请求作用域的当前用户。

如果将用户直接注入到`ConsoleTransactionLog`构造函数中，那么在应用程序的整个生命周期中，用户只会被评估一次。这种行为是不正确的，

因为每一个请求的用户都不一样。因此应该使用Provider。

由于提供程序按需生成值，因此它们使您能够安全地混合作用域：

```
@Singleton
public class ConsoleTransactionLog implements TransactionLog {

  private final AtomicInteger failureCount = new AtomicInteger();
  private final Provider<User> userProvider;

  @Inject
  public ConsoleTransactionLog(Provider<User> userProvider) {
    this.userProvider = userProvider;
  }

  public void logConnectException(UnreachableException e) {
    failureCount.incrementAndGet();
    User user = userProvider.get();
    System.out.println("Connection failed for " + user + ": " + e.getMessage());
    System.out.println("Failure count: " + failureCount.incrementAndGet());
  }
```