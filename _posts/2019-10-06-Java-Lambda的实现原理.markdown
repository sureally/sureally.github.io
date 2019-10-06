---
layout: article
title: Java Lambda的实现原理
date: 2019/10/6 17:15
categories: Java
---

# 前言
lambda的要点就在invokedynamic这个指令了，通过invokedynamic指令生成目标对象，接下来我们了解一下invokedynamic指令。
>invokedynamic指令用于调用以绑定了invokedyanmic指令的调用点对象（call site object）作为目标的方法。调用点对象是一个特殊的语法结构，当一条invokedynamic指令首次被Java虚拟机执行前，java虚拟机将会执行一个引导方法（bootstrap method），并以这个方法的运行结果作为调用点对象。因此，每条invokedynamic指令都有独一无二的链接状态，这是其与invokevirtual等等调用指令的差别。

本文的分析主要是参考相关blog自己尝试理解的，写得比较模糊，后续理解更加深入后再更新。

# Lambda表达式样例
```Java
package lambda;

/**
 * @Author shu wj
 * @Date 2019/10/6 03:51
 * @Description
 */
@FunctionalInterface
public interface Print<T> {
  void print(T x);
}

```

```Java
package lambda;

/**
 * @Author shu wj
 * @Date 2019/10/6 03:53
 * @Description
 */
public class Lambda {
  public static void PrintString(String s, Print<String> print) {
    print.print(s);
  }

  public static void main(String[] args) {
    PrintString("test", (x) -> System.out.println(x));
  }
}

```

反编译生成的class文件
```
javap -p Lambda.class
```

```bash
192:lambda shushu$ javap -p Lambda.class 
Compiled from "Lambda.java"
public class lambda.Lambda {
  public lambda.Lambda();
  public static void PrintString(java.lang.String, lambda.Print<java.lang.String>);
  public static void main(java.lang.String[]);
  // 这里多了一个生成的私有静态方法
  private static void lambda$main$0(java.lang.String);
}
```

因此类似于
```Java
public class Lambda {
  public static void PrintString(String s, Print<String> print) {
    print.print(s);
  }

  private static void lambda$main$0(String x){
    System.out.println(x);
  }

  public static void main(String[] args) {
    PrintString("test", /* lambda expression */);
  }
}
```

如何调用这个静态方法的？在执行过程中会进入这个类
`package java.lang.invoke.LambdaMetafactory`

```Java
public class LambdaMetafactory {
    public static CallSite metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType)
            throws LambdaConversionException {
        AbstractValidatingLambdaMetafactory mf;
        mf = new InnerClassLambdaMetafactory(caller, invokedType,
                                             invokedName, samMethodType,
                                             implMethod, instantiatedMethodType,
                                             false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
        mf.validateMetafactoryArgs();
        return mf.buildCallSite();
    }
```

在main函数中加入下面这个语句, 在启动的时候生成的class会保存到设定的目录下面。
```Java
 public static void main(String[] args) {
    // 加入这个
    System.setProperty("jdk.internal.lambda.dumpProxyClasses", "/Users/shushu/IdeaProjects/javaExample/designMode/out");
    PrintString("test", (x) -> System.out.println(x));
  }
```

运行时，会将生成的内部类 class 码输出到指定的路径下

反编译
```bash
192:lambda shushu$ javap -p Lambda\$\$Lambda\$1.class 
final class lambda.Lambda$$Lambda$1 implements lambda.Print {
  private lambda.Lambda$$Lambda$1();
  public void print(java.lang.Object);
}
```

```bash
192:lambda shushu$ javap -c -p Lambda\$\$Lambda\$1.class 
final class lambda.Lambda$$Lambda$1 implements lambda.Print {
  private lambda.Lambda$$Lambda$1();
    Code:
       0: aload_0
       1: invokespecial #10                 // Method java/lang/Object."<init>":()V
       4: return

  public void print(java.lang.Object);
    Code:
       0: aload_1
       1: checkcast     #15                 // class java/lang/String
       4: invokestatic  #21                 // Method lambda/Lambda.lambda$main$0:(Ljava/lang/String;)V
       7: return
}

```

通过上面的字节码指令可以发现实现上调用的是Lambda.lambda$main$0这个私有的静态方法

因此，Lambda表达式等价于以下形式
```
public class Lambda {   
    public static void PrintString(String s, Print<String> print) {
        print.print(s);
    }
    private static void lambda$main$0(String x) {
        System.out.println(x);
    }
    final class $Lambda$1 implements Print{
        @Override
        public void print(Object x) {
            lambda$main$0((String)x);
        }
    }
    public static void main(String[] args) {
        PrintString("test", new Lambda().new $Lambda$1());
    }
}
```

