# Kotlin库的二进制兼容性维护

Kotlin为我们带来了很多开发上的便利。其中各类语法糖发挥了巨大的作用。这些语法糖往往包含了自动生成的函数和类，应当注意到，这些内容会被使用者隐式地调用，同样是[二进制兼容性](https://docs.oracle.com/javase/specs/jls/se7/html/jls-13.html)的一部分。但由于它们往往是隐藏的，维护它们的兼容性并不是一个直观的工作，在开发时十分容易发生疏漏。本文就此类情况进行讨论，并尝试通过引入一定的编码规范来解决它。

## 1 带有默认参数的函数

考虑如下一个带有默认参数的函数

```Kotlin
@JvmOverloads
fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
    // Implementation
}
```

其二进制等同的Java代码如下：

```Java
@JvmOverloads
public static final void bar(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4)

// $FF: synthetic method
public static void bar$default(String var0, int var1, String var2, List var3, int var4, Object var5)

// 只针对JvmOverloads生成
@JvmOverloads
public static final void bar(@NotNull String p1, int p2, @NotNull String p3)

// 只针对JvmOverloads生成
@JvmOverloads
public static final void bar(@NotNull String p1, int p2)
```

当一份Kotlin调用代码只给出一部分参数、另一部分参数隐式地使用默认值时，隐藏的函数`bar$default`就会被使用。

当一份Java调用代码只给出一部分参数时，根据`@JvmOverloads`生成的、重载的两个`bar`函数就会被使用。

接下来，我们尝试着对这份代码作出变动。

### 在末尾添加一个带有默认值的参数

```diff
 @JvmOverloads
-fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
+fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"), p5: Any? = null) {
     // Implementation
 }
```

这个变动是源代码兼容的。新增的参数带有默认值，只要将调用者代码重新编译一次，原本的调用代码将会自动地使用这个默认值。但是它并不是二进制兼容的。其二进制等同Java代码如下：

```diff
 @JvmOverloads
-public static final void bar(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4)
+public static final void bar(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4, @Nullable Object p5)

 // $FF: synthetic method
-public static void bar$default(String var0, int var1, String var2, List var3, int var4, Object var5)
+public static void bar$default(String var0, int var1, String var2, List var3, Object var4, int var5, Object var6)

+@JvmOverloads
+public static final void bar(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4)

 @JvmOverloads
 public static final void bar(@NotNull String p1, int p2, @NotNull String p3)

 @JvmOverloads
 public static final void bar(@NotNull String p1, int p2)
```

可以看到，原本使用了`bar$default`的代码此时就会发生`NoSuchMethodError`。原本使用了`bar`函数本体的代码，仍然可以照常运行，因为`@JvmOverloads`生成了一个与其签名相同的函数。

如果我们不使用`@JvmOverloads`，那么使用了`bar$default`和使用`bar`函数本体的代码均会发生`NoSuchMethodError`。

我们尝试保留原有的函数来维护二进制兼容性：

```diff
 @JvmOverloads
 fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
-    // Implementation
+   bar(p1, p2, p3, p4, null)
 }
+
+@JvmOverloads
+fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"), p5: Any? = null) {
+    // Implementation
+}
```

这回得到一个编译错误：

```
Platform declaration clash: The following declarations have the same JVM signature (bar(Ljava/lang/String;I)V): 
@JvmOverloads public fun bar(p1: String, p2: Int, p3: String = ..., p4: List<String> = ...): Unit defined in root package in file Test.kt
@JvmOverloads public fun bar(p1: String, p2: Int, p3: String = ..., p4: List<String> = ..., p5: Any? = ...): Unit defined in root package in file Test.kt
Platform declaration clash: The following declarations have the same JVM signature (bar(Ljava/lang/String;ILjava/lang/String;)V): 
@JvmOverloads public fun bar(p1: String, p2: Int, p3: String = ..., p4: List<String> = ...): Unit defined in root package in file Test.kt
@JvmOverloads public fun bar(p1: String, p2: Int, p3: String = ..., p4: List<String> = ..., p5: Any? = ...): Unit defined in root package in file Test.kt
Platform declaration clash: The following declarations have the same JVM signature (bar(Ljava/lang/String;ILjava/lang/String;Ljava/util/List;)V): 
@JvmOverloads public fun bar(p1: String, p2: Int, p3: String = ..., p4: List<String> = ...): Unit defined in root package in file Test.kt
@JvmOverloads public fun bar(p1: String, p2: Int, p3: String = ..., p4: List<String> = ..., p5: Any? = ...): Unit defined in root package in file Test.kt
```

显然，两个函数会根据`@JvmOverloads`生成相同签名的函数，Kotlin的编译器不知道如何处理这一情况，因而产生了一个编译错误。

我们试试不给新增的函数添加`@JvmOverloads`：

```diff
 @JvmOverloads
 fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
-    // Implementation
+    bar(p1, p2, p3, p4, null)
 }
+
+fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"), p5: Any? = null) {
+    // Implementation
+}
```

编译错误解决了，其等同的Java代码如下：

```diff
 @JvmOverloads
 public static final void bar(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4)

 // $FF: synthetic method
 public static void bar$default(String var0, int var1, String var2, List var3, int var4, Object var5)

 @JvmOverloads
 public static final void bar(@NotNull String p1, int p2, @NotNull String p3)

 @JvmOverloads
 public static final void bar(@NotNull String p1, int p2)

+public static final void bar(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4, @Nullable Object p5)
+
+// $FF: synthetic method
+public static void bar$default(String var0, int var1, String var2, List var3, Object var4, int var5, Object var6)
```

这应该可以完美地维持二进制兼容性。

但这份代码仍然有一点瑕疵：带有默认值的参数，它的默认值需要在原来的函数里和新增的函数里分别维护。将来若需要变更这个默认值，很可能发生遗漏。

在很多时候，我们可以假设默认值将来不会变更。但这并不总是合适的假设。

可以尝试下面的实践：

```diff
 @JvmOverloads
-fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
+fun bar(p1: String, p2: Int, p3: String = barP3DefaultValue(), p4: List<String> = barP4DefaultValue()) {
+  bar(p1, p2, p3, p4, barP5DefaultValue())
 }
+
+fun bar(p1: String, p2: Int, p3: String = barP3DefaultValue(), p4: List<String> = barP4DefaultValue(), p5: Any? = barP5DefaultValue()) {
+}
+
+private fun barP3DefaultValue() = "123"
+private fun barP4DefaultValue() = listOf("abc")
+private fun barP5DefaultValue() = null
```

这样就可以通过修改几个提供默认值的函数的实现来统一变更默认值。但是显而易见，这份代码麻烦了许多。开发者可能需要根据实际情况对此进行取舍。

若原函数没有使用`@JvmOverloads`注解，则可以采用以下方式维护二进制兼容性：

```diff
+@Deprecated(message = "For binary compatibility", level = DeprecationLevel.HIDDEN)
 fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
-    // Implementation
+    bar(p1, p2, p3, p4, null)
 }
+
+fun bar(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"), p5: Any? = null) {
+    // Implementation
+}
```

其中，`@Deprecated`以及`DeprecationLevel.HIDDEN`是必要的，否则调用处可能会无法选定重载函数，会发生编译错误。

## 2 带有默认参数的构造器

考虑如下构造器：

```Kotlin
class Foo @JvmOverloads constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"))
```

其等同Java代码如下：

```Java
public final class Foo {
   @JvmOverloads
   public Foo(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4)

   // $FF: synthetic method
   public Foo(String var1, int var2, String var3, List var4, int var5, DefaultConstructorMarker var6)

   @JvmOverloads
   public Foo(@NotNull String p1, int p2, @NotNull String p3)

   @JvmOverloads
   public Foo(@NotNull String p1, int p2)
}
```

这与带参数默认值的函数是十分相似的，因此我们面临的问题也是十分相似的。

那么我们仿照带参数默认值函数的解决方案，为构造器添加一个参数：

```diff
-class Foo @JvmOverloads constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"))
+class Foo constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"), p5: Any? = null) {
+    @JvmOverloads
+    constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) : this(p1, p2, p3, p4, null)
+}
```

等同的Java代码如下：

```diff
 public final class Foo {
+   public Foo(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4, @Nullable Object p5)
+
+   // $FF: synthetic method
+   public Foo(String var1, int var2, String var3, List var4, Object var5, int var6, DefaultConstructorMarker var7)

   @JvmOverloads
   public Foo(@NotNull String p1, int p2, @NotNull String p3, @NotNull List p4)

   // $FF: synthetic method
   public Foo(String var1, int var2, String var3, List var4, int var5, DefaultConstructorMarker var6)

   @JvmOverloads
   public Foo(@NotNull String p1, int p2, @NotNull String p3)

   @JvmOverloads
   public Foo(@NotNull String p1, int p2)
}
```

可以看到，这种处理方式同样适合于带参数默认值的构造器。

对于没有使用`@JvmOverloads`注解的构造器，可以如下处理：

```diff
-class Foo @JvmOverloads constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) {
+class Foo constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc"), p5: Any? = null) {
+    @Deprecated("For binary compatibility.", level = DeprecationLevel.HIDDEN)
+    constructor(p1: String, p2: Int, p3: String = "123", p4: List<String> = listOf("abc")) : this(p1, p2, p3, p4, null)
 }
```

## data class

Kotlin 中的 data class 是快速实现POJO的利器。但是它也有一些二进制兼容性的隐患。

我们考察一个简单的data class：

```Kotlin
data class Foo(
    val s1: String,
    val s2: String,
    val i1: Int
)
```

其二进制等同的Java代码如下：

```Java
public final class Foo {
   @NotNull
   public final String getS1()

   @NotNull
   public final String getS2()

   public final int getI1()

   public Foo(@NotNull String s1, @NotNull String s2, int i1)

   @NotNull
   public final String component1()

   @NotNull
   public final String component2()

   public final int component3()

   @NotNull
   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1)

   // $FF: synthetic method
   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, Object var5)

   @NotNull
   public String toString()

   public int hashCode()

   public boolean equals(@Nullable Object var1)
}
```

除了显式声明的构造器、字段及其getter之外，Kotlin为我们生成了以下内容：

- 解构函数`component1`、`component2`和`component3`
- 拷贝函数`copy`和`copy$default`
- POJO实现`toString`、`hashCode`和`equals`

以下代码展示了解构函数和拷贝函数在何时被使用：

```Kotlin
fun testFoo() {
    val foo1 = Foo("s1", "s2", 1)
    val (s1, s2, i1) = foo1
    val foo2 = foo1.copy(s2 = "newS2")
}
```

其等同的Java代码如下：

```Java
public static final void testFoo() {
    Foo foo1 = new Foo("s1", "s2", 1);
    String s1 = foo1.component1();
    String s2 = foo1.component2();
    int i1 = foo1.component3();
    Foo foo2 = Foo.copy$default(foo1, (String)null, "newS2", 0, 5, (Object)null);
}
```

可以看到，解构函数`component1`、`component2`和`component3`会被解构语法`val (s1, s2, i1) = foo1`使用，而拷贝函数和一般的带默认参数的函数一样，可以被显式地调用，并且具有一个用于支持默认参数功能的隐藏实现`copy$default`。

接下来，我们尝试对这个data class作出版本变更。

### 调整参数声明顺序

在 Java POJO 中，调整字段的声明顺序往往是无害的。但在data class当中，情况却并非如此。

我们将s1和i1的声明位置交换：

```diff
  data class Foo(
-     val s1: String,
+     val i1: Int,
      val s2: String,
-     val i1: Int
+     val s1: String
  )
```

其等效的Java代码发生相应的变化：

```diff
 public final class Foo {
-   @NotNull
-   public final String getS1()
+   public final int getI1()
 
    @NotNull
    public final String getS2() {
       return this.s2;
    }
 
-   public final int getI1()
+   @NotNull
+   public final String getS1()
 
-   public Foo(@NotNull String s1, @NotNull String s2, int i1)
+   public Foo(int i1, @NotNull String s2, @NotNull String s1)
 
-   @NotNull
-   public final String component1()
+   public final int component1()
 
    @NotNull
    public final String component2()
 
-   public final int component3()
+   @NotNull
+   public final String component3()
 
    @NotNull
-   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1)
+   public final Foo copy(int i1, @NotNull String s2, @NotNull String s1)
 
    // $FF: synthetic method
-   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, Object var5)
+   public static Foo copy$default(Foo var0, int var1, String var2, String var3, int var4, Object var5)
 
    @NotNull
    public String toString()
 
    public int hashCode()
 
    public boolean equals(@Nullable Object var1)
 }
```

可以观察到，字段和getter的声明顺序发生了变化，不过这一变化并不破坏兼容性。除此之外，构造函数、解构函数、拷贝函数的签名均发生了变化，二进制兼容性均被破坏。

我们很少遇到一定有必要调整字段声明顺序的情况。因此，对于这一问题，只要小心地维护声明顺序与第一个版本相同即可，不需要特别的应对方案。

### 增加字段

在末尾添加一个字段，作出如下变更：
```diff
data class Foo(
    val s1: String,
    val s2: String,
-   val i1: Int
+   val i1: Int,
+   val i2: Int
)
```

对应的Java代码发生如下变化：

```diff
 public final class Foo {
    @NotNull
    public final String getS1()
 
    @NotNull
    public final String getS2()
 
    public final int getI1()

+   public final int getI2()
+
-   public Foo(@NotNull String s1, @NotNull String s2, int i1)
+   public Foo(@NotNull String s1, @NotNull String s2, int i1, int i2)
 
    @NotNull
    public final String component1()
 
    @NotNull
    public final String component2()
 
    public final int component3()
 
+   public final int component4()
+
    @NotNull
-   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1)
+   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1, int i2)
 
    // $FF: synthetic method
-   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, Object var5)
+   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, int var5, Object var6)
 
    @NotNull
    public String toString()
 
    public int hashCode()
 
    public boolean equals(@Nullable Object var1)
 }
 
```

在这里，除了明显的构造函数签名变更，拷贝函数`copy`和`copy$default`的二进制兼容性也被破坏了。

我们试试手动将原来的copy函数添加回来：

```diff
 data class Foo(
     val s1: String,
     val s2: String,
-    val i1: Int
-)
+    val i1: Int,
+    val i2: Int
+) {
+    constructor(s1: String, s2: String, i1: Int) : this(s1, s2, i1, 0)
+
+    @Deprecated("", level = HIDDEN) // For binary compatibility.
+    fun copy(s1: String = this.s1, s2: String = this.s2, i1: Int = this.i1): Foo {
+        return copy(s1 = s1, s2 = s2, i1 = i1, i2 = this.i2)
+    }
+}
```

再对比一下对应Java代码的变化：

```diff
 public final class Foo { 
    @NotNull
    public final String getS1()
 
    @NotNull
    public final String getS2()
 
    public final int getI1()

+   public final int getI2()
+
-   public Foo(@NotNull String s1, @NotNull String s2, int i1)
+   public Foo(@NotNull String s1, @NotNull String s2, int i1, int i2)
+
+   public Foo(@NotNull String s1, @NotNull String s2, int i1)

    @NotNull
    public final String component1()
 
    @NotNull
    public final String component2()
 
    public final int component3()
 
+   public final int component4()
+
    @NotNull
-   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1)
+   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1, int i2)
 
    // $FF: synthetic method
-   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, Object var5)
+   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, int var5, Object var6)
 
+   @NotNull
+   public final Foo copy(@NotNull String s1, @NotNull String s2, int i1)
+
+   // $FF: synthetic method
+   public static Foo copy$default(Foo var0, String var1, String var2, int var3, int var4, Object var5)

    @NotNull
    public String toString()
 
    public int hashCode()
 
    public boolean equals(@Nullable Object var1)
 }
```

原有的`copy`和`copy$default`函数与我们手写的`copy`函数签名相同，行为也一致，这样，我们就手动地维护了这个变动的二进制兼容性。
