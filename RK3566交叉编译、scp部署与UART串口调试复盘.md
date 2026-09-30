# RK3566 Linux 项目阶段性总复盘：命令、代码、踩坑一次梳理清楚

现在正适合停一下统一整理。你已经跑通了最重要的一条链路：

```text
xhr：Ubuntu 18.04 虚拟机，x86_64
        │
        │ VS Code 写代码
        │
        ▼
aarch64-linux-gnu-gcc
        │
        │ 交叉编译
        ▼
ARM64 ELF 可执行程序 rk_gateway
        │
        │ scp
        ▼
lckfb：RK3566，Ubuntu 20.04，aarch64
        │
        ▼
./rk_gateway
```

你现在还不需要把每条命令背下来。重点是先搞懂：**一条 Linux 命令里谁是命令、谁是参数、谁是路径、谁是 Shell 特殊符号。**

---

# 一、先彻底搞懂 Linux 命令到底怎么“分块”

例如：

```bash
aarch64-linux-gnu-gcc -Wall -Wextra -Iinclude src/main.c src/serial.c -o rk_gateway
```

Shell 大致会按空格拆成：

```text
aarch64-linux-gnu-gcc
│
└── 命令

-Wall
-Wextra
-Iinclude
│
└── 参数 / 选项

src/main.c
src/serial.c
│
└── 输入文件

-o
│
└── 一个选项

rk_gateway
│
└── -o 对应的参数：输出文件名
```

所以可以先建立：

```text
命令 参数 参数 文件 参数...
```

但有几个特殊字符**不是普通参数**：

```text
|       管道
>       覆盖式输出重定向
>>      追加式输出重定向
2>      错误输出重定向
*       通配符
~       当前用户 Home
.       当前目录
..      上一级目录
\       转义；放在行尾时表示命令下一行继续
$USER   Shell变量
```

这一块现在必须掌握，因为你刚刚的大坑就是 Shell 语法导致的。

---

# 二、路径相关命令

## `pwd`

```bash
pwd
```

意思：

```text
Print Working Directory
打印当前工作目录
```

例如：

```text
/home/xhr/rk3566_gateway
```

这个命令非常重要。以后怀疑“我到底在哪个目录操作”，先：

```bash
pwd
```

---

## `cd`

```bash
cd ~/rk3566_gateway
```

拆开：

```text
cd
└── Change Directory，切换目录

~/rk3566_gateway
└── 目标路径
```

其中：

```text
~
```

代表当前用户 Home。

对 xhr：

```text
~ = /home/xhr
```

因此：

```text
~/rk3566_gateway
```

就是：

```text
/home/xhr/rk3566_gateway
```

---

## `mkdir -p`

```bash
mkdir -p ~/rk3566_gateway/src
```

拆开：

```text
mkdir
└── 创建目录

-p
└── 父目录不存在就一起创建
   已存在也不报错

~/rk3566_gateway/src
└── 要创建的路径
```

如果：

```text
rk3566_gateway
```

还不存在，`-p` 会一起创建。

---

## `ls`

```bash
ls
```

查看当前目录。

```bash
ls -l
```

`-l`：

> long format，显示详细信息。

例如：

```text
-rwxr-xr-x 1 xhr xhr 9112 Sep 18 14:00 rk_gateway
```

你暂时只要知道能看：

```text
文件大小
权限
时间
文件名
```

---

# 三、查看文件内容的命令

## `cat`

```bash
cat src/main.c
```

意思：

> 把 `src/main.c` 内容原样打印到终端。

这次非常有用，因为：

```text
VS Code里面看起来有代码
≠
磁盘文件一定真的有代码
```

GCC 读取的是**磁盘文件**。

所以：

```bash
cat src/main.c
```

看到的才是真实磁盘内容。

---

## `cat -n`

```bash
cat -n src/main.c
```

`-n`：

> 给每一行加行号。

例如：

```text
1  #include <stdio.h>
2
3  int main(void)
```

排查代码时比纯 `cat` 更好看。

---

## `wc -l`

```bash
wc -l src/main.c
```

拆开：

```text
wc
└── word count，统计

-l
└── 只统计行数
```

如果：

```text
0 src/main.c
```

说明文件是空的。

---

## `stat`