在反编译Lambda.class，有个invokedynamic指令，
```Java
192:lambda shushu$ javap -c -p Lambda.class 
Compiled from "Lambda.java"
public class lambda.Lambda {
  public lambda.Lambda();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void PrintString(java.lang.String, lambda.Print<java.lang.String>);
    Code:
       0: aload_1
       1: aload_0
       2: invokeinterface #2,  2            // InterfaceMethod lambda/Print.print:(Ljava/lang/Object;)V
       7: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #3                  // String jdk.internal.lambda.dumpProxyClasses
       2: ldc           #4                  // String /Users/shushu/IdeaProjects/javaExample/designMode/out
       4: invokestatic  #5                  // Method java/lang/System.setProperty:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
       7: pop
       8: ldc           #6                  // String test
      10: invokedynamic #7,  0              // InvokeDynamic #0:print:()Llambda/Print;
      15: invokestatic  #8                  // Method PrintString:(Ljava/lang/String;Llambda/Print;)V
      18: return

  private static void lambda$main$0(java.lang.String);
    Code:
       0: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
       3: aload_0
       4: invokevirtual #10                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
       7: return
}

```

查看所有常量池，这里顺便简单学习如何阅读反编译的虚拟机汇编代码吧。//## 为自己添加的注释。
```
192:lambda shushu$ javap -v Lambda.class 
Classfile /Users/shushu/IdeaProjects/javaExample/designMode/target/classes/lambda/Lambda.class
  Last modified 2019-10-6; size 1690 bytes   //## 最后一次修改时间
  MD5 checksum a30eac3b86b4f7dc2094cb396f82f086 //## MD5
  Compiled from "Lambda.java" //## 原Java文件
public class lambda.Lambda
  minor version: 0
  major version: 52 //## class文件主版本号，52表示JDK8，即需要JDK8以上才能执行该文件
  flags: ACC_PUBLIC, ACC_SUPER //## 类的访问标志。ACC_SUPER表示当用到invokespecial时，需要对父类做特殊处理(在JDK.1.0.2 都有这个标志)
Constant pool:
   #1 = Methodref          #12.#39        // java/lang/Object."<init>":()V   //## 方法符号引用
   #2 = InterfaceMethodref #40.#41        // lambda/Print.print:(Ljava/lang/Object;)V //## 
   #3 = String             #42            // jdk.internal.lambda.dumpProxyClasses
   #4 = String             #43            // /Users/shushu/IdeaProjects/javaExample/designMode/out
   #5 = Methodref          #44.#45        // java/lang/System.setProperty:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
   #6 = String             #46            // test
   #7 = InvokeDynamic      #0:#52         // #0:print:()Llambda/Print;
   #8 = Methodref          #11.#53        // lambda/Lambda.PrintString:(Ljava/lang/String;Llambda/Print;)V
   #9 = Fieldref           #44.#54        // java/lang/System.out:Ljava/io/PrintStream;
  #10 = Methodref          #55.#56        // java/io/PrintStream.println:(Ljava/lang/String;)V
  #11 = Class              #57            // lambda/Lambda   //## 类的符号引用
  #12 = Class              #58            // java/lang/Object
  #13 = Utf8               <init>  //## 字符串：方法名
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable  //## 字符串：字节码行号与源代码行号的对应关系
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               Llambda/Lambda;
  #20 = Utf8               PrintString
  #21 = Utf8               (Ljava/lang/String;Llambda/Print;)V
  #22 = Utf8               s
  #23 = Utf8               Ljava/lang/String;
  #24 = Utf8               print
  #25 = Utf8               Llambda/Print;
  #26 = Utf8               LocalVariableTypeTable
  #27 = Utf8               Llambda/Print<Ljava/lang/String;>;
  #28 = Utf8               Signature
  #29 = Utf8               (Ljava/lang/String;Llambda/Print<Ljava/lang/String;>;)V
  #30 = Utf8               main
  #31 = Utf8               ([Ljava/lang/String;)V  //##字符串：方法描述符：void(String[] args)
  #32 = Utf8               args
  #33 = Utf8               [Ljava/lang/String;
  #34 = Utf8               lambda$main$0
  #35 = Utf8               (Ljava/lang/String;)V
  #36 = Utf8               x
  #37 = Utf8               SourceFile
  #38 = Utf8               Lambda.java
  #39 = NameAndType        #13:#14        // "<init>":()V  ## 字段或方法的部分符号引用
  #40 = Class              #59            // lambda/Print
  #41 = NameAndType        #24:#60        // print:(Ljava/lang/Object;)V
  #42 = Utf8               jdk.internal.lambda.dumpProxyClasses
  #43 = Utf8               /Users/shushu/IdeaProjects/javaExample/designMode/out
  #44 = Class              #61            // java/lang/System
  #45 = NameAndType        #62:#63        // setProperty:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #46 = Utf8               test
  #47 = Utf8               BootstrapMethods
  #48 = MethodHandle       #6:#64         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #49 = MethodType         #60            //  (Ljava/lang/Object;)V
  #50 = MethodHandle       #6:#65         // invokestatic lambda/Lambda.lambda$main$0:(Ljava/lang/String;)V
  #51 = MethodType         #35            //  (Ljava/lang/String;)V
  #52 = NameAndType        #24:#66        // print:()Llambda/Print;
  #53 = NameAndType        #20:#21        // PrintString:(Ljava/lang/String;Llambda/Print;)V
  #54 = NameAndType        #67:#68        // out:Ljava/io/PrintStream;
  #55 = Class              #69            // java/io/PrintStream
  #56 = NameAndType        #70:#35        // println:(Ljava/lang/String;)V
  #57 = Utf8               lambda/Lambda
  #58 = Utf8               java/lang/Object
  #59 = Utf8               lambda/Print
  #60 = Utf8               (Ljava/lang/Object;)V
  #61 = Utf8               java/lang/System
  #62 = Utf8               setProperty
  #63 = Utf8               (Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
  #64 = Methodref          #71.#72        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #65 = Methodref          #11.#73        // lambda/Lambda.lambda$main$0:(Ljava/lang/String;)V
  #66 = Utf8               ()Llambda/Print;
  #67 = Utf8               out
  #68 = Utf8               Ljava/io/PrintStream;
  #69 = Utf8               java/io/PrintStream
  #70 = Utf8               println
  #71 = Class              #74            // java/lang/invoke/LambdaMetafactory
  #72 = NameAndType        #75:#79        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #73 = NameAndType        #34:#35        // lambda$main$0:(Ljava/lang/String;)V
  #74 = Utf8               java/lang/invoke/LambdaMetafactory
  #75 = Utf8               metafactory
  #76 = Class              #81            // java/lang/invoke/MethodHandles$Lookup
  #77 = Utf8               Lookup
  #78 = Utf8               InnerClasses
  #79 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #80 = Class              #82            // java/lang/invoke/MethodHandles
  #81 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #82 = Utf8               java/lang/invoke/MethodHandles
{
  public lambda.Lambda();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1 //## 一个栈帧, 本地变量表有一个变量, 参数大小为1(每一个方法编译时,编译器会自动添加一个this参数,表示引用当前对象)
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V  //## 调用父类方法、实例初始化方法、私有方法
         4: return
      LineNumberTable:
        line 10: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Llambda/Lambda;

  public static void PrintString(java.lang.String, lambda.Print<java.lang.String>);
    descriptor: (Ljava/lang/String;Llambda/Print;)V //## 方法描述符: void (String str, Print)
    flags: ACC_PUBLIC, ACC_STATIC                   //## 方法访问修饰符public,static
    Code:
      stack=2, locals=2, args_size=2                //## 两个栈帧
         0: aload_1
         1: aload_0
         2: invokeinterface #2,  2            // InterfaceMethod lambda/Print.print:(Ljava/lang/Object;)V //## 调用接口方法
         7: return
      LineNumberTable:
        line 12: 0
        line 13: 7
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       8     0     s   Ljava/lang/String;
            0       8     1 print   Llambda/Print;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            0       8     1 print   Llambda/Print<Ljava/lang/String;>;
    Signature: #29                          // (Ljava/lang/String;Llambda/Print<Ljava/lang/String;>;)V

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V //## 方法描述符: void (String[] args)
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1 //## 两个栈帧，本地变量表存在一个变量(this)
         0: ldc           #3                  // String jdk.internal.lambda.dumpProxyClasses  //##将常量值从常量池推送至栈顶
         2: ldc           #4                  // String /Users/shushu/IdeaProjects/javaExample/designMode/out
         4: invokestatic  #5                  // Method java/lang/System.setProperty:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String; //## 调用静态方法
         7: pop
         8: ldc           #6                  // String test
        10: invokedynamic #7,  0              // InvokeDynamic #0:print:()Llambda/Print; //## 调用动态链接方法，会先指向BootstrapMethods中编号为0的引导方法
        15: invokestatic  #8                  // Method PrintString:(Ljava/lang/String;Llambda/Print;)V
        18: return
      LineNumberTable:
        line 16: 0
        line 17: 8
        line 18: 18
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      19     0  args   [Ljava/lang/String;
}
SourceFile: "Lambda.java"
InnerClasses:
     public static final #77= #76 of #80; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #48 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #49 (Ljava/lang/Object;)V
      #50 invokestatic lambda/Lambda.lambda$main$0:(Ljava/lang/String;)V
      #51 (Ljava/lang/String;)V

```

