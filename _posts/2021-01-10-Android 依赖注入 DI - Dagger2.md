---
layout: post
title: "Android 依赖注入 DI - Dagger2"
date: 2021-01-10
description: "Android 依赖注入 DI - Dagger2"
tag: Dagger2
---

### 1.依赖注入 (Dependency Injection) 
#### 1.1 面向接口编程

```java
public interface Drivable {
    void drive();
}

public class Bike implements Drivable {
    @Override
    public void drive() {
        System.out.println("骑车");
    }
}

public class Car implements Drivable {
    @Override
    public void drive() {
        System.out.println("开车");
    }
}

public class Subway implements Drivable {
    @Override
    public void drive() {
        System.out.println("乘地铁");
    }
}

public class Worker {

    private void gotoWork() {
        // 1.依赖具体,非面向接口编程
        Bike bike = new Bike();
        bike.drive();
        // 2.依赖抽象,面向接口
        Drivable drivable = new Bike();
//        Drivable drivable = new Car();
//        Drivable drivable = new Subway();
        drivable.drive();
    }
    public static void main(String[] args) {
        Worker worker = new Worker();
        worker.gotoWork();
    }
}
```
方式 1 中直接依赖 Bike 类，Worker 依赖具体的实现类，一旦改变具体的实现类，就需要改动。对于业务开发这是允许的。

方式 2 使用一个 Drivable 接口，采用面向接口编程，对于 Worker 这个上层类来说，不再依赖具体，而是依赖抽象，如果改变 gotoWork 的方式，无需做出改变，扩展性得到了很大提升，符合**依赖倒置原则**。

>**依赖倒置原则**：高层模块不要依赖低层模块。高层模块和低层模块应该通过抽象来互相依赖。除此之外，抽象不要依赖具体实现细节，具体实现细节依赖抽象。

#### 1.2 什么是依赖注入？

上面的例子，改成面向接口编程后，虽然扩展性提升，但是仍然有具体的对象创建，如果 Bike 类的构造函数改变，如改成有参构造，那么 Worker 也要跟着变；或者更换为其他出行方式，需要创建其他对象，Worker 依然要改。那么有没有方式不用修改 Worker 类呢？

**计算机领域的所有问题都可以通过增加一个中间层来解决**

可以想象将具体的对象构建放在 Worker 类的外面，把依赖的对象传入进来，这样就不再需要改变 Worker 类，这就用到了**依赖注入（Dependency Injection）**，依赖注入是**控制反转（Inversion Of Control）**的实现手段之一，其实对于这个例子来说，就是将对象创建的过程交给其他职责类，不再自己完成创建，这就是控制反转。那么如果实现依赖注入呢？

* 方式一:构造函数传入参数

```java
public class Worker {

    private Drivable mDrivable;

    public Worker(Drivable mDrivable) {
        this.mDrivable = mDrivable;
    }

    private void gotoWork() {
        // 3.1依赖注入
        mDrivable.drive();
    }
    
    public static void main(String[] args) {
        // 3.1交给使用者创建具体对象，IOC
        Worker worker = new Worker(new Bike());
        worker.gotoWork();
    }
}
```

* 方式二:通过 setter 方式注入

```java
public class Worker {

    private Drivable mDrivable;

    public void setDrivable(Drivable drivable) {
        this.mDrivable = drivable;
    }

    private void gotoWork() {
        // 3.2 依赖注入
        mDrivable.drive();
    }
    
    public static void main(String[] args) {
        // 3.2 交给使用者创建具体对象，IOC
        Worker worker = new Worker();
        worker.setDrivable(new Bike());
        worker.gotoWork();
    }
}
```

* 方式三:通过接口方式注入

```java
public interface DependencySetter {
    void set(Drivable drivable);
}

public class Worker implements DependencySetter { {

    private Drivable mDrivable;

    @Override
    public void set(Drivable drivable) {
        this.mDrivable = drivable;
    }

    private void gotoWork() {
        // 3.3依赖注入
        mDrivable.drive();
    }
    
    public static void main(String[] args) {
        // 3.3 作为接口配置
        Worker worker = new Worker();
        worker.set(new Bike());
        worker.gotoWork();
    }
}
```

