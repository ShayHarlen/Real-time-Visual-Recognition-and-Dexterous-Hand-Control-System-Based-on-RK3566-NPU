# RK3566 SSH Key + sudo 免密笔记

你现在最需要建立的是一个**工程模型**，不是背命令。

当前必须理解的只有三件事：

1. `ssh`：让我远程进入另一台 Linux。
    
2. SSH Key：让我不用每次输入 SSH 登录密码。
    
3. `sudo`：我虽然以普通用户登录，但某条命令需要管理员权限时，可以临时以 `root` 身份执行。
    

至于 `chmod 440`、`visudo -f` 这些具体参数，**现在知道作用、留好笔记即可，不需要背。**

---

## 一、先把你刚才的几个命令彻底看懂

你现在登录进去看到：

```bash
lckfb@MiWiFi-RD03v2-srv:~$
```

说明当前身份是：

```text
用户：lckfb
权限：普通用户
```

Linux 里最高权限用户叫：

```text
root
```

类似 Windows 的“超级管理员”。

所以：

```bash
whoami
```

会得到：

```text
lckfb
```

而：

```bash
sudo whoami
```

相当于：

> “请以管理员身份执行 `whoami`。”

于是得到：

```text
root
```

### `sudo` 到底是什么？

可以先把：

```bash
sudo
```

理解成：

> **“这一条命令请临时使用管理员权限执行。”**

例如普通用户可能没权限：

```bash
apt install nginx
```

所以通常写：

```bash
sudo apt install nginx
```

又例如修改系统文件：

```bash
sudo vim /etc/xxx.conf
```

重启服务：

```bash
sudo systemctl restart ssh
```

查看某些受权限保护的信息：

```bash
sudo dmesg
```

所以以后你看到：

```bash
sudo xxx
```

先在脑子里翻译成：

> **“管理员权限执行 xxx。”**

这个现在必须理解。

---

# 二、为什么不直接登录 root？

你现在：

```text
SSH
 ↓
lckfb
 ↓
需要时 sudo
 ↓
root
```

比：

```text
SSH
 ↓
root
```

更合理。

因为平常：

```bash
rm xxx
```

只是普通用户权限。

只有你明确写：

```bash
sudo rm xxx
```

才提升权限。

这样等于多了一道保险。

不过我们为了让 Agent 自动操作，把：

```text
sudo 要输入密码
```

改成了：

```text
sudo 不需要输入密码
```

所以现在 Agent 可以：

```bash
ssh rk3566 "sudo systemctl restart ssh"
```

直接执行。

---

# 三、你刚才配置 sudo 的命令是什么意思？

你执行的是：

```bash
sudo visudo -f /etc/sudoers.d/lckfb-nopasswd
```

拆开看：

```text
sudo
│
└─ 管理员权限

visudo
│
└─ 专门安全编辑 sudo 配置文件的工具

-f
│
└─ 指定我要编辑哪个文件

/etc/sudoers.d/lckfb-nopasswd
│
└─ 我们创建的配置文件
```

为什么不直接：

```bash
sudo vim /etc/sudoers
```

？

因为 `sudoers` 文件一旦语法写错，可能导致：

```text
sudo 整个不能用了
```

而 `visudo` 会帮你检查语法。

所以：

> **修改 sudo 配置，优先使用 `visudo`。**

这个是值得记住的习惯。

---

你在里面写了：

```text
lckfb ALL=(ALL:ALL) NOPASSWD: ALL
```

先不用研究 sudoers 的所有语法。

现在只要知道它表示：

> 用户 `lckfb` 可以使用 `sudo` 执行所有命令，而且不要求输入密码。

拆一下：

```text
lckfb
│
└─ 哪个用户

NOPASSWD:
│
└─ sudo 不询问密码

ALL
│
└─ 所有命令
```

中间：

```text
ALL=(ALL:ALL)
```

属于 sudoers 更完整的权限描述。

**现在知道结论就够了，暂时不要继续钻 sudoers 语法。**

---

然后你执行：

```bash
sudo chmod 440 /etc/sudoers.d/lckfb-nopasswd
```

这里第一次碰到 Linux 很重要的命令：

```bash
chmod
```

它是：

> **change mode，修改文件权限。**

`440` 表示大致是：

