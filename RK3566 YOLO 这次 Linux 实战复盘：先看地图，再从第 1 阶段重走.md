# RK3566 YOLO 这次 Linux 实战复盘：先看地图，再从第 1 阶段重走

这份日志非常适合拿来学 Linux，因为它不是“干净教程”，而是真实工程现场：**交叉编译器检查 → Git 下载 → 权限错误 → 缺 CMake → CMake 版本冲突 → PATH → root 带来的路径/权限问题 → Python 虚拟环境 → pip 网络问题 → 模型转换 → 文件部署**。

先不逐条翻译。第一遍只建立地图和排错模型。

---

## 一、这次到底做了什么：流程地图

### 阶段 1：确认 PC 具备 RK3566 交叉编译能力

目标：

```text
x86 Ubuntu PC
    ↓
aarch64-linux-gnu-gcc/g++
    ↓
生成 ARM64 / RK3566 可执行程序
```

你检查了：
[[aarch64-linux-gnu-gcc 到底是什么]]

```bash
which aarch64-linux-gnu-gcc
aarch64-linux-gnu-gcc --version
aarch64-linux-gnu-g++ --version
aarch64-linux-gnu-g++ -dumpmachine
```

最后确认目标是：

```text
aarch64-linux-gnu
```

这一阶段本质上是在回答：

> **“我的电脑上有没有一个能替 RK3566 编译程序的编译器？”**

而不是在编译 YOLO。

---

### 阶段 2：下载 Rockchip 官方源码

你执行：

```bash
git clone https://github.com/airockchip/rknn_model_zoo.git
```

前两次出现：

```text
Failed to connect to 192.168.36.1 port 7890
```

第三次才成功。

这是一个很典型的：

```text
Git 命令没问题
↓
网络/代理有问题
```

而不是：

```text
Git 仓库不存在
```

日志里只记录了失败后再次尝试，并没有记录代理到底怎样恢复，所以这里不能凭空认为是哪条命令修好的。

这里修改好是因为在clash verge里开启了局域网连接,因为这个Linux是VM里的虚拟机

我记得我之前在里面配置过让他可以使用我windows下的clash verge

只要我clash verge打开局域网连接就可以了


---

### 阶段 3：配置目标平台并尝试编译

你设置：

[[`export GCC_COMPILER=aarch64-linux-gnu` 是在干嘛？]]
[[换一个终端后通常要重新 `export`]]
```bash
export GCC_COMPILER=aarch64-linux-gnu
```

然后运行官方脚本：

```bash
./build-linux.sh -t rk3566 -a aarch64 -d yolov8
```

[[“脚本作者规定的”]]

结果：

```text
Permission denied
```

这里出现了第一个非常值得学习的 Linux 排错点。

你当时的思路是：

```text
Permission denied
↓
是不是权限不够？
↓
切 root
```

但实际情况是：

> **root 也执行不了，因为脚本文件本身没有 execute 权限。**

所以切 root 并没有解决真正原因。后来：

```bash
bash build-linux.sh ...
```

反而成功进入脚本。

这是这次最典型的一个“看到权限错误就 sudo/root”的弯路。

---

### 阶段 4：发现缺少构建工具 CMake

脚本终于执行以后出现：

```text
cmake: command not found
```

这句话的信息其实非常直接：

```text
Shell 想执行 cmake
↓
但找不到叫 cmake 的程序
```

所以你安装：

```bash
apt update
apt install -y cmake make
```

这里解决的是：

> **开发机缺少构建工具。**

不是 RK3566、YOLO 或交叉编译器有问题。

---

### 阶段 5：CMake 有了，但版本太旧

重新运行后出现：

```text
CMake 3.15 or higher is required.
You are running version 3.10.2
```

这时候问题已经从：

```text
“找不到 cmake”
```

升级成：

```text
“能找到 cmake，但版本不符合要求”
```

这是两类完全不同的问题。

后来你通过 Snap 安装了 CMake 4.4.3，但：

```bash
cmake --version
```

