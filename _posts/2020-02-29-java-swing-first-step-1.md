---
layout: post
title: java-swing-first-step-1
postTitle: Java swing桌面应用初探（一）——初步
categories: [Java, GUI, Software Construction]
description: 软件构造第2次博客
keywords: Java, GUI, Software Construction, Desktop Applications, Swing
mathjax: false
typora-root-url: ..
---

## 用Swing开发桌面应用

在软件构造Lab1 Problem3（朋友关系图）中，我尝试用Java实现一个有简单GUI（图形界面）的桌面应用程序。与Java Web应用不同，Java为我们提供了桌面GUI应用的开发工具包swing。它是Java基础类的一部分，使用纯Java实现，且无需依赖外部框架，可以说相当地「原生」了。

国内有高人写了一份详细的[Java Swing开发教程](https://blog.csdn.net/xietansheng/article/details/72814492)（以下简称「教程」），供参考。

组件按照不同的功能，可分为**顶层容器**、**中间容器**和**基本组件**。一个简单窗口的组成，如下层级结构所示：

- 顶层容器
  - 菜单栏
  - 中间容器
    - 基本组件A
    - 基本组件B

其中，顶层容器属于窗口类组件，可以独立显示，一个图形界面至少需要一个窗口；中间容器充当基本组件的载体，也可嵌套其他中间容器；基本组件是直接实现人机交互的组件。

swing包含了构建GUI的各种组件`Components`，如窗口（`JFrame`和`JDialog`）、标签
`JLabel`、按钮`JButton`、文本框`JText`等。本主题接下类的文章，我们还将用到表格`JTable`这一较为复杂的组件。

此外，swing还支持各种布局`Layout`以及各种其他特性，如与绘图工具包Graphics的互动。Lab1的小乌龟画图任务中，MIT的绘图动画就是用Graphics实现的。

## 构思功能和GUI布局

写前端代码之前，必须先要熟悉应用的功能和对应的API，并对GUI布局有初步的构思。

本应用要做的是，实现一个简单的图模型（朋友关系网络），要求支持三个主要功能：

- 新建结点
- 新建有向边
- 求解任意两节点间最短距离

这里，我们的后端API已经写好，即`Person`和`FriendshipGraph`两个类。

接下来，对GUI整体布局做个构思，如下图所示：

![image-20200226155421564](/images/asserts/image-20200226155421564.png)

界面左侧为功能区，从上至下分别放置三个主要功能；界面右侧为信息区，包含两个表，用于展示目前图中的节点信息和边信息。

## GUI初步建立

### 新建窗口`JFrame`

用Swing搭建GUI，首先需要建立窗口。窗口类组件属于顶层容器。像这样新建窗口并初始化：

```java
// 新建以title为标题的窗口
JFrame frame = new JFrame(title);

// 设置窗口大小
frame.setSize(1150, 600);

// 设置默认关闭方式
frame.setDefaultCloseOperation(EXIT_ON_CLOSE);
```

代码中，`JFrame`是普通窗口类。绝大多数 swing 图形界面程序都使用`JFrame`作为顶层容器。

### 添加面板`JPanel`

有了顶层容器，接下来需要新建中间容器，用以承载、管理具体组件。这里选用普通的轻量级	面板容器组件`JPanel`：

```java
// 新建面板并设置Layout为null（绝对布局）
JPanel panel = new JPanel(null);

// 向窗口中添加panel
frame.add(panel);
```

在新建面板的时候，需要指明采用何种布局方式。以我个人愚见，Java swing所提供的布局方式基本都很难用，想做更新和修改也要费很大力气。因此，不如直接采用绝对布局，通过指定各个组件的绝对坐标来布局页面。再加上一些小技巧（如使用变量储存横纵坐标值，以便移动和修改等），绝对布局反而是省心省力的了。其他的布局方式可以参看教程。

### 添加基本组件

接下来，就需要往面板中添加组件。简单起见，我们只使用三种最基本、最常用的组件：

|     组件     |  描述  |           功能           |
| :----------: | :----: | :----------------------: |
|   `JLabel`   |  标签  |      展示文字或图片      |
| `JTextField` | 文本框 |       编辑单行文本       |
|  `JButton`   |  按钮  | 用户点击时触发特定的事件 |

其他的大部分组件都是大同小异的。这三个组件的详细教程在此：[标签教程](https://blog.csdn.net/xietansheng/article/details/74362076)；[文本框教程](https://blog.csdn.net/xietansheng/article/details/74363582)；[按钮教程](https://blog.csdn.net/xietansheng/article/details/74363221)。

#### 添加标签`JLabel`

标签`JLabel`用于展示文字或图片。刚开始用swing的时候，我花了好些时间寻找如何让文字换行。事实上，不同于某些图文编辑软件中的「文本框」，`JLabel`是不能让其内容主动换行的，且其中所有内容都应用一个格式。要想实现多行、多格式效果，只能用多个`JLabel`实现。

```java
// 新建标签，文字内容为“Add a new person”
JLabel addPersonLabel = new JLabel("Add a new person");

// 设置位置和大小
addPersonLabel.setBounds(xCoord, yCoord, 200, 25);

// 设置字体
addPersonLabel.setFont(functionTitleFont);

// 向面板添加这个JLabel
panel.add(addPersonLabel);
```

#### 添加文本框`JTextField`

文本框`JTextField`用以让用户编辑单行的文本。在本应用中，用户需要输入新结点的名称等。向面板中添加文本框的代码如下：

```java
// 新建文本框，宽度为20
JTextField personNameText = new JTextField(20);

// 设置位置和大小
personNameText.setBounds(70, yCoord, 125, 25);

// 向面板添加这个JTextField
panel.add(personNameText);
```

#### 添加按钮`JButton`

按钮`JButton`在用户点击时触发特定的事件。下面是向面板添加用于新建结点的「Add」按钮的代码：

```java
// 新建按钮，设置文字为“Add”
JButton addPersonButton = new JButton("Add");

// 设置位置和大小
addPersonButton.setBounds(210, yCoord, 80, 25);

// 添加动作监听器，设置点击按钮时要做的事
addPersonButton.addActionListener(e -> {
    // 获取文本框中的文本字符串，命名为name
    String name = personNameText.getText();
    
    // 若输入为空，忽略此次点击
    if (name.length() > 0) {
        
        // 调用FriendshipGraph类的API，尝试在图中新建名为name的结点
        if (graph.addVertex(name)) {
            // 若新建成功，改变那些显示反馈信息的标签中的文字
            addPersonInfoLabel.setText("Successfully added a person named '" + name + "' !");
            printPersonLabel.setText("Total: " + graph.getVerticesNum() + " People");
        }
        else {
            // 若新建失败，显示错误信息
            addPersonInfoLabel.setText("Error: Cannot add the person.");
        }
        // 清空文本框
        personNameText.setText("");
    }
});
// 向面板添加这个JButton
panel.add(addPersonButton);
```

像这样添加其他基本组件，即可形成一个基本的功能面板，支持新建结点、新建单向边、查询任意两结点间的距离。界面如下：

![image-20200229003137329](/images/asserts/image-20200229003137329.png)

下一篇文章中，我将着重探讨如何使用表格组件`JTable`和表格模型`TableModel`进行数据的展示和实时更新，并发布这个小应用的GUI部分的源代码。

## 参考资料

- [CSDN - Java Swing开发教程](https://blog.csdn.net/xietansheng/article/details/72814492)