`bootstrap_method_attr_index`索引`bootstrap_methods`表，`bootstrap_methods`位于class文件的`attributes`表里。

如果类的常量池中存在`CONSTANT_InvokeDynamic_info`的话，那么`attributes`表中就必定有且仅有一个`BootstrapMethods`属性。`BootstrapMethods`属性是个变长的表.

确实存在一个`BootstrapMethods`表，这个表中只有一个`BootstrapMethod`，它的`bootstrap_method_ref`是常量池#48，有三`个bootstrap_arguments`，分别指向常量池#49，#50和#51：
```bash
BootstrapMethods:
  0: #48 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #49 (Ljava/lang/Object;)V
      #50 invokestatic lambda/Lambda.lambda$main$0:(Ljava/lang/String;)V
      #51 (Ljava/lang/String;)V
```


JVM规范：
```
CONSTANT_MethodHandle_info {
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}
```
`reference_kind`是一个1到9之间的整数，6 (REF_invokeStatic)。引用的是`java.lang.invoke.LambdaMetafactory`类的静态方法`metafactory()`


## lambda机制总结
编译时：

1. lambda表达式生产一个静态方法`lambda$main$0`，方法实现了表达式的代码逻辑；
2. 编译生成`invokedynamic`指令，调用`bootstrap`方法，由`java.lang.invoke.LambdaMetafactory. metafactory()`方法实现；   