仍然显示：

```text
3.10.2
```

于是用：

```bash
which cmake
```

发现终端实际调用的是：

```text
/usr/bin/cmake
```

而新版在：

```text
/snap/bin/cmake
```

最后：

```bash
export PATH=/snap/bin:$PATH
```

之后：

```bash
which cmake
```

才变成：

```text
/snap/bin/cmake
```

这就是一次非常完整的 **PATH 排错实战**。

---

### 阶段 6：重新编译，得到 ARM64 程序

新版 CMake 工作后：

```text
Configuring done
Generating done
Building...
Linking...
[100%] Built target rknn_yolov8_demo
```

说明：

```text
源码
↓
CMake生成构建配置
↓
aarch64-linux-gnu-gcc/g++
↓
编译 + 链接
↓
ARM64程序
```

成功了。

然后你最终用：

```bash
file rknn_yolov8_demo
```

确认：

```text
ARM aarch64
```

这一步非常专业：

> **不要因为“编译没报错”就默认目标架构正确，直接用 `file` 验证产物。**

---

### 阶段 7：准备 RKNN 模型转换环境

这又是另一条链：

```text
YOLOv8n ONNX
↓
RKNN-Toolkit2
↓
RK3566 使用的 .rknn
```

注意：

> **这不是 C/C++ 交叉编译。**

你创建 Python 虚拟环境：

```bash
python3 -m venv ~/venvs/rknn
source ~/venvs/rknn/bin/activate
```

然后安装专门匹配 Python 3.6 的 RKNN Toolkit2 环境。

这里 Linux 学习重点不是 Torch/ONNX 本身，而是：

```text
Python虚拟环境
pip
目录
软件版本
```

---

### 阶段 8：处理 pip 下载慢和镜像问题

官方 PyPI 下载 `torch` 时只有几十 KB/s，881.9 MB 包需要很久，所以你主动 `Ctrl+C`。

这个判断是合理的。

日志明确显示 `torch` 下载到约 238 MB 时只有几十 KB/s，之后你终止了操作。

后来：

```text
清华源
↓
protobuf 找不到

阿里源
↓
protobuf 成功
↓
torch 成功
↓
其余 requirements 成功
```

这是一次软件源排错。

---

### 阶段 9：ONNX → RKNN

最后：

```bash
python convert.py ../model/yolov8n.onnx rk3566
```

生成：

```text
yolov8.rknn
```

这里属于：

> **项目工具知识，不是 Linux 命令知识。**

`convert.py` 后面怎么规定参数，是这个项目自己的事。

---

### 阶段 10：整理部署包并通过 SCP 发送到 RK3566

你遇到了：

```text
cp: Permission denied
```

原因是前面曾经用 root 编译，生成的 `install/` 文件归 root 所有。

最后：

```bash
sudo chown -R xhr:xhr ~/rknn_model_zoo
```

把工程所有权还给 `xhr`。

然后：

```bash
scp -r ...
```

把：

```text
可执行程序
动态库
模型
标签
测试图片
```

整体部署到 RK3566。

这就是完整的：

```text
开发机
↓
构建
↓
生成目标程序和模型
↓
打包运行依赖
↓
SCP
↓
目标板
```

---

# 二、真正值得你学的 Linux 核心知识

这 795 行日志看起来很多，但真正值得带走的 Linux 能力没有那么多。

