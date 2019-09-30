**此文翻译自[Create a Standalone Gradle plugin for Android - a step-by-step guide](https://afterecho.uk/blog/create-a-standalone-gradle-plugin-for-android-a-step-by-step-guide.html)，作者AfterEcho。**

下文中的“我”均指代原作者。

============正文分割线==========

本文中我会详细描述从零开始为安卓创建一个Gradle插件所需的步骤。从我写了我的第一篇文章[Monkeying around with gradle to generate code](https://afterecho.uk/blog/monkeying-around-with-gradle-to-generate-code.html)之后，我就有写这样一篇文章的想法了。而最近Annyce Davis在caster.io上写的[Gradle Plugin Basics](https://caster.io/episodes/gradle-plugin-basics/)鼓舞了我去落实这个想法。

我发现为Android Studio创建独立的gradle插件非常困难。虽然有很多文档，但是这些文档似乎总是缺少一些什么。你试过Google搜索[*create a gradle plugin for android*](https://www.google.com/search?q=create+a+gradle+plugin+for+android)吗？我搜过，但获得的结果用处不大。

甚至[gradle自己在这个问题上的文档](https://docs.gradle.org/current/userguide/custom_plugins.html)都帮助不大。关于创建独立工程的部分只提到了如何在`build.gradle`添加几行代码，以及编写properties文件，并没有提到如何进行build。而关于发布的部分假定了你了解发布并打算将插件公开发布。但是如果我只想生成插件并本地使用，或者只想尝试一下是否能正常运行呢？

如果想要作为Android Studio编译过程的一部分来运行呢？有没有办法？

本文中我不会覆盖你可能想实现的每一种情况。但是我做过的调研可以提供一个好的起点。

接下来正式开始。

# 0 假设

我会假定你已经设置好Android开发环境，包括安装Java、Android Studio和合适的编译工具。我认为这是个合理的假设，如果你是个Android开发新手，你不太可能一上来就想要编写gradle插件。

从你的角度而言，你得假定我并不精通gradle。我实在并不了解多少gradle相关知识。我所写的是我探索得出的结论，是对我来说有效的方法。如果你乐意的话，你也可以假设我是个帅到炸裂并且健谈的人。不过对于本文的目标而言，这些都不是必须的。

# 1 基本配置

这是一个我完全没法找到任何详细信息的步骤。而这只是基本的配置而已。

安装[groovy](http://www.groovy-lang.org/)和[gradle](http://gradle.org/)。在我的Mac上我是用[Homebrew](http://brew.sh/)安装的。你也可以直接下载。

Android Studio是定制化的IntellJ。但它并不适用于编写独立Gradle插件。如果你新建一个工程，这个工程会直接是Android工程。这很合理，因为它就是编写Android专用的。所以第一步，下载并安装IntelliJ。

从[https://www.jetbrains.com/idea/](https://www.jetbrains.com/idea/)获取社区版。撰写本文的时候版本是15.0.3。如果你在将来的某个时间阅读本文，而我又没有更新到新版的话，你就需要自己去搞定版本差异了。

社区版是免费的，而且能解决我们的需求。如果你使用过Android Studio，你就会觉得界面非常熟悉。（对用Eclipse的就只能说抱歉了。）

安装IntelliJ。所有的选项都默认即可。如果你确实想编写插件，确认你安装了groovy，gradle和maven。

安装完成之后，启动IntelliJ，点击*Create New Project*。如果一切正常的话，你会看到这样一个窗口：

![](http://upload-images.jianshu.io/upload_images/4032216-e6ba59752abd813a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果左边栏没有Gradle或者右边栏没有Groovy那么你可能缺少必要的插件。这样的话点击*Cancel*，从主屏幕点击*Configure -> Plugins*，安装插件，并再次新建工程。

如果Project SDK没有显示Java版本，点击*New...*按钮，选择你的Java安装路径。

勾选*Groovy*，点击*Next*。

在下一屏你需要填写GroupID和ArtifactId。尽管你可以任意命名，我们还是坚持传统。GroupID和你开发Android app一样使用公司域名，而ArtifactId使用插件的名称。（本文中我使用`com.afterecho`作为GroupID，`blogpost`作为ArtifactId。）暂时不要去管版本（目前是1.0-SNAPSHOT），点击*Next*。

下一屏中不要做任何更改。默认建议“Use default gradle wrapper”，我们有什么资格反对呢？点击*Next*。

在下一屏中你很可能也不需要做任何更改。除非你想改变工程路径。完成之后，点击*Finish*。

# 2 旅途开始

IntelliJ创建工程完成后，你会注意到工程中完全缺乏source目录。在工程树的顶部右击，选择*New*，然后*Directory*。

![](http://upload-images.jianshu.io/upload_images/4032216-5e671f3ab053ab53.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

输入路径名`src/main/groovy`。如果一切正常，groovy文件夹的颜色会与别的文件夹不同，指示这是一个source文件夹。


![](http://upload-images.jianshu.io/upload_images/4032216-1e1cfcff9624eba2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果你右击groovy文件夹你会看到，在*New*选项下，你可以新建package和class，而不是只能新建文件和目录了。

修改类的定义，增加`implements Plugin<Project>`，像这样：

        class BlogPlugin implements Plugin<Project> {
        }

随后你会发现`Project`类是未定义的。自动import在这里也是不行的。

# 3 编译配置

我们一定缺了什么。为什么无法import `Project`呢？据Gradle的文档所说，我们把：

    dependencies {
      compile gradleApi()
      compile localGroovy()
    }

添加到`build.gradle`。那很好。将这两行添加到`build.gradle`中已有的*dependecies*块。

BTW，当你首次加载build.gradle文件的时候IntelliJ可能会提示“You can configure Gradle wrapper to use distribution with sources. It will provide IDE with Gradle API/DSL documentation.”，这应该是OK的，点击确认就好了。如果你没看到这个，可能你的IntelliJ不大喜欢你。不过没关系，这不会影响我们继续编码。

这两行依赖添加好之后我们就可以回到Plugin类并且利用自动import功能了，对吗？有可能。

如果你在遇到上面所说的提示之前添加了依赖，并且点击了确定的话，那么就确实是可以了。可能会有多个类可供import，注意选包名是`org.gradle.api`的那个。

如果你的操作顺序不是这样的，那么这个类就依然是缺失的。我操作的时候发现只有一个`org.apache.tools.ant.Project`，不过这个是错的。

我比较习惯用Android Studio，在这个里面当你向`build.gradle`中添加依赖之后Android Studio会告诉你同步工程（Sync the project）。但是IntelliJ不会。你需要手动同步。每次向`build.gradle`中添加依赖都需要手动同步一次。

怎么做同步呢？点击*View*菜单，选*Tool Windows*，然后*Gradle*。

![](http://upload-images.jianshu.io/upload_images/4032216-f8ddfc1c24f2d5d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Gradle窗口打开之后，点击左上角的刷新按钮。

![](http://upload-images.jianshu.io/upload_images/4032216-fc9edc474165d8ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

你一定无法想象我在没法导入*Project*类这件事上浪费了多少时间。

导入*Project*和*Plugin*两个类之后你就可以看到类定义这一行下面红色的波浪线了。我们需要实现一个抽象方法。

# 4 apply插件自身

缺失的方法是`apply()`。使用IntelliJ的intention快捷键（CTRL + Return，ALT + Return，取决于你的操作系统和快捷键配置）自动添加方法。

这里有一点魔法。如果你在`apply()`中放一个`println`，你会在编译时看到信息输出发生在编译刚开始的时刻，在任何有意义的步骤之前。这显然是没有用的。

所以这是做什么用的呢？我们可以在这里创建Gradle Task，用于在合适的时间动作。

（为了简化剩下的工作，假定我创建的插件类叫作`BlogPlugin`，包名`com.afterecho.gradle`。注意插件的包名和GroupID并不强制相同。）

这是我们创建Gradle Task的地方。最简单的情况，我们创建一个仅仅在手动调用的时候才会执行的Task，暂时不去考虑嵌入编译流程的情况。在Android插件的task集中也有类似的Task，叫作`uninstallAll`，它会从所有已连接的设备上卸载所有build和flavor配置的app。在正常的编译过程中并不会执行`uninstallAll`，它只能手动运行。

我们建立一个名称为`showDevices`的Task，它简单地执行`adb devices`。

将以下代码填入`apply()`：

    @Override
    void apply(Project target) {
        def showDevicesTask = target.tasks.create("showDevices") << {
            def adbExe = target.android.getAdbExe().toString()
            println "${adbExe} devices".execute().text
        }
        showDevicesTask.group = "blogplugin"
        showDevicesTask.description = "Runs adb devices command"
    }

很简单。创建了一个task，名称`showDevices`。这个task会获取`adb`的路径，并执行一个command。工程（`target`）具有一个`android`属性（//译者注：这里的“属性”是指groovy中的`property`概念，下文均依此翻译），我们可以从这个属性获取一些有用的信息。在这里，我们可以获取`adb`的路径。

设置description是为了提供一个说明，在运行`./gradlew tasks`或者查看Android Studio中的功能提示时可以看到这个说明，让使用者了解这个task的功能，这样对使用者更加友好。

最终我们的类像这样：

    package com.afterecho.gradle

    import org.gradle.api.Plugin
    import org.gradle.api.Project

    class BlogPlugin implements Plugin<Project> {
        @Override
        void apply(Project target) {
            def showDevicesTask = target.tasks.create("showDevices") << {
                def adbExe = target.android.getAdbExe().toString()
                println "${adbExe} devices".execute().text
            }
            showDevicesTask.group = "blogplugin"
            showDevicesTask.description = "Runs adb devices command"
        }
    }

接下来我们如何使用这个插件呢？这又是一件让人沮丧的事。我尝试了很多种方法去生成一个Jar文件来放到目标工程内，但是都失败了。而我并不想将这个刚刚写出来的插件发布到Maven Central或者JCenter上去。

# 5 使用repo

你可以在本地文件系统创建类似maven的repo。Yay！

在你的机器上某个地方创建一个目录。这里我们需要再次修改`build.gradle`，将以下代码添加到结尾：

    apply plugin: 'maven'

    uploadArchives {
        repositories {
            mavenDeployer {
                repository(url: uri('repo'))
            }
        }
    }

将'repo'替换为你的目录的全路径。如果你像我一样仅仅写一个'repo'，它会进入plugin工程下的'repo'目录。这也有个好处，你可以直接在IntelliJ中查看它。

不要忘记调出Gradle工具窗口并点击刷新。将窗口保留在打开状态，你会看到一个新的task：`uploadArchives`。

![](http://upload-images.jianshu.io/upload_images/4032216-0ee07c55e981f3c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

双击`uploadArchives`来运行它。首次运行的时候你会在console中看到信息说无法找到metadata。不用担心，metadata会自动创建，后续的执行就不会出现提示了。

如果你打开repo目录，你会发现如下文件：

![](http://upload-images.jianshu.io/upload_images/4032216-6be9c7dd014f2851.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

这就是你自己的maven repo了。:)

可以看到每个文件都带有时间戳。这是因为你指定了这一次build是一个快照（snapshot）。（还记得刚开始的时候默认的版本号吗？）每次你运行`uploadArchives`任务的时候，都会创建一组时间戳不同的文件。

为了避免把事情搞复杂，在`build.gradle`文件里把版本号后面的"-SNAPSHOT"去掉，删除repo目录下的文件，然后重新执行`uploadArchives`。

# 6 搜索你的插件

我们需要另一个文件来告诉Gradle我们的插件在哪个类里面。回到我们的工程结构，右击"main"，并创建目录`resources/META-INF/gradle-plugins`。

![](http://upload-images.jianshu.io/upload_images/4032216-d7bb22af9cfa984e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个目录下创建一个新文件。文件名为你给plugin取的名字带上后缀".properties"。

不推荐你把名字叫做*com.android.application*。我这里用名字`com.afterecho.blogplugin.properties`。

在这个文件里添加一行，`implementation-class='你实现的`Plugin<Project>`的子类的全限定名。我这里的值就是：

    implementation-class=com.afterecho.gradle.BlogPlugin

# 7 发布

再次运行`uploadArchives`任务。现在我们可以暂时离开IntelliJ，去到Android Studio的环境下了。

创建一个新的Android工程，或者打开一个你原有的工程。打开*工程（project）*层级的`build.gradle`，在`buildscript`块内的`repositories`块下添加

    maven {
        url uri('/path/to/the/repo/directory/created/earlier')
    }

（将路径改为你之前放置repo的路径。）你的工程就可以到这个目录下寻找插件了。

在`buildscript`内的`dependencies`块下添加

    classpath 'com.afterecho.gradle:blogpost:1.0'

这一句声明了安卓编译依赖于我们的插件。这里的名称与我们刚开始创建插件的时候相关。第一个冒号前的是*GroupId*，两个冒号之间的是*ArtifactId*。

如果你的工程是新创建的，这时候文件应该长这样：

    buildscript {
        repositories {
            jcenter()
            maven {
                url uri('/tmp/repo')
            }
        }
        dependencies {
            classpath 'com.android.tools.build:gradle:1.5.0'
            classpath 'com.afterecho:blogpost:1.0'
            // NOTE: Do not place your application dependencies here; they belong
            // in the individual module build.gradle files
        }
    }

    allprojects {
        repositories {
            jcenter()
        }
    }

现在打开*模块（module）*级别的`build.gradle`，在`apply plugin: com.android.application`之后添加

    apply plugin: 'com.afterecho.blogplugin'

这里是填写PluginId的地方——还记得properties文件的名字吗？

总结一下几个概念：
 · GroupId
 · ArtifictId
 · PluginId

PluginId可以和GroupId以及ArtifactId完全无关，但是一般来说不应当这样命名。

Android Studio这个时候应当会要求Sync了。执行它。但是这时候结果有点让人扫兴：并没有特别的现象发生。

# 8 最终步骤

没有现象的原因是虽然我们创建了一个task，但是并没有调用它。你可以从Android Studio里的Gradle工具窗口执行它。

打开"blogplugin"块，可以看到`showDevices`。如果你在上面悬停鼠标，你可以看到我们之前设定的任务描述。双击`showDevices`，静待窗口输出结果。

你也可以从工程目录下的命令行运行task。执行：

    ./gradlew tasks

你可以看到"Blogplugin tasks"模块下有我们的"showDevices"，并会显示描述。再执行

    ./gradlew showDevices

这样我们就成功调用了我们的第一个gradle插件。我知道这还没有实际的作用，但是这是一个开端。在后续的博客里我会继续深入了解如何编写具有实际作用的gradle插件。
