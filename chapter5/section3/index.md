# Instance Bindings

通常，一个类没有其它依赖时，可以给它set一个它自己类型的实例对象。如：

```
bind(String.class)
    .annotatedWith(Names.named("JDBC URL"))
    .toInstance("jdbc:mysql://localhost/pizza");
bind(Integer.class)
    .annotatedWith(Names.named("login timeout seconds"))
    .toInstance(10);
```

`toInstance`会降低程序启动速度，可以使用`@Provides`替代。

也可以使用`bindConstant`绑定常量：

```
bindConstant()
    .annotatedWith(HttpPort.class)
    .to(8080);
```

`bindConstant`是绑定基本数据类型(primitive)和其他常量类型(如String、enum和Class)的快捷方式。