```bash
stat src/main.c
```

可以看文件：

```text
大小
修改时间
权限
inode
```

排查“这个文件什么时候被改空的”很有用。

平时不用背。

---

# 四、查文件在哪里

## `find`

```bash
find . -name "main.c" -print
```

拆开：

```text
find
└── 查找

.
└── 从当前目录开始

-name "main.c"
└── 文件名叫 main.c

-print
└── 把结果打印出来
```

例如：

```text
./src/main.c
```

`.` 就是当前目录。

这个命令当时是为了排除：

> VS Code 编辑了一个 `main.c`，GCC 编译的是另一个 `main.c`。

---

## `realpath`

```bash
realpath src/main.c
```

把相对路径：

```text
src/main.c
```

转换成完整绝对路径：

```text
/home/xhr/rk3566_gateway/src/main.c
```

以后怀疑“到底是不是同一个文件”，很好用。

---

# 五、系统信息命令

## `uname -m`

```bash
uname -m
```

查看 CPU 架构。

RK3566：

```text
aarch64
```

你的 VM 大概率：

```text
x86_64
```

这就是为什么需要**交叉编译**。

---

## `uname -a`

```bash
uname -a
```

查看更完整的信息：

```text
内核版本
主机名
CPU架构
```

---

## `/etc/os-release`

```bash
cat /etc/os-release
```

查看发行版。

RK3566 是：

```text
Ubuntu 20.04.6 LTS
```

你的 VM 是 Ubuntu 18.04。

---

# 六、APT 软件安装

## `sudo apt update`

```bash
sudo apt update
```

拆开：

```text
sudo
└── 以管理员 root 权限执行

apt
└── Ubuntu 软件包管理工具

update
└── 更新软件包目录
```

特别注意：

```text
apt update
```

**不是升级软件。**

只是：

```text
去软件源看看现在有哪些版本可以安装
```

---

## `apt install`

例如：

```bash
sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu vim
```

意思：

```text
安装 ARM64 C 交叉编译器
安装 ARM64 C++ 交叉编译器
安装 Vim
```

你现在已经装好了：

```text
aarch64-linux-gnu-gcc 7.5
aarch64-linux-gnu-g++ 7.5
```

---

# 七、你碰到的 APT 锁问题

你遇到：

```text
Could not get lock /var/lib/dpkg/lock-frontend
```

当时原因是：

> Ubuntu 正在安装中文语言包。

APT/dpkg 同一时刻一般只允许**一个安装事务修改软件包数据库**。

类似：

```text
中文语言包正在安装
       ↓
占用了 dpkg lock
       ↓
你又 apt install gcc
       ↓
拒绝
```

我们检查过：

```bash
sudo lsof /var/lib/dpkg/lock-frontend
```

其中：

```text
lsof
```

可以理解成：

> 查看“哪个进程正在打开这个文件”。

---

还用过：

```bash
ps aux | grep -E 'apt|dpkg|unattended'
```

这里非常值得理解。

```text
ps aux
│
└── 查看当前系统进程

|
│
└── 管道：把前面的输出送给后面的命令

grep
│
└── 搜索文字

-E
│
└── 使用扩展正则表达式

'apt|dpkg|unattended'
│
└── 查 apt 或 dpkg 或 unattended
```

这里的：

```text
|
```

有两个不同角色。

外面的：

```bash
ps aux | grep ...
```

是 **Shell 管道**。

引号里面：

```text
apt|dpkg|unattended
```

是正则表达式的“或”。

这是两个完全不同的东西。

---

# 八、RK3566 上原本的 APT 依赖问题

我们还查过：

```bash
sudo dpkg --configure -a
```

意思：

> 把之前安装到一半、还没配置完成的软件包继续配置完。

---

```bash
sudo apt --fix-broken install
```

意思：

> 尝试修复破损的软件包依赖关系。

---

```bash
apt-mark showhold
```

查看哪些包被：

```text
hold
```

也就是：

> 锁定，不允许自动升级。

你的 RK3566 厂商镜像有**大量包被 hold**。

后来我们决定：

> 不在 RK3566 本机搭完整编译环境了。

所以这条支线现在已经结束。

**这些命令知道作用即可，不用记。**

---

# 九、为什么改成 VM 交叉编译