```text
文件所有者：可读
文件所属组：可读
其他人：无权限
```

而且没有写权限。

为什么？

因为 `/etc/sudoers.d/` 里的配置是安全敏感文件，不应该谁都能乱改。

现在不用背：

```text
440
600
644
755
777
```

以后专门学 Linux 文件权限时再统一讲。

---

你又执行：

```bash
sudo -n whoami
```

这里：

```text
sudo
```

管理员执行。

```text
-n
```

表示：

> non-interactive，不允许询问密码。

所以：

```bash
sudo -n whoami
```

实际上是在测试：

> “如果我绝对不给你输入密码，你能不能以 root 身份执行？”

结果：

```text
root
```

说明：

> **NOPASSWD 配置成功。**

Agent 特别喜欢 `-n`，因为如果 sudo 需要密码，它不会一直卡在：

```text
[sudo] password for lckfb:
```

而是直接报错。

---

最后你 Windows 执行：

```powershell
ssh rk3566 "sudo -n id"
```

这个更重要。

意思是：

```text
Windows
  ↓
SSH 连接 rk3566
  ↓
在 RK3566 执行
sudo -n id
```

返回：

```text
用户id=0(root)
组id=0(root)
```

Linux 中：

```text
UID 0 = root
```

所以这证明：

> **你的 Windows / Agent 已经能够无人值守地获得 RK3566 的 root 权限。**

---

# 四、从生成 SSH Key 到 sudo 免密，完整笔记

你以后可以直接把这一段放到 Obsidian。

````markdown
# RK3566 SSH 无密码远程管理配置

## 目标

实现：

Windows / Agent
↓
ssh rk3566
↓
无需 SSH 密码
↓
登录 lckfb
↓
sudo 无需密码
↓
必要时获得 root 权限


## 1. RK3566 安装 SSH Server

```bash
sudo apt update
sudo apt install openssh-client openssh-sftp-server openssh-server
sudo systemctl enable --now ssh
````

检查：

```bash
systemctl status ssh
```

## 2. Windows SSH Config

文件：

C:\Users\a1500.ssh\config

配置：

```sshconfig
Host rk3566
    HostName 192.168.31.58
    User lckfb
    Port 22
    IdentityFile ~/.ssh/rk3566_ed25519
    IdentitiesOnly yes
```

以后：

```powershell
ssh rk3566
```

等价于：

```powershell
ssh lckfb@192.168.31.58
```

## 3. Windows 创建 SSH Key

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\rk3566_ed25519"
```

生成：

```text
rk3566_ed25519       私钥，只保存在 Windows
rk3566_ed25519.pub   公钥，可以放到服务器
```

## 4. 把公钥放到 RK3566

```powershell
Get-Content "$env:USERPROFILE\.ssh\rk3566_ed25519.pub" | ssh rk3566 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

公钥最终位于 RK3566：

```text
/home/lckfb/.ssh/authorized_keys
```

## 5. 测试 SSH Key

```powershell
ssh rk3566
```

如果不再要求输入密码，则成功。

测试非交互命令：

```powershell
ssh rk3566 "uname -a"
```

## 6. 配置 sudo 不需要密码

RK3566：

```bash
sudo visudo -f /etc/sudoers.d/lckfb-nopasswd
```

写入：

```text
lckfb ALL=(ALL:ALL) NOPASSWD: ALL
```

设置权限：

```bash
sudo chmod 440 /etc/sudoers.d/lckfb-nopasswd
```

## 7. 测试 sudo 免密

RK3566：

```bash
sudo -n whoami
```

输出：

```text
root
```

Windows：

```powershell
ssh rk3566 "sudo -n id"
```

出现：

```text
uid=0(root)
```

说明 Agent 可以远程获得 root 权限。

````

---

# 五、这些命令要不要背？

**不要现在背。**

Linux 正常的学习过程并不是：

```text
今天看到
↓
全部背下来
↓
以后永远记住
````

实际上工程师通常是：

```text
第一次：
完全不会
↓
搜索 / AI / 文档

第二次：
“这个我好像见过”

第五次：
“哦，应该是 chmod”

第十次：
直接顺手敲出来
```

我建议你建立三个层次。

### 第一层：必须记住概念

这些应该慢慢变成常识：

