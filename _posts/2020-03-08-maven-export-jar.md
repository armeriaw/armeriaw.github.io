---
layout: post
title: maven-export-jar
postTitle: 纯Maven一键导出jar流程
categories: [Java, Maven, Software Construction]
description: 软件构造第3次博客
keywords: Java, Software Construction, Maven
published: true
mathjax: false
typora-root-url: ..
---

<div style='display: none'>

# 纯Maven一键导出jar流程
**目录**
[TOC]

</div>


## 0 要干什么

这套流程的目标是用Maven实现高效的一键build。最后，**你只需用一条指令（或一次点击），就能完全按照实验要求，在正确的目录结构下，自动化地编译、测试、打包jar文件，且能够在根目录下正常运行各P的jar。**

以下是你可能会问的一些问题：

1. - **Q：Maven是什么？**
   - **A**：Maven是专门为Java项目打造的管理和构建工具。它像是一条标准化的流水线，只要我们把源代码、测试数据按照Maven的要求组织好，它就能自动完成、编译、测试、打包等整套流程。此外，它还是个包管理器，帮你打理好你所需的各种外部库。

2. - **Q：为什么要用Maven？**
   - **A**：因为它极为高效，而且简单。  

3. - **Q：Maven好学吗？**
   - **A**：非常好学。我们所要做的几乎只是写好`pom.xml`这一个配置文件而已，而且本文还会提供亲测有效的模板。Maven也是早晚都得用的东西，不如趁早把它学了，提高今后的开发效率。

4. - **Q：我已经基本写完了实验，换Maven麻烦吗，值得换吗？**
   - **A**：不麻烦。你只需要备份好`src`、`test`等文件夹。当然，如果确定实验代码不用再改，也不认为自己目前的打包流程麻烦，也可不用Maven，或等到之后的实验再尝试。

<div style='display: none'>

### V1.1更新说明
- V1.0版的流程可能出现运行`Px.jar`后无法正常向命令行中输入和交互的情况。这一版中，增加了`Terminal`类，修复了上述问题。
- 其他格式和表述改进。

