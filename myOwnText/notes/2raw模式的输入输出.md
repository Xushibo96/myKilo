# 按 ctrl-Q 退出

```C
/*** includes ***/
#include <ctype.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
/*** defines ***/
#define CTRL_KEY(k) ((k) & 0x1f)
/*** data ***/
/*** terminal ***/
/*** init ***/
int main() {
  enableRawMode();
  while (1) {
    char c = '\0';
    if (read(STDIN_FILENO, &c, 1) == -1 && errno != EAGAIN) die("read");
    if (iscntrl(c)) {
      printf("%d\r\n", c);
    } else {
      printf("%d ('%c')\r\n", c, c);
    }
    if (c == CTRL_KEY('q')) break;
  }
  return 0;
}
```

上一章我们看到，Ctrl 键和字母键组合起来似乎映射到字节 1-26。我们可以使用它来检测 Ctrl 键组合，并将它们映射到编辑器中的不同操作。我们首先将 `Ctrl-Q` 映射到 `quit` 操作。`CTRL_KEY`按位和一个二进制字符，其值为 `0001111`。（在 C 中，通常使用十六进制指定位掩码，因为 C 没有二进制文字，一旦习惯了十六进制，它会更简洁易读。）换句话说，它将字符的上 3 位设置为 0。这反映了 Ctrl 键在终端中的作用：它将从您与 Ctrl 组合按下的任何键中删除位 5 和 6，并发送该位。（按照惯例，位编号从 0 开始。）ASCII 字符集似乎是故意这样设计的。清除掉第五第六位就可以实现 ASCII 大小写的转换还是挺神奇的

# 重构键盘输入

让我们创建一个用于低级按键读取的函数，以及另一个用于将按键映射到编辑器操作的函数。我们也将停止打印按键输入。

```c
char editorReadKey() {
  int nread;
  char c;
  while ((nread = read(STDIN_FILENO, &c, 1)) != 1) {
    if (nread == -1 && errno != EAGAIN) die("read");
  }
  return c;
}
/*** input ***/
void editorProcessKeypress() {
  char c = editorReadKey();
  switch (c) {
    case CTRL_KEY('q'):
      exit(0);
      break;
  }
}
```

`editorReadKey（）`的工作是等待一次按键，然后返回。稍后，我们将扩展此函数以处理转义序列，这涉及读取代表单个按键的多个字节，箭头键就是这样。

`editorProcessKeypress（）`等待按键，然后处理它。稍后，它会将各种 Ctrl 键组合和其他特殊键映射到不同的编辑器功能，并将任何字母数字和其他可打印键的字符插入正在编辑的文本中。

请注意，`editorReadKey（）`属于`/**terminal**/`部分，因为它处理低级终端输入，而 `editorProcessKeypress（）`属于新的`/**input**/`部分，因为它处理将键映射到更高级别的编辑器函数。

现在 `main` 函数编的整洁了，继续保持 `main` 的整洁。

# 清屏

```c
/*** includes ***/
/*** defines ***/
/*** data ***/
/*** terminal ***/
void die(const char *s) { … }
void disableRawMode() { … }
void enableRawMode() { … }
char editorReadKey() { … }
/*** output ***/
void editorRefreshScreen() {
  write(STDOUT_FILENO, "\x1b[2J", 4);
}
/*** input ***/
/*** init ***/
int main() {
  enableRawMode();
  while (1) {
    editorRefreshScreen();
    editorProcessKeypress();
  }
  return 0;
}
```

我们将在每次按键后将编辑器的用户界面呈现到屏幕上。让我们从清理屏幕开始。

write（）调用中的 4 表示我们正在向终端写入 4 个字节。第一个字节是`\x1b`，它是转义字符，或十进制的 `27`。（请记住`\x1b`，我们将经常使用它。）其他三个字节是`[2J`。

我们正在给终端写一个逃逸序列`escape sequence `。转义序列总是以转义字符（27）和`[`字符开头。转义序列指示终端执行各种文本格式化任务，例如为文本着色、移动光标和清除屏幕的部分

我们正在使用 J 命令（在显示中擦除）清除屏幕。转义序列命令接受位于命令之前的参数。在本例中，参数为 2，表示清除整个屏幕`<esc>[1J `将清除屏幕直到光标所在的位置，`<esc>[0J` 将清除屏幕从光标到屏幕末端的位置。此外，0 是 J 的默认参数，因此仅`<esc>[J `本身也将清除屏幕从光标到屏幕末端的位置。