|能力|这次在哪里出现|重要程度|
|---|---|---|
|当前目录与路径|`cd`、`~`、`..`|**必须熟练**|
|查看文件|`ls`、`ls -lh`|**必须熟练**|
|查命令位置|`which cmake`|**必须熟练**|
|用户与权限|`sudo`、root、`chown`|**必须理解**|
|文件执行权限|`./script` vs `bash script`|**必须理解**|
|环境变量|`export GCC_COMPILER=...`、`PATH`|**必须理解**|
|软件安装|`apt update/install`|**必须熟悉**|
|文件复制|`cp`|**必须熟练**|
|远程复制|`scp`|**必须熟悉**|
|Shell 换行|`\`|**见到能懂**|
|虚拟环境|`venv`、`source`|**开发中常用**|
|`file`|查看 ELF 架构|**嵌入式 Linux 很实用**|
|`rm -rf`|清 CMake 构建目录|**必须懂，但谨慎使用**|

而下面这些不要错当成 Linux 语法：

```text
build-linux.sh -t rk3566 -a aarch64 -d yolov8
convert.py ../model/yolov8n.onnx rk3566
requirements_cp36-2.3.2.txt
```

它们是**项目自己的工具和参数规则**。

---

# 三、你这次明显是在机械复制的地方

比较明显的有：

```bash
export GCC_COMPILER=aarch64-linux-gnu
```

你当时大概率知道“要输”，但还没有建立：

```text
变量名
↓
变量值
↓
当前 Shell 保存
↓
build-linux.sh 作为子进程读取
```

的模型。

还有：

```bash
export PATH=/snap/bin:$PATH
```

这是本次最值得重新学的一条。因为如果只是复制，你只会记得：

> CMake 不行就输这句话。

真正应该形成的模型是：

```text
系统里有两个 cmake
↓
Shell 按 PATH 从左到右找
↓
旧版 /usr/bin/cmake 先被找到
↓
把 /snap/bin 放前面
↓
新版优先
```

另外这些也有明显复制痕迹：

```bash
sudo chown -R xhr:xhr ...
python3 -m venv ...
source .../activate
pip install -r ... -i ... --default-timeout=120
scp -r ... user@ip:~/
```

这很正常。第一阶段允许复制，现在我们开始把它们变成“看到任务能想到大概该用什么”。

---

# 四、这次最值得复盘的坑

## 坑 1：`Permission denied` → 直接切 root

这是最需要纠正的习惯。

你看到：

```text
./build-linux.sh: Permission denied
```

然后去：

```bash
su root
```

甚至设置了 root 密码。

但 root 后执行：

```bash
./build-linux.sh
```

还是：

```text
Permission denied
```

已经证明问题根本不是“当前用户权限太低”。

专业一些的思路应该是：

```text
Permission denied
↓
先问：到底是哪一种权限问题？
↓
ls -l build-linux.sh
↓
有没有 x？
```

而不是：

```text
权限错误
↓
sudo/root
```

---

## 坑 2：切 root 后失去目录感

这是你这次非常典型的一次。

你人在：

```text
root@ubuntu:/home/xhr/rknn_model_zoo#
```

但：

```bash
cd ~/rknn_model_zoo/examples/yolov8/model
```

却跑去了：

```text
/root/rknn_model_zoo/...
```

因为：

```text
当前用户 = root
所以 ~ = /root
```

不是：

```text
~ = 当前工作目录的上级
```

也不是：

```text
~ 永远 = /home/xhr
```

日志里这个错误连续出现了几次，直到退出 root 才恢复正常。

这个坑非常值得记。

---

## 坑 3：软件“安装了”不代表 Shell 用的是它

你装了：

```text
CMake 4.4.3
```

但：

```bash
cmake --version
```

还是：

```text
3.10.2
```

真正有效的排查是：

```bash
which cmake
```

结果：

```text
/usr/bin/cmake
```

这才找到根因。

以后遇到：

```text
我明明装了新版 Python/GCC/CMake，为什么还是旧版？
```

第一反应应该逐渐变成：

```bash
which xxx
xxx --version
```

---

## 坑 4：用 root 编译，后面留下所有权问题

前面图省事在 root 下编译：

```text
root
↓
生成 build/
install/
```

于是后面普通用户 `xhr` 想：

```bash
cp yolov8.rknn install/.../
```

得到：

```text
Permission denied
```

这就是一个“前面的不规范操作，过几个阶段才爆雷”的典型 Linux 问题。

所以更专业的习惯是：

> **源码、编译、Git、Python 环境都用普通用户。只有真正修改系统时才 sudo。**

---

# 五、现在开始重新走第 1 阶段：确认交叉编译环境

这一轮先只学这一阶段，不继续讲 CMake、PATH、root。

## 【现在必须理解】我们当时究竟在确认什么？

你的 PC：

```text
Ubuntu x86_64
```

你的板子：

```text
RK3566 ARM64
```

普通 PC 编译器：

```text
gcc
```

默认生成：

```text
x86_64 程序
```

RK3566不能直接运行。

所以需要：

```text
aarch64-linux-gnu-gcc
```

它运行在：

```text
x86 PC
```

但生成：

```text
ARM64 Linux程序
```

这就是**交叉编译器**。

类比 MCU：

```text
Windows PC
↓
arm-none-eabi-gcc
↓
STM32程序
```

这里：

```text
Ubuntu PC
↓
aarch64-linux-gnu-gcc
↓
RK3566 Linux程序
```

类比非常接近。

区别是 RK3566 最终跑的是 Linux 用户态 ELF，不是 MCU 裸机 firmware。

---

## 第一条

```bash
which aarch64-linux-gnu-gcc
```

### 分类

**Linux 通用能力：`which`**

**开发工具：`aarch64-linux-gnu-gcc`**

### Shell 怎么读？

Shell 看到两个由空格分开的单词：

```text
which
aarch64-linux-gnu-gcc
```

这里：

```text
which
└─ 要执行的程序