我们最终确定：

```text
xhr
Ubuntu VM
x86_64
↓
aarch64-linux-gnu-gcc
↓
ARM64程序
↓
scp
↓
RK3566
aarch64
```

这就是：

> Cross Compilation，交叉编译。

最重要的命令：

```bash
aarch64-linux-gnu-gcc --version
```

查看交叉编译器。

---

# 十、`which`

```bash
which aarch64-linux-gnu-gcc
```

查看：

> 你输入这个命令时，Shell 最终实际执行的是哪个程序？

一般：

```text
/usr/bin/aarch64-linux-gnu-gcc
```

这个特别适合回答你之前：

> “这些东西到底安装在哪里？”

---

# 十一、`dpkg -L`

```bash
dpkg -L gcc-aarch64-linux-gnu
```

拆开：

```text
dpkg
└── Debian/Ubuntu 底层软件包工具

-L
└── List files

gcc-aarch64-linux-gnu
└── 软件包名字
```

作用：

> 查看这个软件包到底往系统哪些目录安装了什么文件。

所以 Linux 安装软件并不是“乱扔文件”。

包管理器知道这些文件属于谁。

---

# 十二、VS Code

我们用过：

```bash
code .
```

拆开：

```text
code
└── 启动 VS Code

.
└── 打开当前目录
```

所以：

```bash
cd ~/rk3566_gateway
code .
```

就是：

> 用 VS Code 打开这个工程目录。

---

# 十三、真正最核心的 GCC 命令

现在这条一定要会看：

```bash
aarch64-linux-gnu-gcc -Wall -Wextra -Iinclude src/main.c src/serial.c -o rk_gateway
```

完整拆解：

```text
aarch64-linux-gnu-gcc
│
└── ARM64 Linux C交叉编译器

-Wall
│
└── 打开常用警告

-Wextra
│
└── 再打开一批额外警告

-Iinclude
│
├── -I = Include路径
└── 去 include/ 目录找头文件

src/main.c
src/serial.c
│
└── 要参与编译的C源文件

-o
│
└── output

rk_gateway
│
└── 最终可执行程序名字
```

也就是说：

```text
main.c
     ┐
     ├→ GCC → rk_gateway
     │
serial.c
     ┘
```

---

## GCC 实际内部还经历四步

现在只建立地图：

```text
.c
↓
预处理
↓
编译
↓
汇编
↓
链接
↓
ELF可执行文件
```

你遇到：

```text
undefined reference to `main'
```

是：

> **链接阶段错误。**

不是语法错误。

链接器在组装整个程序时发现：

```text
程序需要 main()
但找不到 main()
```

---

# 十四、为什么 GCC 成功以后没有任何输出？

Linux 工具有一个很常见的习惯：

> **没有消息就是好消息。**

例如：

```bash
aarch64-linux-gnu-gcc ...
```

如果成功：

```text
什么都不打印
```

然后终端直接回到：

```text
xhr@ubuntu:~/rk3566_gateway$
```

这是正常的。

---

# 十五、`file rk_gateway`

```bash
file rk_gateway
```

这是你现在很应该记住的一个命令。

作用：

> 判断文件究竟是什么类型。

你得到：

```text
ELF 64-bit LSB shared object, ARM aarch64...
```

这里目前最重要：

```text
ELF 64-bit
ARM aarch64
```

意味着：

> 这是 ARM64 Linux 程序。

你看到：

```text
shared object
```

暂时不用纠结。Ubuntu/GCC 默认 PIE 等机制可能导致 `file` 这样描述，它仍然是正常可执行程序。

这是：

> **知道结论即可，暂时不要往 ELF/PIE 里钻。**

---

# 十六、`scp`

你的核心命令：

```bash
scp rk_gateway lckfb@192.168.31.58:/home/lckfb/rk3566_gateway/
```

完整拆开：

```text
scp
│
└── Secure Copy，通过SSH传文件

rk_gateway
│
└── xhr本机源文件

lckfb@
│
└── RK3566用户名

192.168.31.58
│
└── RK3566 IP

:
│
└── 后面开始是远程路径

/home/lckfb/rk3566_gateway/
│
└── RK3566目标目录
```

逻辑：

```text
xhr/rk_gateway
      ↓
