# 自己写一个文本编辑器
## makefile
```makefile
kilo: kilo.c
	$(CC) kilo.c -o kilo -Wall -Wextra -pedantic -std=c99
```
kilo是需要构建出来的，kilo.c是为了构建kilo 需要的东西。  
第二行是为了构建这个kilo真正需要运行的东西。前面必须是tab不是空格
- `$(CC)` 是一个变量，make 默认扩展为cc
- -Wall表示全部警告
- Wextra和-pedantic打开了更多的警告。对于本教程中的每个步骤，如果程序正在编译，则在某些情况下，除了“未使用的变量”警告外，它不应生成任何警告。如果收到任何其他警告，请检查以确保您的代码与该步骤中的代码完全匹配。
- -std=c99 指定我们使用的 C 语言标准的确切版本，即 C99。C99允许我们在函数中的任何位置声明变量，而ANSI C要求所有变量都在函数或块的顶部声明。

# 进入raw模式
正常情况下终端处在一种 **canonical mode, also called cooked mode**但是我们想要的是一种**raw模式**但是这个模式得慢慢进去，要改掉很多部分所以在后面慢慢改。
# 按‘q’退出
```c
#include <unistd.h>
int main() {
  char c;
  while (read(STDIN_FILENO, &c, 1) == 1 && c != 'q');
  return 0;
}
```
# 关闭回显：
```c
#include <termios.h>
#include <unistd.h>
void enableRawMode() {
  struct termios raw;
  tcgetattr(STDIN_FILENO, &raw);
  raw.c_lflag &= ~(ECHO);
  tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}
int main() {
  enableRawMode();
  char c;
  while (read(STDIN_FILENO, &c, 1) == 1 && c != 'q');
  return 0;
}
```

终端属性可以通过 `tcgetattr（）` 读入 `termios `结构体中。修改它们后，您可以使用`tcsetattr（） `将它们应用到终端。`TCSAFLUSH` 参数指定何时应用更改：在这种情况下，它等待所有挂起的输出写入终端，并且还会丢弃任何尚未读取的输入。  

`c_lflag`字段用于“本地标志”。macOS的<termios.h>评论将其描述为“dumping ground for other state”。因此，也许应该将其视为“杂项标志”。  
其他标志字段是`c_iflag`（输入标志），`c_oflag`（输出标志）和`c_cflag`（控制标志），所有这些都是我们必须修改以启用原始模式。  

`ECHO` 是位标志，定义为二进制`00000000000000000000000000001000`。我们对此值使用按位 NOT 运算符 （~） 来获取`11111111111111111111111111110111`。然后，我们将此值与 flags 字段按位与(AND)，这会强制 flags 字段中的第四位变为 0，其他位保持不变。像这样的翻转位在C中很常见。并导致每隔一个位保留其当前值


# 退出时关闭raw模式
在程序退出时恢复其终端的原始属性。我们将以原始状态保存 `termios` 结构的副本，并在程序退出时使用 `tcsetattr（）` 将其应用于终端。
```c
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
struct termios orig_termios;
void disableRawMode() {
  tcsetattr(STDIN_FILENO, TCSAFLUSH, &orig_termios);
}
void enableRawMode() {
  tcgetattr(STDIN_FILENO, &orig_termios);
  atexit(disableRawMode);
  struct termios raw = orig_termios;
  raw.c_lflag &= ~(ECHO);
  tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}
int main() { … }
```
`atexit()` comes from <stdlib.h>. 通过这个函数让咱们的disableRawMode在main函数结束的时候自动调用一些函数。

我们将原始终端属性存储在全局变量中，`orig_termios`。我们将`orig_termios`结构赋值给`raw`，以便在开始进行更改之前复制它。

您可能会注意到，在程序退出后，剩余的输入不再输入到 `shell` 中。这是因为当程序退出时，`TCSAFLUSH` 选项将传递给 `tcsetattr`（）。如前所述，它会在将更改应用于终端之前丢弃任何未读输入。（注意：由于某种原因，这在Cygwin中不会发生，但是一旦我们一次读取一个字节的输入，这就无关紧要了。

# 关闭规范模式
有一个`ICANON`标志，允许我们关闭规范模式。这意味着我们最终将逐字节读取输入，而不是逐行读取。

`ICANON`来自`<termios.h>`。输入标志（c_iflag字段中的标志）通常以I开头，像`ICANON`。但是，`ICANON`不是输入标志，它是c_lflag字段中的“本地”标志。所以这很令人困惑。

