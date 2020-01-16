​	

![blue water and island](assets/%E4%B8%8D%E9%9C%80%E8%A6%81%E6%AD%BB%E8%AE%B0%E7%A1%AC%E8%83%8C%E7%9A%84jvm%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/photo-1579069260854-450fadfff665?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1000&q=80)

# 前言

> 对基本原理的了解,动手是最好的;

# 哪里入手

例子

```java
package com.java.study.jvm;

/**
 * @author zhangpeng
 * @since 2020/1/15 3:33 下午
 */
public class JvmHello {
    public static final int i = 2020;

    public static void main(String[] args) {
        JvmHello jvmHello = new JvmHello();
        int a = 1;
        int b = 2;
        int c = jvmHello.calculate1(a, b);
        int d = jvmHello.calculate2(a, b);
    }

    private int calculate2(int a, int b) {
        int x = 666;
        return x / (a + b);
    }

    private int calculate1(int a, int b) {
        return (a + b) * 2333;
    }
}
```

这段代码我就不解释了 直接编译字节码搞起

```shell
# 编译生成 JvmHello.class文件
javac JvmHello.java
# 反编译字节码内容
javap -verbose -p JvmHello.class
```

记得之前书里提到的,编译一次到处执行,那么首先文件要被加载进来,运行在一个环境里面;所以我们有了初步的图

![image-20200115172146495](assets/%E4%B8%8D%E9%9C%80%E8%A6%81%E6%AD%BB%E8%AE%B0%E7%A1%AC%E8%83%8C%E7%9A%84jvm%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20200115172146495.png)

JvmHello.java  -> JvmHello.class -> 类装载系统加载进来 -> 在虚拟机环境执行

接着我们看下JVMHello.class的内容

```java
Classfile /Users/zhangpeng/workspacke/mytest/study/src/main/java/com/java/study/jvm/JvmHello.class
  Last modified 2020-1-15; size 530 bytes
  MD5 checksum d1725552383bf6c86a00f1517d2b4c51
  Compiled from "JvmHello.java"
public class com.java.study.jvm.JvmHello
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#22         // java/lang/Object."<init>":()V
   #2 = Class              #23            // com/java/study/jvm/JvmHello
   #3 = Methodref          #2.#22         // com/java/study/jvm/JvmHello."<init>":()V
   #4 = Methodref          #2.#24         // com/java/study/jvm/JvmHello.calculate1:(II)I
   #5 = Methodref          #2.#25         // com/java/study/jvm/JvmHello.calculate2:(II)I
   #6 = Class              #26            // java/lang/Object
   #7 = Utf8               i
   #8 = Utf8               I
   #9 = Utf8               ConstantValue
  #10 = Integer            2020
  #11 = Utf8               <init>
  #12 = Utf8               ()V
  #13 = Utf8               Code
  #14 = Utf8               LineNumberTable
  #15 = Utf8               main
  #16 = Utf8               ([Ljava/lang/String;)V
  #17 = Utf8               calculate2
  #18 = Utf8               (II)I
  #19 = Utf8               calculate1
  #20 = Utf8               SourceFile
  #21 = Utf8               JvmHello.java
  #22 = NameAndType        #11:#12        // "<init>":()V
  #23 = Utf8               com/java/study/jvm/JvmHello
  #24 = NameAndType        #19:#18        // calculate1:(II)I
  #25 = NameAndType        #17:#18        // calculate2:(II)I
  #26 = Utf8               java/lang/Object
{
  public static final int i;
    descriptor: I
    flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
    ConstantValue: int 2020

  public com.java.study.jvm.JvmHello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=6, args_size=1
         0: new           #2                  // class com/java/study/jvm/JvmHello
         3: dup
         4: invokespecial #3                  // Method "<init>":()V
         7: astore_1
         8: iconst_1
         9: istore_2
        10: iconst_2
        11: istore_3
        12: aload_1
        13: iload_2
        14: iload_3
        15: invokespecial #4                  // Method calculate1:(II)I
        18: istore        4
        20: aload_1
        21: iload_2
        22: iload_3
        23: invokespecial #5                  // Method calculate2:(II)I
        26: istore        5
        28: return
      LineNumberTable:
        line 11: 0
        line 12: 8
        line 13: 10
        line 14: 12
        line 15: 20
        line 16: 28

  private int calculate2(int, int);
    descriptor: (II)I
    flags: ACC_PRIVATE
    Code:
      stack=3, locals=4, args_size=3
         0: sipush        666
         3: istore_3
         4: iload_3
         5: iload_1
         6: iload_2
         7: iadd
         8: idiv
         9: ireturn
      LineNumberTable:
        line 19: 0
        line 20: 4

  private int calculate1(int, int);
    descriptor: (II)I
    flags: ACC_PRIVATE
    Code:
      stack=2, locals=3, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: sipush        2333
         6: imul
         7: ireturn
      LineNumberTable:
        line 24: 0
}
SourceFile: "JvmHello.java"
```

