# Provider Bindings

当@Provides注解过的方法可以称之为Provider，因为它们其实在内部是实现了一个叫Provider的接口：

```
public interface Provider<T> {
  T get();
}
```

每一个方法在Guice内部都是Provider的一个实现类，因此我们也可以去实现Provider来实现binding：

```
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  private final Connection connection;

  @Inject
  public DatabaseTransactionLogProvider(Connection connection) {
    this.connection = connection;
  }

  public TransactionLog get() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setConnection(connection);
    return transactionLog;
  }
}
```

最后我们需要使用`.toProvider`来将`binding`放进`Guice map`：

```
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class)
        .toProvider(DatabaseTransactionLogProvider.class);
  }
}
```