网络
      ↓
RK3566:/home/lckfb/rk3566_gateway/rk_gateway
```

### 重要坑

目标存在同名文件：

```text
scp 会直接覆盖。
```

不需要：

```bash
rm rk_gateway
```

以后别多删一步。

---

# 十七、`ssh`

```bash
ssh lckfb@192.168.31.58
```

含义：

```text
ssh
└── 远程登录

lckfb
└── 远程用户名

192.168.31.58
└── 目标机器
```

于是：

```text
xhr
↓ SSH
lckfb
```

你就进入 RK3566。

---

# 十八、运行程序

```bash
./rk_gateway
```

这里：

```text
.
```

代表：

> 当前目录。

因此：

```text
./rk_gateway
```

就是：

> 执行当前目录下的 `rk_gateway`。

为什么不能总直接：

```bash
rk_gateway
```

因为 Linux 默认不会自动去当前目录找程序。

---

# 十九、`sha256sum`

当时为了判断：

> “scp 到底有没有覆盖？”

我们用了：

```bash
sha256sum rk_gateway
```

在 xhr 算一次。

RK3566：

```bash
sha256sum /home/lckfb/rk3566_gateway/rk_gateway
```

如果两边：

```text
abcedf123...
```

完全一样：

> 两个文件内容 100% 一样。

这是非常可靠的验证方式。

---

# 二十、`strings`

我们用过：

```bash
strings rk_gateway | grep "Hello"
```

拆开：

```text
strings rk_gateway
│
└── 从二进制里找可读字符串

|
│
└── 管道

grep "Hello"
│
└── 只留下包含 Hello 的行
```

如果还看到：

```text
Hello RK3566!
```

就说明你编出来的程序里面仍然有旧字符串。

很好用。

---

# 二十一、串口设备查看

我们执行：

```bash
ls /dev/ttyUSB* 2>/dev/null
```

这个是你之前特别想弄懂的类型。

完整拆解：

```text
ls
│
└── 查看文件

/dev/ttyUSB*
│
├── /dev/
│   └── Linux设备文件目录
│
├── ttyUSB
│   └── USB串口设备名
│
└── *
    └── 通配符，匹配任意字符

2>
│
├── 2 = stderr，错误输出
└── > = 重定向

/dev/null
│
└── 黑洞，扔进去的数据直接丢弃
```

所以：

```bash
ls /dev/ttyUSB* 2>/dev/null
```

意思：

> 找所有 `/dev/ttyUSBxxx`，如果一个都没有，不要把错误信息显示出来。

同理：

```bash
ls /dev/ttyACM* 2>/dev/null
```

找 USB CDC 串口。

```bash
ls /dev/ttyS* 2>/dev/null
```

找板载串口。

你目前 RK3566 有：

```text
/dev/ttyS1
/dev/ttyS3
```

---

# 二十二、`dmesg`

```bash
sudo dmesg | grep -iE 'tty|serial'
```

拆解：

```text
sudo dmesg
│
└── 查看 Linux 内核日志

|
└── 管道

grep
│
├── -i
│   └── 忽略大小写
│
├── -E
│   └── 扩展正则
│
└── 'tty|serial'
    └── 找 tty 或 serial
```

用来找：

```text
串口驱动
USB串口识别
设备节点创建
```

---

还有：

```bash
sudo dmesg | tail -30
```

这里：

```text
tail
└── 看文本最后几行

-30
└── 最后30行
```

通常刚插 USB 设备以后特别好用。

---

# 二十三、你踩得最大的坑：`\` 和 `>`

这是目前最值得记住的一次。

我最开始给：

```bash
aarch64-linux-gnu-gcc \
    -Wall \
    -Wextra \
    -Iinclude \
    src/main.c \
    src/serial.c \
    -o rk_gateway