## 字节码分析

### 1-8行

> 描述了类的基本信息

- 它是由哪个 *.java 文件编译而成的
- 最后编译时间
- 编译后的大小
- MD5校验值
- 遵循的java版本
- 访问标识,ACC_PUBLIC字面意思公有的嘛;ACC_SUPER不清楚是什么,但是应该和super方法有关系

### 9-35行   Constant pool 

> 运行时常量池

我们先分析下第一个常量,位于JVMHello.class第10行,我们会发现后面有关联项 一起放进来

```java
   #1 = Methodref          #6.#22         // java/lang/Object."<init>":()V
   #6 = Class              #26            // java/lang/Object
  #11 = Utf8               <init>
  #12 = Utf8               ()V
  #22 = NameAndType        #11:#12        // "<init>":()V
  #26 = Utf8               java/lang/Object
```

Methodref表示方法定义,右侧的注释内容(表示是由这几行组合起来的)

```java
java/lang/Object."<init>":()V
```

这段可以理解为该类的实例父类构造器的声明,此处也说明了JvmHello类的直接父类是Object.该方法默认返回值是V,也就是void,无返回值

同理分析下第二个常量,位于JVMHello.class第12行

```java
   #2 = Class              #23            // com/java/study/jvm/JvmHello
   #3 = Methodref          #2.#22         // com/java/study/jvm/JvmHello."<init>":()V
  #11 = Utf8               <init>
  #12 = Utf8               ()V
  #22 = NameAndType        #11:#12        // "<init>":()V
  #23 = Utf8               com/java/study/jvm/JvmHello
```

这里描述的是默认的构造器JvmHello(),因为后面在main()方法里面new了对象 所以这里会初始化到常量池

同理分析下第三个常量,位于JVMHello.class第13行

```java
   #2 = Class              #23            // com/java/study/jvm/JvmHello   
   #4 = Methodref          #2.#24         // com/java/study/jvm/JvmHello.calculate1:(II)I
  #18 = Utf8               (II)I
  #19 = Utf8               calculate1
  #24 = NameAndType        #19:#18        // calculate1:(II)I
```

这里描述的是JvmHello类里面calculate1方法的定义

```java
	com/java/study/jvm/JvmHello.calculate1:(II)I
```

**(II)** 表示入参为两个基本类型int

(II)**I** 右边的这个**I**表示返回值也是基本类型int

连起来说就是 calculate1方法入参是两个int,返回值是int

那么同理可得 位于JVMHello.class第14行的变量表示的是 calculate2方法入参也是两个int,返回值也是int

> 上述就是运行时常量池信息的分析,常量池用于存放编译期生成的各种字面量和符号引用,常量池是被划分在了**方法区**这个里面,**方法区**用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据到这我们补充一下我们的jvm图

![image-20200116164521074](assets/%E4%B8%8D%E9%9C%80%E8%A6%81%E6%AD%BB%E8%AE%B0%E7%A1%AC%E8%83%8C%E7%9A%84jvm%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20200116164521074.png)

### 36-115行 类内部方法描述

> 方法表集合

#### 36-41行 静态常量i的定义

> 先看下静态常量的定义,位于JVMHello.class 

```java
  public static final int i;
  descriptor: I
  flags: ACC_PUBLIC, ACC_STATIC, ACC_FINAL
  ConstantValue: int 2020
```

- 声明了一个公有变量i,类型为int
- 返回值为int
- 访问标识公共的、静态的、最终的
- 常量值为2020

#### 42-52行 类的构造器定义

```java
  public com.java.study.jvm.JvmHello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
```

