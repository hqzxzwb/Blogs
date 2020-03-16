# R8与proguard的策略差异

[R8](https://developer.android.com/studio/build/shrink-code)是谷歌推出的代码混淆/优化工具，用以代替proguard。它的混淆和优化策略与proguard有所不同，其中一些在我们由proguard向R8迁移的时候可能会带来问题，这篇文章主要关注这一类差异以及解决方案。

## 1. R8无实例假设

### 优化策略
如果一个类，在代码中未被显式地实例化，即没有其他代码调用了这个类的构造方法，那么R8会作出运行时不会出现该类的实例的假设。据此假设，R8会删除该类的所有非静态方法。

### 潜在问题
一个常见的场景是，我们编写的某个类只用于填充某个注解的值，然后在运行时由提供该注解的框架使用反射进行实例化，这时R8进行的这项优化就会导致运行时崩溃。

以下是一个使用Gson当中的`JsonAdapter`的例子：

```Kotlin
// 这是一个自定义的解析器
class Foo : JsonDeserializer<Bar>, JsonSerializer<Bar> {
    fun deserialize(...): Bar {
        // Some code
    }

    fun serialize(...): JsonElement {
        // Some code
    }
}

// 这是一个Bean
class Bean {
    @JsonAdapter(Foo::class)
    var bar: Bar? = null
}
```

在这个例子当中，`Foo`这个类不会被显式地实例化，只会被Gson使用反射实例化。R8无法探测反射实例化，因而会假设`Foo`类不会被实例化，从而移除`deserialize`和`serialize`两个方法，导致运行时发生崩溃。

```Kotlin
// 混淆后的Foo
// 这段伪代码实际上无法通过编译，但R8在字节码层面做优化，可以制造出这种残缺的类。
class Foo : JsonDeserializer<Bar>, JsonSerializer<Bar> {
    // 实例方法deserialize被移除

    // 实例方法serialize被移除
}
```

### 解决方案
我们知道，`JsonAdapter`注解可以接受`TypeAdapterFactory`、`JsonSerializer`、`JsonDeserializer`和`TypeAdapter`这几种类型的子类型作为参数，那么只需要针对这几个类及其子类编写keep规则即可：

```
-keep,allowobfuscation class * implements com.google.gson.TypeAdapterFactory
-keep,allowobfuscation class * implements com.google.gson.JsonSerializer
-keep,allowobfuscation class * implements com.google.gson.JsonDeserializer
-keep,allowobfuscation class * implements com.google.gson.TypeAdapter
```

这几条规则已被添加到Gson的[proguard.cfg](https://github.com/google/gson/blob/master/examples/android-proguard-example/proguard.cfg)文件中。

## 2. R8合并接口与实现类

### 优化策略
若一个接口只存在一种实现，R8会假设所有该接口的实例，都是这个类的实例，并将所有的接口方法都移动到实现类当中。若接口类型没有其他引用，则会将接口类型删除。以下是一个示例：

```Kotlin
interface IFoo {
    fun bar()
}

// Suppose this is the only implementation of IFoo.
class FooImpl : IFoo {
    override fun bar() {
        // Impl of bar
    }
}

class Main {
    fun test(foo: IFoo) {
        foo.bar()
    }
}
```

优化后的等效代码：
```Kotlin
// interface IFoo is removed

class FooImpl {
    fun bar() {
        // Impl of bar
    }
}

class Main {
    fun test(foo: FooImpl) {
        foo.bar()
    }
}
```

### 潜在问题

R8在此假设接口`IFoo`只存在唯一的实现类。若是我们使用动态代理来生成`IFoo`的实例，那么就会破坏这一假设，进而带来问题。

考虑如下代码：

```Kotlin
interface IFoo {
    fun bar()
}

// Suppose this is the only implementation of IFoo.
class FooImpl : IFoo {
    override fun bar() {
        println("This is FooImpl")
    }
}

// This is a pool holding implementations of each interface.
object InterfacePool {
    val implMap = mutableMapOf<Class<*>, MutableList<Any?>>()

    fun <T> addImpl(interfaceClass: Class<T>, impl: T) {
        val impls = implMap.getOrPut(interfaceClass) { arrayListOf() }
        impls.add(impl)
    }

    // The returned implementation invokes every added implementation of  given interface.
    fun <T> getImpl(interfaceClass: Class<T>): T {
        return Proxy.newProxyInstance(interfaceClass.classLoader, arrayOf(interfaceClass), object : InvocationHandler {
            override fun invoke(proxy: Any?, method: Method, args: Array<out Any>?): Any? {
                val impls = implMap[interfaceClass] ?: return null
                impls.forEach { method.invoke(it, *(args ?: arrayOf())) }
                return null
            }
        }) as T
    }
}

class Main {
    fun test() {
        InterfacePool.addImpl(IFoo::class.java, FooImpl())
        InterfacePool.addImpl(IFoo::class.java, FooImpl())
        InterfacePool.getImpl(IFoo::class.java).bar()
    }
}
```

优化之后，部分代码变成了这样：

```Kotlin
interface IFoo {
    // bar 函数被移除
}

// Suppose this is the only implementation of IFoo.
class FooImpl : IFoo {
    // bar 不再 override 任何函数
    fun bar() {
        println("This is FooImpl")
    }
}

object InterfacePool {
    ...
    // 这个类基本没有变化，省略
}

class Main {
    fun test() {
        InterfacePool.addImpl(IFoo::class.java, FooImpl())
        InterfacePool.addImpl(IFoo::class.java, FooImpl())
        // 注意！！这里发生了变化，会导致ClassCastException。
        (InterfacePool.getImpl(IFoo::class.java) as FooImpl).bar()
    }
}
```

可以注意到，`IFoo`接口的`bar`方法被移除。但由于代码中存在`IFoo::class.java`的调用，`IFoo`本身并没有被移除。而调用`InterfacePool.getImpl`得到一个`IFoo`对象之后，R8会假设这个实例一定是`FooImpl`的实例，并进行类型强制转换。但这其实是错误的，因为`getImpl`返回的是一个动态代理对象，而不是`FooImpl`的实例。这里就会发生`ClassCastException`，引起崩溃。

### 解决方案

这类问题的解决方案需要具体问题具体分析。就上面的示例代码而言，我们可以限定`InterfacePool`只接受某个接口的子接口类型，这样就可以统一编写混淆规则。以下是改造过的代码示例：

```Kotlin
interface PoolInterface

interface IFoo : PoolInterface {
    fun bar()
}

object InterfacePool {
    val implMap = mutableMapOf<Class<*>, MutableList<Any?>>()

    fun <T : PoolInterface> addImpl(interfaceClass: Class<T>, impl: T) {
        // Stays the same
    }

    // The returned implementation invokes every added implementation of  given interface.
    fun <T : PoolInterface> getImpl(interfaceClass: Class<T>): T {
        // Stays the same
    }
}

```

并添加规则

```
-keep,allowobfuscation class * implements PoolInterface
```

### 一点不确定的问题

理论上，此策略也可能应用于类之间的继承关系。即，若类A继承了类B，且类B不存在其他子类，且类B没有被实例化，那么可以假设：所有类B的对象都是类A的对象。R8实际上有没有作此优化，我目前没有进行探索。实际上，我们只要有了这个思路，遇到问题时自然可以解决之，因此我没有花更多精力探索这个优化的精确边界条件。

## 3. 方法外联

### 优化策略

R8会分析代码中频繁出现的重复代码，并将这类代码抽象为函数，以减少编译产物中字节码的数量。具体的介绍可以参考[这篇文章](https://jakewharton.com/r8-optimization-method-outlining/)。

例如，我们经常会写字符串拼接的代码：

```Kotlin
fun test1(): String {
    return this.a + "a" + "c"
    // 实际上的字节码是
    // return StringBuilder().append(this.a).append("a").append("c").toString()
}

fun test2(): String {
    return "f" + this.b + "g"
    // 实际上的字节码是
    // return StringBuilder().append("f").append(this.b).append("g").toString()
}
```

优化之后，它可能变成这样：

```Kotlin
fun test1(): String {
    return Outline.f1(this.a, "a", "c")
}

fun test2(): String {
    return Outline.f1("f", this.b, "g")
}

// 这是R8生成的，统一存放外联函数的类。
object Outline {
    fun f1(s1: String, s2: String, s3: String): String {
        return StringBuilder().append(s1).append(s2).append(s3).toString()
    }
}

```

当这类重复代码足够多时，外联策略就会为我们可观地降低apk的大小。

### 潜在问题

一般来说，这个优化在代码逻辑层面不会带来任何问题。但是，我仍然遇到了一个与此有关的问题。这个问题与热修复密切相关。

这里我使用我熟悉的Tinker框架做案例分析。其他热修复框架，按照原理不同，可能会、也可能不会遇到这个问题，相信读者可以自行判断。

我们知道，Tinker框架通过修改`ClassLoader`来实现热修复。修改`ClassLoader`之后，我们在应用中加载的就是新包当中的类。但在修改`ClassLoader`之前，Tinker需要做一些准备工作，我们把这个阶段叫作启动阶段，而其后的阶段称为运行阶段。启动阶段执行的代码在`com.tencent.tinker.loader`下面，这部分代码一定是通过默认的类加载器来加载的。也就是说，启动阶段永远是执行原包的代码，而不会执行热修复新包的代码。

当R8进行优化之后，`com.tencent.tinker.loader`下的代码和我们的业务代码就有可能共同引用了R8生成的`Outline`这个类。那么在启动阶段，原包版本的`Outline`类可能率先被加载。进入运行阶段后，业务代码调用`Outline`类时，就会使用已经被加载的、原包版本的`Outline`类。由于原包和底包有所不同，这时往往会产生崩溃。

### 解决方案

我们通过编译后修改apk解决了这一问题。我们先把`com.tencent.tinker.loader`下的代码不加混淆编译成dex，再反编译成smali。在生成apk之后，我们将apk进行反编译，得到smali代码。将其中`com.tencent.tinker.loader`下的代码替换成预先生成的smali，然后再回编译。这样最终生成的apk当中，`com.tencent.tinker.loader`就不可能引用`Outline`这个类了。

## 4. 成员变量未写入

### 优化策略

当一个类某个成员变量（Field）在整个app中没有检测不到赋值代码的时候，R8会假设该成员变量的值只可能是null（或者0等原始类型的初始值），并将读取该成员变量的代码替换为null或0等初始值。

### 潜在问题

如果字段只被反射写入的话，该优化就会被破坏。

### 解决方案

当存在涉及某个成员变量的keep规则时，R8不会针对该变量作上述优化。

这种场景在反射式的Json解析当中常常遇到。例如，当我们使用Gson时，很多类的成员变量只会被Gson反射赋值。

在大多数情况下，我们的项目中已经对使用Gson解析的类做了完整的keep，这时不必担心上述问题。

但是，如果一个类，它的每个字段均使用`@SerializedName`注解标记，而且没有任何针对它的keep规则，那么这个类虽然在proguard混淆后可以正常使用，但在R8混淆后却会发生字段值被当做null的问题。

为此，Gson在[proguard.cfg](https://github.com/google/gson/blob/master/examples/android-proguard-example/proguard.cfg)中增加了以下规则：

```
# Prevent R8 from leaving Data object members always null
-keepclassmembers,allowobfuscation class * {
  @com.google.gson.annotations.SerializedName <fields>;
}
```