# 显示按下了什么
为了更好地了解原始模式下的输入是如何工作的，让我们打印出我们读取的每个字节。我们将打印每个字符的数字 ASCII 值，以及它所表示的字符（如果它是可打印字符）。
```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
struct termios orig_termios;
void disableRawMode() { … }
void enableRawMode() { … }
int main() {
  enableRawMode();
  char c;
  while (read(STDIN_FILENO, &c, 1) == 1 && c != 'q') {
    if (iscntrl(c)) {
      printf("%d\n", c);
    } else {
      printf("%d ('%c')\n", c, c);
    }
  }
  return 0;
}
```
`iscntrl()` comes from `<ctype.h>`, and `printf()` comes from` <stdio.h>`
`iscntrl（）`测试字符是否为控制字符。控制字符是我们不想打印到屏幕上的不可打印字符。ASCII码0–31都是控制字符，127也是控制字符。ASCII码32–126都可以打印。（查看ASCII表以查看所有字符。）
# 关闭ctrl+c ctrl+z信号
默认情况下，`Ctrl-C`向当前进程发送一个`SIGINT`信号，使其终止，而`Ctrl-Z`向当前进程发送一个`SIGTSTP`信号，使其暂停。让我们关闭这两个信号的发送。
```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
struct termios orig_termios;
void disableRawMode() { … }
void enableRawMode() {
  tcgetattr(STDIN_FILENO, &orig_termios);
  atexit(disableRawMode);
  struct termios raw = orig_termios;
  raw.c_lflag &= ~(ECHO | ICANON | ISIG);
  tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}
int main() { … }
```
# 关闭ctrl+s ctrl+q
默认情况下，`Ctrl-S`和`Ctrl-Q`用于软件流控制。在按下`Ctrl-Q`键之前，`Ctrl-S`会阻止数据传输到终端。这始于您可能希望暂停数据传输以让打印机等设备赶上的日子。让我们关掉这个功能。  
```c
raw.c_iflag &= ~(IXON);
```
IXON来自termios。h> 。I代表“input flag”（与我们目前看到的其他I标志不同），XON来自Ctrl-S和Ctrl-Q产生的两个控制字符的名称：XOFF暂停传输，XON恢复传输。
# 关闭ctrl+v
在某些系统中，当您键入Ctrl-V时，终端会等待您键入另一个字符，然后按字面意思发送该字符。例如，在禁用Ctrl-C之前，您可能已经能够键入Ctrl-V，然后键入Ctrl-C以输入一个3字节。我们可以使用`IEXTEN`标志关闭此功能。
# 修复ctrl+M
如果你现在运行这个程序，在按住Ctrl键的同时浏览整个字母表，你应该会看到除了M之外，我们有所有的字母。Ctrl-M很奇怪：它被读为10，而我们期望它被读为13，因为它是字母表中的第13个字母，而Ctrl-J已经产生了10。还有一个按键会生成10 回车键。 

结果表明，终端正在帮助将用户输入的任何回车（13，“\r”）转换为换行符（10，“\n”）。让我们关闭这个功能。
`raw.c_iflag &= ~(ICRNL | IXON);`
# 关闭输出处理
结果表明，终端在输出端进行了类似的转换。它将我们打印的每一个换行符（“\n”）转换为一个回车符，后跟一个换行符（“\r\n”）。终端需要这两个字符才能开始新的文本行。回车键将光标移回当前行的开头，换行符将光标向下移动一行，必要时滚动屏幕。（这两种截然不同的操作起源于打字机和电传打字机时代。）

我们将通过关闭`OPOST`标志来关闭所有输出处理功能。实际上，“\n”到“\r\n”的翻译可能是默认情况下唯一打开的输出处理功能。
```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
struct termios orig_termios;
void disableRawMode() { … }
void enableRawMode() {
  tcgetattr(STDIN_FILENO, &orig_termios);
  atexit(disableRawMode);
  struct termios raw = orig_termios;
  raw.c_iflag &= ~(ICRNL | IXON);
  raw.c_oflag &= ~(OPOST);
  raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);
  tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}
int main() { … }
```
`OPOST`来自`termios.h` 。`O`表示它是一个输出标志，我假设POST代表“输出的后处理”

如果你现在运行这个程序，你会看到我们正在打印的换行符只是向下移动光标，而不是移动到屏幕的左侧。为了解决这个问题，我们将回车添加到printf（）语句中。