- **()V** 参考之前方法的定义描述 这里是空参的方法;V表示特殊类型void无返回值
- **ACC_PUBLIC** 访问标识公共的
- **stack** 最大操作数栈 JVM会根据这个值来分配帧栈的操作栈深度,这里是1
- **locals** 局部变量所需存储空间,单位Slot,1Slot=4B,那么这里就是4个字节
- **args_size**方法参数的个数,这里是1,因为每个实例方法都会有一个隐藏参数this
- **aload_0** 当中的0正是局部变量表里的Slot 0的含义。意思是将局部变量表里的Slot 0的东西压入操作数栈,这个Slot 0里的东西name正是this,也就是JvmHello的实例
- **invokespecial #1** **invokespecial**表示根据编译时类型来调用实例方法 **#1**表示执行 常量池里面定义的实例方法,即JvmHello();
- **return** 从方法中返回,返回值为void
- **LineNumberTable** 该属性的作用是描述源码行号与字节码行号(字节码偏移量)之间的对应关系

这里我们产生了另一个概念**栈**,方法执行会进行压栈出栈 

#### 53-84行

main方法分析

```java
  public static void main(java.lang.String[]);// main方法
    descriptor: ([Ljava/lang/String;)V        // 入参String[],出参V(void)
    flags: ACC_PUBLIC, ACC_STATIC							// 公共的、静态的
    Code:
      stack=3, locals=6, args_size=1					// 操作数栈3,局部变量6 Slot,参数个数为1
         0: new           #2                  // class com/java/study/jvm/JvmHello    new对象
         3: dup																// 复制栈顶部一个字长内容
         4: invokespecial #3                  // Method "<init>":()V        执行JvmHello构造器
         7: astore_1													// 将returnAddress类型(引用类型)存入到局部变量[1]
         8: iconst_1													// 将int类型常量[1]压入到操作数栈
         9: istore_2													// 将int类型值存入局部变量[2]
        10: iconst_2													// 将int类型常量[2]压入到操作数栈
        11: istore_3													// 将int类型值存入局部变量[3]
        12: aload_1														// 从局部变量[1]中装载引用类型值
        13: iload_2														// 从局部变量[2]中装载int类型值 
        14: iload_3														// 从局部变量[3]中装载int类型值 
        15: invokespecial #4                  // Method calculate1:(II)I      执行calculate1方法
        18: istore        4										// 将int类型值存入局部变量[4]
        20: aload_1														// 从局部变量[1]中装载引用类型值
        21: iload_2														// 从局部变量[2]中装载int类型值 
        22: iload_3														// 从局部变量[3]中装载int类型值 
        23: invokespecial #5                  // Method calculate2:(II)I			执行calculate2方法
        26: istore        5										// 将int类型值存入局部变量[5]
        28: return														// void返回
      LineNumberTable:												
        line 11: 0
        line 12: 8
        line 13: 10
        line 14: 12
        line 15: 20
        line 16: 28
```

从第一行new对象说起

```java
  JvmHello jvmHello = new JvmHello();
  // 这里的jvmHello就是局部变量[1];
```

那么new出来的对象放在哪里的,看过jvm相关内容的同学都知道对象是分配在**堆**里面的

关于**堆**的定义

> 对于大多数应用来说，Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。这一点在Java虚拟机规范中的描述是：所有的对象实例以及数组都要在堆上分配,但是随着JIT编译器的发展与逃逸分析技术逐渐成熟，栈上分配、标量替换 优化技术将会导致一些微妙的变化发生，所有的对象都分配在堆上也渐渐变得不是那么“绝对”了。

接着看下面两行

```java
 int a = 1;
 int b = 2;
 // 1.这里先将常量[1] = 1压入到操作数栈
 // 2.再将整个常量[1]的int类型的值赋值给 局部变量[2]也就是 a = 1;
 // 同理 b=2也是同样的过程
```

然后看执行calculate1、calculate2方法

```java
int c = jvmHello.calculate1(a, b);
int d = jvmHello.calculate2(a, b);
// 1.从局部变量[1]中装载引用类型值 即jvmHello的值
// 2.从局部变量[2]中装载int类型值 即值为2
// 3.从局部变量[3]中装载int类型值 即值为2
// 4.使用jvmHello执行calculate1方法
// 同理 calculate2执行过程类似
```

