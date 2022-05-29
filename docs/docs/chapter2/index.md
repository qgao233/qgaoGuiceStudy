# Guice 快速上手

用一个简单的例子说明，

废话不多说，直接上最终的结果，通过注释轻松上手：

```
import static java.lang.annotation.RetentionPolicy.RUNTIME;

import com.google.inject.AbstractModule;
import com.google.inject.Guice;
import com.google.inject.Injector;
import com.google.inject.Key;
import com.google.inject.Provides;
import java.lang.annotation.Retention;
import javax.inject.Inject;
import javax.inject.Qualifier;

public class GuiceDemo {
  @Qualifier                // javaEE提供的注解
  @Retention(RUNTIME)
  @interface Message {}

  @Qualifier
  @Retention(RUNTIME)
  @interface Count {}

  /**
   * 用来绑定message和count的module，会被Greeter类依赖
   * 除了下面的注解方法，还可以使用上一章中提到的bind方法进行绑定
   */
  static class DemoModule extends AbstractModule {
    @Provides               // Guice提供的注解，表示会被放入Guice容器中
    @Count                  // 用来帮助注入时别找错了bean
    static Integer provideCount() {
      return 3;
    }

    @Provides
    @Message
    static String provideMessage() {
      return "hello world";
    }
  }

  static class Greeter {
    private final String message;
    private final int count;

    // Greeter类的构造方法表示它需要1个String类的参数和1个用来打印消息次数的int类参数
    // 而@Inject表示这个构造方法可以被Guice使用
    @Inject
    Greeter(@Message String message, @Count int count) {
      this.message = message;
      this.count = count;
    }

    void sayHello() {
      for (int i=0; i < count; i++) {
        System.out.println(message);
      }
    }
  }

  public static void main(String[] args) {
    /*
     * Guice.createInjector() 可以传入任意多个module，然后返回一个Injector实例
     * 大多数程序都会在Main方法中调用这个方法一次。
     */
    Injector injector = Guice.createInjector(new DemoModule());

    /*
     * 现在我们得到了injector实例，可以构建对象了。
     * Guice因为是惰性加载bean的原因，只有当getInstance调用时，才表示要叫Guice去创建Greeter实例，
     * 构造实例时发现其构造方法需要两个参数，这时会去自己容器里找是否存在这两个参数，找到后设值，调用构造方法并返回。
     * 这种一层一层地找依赖称之为传递性（transitive）
     */
    Greeter greeter = injector.getInstance(Greeter.class);

    // 在控制台打印3次"hello world"
    greeter.sayHello();
  }
}
```