对于我们的文本编辑器，我们将主要使用 VT100 转义序列，现代终端模拟器非常广泛地支持它。有关每个转义序列的完整文档，请参阅 VT100 用户指南。

如果我们想支持最大数量的终端，我们可以使用 ncurses 库，它使用 terminfo 数据库来计算终端的功能，以及该特定终端使用的转义序列。

# 重新定位光标

您可能会注意到，`<esc>[2J`命令将光标留在屏幕底部。让我们将其重新定位在左上角，以便准备从上到下绘制编辑器界面。

```c
void editorRefreshScreen() {
  write(STDOUT_FILENO, "\x1b[2J", 4);
  write(STDOUT_FILENO, "\x1b[H", 3);
}

```

这个转义序列只有 3 字节长，并使用 H 命令（光标位置）来定位光标。H 命令实际上有两个参数：行号和列号，光标的位置。因此，如果您有一个 80×24 大小的终端，并且希望光标位于屏幕的中心，可以使用命令`<esc>[12；40H`（多个参数由一个；字符分隔）H 的默认参数恰好都是 1，因此我们可以忽略这两个参数，它会将光标定位在第一行和第一列，就像我们发送了`<esc>[1`；1H 命令一样（行和列从 1 开始编号，而不是从 0 开始编号）

# 退出时清除屏幕

让我们在程序退出时清除屏幕并重新定位光标。如果在渲染屏幕中间出现错误，我们不希望在屏幕上留下一堆垃圾，并且我们不希望在光标碰巧在那一点上打印错误。我们有两个退出点需要清除屏幕：die（），当用户按 Ctrl-Q 退出时。

我们可以在程序退出时使用 atexit（）清除屏幕，但是在打印完之后，由 die（）打印的错误消息会立即被删除。因此是在两个退出位置上都进行了清屏，这样 die 时打印的错误信息不会被清除。

# Tildes

该开始画画了。让我们像 vim 一样，在屏幕左侧绘制一列波浪线（~）。在文本编辑器中，我们将在所编辑文件末尾之后的任何行的开头绘制一个平铺。

```c
/*** includes ***/
/*** defines ***/
/*** data ***/
/*** terminal ***/
/*** output ***/
void editorDrawRows() {
  int y;
  for (y = 0; y < 24; y++) {
    write(STDOUT_FILENO, "~\r\n", 3);
  }
}
void editorRefreshScreen() {
  write(STDOUT_FILENO, "\x1b[2J", 4);
  write(STDOUT_FILENO, "\x1b[H", 3);
  editorDrawRows();
  write(STDOUT_FILENO, "\x1b[H", 3);
}
/*** input ***/
/*** init ***/
```

editorDrawRows（）将处理绘制正在编辑的文本缓冲区的每一行。目前，它在每一行中绘制一个`~`，这意味着该行不是文件的一部分，不能包含任何文本。
我们还不知道终端的大小，所以我们不知道要画多少行。现在我们只画 24 行。
完成绘图后，我们再进行一次`<esc>[H` escape sequence，将光标重新定位到左上角。

# 全局状态

我们的下一个目标是获得终端的大小，以便知道在 editorDrawRows（）中要绘制多少行。但首先，让我们设置一个包含编辑器状态的全局结构，我们将使用它存储终端的宽度和高度。现在，让我们把我们的原始 termios global 放到结构中。

```c
/*** includes ***/
/*** defines ***/
/*** data ***/
struct editorConfig {
  struct termios orig_termios;
};
struct editorConfig E;
/*** terminal ***/
void die(const char *s) { … }
void disableRawMode() {
  if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &E.orig_termios) == -1)
    die("tcsetattr");
}
void enableRawMode() {
  if (tcgetattr(STDIN_FILENO, &E.orig_termios) == -1) die("tcgetattr");
  atexit(disableRawMode);
  struct termios raw = E.orig_termios;
  raw.c_iflag &= ~(BRKINT | ICRNL | INPCK | ISTRIP | IXON);
  raw.c_oflag &= ~(OPOST);
  raw.c_cflag |= (CS8);
  raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);
  raw.c_cc[VMIN] = 0;
  raw.c_cc[VTIME] = 1;
  if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw) == -1) die("tcsetattr");
}
```

