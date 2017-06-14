# Robolectric
---
## 在 Android Studio 中使用 Robolectric
1. 添加依赖到 `build.gradle` 中

  ```groovy
  testCompile 'org.robolectric:robolectric:3.3.2'
  ```
2. 如果在**非 Windows 平台**上使用，修改 Run Configuration，在默认配置的 `Android JUnit` 项目中，修改 `Working directory` 为 `$MODULE_DIR`

  ![](http://robolectric.org/images/android-studio-configure-defaults.png)
  
## Robolectric 能做什么
### 基于 X86 架构的 Android 代码单元测试
只使用 JUnit 是不能在 X86 主机上测试引用到 Android SDK 的代码的，因为 SDK 中存在大量的本地方法；如果选择在真机 / 模拟器上运行集成测试，那么在 X86 JVM 上运行测试在速度上的巨大优势就丧失了。Robolectric 提供了一套 Android SDK 在 X86 主机上的 Mock 实现，使几乎所有 Android 项目代码可以在开发机或者 CI 环境的 JVM 中被迅速地测试。
### Mock 对象
Robolectric 的 [Shadow 功能](#Shadow)提供给开发者编写 Mock 对象的能力，不需再额外引入诸如 [Mockito](http://site.mockito.org/) 之类的框架。

## 使用 Robolectric
### 测试运行器
Robolectric 可以视作一个为 Android 项目设计的 JUnit 的扩展，要使用它的功能，就需要使用特殊的测试运行器。`RobolectricTestRunner` 是最常用的一个，此外还有 `ParameterizedRobolectricTestRunner` 可用来实现 JUnit 的参数化测试。

### 配置
#### 配置方式
Robolectric 支持数种配置测试的方法。
##### 使用 `@Config` 注解
`@Config` 注解用于测试类、测试方法级的配置。加在测试方法上的注解配置会覆盖加在测试类上的注解配置。

##### 使用 `robolectric.properties` 配置文件
使用配置文件可以为整个测试包指定配置。在项目中的 `src/test/resources` 目录下创建、编写与测试代码包名对应的配置文件。如果某个测试类 / 测试方法被 `@Config` 标注，则相应配置会被注解中的覆盖。

##### 全局配置
继承 `RobolectricTestRunner` 并覆盖 `buildGlobalConfig()` 方法，然后在 `@RunWith` 注解中传入这个自定义测试运行器。

#### 常用的可配置的项目

项目名称|配置项名称|描述
------|--------|---
目标 SDK 级别|`sdk`、`minSdk`、`maxSdk`|默认情况下，Robolectric 会在清单文件的 `targetSdkVersion` 版本上运行测试，如果需要在不同的 SDK 级别上运行测试，可以指定 `sdk`、`minSdk` 和 `maxSdk`。注意，`sdk` 与后两个项目**不可同时存在于一个配置**中。
清单文件路径|`manifest`|默认值是 `AndroidManifest.xml`（相对于当前目录）
Application|`application`|默认情况下，Robolectric 会创建在清单文件中指定的 `Application` 类型，如果需要提供其它的 `Application`，可以通过该配置项来指定。
构建配置|`constants`|由 Gradle 生成的构建配置 `class` 对象。
[自定义 Shadow](#使用 Shadow)|`shadows`|自定义 Shadow 的 `class` 对象列表，用于创建 Mock 对象
资源路径|`resourceDir`|默认值是 `res` （相对于清单文件目录）
资产路径|`assetDir`|默认值是 `assets` （相对于清单文件目录）

### 测试 Activity
Activity 是 Android 组件中最重要的一部分。Robolectric 的 `ActivityController` 流式 API 用于测试 Activity，可以用它来控制被测 Activity 的生命周期，并获得被测 Activity 的引用。类似的 API 还可以用来测试 Fragment 和 Service。

```java
ActivityController controller = Robolectric.buildActivity(MyAwesomeActivity.class, intent).create().start();
Activity activity = controller.get();
// assert that something hasn't happened
activityController.resume();
// assert it happened!
```
#### 控制 Activity 布局 attach 到 Window 的时机
在真机或模拟器上，Activity 的布局在 `Activity.onPostResume()` 后才会 attach 到 `Window` 上，在此之前布局中的视图都不可见，并且无法被点击。Robolectric 把这个时机的控制权交给的编写测试的开发者，在对 `ActivityController` 的流式调用中，在适当的时机调用 `visible()`，如下所示。

```java
Activity activity = Robolectric.buildActivity(MyAwesomeActivity.class).create().start().resume().visible().get();
```

### Shadow
除了测试 Activity、Fragment 与 Service 之外，Robolectric 另一重要的功能非 Shadow 莫属。使用 Shadow，开发者可以在测试时创建任何类型的 Mock 对象。事实上，Robolectric 对 Android SDK 的 Mock 也是使用 Shadow 实现的。
#### 创建 Shadow
新建一个类，并使用 `@Implements` 注解标注它。如果在继承链上存在被 Shadow 的类型，则应该**继承 Shadow，而不是原类型**，比如下面这个样例：

```java
@Implements(ViewGroup.class)
public class ShadowViewGroup extends ShadowView {
}
```
注意，Shadow 类**必须有一个无参的构造器**，以供 Robolectric 通过反射构造它。然后，在这个类中声明想要 Mock 的方法，并使用 `@Implementation` 标注它，有一点像继承被 Shadow 的类并覆盖，但是它可以突破 `final` 的限制：

```java
@Implementation
public void setImageResource(int resId) {
  // implementation here.
}
```
如果需要 Mock 构造方法，那么申明一个名为 `__constructor__` 的方法，接收的参数与被 Shadow 的类的构造方法一致。

```java
@Implements(TextView.class)
public class ShadowTextView {
  
  public void __constructor__(Context context) {
    this.context = context;
  }
}
```
如果需要访问被 Shadow 的对象，申明一个被 Shadow 类型的成员并使用 `@RealObject` 标注它。

```java
@Implements(Point.class)
public class ShadowPoint {
  @RealObject 
  private Point realPoint;
  
  public void __constructor__(int x, int y) {
    realPoint.x = x;
    realPoint.y = y;
  }
}
```
#### 使用 Shadow
在需要使用 Shadow 类的测试类 / 测试方法的 `@Config` 注解（或测试包的 properties）的 `shadows` 参数中传入需要使用的 Shadow 类数组即可。接下来，在测试代码中，如果创建、使用到被 Shadow 的类，这些调用会被 Robolectric 自动转发到 Shadow 类上。
##### 在测试代码中获取 Shadow 类对象的引用
通常，测试代码不需要关心它使用的哪个对象被 Shadow 了。如果的确有这样的需求，可以调用 `Shadow.extract(Object)` 来获取 Shadow 类对象的引用。

### 测试版 Application
如果 Application 类型是在清单文件中指定的，Robolectric 会自动尝试使用一个测试版的 Application。这个 Application 应该继承在清单文件中声明的那个，并且实现 `TestLifecycleApplication` 接口。有时，你可能想要在测试运行器开始运行前，或是在任何一个测试方法执行前执行一些操作，比如，在测试时需要为 DI 框架提供一份不同的依赖（下面这个样例中使用的 DI 框架是 RoboGuice）：

```java
public class FooApplication extends Application {
  @Override
  public void onCreate() {
    super.onCreate();
      
    ApplicationModule module = new ApplicationModule();
    setBaseApplicationInjector(this, DEFAULT_STAGE, newDefaultRoboModule(this), module);
  }
}

public class TestFooApplication extends FooApplication implements TestLifecycleApplication {
  @Override
  public void onCreate() {
    super.onCreate();

    // 使用了测试专用的 Module
    TestApplicationModule module = new TestApplicationModule();
    setBaseApplicationInjector(this, DEFAULT_STAGE, newDefaultRoboModule(this), module);
  }

  @Override
  public void beforeTest(Method method) {
  }

  @Override
  public void prepareTest(Object test) {
    // 在每一个测试运行前先注入依赖
    getInjector(this).injectMembers(test);
  }

  @Override
  public void afterTest(Method method) {
  }
}
```