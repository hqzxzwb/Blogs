# Android编译时修改字节码——以Logcat自动打TAG为例

## 背景

虽然Java语言提供了良好的封装特性，但是受限于语法，仍然有一些需求实现起来比较麻烦。在编译时对字节码进行操作可以很大程度上突破语言的限制，实现“对程序编程”。

例如，最近经常讨论的“面向切面编程”就可以通过这一方案来实现：所谓切面，反映到程序里就是具有一定特征的代码，因此在编译时对字节码进行分析和修改可以方便地实现这一需求。

## 本文的例子

一般来说，我们给Logcat指定TAG会以如下方式：

    public class SomeClass {
        private static final String TAG = "SomeClass";
        
        void someMethod() {
            Log.d(TAG, "some log ...");
        }
    }

为了避免给每个类来这么一个常量，可以稍微做一点封装：

    public class LogUtil {
        public static void d(Object target, String log) {
            Log.d(target.getClass().getSimpleName(), log);
        }
    }

调用时写作：

    LogUtil.d(this, "some log...");

还有一种做法，是通过追溯stacktrace，获得调用者的类名。这种做法可以省去一个"this"。代码如下：

    public static void d(String log) {
        StackTraceElement[] elements = Thread.currentThread().getStackTrace();
        String tag = "";
        if (elements.length >= 2) {
            tag = elements[1].getClassName();
        }
        Log.d(tag, log);
    }

这样调用的时候可以写作：

    LogUtil.d("some log...");

以上两种方法有一个明显的问题：进行代码混淆之后，这些日志将变得不知所云。此外，实时获取stacktrace是一个相对比较耗费性能的动作，用在日志上不太合适。

本篇文章我们将通过实现一个gradle plugin，在编译的时候去修改字节码，以达到给logcat自动添加TAG的目的。

