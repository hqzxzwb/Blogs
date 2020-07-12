# Kotlin 语法的背后

Kotlin 带有许多方便高效的语法设计，但又与 Java 完美兼容。本文探究 Kotlin 的一些语法特性在编译后的样子。

Intellij idea 中提供了一项功能，可以快速查看 Kotlin 文件在编译后的字节码以及等价的 Java 代码，这为我们了解 Kotlin 各种语法背后的机制提供了极大的便利。这一功能的路径是：菜单栏 -> Tools -> Kotlin -> Show Kotlin Bytecode。自然，基于 Intellij Idea 的 Android Studio 也具有这项功能。

## 顶级函数与顶级变量

在 Java 中，函数和变量必须包含在类当中。而在 Kotlin 当中，有“顶级”(top-level)的概念，函数和变量可以声明在文件最外层：

```Kotlin
// Foo.kt

fun barFun() {
    println("bar")
}

val bar: String = "bar"
```

我们查看其等价的 Java 代码：

```Java
public final class FooKt {
   @NotNull
   private static final String bar = "bar";

   public static final void barFun() {
      String var0 = "bar";
      boolean var1 = false;
      System.out.println(var0);
   }

   @NotNull
   public static final String getBar() {
      return bar;
   }
}
```

可见，函数和变量实际上仍然处于一个类当中。一般来说，这个类的名字为“文件名+Kt”。但我们也可以通过注解`@JvmName`指定文件中的顶级成员处于哪个类中：

```Kotlin
// Foo.kt
@file:JvmName("Foo1")

// Functions and properties
```

这时，该文件中的顶级成员将处于一个名为`Foo1`的类中：

```Java
@JvmName(
   name = "Foo1"
)
public final class Foo1 {
    // Functions and properties
}
```

## 扩展(Extensions)

Kotlin 支持为已存在的类型声明扩展函数与扩展属性。

```Kotlin
// Foo.kt

fun String.toUri(): Uri {
    return Uri.parse(this)
}

val String.uri: Uri
    get() = Uri.parse(this)
```

等价的 Java 代码是：

```Java
public final class FooKt {
   @NotNull
   public static final Uri toUri(@NotNull String $this$toUri) {
      Intrinsics.checkParameterIsNotNull($this$toUri, "$this$toUri");
      Uri var10000 = Uri.parse($this$toUri);
      Intrinsics.checkExpressionValueIsNotNull(var10000, "Uri.parse(this)");
      return var10000;
   }

   @NotNull
   public static final Uri getUri(@NotNull String $this$uri) {
      Intrinsics.checkParameterIsNotNull($this$uri, "$this$uri");
      Uri var10000 = Uri.parse($this$uri);
      Intrinsics.checkExpressionValueIsNotNull(var10000, "Uri.parse(this)");
      return var10000;
   }
}
```

实际上，函数和属性的接收者(receiver)是作为函数的第一个参数出现的。属性成为了 getter 函数。当然，若属性是可变的，将会额外存在一个 setter 函数。这里函数和属性声明在顶级，因此其对应的函数是静态的。若声明在类的内部，其对应的函数就可能不是静态的。

## 默认参数(Default arguments)

Kotlin 提供了默认参数功能，该功能可以代替重载的一部分用法，使用上也很简便。同时，Kotlin 还提供了一个注解`@JvmOverloads`来方便 Java 代码调用带默认参数的函数。

```Kotlin
// Foo.kt

@JvmOverloads
fun bar(v1: Int, v2: String = "v2", v3: Long, v4: String? = "v4"): String {
    return "$v1$v2$v3$v4"
}
```

```Java
public final class FooKt {
   @JvmOverloads
   @NotNull
   public static final String bar(int v1, @NotNull String v2, long v3, @Nullable String v4) {
      Intrinsics.checkParameterIsNotNull(v2, "v2");
      return v1 + v2 + v3 + v4;
   }

   // $FF: synthetic method
   public static String bar$default(int var0, String var1, long var2, String var4, int var5, Object var6) {
      if ((var5 & 2) != 0) {
         var1 = "v2";
      }

      if ((var5 & 8) != 0) {
         var4 = "v4";
      }

      return bar(var0, var1, var2, var4);
   }

   @JvmOverloads
   @NotNull
   public static final String bar(int v1, @NotNull String v2, long v3) {
      return bar$default(v1, v2, v3, (String)null, 8, (Object)null);
   }

   @JvmOverloads
   @NotNull
   public static final String bar(int v1, long v3) {
      return bar$default(v1, (String)null, v3, (String)null, 10, (Object)null);
   }
}
```

可以看出，其方案的核心在于隐藏的`bar$default`方法。该方法除了接受原函数的所有参数之外，另外接受一个`int`类型参数和一个`Object`类型的参数。根据该方法内的代码，可以推测，`int`类型参数的作用是标示调用者未传递哪些参数，是一个典型的标记位模式。在调用处，这个值会由编译器根据实际的代码传递了哪些参数来生成。最后一个`Object`类型的参数，调用方只会传递`null`，设计这样一个参数的原因不明。

