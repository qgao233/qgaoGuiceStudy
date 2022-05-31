# Untargeted Bindings

绑定没有实现类的`bean`，即有`bind()`没有`to()`的`bean`。

>This is most useful for concrete classes and types annotated by either @ImplementedBy or @ProvidedBy. An untargetted binding informs the injector about a type, so it may prepare dependencies eagerly. Untargetted bindings have no to clause, like so:

因为没有具体的实现类，所以`injector`会采用饿汉模式，去找到容器中对应的父类`bean`或子类`bean`，赶紧`set`给这个`binding`，这对容器中被`@ImplementedBy`或`@ProvidedBy`注解过的父类并且还有具体的实现类的情况很有用：

```
bind(MyConcreteClass.class);
bind(AnotherConcreteClass.class).in(Singleton.class);
```