包含编辑器状态的全局变量名为 E。我们必须用 E.orig_termios 替换所有出现的 orig_termios。

# 窗口大小，简单的方法

在大多数系统上，只需通过 TIOCGWINSZ 请求调用 ioctl（）即可获得终端的大小。（据我所知，它代表终端 IOCtl（它本身代表输入/输出控制）获取窗口大小。）

```c
int getWindowSize(int *rows, int *cols) {
  struct winsize ws;
  if (ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) == -1 || ws.ws_col == 0) {
    return -1;
  } else {
    *cols = ws.ws_col;
    *rows = ws.ws_row;
    return 0;
  }
}
```

成功后，ioctl（）将把终端所在的列宽和行高的数量放入给定的 winsize 结构中。失败时，ioctl（）返回-1。我们还检查以确保它返回的值不是 0，因为这显然是一个可能的错误结果。如果 ioctl（）以任何一种方式失败，我们将通过返回-1 来报告失败。如果成功，我们通过设置传递给函数的 int 引用来传递回值。（这是让函数在 C 中返回多个值的常见方法。它还允许您使用返回值来指示成功或失败。）

ioctl（）不能保证在所有系统上都能请求窗口大小，因此我们将提供一种获取窗口大小的后备方法。
策略是将光标定位在屏幕的右下角，然后使用转义序列来查询光标的位置。这告诉我们屏幕上必须有多少行和列。
让我们首先将光标移动到右下角。

```c
int getWindowSize(int *rows, int *cols) {
  struct winsize ws;
  if (1 || ioctl(STDOUT_FILENO, TIOCGWINSZ, &ws) == -1 || ws.ws_col == 0) {
    if (write(STDOUT_FILENO, "\x1b[999C\x1b[999B", 12) != 12) return -1;
    editorReadKey();
    return -1;
  } else {
    *cols = ws.ws_col;
    *rows = ws.ws_row;
    return 0;
  }
}
```