### V1.2更新说明
- V1.1版的流程可能出现运行`Px.jar`后无法自动终止的情况。这一版中，改进了`Terminal`类，修复了上述问题。
- 增加了目录。
- 其他格式和表述改进。
- 本文最新版[在线阅读点我](http://armeriaw.github.io/2020/03/08/maven-export-jar/)。
- **今后本文仅在线上更新，不再发布pdf版**。

</div>

## 1 准备工作

### 1.1 所需材料

- 已经基本写好的各P代码（包括可以正常运行的`main`函数）。
- 安装好Maven的Eclipse或IntelliJ IDEA。目前新版的Eclipse和IDEA都是有自带Maven的，不用额外下载安装。不过，自带的版本不是最新。如果你想配置最新版，或者你无法使用自带版本，可以搜索相关安装教程。文末也会给出几篇高质量教程的链接。

### 1.2 建立Maven项目

**注意：在继续流程之前，请务必备份自己的项目，尤其是`src`和`test`目录下所有已完成的代码、`doc`目录下的实验报告等！**

#### 1.2.1 新建项目

备份完成之后，移走整个`Lab1-XXX`文件夹。然后在同样的目录下新建一个同名的空文件夹。打开IDE，以这个文件夹为根目录新建Maven项目。

**Eclipse操作**

File > New > Project... > Maven > Maven Project，下一步勾选「Create a simple project」；Group Id、Artifact Id随意，Parent Project全部留空，其余参数保持默认。

新建完成后，删除IDE自动构建的`src`、`test`文件夹，把先前备份的`src`和`test`复制进来。然后别忘了在这个文件夹下初始化一下git。

**IDEA操作**

在新建项目窗口左侧点击Maven即可。其余参考Eclipse操作。

#### 1.2.2 配置`pom.xml`

`pom.xml`是Maven的配置文件，其中包含了项目构建设置、项目所需的外部依赖库（如JUnit）信息等。IDE已经帮我们创建了`pom.xml`，现在我们对它进行修改。关于这个文件的详细功能，可以直接用搜索引擎搜索`pom.xml`。这套流程中，不必对每个配置标签都了解得很清楚。

`xml`文件很像是一棵树，用尖括号`<>`括起来的叫**标签**，它们总是成对出现，分别标志着这个元素的开始和结束。如：

```xml
<root>    <!-- root元素开始 -->
  <child>
    <subchild>.....</subchild>
  </child>
</root>   <!-- root元素结束 -->
```

在`pom.xml`的`<project>`根标签中找到或添加`<proprties>`标签，设置项目编码为UTF-8，

```xml
<project>
    ...
    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
    ...
</project>
```

在根标签中找到或添加`<dependencies>`标签，添加JUnit依赖，

```xml
<project>
    ...
    <dependencies>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13</version>
            <scope>compile</scope>
        </dependency>
        
    </dependencies>
    ...
</project>
```

接着找到或添加`<build>`标签，添加Maven插件，

```xml
<project>
    ...
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>10</source>  <!-- 或改成你使用的JDK版本 -->
                    <target>10</target>  <!-- 或改成你使用的JDK版本 -->
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.0.0-M4</version>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>  <!-- 重设test文件模板 -->
                    </includes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <!-- 指定输出目录为项目根目录 -->
                            <outputDirectory>${basedir}</outputDirectory>
                            <!-- 指定输出文件名 -->
                            <outputFile>Lab1-XXX.jar</outputFile>
                            <createDependencyReducedPom>false</createDependencyReducedPom>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>root.Root</mainClass>
                                    <!-- root.Root是之后要创建的入口类，若IDE提示错误可忽略 -->
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            
        </plugins>
    </build>
    ...
</project>
```

最后，由于实验要求的目录结构和Maven的默认设置有差别，所以我们需要在`pom.xml`中更改。还是找到`<build>`标签，加入如下两行，

```xml
<build>
    
    <sourceDirectory>src</sourceDirectory>
    <testSourceDirectory>test</testSourceDirectory>
    
    ......

</build>
```

这两行分别设置源代码目录和测试目录。

实验要求把P1、P2、P3分开打包，但正常情况下，**Maven一次**（一个LifeCycle内）**只能以某一个class为主类，打包一个jar**。如果我们分别以三个问题的三个类为主类分别打包，就要在每次build前修改设置和路径，很麻烦。有没有一劳永逸的办法呢？

## 2 建立辅助类

为了完美符合实验要求，需要实现三个辅助类。在`src`目录下新建一个`root`文件夹，在里面新建三个类，分别命名为`Root`、`Terminal`和`RunPx`。

### 2.1 入口类`Root`

`Root`是入口类，我们希望通过它，能够自由选择P1、P2、P3的程序运行。代码如下，

```java
/*
    /src/root/Root.java
*/

package Root;

import P1.MagicSquare;
import P2.turtle.TurtleSoup;
import P3.FriendshipGraph;

import java.io.FileNotFoundException;
import java.util.HashSet;
import java.util.Scanner;
import java.util.Set;
import java.util.concurrent.TimeUnit;

public class Root {
    public static void main(String[] args) throws InterruptedException {
        int pid;
        
        // 解析参数列表args
        if (args.length > 0) {
            try {
                pid = Integer.parseInt(args[0]);
            }
            catch (NumberFormatException e) {
                System.out.println("Invalid Problem ID!");
                pid = -1;
            }
            if (!((pid >= 1 && pid <= 3)) {
                System.out.println("Invalid Problem ID!");
                pid = -1;
            }
        }
        else {
            pid = -1;
        }
        
        // 根据args参数指定的题目编号，运行对应的程序
        if (pid == 1) {
            try {
                // 运行P1的main函数，之后的Lab改成另外的名称即可，以下同理
                MagicSquare.main(null);
            }
            catch (FileNotFoundException e) {
                System.out.println("Error: File Not Found!\n");
            }
        }
        else if (pid == 2) {
            // 运行P2的main函数
            TurtleSoup.main(null);
        }
        else if (pid == 3) {
            // 运行P3的main函数
            FriendshipGraph.main(null);
        }
        System.out.println();
    }
}
```

这个`Root`类打包成jar后，可以实现通过参数列表的第一个值指定运行哪个程序。例如，命令

```shell
$ java -jar ./Lab1-XXX.jar 1
```

可以实现P1的运行。

### 2.2 Maven自动打包

到这里，我们可以先试一下Maven强大的自动化流水线。

**IDE Maven**

打开IDE的Maven窗口（以IDEA为例），先双击clean按钮（可选，清除之前的build文件），再双击package按钮，让Maven自动实现从编译compile到测试test和打包package的全过程。不出意外，你将在项目根目录下发现Maven打包的`Lab1-XXX.jar`。

![image-20200308192744951](/images/asserts/image-20200308192744951.png)

**命令行Maven**

如果你设置了Maven的环境变量，就可以在命令行中使用Maven。在根目录下键入如下命令

```shell
$ mvn clean package
```

即可自动完成从clean到package的全过程。

### 2.3 模拟cmd类Terminal

为了之后能正常与命令行交互，还是在`root`目录下，新建类`Terminal`，以模拟cmd的运行，

```java
/*
    /src/root/Terminal.java
*/

package root;

import java.io.*;
import java.nio.charset.StandardCharsets;

public class Terminal {

    static class ReaderConsole implements Runnable {
        private InputStream is;

        public ReaderConsole(InputStream is) {
            this.is = is;
        }

        public void run() {
            InputStreamReader isr = null;
            isr = new InputStreamReader(is, StandardCharsets.UTF_8);
            BufferedReader br = new BufferedReader(isr);

            int c = -1;
            try {
                while ((c = br.read()) != -1) {
                    System.out.print((char) c);
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    static class WrittenConsole implements Runnable {
        private OutputStream os;

        public WrittenConsole(OutputStream os) {
            this.os = os;
        }

        public void run() {
            try {
                while (true) {
                    String line = this.getConsoleLine();
                    line += "\n";
                    os.write(line.getBytes());
                    os.flush();
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        private String getConsoleLine() throws IOException {
            String line = null;
            InputStreamReader input = new InputStreamReader(System.in);
            BufferedReader br = new BufferedReader(input);
            line = br.readLine();
            return line;
        }
    }

    public void execute(String cmd) throws Exception {
        Process process = Runtime.getRuntime().exec(cmd);
        InputStream is = process.getInputStream();
        OutputStream os = process.getOutputStream();

        // 输入流线程
        Thread t1 = new Thread(new ReaderConsole(is));
        
        // 输出流线程
        Thread t2 = new Thread(new WrittenConsole(os));
        
        t1.start();
        t2.start();
    }

} 
```

需要注意的是，由于`while`循环没有设置终止条件，所以这个`Terminal`类有时无法自动结束，需要`Ctrl+C`强制结束。如果不介意这一点，可以直接进入下一节。

事实上，笔者研究出了一种自动结束的实现方法。注意到调用的`main`函数结束时，当前进程的`InputStream`会出现`EOF`符（ASCII码为0）。利用这一点，我们可以设置一个`isRunning`标记，用作终止条件。因此，需要把两个`Console`的`run`方法修改如下：

```java
package root;

import java.awt.*;
import java.awt.event.KeyEvent;
import java.io.*;
import java.nio.charset.StandardCharsets;

public class Terminal {

    // 终止标记isRunning
    private volatile boolean isRunning;

    class ReaderConsole implements Runnable {

        ...
            
        public void run() {
            InputStreamReader isr = null;
            isr = new InputStreamReader(is, StandardCharsets.UTF_8);
            BufferedReader br = new BufferedReader(isr);

            int c = -1;
            try {
                while ((c = br.read()) != -1) {
                    System.out.print((char) c);
                }
            }
            catch (Exception e) {
                e.printStackTrace();
            }
            // 当前进程输出结束，isRunning标记设为false
            isRunning = false;
        }
    }

    class WrittenConsole implements Runnable {

        ...

        public void run() {
            try {
                // 将isRunning作为线程终止条件
                while (isRunning) {
                    String line = this.getConsoleLine();
                    line += "\n";
                    
                    // 读入一行后及时检查isRunning标记
                    if (!isRunning) {
                        break;
                    }
                    
                    os.write(line.getBytes());
                    os.flush();
                }
            }
            catch (Exception e) {
                e.printStackTrace();
            }
        }

        ...
    }

    public void execute(String cmd) throws Exception {

        // 初始化isRunning为true
        isRunning = true;
        
        ...
    }

}
```

不过我们发现，即使改成这样，由于`BufferedReader.readLine()`方法是阻塞方法（在没接收到换行符`\n`前会一直等待），`WrittenConsole`需要多输入一次回车才能结束。这是一个非常令人困扰的问题，CSDN论坛和StackOverflow上对类似的主题有过非常多的讨论，但都没能得出有效的解决方案。有办法避免吗？笔者多方查阅资料、不断尝试，终于找到一种方便但不那么优雅的解决方式——用JDK自带的`Robot`类，模拟键盘输入一个回车！

`Robot`本来是用于测试自动化、自运行演示程序和其他需要控制鼠标和键盘的应用程序生成本机系统输入事件的类。这里，我们用它来四两拨千斤地修复「多一个回车」这个小bug。事实上，我们在`ReaderConsole`的`run`方法中再多加两行代码即可：

```java
class ReaderConsole implements Runnable {

    ...

    public void run() {
        InputStreamReader isr = null;
        isr = new InputStreamReader(is, StandardCharsets.UTF_8);
        BufferedReader br = new BufferedReader(isr);

        int c = -1;
        try {
            while ((c = br.read()) != -1) {
                System.out.print((char) c);
            }
            // 创建新Robot实例
            Robot robot = new Robot();
            // 模拟键盘输入回车
            robot.keyPress(KeyEvent.VK_ENTER);
        }
        catch (Exception e) {
            e.printStackTrace();
        }
        isRunning = false;
    }
    
}
```

这样，我们就完美地实现了各P程序的单独运行。

改进后的`Terminal`类的完整代码展示如下，

```java
/*
    /src/root/Terminal.java
*/

package root;

import java.awt.*;
import java.awt.event.KeyEvent;
import java.io.*;
import java.nio.charset.StandardCharsets;

public class Terminal {

    private volatile boolean isRunning;

    class ReaderConsole implements Runnable {

        private InputStream is;

        public ReaderConsole(InputStream is) {
            this.is = is;
        }

        public void run() {
            InputStreamReader isr = null;
            isr = new InputStreamReader(is, StandardCharsets.UTF_8);
            BufferedReader br = new BufferedReader(isr);

            int c = -1;
            try {
                while ((c = br.read()) != -1) {
                    System.out.print((char) c);
                }
                Robot robot = new Robot();
                robot.keyPress(KeyEvent.VK_ENTER);
            }
            catch (Exception e) {
                e.printStackTrace();
            }
            isRunning = false;
        }
    }

    class WrittenConsole implements Runnable {

        public OutputStream os;

        public WrittenConsole(OutputStream os) {
            this.os = os;
        }

        public void run() {
            try {
                while (isRunning) {
                    String line = this.getConsoleLine();
                    line += "\n";
                    if (!isRunning) {
                        break;
                    }
                    os.write(line.getBytes());
                    os.flush();
                }
            }
            catch (Exception e) {
                e.printStackTrace();
            }
        }

        private String getConsoleLine() throws IOException {
            String line = null;
            InputStreamReader input = new InputStreamReader(System.in);
            BufferedReader br = new BufferedReader(input);
            line = br.readLine();
            return line;
        }
    }

    public void execute(String cmd) throws Exception {

        isRunning = true;

        Process process = Runtime.getRuntime().exec(cmd);
        InputStream is = process.getInputStream();
        OutputStream os = process.getOutputStream();

        Thread t1 = new Thread(new ReaderConsole(is));
        Thread t2 = new Thread(new WrittenConsole(os));
        t1.start();
        t2.start();
    }

}
```

### 2.4 命令运行类`RunPx`

接下来要实现P1、P2、P3的分开打包。我们只需额外打包三个类，利用入口`Root`分别运行三个对应的命令即可。**这三个jar只要第一次打包完成后放到根目录下就行，之后再也不用管它们，再也不用任何的修改**。我们在`root`目录下新建`RunPx`类，代码如下，

```java
/*
    /src/root/RunPx.java
*/

package root;

import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;

public class RunPx {
    public static void main(String[] args) throws IOException {
        
        // pid为当前要打包的类
        int pid = 1;
        
        // 要输入的命令
        String cmd = "java -jar ./Lab1-XXX.jar " + pid;
        System.out.println(cmd);
        
        // 利用Terminal类执行命令
        Terminal terminal = new Terminal();
        terminal.execute(cmd);
    }
}
```

需要注意的是，如果你编写了GUI客户端程序，希望可以直接点开`.jar`文件运行，那么就不要调用`Terminal`类。如果调用，会发生「关不掉」的情况，因为后台的命令行程序始终在运行。

将`pom.xml`中的`<mainClass>`改为`root.RunPx`，`<outputFile>`先后设为`P1.jar`、`P2.jar`、`P3.jar`，`RunPx`类的值对应地先后设为1、2、3，分别用Maven打出三个包即可。

不出意外，在根目录下运行命令

```shell
$ java -jar P1.jar
```

即可运行P1的程序了。

完成`P1.jar`、`P2.jar`和`P3.jar`的打包之后，别忘了将`pom.xml`中的`<mainClass>`和`<outputFile>`改回原先基于入口类`Root`的值。今后，无论怎么修改代码，都只需要更新基于`Root`的`Lab1-XXX.jar`这一个包。所以，用Maven的一键package流即可完成编译和打包了。

对于Lab2、Lab3等之后的实验，也只需将所有Lab1改成LabX，并修改`Root`入口类中各`main`方法所属的类名做对应的更改即可，对附加代码几乎不用做任何大的改动，非常方便。

## 3 附录

### 3.1 `pom.xml`完整模板

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>Lab1-XX</groupId>
    <artifactId>Lab1-XXX</artifactId>
    <version>1.0-SNAPSHOT</version>

    <build>

        <sourceDirectory>src</sourceDirectory>
        <testSourceDirectory>test</testSourceDirectory>

        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>10</source>
                    <target>10</target>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.0.0-M4</version>
                <configuration>
                    <includes>
                        <include>**/*Test.java</include>
                    </includes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${basedir}</outputDirectory>
                            <outputFile>Lab1-XXX.jar</outputFile>
                            <createDependencyReducedPom>false</createDependencyReducedPom>
                            <transformers>
                                <transformer  implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>root.Root</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

        </plugins>

    </build>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13</version>
            <scope>compile</scope>
        </dependency>
    </dependencies>

</project>
```

### 3.2 Lab1目录结构参考

```
Lab1-XXX
|-- src
|---|-- P1
|---|---|-> MagicSquare.java
|---|---|-- txt

|---|-- P2
|---|-- rules
|---|-- turtle
|---|---|-> TurtleSoup.java
|---|---|-  ...

|---|-- P3
|---|---|-> Person.java
|---|---|-> FriendshipGraph.java

|---|-- root
|---|---|-> Root.java
|---|---|-> RunPx.java
|---|---|-> Terminal.java

|-- test
|---|-- P1
|---|-- P2
|---|-- P3

|-- lib

|-- target

|-- doc

|-> Lab1-XXX.jar
|-> P1.jar
|-> P2.jar
|-> P3.jar
```

### 3.3 参考资料

- [廖雪峰的Maven教程](https://www.liaoxuefeng.com/wiki/1252599548343744/1255945359327200)

- [CSDN - Eclipse使用Maven教程](https://blog.csdn.net/learn_tech/article/details/82491412)

- [CSDN - Windows下Eclipse整合Maven](https://www.cnblogs.com/sdf-txt/p/7705285.html)

- [CSDN - IDEA配置Maven](https://blog.csdn.net/qq_34598667/article/details/92968550)

- [CSDN - Maven打包配置说明](https://blog.csdn.net/sdut406/article/details/89211538)