另外两个名称为`bar`的方法只在标记了`@JvmOverloads`时才会生成，是提供给 Java 代码调用的，Kotlin 代码的编译产物并不会使用它们。

如果参数个数超过了32个，那么`bar$default`的倒数第二个参数的位数将不够用，这时会发生什么？写一个33参数的函数试验一下：

Kotlin 代码略。

```Java
public static String bar$default(String var0, String var1, String var2, String var3, String var4, String var5, String var6, String var7, String var8, String var9, String var10, String var11, String var12, String var13, String var14, String var15, String var16, String var17, String var18, String var19, String var20, String var21, String var22, String var23, String var24, String var25, String var26, String var27, String var28, String var29, String var30, String var31, String var32, int var33, int var34, Object var35) {
    // 略
}
```

末尾出现了两个`int`类型的标记位参数，于是就够用了。

## 解构(Destructuring)语法

Kotlin 中有时可以对多个变量同时赋值，如：

```
// Foo.kt

fun bar() {
    val v1 = "a" to "b"
    val (a, b) = v1 // a = "a", b = "b"

    val v2 = listOf(1, 2, 3)
    val (m, n, o) = v2 // m = 1, n = 2, o = 3

    val foo = Foo("zhangsan", 18)
    val (name, age) = foo // name = "zhangsan", age = 18
    
    val (c1, c2) = "abcde" // c1 = 'a', c2 = 'b'
}

data class Foo(
    val name: String,
    val age: Int
)

operator fun String.component1() = this[0]

operator fun String.component2() = this[1]
```

仍然是观察等价的 Java 代码：

```Java
public final class FooKt {
   public static final void bar() {
      Pair v1 = TuplesKt.to("a", "b");
      String var1 = (String)v1.component1();
      String b = (String)v1.component2();

      List v2 = CollectionsKt.listOf(new Integer[]{1, 2, 3});
      boolean var9 = false;
      int var4 = ((Number)v2.get(0)).intValue();
      var9 = false;
      int var5 = ((Number)v2.get(1)).intValue();
      var9 = false;
      int o = ((Number)v2.get(2)).intValue();

      Foo foo = new Foo("zhangsan", 18);
      String var8 = foo.component1();
      int age = foo.component2();

      String var12 = "abcde";
      char var10 = component1(var12);
      char c2 = component2(var12);
   }

   public static final char component1(@NotNull String $this$component1) {
      Intrinsics.checkParameterIsNotNull($this$component1, "$this$component1");
      return $this$component1.charAt(0);
   }

   public static final char component2(@NotNull String $this$component2) {
      Intrinsics.checkParameterIsNotNull($this$component2, "$this$component2");
      return $this$component2.charAt(1);
   }
}
```

从`bar`方法的内容不难发现，解构的过程基本上是在调用形如`componentN`的函数。其中对`String`的解构不是天然支持的，是我们通过编写`operator fun String.componentN`获得的功能。

但有一项例外，对`List`对象的结构直接调用了`get`方法，而没有出现`componentN`。这是因为对`List`声明的`componentN`函数皆是内联的，因此这些函数没有在编译产物中出现。Kotlin 中相关代码如下：

```Kotlin
public inline operator fun <T> List<T>.component1(): T {
    return get(0)
}

public inline operator fun <T> List<T>.component2(): T {
    return get(1)
}

public inline operator fun <T> List<T>.component3(): T {
    return get(2)
}

public inline operator fun <T> List<T>.component4(): T {
    return get(3)
}

public inline operator fun <T> List<T>.component5(): T {
    return get(4)
}
```

而在`Foo`类型的代码中，我们并没有声明`componentN`这样的函数，但代码里仍然出现了这样的调用。这是因为解构是 data class 默认提供的一项功能，编译器会自动为 data class 添加这一系列函数。

## 函数类型

Kotlin 支持声明函数类型的函数参数、属性和变量等。在 Java 中，相似的功能往往需要声明接口并将接口作为参数、变量类型来实现。考虑如下代码：

```Kotlin
// Foo.kt

fun bar(block: (String) -> Int): Int {
    val s = "Hello"
    return block(s)
}
```

对应的 Java 代码：

```Java
public final class FooKt {
   public static final int bar(@NotNull Function1 block) {
      Intrinsics.checkParameterIsNotNull(block, "block");
      String s = "Hello";
      return ((Number)block.invoke(s)).intValue();
   }
}
```

`block`参数的类型是一个叫`Function1`的东西。调用`block`这个函数变量，实际上是在调用`Function1`的`invoke`方法。那么`Function1`是什么呢？