从现在开始，每当我们想开始新的一行时，我们都必须写出完整的“\r\n”。
# 杂项标志位
```c
  raw.c_iflag &= ~(BRKINT | ICRNL | INPCK | ISTRIP | IXON);
  raw.c_oflag &= ~(OPOST);
  raw.c_cflag |= (CS8);
  raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);
```
这一步可能不会对您产生任何明显的影响，因为这些标志要么已经关闭，要么它们并不真正适用于现代终端模拟器。但在某个时候，关闭它们被（某人）认为是启用“原始模式”的一部分，所以我们在程序中继承了（某人的）传统。

据我所知：
- 当`BRKINT`打开时，中断条件将导致向程序发送SIGINT信号，如按下Ctrl-C。
- `INPCK`支持奇偶校验，这似乎不适用于现代终端模拟器。
- `ISTRIP`会导致每个输入字节的第8位被剥离，这意味着它会将其设置为0。这可能已经关闭了。
- `CS8`不是一个标志，它是一个包含多个位的位掩码，我们使用按位或（|）运算符设置它，这与我们关闭的所有标志不同。它将字符大小（CS）设置为每字节8位。在我的系统中，它已经设置为那样。

# read()的超时
```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
struct termios orig_termios;
void disableRawMode() { … }
void enableRawMode() {
  tcgetattr(STDIN_FILENO, &orig_termios);
  atexit(disableRawMode);
  struct termios raw = orig_termios;
  raw.c_iflag &= ~(BRKINT | ICRNL | INPCK | ISTRIP | IXON);
  raw.c_oflag &= ~(OPOST);
  raw.c_cflag |= (CS8);
  raw.c_lflag &= ~(ECHO | ICANON | IEXTEN | ISIG);
  raw.c_cc[VMIN] = 0;
  raw.c_cc[VTIME] = 1;
  tcsetattr(STDIN_FILENO, TCSAFLUSH, &raw);
}
int main() {
  enableRawMode();
  while (1) {
    char c = '\0';
    read(STDIN_FILENO, &c, 1);
    if (iscntrl(c)) {
      printf("%d\r\n", c);
    } else {
      printf("%d ('%c')\r\n", c, c);
    }
    if (c == 'q') break;
  }
  return 0;
}
```
当前，read（）将无限期地等待键盘输入，然后返回。如果我们想在等待用户输入的同时在屏幕上制作动画，该怎么办？我们可以设置一个超时，这样read（）在一定时间内没有任何输入时就会返回。

VMIN和VTIME来自<termios。h> 。它们是c_cc字段的索引，该字段代表“控制字符”，一个控制各种终端设置的字节数组。

VMIN值设置read（）返回之前所需的最小输入字节数。我们将其设置为0，以便read（）在有任何输入要读取时立即返回。VTIME值设置read（）返回前等待的最长时间。它以十分之一秒为单位，所以我们将其设置为1/10秒，即100毫秒。如果read（）超时，它将返回0，这是有意义的，因为它通常的返回值是读取的字节数。
# 错误处理
enableRawMode（）现在让我们完全进入原始模式。是时候通过添加一些错误处理来清理代码了。

首先，我们将添加一个die（）函数，用于打印错误消息并退出程序。
```c
#include <ctype.h>
#include <stdio.h>
#include <stdlib.h>
#include <termios.h>
#include <unistd.h>
struct termios orig_termios;
void die(const char *s) {
  perror(s);
  exit(1);
}
```

大多数失败的C库函数都会设置全局errno变量来指示错误是什么。perror（）查看全局errno变量并为其打印一条描述性错误消息。它还会在打印错误消息之前打印给定的字符串，这意味着要提供有关代码中哪一部分导致错误的上下文。  
打印出错误消息后，我们将退出程序，退出状态为1，这表示失败（与任何非零值一样）。

`errno` and `EAGAIN` come from `<errno.h>`.
在Cygwin中，read（）超时时返回-1，errno为EAGAIN，而不是像预期的那样只返回0。为了让它在Cygwin中发挥作用，我们不会将EAGAIN视为一个错误。

使tcgetattr（）失败的一个简单方法是给程序一个文本文件或管道作为标准输入，而不是终端。要将文件作为标准输入，请运行`./kilo <kilo.c`给它一个管道，运行回声测试`echo test | ./kilo`这两种方法都会导致tcgetattr（）中出现相同的错误，比如设备的ioctl不合适。