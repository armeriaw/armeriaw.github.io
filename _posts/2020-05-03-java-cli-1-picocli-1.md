---
layout: post
title: java-cli-1-picocli-1
postTitle: Java CLI设计（一）——picocli（上）
categories: [Java, Software Construction, CLI, picocli]
description: 软件构造第8次博客
keywords: Java, Software Construction, CLI, picocli
published: true
mathjax: false
typora-root-url: ..
---

在软件构造Lab3中，我尝试用[picocli](https://github.com/remkop/picocli)搭建了一个命令行交互客户端（*Command-Line Interface*，CLI）。不同于[Common CLI](http://commons.apache.org/proper/commons-cli/)，picocli的设计更为现代化，功能更为强大，[文档](https://picocli.info/)也十分丰富。

## picocli简介

picocli是一个现代库框架，适用于在 JVM 上构建命令行应用。它支持Java、Groovy、Kotlin 和 Scala。它推出的时间还不到 3 年，但是非常受欢迎，每月的下载量超过了 50 万次。Groovy 语言使用它来实现其`CliBuilder` DSL。

picocli致力于提供「最简便的方式来创建富命令行应用，使其可以在JVM上和JVM之外运行」。它提供了彩色输出、TAB键自动完成、子命令，与其他的JVM CLI相比，它还提供了一些独特的特性，比如可否定选项、重复复合参数组、重复子命令和对引用参数的复杂处理。它的源代码在单个文件中，因此我们可以选择将其作为源代码包含进来，避免添加依赖项。picocli对其丰富和细致的文档颇感自豪。

![Screenshot of usage help with Ansi codes enabled](/images/asserts/checksum-usage-help.png)

上图为picocli生成的CLI应用示例。

picocli的另一个显著特征是，它致力于让用户运行基于picoci的应用程序，而不需要将picocli作为外部依赖项：所有源代码都存在于一个文件中，以鼓励应用程序作者将其以**源代码形式**包含进来。达成这一点的工作原理是**注释类**，picocli从命令行参数初始化它，将输入转换为类字段中的强类型值。

利用picocli进行Lab3的客户端开发分为上下两篇。上篇（本篇）主要介绍CLI应用的基本特征、基本概念和picocli的基本用法等，下篇则讲述利用picocli开发具体应用的过程。

## 准备工作

用Maven引用picocli的方式十分简单：

```xml
<dependency>
  <groupId>info.picocli</groupId>
  <artifactId>picocli</artifactId>
  <version>4.3.0</version>
</dependency>
```

以及

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <!-- annotationProcessorPaths requires maven-compiler-plugin version 3.5 or higher -->
  <version>${maven-compiler-plugin-version}</version>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>info.picocli</groupId>
        <artifactId>picocli-codegen</artifactId>
        <version>4.3.0</version>
      </path>
    </annotationProcessorPaths>
  </configuration>
</plugin>
```

## CLI应用中的基本概念

### 命令行`CommandLine`

基本的命令行模板如下：

```shell
Prompt $command param1 param2 param3 … paramN
```

![File:COMMAND LINE.svg](/images/asserts/518px-COMMAND_LINE.svg.png)

- `Prompt`是命令行程序（如Shell等）为客户端提供的运行环境。
- `Command`是由客户端提供的命令。命令通常可以分为三类：
  - **内部命令**：由命令行解释器本身识别和处理，不依赖于任何外部可执行文件。
  - **包含命令**：一个单独的可执行文件，通常被认为是操作环境的一部分，并且总是包含在操作系统中。
  - **外部命令**：外部的可执行文件提供的命令。它不是基本操作系统的一部分，而是由其他的人为特定的目的和应用程序添加的。

- `param1 … paramN`是客户端提供的可选参数。参数的格式和含义取决于要执行的命令。如果是包含命令或外部命令，参数的值在操作系统启动程序时交付给程序（由命令程序指定）。它们可以是参数（Arguments），也可以是选项（Options）。

### 命令行参数

命令行参数（Command-line Argument或Command-line parameter）是程序开始时客户端提供的一系列信息。一个程序可以有许多命令行参数来标识信息的来源或目标，或者改变程序的操作。当一个命令处理器处于活动状态时，程序通常是通过输入它的名称和命令行参数（如果有）来调用的。例如，在Unix和类Unix环境中，命令行参数的一个例子是：

```shell
$rm file.s
```

这个例子中，`file.s`就是一个命令行参数，它告诉程序`rm`，执行对象是文件`file.s`。

C、C++和Java等高级语言允许程序通过在主函数中将命令行参数处理为字符串参数来解释命令行参数。

## picoli基本用法

### 选项`Option`

在picocli中，选项以单横杠`-`或双横杠`--`开头，且必须要有名字。单字母选项的含义有一些约定俗成的规定，如`-a`表示「所有的」，`-d`表示「调试模式」等，详情见[此页面](http://catb.org/~esr/writings/taoup/html/ch10s05.html)。

例如，Linux系统中，如下的`tar`命令可以实现「将`file1.txt`和`file2.txt`一起打包为`result.tar`」的功能：

```shell
$tar -c --file result.tar file1.txt file2.txt
```

其中`-c`、`--file`就都是**选项**（在picocli中用`@Option`记号标明）。前者是一个布尔型的选项（只需指明，无需参数），表示「创建一个新压缩文档」。后者则是一个需要参数的选项（使用方式为`-c=ARCHIVE`），用于设定新创建的压缩文档的文件名。此外，布尔型选项在输入时还可以叠加，以简化输入，如以下命令输入都是等价的：

```shell
$command -abcfInputFile.txt
$command -abcf=InputFile.txt
$command -abc -f=InputFile.txt
$command -ab -cf=InputFile.txt
$command -a -b -c -fInputFile.txt
$command -a -b -c -f InputFile.txt
$command -a -b -c -f=InputFile.txt
...
```

而最后的`file1.txt`和`file2.txt`则属于参数（在picocli中用`@Parameters`标明）。

在picocli中，一个命令需要封装在一个类中。上面这个`tar`命令就可以简单地封装在这个`Tar`类里面：

```java
class Tar {
    @Option(names = "-c", description = "create a new archive")
    // 添加参数的基本格式是@Annotation(属性列表 names = ..., paramLabel = ...)
    // 布尔型选项-c对应的标记变量为create
    boolean create;

    @Option(names = { "-f", "--file" }, paramLabel = "ARCHIVE", description = "the archive file")
    // -f选项对应的文件实例archive
    File archive;

    @Parameters(paramLabel = "FILE", description = "one ore more files to archive")
    File[] files;

    @Option(names = { "-h", "--help" }, usageHelp = true, description = "display a help message")
    private boolean helpRequested = false;
}
```

在主程序中，只要如下两行代码，

```java
Tar tar = new Tar();

// args为外部传入的参数
new CommandLine(tar).parseArgs(args);
```

就能够自动实现参数的解析了，且解析的结果自动存储到`tar`实例的各个对应成员变量中。例如，如果参数为

```java
String[] args = { "-c", "--file", "result.tar", "file1.txt", "file2.txt" }
```

那么以下断言都成立：

```java
assert !tar.helpRequested;
assert  tar.create;
assert  tar.archive.equals(new File("result.tar"));
assert  Arrays.equals(tar.files, new File[] {new File("file1.txt"), new File("file2.txt")});
```

注意到，输入的参数列表都是`String`类型的，但`-f`选项对应的变量却是`File`类——picocli利用`File`的相应的构造方法自动实现了`String`类到`File`类的转换。picocli可以自动转换像`File`等JDK内置类自定义类，无需任何设置；而对于自定义类，也只需在`CommandLine`实例中注册相应的构造方法，也可以让picocli自动实现这些转化，十分方便。

### 多参数选项与`arity`

有时，我们希望一个选项后面提供多个参数。`arity`属性可以实现这一点，它可以精确地控制每个选项的数量。`arity`属性既可以指定所需参数的确切数量，也可以指定具有最小和最大参数数量的范围。最大值可以是一个精确的上界，也可以是`*`来表示任意数量的参数。例如：

```java
class ArityDemo {
    @Parameters(arity = "1..3", description = "one to three Files")
    File[] files;

    @Option(names = "-f", arity = "2", description = "exactly two floating point numbers")
    double[] doubles;

    @Option(names = "-s", arity = "1..*", description = "at least one string")
    String[] strings;
}
```

需要注意的是，一旦使用了最小数量的参数，picocli将检查每个后续命令行参数，以确定它是一个附加参数还是一个新选项。

### 选项关系与分组`ArgGroups`

实际应用中，我们常会设计出一些相互冲突的选项，而另一些可能是一个整体的若干组成部分，缺一不可。这时，我们就需要对选项进行分组，并定义选项组内部的逻辑关系。

picocli中的选项组主要分为两种：**冲突**和**依赖**。

例如，冲突的选项组可以这么定义：

```java
@Command(name = "exclusivedemo")
public class MutuallyExclusiveOptionsDemo {

    @ArgGroup(exclusive = true, multiplicity = "1")
    Exclusive exclusive;

    static class Exclusive {
        @Option(names = "-a", required = true) int a;
        @Option(names = "-b", required = true) int b;
        @Option(names = "-c", required = true) int c;
    }
}
```

`Exclusive`就是一个冲突的选项组。属性`multiplicity = "1"`的含义是，`Exclusive`的三个选项中，至少且最多选1个选项。该属性的默认值是`"0..1"`，即要么全不选，要么最多选1个。`exclusivedemo`的使用说明长这样：

```
Usage: exclusivedemo (-a=<a> | -b=<b> | -c=<c>)
```

另外注意到`Exclusive`中的每个选项全都加了`required = true`属性。这个`required = true`是针对选项组`Exclusive`内部而言的，并不是说整个`exclusivedemo`都必须要选上这三个选项。

同理，依赖的选项组可以这样定义：

```java
@Command(name = "co-occur")
public class DependentOptionsDemo {

    @ArgGroup(exclusive = false)
    Dependent dependent;

    static class Dependent {
        @Option(names = "-a", required = true) int a;
        @Option(names = "-b", required = true) int b;
        @Option(names = "-c", required = true) int c;
    }
}
```

「依赖」的含义就是「同时出现」。`co-occur`的使用说明就长这样：

```
Usage: co-occur [-a=<a> -b=<b> -c=<c>]
```

此外，选项组还可以嵌套使用。例如：

```java
@Command(name = "repeating-composite-demo")
public class CompositeGroupDemo {

    @ArgGroup(exclusive = false, multiplicity = "1..*")
    List<Composite> composites;

    static class Composite {
        @ArgGroup(exclusive = false, multiplicity = "0..1")
        Dependent dependent;

        @ArgGroup(exclusive = true, multiplicity = "1")
        Exclusive exclusive;
    }

    static class Dependent {
        @Option(names = "-a", required = true) int a;
        @Option(names = "-b", required = true) int b;
        @Option(names = "-c", required = true) int c;
    }

    static class Exclusive {
        @Option(names = "-x", required = true) boolean x;
        @Option(names = "-y", required = true) boolean y;
        @Option(names = "-z", required = true) boolean z;
    }
}
```

`repeating-composite-demo`的使用说明就是：

```
Usage: repeating-composite-demo ([-a=<a> -b=<b> -c=<c>] (-x | -y | -z))...
```

这个命令类中，最外层的`Composite`是一个依赖命令组，其内部包含`Dependent`和`Exclusive`两个命令组，分别为依赖的和冲突的。

### 子命令`subcommmands`

子命令是扩充CLI应用功能的最佳方式。许多CLI应用如`apt-get`、`git`和`conda`等，都提供了一些列功能各异的子命令，它们分工明确，各司其职又相互联系，共同组成了一个完整的强大的CLI应用。

用picocli为命令类增加子命令主要有三种方式：

- 在命令类的`@Command`标记中添加子命令类
- 在命令类的方法上添加`@Command`标记，将其转变成子命令
- 在程序中调用`addSubcommand`方法，动态添加子命令

一般来说，第一种方法最为常用，且最能保证代码的整洁清楚。

例如，对于`git`命令，我们可以在`Git`命令类上添加子命令属性：

```java
@Command(
  name = "git",
  subcommands = {
      GitAddCommand.class,
      GitCommitCommand.class
  },
  ...
)
class GitCommand implements Runnable {
	...
}
```

其中，`GitAddCommand`和`GitCommitCommand`就是两个子命令类，可以定义如下：

```java
@Command(
  name = "add"
)
public class GitAddCommand implements Runnable {
    @Override
    public void run() {
        System.out.println("Adding some files to the staging area");
    }
}

@Command(
  name = "commit"
)
public class GitCommitCommand implements Runnable {
    @Override
    public void run() {
        System.out.println("Committing files in the staging area, how wonderful?");
    }
}
```

整个命令结构非常清楚。

那么，这些命令为什么要实现`Runnable`接口呢？这起到什么作用？Lab3需要实现三个相对独立的客户端，且功能繁多，应该如何整合设计？且听下篇分解。

## 参考资料

- [借助Graalvm和Picocli构建 Java原生CLI应用](https://www.infoq.cn/article/4RRJuxPRE80h7YsHZJtX)
- [Github - picocli](https://github.com/remkop/picocli)
- [Wikipedia - CLI](https://en.wikipedia.org/wiki/Command-line_interface)
- [W3schools - CLI](https://www.w3schools.com/whatis/whatis_cli.asp)
- [Wikipedia - Prompt](https://en.wikipedia.org/wiki/Command-line_interface#Command_prompt)
- [picocli文档](https://picocli.info/)
- [Baeldung - Create a Java Command Line Program with Picocli](https://www.baeldung.com/java-picocli-create-command-line-program)