根据省略掉的 import 中的信息，找到了一个`Functions.kt`文件，主要内容是这样的：
```Kotlin
// Functions.kt

/** A function that takes 0 arguments. */
public interface Function0<out R> : Function<R> {
    /** Invokes the function. */
    public operator fun invoke(): R
}
/** A function that takes 1 argument. */
public interface Function1<in P1, out R> : Function<R> {
    /** Invokes the function with the specified argument. */
    public operator fun invoke(p1: P1): R
}
/** A function that takes 2 arguments. */
public interface Function2<in P1, in P2, out R> : Function<R> {
    /** Invokes the function with the specified arguments. */
    public operator fun invoke(p1: P1, p2: P2): R
}

// 若干类似代码

/** A function that takes 22 arguments. */
public interface Function22<in P1, in P2, in P3, in P4, in P5, in P6, in P7, in P8, in P9, in P10, in P11, in P12, in P13, in P14, in P15, in P16, in P17, in P18, in P19, in P20, in P21, in P22, out R> : Function<R> {
    /** Invokes the function with the specified arguments. */
    public operator fun invoke(p1: P1, p2: P2, p3: P3, p4: P4, p5: P5, p6: P6, p7: P7, p8: P8, p9: P9, p10: P10, p11: P11, p12: P12, p13: P13, p14: P14, p15: P15, p16: P16, p17: P17, p18: P18, p19: P19, p20: P20, p21: P21, p22: P22): R
}
```

这是一系列预定义的接口，使用函数作为类型的元素实际上就是使用这些接口作为类型的了。不难猜想，Lambda 表达式实际上是实现了这些接口之一的类。

但接口数量是有限的，这里最后一个接口接受22个参数。如果声明具有23个参数的函数类型，会如何？仍然做试验：

Kotlin 代码略。

```Java
public final class FooKt {
   public static final int bar(@NotNull FunctionN block) {
      Intrinsics.checkParameterIsNotNull(block, "block");
      return 0;
   }
}
```

参数类型变成了`FunctionN`。来看这个接口的源码：
```Kotlin
// FunctionN.kt
interface FunctionN<out R> : Function<R>, FunctionBase<R> {
    /**
     * Invokes the function with the specified arguments.
     *
     * Must **throw exception** if the length of passed [args] is not equal to the parameter count returned by [arity].
     *
     * @param args arguments to the function
     */
    operator fun invoke(vararg args: Any?): R

    /**
     * Returns the number of arguments that must be passed to this function.
     */
    override val arity: Int
}
```

它的`invoke`函数接受的参数是可变数量的。类似 Java 的可变数量参数，它实际接受的是一个数组对象。调用该函数时，编译器生成的代码会将参数先组织到一个数组当中，再传递给`block`。

## 本地函数(Local functions)

Kotlin 提供一种在函数内声明函数的语法。这种语法提供了比 private 更精细的封装范围，有利于提高代码的可读性。以下代码实现了一个简易的二分查找：

```Kotlin
// Foo.kt

fun binarySearch(arr: IntArray, target: Int): Boolean {
    fun binarySearch(arr: IntArray, target: Int, start: Int, end: Int): Boolean {
        if (start > end) {
            return false
        }
        if (start == end) {
            return arr[start] == target
        }
        val middleIndex = (start + end) / 2
        val middleValue = arr[middleIndex]
        return when {
            middleValue == target -> true
            middleValue < target -> binarySearch(arr, target, middleIndex + 1, end)
            else -> binarySearch(arr, target, start, middleIndex - 1)
        }
    }

    return binarySearch(arr, target, 0, arr.lastIndex)
}
```

内部用于递归的函数作用域局限于外部函数之内，同一类中的其他成员不能访问该内部函数。那么我们将它转换为 Java 代码看看：

```Java
public final class FooKt {
  public static final boolean binarySearch(@NotNull int[] arr, int target) {
    Intrinsics.checkParameterIsNotNull(arr, "arr");
    FooKt$binarySearch$1 $fun$binarySearch$1 = FooKt$binarySearch$1.INSTANCE;
    return $fun$binarySearch$1.invoke(arr, target, 0, ArraysKt.getLastIndex(arr));
  }

  static final class FooKt$binarySearch$1 extends Lambda implements Function4<int[], Integer, Integer, Integer, Boolean> {
    public static final FooKt$binarySearch$1 INSTANCE = new FooKt$binarySearch$1();

    // Synthetic
    public final Object invoke(Object var1, Object var2, Object var3, Object var4) {
      return this.invoke((int[]) var1, (int) var2, (int) var3, (int) var4);
    }

    public final boolean invoke(@NotNull int[] arr, int target, int start, int end) {
      Intrinsics.checkParameterIsNotNull(arr, "arr");
      if (start > end)
        return false; 
      if (start == end)
        return (arr[start] == target); 
      int middleIndex = (start + end) / 2;
      int middleValue = arr[middleIndex];
      return (middleValue == target) ? true : ((middleValue < target) ? invoke(arr, target, middleIndex + 1, end) : invoke(arr, target, start, middleIndex - 1));
    }
    
    FooKt$binarySearch$1() {
      super(4);
    }
  }
}
```

这个声明在内部的函数对应着一个内部类`FooKt$binarySearch$1`。这个类实现了接口`Function4`，我们之前已经了解到这是 Kotlin 用于代表拥有四个参数的函数的接口。理论上，这相对于直接使用私有函数是存在额外开销的，这里在可读性和运行性能上需要做一点权衡取舍。