上面这段我们知道,jvm在执行代码的时候,是基于**栈**的执行,也就是**操作栈** 每个栈里面有局部变量,局部变量是分配在**局部变量表**里面

关于java栈的定义,他有两个栈:java虚拟机栈和本地方法栈

> **Java虚拟机栈**（Java Virtual Machine Stacks）也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧（Stack Frame )用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。
>
> **本地方法栈**（Native Method Stack）与虚拟机栈所发挥的作用是非常相似的，它们之间的区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则为虚拟机使用到的Native方法服务

既然**虚拟机栈**里面提到线程,那么这里顺便介绍下**程序计数器**

> 程序计数器（Program Counter Register）是一块较小的内存空间，它可以看作是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能会通过一些更高效的方式去实现），字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

知道了**堆**和**栈**,继续补充一下我们的图

![image-20200116170901654](assets/%E4%B8%8D%E9%9C%80%E8%A6%81%E6%AD%BB%E8%AE%B0%E7%A1%AC%E8%83%8C%E7%9A%84jvm%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20200116170901654.png)

------

我们继续分析下calculate1和calculate2 

```java
private int calculate2(int, int);
    descriptor: (II)I														// 入参2个int类型 出参int类型
    flags: ACC_PRIVATE
    Code:
      stack=3, locals=4, args_size=3						// 操作数栈3,局部变量4 Slot,参数个数3个
         0: sipush        666										// 将16位带符号整数(这里指666)压入栈
         3: istore_3														// 将int类型值(即666)存入局部变量[3]
         4: iload_3															// 从局部变量[3]中装载int类型值
         5: iload_1															// 从局部变量[1]中装载int类型值
         6: iload_2															// 从局部变量[3]中装载int类型值
         7: iadd																// 执行int类型的加法,即 1+2
         8: idiv																// 执行int类型的除法,即 666/3
         9: ireturn															// 返回int类型的值
      LineNumberTable:
        line 19: 0
        line 20: 4

  private int calculate1(int, int);	
    descriptor: (II)I														// 入参2个int类型 出参int类型
    flags: ACC_PRIVATE													// 私有的
    Code:			
      stack=2, locals=3, args_size=3						// 操作数栈2,局部变量3 Slot,参数个数3个
         0: iload_1															// 从局部变量[1]中装载int类型值 
         1: iload_2															// 从局部变量[2]中装载int类型值
         2: iadd																// 执行int类型的加法,即 1+2
         3: sipush        2333									// 将16位带符号整数(这里指2333)压入栈
         6: imul																// 执行int类型的乘法  3*2333
         7: ireturn															// 返回int类型的值
      LineNumberTable:
        line 24: 0
```

其实到这里我有个疑问 为什么calculate1和calculate2的入参明明只有2个,反编译后会显示2个呢?我去搜了下

> 原来在计算args_size时,有判断方法是否为static方法,如果不是static方法,则会在方法原有参数数量上再加一，这是因为非static方法会添加一个默认参数到参数列表首位:方法的真正执行者,即方法所属类的实例对象。那对应我们这多出来的参数就是 jvmHello了

最后关于操作栈的过程  这里我以calculate1为例

![image-20200116161136903](assets/%E4%B8%8D%E9%9C%80%E8%A6%81%E6%AD%BB%E8%AE%B0%E7%A1%AC%E8%83%8C%E7%9A%84jvm%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20200116161136903.png)

上面提到的虚拟机栈的概念也提过,方法执行的同时会创建栈帧,存储局部变量表、操作数栈、动态链接、方法出口;所以上图就是一个栈帧在虚拟机中入栈到出栈的过程.基于这点最后补充一下栈里面的信息内容

![image-20200116172500427](assets/%E4%B8%8D%E9%9C%80%E8%A6%81%E6%AD%BB%E8%AE%B0%E7%A1%AC%E8%83%8C%E7%9A%84jvm%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/image-20200116172500427.png)

### 116行

> 表示源文件JvmHello.java

# 技术总结

通过分析字节码,可以加深对虚拟机内存结构,java代码从编译到加载,和运行的整个过程,而不是去死记书里的那些概念。

# 参考

- 深入理解java虚拟机-周志明
- [官方指令集文档](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html)
- [你知道Java方法能定义多少个参数吗？](https://zhuanlan.zhihu.com/p/44086976)

# END

> 喜欢的记得一键三连