正如您可能从代码中了解到的，没有简单的“将光标移动到右下角”命令。
我们正在一个接一个地发送两个逃逸序列。C 命令（光标向前）将光标向右移动，B 命令（光标向下）将光标向下移动。争论的焦点是它应该向右或向下移动多少。我们使用了一个非常大的值 999，它应该确保光标到达屏幕的右边缘和下边缘。
C 和 B 命令专门用于阻止光标越过屏幕边缘。我们不使用<esc>[999；999H 命令的原因是，文档没有指定当您尝试将光标移出屏幕时会发生什么。
请注意，我们暂时在 if 条件的前面粘贴一个 1 | |，以便测试我们正在开发的这个后备分支。
因为此时我们总是从 GetWindowsSize（）返回-1（表示发生了错误），所以我们调用 editorReadKey（），以便在程序调用 die（）并清除屏幕之前观察转义序列的结果。当你运行程序时，你应该看到光标位于屏幕的右下角，然后当你按下一个键时，你会看到 die（）在清除屏幕后打印的错误消息。
接下来我们需要得到光标的位置。n 命令（设备状态报告）可用于查询终端的状态信息。我们想给它一个 6 的参数来询问光标的位置。然后我们可以从标准输入中读取回复。让我们打印出标准输入中的每个字符，看看回复是什么样子。

回答是一个转义序列！它是一个转义字符（27），后跟一个[字符，然后是实际响应：24；80R 或类似的字符（这个转义序列记录为光标位置报告）

和前面一样，我们插入了对 editorReadKey（）的临时调用，以便在退出时清除屏幕之前观察调试输出。

我们将不得不解析这个响应。但首先，让我们把它读入缓冲区。我们将继续阅读字符，直到找到 R 字符。

# 最后一行

也许你注意到屏幕的最后一行似乎没有`~`。这是因为我们的代码中有一个小错误。当我们打印最后一行时，我们会像在任何其他行上一样打印一个“\r\n”，但这会导致终端滚动，以便为新的空白行腾出空间。打印“\r\n”时，让我们将最后一行作为例外。

```c
void editorDrawRows() {
  int y;
  for (y = 0; y < E.screenrows; y++) {
    write(STDOUT_FILENO, "~", 1);
    if (y < E.screenrows - 1) {
      write(STDOUT_FILENO, "\r\n", 2);
    }
  }
}
```

用这样的方式，除了最后一行不打\r\n 以外其他行都打\r\n 就可以保留最后一行的`~`

# 附加缓冲区

每次刷新屏幕时都要做一大堆小的 write（）不是个好主意。最好做一个大的 write（），以确保整个屏幕立即更新。否则，write（）之间可能会出现不可预测的小停顿，这会导致恼人的闪烁效果。
我们希望用将字符串附加到缓冲区的代码替换所有 write（）调用，然后在最后写出（）这个缓冲区。不幸的是，C 没有动态字符串，所以我们将创建自己的动态字符串类型，它支持一个操作：追加。
让我们首先创建一个新的/**_附加缓冲区_**/节，并在其下定义 abuf 结构。

追加缓冲区由指向内存中缓冲区的指针和长度组成。我们定义了一个 ABUF_INIT 常量，它代表一个空缓冲区。它充当 abuf 类型的构造函数。
接下来，让我们定义 `abAppend（）`操作以及` abFree（）`析构函数。

```c
#include <string.h>
void abAppend(struct abuf *ab, const char *s, int len) {
  // 现在想要把字符串s追加到我们设置的缓冲区结构体里
  //s 是进来的字符串，ab是缓冲区指针。
  char *new = realloc(ab->b, ab->len + len);
  // 先用realloc保证得到一个够用的堆空间能够放下缓冲区和要加进来的字符串
  if (new == NULL) return;
  memcpy(&new[ab->len], s, len);
  // 然后用mencpy把字符串追加到缓冲区后面部分去
  // &new[ab->len]这个是表示ab中第一个空的位置，因为ab->len是ab里面的长度
  ab->b = new;
  ab->len += len;
  // 更新缓冲区里面的堆空间位置和缓冲区现在的长度，加上刚加进来的s的长度
}
void abFree(struct abuf *ab) {
  free(ab->b);
}

```

要将字符串 `s` 附加到 `abuf`，我们要做的第一件事是确保分配足够的内存来保存新字符串。我们要求 `realloc（`）给我们一个内存块，它是当前字符串的大小加上我们要追加的字符串的大小。`realloc（）`将扩展我们已经分配的内存块的大小，或者它将负责释放当前内存块，并在其他地方分配一个足够容纳新字符串的新内存块。
然后我们使用 `memcpy（）`在缓冲区中当前数据结束后复制字符串 s，并将指针和 abuf 的长度更新为新值。
`abFree（）`是一个析构函数，用于释放 `abuf` 使用的动态内存。
好的，我们的 `abuf` 型已经准备好投入使用了。

```c
/*** includes ***/
/*** defines ***/
/*** data ***/
/*** terminal ***/
/*** append buffer ***/
/*** output ***/
void editorDrawRows(struct abuf *ab) {
  int y;
  for (y = 0; y < E.screenrows; y++) {
    abAppend(ab, "~", 1);
    if (y < E.screenrows - 1) {
      abAppend(ab, "\r\n", 2);
    }
  }
}
void editorRefreshScreen() {
  struct abuf ab = ABUF_INIT;
  abAppend(&ab, "\x1b[2J", 4);
  abAppend(&ab, "\x1b[H", 3);
  editorDrawRows(&ab);
  abAppend(&ab, "\x1b[H", 3);
  write(STDOUT_FILENO, ab.b, ab.len);
  abFree(&ab);
}
```

在 editorRefreshScreen（）中，我们首先初始化一个名为 ab 的新 abuf，方法是将 abuf_INIT 分配给它。然后，我们替换每 write（STDOUT_FILENO，…）用 abAppend（&ab，…）。我们还将 ab 传递给 editorDrawRows（），因此它也可以使用 abAppend（）。最后，我们将缓冲区的内容写入标准输出，并释放 abuf 使用的内存。

# 重新绘制时隐藏光标

还有另一个可能的恼人的闪烁效应的来源，我们现在要处理。当终端在屏幕上绘制时，光标可能会在屏幕中间显示一秒钟。为了确保不会发生这种情况，让我们在刷新屏幕之前隐藏光标，并在刷新完成后立即再次显示。

我们使用转义序列告诉终端隐藏和显示光标。h 和 l 命令（设置模式、重置模式）用于打开和关闭各种终端功能或“模式”。刚刚链接到的 VT100 用户指南没有记录参数？我们在上面使用的 25。光标隐藏/显示功能似乎出现在后来的 VT 机型中。因此，一些终端可能不支持隐藏/显示光标，但如果不支持，那么它们只会忽略这些转义序列，在这种情况下这并不是什么大问题。

```c
void editorRefreshScreen() {
  struct abuf ab = ABUF_INIT;
  abAppend(&ab, "\x1b[?25l", 6);
  abAppend(&ab, "\x1b[2J", 4);
  abAppend(&ab, "\x1b[H", 3);
  editorDrawRows(&ab);
  abAppend(&ab, "\x1b[H", 3);
  abAppend(&ab, "\x1b[?25h", 6);
  write(STDOUT_FILENO, ab.b, ab.len);
  abFree(&ab);
}
```

# 一次清除一行

与其在每次刷新之前清除整个屏幕，不如在重新绘制时清除每一行。让我们移除`<esc>[2J`（清除整个屏幕）转义序列，而是在我们绘制的每一行的末尾放置一个`<esc>[K`序列。

K 命令（行内擦除）擦除当前行的一部分。它的参数类似于 J 命令的参数：2 删除整行，1 删除光标左侧的部分，0 删除光标右侧的部分。0 是默认参数，这就是我们想要的，所以我们省略了该参数，只使用 K。

# 欢迎语

也许是时候展示欢迎信息了。让我们在屏幕的三分之一处显示编辑器的名称和版本号。

```c
#define KILO_VERSION "0.0.1"

void editorDrawRows(struct abuf *ab) {
  int y;
  for (y = 0; y < E.screenrows; y++) {
    if (y == E.screenrows / 3) {
      char welcome[80];
      int welcomelen = snprintf(welcome, sizeof(welcome),
        "Kilo editor -- version %s", KILO_VERSION);
      if (welcomelen > E.screencols) welcomelen = E.screencols;
      abAppend(ab, welcome, welcomelen);
    } else {
      abAppend(ab, "~", 1);
    }
    abAppend(ab, "\x1b[K", 3);
    if (y < E.screenrows - 1) {
      abAppend(ab, "\r\n", 2);
    }
  }
}

```

我们使用欢迎缓冲区和 snprintf（）将我们的 KILO\_版本字符串插入欢迎消息中。我们还将截断字符串的长度，以防终端太小，无法容纳我们的欢迎信息。

现在让它居中

```c
void editorDrawRows(struct abuf *ab) {
  int y;
  for (y = 0; y < E.screenrows; y++) {
    if (y == E.screenrows / 3) {
      char welcome[80];
      int welcomelen = snprintf(welcome, sizeof(welcome),
        "Kilo editor -- version %s", KILO_VERSION);
      if (welcomelen > E.screencols) welcomelen = E.screencols;
//
      int padding = (E.screencols - welcomelen) / 2;
      if (padding) {
        abAppend(ab, "~", 1);
        padding--;
      }
      while (padding--) abAppend(ab, " ", 1);
      //
      abAppend(ab, welcome, welcomelen);
    } else {
      abAppend(ab, "~", 1);
    }
    abAppend(ab, "\x1b[K", 3);
    if (y < E.screenrows - 1) {
      abAppend(ab, "\r\n", 2);
    }
  }
}
```

要使字符串居中，可以将屏幕宽度除以 2，然后从中减去字符串长度的一半。换句话说：`E.screencols/2-welcomelen/2`，它简化为（`E.screencols-welcomelen）/2`。这将告诉您应该从屏幕左边缘开始打印字符串的距离。所以我们用空格字符填充这个空间，除了第一个字符，它应该是一个`~`。

# 移动光标

现在让我们关注输入。我们希望用户能够移动光标。第一步是在全局编辑器状态下跟踪光标的 x 和 y 位置。

```c
/*** includes ***/
/*** defines ***/
/*** data ***/
struct editorConfig {
  int cx, cy;
  int screenrows;
  int screencols;
  struct termios orig_termios;
};
struct editorConfig E;
/*** terminal ***/
/*** append buffer ***/
/*** output ***/
/*** input ***/
/*** init ***/
void initEditor() {
  E.cx = 0;
  E.cy = 0;
  if (getWindowSize(&E.screenrows, &E.screencols) == -1) die("getWindowSize");
}
```

`E.cx` 是光标（列）的水平坐标，`E.cy` 是垂直坐标（行）。我们将它们都初始化为 0，因为我们希望光标从屏幕左上角开始。（由于 C 语言使用从 0 开始的索引，我们将尽可能使用 0 索引值。）
现在，让我们向 `editorRefreshScreen（）`添加代码，将光标移动到存储在`E.cx` 和 `E.cy `中的位置。

```c
void editorRefreshScreen() {
  struct abuf ab = ABUF_INIT;
  abAppend(&ab, "\x1b[?25l", 6);
  abAppend(&ab, "\x1b[H", 3);
  editorDrawRows(&ab);
  char buf[32];
  snprintf(buf, sizeof(buf), "\x1b[%d;%dH", E.cy + 1, E.cx + 1);
  abAppend(&ab, buf, strlen(buf));
  abAppend(&ab, "\x1b[?25h", 6);
  write(STDOUT_FILENO, ab.b, ab.len);
  abFree(&ab);
}
```

我们将旧的 H 命令更改为带有参数的 H 命令，指定了光标要移动到的确切位置。（请确保删除了旧的 H 命令，因为上面的差异很容易使其丢失。）
我们将 1 添加到 `E.cy` 和 `E.cx` 中，以将 0 索引值转换为终端使用的 1 索引值。
此时，您可以尝试将 `E.cx` 初始化为 10 或其他值，或将 `E.cx++`插入主循环中，以确认代码是否按预期工作。
接下来，我们将允许用户使用 wasd 键移动光标。（如果您不熟悉将这些键用作箭头键：w 是向上箭头，s 是向下箭头，a 是左，d 是右。）

```c
/*** includes ***/
/*** defines ***/
/*** data ***/
/*** terminal ***/
/*** append buffer ***/
/*** output ***/
/*** input ***/
void editorMoveCursor(char key) {
  switch (key) {
    case 'a':
      E.cx--;
      break;
    case 'd':
      E.cx++;
      break;
    case 'w':
      E.cy--;
      break;
    case 's':
      E.cy++;
      break;
  }
}
void editorProcessKeypress() {
  char c = editorReadKey();
  switch (c) {
    case CTRL_KEY('q'):
      write(STDOUT_FILENO, "\x1b[2J", 4);
      write(STDOUT_FILENO, "\x1b[H", 3);
      exit(0);
      break;
    case 'w':
    case 's':
    case 'a':
    case 'd':
      editorMoveCursor(c);
      break;
  }
}
```

# 方向键

现在我们有了一种映射按键以移动光标的方法，让我们用箭头键替换 `wasd` 键。上一章我们看到，按箭头键会向程序发送多个字节作为输入。这些字节的形式是转义序列，以`“\x1b”、“[”开头，然后是“A”、“B”、“C”或“D”`，具体取决于四个箭头键中按下的是哪一个。让我们修改 editorReadKey（），将此形式的转义序列作为单个按键读取。

```c
/*** includes ***/
/*** defines ***/
#define KILO_VERSION "0.0.1"
#define CTRL_KEY(k) ((k) & 0x1f)

enum editorKey {
  ARROW_LEFT = 'a',
  ARROW_RIGHT = 'd',
  ARROW_UP = 'w',
  ARROW_DOWN = 's'
};
/*** data ***/
/*** terminal ***/
void die(const char *s) { … }
void disableRawMode() { … }
void enableRawMode() { … }
char editorReadKey() {
  int nread;
  char c;
  while ((nread = read(STDIN_FILENO, &c, 1)) != 1) {
    if (nread == -1 && errno != EAGAIN) die("read");
  }
  if (c == '\x1b') {
    char seq[3];
    if (read(STDIN_FILENO, &seq[0], 1) != 1) return '\x1b';
    if (read(STDIN_FILENO, &seq[1], 1) != 1) return '\x1b';
    if (seq[0] == '[') {
      switch (seq[1]) {
        case 'A': return ARROW_UP;
        case 'B': return ARROW_DOWN;
        case 'C': return ARROW_RIGHT;
        case 'D': return ARROW_LEFT;
      }
    }
    return '\x1b';
  } else {
    return c;
  }
}
int getCursorPosition(int *rows, int *cols) { … }
int getWindowSize(int *rows, int *cols) { … }
/*** append buffer ***/
/*** output ***/
/*** input ***/
void editorMoveCursor(char key) {
  switch (key) {
    case ARROW_LEFT:
      E.cx--;
      break;
    case ARROW_RIGHT:
      E.cx++;
      break;
    case ARROW_UP:
      E.cy--;
      break;
    case ARROW_DOWN:
      E.cy++;
      break;
  }
}
void editorProcessKeypress() {
  char c = editorReadKey();
  switch (c) {
    case CTRL_KEY('q'):
      write(STDOUT_FILENO, "\x1b[2J", 4);
      write(STDOUT_FILENO, "\x1b[H", 3);
      exit(0);
      break;
    case ARROW_UP:
    case ARROW_DOWN:
    case ARROW_LEFT:
    case ARROW_RIGHT:
      editorMoveCursor(c);
      break;
  }
}
```

现在我们只需要在 EditWorky 枚举中为箭头键选择一个与 wasd 不冲突的表示形式。我们将给它们一个超出字符范围的大整数值，这样它们就不会与任何普通按键冲突。我们还必须将存储按键的所有变量更改为 int 类型而不是 char 类型。

# 防止移动到屏幕外面

```c
void editorMoveCursor(int key) {
  switch (key) {
    case ARROW_LEFT:
      if (E.cx != 0) {
        E.cx--;
      }
      break;
    case ARROW_RIGHT:
      if (E.cx != E.screencols - 1) {
        E.cx++;
      }
      break;
    case ARROW_UP:
      if (E.cy != 0) {
        E.cy--;
      }
      break;
    case ARROW_DOWN:
      if (E.cy != E.screenrows - 1) {
        E.cy++;
      }
      break;
  }
}
```

# 翻页键

为了完成我们的低级终端代码，我们需要检测更多使用转义序列的特殊按键，就像箭头键那样。我们将从 Page Up 和 Page Down 键开始。向上翻页作为`<esc>[5`发送，向下翻页作为`<esc>[6`发送。

```c
int editorReadKey() {
  int nread;
  char c;
  while ((nread = read(STDIN_FILENO, &c, 1)) != 1) {
    if (nread == -1 && errno != EAGAIN) die("read");
  }
  if (c == '\x1b') {
    char seq[3];
    if (read(STDIN_FILENO, &seq[0], 1) != 1) return '\x1b';
    if (read(STDIN_FILENO, &seq[1], 1) != 1) return '\x1b';
    if (seq[0] == '[') {
      if (seq[1] >= '0' && seq[1] <= '9') {
        if (read(STDIN_FILENO, &seq[2], 1) != 1) return '\x1b';
        if (seq[2] == '~') {
          switch (seq[1]) {
            case '5': return PAGE_UP;
            case '6': return PAGE_DOWN;
          }
        }
      } else {
        switch (seq[1]) {
          case 'A': return ARROW_UP;
          case 'B': return ARROW_DOWN;
          case 'C': return ARROW_RIGHT;
          case 'D': return ARROW_LEFT;
        }
      }
    }
    return '\x1b';
  } else {
    return c;
  }
}


void editorProcessKeypress() {
  int c = editorReadKey();
  switch (c) {
    case CTRL_KEY('q'):
      write(STDOUT_FILENO, "\x1b[2J", 4);
      write(STDOUT_FILENO, "\x1b[H", 3);
      exit(0);
      break;
    case PAGE_UP:
    case PAGE_DOWN:
      {
        int times = E.screenrows;
        while (times--)
          editorMoveCursor(c == PAGE_UP ? ARROW_UP : ARROW_DOWN);
      }
      break;
    case ARROW_UP:
    case ARROW_DOWN:
    case ARROW_LEFT:
    case ARROW_RIGHT:
      editorMoveCursor(c);
      break;
  }
}
```

处理上下翻页键的时候往下翻一个屏幕那么多，先计算一个屏幕多少行，然后循环那么多次

# home 和 end 键

现在让我们实现 Home 和 End 键。与前面的键一样，这些键也发送转义序列。与前面的键不同，这些键可以发送许多不同的转义序列，具体取决于您的操作系统或终端模拟器。Home 键可以作为`<esc>[1~，<esc>[7~，<esc>[H]或<esc>[OH]`发送。类似地，End 键可以作为`<esc>[4~，<esc>[8~，<esc>[F]或<esc>OF`发送。让我们来处理所有这些情况。

# delete 键

最后，让我们检测一下何时按下 Delete 键。它只发送转义序列`<esc>[3~`，因此很容易添加到我们的 switch 语句中。我们现在不会让这个键做任何事情。