```

这里：

```text
\
```

放在行尾表示：

> 命令还没完，下一行继续。

Bash 自己会显示：

```text
>
```

例如：

```text
xhr$ aarch64-linux-gnu-gcc \
> -Wall \
> -Wextra \
> ...
```

重点：

```text
这个 > 是 Bash 自动显示的提示符。
```

你不需要输入。

---

## 你误输入 `>` 后发生了什么？

Shell 中真正的：

```text
>
```

叫：

> 输出重定向。

例如：

```bash
echo hello > test.txt
```

含义：

```text
hello
↓
写进 test.txt
```

而：

```bash
> src/main.c
```

会：

```text
把 main.c 截断成 0 字节
```

所以你的：

```text
main.c
```

直接被清空。

而项目目录突然出现：

```text
-Wall
-Wextra
-Iinclude
-o
```

也是因为你实际上做了类似：

```bash
> -Wall
> -Wextra
> -Iinclude
> -o
```

Shell 就真的创建了这些文件。

### 所以现在先坚持：

```bash
aarch64-linux-gnu-gcc -Wall -Wextra -Iinclude src/main.c src/serial.c -o rk_gateway
```

**一整行输入。**

等 Shell 更熟以后再写多行。

---

# 二十四、删除那些奇怪文件

我们用了：

```bash
rm -- -Wall -Wextra -Iinclude -o
```

为什么不是：

```bash
rm -Wall
```

因为：

```text
-Wall
```

以 `-` 开头。

很多 Linux 程序会认为：

```text
-xxxx
```

是参数。

所以：

```text
--
```

表示：

> 参数结束，后面的全部当普通文件名。

因此：

```bash
rm -- -Wall
```

意思：

```text
rm
│
├── --
│   └── 参数到此结束
│
└── -Wall
    └── 文件名
```

---

# 二十五、输出重定向和 `/dev/null`

你现在至少区分这几个：

```text
>
覆盖写入文件

>>
追加写入文件

2>
错误输出重定向

2>/dev/null
错误直接丢弃

|
把一个程序的正常输出送到另一个程序
```

例如：

```bash
ls /dev/ttyUSB* 2>/dev/null
```

不是管道。

而：

```bash
dmesg | grep tty
```

才是管道。

---

# 二十六、`history`

我们排查误输入时提过：

```bash
history | tail -30
```

意思：

```text
history
└── 显示你之前输入过的命令

|
└── 管道

tail -30
└── 只看最后30条
```

以后怀疑：

> “我刚才到底输入了什么？”

非常好用。

---

# 二十七、CMake 是解决什么问题的？

你问过：

> 每次加一个 `.c`，难道都要重新写 GCC 命令？

当然不会。

纯 GCC：

```bash
gcc main.c serial.c tcp.c protocol.c logger.c ...
```

文件一多就很麻烦。

所以我们接下来会使用：

```text
CMake
```

你可以先把它理解成：

> **工程构建配置管理器。**

类似 Keil 工程文件知道：

```text
工程里有哪些 .c
头文件在哪
用什么编译参数
输出什么程序
```

---

# 二十八、我们的 `CMakeLists.txt`

目前计划：

```cmake
cmake_minimum_required(VERSION 3.10)

project(rk3566_gateway C)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_compile_options(
    -Wall
    -Wextra
)

include_directories(
    include
)

add_executable(
    rk_gateway
    src/main.c
    src/serial.c
)
```

逐块理解：

```cmake
cmake_minimum_required(VERSION 3.10)
```

要求最低 CMake 版本。

---

```cmake
project(rk3566_gateway C)
```

项目名字：

```text
rk3566_gateway
```

语言：

```text
C
```

---

```cmake
set(CMAKE_C_STANDARD 11)
```

使用：

```text
C11
```

标准。

---

```cmake
add_compile_options(
    -Wall
    -Wextra
)
```

以后所有源码自动带上这些警告参数。

---

```cmake
include_directories(
    include
)
```

等价于 GCC 的：

```text
-Iinclude
```

---

```cmake
add_executable(
    rk_gateway
    src/main.c
    src/serial.c
)
```

告诉 CMake：

```text
main.c
serial.c
↓
生成
rk_gateway
```

---

# 二十九、交叉编译 Toolchain 文件

我们准备创建：

```text
cmake/aarch64.cmake
```

内容：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER aarch64-linux-gnu-g++)
```

意思：

```text
目标操作系统 = Linux
目标CPU = ARM64
C编译器 = aarch64-linux-gnu-gcc
C++编译器 = aarch64-linux-gnu-g++
```

---

第一次配置：

```bash
cmake -S . -B build -DCMAKE_TOOLCHAIN_FILE=cmake/aarch64.cmake
```

