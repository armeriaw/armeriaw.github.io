---
layout: post
title: java-swing-first-step-3
postTitle: Java swing桌面应用初探（三）——棋类游戏
categories: [Java, GUI, Software Construction]
description: 软件构造第5次博客
keywords: Java, GUI, Software Construction, Desktop Applications, Swing
mathjax: false
typora-root-url: ..
---

## 棋类游戏GUI

我针对软件构造Lab2的P3棋类游戏做了一个简易的GUI客户端，其主要界面如下。

### 开始界面

![TIM截图20200327230946](https://i.loli.net/2021/06/15/k5VCy4zRHrJlQSm.png)

运行客户端后显示开始界面，可以选择国际象棋（Chess）或围棋（Go）之一游玩。

### 玩家姓名输入界面

![TIM截图20200327230955](https://i.loli.net/2021/06/15/z1seBgiZ4ovI6bM.png)

游戏开始前，要求双方玩家输入姓名。界面上方的两行文字给出了对输入的要求。

### 国际象棋游戏界面

![TIM截图20200327231014](https://i.loli.net/2021/06/15/p2nhkAaNZsIbw87.png)

左侧显示当前局面的棋盘棋子，右侧为操作面板。

右侧上方显示双方玩家各自的棋子数，下面是当前游戏轮数、当前玩家姓名和所执棋子颜色（W为白色，B为黑色）。接着是两对（4个）输入框，分别对应移动棋子的起终点横纵坐标。

界面中有三个按钮，各自功能如下：

- **Action**。根据输入框中的坐标执行动作（自动判断是「吃子」还是「移子」）。
- **Skip**。当前玩家不做动作，跳过本轮。
- **End**。弹出对话框询问玩家是否终止当前游戏：若是，展示游戏历史记录；若否，返回游戏界面继续游戏。

此外，点击窗口右上角的「×」按钮，效果与点击「End」按钮相同。

### 围棋游戏界面

![TIM截图20200327231052](https://i.loli.net/2021/06/15/xPgImzHudOLZ1NJ.png)

围棋游戏界面与国际象棋相似。不同点有：

- 仅有一对（两个）坐标输入框。
- 国际象棋中的「Action」按钮被拆成「Place」和「Remove」两个按钮，分别对应「落子」和「提子」两个功能。

### 游戏历史记录显示界面

![TIM截图20200327231027](https://i.loli.net/2021/06/15/lPoiGfW3Cwnd2qH.png)

若玩家终止游戏，将显示游戏历史记录，用纯文本形式呈现。支持拖拽滑块浏览长文本。点击界面下方的「OK」，彻底关闭游戏。

## 国际象棋游戏界面实现

### 界面与交互设计

我们见到的计算机上的棋类游戏程序，一般都是展示出棋盘，玩家直接点击棋盘上的位置，以进行落子等操作。限于本人水平和实验时间限制，本棋类游戏GUI客户端没有实现点击操作的功能，而是通过「输入坐标、点击按钮」的方式间接地进行游戏动作。玩家每次执行动作成功，就立即更新棋盘和棋子的画面。这类似于命令行的操作方式，但更直观易用。

基于以上构思，我将游戏界面划分为左右两个部分：

- 左边是**棋局面板**，展示棋局画面
- 右边是**操作面板**，信息展示和交互组件

### 分层面板`JLayeredPane`

为了将背景、棋盘、棋子和其他各组件有序地摆放在窗口中，避免出现意外的覆盖重叠的情况，宜采用分层面板`JLayeredPane`实现界面。相关教程[见此](https://blog.csdn.net/xietansheng/article/details/74366560)。

简单来说，`JLayeredPane`为容器添加了深度，允许组件在需要时互相重叠。它将深度范围按**层**划分，在同一层内又对组件按位置进一步划分，将组件放入容器时需要指定组件所在的层，以及组件在该层内的**位置**。

我们可以像这样新建窗口和分层面板：

```java
// 新建窗口，命名为frame
frame = new JFrame(title);
setFrame(frame, game, frameWidth, frameHeight);

// 新建分层面板，命名为panel
JLayeredPane panel = new JLayeredPane();

// 将frame的内容面板设置为panel
frame.setContentPane(panel);

// 通过addLayers方法向panel中添加layer
addLayers(panel);
```

`addLayers`方法中，我们需要向`panel`中添加层。在`JLayeredPane`中，层的编号越大显示在越前面；在同层内，位置编号越大越靠近底部。通过`setLayer`方法可设置组件所在的层数，同一层内的组件，可通过调用`moveToFrontComponent`、`moveToBack`和`setPosition`调整层内的位置。

下面给出`addLayers`方法的片段：

```java
// 新建背景色面板backgroundPanel
backgroundPanel = createPanel(Color.DARK_GRAY, 0, 0, frameWidth, frameHeight);
// 将背景色面板放在分层面板的最底层
layeredPane.add(backgroundPanel, JLayeredPane.DEFAULT_LAYER);

...
    
// 新建棋盘面板（仅放置棋盘图片这一个组件）
boardPanel = new JPanel(null);
// 添加图片标签
boardPanel.add(getBoardImageLabel(0, 0, boardWidth, boardHeight, resourcePath, "ChessBoard.png"));
// 设置位置和边界
boardPanel.setBounds(boardX, boardY, boardWidth, boardHeight);
// 设为透明
boardPanel.setOpaque(false);
// 将棋盘面板放在次底层
layeredPane.add(boardPanel, JLayeredPane.PALETTE_LAYER);

// 新建棋子面板，并将其放在棋盘面板的上面
piecesPanel = new JPanel(null);
piecesPanel.setBounds(boardX, boardY, boardWidth, boardHeight);
placeChessPieceLabels(piecesPanel);
setChessPieceImages(game.getBoard());
piecesPanel.setOpaque(false);
layeredPane.add(piecesPanel, JLayeredPane.MODAL_LAYER);

...
```

需要注意的是，添加到`JLayeredPane`内的组件需要明确指定组在位置和宽高，否则将不予显示（类似绝对布局）。

下面依次介绍棋局面板、操作面板和游戏历史记录界面的设计和实现。

### 棋局面板

国际象棋棋局面板由棋盘、棋子和行列坐标三部分组成。由于我们需要准确地展示棋局，所以必须对这些组件在界面中的大小和位置进行精确的控制。

首先在网络上找素材。对于国际象棋，棋盘素材图最好是规整的、无边框的方格布局；棋子素材要保证图样周围是透明的，以免遮盖棋盘。

可以用如下方法获取棋盘图片标签：

```java
/**
 * 获取棋盘图片标签
 *
 * @param x 横坐标
 * @param y 纵坐标
 * @param width 宽度
 * @param height 高度
 * @param resourcePath 路径
 * @param fileName 文件名
 * @return 棋盘图片标签
 */
protected static JLabel getBoardImageLabel(int x, int y, int width, int height,
                                           String resourcePath, String fileName) {
    // 新建棋盘图片标签
    JLabel boardImage = new JLabel();
    
    // 用awt中的Toolkit获取棋盘图片对象
    Image image = Toolkit.getDefaultToolkit().getImage(resourcePath + fileName);
    
    // 对图片进行缩放，缩放算法用SCALE_SMOOTH
    image = image.getScaledInstance(width, height, Image.SCALE_SMOOTH);
    
    // 将棋盘图片标签的内容设为放缩好的图片
    boardImage.setIcon(new ImageIcon(image));
    
    // 设置位置和大小并返回
    boardImage.setBounds(x, y, width, height);
    return boardImage;
}
```

对于棋子，我们可以在GUI类中设置一个私有数组，用于存储棋子的图片标签。

```java
// 棋子图片标签数组
private JLabel pieceImageLabels[][] = new JLabel[8][8];

// 棋盘单个方格的边长（单位默认像素）
private static int squareSize = 70;

// 设置图片标签数组中各标签的属性（位置、大小、对齐方式等）
private void placeChessPieceLabels(JPanel panel) {
    for (int i = 0; i < 8; i++) {
        for (int j = 0; j < 8; j++) {
            Position position = new Position(i, j);
            pieceImageLabels[i][j] = new JLabel();
            JLabel label = pieceImageLabels[i][j];
            // 根据棋盘坐标获取棋子标签的位置和大小
            label.setBounds(getChessPieceImageRectangle(position));
            label.setHorizontalAlignment(SwingConstants.LEFT);
            label.setVerticalAlignment(SwingConstants.TOP);
            panel.add(label);
        }
    }
}

// 获取棋子标签的位置和大小
private Rectangle getChessPieceImageRectangle(Position position) {
    return new Rectangle(squareSize * position.getY(),
                         squareSize * (7 - position.getX()),
                         squareSize, squareSize);
}

// 设置图片标签数组中各标签所显示的内容（棋子图片或空）
private void setChessPieceImages(ChessBoard board) {
    for (int i = 0; i < 8; i++) {
        for (int j = 0; j < 8; j++) {
            Position position = new Position(i, j);
            JLabel label = pieceImageLabels[i][j];
            ChessPiece piece = board.getPiece(position);
            if (piece != null) {
                // 根据棋子种类选择对应的棋子图片
                label.setIcon(getChessPieceImageIcon(piece));
            }
            else {
                label.setIcon(null);
            }
        }
    }
}
```

最后，添加横纵坐标轴标签，注意对齐棋盘方格即可，代码方式不再赘述。最终实现效果如下图所示：

![image-20200404144011224](https://i.loli.net/2021/06/15/vnZOXrQ6eWpSutB.png)

### 操作面板

操作面板由若干文字信息展示标签和若干功能按钮构成，现列简表如下：

| 类型                       | 变量名                                                       | 功能                                               |
| -------------------------- | ------------------------------------------------------------ | -------------------------------------------------- |
| `JLabel`的自定义拓展类     | `firstPlayerPieceNumLabel`                                   | 展示先手玩家姓名及棋子数量                         |
| `JLabel`的自定义拓展类     | `secondPlayerPieceNumLabel`                                  | 展示后手玩家姓名及棋子数量                         |
| `JLabel`的自定义拓展类     | `turnNumLabel`                                               | 展示游戏轮数                                       |
| `JLabel`                   | `currentPlayerLabel`                                         | 展示文字「Current Player」                         |
| `JLabel`                   | `currentPlayerNameLabel`                                     | 展示玩家姓名                                       |
| `JTextField`的自定义拓展类 | `originYInput`<br />`originXInput`<br />`destinationYInput`<br />`destinationXInput` | 坐标输入框                                         |
| `JLabel`                   | `informationLabel`                                           | 展示反馈信息                                       |
| `JButton`                  | `endButton`                                                  | 结束游戏                                           |
| `JButton`                  | `skipButton`                                                 | 跳过当前回合                                       |
| `JButton`                  | `ActionButton`                                               | 根据输入的坐标尝试执行动作<br />自动识别移子和吃子 |

大部分组件的添加都是比较简单的。对于信息的更新，也只需要在相应按钮的事件监听器中写入更新语句即可。下面重点介绍用于实现坐标输入框的`JTextField`自定义拓展类`JTextFieldLimited`和各个按钮的动作监听器的实现。

#### 带有限制的文本输入域`JTextFieldLimited`

根据国际象棋的相关标准，棋盘坐标的表示要遵循一定的格式。横坐标用1至8之间的整数表示，从白方底线向黑方底线递增。纵坐标用a至h之间的拉丁字母表示（大小写不限），从白方左手边向白方右手边递增。书写时，纵坐标在前，横坐标在后。因此，从代码内部的(0, 0)位置移动到(7, 7)位置应该写成`(a, 1) => (h, 8)`，也就是从棋盘左下角移动到右上角（白方视角）。

基于此，坐标输入框应该对输入的字符有所限制。我们可以通过拓展swing的`JTextField`类做到这一点。具体实现如下：

```java
// 限制输入字符的JTextField
public class JTextFieldLimited extends JTextField {

    // 限制长度
    private final int limit;
    // 限制输入字符集
    private Set<Character> charSet;

    // 构造方法，传入限制字符集（字符串形式）和长度限制
    public JTextFieldLimited(String chars, int limit) {
        super();
        this.limit = limit;
        charSet = new HashSet<>();
        for (int i = 0; i < chars.length(); i++) {
            charSet.add(chars.charAt(i));
        }
    }

    // 重写createDefaultModel方法，返回自定义的LimitDocument
    @Override
    protected Document createDefaultModel() {
        return new LimitDocument();
    }

    // 继承PlainDocument，自定义LimitDocument
    private class LimitDocument extends PlainDocument {

        // 重写insertString方法
        // 尝试在现有字符串的offset位置插入新字符串str
        @Override
        public void insertString(int offset, String str, AttributeSet attr) throws BadLocationException {
            // 如果新字符串为空，或者插入后的总长度超过限制，则不予插入
            if (str == null || (getLength() + str.length()) > limit) {
                return;
            }

            // 如果出现合法字符集之外的字符，则不予插入
            for (int i = 0; i < str.length(); i++) {
                if (!charSet.contains(str.charAt(i))) {
                    return;
                }
            }

            // 合法性检查通过，调用父类的insertString方法
            super.insertString(offset, str, attr);
        }

    }

}

```

这样，四个坐标输入框就可以利用`JTextFieldLimited`这个类，轻松地进行输入限制了。当然，也可以让用户自由输入字符，而只在按下按钮时进行合法性检查，但其交互比直接限制输入要啰嗦一些，所以没有采用。

#### 跳过按钮`skipButton`

按下跳过按钮后，客户端应该做这么几件事：

- 反馈信息显示提示「Player ? skipped turn ?」

- 调用`Game`ADT的`turn`接口，传入一个新的`SkipAction`实例
- 更新游戏轮数和当前玩家姓名等信息
- 清除坐标输入框，方便玩家下一轮输入

代码实现如下：

```java
skipButton.addActionListener(e -> {
    // 更新反馈信息标签
    informationLabel.setText(String.format("Player %s Skipped turn %d.",
            game.getCurrentPlayerInfo(),
            game.getFollowingTurnNum()
    ));
    
    // 调用turn方法，跳过本轮
    game.turn(new SkipAction());
    
    // 更新游戏轮数和当前玩家
    turnNumLabel.updateTurnNumText();
    currentPlayerNameLabel.setText(game.getCurrentPlayerInfo());
    
    // 清除所有坐标输入框中的内容
    clearAllInputTextField();
});
```

#### 动作按钮`ActionButton`

按下跳过按钮后，客户端应该做这么几件事：

- 检查4个输入框是否都是非空的。若有空的，直接返回
- 检查终点是空的还是有棋子
  - 若为空，调用`turn`方法，传入`MoveAction`的实例
  - 若有棋子，调用`turn`方法，传入`CaptureAction`实例
- 检查上一步调用的`turn`方法的返回值（表示动作是否执行成功）
  - 若不成功，显示不合法动作的反馈信息，直接返回
  - 若成功，更新棋盘、双方棋子数量、当前玩家等信息，并清除反馈信息

代码实现如下：

```java
actionButton.addActionListener(e -> {
    String originY = originYInput.getText().toLowerCase();
    String originX = originXInput.getText();
    String destinationY = destinationYInput.getText().toLowerCase();
    String destinationX = destinationXInput.getText();

    // 检查4个输入框是否都非空
    if (originY.length() == 0 || originX.length() == 0 ||
            destinationY.length() == 0 || destinationX.length() == 0) {
        return;
    }

    // 将输入的字符串转化为Position
    Position originPosition = ChessUtils.formal2Inside(new ChessPositionFormat(
            originY.charAt(0),
            Integer.parseInt(originX)
    ));
    Position destPosition = ChessUtils.formal2Inside(new ChessPositionFormat(
            destinationY.charAt(0),
            Integer.parseInt(destinationX)
    ));

    // 动作执行成功标记
    boolean success;
	// 根据终点处是否有棋子新建相应的ChessAction实例
    ChessAction action =
        game.getPiece(destPosition) != null ?
        new CaptureAction(originPosition, destPosition) :
    	new MoveAction(originPosition, destPosition);
    
    // 执行turn方法
    // 若不成功，显示无效信息并直接返回
    if (!game.turn(action)) {
        informationLabel.setText("Invalid action!");
        return;
    }
	
    // 执行成功，更新棋盘
    JLabel originLabel = pieceImageLabels[originPosition.getX()][originPosition.getY()];
    // 终点处置为原先起点的棋子
    pieceImageLabels[destPosition.getX()][destPosition.getY()].setIcon(originLabel.getIcon());
    // 起点处置为空
    pieceImageLabels[originPosition.getX()][originPosition.getY()].setIcon(null);
    
    // 更新信息
    firstPlayerPieceNumLabel.updatePlayerPieceNum();
    secondPlayerPieceNumLabel.updatePlayerPieceNum();
    currentPlayerNameLabel.setText(game.getCurrentPlayerInfo());
    turnNumLabel.updateTurnNumText();
    
    // 清空输入框和反馈信息标签
    clearAllInputTextField();
    informationLabel.setText("");
});
```

#### 终止游戏按钮`EndButton`

`EndButton`在国际象棋和围棋中对应的点击事件应该是相同的。所以，我把这个事件单独拿出来放在`GUIUtils`工具类中，以便两个GUI类复用。

按下终止游戏按钮后，客户端应该做这么几件事：

- 显示「是否真的要终止」的确认对话框
- 若用户确认关闭，显示游戏历史记录窗口，并结束游戏

```java
endButton.addActionListener(e -> exitClickEvent(frame, game));

// GUIUtils.java
// 终止游戏自定义事件exitClickEvent方法
protected static void exitClickEvent(JFrame frame, Game game) {
    // 显示确认对话框
    int result = JOptionPane.showConfirmDialog(
        frame,
        "Terminate this Game?\nClick 'YES' to view the history records and leave",
        "Exit Confirm",
        // 选项为YES or NO二选一
        JOptionPane.YES_NO_OPTION
    );
    
    // 如果确认终止
    if (result == JOptionPane.YES_OPTION) {
        // 显示游戏历史记录
        JDialogEndRecords endRecordsDialog = new JDialogEndRecords(frame, getEndRecords(game));
        endRecordsDialog.showMe();
        // 关闭游戏窗口
        frame.dispose();
    }
}
```

此外，为了防止用户点击窗口右上角的关闭按钮或用Alt + F4组合键等方式直接关掉游戏，我们可以对窗口本身的关闭也做同样的设置，代码如下：

```java
protected static void setFrame(JFrame frame, Game game, int frameWidth, int frameHeight) {
    // 设置窗口大小
    frame.setSize(frameWidth, frameHeight);
    // 默认关闭行为设为什么都不做
    frame.setDefaultCloseOperation(DO_NOTHING_ON_CLOSE);
    // 将窗口大小设为不可变
    frame.setResizable(false);
    // 窗口显示位置设在屏幕正中央
    frame.setLocationRelativeTo(null);
    // 设置窗口动作监听器
    frame.addWindowListener(new WindowAdapter() {
        // 窗口关闭动作
        public void windowClosing(WindowEvent e) {
            super.windowClosing(e);
            // 调用自定义事件exitClickEvent
            exitClickEvent(frame, game);
        }
    });
    frame.setVisible(true);
}
```

至此，游戏界面基本完成。效果如下图所示：

![TIM截图20200327231014](https://i.loli.net/2021/06/15/p2nhkAaNZsIbw87.png)

### 游戏历史记录界面

游戏历史记录界面可以用一个自定义对话框显示。对话框中，用`JTextArea`（文本区域）组件显示游戏历史记录，下方设置一个「OK」按钮，点击即可结束游戏。

自定义对话框`JDialogEndRecords`可以这样实现：

```java
public class JDialogEndRecords extends JDialog {

    // 文本区域中要显示的信息
    private final String message;

    // 构造器，传入关联的父窗口和文本区域中要显示的信息
    public JDialogEndRecords(JFrame owner, String message) {
        super(owner, "Game Records", true);
        this.message = message;
    }

    // 显示本对话框
    public void showMe() {
        int dialogWidth = GUIUtils.frameWidth, dialogHeight = 600;

        // 设置对话框大小和位置
        setSize(GUIUtils.frameWidth, 600);
        setResizable(false);
        setLocationRelativeTo(super.getOwner());

        // 将对话框内容设置为一个新的自定义JPanel面板
        JPanel panel = new JPanel(null);

        // 添加带滚动条的文本区域messagePane，显示message
        JTextArea messageArea = new JTextArea(message);
        JScrollPane messagePane = new JScrollPane(messageArea);
        messageArea.setFont(GUIUtils.endRecordsFont);
        messageArea.setEditable(false);
        messagePane.setBounds(50, 30, 900, 470);
        System.out.println(message);

        // 添加OK按钮
        JButton okButton = new JButton("OK");
        okButton.setBounds(dialogWidth / 2 - 50, dialogHeight - 85, 100, 30);
        okButton.addActionListener(e -> dispose());

        // 向panel中添加messagePane和okButton
        panel.add(messagePane);
        panel.add(okButton);
        
        setContentPane(panel);
        setVisible(true);
    }

}
```

最终效果如下图所示：

![TIM截图20200327231027](https://i.loli.net/2021/06/15/lPoiGfW3Cwnd2qH.png)

## 围棋游戏界面实现

为其游戏界面与国际象棋大同小异。这里仅指出几个不同点：

1. 围棋棋盘的行列数较多，找到的图片素材大多也是带边框的。因此需要手动二分出棋盘位置的像素坐标，以便放置棋子图片标签。

2. 围棋的操作面板中只有一对坐标输入窗口，其中只能输入1~19之间的整数。这个限制相对难写一些，下面给出我`insertString`方法的实现：

   ```java
   // 尝试在现有字符串的offset位置插入新字符串str
   @Override
   public void insertString(int offset, String str, AttributeSet attr) throws BadLocationException {
       // 如果新字符串为空，或者插入后的总长度超过限制（2），则不予插入
       if (str == null || (getLength() + str.length()) > limit) {
           return;
       }
       // 如果出现合法字符集之外符（即非数字的字符），则不予插入
       for (int i = 0; i < str.length(); i++) {
           if (!charSet.contains(str.charAt(i))) {
               return;
           }
       }
       // 如果当前为空且新字符串是0开头的，则不予插入
       if (getLength() == 0 && str.charAt(0) == '0') {
           return;
       }
       // 如果当前长度为1
       if (getLength() == 1) {
           // 如果在这个字符之前插入，且新字符串不是“1”，则不予插入
           if (offset == 0) {
               if (str.charAt(0) != '1') {
                   return;
               }
           }
           // 如果在这个字符之后插入，如果当前字符串不为“1”，则不予插入
           else {
               if (getText(0, 1).charAt(0) != '1') {
                   return;
               }
           }
       }
       // 合法性检查通过，调用父类的insertString方法
       super.insertString(offset, str, attr);
   }
   ```

3. 为了防止误操作，我将设置了下子和提子两个按钮，分别对应调用下子和提子两个动作。

## 入口实现

入口界面需要让玩家选择国际象棋和围棋之一进行游戏。选择后，要让玩家输入姓名，输入两个合法的姓名才能进入游戏。

### 界面设计

需要注意的地方是交互逻辑，我设计了如下流程：

- 进入游戏开始界面，用户点选需要进行的游戏

- 无论点选哪个游戏，显示玩家姓名输入对话框
  - 若玩家点击对话框的OK按钮，检查输入是否合法
    - 若合法，进入游戏
    - 若不合法，什么都不做
  - 若玩家点击对话框的关闭按钮，回到开始界面

### 玩家姓名输入对话框

可以发现，开始界面的窗口是组织上述逻辑的核心。在开始界面窗口中，我们需要获知姓名输入对话框「是怎么关闭的」——是点击OK按钮，还是关闭按钮。如果是前者，我们需要新建对应的游戏`Game`对象，而如果是后者，我们什么都不需要做。

为此，我们需要在自定义的对话框中做一些手脚，让它用两种方式关闭有两种不同的返回值，代码如下：

```java
// 自定义对话框JDialogInputPlayerName，用于玩家姓名的输入
public class JDialogInputPlayerName extends JDialog {

    // 两个姓名输入域，用JTextFieldLimited自定义类限制其长度和合法字符
    private JTextFieldLimited nameTextField1 = new JTextFieldLimited(letters, 7);
    private JTextFieldLimited nameTextField2 = new JTextFieldLimited(letters, 7);
    
    // 标记是否通过OK按钮关闭，默认为false
    private boolean closeThroughOK = false;

    // 新建对话框实例，传入父窗口
    public JDialogInputPlayerName(JFrame owner) {
        super(owner, "Player Names", true);
        this.nameTextField1 = nameTextField1;
        this.nameTextField2 = nameTextField2;
    }

    public boolean showMe() {
        
        ... set format ...

        JPanel panel = new JPanel(null);

        // 上方输入框为先手玩家，下方为后手玩家。
        // 姓名字符串要求只能包含小写字符串，且长度不能超过7；此外两位玩家姓名不能相同。
        ... add components ...

        // 设置OK按钮行为
        okButton.addActionListener(e -> {
            String name1 = nameTextField1.getText();
            String name2 = nameTextField2.getText();
            // 若输入框为空，直接返回
            if (name1.length() == 0 || name2.length() == 0) {
                return;
            }
            // 如果两个姓名字符串相等，直接返回
            if (name1.equals(name2)) {
                return;
            }
            // 输入合法并按下OK按钮，「通过OK按钮关闭」标记置为true
            closeThroughOK = true;
            dispose();
        });

        setContentPane(panel);
        setVisible(true);

        // 返回标记的值
        return closeThroughOK;
    }
    
    // 获取姓名输入框1中的字符串
    public String getPlayerName1() {
        return nameTextField1.getText();
    }

    // 获取姓名输入框2中的字符串
    public String getPlayerName2() {
        return nameTextField2.getText();
    }

}
```

这样，对话框的`showMe`方法返回值就能体现它的关闭方式了。

### 开始界面

做好了上述准备工作，开始界面的实现就很简单了。布局方面，放上大大的标题和两个大大的按钮即可。按钮的动作监听器可以这么写（以Chess按钮为例）：

```java
// Chess按钮的动作监听器
chessButton.addActionListener(e -> {
    
    // 新建自定义对话框类JDialogInputPlayerName实例
    JDialogInputPlayerName nameDialog = new JDialogInputPlayerName(frame);
    
    // 按下按钮，显示对话框
    // 如果通过按下OK按钮返回
    if (nameDialog.showMe()) {
        // 新建国际象棋游戏实例
        ChessGame game = new ChessGame(nameDialog.getPlayerName1(), nameDialog.getPlayerName2());
        // 新建国际象棋游戏GUI实例
        ChessGameGUI chessGameGUI = new ChessGameGUI(game);
        // 运行GUI
        chessGameGUI.run("Chess Game");
        // 运行完毕（运行到这里，游戏历史记录已经展示过），关闭窗口
        frame.dispose();
    }
});
```

整体效果如下图所示：

![image-20200404215527740](https://i.loli.net/2021/06/15/EeIYWR1oSlfcBmZ.png)

至此，整个棋类游戏GUI设计基本完成。

## 参考资料

- [CSDN - Java Swing层级面板](https://blog.csdn.net/xietansheng/article/details/74366560)

- [CSDN - Java Swing对话框](https://blog.csdn.net/xietansheng/article/details/75948933)

- [国际象棋在线](https://chess.com)

- [CSDN - JTextField限制输入](https://blog.csdn.net/mmmmmk_/article/details/78608917)