* 方式四:使用依赖注入框架

（1）Spring 中的依赖注入，依赖接口，通过注解可以完成实现类的注入

![Spring](https://github.com/RalfNick/PicRepository/raw/master/dagger2/spring.png)

（2）Android 中 dagger 依赖注入框架：dagger 是由 Square 公司开发的，Dagger2 是在 Dagger 的基础上来的，Dagger2 由 Google 公司开发并维护。现在比较『先进』的框架都会采用注解的方式，Dagger2 也是基于注解开发的，而 Dagger2 中所涉及到的注解其实是基于 [javax.inject](https://docs.oracle.com/javaee/7/api/javax/inject/package-summary.html) 上开发的，它出自 [JSR330](https://jcp.org/en/jsr/detail?id=330)。Dagger2 是适应于 Java 和 Android 开发的依赖注入框架，它不仅仅对 Android 开发有效。但是对于后端一般很少使用 dagger2 作为依赖注入框架，因为 Spring 的依赖注入框架更强大。

对于较大的 App，Google 工程师推荐使用 Dagger，对于小型 App 手动注入或者使用其他工具，甚至不使用依赖注入，基本都能满足需求。但是把 Dagger 引入到项目中，对于 App 的长久开发是收益比较大的，特别是开发成本上。


![app_size](https://github.com/RalfNick/PicRepository/raw/master/dagger2/app_size.png)

![dagger_app_get](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger_app_get.png)

从曲线中可以看出，使用 Dagger 开始时，可能相对成本比较高，学习成本等，但是随着 App 越来越大，开发成本等各方面收益是很大的。Dagger 的优势主要在三个方面：

> 性能（Performance）：Dagger >生成的中间代码是在编译期完成的，没有使用反射
>
> 正确性（Correctness）：Dagger >在编译其能保证类型的准确性，如果注入有问题在工程编译时会报错，不至于在>运行时出现 Crash
>
>扩展性（Scalability）：Dagger 具有很好的扩展性，特别是大型 App，如 Gmail、Google Photos、 YouTube.

### 2.Dagger2

#### 2.1 Java 注解

![javax](https://github.com/RalfNick/PicRepository/raw/master/dagger2/javax_annotation.png)

```java
1、Inject 注解用于构造函数，方法，字段的注解；
（1）用于字段上代表要给改字段注入，即赋值
（2）用于构造函数上时，代表该对象可以被注入，注入时会通过该构造函数创建对象

2、Scope 注解用来表明一个注解器如何使用被注入的对象的，如果没有使用 @Scope，每次创建时都会重新创建一个实例，如果使用 @Singleton 这样一个注解，那么会保留同一个实例，代表单例。@Singleton 是使用 @Scope 注解的注解，也可以使用 @Scope 自己定义一个注解。

3、Qualifier 用来定义注解的注解，如 @Name，用于区分有歧义的注入，比如注入两个相同类型的变量，可以使用 @Name 来给提供值得地方和被注入的地方打上相同的注解，这样就能正确的注入。
```
#### 2.2 dagger2 注解

除了 javax 中提供的注解，Dagger2 中也提供了很多注解，常用的注解：@Component、@Module、@Provides、@Binds、@Subcomponent，还有提供集合类的注解，@IntoSet、@IntoMap、@@Multibinds、@ClassKey、@StringKey 等

* 1.@Component 相当于联系纽带，将 @inject 标记的需求方和依赖绑定起来，并建立了联系，而 Dagger2 在编译代码时会依靠这种关系来进行对应的依赖注入。使用时一般我们只顶一个一个 Component 接口，Dagger 会为我们生成 实现类，通过实现类来进行注入

* 2.@Component 来完成注入，那么注入需要的对象实例是由 @Module 标注的类提供的，或者由 @Inject 的类。@Module 中的方法一般使用 @Provides 注解的方法（可以是静态方法） 来提供实例，或者 @Binds 注解的方法（抽象方法）

* 3.@Subcomponent 能够用父 Component 的实例集合，同时可以有自己的实例对象，来进行注入，但是 @Scope 不能和父 Component 相同

#### 2.3 dagger2 基本用法

**（1）基本用法**

* 定义 Component

@Component 定义一个接口，接口中声明我们需要的功能，如给 Activity 进行注入，或者返回需要的类

```java
@Singleton
@Component(modules = {EntityModule.class, BindsEntityModule.class})
public interface EntityComponent {

  void injectEntity(InjectorDemoActivity activity);

  void injectEntity(InjectorDemoActivity2 activity);
}
```
* 定义 Module 或者使用 @Inject 修饰构造函数，注意需要注入对象的位置，不能同时使用 Module 和 @Inject 提供，否则会报错

```java
// 使用 Module 方式
@Module
public abstract class BindsEntityModule {

  @Provides
  @Named("static provide BindsEntity")
  public static BindsEntity provideFirstEntity(@NonNull BindsEntity entity) {
    return entity;
  }


  @Binds
  @Named("Binds provide BindsEntity")
  public abstract BindsEntity provideSecondEntity(@NonNull BindsEntity entity);
}

// 使用 @Inject 修饰构造函数
public class InjectEntity {

  @Inject
  public InjectEntity() {
  }

  @NonNull
  @Override
  public String toString() {
    return InjectEntity.class.getSimpleName();
  }
}
```
```java
public class InjectorDemoActivity extends AppCompatActivity {

  public static final String TAG = "InjectorDemoActivity";

  @Inject
  InjectEntity mInjectEntity;
  @Inject
  SingletonEntity mSingletonEntity1;
  @Inject
  SingletonEntity mSingletonEntity2;
  @Inject
  ModuleEntity mModuleEntity;
  @Named("static provide BindsEntity")
  @Inject
  BindsEntity mBindsEntity1;
  @Named("Binds provide BindsEntity")
  @Inject
  BindsEntity mBindsEntity2;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_injector_demo);
    findViewById(R.id.button).setOnClickListener(v ->
        startActivity(new Intent(this, InjectorDemoActivity2.class)));
    // 构建 Component 进行注入
    DaggerEntityComponent.builder()
        .build()
        .injectEntity(this);
  }

  @Override
  protected void onResume() {
    super.onResume();
    // mInjectEntity
    Log.e(TAG, mInjectEntity.toString());
    // mModuleEntity和mSingletonEntity
    Log.e(TAG, mModuleEntity.toString());
    Log.e(TAG, mSingletonEntity1.toString());
    Log.e(TAG, mSingletonEntity2.toString());
    // mBindsEntity1 和 mBindsEntity2
    Log.e(TAG, mBindsEntity1.toString());
    Log.e(TAG, mBindsEntity2.toString());
  }
}
```

**（2）multibinds 集合注入**

定义 MultiBindsComponent，给 MultiInjectActivity 中的变量进行注入。MultiBindsComponent 中使用的 modules 是 MultiBindsModule.class，同时用到了 dependencies，可以依赖 SetComponent.class，MapComponent.class。modules 和 dependencies 都可以是一个或者多个 Class，dependencies 可以理解为组合的形式。

```java
@Component(modules = MultiBindsModule.class,
dependencies = {SetComponent.class, MapComponent.class})
public interface MultiBindsComponent {

  void inject(MultiInjectActivity activity);
}
```

```java
public class MultiInjectActivity extends AppCompatActivity {

  public static final String TAG = "MultiInjectActivity";
  // MultiBinds
  @Inject
  Set<String> stringSet;
  @Inject
  Set<Integer> integerSet;

  @Inject
  Map<String, Integer> mSIMap;

  @Inject
  Map<String, String> mSSMap;

  @Inject
  Map<MapEntityType, MapEntity> mEntityMap;

  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_muiti_inject);

    inject();
    print();
  }

  private void inject() {
    DaggerMultiBindsComponent.builder()
        .setComponent(DaggerSetComponent.create())
        .mapComponent(DaggerMapComponent.builder().build())
        .build()
        .inject(this);
  }
}
```
MultiBindsModule 是 抽象类，@Multibinds 的使用时要求抽象类，Dagger2 会创建一个空的 Map 或者 Set 集合，如果我们需要注入的集合可能为空的时候才使用这个种方式。

```java
@Module
public abstract class MultiBindsModule {

  @Multibinds
  abstract Set<Integer> provideIntegerSet();

  @Multibinds
  @Named("empty map")
  abstract Map<String, Integer> provideStringMap();
}
```
在看下依赖的 Component，这里只看下 MapComponent。MapComponent 的定义和 MultiBindsComponent 的定义是一样的，只不过 MapComponent 不会对 Activity 直接注入，它负责给 MultiBindsComponent 的提供依赖。

```java
@Component(modules = MapModule.class)
public interface MapComponent {

  Map<String, Integer> provideSIMap();

  Map<String, String> provideSSMap();

  Map<MapEntityType, MapEntity> provideClassMap();

  @Component.Builder
  interface Builder {
    MapComponent build();
  }
}
```
MapModule 中提供 Map 集合中的元素，Key 对用方法注解上的 Key，方法返回值对应的是 Value，Dagger2 会把相同 Key - Value 类型的元素放入同一个 Map 中。
```java
@Module
public class MapModule {

  // 放入 Map<String,Integer> 集合中
  @Provides
  @IntoMap
  @StringKey("first")
  static Integer provideFirstInteger() {
    return 11;
  }
  // 放入 Map<String,Integer> 集合中
  @Provides
  @IntoMap
  @StringKey("second")
  static Integer provideSecondInteger() {
    return 22;
  }
  // 放入 Map<String,String> 集合中
  @Provides
  @IntoMap
  @StringKey("first")
  static String provideFirstString() {
    return "AA";
  }
  // 放入 Map<String,String> 集合中
  @Provides
  @IntoMap
  @StringKey("second")
  static String provideSecondString() {
    return "BB";
  }
  
  // 放入 Map<MapEntityType,MapEntity> 集合中
  @Provides
  @IntoMap
  @MapEntityKey(MapEntityType.TYPE_A)
  static MapEntity provideFirstClass(MapEntity entity) {
    return entity;
  }

  // 放入 Map<MapEntityType,MapEntity> 集合中
  @Provides
  @IntoMap
  @MapEntityKey(MapEntityType.TYPE_B)
  static MapEntity provideSecondClass(MapEntity entity) {
    return entity;
  }
}
```
MapEntityKey 是自定义的一个 MapKey, Dagger2 中提供了 4 种 MapKey，ClassKey、IntKey、StringKey 和 LongKey，这个 MapKey 对用 Map 集合中的 Key，value 对应 Module 中返回的对象。

```java
@MapKey
public @interface MapEntityKey {
  MapEntityType value();
}

public enum MapEntityType {
  TYPE_A, TYPE_B
}
```

利用 multibinds 可以实现一个 spi（Service Provider Interface） 的 plugin，这种方式实现的 plugin 相比 APT 方式，相对代码会多出一些，这里仅给出一个简单的例子。

![dagger_plugin](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger_plugin.png)

```java
@Singleton
@Component(modules = {UserModule.class, ShoppingModule.class})
public interface PluginComponent {
  Map<Class<?>, Plugin> getPlugins();
}

// module-user --> UserModule
@Module
public class UserModule {
  @Singleton
  @Provides
  @IntoMap
  @ClassKey(UserPlugin.class)
  static Plugin provideUserPlugin(UserPluginImpl userPlugin) {
    return userPlugin;
  }
}

// module-shopping --> ShoppingModule
@Module
public class ShoppingModule {

  @Singleton
  @Provides
  @IntoMap
  @ClassKey(ShoppingPlugin.class)
  static Plugin provideUserPlugin(ShoppingPluginImpl shoppingPlugin) {
    return shoppingPlugin;
  }
}
```

```java
public class PluginManager {
  public static final String TAG = "PluginManager";
  private static final PluginManager INSTANCE = new PluginManager();
  private Map<Class<?>, Plugin> mPluginMaps = new HashMap<>();

  public static PluginManager getInstance() {
    return INSTANCE;
  }

  private PluginManager() {
    //no instance
  }

  @Nullable
  public <T extends Plugin> T getPlugin(Class<T> clazz) {
    if (mPluginMaps != null && !mPluginMaps.isEmpty()) {
      return (T) mPluginMaps.get(clazz);
    }
    return null;
  }

  public void addPlugin(Map<Class<?>, ? extends Plugin> pluginMaps) {
    if (pluginMaps == null || pluginMaps.isEmpty()) {
      return;
    }
    for (Map.Entry<Class<?>, ? extends Plugin> entry : pluginMaps.entrySet()) {
      if (!mPluginMaps.containsKey(entry.getKey())) {
        mPluginMaps.put(entry.getKey(), entry.getValue());
      }
    }
  }
```

plugin 接口
```java
public interface Plugin {
}

public interface ShoppingPlugin extends Plugin {
  void getShopping();
}

public interface UserPlugin extends Plugin {
  void getUserInfo();
}
```

**（3）subcomponent/**

Subcomponent 的使用有一定的规则: 

* Subcomponent 能获取到父 Component 中能提供的所有对象
* Subcomponent 只能有一个父 Component
* Subcomponent 需要显示的声明一个 Builder，父 Component 中需要对外界提供 Subcomponent 接口
* Subcomponent 的 Scope 不能和 父 Component 的 Scope 相同

```java
// ParentComponent
@Singleton
@Component(modules = ParentModule.class)
public interface ParentComponent {

  ChildComponent.Builder childComponentBuilder();
}

// ParentModule
@Module(subcomponents = ChildComponent.class)
public class ParentModule {

  @Singleton
  @Provides
  public ParentEntity provideParentEntity() {
    return new ParentEntity();
  }
}
```
```java
// ChildComponent
@Subcomponent(modules = ChildModule.class)
public interface ChildComponent {

  void inject(SubComponentDemoActivity activity);

  // Subcomponent 需要 定义一个 Builder
  @Subcomponent.Builder
  interface Builder {
    ChildComponent build();
  }
}

// ChildModule
@Module
public class ChildModule {

  @Provides
  public SubEntity provideEntity(ParentEntity entity){
    return new SubEntity(entity);
  }
}
```
对比 dependencies 形式的依赖，dependencies 形式上类似于组合，subcomponents 形式上类似于继承，可以理解为横向扩展和纵向扩展

![subcomponents](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger_subcomponent.png)

我们知道在设计上优先使用组合，具有更好的扩展性，但 Subcomponent 也是有一定应用场景的，父 Component 中的资源需要注入到 Subcomponent 中。这种情况虽然可以也使用组合依赖的形式，但是使用组合依赖，会重新创建一个 Component。如：ComponentB 和 ComponentC 均依赖 ComponentA，使用组合形式，那么 ComponentB 和 ComponentC 中的 ComponentA 不是同一个，那么导致 ComponentB 和 ComponentC 不能使用相同的资源，在 Android 中可以保证 ComponentA 是 Application 级别单例，这种情况是没问题的。假设一个 Activity 中有个多个 Fragment，这种情况下每个 Fragment 都需要 Activity 级别的  Component 注入，那么每个 Fragment 需要拿到同一个 Activity 的  Component，这个 Component 就需要维护，这样的情况使用 Subcomponent 会更好一些，每个 Fragment 中的 SubComponent 都是在 Activity 中的 Component 创建的，并且 能保证 Fragment 的 SubComponent 的 Scope 小于 Activity 的 Scope。

![Subcomponent](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger_subcomponent_apply.png)

#### 2.4 Scope

@Scope 代表作用域，实际上是 Component 的作用域，并不是说打上 @Singleton ，就一定代表是单例，单例的维护主要靠对应的 Component 来维护，将 Component 设置为 Application 级别的对象，也是在整个应用中 Component 只有一个，那么它提供的对应也只有一个。

```java
@Scope
@Documented
@Retention(RUNTIME)
public @interface ActivityScope {
}

@Scope
@Documented
@Retention(RUNTIME)
public @interface FragmentScope {
}
```
上面提到 Subcomponent 不能和父 Component 的 Scope 一致，对于组合形式依赖的 Component，之间也不能有相同的 Scope，也就是说不同 Component 之间的 Scope 不能相同。JakeWharton 给出一个解释，如果两个不同 Component 有相同的 Scope，会打破 Scope 的限制。

![scope](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger_scope_question.png)

#### 2.5 Dagger 在 Android 上使用的问题

1、因为Android 中的四大组件是有生命周期的，而且实例的创建是有系统来完成的，而最好的方式使用 Dagger，是完全有 Dagger 来创建实例，所以就需要我们在生命周期中来完成注入的工作，就会出现这样的代码：

```java
    DaggerEntityComponent.builder()
        .build()
        .injectEntity(this);
```
这样带来的问题：大量类似的代码导致后期的维护和重构问题。根本上来讲，它要求需要被注入的类型，如 Activity 依赖 injector，也就是 Component，，尽管是面向接口的，但是仍旧破坏了依赖注入的原则，需要被注入的类，不应该知道如何被注入，只关心被注入的值。

### 3 Dagger 原理简单分析

#### 3.1 APT

APT(Annotation Processing Tool)是一种处理注释的工具,它对源代码文件进行检测找出其中的 Annotation，根据注解自动生成代码。Annotation 处理器在处理 Annotation 时可以根据源文件中的 Annotation 生成额外的源文件和其它的文件(文件具体内容由Annotation 处理器的编写者决定),APT还会编译生成的源文件和原来的源文件，将它们一起生成 class 文件。ButterKnife，dagger 等开源注解框架都采用了 APT 技术。APT 的使用能很好的简化开发工作，提高效率。

APT 主要流程

* 定义注解（如@automain）
* 定义注解处理器，自定义需要生成代码
* 使用处理器
* APT自动完成如下工作。

#### 3.2 Dagger 原理

![dagger_原理](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger2_how.png)

1、APT 生成 DaggerComponent 生成，名称为 Dagger + 自定义 Component 名字，如自定义 EntityComponent，生成后名字为 DaggerEntityComponent，提供构建方法有两种，通过 Builder 构建，也可以通过 create() 静态方法构建，前提是 Builder 没有参数，create() 实际上也是调用 Builder 构建。我们可以在自定义 Component 显示生命 @Component.Builder,否则 Dagger2 为默认生成一个 Builder，参数则是依赖的 Module 和 Component

* Builder 构建 Component

```java
  public static final class Builder {
    private EntityModule entityModule;

    private Builder() {
    }

    public Builder entityModule(EntityModule entityModule) {
      this.entityModule = Preconditions.checkNotNull(entityModule);
      return this;
    }

    public EntityComponent build() {
      if (entityModule == null) {
        this.entityModule = new EntityModule();
      }
      return new DaggerEntityComponent(entityModule);
    }
  }
```
* 使用 MemberInjector 注入

MemberInjector 的情况是 Component 接口中函数中带参数，所带的参数就是需要被注入的对象，对该对象中 @Inject 的成员变量进行注入。没有参数的情况则不会生成 MemberInjector，而是对函数的返回参数提供实例。这里仅分析一下函数中带参数生成 MemberInjector 情况。

生成 InjectorDemoActivity_MembersInjector，DaggerEntityComponent 通过 InjectorDemoActivity_MembersInjector 对  InjectorDemoActivity 中参数进行注入。

```java
  @Override
  public void injectEntity(InjectorDemoActivity activity) {
    injectInjectorDemoActivity(activity);
  }

  private InjectorDemoActivity injectInjectorDemoActivity(InjectorDemoActivity instance) {
    InjectorDemoActivity_MembersInjector.injectMInjectEntity(instance, new InjectEntity());
    InjectorDemoActivity_MembersInjector.injectMSingletonEntity1(instance, singletonEntityProvider.get());
    InjectorDemoActivity_MembersInjector.injectMSingletonEntity2(instance, singletonEntityProvider.get());
    InjectorDemoActivity_MembersInjector.injectMModuleEntity(instance, EntityModule_ProvideEntityFactory.provideEntity(entityModule));
    InjectorDemoActivity_MembersInjector.injectMBindsEntity1(instance, namedBindsEntity());
    InjectorDemoActivity_MembersInjector.injectMBindsEntity2(instance, new BindsEntity());
    return instance;
  }
```

InjectorDemoActivity_MembersInjector 需要注入的对象实例来源主要有以下几种情况：

 （1）Component 中的 Provider,这种一般是单例，或者是 Module 中的方法中有参数，对应的参数会生成 Provider
 
 （2）直接 new 一个实例，对应构造函数上有 @Inject 注解，或者 Module 中使用 @Binds 抽象方法中参数，官方文档一般建议使用 @Binds,因为能够少少生成一些辅助代码
 
 （3）Component 中有 Module 实例或者其他 Component 实例，这种情况则使用 Module 实例或者其他 Component 对应的方法提供的实例对象，如果是 Module,则不是直接调用 Module 实例的方法，而是通过 Module_Factory 中间类来调用 Module 中的方法来提供实例
 
 （4）Component 依赖的 Module 中提供实例的方式是静态的，同样 MemberInjector 会通过 Module_Factory 中间类来调用 Module 中方法来提供实例。

上述就是注入过程需要的核心类，注入的出发过程就是调用 Component 的中方法。

### 4 Dagger2 在 MVP 中的使用

Dagger2 是一个依赖注入框架，在 Java 开发或者 Android 开发中都可以使用，这里看下在 MVP 框架中的使用，当然在 MVVM 同样是可以使用。

![MVP](https://github.com/RalfNick/PicRepository/raw/master/dagger2/mvp_basic.png)

没有使用 Dagger 的 MVP，P 层和 M 层都需要自己创建，同时 P 层和 M 层中依赖的对象也要手动获取，就前面所说，如果 App 工程越来越大，对于开发成本从长期角度会很高， Dagger 的引入主要就是解决这个问题。
MVP 的注入从以下两个方面来考虑：

* 每一层需要注入的实例

V 层 ：对于 V 层（四大组件，Fragment），如果依赖 P 层，则需要给 V 层注入 P；

P 层：依赖 M 层和 V 层，需要注入 M 层以及对应 V 实现的 IView 接口实例,实际上就是四大组件或 Fragment；

M 层：依赖一些全局的实例，如网络请求配置，缓存等 

* 注入的时机

由于 Android 中四大组件是由系统来创建实例，所以对于这些组件中的 P 层注入就需要在合适时机，即根据他们生命周期来注入，一般会在生命周期回调最先调用的方法中进行注入，对于全局的 Component，会设置成单例，在 Application 中进行创建和保存，然后会传递给四大组件的 Component 作为依赖的 Component。

这里分析一个比较好的 MVP 的开源框架 [**MVPArms**](https://github.com/JessYanCoding/MVPArms),该框架采用 MVP+Dagger2+Retrofit+RxJava，集成了多个开源框架，开发起来相对比较容易，前提需要对 Dagger2 有一定的理解。

#### 4.1 MVPArms 简单说明

（1）需要全局配置 GlobalConfiguration，需要包括网络相关配置，异常处理，缓存配置，Gson 解析配置等，需要在 AndroidManifest.xml 中注册

```xml
 <!-- Arms 配置 -->
 <meta-data android:name="me.jessyan.mvparms.demo.app.GlobalConfiguration"
 android:value="ConfigModule" />
```

（2）从分包结构上可以看出，di 包是依赖注入的相关类，Component 和 Module,mvp 包除了 Model、Presenter、View,还有一个 contract，用于生命 Model 和 View 的接口，只是存放的一个规则，也可以不放在一起。

![dagger_arms_apply.png](https://github.com/RalfNick/PicRepository/raw/master/dagger2/dagger_arms_apply.png)

一个页面中不一定仅仅有一个 Component，可能有多个 Component 和对应的 Module，但是仅有一个 Component 对 Activity 或者 Fragment 进行注入，Component 提供的对象，都是 M 层和 P 层需要的对象。

#### 4.2 MVPArms 框架分析

![MVPArms](https://github.com/RalfNick/PicRepository/raw/master/dagger2/mvp_arms.png)

Activity 级别的 Component 依赖 Application 级别的 Component（单例），对 Presenter 和 Model 进行注入，这里的注入时机设计的比较好：首先 AppComponent 在 Application 的 onCreate(）方法中创建和注入，并保存成全局的对象，然后注册 Application.ActivityLifecycleCallbacks 实现，在 onActivityCreated 回调中对 Activity 进行注入，对于 Fragment 同样，在 
Activity 回调 onActivityCreated 中，判断如果有 Fragment 则注册 FragmentLifecycleCallbacks,在 FragmentLifecycleCallbacks 的 onFragmentCreated 回调中对 Fragment 进行注入。

该框架一个比较巧妙的设计：将配置信息配置在 AndroidManifest.xml 中，对于不同的模块可以设置自己的独特的配置，解析 AndroidManifest.xml 时会把所有的配置 Configuration 读取出来，作为依赖注入的提供者。

优势：该框架高度集成，也具有很好的可配置性，对于前中期 APP 开发，能够完成快速开发，对于 Component 和 Module，甚至是 Presenter 也能够做到复用。

缺点 ：多了 Module 和 Component，这个也是属于 Dagger2 的缺点，由于 Android 组件生命周期的存在，不能像 Spring 那样仅仅用一个注解就完成依赖注入。此外框架中使用 Presser 的主要是在 Activity 或者 Fragment 中，Recyclerview 中 item 没有使用 presenter，即没有采用 MVP 模式，对于简单的列表页面可以不用采用，对于复杂的列表情况，可以考虑集成 MVP 模式。

### 5 总结

* 介绍了依赖注入 (Dependency Injection) 的概念，是 IOC 的一种实现方式，以及 DI 的几种方式，构造函数传入；setter 设置；接口传入；使用 DI 框架，Android 中的 Dagger2。
* Dagger2 是基本 javax 中的 @Inject、@Scope 等注解基础上开发的，以及在 性能（Performance）、正确性（Correctness）、扩展性方面的特点，在中型和大型 App 中使用 Dagger2 收益更高。
* 介绍 Dagger2 中常用几个注解，以及基本使用方式，利用 multibinds 实现一个简单的 plugin。
* Dagger2 是编译器框架，利用 APT 技术生成辅助代码，供开发者在运行时调用，在合适的时机进行注入，Dagger2 没有使用反射等操作，所以性能比较高
* 分析了一个开源框架 MVPArms 对于 Dagger2 的应用以及分析了其基本原理
* 对于 Dagger2 的应用，官方也提供了一套 Android-Dagger2 框架，个人认为集成度有点高，理解起来相对困难，


### 参考

[Demo](https://github.com/RalfNick/DaggerPlugin.git)

[Dagger2](https://google.github.io/dagger)

[Dagger 官方文档](https://dagger.dev/dev-guide/)

[Using Dagger in Android apps](https://developer.android.com/training/dependency-injection/dagger-android)

[Dependency Injection guidance on Android — ADS 2019](https://medium.com/androiddevelopers/dependency-injection-guidance-on-android-ads-2019-b0b56d774bc2)

[MVPArms](https://github.com/JessYanCoding/MVPArms)

[Dagger 2. Part II. Custom scopes, Component dependencies, Subcomponents](https://proandroiddev.com/dagger-2-part-ii-custom-scopes-component-dependencies-subcomponents-697c1fa1cfc)

[轻松学，浅析依赖倒置（DIP）、控制反转(IOC)和依赖注入(DI)](https://juejin.cn/post/6844903487579357191#heading-9)

[轻松学，听说你还没有搞懂 Dagger2](https://frank909.blog.csdn.net/article/details/75578459)
