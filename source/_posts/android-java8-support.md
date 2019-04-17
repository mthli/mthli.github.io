---
title : Android 兼容 Java 8 的原理
date  : 2019.04.16
tags  :
    - Android
    - Java
---

*本文译自 Jake Wharton 的博客 [Android's Java 8 Support](https://jakewharton.com/androids-java-8-support/).*

我虽然在家办公了几年，但人们对 Android 不同 Java 版本的支持问题的吐槽我也有所耳闻。每年的 Google I/O 的提问环节你会发现我也在咨询这个问题；在其他相关的会议中，也会或多或少地提及。显然这也是一个复杂的问题，我们需要明确我们在讨论什么。对 Java 的支持可以有很多维度，例如语法糖、字节码、工具链、新的 API 、新的 JVM 实现等。

通常大家谈论到 Android 对 Java 8 的支持，都是在说语法糖。所以我们就来看看 Android 的工具链是如何处理 Java 8 的语法糖的。

## Lambda

Java 8 标志性的语法糖就是加入了 lambda 表达式的支持，相比之前使用匿名类更加简洁清爽：

```Java
class Java8 {
  interface Logger {
    void log(String s);
  }

  public static void main(String... args) {
    sayHi(s -> System.out.println(s));
  }

  private static void sayHi(Logger logger) {
    logger.log("Hello!");
  }
}
```

在使用 javac 编译之后，使用 dx 直接运行会报错：

```Text
$ javac *.java

$ ls
Java8.java Java8.class Java8$Logger.class

$ $ANDROID_HOME/build-tools/28.0.2/dx --dex --output . *.class
Uncaught translation error: com.android.dx.cf.code.SimException:
  ERROR in Java8.main:([Ljava/lang/String;)V:
    invalid opcode ba - invokedynamic requires --min-sdk-version >= 26
    (currently 13)
1 error; aborting
```

这是因为 lambda 的实现使用到了 Java 7 新增加的字节码 invokedynamic. 正如报错信息提示的那样，Android 对这个字节码的支持是在 API 26 以上才实现的 ———— 这竟然有点不可思议。相反地，一个名为 desugaring 的编译流程被用来将 lambda 转换为所以 API 都兼容的形式。

## Desugaring 的历史

Desugaring 的历史可以说是非常精彩，它的目标始终如一：新的语法糖可以运行在所有设备上。

最开始，我们使用一款名为 [Retrolambda](https://github.com/luontola/retrolambda) 来实现相关的功能。它使用 JVM 的内建机制，在运行时而不是编译时将 lambda 转换为类的实现。生成新的类很容易增加方法数，但是如果权衡利弊这些成本还是可以接受的 (but work on the tool over time reduced the cost to something reasonable).

随后 Android 的工具链团队[发布了一款新的编译器](https://android-developers.googleblog.com/2014/12/hello-world-meet-our-new-experimental.html)，称其可以将 Java 8 的语法糖脱糖的同时还兼备更好的性能。这款编译器是基于 Eclipse 的 Java 编译器开发的，但是目标是 Dalvik 字节码而不是 Java 字节码。这个版本的 Java 8 的脱糖实现代价高昂，并且使用率低、性能差，与其他工具链不兼容。

当上述的新的编译器最终被弃用时（感谢），一款新的将 Java 字节码翻译到 Java 字节码的脱糖转换器[被集成到了 Android Gradle Plugin 中](https://android-developers.googleblog.com/2017/04/java-8-language-features-support-update.html)，它实际上源自 Google 自己的构建工具 Bazel. 其脱糖过程挺高效的，但是性能表现仍然不是很理想。事实上它是一个渐进式的解决方案，不停地在寻找更好的解决方案。

[随后 D8 发布了](https://android-developers.googleblog.com/2017/08/next-generation-dex-compiler-now-in.html)，被用来取代传统的 dx 工具链，承诺在 dex 过程中脱糖而不是使用标准的 Java 字节码做转换。相对于 dx 而言，D8 在性能上取得了巨大的成功，并且带来了更高效的脱糖字节码。从 Android Gradle Plugin 3.1 版本开始，D8 成为了默认的 dex 工具，在 3.2 版本开始负责脱糖。

## D8

使用 D8 将之前的例子编译为 Dalvik 就成功了：

```Text
$ java -jar d8.jar \
    --lib $ANDROID_HOME/platforms/android-28/android.jar \
    --release \
    --output . \
    *.class

$ ls
Java8.java Java8.class Java8$Logger.class classes.dex
```

我们可以使用 Android SDK 中提供的 dexdump 来看看 D8 是如何将 lambda 脱糖的。这个工具真的会输出很多玩意儿，但我们只看相关的内容：

```Text
$ $ANDROID_HOME/build-tools/28.0.2/dexdump -d classes.dex
[0002d8] Java8.main:([Ljava/lang/String;)V
0000: sget-object v0, LJava8$1;.INSTANCE:LJava8$1;
0002: invoke-static {v0}, LJava8;.sayHi:(LJava8$Logger;)V
0005: return-void

[0002a8] Java8.sayHi:(LJava8$Logger;)V
0000: const-string v0, "Hello"
0002: invoke-interface {v1, v0}, LJava8$Logger;.log:(Ljava/lang/String;)V
0005: return-void
…
```

如果你之前不了解字节码（不论是 Dalvik 还是其他什么的）你也不用担心 ———— 大多数都很容易被理解。

在第一个代码块中，我们的 main 方法，在名为 Java8$1 的类中，字节序 0000 返回了一个静态的 INSTANCE 引用。考虑到源码中并没有包含名为 Java8$1 的类，我们可以推断这是一个脱糖过程中生成的类。main 方法的字节码中也没有包含任何 lambda 的痕迹，所以一定是在 Java8$1 中干了些什么。

字节序 0002 接着调用到了 sayHi 和 INSTANCE. sayHi 需要一个 Java8$Logger 参数，看起来 Java8$1 实现了相关的接口。我们同样可以在 D8 的输出中进行验证：

```Text
Class #2            -
  Class descriptor  : 'LJava8$1;'
  Access flags      : 0x1011 (PUBLIC FINAL SYNTHETIC)
  Superclass        : 'Ljava/lang/Object;'
  Interfaces        -
    #0              : 'LJava8$Logger;'
```

SYNTHETIC 标志着相关的类是被生成的；并且在 Interfaces 也包含了 Java8$Logger.

于是 Java8$1 现在就代表了 lambda. 如果你去查看它的 log 方法的实现，你可能会期望发现缺失的 lambda 代码块：

```Text
…
[00026c] Java8$1.log:(Ljava/lang/String;)V
0000: invoke-static {v1}, LJava8;.lambda$main$0:(Ljava/lang/String;)V
0003: return-void
…
```

可是什么都没有哦。相反地，它调用了 Java8 这个类中的名为 lambda$main$0 的静态方法。同样地，源码中没有包含这个方法，但是它存在于字节码中：

```Text
…
    #1              : (in LJava8;)
      name          : 'lambda$main$0'
      type          : '(Ljava/lang/String;)V'
      access        : 0x1008 (STATIC SYNTHETIC)
[0002a0] Java8.lambda$main$0:(Ljava/lang/String;)V
0000: sget-object v0, Ljava/lang/System;.out:Ljava/io/PrintStream;
0002: invoke-virtual {v0, v1}, Ljava/io/PrintStream;.println:(Ljava/lang/String;)V
0005: return-void
```

SYNTHETIC 标志着这个方法是被生成的。并且它的字节码包含了 lambda 的代码块：调用 System.out.println(). lambda 代码块存在于原来的类的内部的原因在于，它可能需要访问该类的私有成员变量，而生成的类却是访问不到的。

通过上述描述你应该就能理解脱糖的原理了。当然在 Dalvik 字节码中看到它们可能会有点密集而令人生畏（密集恐惧症）。

## 代码转换

为了更好地理解脱糖的原理，我们可以在代码层面进行一层转换。这并不意味着脱糖实际上是这样工作的，但是有助于我们学习并理解字节码中发生的事情。

再一次地，我们从最初的例子开始：

```Java
class Java8 {
  interface Logger {
    void log(String s);
  }

  public static void main(String... args) {
    sayHi(s -> System.out.println(s));
  }

  private static void sayHi(Logger logger) {
    logger.log("Hello!");
  }
}
```

首先，lambda 代码块被处理为与 main 同级的，package-private 的方法：

```Diff
   public static void main(String... args) {
-    sayHi(s -> System.out.println(s));
+    sayHi(s -> lambda$main$0(s));
   }
+
+  static void lambda$main$0(String s) {
+    System.out.println(s);
+  }
```

接着，一个实现了目标接口的类被生成了，它拥有一个调用 lambda 的方法：

```Diff
   public static void main(String... args) {
-    sayHi(s -> lambda$main$0(s));
+    sayHi(new Java8$1());
   }
@@
 }
+
+class Java8$1 implements Java8.Logger {
+  @Override public void log(String s) {
+    Java8.lambda$main$0(s);
+  }
+}
```

最后，因为这个 lambda 不需要捕获任何状态 (because the lambda doesn’t capture any state), 一个存储在 INSTANCE 中的静态单例被创建了出来：

```Diff
   public static void main(String... args) {
-    sayHi(new Java8$1());
+    sayHi(Java8$1.INSTANCE);
   }
@@
 class Java8$1 implements Java8.Logger {
+  static final Java8$1 INSTANCE = new Java8$1();
+
   @Override public void log(String s) {
```

这样就脱糖出了一份在所有 API 版本都能使用的代码：

```Java
class Java8 {
  interface Logger {
    void log(String s);
  }

  public static void main(String... args) {
    sayHi(Java8$1.INSTANCE);
  }

  static void lambda$main$0(String s) {
    System.out.println(s);
  }

  private static void sayHi(Logger logger) {
    logger.log("Hello!");
  }
}

class Java8$1 implements Java8.Logger {
  static final Java8$1 INSTANCE = new Java8$1();

  @Override public void log(String s) {
    Java8.lambda$main$0(s);
  }
}
```

不过如果你去看了对应 lambda 生成的类的 Dalvik 字节码的话，你会发现其中并没有类似名为 Java8$1 的东西。真实情况下的名称实际上类似于 -$$Lambda$Java8$QkyWJ8jlAksLjYziID4cZLvHwoY. 至于为什么这样命名，并且带来的好处，就需要另写一篇文章来解释了...

## 原生的 Lambda

当我们使用 dx 去尝试编译包含 lambda 的 Java 字节码为 Dalvik 字节码时，报错信息会提示你说至少得在 API 26 及其以上版本才能使用：

```Text
$ $ANDROID_HOME/build-tools/28.0.2/dx --dex --output . *.class
Uncaught translation error: com.android.dx.cf.code.SimException:
  ERROR in Java8.main:([Ljava/lang/String;)V:
    invalid opcode ba - invokedynamic requires --min-sdk-version >= 26
    (currently 13)
1 error; aborting
```

如果你用 D8 配合 --min-api 26 参数编译的话，它会假定你将使用原生的 lambda 实现而不会进行脱糖：

```Text
$ java -jar d8.jar \
    --lib $ANDROID_HOME/platforms/android-28/android.jar \
    --release \
    --min-api 26 \
    --output . \
    *.class
```

但如果你去查看生成的 .dex 文件，你还是会发现类似 -$$Lambda$Java8$QkyWJ8jlAksLjYziID4cZLvHwoY 的类被生成了，这也许是 D8 的 bug?

为了探究为什么脱糖过程总是在运行，我们需要深入 Java8 这个类的字节码中一探究竟：

```Text
$ javap -v Java8.class
class Java8 {
  public static void main(java.lang.String...);
    Code:
       0: invokedynamic #2, 0   // InvokeDynamic #0:log:()LJava8$Logger;
       5: invokestatic  #3      // Method sayHi:(LJava8$Logger;)V
       8: return
}
…
```

上述输出已经被我简化成可读的形式，在 main 方法中你会看到 invokedynamic 位于索引 0. 对应的字节码中的第二个参数 0 是与启动引导的方法挂钩的，该方法在代码被第一次运行时会定义一些行为。在输出文件的底部有列出这些引导方法的列表：

```Text
…
BootstrapMethods:
  0: #27 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(
                        Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;
                        Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;
                        Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)
                        Ljava/lang/invoke/CallSite;
    Method arguments:
      #28 (Ljava/lang/String;)V
      #29 invokestatic Java8.lambda$main$0:(Ljava/lang/String;)V
      #28 (Ljava/lang/String;)V
```

在我们的例子中，引导方法是位于 java.lang.invoke.LambdaMetafactory 这个类中的名为 metafactory 的方法。[这个方法存在于 JDK 中](https://docs.oracle.com/javase/8/docs/api/java/lang/invoke/LambdaMetafactory.html)，它的职责是在运行时中即时创建 lambda 相关的匿名类，这与 D8 在编译时干的事情类似。

如果你有看过 java.lang.invoke 相关的 [Android 官方文档](https://developer.android.com/reference/java/lang/invoke/package-summary) 或 [AOSP 的源码](https://android.googlesource.com/platform/libcore/+/master/ojluni/src/main/java/java/lang/invoke/)，你会发现 Android 运行时里并没有这个类。Android VM 有与 invokedynamic 相同效果的字节码支持，但是 JDK 内建的 LambdaMetafactory 并不可用。

## Method References

作为 lambda 的补充，方法引用这个语法糖同样被引入到 Java 8 中，使得创建 lambda 去指向一个已有的方法的操作变得高效。

在我们的 logger 的例子中，lambda 表达式直接调用了已有的 System.out.println() 方法，我们可以简单把它转换为方法引用的形式简化代码：

```Diff
   public static void main(String... args) {
-    sayHi(s -> System.out.println(s));
+    sayHi(System.out::println);
   }
```

使用 javac 编译后，再用 D8 处理，我们会发现与之前的 lambda 有一处显著的不同。当我们查看生成的 Dalvik 字节码时，会发现生成的 lambda 类的代码块被改变了：

```Text
[000268] -$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM.log:(Ljava/lang/String;)V
0000: iget-object v0, v1, L-$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM;.f$0:Ljava/io/PrintStream;
0002: invoke-virtual {v0, v2}, Ljava/io/PrintStream;.println:(Ljava/lang/String;)V
0005: return-void
```

与之前调用生成的 Java8.lambda$main$0 类中包含 System.out.println() 的方法不同，log 的实现直接调用了 System.out.println().

生成的 lambda 类不再是静态单例。字节序 0000 直接读取了 PrintStream 的实例引用，该引用即是 System.out, 它在 main 方法中被调用，并且被传递给相应的构造器（名为 <init\> 的字节码）(This reference is System.out which is resolved at the call-site in main and passed into the constructor (which is named <init\> in bytecode)):

```Text
[0002bc] Java8.main:([Ljava/lang/String;)V
0000: sget-object v1, Ljava/lang/System;.out:Ljava/io/PrintStream;
0003: new-instance v0, L-$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM;
0004: invoke-direct {v0, v1}, L-$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM;.<init>:(Ljava/io/PrintStream;)V
0008: invoke-static {v0}, LJava8;.sayHi:(LJava8$Logger;)V
```

将其进行源码级别的转换，可以说是非常地直截了当：

```Diff
   public static void main(String... args) {
-    sayHi(System.out::println);
+    sayHi(new -$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM(System.out));
   }
@@
 }
+
+class -$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM implements Java8.Logger {
+  private final PrintStream ps;
+
+  -$$Lambda$1Osqr2Z9OSwjseX_0FMQJcCG_uM(PrintStream ps) {
+    this.ps = ps;
+  }
+
+  @Override public void log(String s) {
+    ps.println(s);
+  }
+}
```

## Interface Methods

Java 8 另一个显著的特性是可以在接口中定义静态方法和默认方法。接口中的静态方法可以被用来提供相关的工厂方法，或者其他有助于操作接口的方法。接口中的默认方法则允许你给已有接口中添加默认的方法实现，同时保持兼容性（你不需要给所有实现了该接口的类再全部实现一个新的方法）：

```Java
interface Logger {
  void log(String s);

  default void log(String tag, String s) {
    log(tag + ": " + s);
  }

  static Logger systemOut() {
    return System.out::println;
  }
}
```

D8 同样也支持这两种语法糖。你可以按照上文提到的方式来看看 D8 是如何脱糖的。

值得注意的是，这两种语法糖在 API 24 以上都是使用的原生实现。因此不像 lambda 和方法引用，--min-api 24 不会触发 D8 的脱糖操作。

## 只使用 Kotlin?

到此为止，肯定有大量读者开始考虑 Kotlin. 没错，Kotlin 的确在语言层面上直接提供了 lambda 和方法引用，的确在接口中提供了默认方法和类似静态方法的语法糖。但这些特性都是由 kotlinc 做的实现，和 D8 支持 Java 8 的字节码干的事情差不多（具体的实现细节可能有所不同）。

即时你完全使用 Kotlin 来写代码，Android 的开发工具链和 VM 对新语言特性的支持还是非常重要的。新版本的 Java 在字节码和 VM 方面更加高效，才能释放 Kotlin 更多的潜力。

Kotlin 在未来的某个时刻可能会放弃对 Java 6 和 Java 7 的支持。[Intellij 平台已经在 2016.1 迁移到了 Java 8](https://blog.jetbrains.com/idea/2015/12/intellij-idea-16-eap-144-2608-is-out/), Gradle 5.0 也迁移到了 Java 8. 运行在老 VM 上的平台越来越少。如果不支持 Java 8 的字节码和 VM 提供的功能，那么 Android 的生态系统就危险啦。感谢 D8 和 ART 让这一切不会发生。

## Desugaring APIs

到此为止，本文主要关注的是 Java 新版本语法糖和字节码。Java 8 其他的好处在于加入了一些新的 API, 如 streams, Optional, functional interfaces, CompletableFuture 和新的 date/time API.

回到先前的 logger 的例子上，我们可以使用新的 date/time API 得知我们是什么时候打的 log:

```Java
import java.time.*;

class Java8 {
  interface Logger {
    void log(LocalDateTime time, String s);
  }

  public static void main(String... args) {
    sayHi((time, s) -> System.out.println(time + " " + s));
  }

  private static void sayHi(Logger logger) {
    logger.log(LocalDateTime.now(), "Hello!");
  }
}
```

我们可以使用 javac 编译它并使用 D8 将其转换为 Dalvik 字节码：

```Text
$ javac *.java

$ java -jar d8.jar \
    --lib $ANDROID_HOME/platforms/android-28/android.jar \
    --release \
    --output . \
    *.class
```

你可以将其 push 到真机或者模拟器上进行验证，这是我们在先前的例子中没有做的：

```Text
$ adb push classes.dex /sdcard
classes.dex: 1 file pushed. 0.5 MB/s (1620 bytes in 0.003s)

$ adb shell dalvikvm -cp /sdcard/classes.dex Java8
2018-11-19T21:38:23.761 Hello!
```

在 API 26 及其以上的环境中，你会看见一条包含时间戳的字符串 "Hello!". 但在低版本环境中会报错：

```Text
java.lang.NoClassDefFoundError: Failed resolution of: Ljava/time/LocalDateTime;
  at Java8.sayHi(Java8.java:13)
  at Java8.main(Java8.java:9)
```

D8 只是对部分语法糖例如 lambda 进行了脱糖，但是却并没有提供诸如 LocalDateTime 这类的新的 API. 这当然是令人失望的，因为我们没法使用完整的 Java 8 新特性。

开发者当然可以自己实现一套 Optional 并且使用诸如 ThreeTenBP 这类 date/time 第三方库。但是既然我们修改打包的代码，为什么我们不能让 D8 提供这些新的 APi 呢？

看起来 D8 的确做了类似的事，但是只支持了一个 API: Throwable.addSuppressed(), 这个 API 允许 Java 7 的 try-with-resources 语法糖可以运行在全版本系统上。

我们需要的是能支持全版本系统的 Java 8 的新 API 的兼容实现。[看起来 Bazel 团队已经在做这件事了](https://blog.bazel.build/2018/07/19/java-8-language-features-in-android-apps.html)。他们重写的代码无法直接使用，但是将其重新打包是可以的 (Their code that does the rewriting can’t be used, but the standalone repackaging of these JDK APIs can be). 我们需要的就是 D8 团队将其添加到工具链中，[你可以在这个链接进行投票进行支持](https://issuetracker.google.com/issues/114481425)。

---

尽管对新语法糖的脱糖操作已经在多方面可用，但是新 API 的缺乏意味着还是与 Java 的生态系统有着巨大的差距。在大多数 App 的 minSdkVersion 26 之前，Android 的工具链只会阻碍 Java 生态系统的发展。需要同时支持 Android 和 JVM 的第三方库不能使用 Java 8 的新 API 至少五年！

尽管对 Java 8 的脱糖操作已经成为 D8 的一部分，但这也不是默认的。开发者必须明确声明他们的代码需要用到 Java 8. 第三方库的开发者可以通过强制使用 Java 8 而提升这种趋势（即时你没有使用到它的任何特性）。

鉴于 D8 已经开始起到实际作用了，所以前途还是光明的。即使你是一个 Kotlin 用户，你也有义务敦促 Android 支持新版本的 Java 字节码和 API 以获取更好的性能。实际上，在某些情况下，D8 还是要领先于 Java 8 的，我们将在下一篇博客中展现。

*（这篇博客是作为我的 [Digging into D8 and R8](https://jakewharton.com/digging-into-d8-and-r8/) 的分享的一部分，该分享并没有被直接公布出来。你可以看看其中的视频，并且关注我后续的博客）*

*———— Jake Wharton*