运行时：

3.` invokedynamic`指令调用`metafactory`方法，返回一`个callsite`，此`callsite`返回目标类型的一个匿名实现类（`MethodHandles.Lookup caller` 的内部类），此类关联编译时产生的方法；
4. lambda表达式调用时会调用匿名实现类`Lambda$$Lambda$1`关联的方法`lambda$main$0`


# 另一种形式的分析
```Java
package lambda;

import java.lang.invoke.LambdaMetafactory;

/**
 * @Author shu wj
 * @Date 2019/10/6 03:53
 * @Description
 */
public class Lambda {
  public static void PrintString(String s, Print<String> print) {
    print.print(s);
  }

  public static void main(String[] args) {
    System.setProperty("jdk.internal.lambda.dumpProxyClasses", "/Users/shushu/IdeaProjects/javaExample/designMode/out");
    // 推荐的表达方式
    PrintString("test", System.out::println);
  }

}

```

现在反编译，此时并没有生成静态方法
```bash
192:lambda shushu$ javap -p Lambda.class 
Compiled from "Lambda.java"
public class lambda.Lambda {
  public lambda.Lambda();
  public static void PrintString(java.lang.String, lambda.Print<java.lang.String>);
  public static void main(java.lang.String[]);
}

```

但是同样会生成一个内部类，反编译
```bash
192:lambda shushu$ javap -p Lambda\$\$Lambda\$1.class 
final class lambda.Lambda$$Lambda$1 implements lambda.Print {
  private final java.io.PrintStream arg$1;
  private lambda.Lambda$$Lambda$1(java.io.PrintStream);
  private static lambda.Print get$Lambda(java.io.PrintStream);
  public void print(java.lang.Object);
}

```

```bash
192:lambda shushu$ javap -c -p Lambda\$\$Lambda\$1.class 
final class lambda.Lambda$$Lambda$1 implements lambda.Print {
  private final java.io.PrintStream arg$1;

  private lambda.Lambda$$Lambda$1(java.io.PrintStream);
    Code:
       0: aload_0
       1: invokespecial #13                 // Method java/lang/Object."<init>":()V
       4: aload_0
       5: aload_1
       6: putfield      #15                 // Field arg$1:Ljava/io/PrintStream;
       9: return

  private static lambda.Print get$Lambda(java.io.PrintStream);
    Code:
       0: new           #2                  // class lambda/Lambda$$Lambda$1
       3: dup
       4: aload_0
       5: invokespecial #19                 // Method "<init>":(Ljava/io/PrintStream;)V
       8: areturn

  public void print(java.lang.Object);
    Code:
       0: aload_0
       1: getfield      #15                 // Field arg$1:Ljava/io/PrintStream;
       4: aload_1
       5: checkcast     #24                 // class java/lang/String
       8: invokevirtual #30                 // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      11: return
}

```



# 参考文档
- https://blog.csdn.net/liupeifeng3514/article/details/80759907
- https://www.jianshu.com/p/c2c1c5bd4971