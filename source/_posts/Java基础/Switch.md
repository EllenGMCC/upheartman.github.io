---
title: Switch
categories: 
- Java基础
---

**原文：** https://mp.weixin.qq.com/s/zZtZfRgwlFU3U_8FGpcABA

# 前言

前几天重新看 《阿里巴巴Java开发手册》有一条这样的规约：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8b27b6e3b32649309befcf494d4a54a5~tplv-k3u1fbpfcp-zoom-1.image)

**出于好奇，打算研究一下！，强迫症，没办法！**

我们先用一个案例测试一下：

```java
public class Test {
    public static void main(String[] args) {
        String param = null;
        switch (param) {
            case "null":
                System.out.println("匹配null字符串");
                break;
            default:
                System.out.println("进入default");
        }
    }
}
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ba0121408a648f79a29b30f6fc6aec6~tplv-k3u1fbpfcp-zoom-1.image)

显而易见，如果switch传入空值，会抛空指针！

**看到这，我们先可以思考下面几个问题：**

* switch 除了 String 还支持哪种类型? 

* 为什么《阿里巴巴Java开发手册》规定String类型参数要先进行 null 判断?
* 为什么可能会抛出空指针异常?

**下面开始对上面的问题进行分析**

# 问题分析

首先参考[官方文档](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html%23jls-14.11/)对swtich 语句相关描述。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fae6748cdadf4bd48eb9b9b77d7f3c30~tplv-k3u1fbpfcp-zoom-1.image)

**翻译如下：**

> switch 的表达式必须是 char, byte, short, int, Character, Byte, Short, Integer, String, 或者 enum 类型，否则会发生编译错误

同时switch 语句必须满足以下条件，否则会出现编译错误:

* 与 switch 语句关联的每个 case 都必须和 switch 的表达式的类型一致; 

* 如果 switch 表达式是枚举类型，case 常量也必须是枚举类型; 

* 不允许同一个 switch 的两个 case 常量的值相同;

* 和 switch 语句关联的常量不能为 null ;

* 一个 switch 语句最多有一个 default 标签。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47584db1f66a4e39bdde5fd935bf781a~tplv-k3u1fbpfcp-zoom-1.image)

**翻译如下：**

switch 语句执行的时候，首先将执行 switch 的表达式。如果表达式为 null, 则会抛出 NullPointerException，整个 switch 语句的执行将被中断。

**另外从《Java虚拟机规范》这本书，我们可以学习到：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/29036d99db754cef8039d8cc18265a4d~tplv-k3u1fbpfcp-zoom-1.image)

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/743e737d767344288c062bfff9e19b74~tplv-k3u1fbpfcp-zoom-1.image)

**总结一下就是：**

> 1.编译器使用 tableswitch 和 lookupswitch 指令生成 switch 语句的编译代码。 
>
> 2.Java 虚拟机的 tableswitch 和 lookupswitch 指令只能支持 int 类型的条件值。如果 swich 中使用其他类型的值，那么就必须转化为 int 类型。

所以可以了解到空指针出现的根源在于：虚拟机为了实现 switch 的语法，将参数表达式转换成 int。而这里的参数为 null， 从而造成了空指针异常。

**下面对官方文档的内容采用反汇编方式进一步分析下**

不熟悉字节码的，推荐看看美团的这篇文章：[https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html](https://tech.meituan.com/2019/09/05/java-bytecode-enhancement.html)

**下面开始硬货！**

## 反汇编看看

一个例子：

```java
public class Test {
    public static void main(String[] args) {
        String param = "月伴飞鱼";
        switch (param) {
            case "月伴飞鱼1":
                System.out.println("月伴飞鱼1");
                break;
            case "月伴飞鱼2":
                System.out.println("月伴飞鱼2");
                break;
            case "月伴飞鱼3":
                System.out.println("月伴飞鱼3");
                break;
            default:
                System.out.println("default");
        }
    }
}
```

反汇编代码得到：

```c++
Compiled from "Test.java"
public class com.zhou.Test {
  public zhou.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #2                  // String 月伴飞鱼
       2: astore_1
       3: aload_1
       4: astore_2
       5: iconst_m1
       6: istore_3
       7: aload_2
       8: invokevirtual #3                  // Method java/lang/String.hashCode:()I
      11: tableswitch   { // -768121881 to -768121879
            -768121881: 36
            -768121880: 50
            -768121879: 64
               default: 75
          }
      36: aload_2
      37: ldc           #4                  // String 月伴飞鱼1
      39: invokevirtual #5                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      42: ifeq          75
      45: iconst_0
      46: istore_3
      47: goto          75
      50: aload_2
      51: ldc           #6                  // String 月伴飞鱼2
      53: invokevirtual #5                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      56: ifeq          75
      59: iconst_1
      60: istore_3
      61: goto          75
      64: aload_2
      65: ldc           #7                  // String 月伴飞鱼3
      67: invokevirtual #5                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      70: ifeq          75
      73: iconst_2
      74: istore_3
      75: iload_3
      76: tableswitch   { // 0 to 2
                     0: 104
                     1: 115
                     2: 126
               default: 137
          }
     104: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     107: ldc           #4                  // String 月伴飞鱼1
     109: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     112: goto          145
     115: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     118: ldc           #6                  // String 月伴飞鱼2
     120: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     123: goto          145
     126: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     129: ldc           #7                  // String 月伴飞鱼3
     131: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     134: goto          145
     137: getstatic     #8                  // Field java/lang/System.out:Ljava/io/PrintStream;
     140: ldc           #10                 // String default
     142: invokevirtual #9                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
     145: return
}
```

**先介绍一下下面会用到的字节码指令**

> invokevirtual：调用实例方法
>
> istore_0 将int类型值存入局部变量0
>
> istore_1 将int类型值存入局部变量1
>
> istore_2 将int类型值存入局部变量2
>
> istore_3 将int类型值存入局部变量3
>
> aload_0 从局部变量0中装载引用类型值
>
> aload_1 从局部变量1中装载引用类型值
>
> aload_2 从局部变量2中装载引用类型值

**更详细的指令集大全关注我公众号，加我微信发你哈！**

我们继续看汇编代码：

先看偏移为 8 的指令，调用了参数的 hashCode() 函数来获取字符串 "月伴飞鱼" 的哈希值。

```
8: invokevirtual #3                  // Method java/lang/String.hashCode:()I
```

**接下来我们看偏移为 11 的指令处:** 

tableswitch 是跳转引用列表， 如果值小于其中的最小值-768121881 或者大于其中的最大值-768121879，跳转到 default 语句。

```
11: tableswitch   { // -768121881 to -768121879
            -768121881: 36
            -768121880: 50
            -768121879: 64
               default: 75
          }
