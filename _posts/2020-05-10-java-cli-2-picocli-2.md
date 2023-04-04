---
layout: post
title: java-cli-2-picocli-2
postTitle: Java CLI设计（二）——picocli（下）
categories: [Java, Software Construction, CLI, picocli]
description: 软件构造第9次博客
keywords: Java, Software Construction, CLI, picocli
published: true
mathjax: false
typora-root-url: ..
---

在软件构造Lab3中，我尝试用[picocli](https://github.com/remkop/picocli)搭建了一个命令行交互客户端（*Command-Line Interface*，CLI）。不同于[Common CLI](http://commons.apache.org/proper/commons-cli/)，picocli的设计更为现代化，功能更为强大，[文档](https://picocli.info/)也十分丰富。

上篇中主要介绍CLI应用的基本特征、基本概念和picocli的基本用法等内容。下篇将讲述利用picocli开发面向Lab3需求的实际应用的过程。

## 可执行命令

第一步是解析命令行参数。一个健壮的实际应用程序需要处理许多场景：

- 用户输入无效
- 用户请求使用帮助（可用于子命令）
- 运行业务逻辑（可能用于子命令）
- 业务逻辑可能会抛出异常

picocli 4.0引入了`execute`方法，可以在一行代码中处理上述所有场景。例如：

```java
new CommandLine(new MyApp()).execute(args);
```

用`execute`方法，可以用相当紧凑的应用代码运行程序：

```java
@Command(name = "myapp", mixinStandardHelpOptions = true, version = "1.0")
class MyApp implements Callable<Integer> {

    @Option(names = "-x") int x;

    @Override
    public Integer call() { // business logic
        System.out.printf("x=%s%n", x);
        return 123; // exit code
    }

    public static void main(String... args) { // bootstrap the application
        System.exit(new CommandLine(new MyApp()).execute(args));
    }
}
```

注意到`MyApp`类实现了`Callable`接口，并重写了`call`方法。要通过`execute`方法运行的命令类，必须实现`Callable`（或`Runable`）接口。用`execute`方法运行命令时，就是用运行其`call`方法（或`run`方法）。

## `Root`类

Lab3中需要实现三个应用。因此，在整个程序的入口处（即在shell中用`java -jar`命令运行的）应该让用户「选择运行哪个应用」。我把这项功能放在`Root`类中。

`Root`接受三个不同的选项，分别代表三个应用。用户从三个应用中选择一个。

```java
@Command(name = "Lab3-1183710106")
public class Root implements Callable<Integer> {

    @ArgGroup(exclusive = true, multiplicity = "1")
    Exclusive exclusive;

    // 三个应用选项互相冲突
    static class Exclusive {
        // 航班应用
        @Option(names = {"-f", "--fsa"},
                paramLabel = "FLIGHT",
                description = "run FlightScheduleApp.")
        private boolean flight;

        // 高铁车次应用
        @Option(names = {"-t", "--tsa"},
                paramLabel = "TRAIN",
                description = "run TrainScheduleApp.")
        private boolean train;

        // 学习活动应用
        @Option(names = {"-a", "--aca"},
                paramLabel = "ACTIVITY",
                description = "run ActivityCalendarApp.")
        private boolean activity;
    }

    @Option(names = {"-h", "--help"}, usageHelp = true, description = "display this help message.")
    private boolean helpRequested = false;

    @Override
    public Integer call() throws Exception {
        // call方法中，根据用户的选择分别运行不同应用的main方法
        if (exclusive.flight) {
            System.out.println("Executing Flight-Schedule-App...");
            FlightScheduleApp.main(null);
        }
        else if (exclusive.train) {
            System.out.println("Executing Train-Schedule-App...");
            TrainScheduleApp.main(null);
        }
        else if (exclusive.activity) {
            System.out.println("Executing Activity-Calender-App...");
            ActivityCalenderApp.main(null);
        }
        return 0;
    }

    public static void main(String[] args) {
        // 如果参数列表为空，则显示帮助信息
        if (args.length == 0) {
            new CommandLine(new Root()).execute("-h");
        }
        else {
            new CommandLine(new Root()).execute(args);
        }
    }
}
```

这样，在shell中，只需输入指定的选项，就能选择应用运行了。例如，输入命令

```shell
$ java -jar Lab3-1183710106 -fsa
```

就能运行航班管理应用了。

## 三个实际应用类

### 通用设计

与一般CLI应用不同的是，Lab3的使用场景不是执行「单行命令式的任务」，而是需要用户**连续地**输入命令。因此，在各个App的`main`方法中，需要用`while`循环不断地接受用户输入，手动地把输入字符串分离成参数列表，然后扔给`execute`方法运行。这部分代码如下：

```java
public static void runCliApp(Scanner scanner, CommandLine cmd, String reminder) {
    // 注册构造器
    cmd.registerConverter(Timeslot.class, Timeslot::new);
    cmd.registerConverter(EntryLabel.class, EntryLabel::new);
    cmd.registerConverter(TimeDate.class, TimeDate::new);
    // 指定运行策略：只运行最后一个命令
    cmd.setExecutionStrategy(new CommandLine.RunLast());
	// 当有下一行时，不断读取下一行输入
    while (scanner.hasNextLine()) {
        // 从scanner读取下一行
        String line = scanner.nextLine().trim();
        // 如果本行为空，忽略
        if (line.length() == 0) {
            System.out.print(reminder);
            continue;
        }
        // 用String.split方法分离参数列表，分隔符为空格和制表符\t
        String[] arguments = line.split("(( +)|(\t+))+");
        // 执行命令
        cmd.execute(arguments);

        // 每行命令执行结束后暂停100毫秒，以免出现输入输出混乱
        try {
            TimeUnit.MILLISECONDS.sleep(100);
        }
        catch (InterruptedException e) {
            System.exit(1);
        }
        // 输出提示符，如" > fsa "（fsa是FlightScheduleApp的缩写）
        System.out.print(reminder);
    }
}
```

在具体应用中，为了存储信息，需要在主命令类（如`FlightScheduleApp`）中设置一个面向应用的`PlanningEntryCollection`成员（如`FlightEntryCollection`）。例如，`FlightScheduleApp`的代码可以这样编写：

```java
// 标明命令名和子命令
@Command(name = "fsa", subcommands = {
        AddEntry.class,
    	ReadFile.class,
        ManagePlane.class,
        ManageAirport.class,
        Allocate.class,
        Cancel.class,
        Run.class,
        End.class,
        Show.class
})
public class FlightScheduleApp {

    // PlanningEntryCollection成员
    private final FlightEntryCollection collection = new FlightEntryCollection();
    
    // 提示符
    private final static String reminder = " > fsa ";


    @Option(names = {"-h", "--help"}, usageHelp = true, description = "display this help message.")
    private boolean helpRequested = false;

    // protected权限的getter方法，子命令可以通过它获取collection
    protected FlightEntryCollection getCollection() {
        return collection;
    }

    public static void main(String[] args) {
        System.out.print(reminder);
        Scanner scanner = new Scanner(System.in);
        CommandLine cmd = new CommandLine(new FlightScheduleApp());
        cmd.registerConverter(FlightNumber.class, FlightNumber::new);
        // 调用runCliApp方法运行应用
        runCliApp(scanner, cmd, reminder);
    }

}
```

而子命令则需要用`@ParentCommand`标识指明父命令类，以获得其信息。例如，`AddEntry`命令的框架是这样的：

```java
@Command(name = "add-entry", aliases = "ae", description = "Add flight planning entries.")
public class AddEntry implements Callable<Integer> {

    // 指明父命令
    @ParentCommand
    private FlightScheduleApp app;

    ...

    @Option(names = {"-h", "--help"}, usageHelp = true, description = "display this help message.")
    private boolean helpRequested = false;

    @Override
    public Integer call() {
        // 获取父命令类实例的FlightEntryCollection
        FlightEntryCollection collection = app.getCollection();
        try {
            ...
            // 执行命令动作
        }
        catch (IllegalArgumentException e) {
            ...
            // 捕获异常，异常处理
        }
        
        return 0;
    }
}
```

但这就出现一个问题：前一次执行命令时存入子命令类中的数据是不会清空的，而是直接带入后一次命令的执行中。因此，更好的方法是，把`PlanningEntryCollection`中的数据用一个临时变量储存和更新，每次执行命令前，用前一次的`PlanningEntryCollection`新建`CommandLine`类，命令执行完后，再更新到`PlanningEntryCollection`中。改进后的`FlightScheduleApp`的`main`方法如下所示：

```java
public static void main(String[] args) {
    System.out.print(reminder);
    Scanner scanner = new Scanner(System.in);
    // FlightEntryCollection临时变量
    FlightEntryCollection collection = new FlightEntryCollection();
    while (scanner.hasNextLine()) {
        String line = scanner.nextLine().trim();
        if (line.length() == 0) {
            System.out.print(reminder);
            continue;
        }
        String[] arguments = line.split("(( +)|(\t+))+");
        
        // 用前一次FlightEntryCollection新建FlightScheduleApp和对应的CommandLine
        FlightScheduleApp app = new FlightScheduleApp(collection);
        CommandLine cmd = new CommandLine(app);
        
        // 注册自动构造器，设置运行策略
        cmd.registerConverter(Timeslot.class, Timeslot::new);
        cmd.registerConverter(EntryLabel.class, EntryLabel::new);
        cmd.registerConverter(TimeDate.class, TimeDate::new);
        cmd.registerConverter(FlightNumber.class, FlightNumber::new);
        cmd.setExecutionStrategy(new CommandLine.RunLast());

        // 运行命令
        cmd.execute(arguments);
        // 更新FlightEntryCollection数据
        collection = app.getCollection();

        try {
            TimeUnit.MILLISECONDS.sleep(100);
        }
        catch (InterruptedException e) {
            System.exit(1);
        }
        System.out.print(reminder);
    }
}
```

明确了上述框架之后，三个实际应用只需要往里面塞入具体的内容即可。

### 航班管理应用`FlightScheduleApp`

我为航班管理应用`FlightScheduleApp`（简写为`fsa`）设置了9个子命令：

|   子命令类名    |         子命令          |                             功能                             |
| :-------------: | :---------------------: | :----------------------------------------------------------: |
|   `AddEntry`    |    `add-entry`, `ae`    |                          添加计划项                          |
|   `ReadFile`    |    `read-file`, `rf`    |                     从文件读入航班计划项                     |
| `ManageAirport` | `manage-airport`, `ma`  |                    管理机场（增加、删除）                    |
|  `ManagePlane`  |  `manage-plane`, `mp`   |                    管理飞机（增加、删除）                    |
|   `Allocate`    |    `allocate`, `al`     |                    为某计划项分配飞机资源                    |
|      `Run`      |        `depart`         | 某一航班计划项从始发机场起飞（`master`）<br />某一航班计划项从当前机场起飞（`314change`） |
|      `End`      |  `arrive`（`master`）   |               某一航班抵达到达机场（`master`）               |
|    `Cancel`     |        `cancel`         |                         取消某一航班                         |
|     `Block`     | `arrive`（`314change`） |             某一航班抵达下一机场（`314change`）              |
|     `Show`      |       `show`, `s`       |                    展示当前计划项集合信息                    |

需要注意的几点：

- 用户通过航班号和出发日期来唯一选择一个航班计划项

- 「展示某个地点的`Board`」的功能整合在了`show`命令中（`show --board=AIRPORT_LABEL`）

- 「列出使用指定资源的所有计划项」的功能整合在了`show`命令中（`show --only-plane=PLANE_LABEL`）。这一命令将会**按起飞时间顺序**列出所有满足条件的计划项，因此没有再显式地实现「前序计划项」功能（但`PlanningEntryAPIs`中，我编写了这个功能，并通过了测试）。

- 如需使用说明，可以使用`-h`选项：

  ```shell
   > fsa -h
  Usage: fsa [-h] [COMMAND]
    -h, --help   display this help message.
  Commands:
    add-entry, ae       Add flight planning entries.
    read-file, rf       Read flights from file.
    manage-plane, mp    Manage planes. The default operation is to add a plane,
                          and you can specify '-d' option to delete one.
    manage-airport, ma  Manage airports. The default operation is to add an
                          airport, and you can specify '-d' option to delete one.
    allocate, al        Assign a plane for a flight.
    cancel              Cancel a flight entry.
    run                 A flight departs.
    block               A flight arrives at the next airport.
    show, s             Show information structures.
  ```
  
- 每个子命令也都有使用说明。例如，`add-entry`命令（简写为`ae`）的使用说明为

  ```shell
   > fsa add-entry -h
  Usage: fsa add-entry [-h] -a=ARRIVAL_AIRPORT_LABEL -d=DEPARTURE_AIRPORT_LABEL
                       -l=FLIGHT_NUMBER [-m=MIDDLE_AIRPORT_LABEL]
                       -t=TIMESLOT_LIST [-t=TIMESLOT_LIST]...
  Add flight planning entries.
    -a, -aa, --arrival-airport=ARRIVAL_AIRPORT_LABEL
                  arrival airport label.
    -d, -da, --departure-airport=DEPARTURE_AIRPORT_LABEL
                 departure airport label.
    -h, --help   display this help message.
    -l, --label=FLIGHT_NUMBER
                 flight number of the entry.
    -m, -ma, --middle-airport=MIDDLE_AIRPORT_LABEL
                  middle airport label (optional).
    -t, --timeslots=TIMESLOT_LIST
                 timeslot list of format (yyyy-MM-dd.hh:mm,yyyy-MM-dd.hh:mm).
                 2 timeslot should be given if you specified -m option.
  ```

下面提供一组测试示例（`master`分支，`#`开头的是注释行，请勿输入）：

```shell
# 增加若干机场
manage-airport -l Beijing -la 15.78 -lo 23.56
ma -l Nanjing -la 15.78 -lo 23.56
ma -l Hefei -la 15.78 -lo 23.56
ma -l Harbin -la 15.78 -lo 23.56

# 增加若干飞机
manage-plane -l SunXiaochuan -a 10.5 -t B787 -s 300
mp -l SunChuan -a 10.6 -t B737 -s 300
mp -l SunXiao -a 10.7 -t B747 -s 300
mp -l Sun -a 10.8 -t B777 -s 300

# 查看现有的机场列表
show --airports

# 查看现有的飞机列表
show --planes

# 增加一条航班计划项（程序会检测唯一性/地点冲突）
add-entry --arrival-airport Beijing --departure-airport Nanjing --label MU5387 --timeslot (2020-05-02.11:50,2020-05-02.11:55)
# 为该航班计划项分配飞机资源（程序会检测资源冲突）
allocate --label MU5387 -t 2020-05-02 --plane Sun

# 查看现有的计划项列表
show -entries

# 尝试删除飞机Sun，预期结果为失败（已被航班使用）
mp --delete -l Sun
# 尝试删除飞机SunChuan，预期结果为成功
mp -d -l SunChuan

# 指定航班出发
depart -l MU5387 -t 2020-05-02

# 指定航班到达
arrive -l MU5387 -t 2020-05-02

# 再增加一个航班，并分配飞机资源
ae -aa Hefei -da Nanjing -l MU5379 -t (2020-05-02.11:56,2020-05-02.11:59)
al -l MU5389 -t 2020-05-02 -p SunXiao

# 显示南京机场的信息板（仅显示计划起飞/到达时间在当前系统时间前后一小时之内的）
show -b Nanjing

# 取消航班
cancel -l MU5389 -t 2020-05-02

# 读取文件中的航班计划项
read-file -f test/apps/flight/FlightSchedule_2.txt
```

对于`314change`分支，改变命令是`add-entry`：可以通过`--middle-airport`（缩写`-ma`）选项指定一个经停机场。注意，如果指定经停机场，就必须要输入两个时间段（不能相交，后一个必须晚于前一个），例如：

```shell
# 增加带经停机场的航班计划项
ae -aa Harbin -ma Beijing -da Hefei -l MU5378 -t (2020-05-02.11:56,2020-05-02.11:59) (2020-05-02.12:56,2020-05-02.12:59)
```

### 高铁车次管理应用`TrainScheduleApp`

我为高铁车次管理应用`TrainScheduleApp`（简写为`tsa`）设置了8个子命令：

|    子命令类名    |         子命令          |               功能               |
| :--------------: | :---------------------: | :------------------------------: |
|    `AddEntry`    |    `add-entry`, `ae`    |            添加计划项            |
| `ManageStation`  | `manage-station`, `ms`  |      管理车站（增加、删除）      |
| `ManageCarriage` | `manage-carriage`, `mc` |      管理车厢（增加、删除）      |
|    `Allocate`    |    `allocate`, `al`     |      为某计划项分配车厢资源      |
|      `Run`       |        `depart`         | 某一高铁车次计划项从当前车站出发 |
|     `Cancel`     |        `cancel`         |           取消某一车次           |
|     `Block`      |        `arrive`         |  某一高铁车次计划项抵达下一车站  |
|      `Show`      |       `show`, `s`       |      展示当前计划项集合信息      |

需要注意的几点：

- 用户通过车次号和出发日期来唯一选择一个高铁车次计划项

- 展示某个地点的`Board`的功能整合在了`show`命令中（`show --board=STATION_LABEL`）

- 「列出使用指定资源的所有计划项」的功能整合在了`show`命令中（`show --only-carriage=CARRIAGE_LABEL`）。这一命令将会**按发车时间顺序**列出所有满足条件的计划项，因此没有再显式地实现「前序计划项」功能（但`PlanningEntryAPIs`中，我编写了这个功能，并通过了测试）。

- 如需使用说明，可以使用`-h`选项：

  ```shell
   > tsa -h
  arguments = [-h]
  Usage: tsa [-h] [COMMAND]
    -h, --help   display this help message.
  Commands:
    add-entry, ae        Add train planning entries.
    manage-carriage, mc  Manage carriages. The default operation is to add a
                           carriage, and you can specify '-d' option to delete
                           one.
    manage-station, ms   Manage stations. The default operation is to add an
                           station, and you can specify '-d' option to delete one.
    allocate, al         Assign carriages for a train.
    cancel               Cancel a train entry.
    depart               A train departs.
    arrive               A train arrives at the next station. A message of
                           current station and whether the train has arrived at
                           the end station or not will be given after
                           successfully executed.
    show, s              Show information structures.
  ```

- 每个子命令也都有使用说明。使用方法同`FlightScheduleApp`。

下面提供一组测试示例（`master`分支，`#`开头的是注释行，请勿输入）：

```shell
# 增加若干车站
manage-station -l Beijing -la 15.78 -lo 23.56
ms -l Nanjing -la 15.78 -lo 23.56
ms -l Hefei -la 15.78 -lo 23.56
ms -l Harbin -la 15.78 -lo 23.56

# 增加若干车厢
manage-carriage -l 001 -r 2011 -t 1 -c 300
mc -l 002 -r 2012 -t 2 -c 300
mc -l 003 -r 2013 -t 3 -c 300
mc -l 004 -r 2014 -t 4 -c 300

# 当前车站列表
show --stations

# 当前车厢列表
show --carriages

# 增加一条高铁计划项，注意车站数量需要比时间段数量恰好多1
add-entry --label G1024 --stations Beijing Nanjing Harbin Hefei --timeslots (2020-05-07.20:00,2020-05-07.20:01) (2020-05-07.20:02,2020-05-07.20:03) (2020-05-07.20:04,2020-05-07.20:05)

# 显示南京站的信息板（仅显示计划从本站出发/到达本站的时间在当前系统时间前后一小时之内的）
show --board Nanjing
# 显示合肥站的信息板
show -b Hefei

# 再增加一条高铁计划项
ae -l G1025 -s Harbin Nanjing Beijing Hefei -t (2020-05-07.20:00,2020-05-07.20:01) (2020-05-07.20:02,2020-05-07.20:03) (2020-05-07.20:04,2020-05-07.20:05)

# 取消G1024
cancel -l G1024 -t 2020-05-07

# 为G1025分配两节车厢001和002
allocate -l G1025 -t 2020-05-07 -c 001 002

# 显示当前所有计划项
show --entries

# 从起始站出发
depart -l G1025 -t 2020-05-07
# 到达第二站（状态为BLOCKED）
arrive -l G1025 -t 2020-05-07
# 从第二站出发
depart -l G1025 -t 2020-05-07
# 到达第三站（状态为BLOCKED）
arrive -l G1025 -t 2020-05-07
# 从第三站出发
depart -l G1025 -t 2020-05-07
# 到达第四站（也是终点站，状态为ENDED）
arrive -l G1025 -t 2020-05-07
```

### 学习活动管理应用`ActivityCalendarApp`

我为学习活动管理应用`ActivityCalendarApp`（简写为`aca`）设置了8个子命令：

|  子命令类名  |       子命令        |              功能               |
| :----------: | :-----------------: | :-----------------------------: |
|  `AddEntry`  |  `add-entry`, `ae`  |           添加计划项            |
| `ManageRoom` | `manage-room`, `mr` |    管理会议室（增加、删除）     |
|  `Allocate`  |  `allocate`, `al`   |     为某计划项分配材料资源      |
|    `Run`     |        `run`        |          某一活动开始           |
|    `End`     |        `end`        |          某一活动结束           |
|   `Cancel`   |      `cancel`       |          取消某一活动           |
|   `Block`    |       `block`       | 某一活动暂停（仅限`314change`） |
|    `Show`    |     `show`, `s`     |     展示当前计划项集合信息      |

需要注意的几点：

- 用户通过车次号和出发日期来唯一选择一个高铁车次计划项

- 展示某个地点的`Board`的功能整合在了`show`命令中（`show --board=STATION_LABEL`）

- `ActivityCalendarApp`没有「管理会议材料资源」子命令，因为材料`Material`类不可分辨。分配材料资源时，只能多次使用`allocate`子命令的`-m`选项逐条输入

- 如需使用说明，可以使用`-h`选项：

  ```shell
   > aca -h
  Usage: aca [-h] [COMMAND]
    -h, --help   display this help message.
  Commands:
    add-entry, ae    Add activity planning entries.
    manage-room, mr  Manage rooms. The default operation is to add a room, and
                       you can specify '-d' option to delete one.
    allocate, al     Assign a material for a activity.
    cancel           Cancel a activity entry.
    run              A activity departs.
    end              A activity finished.
    show, s          Show information structures.
  ```

- 每个子命令也都有使用说明。使用方法同前两个app。

下面提供一组测试示例（`master`分支，`#`开头的是注释行，请勿输入）：

```shell
# 添加若干会议室
manage-room -l EG001 -la 15.78 -lo 23.56
mr -l WS002 -la 15.78 -lo 23.56
mr -l MD003 -la 15.78 -lo 23.56
mr -l ZX004 -la 15.78 -lo 23.56

# 当前所有会议室列表
show --rooms

# 添加若干学习活动计划项
add-entry -l aaaa -r EG001 -t (2020-05-08.17:00,2020-05-08.17:01)
ae -l bbbb -r EG001 -t (2020-05-08.17:01,2020-05-08.17:02)
ae -l dddd -r EG001 -t (2020-05-08.16:51,2020-05-08.16:52)

# 展示会议室EG001的信息板
show --board EG001

# 为aaaa分配两份材料
al -l aaaa -t 2020-05-08 -m m01 -d d01 -r 2020-04-01 -m m02 -d d02 -r 2020-04-01

# 开始活动
run -l aaaa -t 2020-05-08

# 结束活动
end -l aaaa -t 2020-05-08

# 取消另一项活动
cancel -l bbbb -t 2020-05-08

# 展示所有计划项信息
show --entries
```

## 参考资料

- [借助Graalvm和Picocli构建 Java原生CLI应用](https://www.infoq.cn/article/4RRJuxPRE80h7YsHZJtX)
- [Github - picocli](https://github.com/remkop/picocli)
- [Wikipedia - CLI](https://en.wikipedia.org/wiki/Command-line_interface)
- [W3schools - CLI](https://www.w3schools.com/whatis/whatis_cli.asp)
- [Wikipedia - Prompt](https://en.wikipedia.org/wiki/Command-line_interface#Command_prompt)
- [picocli文档](https://picocli.info/)
- [Baeldung - Create a Java Command Line Program with Picocli](https://www.baeldung.com/java-picocli-create-command-line-program)