拆开：

```text
cmake

-S .
│
├── -S = Source
└── 源码目录是当前目录 .

-B build
│
├── -B = Build
└── 构建目录叫 build

-DCMAKE_TOOLCHAIN_FILE=...
└── 指定 ARM64交叉编译配置
```

第一次以后主要用：

```bash
cmake --build build
```

意思：

> 编译 `build/` 里的这个工程。

以后就不需要：

```bash
aarch64-linux-gnu-gcc -Wall ...
```

一长串了。

---

# 三十、现在的 `serial.h`

我们写的是：

```c
#ifndef SERIAL_H
#define SERIAL_H

#include <stddef.h>
#include <sys/types.h>

int serial_open(const char* device, int baudrate);
ssize_t serial_read(int fd, void* buffer, size_t size);
ssize_t serial_write(int fd, const void* buffer, size_t size);
void serial_close(int fd);

#endif
```

## `#ifndef / #define / #endif`

这是：

> Header Guard，头文件保护。

防止：

```text
serial.h
```

被重复包含。

目前知道作用即可。

---

## 为什么有 `.h`？

`serial.h` 对外宣布：

```text
串口模块有哪些功能可以使用
```

例如：

```c
serial_open();
serial_read();
serial_write();
serial_close();
```

`main.c` 不需要知道串口内部怎么配置。

---

# 三十一、现在的 `serial.c`

当前第一版：

```c
#include "serial.h"

#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <termios.h>
#include <unistd.h>

static speed_t serial_get_baudrate(int baudrate)
{
    switch (baudrate)
    {
    case 9600:
        return B9600;

    case 19200:
        return B19200;

    case 38400:
        return B38400;

    case 57600:
        return B57600;

    case 115200:
        return B115200;

    default:
        return 0;
    }
}
```

这里要理解：

Linux `termios` 不是直接接受：

```c
115200
```

而使用：

```c
B115200
```

所以这个函数做转换：

```text
115200
↓
B115200
```

---

# 三十二、`open()`

```c
int fd = open(device, O_RDWR | O_NOCTTY);
```

这是整个 Linux 串口最核心的地方之一。

例如：

```text
device = "/dev/ttyUSB0"
```

所以实际：

```c
open("/dev/ttyUSB0", ...);
```

---

## `O_RDWR`

```text
Read + Write
```

表示：

> 我要读，也要写。

---

## `O_NOCTTY`

表示：

> 不让这个串口成为当前进程的控制终端。

现在知道结论即可。

---

# 三十三、文件描述符 `fd`

成功后：

```c
int fd
```

可能得到：

```text
3
```

为什么串口会变成一个数字？

Linux 用户程序不是以后每次都传：

```text
"/dev/ttyUSB0"
```

而是：

```text
open("/dev/ttyUSB0")
        ↓
Linux返回 fd
        ↓
以后 read(fd)
write(fd)
close(fd)
```

你可以理解：

```text
fd = Linux给这个打开对象发的一张号码牌
```

**这个现在必须掌握。**

---

# 三十四、`termios`

```c
struct termios tty;
```

`termios` 就是：

> Linux 用户态配置串口的一套接口。

---

```c
tcgetattr(fd, &tty);
```

意思：

> 把当前串口配置读取到 `tty`。

---

```c
cfsetispeed(&tty, speed);
cfsetospeed(&tty, speed);
```

分别配置：

```text
输入波特率
输出波特率
```

---

# 三十五、115200 8N1

这些：

```c
tty.c_cflag &= ~PARENB;
tty.c_cflag &= ~CSTOPB;
tty.c_cflag &= ~CSIZE;
tty.c_cflag |= CS8;
```

最终目的：

```text
8 data bits
No parity
1 stop bit
```

也就是：

```text
8N1
```

现在**不要逐位研究这些宏具体是哪一 bit**。

那是支线。

---

# 三十六、`CRTSCTS`

我们原来写：

```c
tty.c_cflag &= ~CRTSCTS;
```

作用：

> 关闭 RTS/CTS 硬件流控。

你 VS Code 报：

```text
未定义标识符 CRTSCTS
```

于是改成：

```c
#ifdef CRTSCTS
    tty.c_cflag &= ~CRTSCTS;
#endif
```