之前我翻译的一篇文章[【译】一步一步为Android实现Gradle插件](http://www.jianshu.com/p/95b45d97c00c)中，介绍了如何创建一个gradle plugin工程，并将其应用到编译过程中。本篇中我将略过创建工程与部署插件的过程，专注于代码实现。

编译过程中我们需要操纵字节码，这里我选用的工具是[ASM](http://asm.ow2.org/)。ASM相关的资料网上并不是特别多，但好在其文档和教程均比较详细，可以到官网下载教程深入研究一下。

ASM的API设计称作“访问者模式”：在实际操作的过程中，我们会使用`ClassReader`来读入原有的字节码，ASM库会对字节码进行扫描，并针对其中的每一个字节码指令调用其API中对应的函数。我们可以通过覆盖这些函数，将这些函数调用替换为我们想要的指令所对应的函数，而我们覆盖之后的函数访问流程会通过`ClassWriter`转换为对应的字节码，并写入到目标文件里。

那么接下来我们来动手写代码。

## Android工程部分

首先我们写一个LogUtil类。这里面不会写有效的代码，真正的代码在编译的时候通过插件生成。

    package com.xxx.util;

    public class LogUtil {
        public static int d(String log) {//留意这里返回int，为了与系统API统一。
            throw new RuntimeException("Stub!");//这个不会真的被调用到
        }

        //所有其他的logcat函数，例如v，i，w，e等。
    }

随后我们会这样调用：

    LogUtil.d("some log...");

## 插件部分

### Transform API

[Transform API](http://tools.android.com/tech-docs/new-build-system/transform-api)是Android Gradle Plugin在1.5版本引入的功能。它提供了操作编译生成的文件的接口，包括 .class文件和 .jar文件。我们这里只关心自己编写的类，因此只处理 .class文件是妥当的。而安卓自己编译的很多流程也已经是用内部的Transform来实现。在Android Studio中查看gradle task列表，可以看到很多名称中带有Transform的任务。

我们在gradle plugin工程中先实现一个Plugin：

    import org.gradle.api.Plugin
    import org.gradle.api.Project

    class InjectPlugin implements Plugin<Project> {
        @Override
        void apply(Project target) {
            target.android.registerTransform(new MyTransform())
        }
    }

这里的语言用的是Groovy。`MyTransform`类待实现。我们把这一个Transform注册到编译流程中去。

    class MyTransform extends Transform {
        @Override
        String getName() {//这个名称会用于生成的gradle task名称
            return "LogInject"
        }

        @Override
        Set<QualifiedContent.ContentType> getInputTypes() {
            //接受输入的类型。我们这里只处理Java类。
            return [QualifiedContent.DefaultContentType.CLASSES]
        }

        @Override
        Set<QualifiedContent.Scope> getScopes() {
            //作用范围。这里我们只处理工程中编写的类。
            return [QualifiedContent.Scope.PROJECT]
        }

        @Override
        boolean isIncremental() {//是否支持增量。这里暂时不实现增量。
            return false
        }

        @Override
        void transform(TransformInvocation ti) throws TransformException, InterruptedException, IOException {
            //待实现
        }
    }

 上面是对Transform的一些基础配置。如果找不到`Transform`这个父类，那么你需要在插件工程的gradle中配置一下依赖。

    compile 'com.android.tools.build:gradle-api:2.2.3'//版本请自选

接下来实现`transform`方法：

    @Override
    void transform(TransformInvocation ti) throws TransformException, InterruptedException, IOException {
        TransformOutputProvider outputProvider = ti.outputProvider
        //获取输出路径
        def outDir = outputProvider.getContentLocation("inject", outputTypes, scopes, Format.DIRECTORY)

        outDir.deleteDir()
        outDir.mkdirs()

        ti.inputs.each {
            //directoryInputs就是class文件所在的目录。如果需要处理jar文件，那么也要对it.jarInputs进行处理。本文中不需要。
            it.directoryInputs.each {
                int pathBitLen = it.file.toString().length()

                it.file.traverse {
                    if (!it.isDirectory()) {
                        File file = it
                        File outputFile = new File(outDir, "${f.toString().substring(pathBitLen)}")
                        outputFile.getParentFile().mkdirs()
                        byte[] classFileBuffer = file.bytes
                        //doTransfer待实现
                        outputFile.bytes = doTransfer(classFileBuffer)
                    }
                }
            }
        }
    }


也就是遍历输入文件，转换后写入对应的输出文件。如果你打算实现增量编译的话，可以通过方法`DirectoryInput.getChangedFiles`来获取文件变化的状况，并作出相应的处理。本文中不再讨论。

接下来我们看`doTransfer`的实现：

    private static byte[] doTransfer(byte[] input) {
        String s = new String(input)
        int flags = 0
        if (s.contains(LogcatVisitor.UTIL_CLASS)) {//UTIL_CLASS="com/xxx/LogUtil"
            //一个小trick：由于被调用的方法和类会被放入常量池，我们可以直接忽略字节码中不含有字符串"com/xxx/LogUtil"的文件
            flags = 1
        }
        if (flags) {
            ClassReader reader = new ClassReader(input)
            ClassWriter writer = new ClassWriter(reader, ClassWriter.COMPUTE_MAXS)//COMPUTE_MAXS会让我们免于手动计算局部变量数和方法栈大小
            ClassVisitor cv = writer
            //LogcatVisitor会承载我们核心的修改字节码的任务
            cv = new LogcatVisitor(cv)
            reader.accept(cv, 8)
            return writer.toByteArray()
        } else { //无需转换
            return input
        }
    }

### ASM

接下来我们来实现`LogcatVisitor`。

首先将ASM引入工程。我这里的做法是把源码下载了直接放到工程里，当然这个并不讲究。

其次明确一下我们要做什么：

(1)由于每处logcat的tag是不同的，虽然我们代码里没有留tag的参数，但是我们还是要想办法把它传进来。怎么办？对LogUtil里面的每个方法生成一个影子方法，这个影子方法与原方法相比多一个参数。
(2)这个参数怎么加进去比较好？作为最后一个参数加入比较好。因为方法调用的时候，最后一个参数在栈顶部，也就是说它入栈的字节码和调用方法的字节码是紧贴着的，我们可以轻松地在调用方法的位置上加入这个参数。
(3)插入影子方法后，原来的方法可不可以删掉？我之前是删掉的，并且删掉之后直接运行并没有出现问题。但是后来混淆的时候出现了一个问题：原来的方法虽然实际上没有被调用了，但是它还存在于调用者类的常量池里，所以proguard仍然会尝试去找原有的方法，当然它肯定找不到，会报错。所以，我们不删除原方法。
(4)对于调用者，我们需要在调用日志方法之前，在栈顶推入一个字符串（当前类名），然后将调用原方法修改为调用影子方法。

下面来看`LogcatVisitor`实现。这是个用JAVA写的类。

    import org.objectweb.asm.ClassVisitor;
    import org.objectweb.asm.MethodVisitor;
    import org.objectweb.asm.Type;
    import org.objectweb.asm.tree.MethodNode;
    
    import static org.objectweb.asm.Opcodes.*;
    
    public class LogcatVisitor extends ClassVisitor {
        static final String UTIL_CLASS = "com/xxx/LogUtil";
        private boolean injectUtil;//标记正在对Util类进行注入，还是对调用者进行注入
        private String simpleClassName;
    
        public LogcatVisitor(ClassVisitor cv) {
            super(ASM5, cv);
        }
    
        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(version, access, name, signature, superName, interfaces);
            System.out.println("Doing logcat injection on class " + name);
            //这里的name是当前类名
            if (name.equals(UTIL_CLASS)) {
                injectUtil = true;
            } else {
                simpleClassName = name.substring(name.lastIndexOf('/') + 1);
            }
        }
    
        @Override
        public MethodVisitor visitMethod(int access, String name, String desc, String signature, String[] exceptions) {
            MethodVisitor mv;
            if (injectUtil && name.length() == 1) {//这里是对Util类的操作
                //super调用结果用于返回。即保留原有的方法。
                mv = super.visitMethod(access, name, desc, signature, exceptions);
    
                //给方法描述符在开头添加一个参数。这个描述符用于调用android/util/Log.*，我们和它恰好相差一个tag参数。
                String targetDesc = "(Ljava/lang/String;" + desc.substring(1);
                int i = desc.indexOf(')');
                //这里是给方法添加最后一个参数为String。这个描述符用于生成影子方法。
                desc = desc.substring(0, i) + "Ljava/lang/String;" + desc.substring(i);
    
                //MethodNode用于生成影子方法
                MethodNode mn = new MethodNode();
                mn.visitCode();//方法开始
                int localVarIndex = Type.getArgumentTypes(targetDesc).length - 1;
                mn.visitVarInsn(ALOAD, localVarIndex);//我们引入的最后一个参数是tag，但调用android/util/Log.*的第一个参数是tag，需要最先入栈。
                for (int j = 0; j < localVarIndex; j++) {//其余参数依次入栈。
                    mn.visitVarInsn(ALOAD, j);
                }
                //调用android/util/Log中的方法
                mn.visitMethodInsn(INVOKESTATIC, "android/util/Log", name, targetDesc, false);
                //android/util/Log返回的值直接返回出去
                mn.visitInsn(IRETURN);
                mn.visitMaxs(0, 0);
                mn.visitEnd();
                //影子方法添加到类中
                mn.accept(cv.visitMethod(access, name, desc, signature, exceptions));
            } else {//这里是对调用者的操作
                mv = super.visitMethod(access, name, desc, signature, exceptions);
                //这里我们修改了MethodVisitor再返回，即修改了这个方法
                mv = new MethodVisitor(ASM5, mv) {    
                    @Override
                    public void visitMethodInsn(int opcode, String owner, String name, String desc, boolean itf) {
                        //覆盖方法调用的过程
                        if (opcode == INVOKESTATIC && owner.equals(UTIL_CLASS)) {
                            //发现在调用LogUtil中的方法。接下来我们要入栈一个字符串，并替换为影子方法。
                            mv.visitLdcInsn(simpleClassName);//入栈类名
                            int i = desc.indexOf(')');
                            desc = desc.substring(0, i) + "Ljava/lang/String;" + desc.substring(i);//方法描述符替换为影子方法
                        }
                        super.visitMethodInsn(opcode, owner, name, desc, itf);
                    }
                };
            }
            return mv;
        }
    }


至此，一个自动打tag的LogUtil就实现完毕了。

## 总结

本文通过一个简单的例子描述了实现安卓编译插件的主要思路和步骤。简单起见，关于ASM和字节码还有很多细节没有详细描述。感兴趣的话可以通过翻阅文档来了解详情。
