# Linked Bindings

说白了就是将父类交给Guice管理，但是父类的引用指向真正的实现类。

>官方说法：Linked bindings map a type to its implementation.

例如：

```
public class BillingModule extends AbstractModule {
    @Provides
    TransactionLog provideTransactionLog(DatabaseTransactionLog impl) {
        return impl;
    }
}
```

使用`injector.getInstance(TransactionLog.class)`时，当injector碰到有依赖该bean时，自动会将该bean的实现类赋给该bean，此例则会把`DatabaseTransactionLog`给`TransactionLog`。

当然，该方式存在传递性，例如：

```
public class BillingModule extends AbstractModule {
    @Provides
    TransactionLog provideTransactionLog(DatabaseTransactionLog databaseTransactionLog) {
        return databaseTransactionLog;
    }

    @Provides
    DatabaseTransactionLog provideDatabaseTransactionLog(MySqlDatabaseTransactionLog impl) {
        return impl;
    }
}
```

使用`injector.getInstance(TransactionLog.class)`时，会返回`MySqlDatabaseTransactionLog`。

一样的，除了使用注解方式外，还可以使用`bind`（在前面的章节中已经大量使用过）。

```
bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
```