```text
ssh      远程登录
sudo     临时管理员权限
cd       切目录
ls       看文件
cp       复制
mv       移动/改名
rm       删除
mkdir    创建目录
cat      看文件内容
vim      编辑文件
```

### 第二层：认识，但不用背参数

例如：

```text
chmod
systemctl
ssh-keygen
visudo
scp
journalctl
```

看到知道大概干什么即可。

### 第三层：完全查

例如：

```bash
ssh-keygen -t ed25519 -f ...
```

这种长命令。

**没有必要背。**

以后：

```text
我想生成 SSH key
```

你知道：

```text
工具叫 ssh-keygen
```

就已经够了。

然后：

```bash
ssh-keygen --help
```

或者：

```bash
man ssh-keygen
```

或者直接问 AI。

---

# 六、别人到底是怎么学 Linux 命令的？

绝大多数不是先看完一本 Linux 命令大全。

而是：

```text
我要完成一个任务
      ↓
发现需要 SSH
      ↓
学 ssh
      ↓
发现要改文件
      ↓
学 vim
      ↓
发现权限不够
      ↓
学 sudo
      ↓
发现文件权限问题
      ↓
学 chmod
```

你现在这个过程其实正好。

例如你今天已经自然碰到了：

```text
SSH
SSH Key
~/.ssh
config
authorized_keys
sudo
root
chmod
visudo
systemctl
```

如果一周前直接给你讲这些，你大概率全忘。

现在因为每个东西都解决了一个实际问题，你会记得快得多。

---

# 七、Vim：现在只学够用的

这里属于**现在必须理解一个东西：Vim 有“模式”**。

这也是你问：

> 我要输入 `:` 怎么办？

的关键。

Vim 不是普通记事本。

最重要有三个模式：

```text
Normal Mode
普通模式
执行命令

Insert Mode
插入模式
输入文字

Command-line Mode
命令行模式
:w :q 等
```

刚进入 Vim：

```bash
vim test.txt
```

默认是：

```text
Normal Mode
```

这时你按：

```text
:
```

Vim 会认为：

> 我要输入 Vim 命令了。

底下出现：

```text
:
```

例如：

```vim
:w
```

保存。

```vim
:q
```

退出。

```vim
:wq
```

保存并退出。

```vim
:q!
```

不保存强制退出。

---

# 八、那我要在文章里真的输入 `:` 呢？

非常简单。

你先按：

```text
i
```

进入：

```text
Insert Mode
```

屏幕下面一般会出现：

```text
-- INSERT --
```

此时：

```text
:
```

就是普通字符。

你输入：

```text
hello: world
```

Vim 就会真的写进去：

```text
hello: world
```

不会执行 Vim 命令。

因为：

```text
Normal Mode 下 :
→ Vim 命令

Insert Mode 下 :
→ 普通字符
```

这就是模式的意义。

---

# 九、Vim 最小操作集

你现在只需要这几个。

进入：

```bash
vim test.txt
```

按：

```text
i
```

开始输入。

输入：

```text
hello: rk3566
```

按：

```text
Esc
```

退出插入模式。

然后：

```text
:w
```

保存。

或者：

```text
:wq
```

保存退出。

如果把文件改坏了，不想保存：

```text
Esc
:q!
```

直接跑路。

所以你现在可以把 Vim 简化成一个循环：

```text
vim 文件
   ↓
i
   ↓
输入文字
   ↓
Esc
   ↓
:wq
```

**先只用这套。**

`dd`、`yy`、`p`、宏、寄存器、visual mode、正则替换这些全部属于第二阶段，现在不要学。

---

## 你现在应该留下的工程模型

最终你只需要先记住：

```text
SSH Key
= 不输入 SSH 登录密码

sudo
= 临时使用管理员/root权限

sudo NOPASSWD
= 使用 sudo 时也不用输密码

vim
= 有模式
i → 输入
Esc → 回普通模式
:wq → 保存退出
```

你如果以后忘了：

```text
chmod 440 到底什么意思？
ssh-keygen 参数怎么写？
```

**完全正常，查笔记就行。**

工程师真正需要记住的不是所有命令，而是：

> **“我知道 Linux 有这个工具，它大概解决什么问题；需要的时候我能迅速把正确用法查出来。”**

这就是你当前阶段应该建立的能力。