意思：

```text
如果系统定义了 CRTSCTS
↓
执行关闭硬件流控

没有定义
↓
直接跳过
```

这个问题目前属于：

> **知道结论即可，不要研究宏定义底层来源。**

另外还有一种可能：

> VS Code IntelliSense 的头文件环境和真实 ARM64 GCC 环境不完全一致。

所以以后判断代码有没有问题，**最终以真实 GCC 编译结果为准**，VS Code 红线只是辅助。

---

# 三十七、这些串口标志现在只记结果

```c
tty.c_cflag |= CREAD | CLOCAL;
```

目前理解：

```text
CREAD
→ 开启接收

CLOCAL
→ 忽略一些 Modem 控制信号
```

---

```c
tty.c_lflag &= ~ICANON;
```

关闭 canonical mode。

简单理解：

> 不要等用户敲一整行再给程序，串口数据来了就按原始方式处理。

---

```c
tty.c_lflag &= ~ECHO;
```

不回显。

---

```c
tty.c_lflag &= ~ISIG;
```

不把一些字符当终端信号处理。

这些都属于：

> **当前知道工程意义即可。**

---

# 三十八、`VMIN` / `VTIME`

我们写：

```c
tty.c_cc[VMIN] = 1;
tty.c_cc[VTIME] = 0;
```

当前效果可以理解成：

> `read()` 至少等到 1 个字节才返回。

所以：

```c
read(...)
```

没有数据时会阻塞。

这以后跟多线程设计关系很大。

但目前不用深入所有组合。

---

# 三十九、`tcsetattr`

```c
tcsetattr(fd, TCSANOW, &tty);
```

意思：

> 把我们修改后的串口配置真正写回这个串口。

其中：

```text
TCSANOW
```

可以理解：

> 立即生效。

---

# 四十、`read / write / close`

我们的封装：

```c
ssize_t serial_read(int fd, void* buffer, size_t size)
{
    return read(fd, buffer, size);
}
```

其实目前就是：

```text
serial_read
↓
Linux read()
```

---

```c
serial_write(...)
```

本质：

```c
write(...)
```

---

```c
serial_close(...)
```

本质：

```c
close(fd)
```

所以这个模块目前没有魔法。

只是把 Linux API 封装成我们项目自己的：

```text
serial_xxx()
```

---

# 四十一、现在的 `main.c`

核心结构：

```c
#define SERIAL_DEVICE "/dev/ttyUSB0"
#define SERIAL_BAUDRATE 115200
```

先指定：

```text
串口设备
波特率
```

以后会移到配置文件，而不是写死。

---

```c
int serial_fd = serial_open(SERIAL_DEVICE, SERIAL_BAUDRATE);
```

打开串口。

如果失败：

```c
if (serial_fd < 0)
{
    return -1;
}
```

程序退出。

---

# 四十二、接收 Buffer

```c
char buffer[256];
```

在栈上申请：

```text
256字节
```

缓冲区。

---

```c
ssize_t len = serial_read(
    serial_fd,
    buffer,
    sizeof(buffer) - 1);
```

其中：

```text
sizeof(buffer)
= 256

sizeof(buffer) - 1
= 255
```

为什么只读 255？

因为后面还要：

```c
buffer[len] = '\0';
```

给 C 字符串结尾留一个：

```text
'\0'
```

位置。

这个细节值得理解。

---

# 四十三、但是这里以后一定要改

现在：

```c
printf("RX: %s", buffer);
```

适合测试：

```text
Hello ESP32
```

这种文本。

但正式协议以后可能是：

```text
AA 55 01 FF 00 ...
```

二进制数据。

二进制里面可能本来就有：

```text
0x00
```

这时不能再简单 `%s` 打印。

以后会改成：

```text
按长度处理
协议缓存
十六进制日志
```

所以当前这只是：

> **UART bring-up 测试代码。**

不是最终协议实现。

---

# 四十四、最重要的串口结论：一次 `read()` ≠ 一帧

例如 ESP32 发：

```text
AA 55 01 02 03 04
```

Linux 可能：

第一次：

```text
AA 55
```

第二次：

```text
01 02 03
```

第三次：

```text
04
```

也可能一次：

```text
AA 55 01 02 03 04
```

