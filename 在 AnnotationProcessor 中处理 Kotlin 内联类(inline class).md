Kotlin 的[内联类](https://kotlinlang.org/docs/reference/inline-classes.html)带来了一种兼顾封装和性能的编程方式。它可以将一种类型按照业务需要包裹成另一种类型，却在运行时用原本的类型实现，避免了额外的运行时开销。

但是，Kotlin 在处理内联类时，为了避免以内联类为参数的函数和以原类型为参数的函数发生签名冲突，在编译时对以内联类为参数的函数做了[特殊处理](https://kotlinlang.org/docs/reference/inline-classes.html#mangling)。
```Kotlin
inline class UInt(val x: Int)

// Represented as 'public final void compute(int x)' on the JVM
fun compute(x: Int) { }

// Also represented as 'public final void compute(int x)' on the JVM!
fun compute(x: UInt) { }
```
为了解决签名冲突的问题，使用内联类作为参数的函数被重命名，函数名被添加了一种稳定的哈希码后缀。因此，函数 `compute(x: UInt)` 的 Java 表达类似于 `public final void compute-<hashcode>(int x)`，这就解决了签名冲突的问题。

这类带有`-`的函数名称在 Java 中是不合法的，因此这类函数无法被 Java 代码调用。除此之外，这类函数也不会出现在 AnnotationProcessor 处理的输入当中。这两个事实都给我们编写 AnnotationProcessor 带来了挑战。

=== To be continued ===