aarch64-linux-gnu-gcc
└─ 给 which 的参数
```

整体意思：

> “帮我查一下，如果我现在输入 `aarch64-linux-gnu-gcc`，Shell 会找到哪个程序？”

结果：

```text
/usr/bin/aarch64-linux-gnu-gcc
```

所以你确定：

```text
① 系统找得到它
② 它位于 /usr/bin
```

注意：

`which` 并不是问：

> “电脑硬盘上有没有这个文件？”

它更接近：

> “按照当前命令搜索规则，我输入这个名字时会执行哪个？”

这也是以后排 PATH 问题的重要工具。

---

## 第二条

```bash
aarch64-linux-gnu-gcc --version
```

Shell 大致拆成：

```text
aarch64-linux-gnu-gcc
└─ 程序

--version
└─ 这个程序自己的选项
```

注意：

> `--version` 不是 Shell 固定语法。

是这个程序支持的选项。

很多程序碰巧都遵循这个惯例，所以：

```bash
gcc --version
cmake --version
python --version
```

经常成立。

但不是 Shell 强制规定的。

---

## 第三条

```bash
aarch64-linux-gnu-g++ -dumpmachine
```

拆成：

```text
aarch64-linux-gnu-g++
└─ C++ 交叉编译器

-dumpmachine
└─ GCC 自己定义的参数
```

输出：

```text
aarch64-linux-gnu
```

这个回答的是：

> “你这个编译器主要面向什么目标平台？”

所以阶段 1 最终得出：

```text
编译器存在 ✅
版本能运行 ✅
目标架构 = aarch64 ✅
```

于是：

> **可以继续进入“拿源码进行交叉编译”阶段。**

---

# 这一阶段你真正应该记住什么

不是背：

```bash
aarch64-linux-gnu-g++ -dumpmachine
```

真正该形成的是排查套路：

```text
我要用一个开发工具
↓
Shell 找不找得到？
→ which

找得到
↓
它到底是什么版本？
→ --version

它是不是我想要的目标工具链？
→ 工具自身提供的查询参数
```

所以以后你碰：

```text
cmake
python
gcc
git
```

都可以自然迁移。

例如：

```bash
which python3
python3 --version
```

---

# 给你一个非常短的小任务

现在先不要往下学第二阶段。

假设以后你同时装了两个 CMake，你怀疑终端现在调用的不是你想要的那个。

**你觉得第一条应该输入什么命令？**

只写命令，不用解释。

你答完后，我继续带你进入 **阶段 2：Git 下载源码 + 第一次真正学习“错误信息该怎么看”**。