```

> 其中 -768121881 为键，36 为对应的目标语句偏移量。

hashCode 和 tableswitch 的键相等，则跳转到对应的目标偏移量，"月伴飞鱼"的哈希值806505866不在最小值-768121881和最大值-768121879之间，因此跳转到 default 对应的语句行(即偏移量为 75 的指令处执行)。

> 月伴飞鱼的hash值计算：("月伴飞鱼").hashCode();

从 36 到 75 行，根据哈希值相等跳转到判断是否相等的指令。

然后调用` java.lang.String#equals `判断 switch 的字符串是否和对应的 case 的字符串相等。

如果相等则分别根据第几个条件得到条件的索引，然后每个索引对应下一个指定的代码行数。

**继续从偏移量75行往下看：**

```
      76: tableswitch   { // 0 to 2
                     0: 104
                     1: 115
                     2: 126
               default: 137
          }
```

default 语句对应 137 行，打印 “default” 字符串，然后执行 145 行 return 命令返回。

通过 tableswitch 判断执行哪一行打印语句。

**总结就是整个流程是先计算字符串参数的哈希值，判断哈希值的范围，然后哈希值相等再判断对象是否相等，然后执行对应的代码块。**

这种先判断 hash 值是否相等(有可能是同一个对象/两个对象有可能相等)，再通过 equals 比较 对象是否相等 的做法，在 Java 的很多 JDK 源码中和其他框架中也非常常见的。

## 分析空指针问题

反汇编前言中的代码：

```java
public class Test {
    public static void main(String[] args) {
        String param = null;
        switch (param) {
            case "null":
                System.out.println("匹配null字符串");
                break;
            default:
                System.out.println("进入default");
        }
    }
}
```

```c++
public class com.zhou.Test {
  public com.zhou.Test();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: aconst_null
       1: astore_1
       2: aload_1
       3: astore_2
       4: iconst_m1
       5: istore_3
       6: aload_2
       7: invokevirtual #2                  // Method java/lang/String.hashCode:()I
      10: lookupswitch  { // 1
               3392903: 28
               default: 39
          }
      28: aload_2
      29: ldc           #3                  // String null
      31: invokevirtual #4                  // Method java/lang/String.equals:(Ljava/lang/Object;)Z
      34: ifeq          39
      37: iconst_0
      38: istore_3
      39: iload_3
      40: lookupswitch  { // 1
                     0: 60
               default: 71
          }
      60: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      63: ldc           #6                  // String 匹配null字符串
      65: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      68: goto          79
      71: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
      74: ldc           #8                  // String 进入default
      76: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
      79: return
}
```

可以猜测3392903 应该是 "null" 字符串的哈希值。 

```
10: lookupswitch  { // 1
               3392903: 28
               default: 39
          }
```

我们可以打印其哈希值去印证:`System.out.println(("null").hashCode()); `

**总结整体流程：**

```java
String param = null;
int hashCode = param.hashCode();
if(hashCode == ("null").hashCode() && param.equals("null")){
	System.out.println("null"); 
}else{
	System.out.println("default"); 
}
```



![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ca0e9af74f9448509f3fe7f1193bd767~tplv-k3u1fbpfcp-zoom-1.image)

因此空指针的原因就一目了然了：**调用了 null 对象的实例方法**。

# 最后

**写文章画图不易，喜欢的话，希望帮忙点赞，转发下哈，谢谢，谢谢**

微信搜索：月伴飞鱼，交个朋友

公众号后台回复666，可以获得免费电子书籍