甚至：

```text
第一帧 + 第二帧
```

一起回来。

所以以后必须：

```text
read()
↓
接收缓存
↓
找帧头
↓
判断长度
↓
CRC
↓
取出完整帧
```

这个是下一阶段非常核心的东西。

---

# 四十五、你目前遇到的几个错误，统一归因

### 1. `undefined reference to main`

第一次：

```text
VS Code没有保存
↓
磁盘 main.c 还是空的
↓
GCC看不到 main()
```

后来又出现一次：

```text
误把 > 输入到命令里
↓
> src/main.c
↓
main.c 被截断为空
↓
又找不到 main()
```

所以：

> **相同报错可以有不同根因。**

以后不要只根据报错文字直接猜原因，要看事实。

---

### 2. `file rk_gateway: No such file`

因为前面的编译：

```text
失败
```

所以：

```text
rk_gateway 根本没生成
```

自然：

```bash
file rk_gateway
```

找不到。

---

### 3. SCP 后仍然 Hello World

你怀疑：

> SCP 没覆盖旧文件。

实际上：

```text
SCP覆盖成功
```

问题是：

```text
你编译出来的本身就是旧程序
```

因为源码没保存。

所以以后排查：

```text
源代码
↓
编译产物
↓
传输文件
↓
目标文件
↓
实际运行文件
```

一层一层验证。

---

### 4. `CRTSCTS` 红线

目前：

```text
VS Code IntelliSense环境差异
或者平台宏没定义
```

使用：

```c
#ifdef CRTSCTS
...
#endif
```

先解决。

---

### 5. APT Lock

原因：

```text
中文语言包装到一半
↓
dpkg正在工作
↓
另一个apt不能同时修改数据库
```

不是 GCC 安装包坏了。

---

### 6. RK3566 装 GCC 失败

厂商 Ubuntu 镜像大量软件包：

```text
hold
```

我们后来发现根本没必要硬搞。

正确主线已经变成：

```text
VM交叉编译
↓
RK3566运行
```

这才是我们继续走的方向。

---

# 四十六、现在真正需要你记住的命令不多

目前我希望你逐渐形成肌肉记忆的是：

```bash
pwd
cd
ls
mkdir
cat
```

文件和目录基础。

然后：

```bash
aarch64-linux-gnu-gcc
file
```

交叉编译和检查。

然后：

```bash
ssh
scp
```

Host ↔ Target。

然后：

```bash
./rk_gateway
```

运行。

再然后：

```bash
ls /dev/tty*
dmesg
```

硬件设备排查。

最后马上会进入：

```bash
cmake --build build
```

正常工程构建。

其他：

```text
apt-mark
dpkg --configure
lsof lock
sed
stat
sha256sum
strings
```

都属于：

> **碰到问题会查、看得懂即可，不需要背。**

---

# 四十七、你当前真正应该形成的工程模型

不是背 50 条 Linux 命令。

而是这张图：

```text
             xhr Ubuntu VM
                  │
              VS Code
                  │
          main.c / serial.c
                  │
                  ▼
      aarch64-linux-gnu-gcc
                  │
                  ▼
             rk_gateway
             ARM64 ELF
                  │
                 SCP
                  │
                  ▼
             RK3566 Linux
                  │
             /dev/ttyXXX
                  │
                 open
                  │
             termios配置
                  │
                 read
                  │
                  ▼
               数据
```

**这个现在必须理解。**

至于：

```text
ELF内部结构
动态链接器
tty内核子系统
VFS
UART驱动模型
设备树
pinctrl
```

全部先放着。

那些属于第二阶段。

---

# 四十八、接下来不要继续扩知识点

我们现在已经有点接近“Linux 命令支线太多”了。

下一条主线应该回到：

```text
① 把 main.c / serial.c / serial.h 保存好
② 单行 GCC 编译一次确认
③ 正式接入 CMake
④ cmake --build build
⑤ SCP 到 RK3566
⑥ 中午接 ESP32
⑦ /dev/ttyUSB0 收到真实数据
```

完成第 ⑦ 步以后，我们再统一学习：

```text
read为什么会半包
怎么设计接收缓存
协议解析状态机
```

那才是这个项目下一块真正有面试价值的内容。