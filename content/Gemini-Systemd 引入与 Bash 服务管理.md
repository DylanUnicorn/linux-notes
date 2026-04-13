# Systemd 引入与 Bash 服务管理

## Prompt:

> 阶段二：v0.5 纯内存极简文件系统 - MD

> 阶段三：v0.6 自动驱动加载与挂载硬盘 - MD

> 阶段一：initrd.img 学习与拆解 - MD

> ![Preview Image](blob:https://gemini.google.com/2826bff3-b373-4de6-a4f3-6908b88ea92c)

我之前在vmware进行了linix系统的简单实操（v0.6），现在需要继续往下走，见图片

## Response:

太棒了！从直接调用 BusyBox 的野生状态，跨越到由 `systemd` 这种现代系统管理器来接管进程，这是构建现代 Linux 系统的核心门槛。这种将服务抽象化、由 `systemd` 进行生命周期监管（如崩溃重启）的机制，正是开发高可靠性嵌入式系统（比如在 Jetson AGX Thor 这类高性能计算平台上部署机器人底层 OS）时必不可少的基础。

根据你的路线图，**v0.7 是在 v0.5（纯内存运行）的基础上进行的**。这意味着我们暂时先把 v0.6 的“挂载物理硬盘”放一边，在一个纯净的内存沙盒里专门攻克 `systemd` 的引入和配置。

`systemd` 和标准的 `/bin/bash` 都是**动态链接**的，它们极其依赖宿主机的 `.so` 动态库文件，这与 v0.5 纯静态的 BusyBox 完全不同。因此，v0.7 的核心难点在于**依赖库的迁移**和 **systemd 的精简配置**。

下面是为您定制的 **v0.7 实操执行方案**：

* * *

### 1\. 准备工作与自动化脚本

我们需要把标准版的 `/bin/bash` 和 `/lib/systemd/systemd` 以及它们依赖的上百个动态库复制到我们的文件系统中。为了避免手动排查 `ldd` 查到吐血，我们先写一个自动拷贝依赖的小工具。

```
# 基于 v0.5 复制一份全新的 v0.7 工作区
cd ~
cp -r my_rootfs_v0.5 my_rootfs_v0.7
cd my_rootfs_v0.7

# 创建存放脚本的临时目录（打包前可以删掉）
mkdir -p build_tools
```

在 `build_tools` 下创建一个名为 `cp_libs.sh` 的脚本：

```
cat << 'EOF' > build_tools/cp_libs.sh
#!/bin/bash
# 自动通过 ldd 查找并拷贝依赖库到目标根文件系统的脚本

TARGET_DIR=$(pwd)
BIN_PATH=$1

if [ -z "$BIN_PATH" ] || [ ! -f "$BIN_PATH" ]; then
    echo "Usage: $0 <absolute_path_to_binary>"
    exit 1
fi

echo "Copying dependencies for $BIN_PATH..."

# 提取依赖并通过循环复制
ldd "$BIN_PATH" | awk '{print $3}' | grep "^/" | while read lib; do
    # 获取库文件所在的目录并在目标系统创建
    lib_dir=$(dirname "$lib")
    mkdir -p "${TARGET_DIR}${lib_dir}"

    # 如果目标系统里还没有这个库，就拷贝过去
    if [ ! -f "${TARGET_DIR}${lib}" ]; then
        cp "$lib" "${TARGET_DIR}${lib}"
        echo "Copied: $lib"
    fi
done

# 单独处理直接链接的 ld-linux 解释器
ld_loader=$(ldd "$BIN_PATH" | grep -o '/lib.*\.[0-9]' | head -n 1)
if [ -n "$ld_loader" ] && [ ! -f "${TARGET_DIR}${ld_loader}" ]; then
    lib_dir=$(dirname "$ld_loader")
    mkdir -p "${TARGET_DIR}${lib_dir}"
    cp "$ld_loader" "${TARGET_DIR}${ld_loader}"
    echo "Copied loader: $ld_loader"
fi
EOF

chmod +x build_tools/cp_libs.sh
```

### 2\. 引入 bash 与 systemd 二进制文件

使用刚才的脚本，把这两个主角以及它们的库文件“请”进我们的 v0.7 系统。

```
# 1. 拷贝标准的 bash
cp /bin/bash ./bin/bash
./build_tools/cp_libs.sh /bin/bash

# 2. 拷贝 systemd 的核心执行文件
mkdir -p ./lib/systemd
cp /lib/systemd/systemd ./lib/systemd/systemd
./build_tools/cp_libs.sh /lib/systemd/systemd

# 因为从 Ubuntu 20.04 开始，/lib 往往是指向 /usr/lib 的软链接
# 我们需要确保根目录下有 lib64 目录（链接器通常需要）
mkdir -p lib64
cp -r /lib64/* ./lib64/ 2>/dev/null || true
```

### 3\. 编写 systemd 配置（核心任务）

根据路线图要求：“需要根据 systemd 配置规则写专门针对 bash 的配置（临时）”，并且实现“bash 退出会立刻重新创建新的 shell”。

我们需要建立 `systemd` 的配置目录骨架，并编写一个 `bash.service`。

```
mkdir -p etc/systemd/system

# 编写专门针对 bash 的服务配置文件
cat << 'EOF' > etc/systemd/system/bash.service
[Unit]
Description=Interactive Bash Shell managed by systemd
# 去除默认依赖，防止 systemd 去寻找网络、日志等我们还没建的基础设施
DefaultDependencies=no 

[Service]
Type=simple
ExecStart=/bin/bash
# 强制绑定到控制台
StandardInput=tty-force
StandardOutput=inherit
StandardError=inherit
TTYPath=/dev/console
# 核心需求：退出后立刻重启
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
EOF
```

接下来，告诉 `systemd` 系统启动后的默认目标（Default Target）就是我们的 `bash.service`：

```
# 建立 default.target 指向 bash.service 的软链接
ln -s /etc/systemd/system/bash.service etc/systemd/system/default.target
```

### 4\. 移交权杖：修改 /init

在 v0.5 中，我们是用一个 Shell 脚本作为 `/init`。现在，我们要让 Linux 内核在解压完内存文件系统后，直接把系统的控制权（PID 1）交给 `systemd`。

删除旧的 `init` 脚本，创建一个指向 `systemd` 的软链接：

```
# 移除之前的 init 脚本
rm init

# 让内核启动的第一个进程变为 systemd
ln -s lib/systemd/systemd init

# systemd 启动时需要知道主机名等基础信息，随意建几个空文件防报错
echo "v0.7-systemd" > etc/hostname
touch etc/os-release
```

### 5\. 打包与测试

现在整个结构已经具备了 `systemd` 的雏形，我们打包进行测试。

```
# 确保在 my_rootfs_v0.7 目录下
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7.img

# 复制到 boot 目录
sudo cp ../my_initrd_v0.7.img /boot/
```

**更新 GRUB：** 打开 `sudo nano /etc/grub.d/40_custom`，追加一段针对 v0.7 的启动项：

```
menuentry "Test My Minimal Rootfs v0.7 (systemd bash)" {
    linux /boot/vmlinuz-5.15.0-xxx-generic rw init=/init
    initrd /boot/my_initrd_v0.7.img
}
```

_(注意替换你的真实内核版本号，这里加上了 `rw` 权限以防 systemd 报只读文件系统的错)_

```
sudo update-grub
sudo reboot
```

### 🎯 成果验证指南

1. **启动观察**：在 GRUB 选择 v0.7 后，你应该能看到熟悉的 `systemd` 的绿色 `[ OK ]` 启动日志（即使只有很少的几行）。
2. **验证接管**：进入 `#` 提示符后，输入 `ps -ef` 或者直接运行 `echo $$`。你应该看到当前的 bash 的 PID 已经不是 1 了，PID 1 变成了 `/init`（即 systemd）。
3. **验证监管（拔插头测试）**：输入 `exit` 回车。
   * **预期现象**：Shell 界面会闪烁一下，然后立刻再次出现 `#` 提示符。这就是你在配置文件中设定的 `Restart=always` 发挥了作用，`systemd` 监测到 bash 进程死亡，瞬间重新拉起了一个新的 bash！

顺利完成这一步，你就真正理解了现代 Linux 发行版是如何保持关键服务永远在线的了。测试成功后，我们就可以往 v0.8 的 `login` 验证机制推进了。遇到任何动态库报错，随时把日志发给我。

## Prompt:

我不知到我的内核版本号是多少

## Response:

没关系，这步非常简单！既然你是在正常运行的 Ubuntu 虚拟机里进行准备工作，你可以通过以下两个简单的命令来找到你的内核版本：

**方法一：直接查看内核文件名（最推荐）** 因为你需要在 GRUB 里写明完整的文件路径，所以直接看一下 `/boot` 目录下有哪些内核文件是最直观的。在终端中输入：

```
ls /boot/vmlinuz-*
```

**输出示例**：`/boot/vmlinuz-5.15.0-100-generic` 你直接把终端里打印出来的这个完整路径，复制替换掉之前配置里的那段路径就可以了。

* * *

**方法二：查看当前正在运行的内核版本号** 如果你只想知道具体的版本号数字，可以输入：

```
uname -r
```

**输出示例**：`5.15.0-100-generic` 拿到这串数字后，你在 GRUB 里要写的完整路径就是 `/boot/vmlinuz-` 加上这串数字。

* * *

**实战套用：** 假设你通过上面命令查出来的结果是 `5.15.0-100-generic`，那么你在编辑 `/etc/grub.d/40_custom` 时，v0.7 的启动项就应该写成这样：

```
menuentry "Test My Minimal Rootfs v0.7 (systemd bash)" {
    linux /boot/vmlinuz-5.15.0-100-generic rw init=/init
    initrd /boot/my_initrd_v0.7.img
}
```

快去终端里敲一下 `ls /boot/vmlinuz-*` 吧！拿到文件名后，替换进去就可以继续愉快的 `sudo update-grub` 并重启测试我们的 systemd 环境啦！如果有其他卡住的地方随时告诉我。

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/4fb8380a-bdb4-4b6d-9fd8-668ef4e5648b)

糟糕，失败了

## Response:

别灰心！看到这个 `Kernel panic` 和 `error -2`，恭喜你，你已经正式触碰到了从“静态编译（v0.5 的 BusyBox）”跨越到“动态链接（v0.7 的 systemd/bash）”时**最经典、必经的踩坑点**！

我们来像外科医生一样解剖这个错误： 在 Linux 内核里，`error -2` 的代号叫 `ENOENT` (Error NO ENTry)，也就是\*\*“找不到文件或目录”\*\*。 内核解压了你的 `initrd.img`，试图去执行 `/init`，但是它抱怨找不到。

这通常有三种可能（按概率从大到小）：

1. **“翻译官”丢了（动态链接器陷阱）：** `systemd` 和 `bash` 都是动态链接的程序。即使 `/init` 和 `systemd` 文件本身存在，如果内核找不到它们在 ELF 文件头里指定的“动态加载器”（通常是 `/lib64/ld-linux-x86-64.so.2`），内核也会报 `error -2`。
2. **软链接断了：** `/init` 这个软链接指向了错误的路径（比如打包时路径错乱）。
3. **Ubuntu 的“软链接地狱”：** 现代 Ubuntu 中，`/lib` 其实是指向 `/usr/lib` 的软链接。如果我们盲目拷贝，可能会导致真实的库文件放错了位置，系统按图索骥找不到。

我们分两步来排查和修复：

### 第一步：不重启，直接“透视”你的 img 文件

我们先看看打包出来的镜像里到底缺了什么。在你的正常 Ubuntu 宿主机终端里运行这个命令：

```
lsinitramfs /boot/my_initrd_v0.7.img | grep -E "^init$|systemd/systemd$|ld-linux|libc.so"
```

**预期的正确输出应该包含这几样东西（路径必须对得上）：**

* `init` （你的入口软链接）
* `lib/systemd/systemd` （真正的可执行文件）
* `lib64/ld-linux-x86-64.so.2` （最核心的加载器，没它必报 error -2）
* `lib/x86_64-linux-gnu/libc.so.6` （或者类似的 C 标准库路径）

如果你发现 `ld-linux` 不在 `lib64` 下，或者 `systemd` 根本没拷进去，那就是我们的 `cp_libs.sh` 脚本或者之前的拷贝命令在你的系统环境下水土不服了。

* * *

### 第二步：变量控制法 —— 先让动态版的 bash 跑起来

`systemd` 太庞大了，依赖成百上千。为了确定到底是“库文件没拷对”，还是“systemd 配置没写对”，我们做一个极简的**控制变量测试**：**让内核跳过 systemd，直接运行动态版的 bash。**

在你的工作目录执行以下操作：

```
cd ~/my_rootfs_v0.7

# 1. 删掉指向 systemd 的软链接
rm init

# 2. 将 init 直接指向刚刚拷贝进来的动态版 bash
ln -s bin/bash init

# 3. 重新打包
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7_debug.img

# 4. 拷贝到 /boot
sudo cp ../my_initrd_v0.7_debug.img /boot/my_initrd_v0.7.img
```

然后 `sudo reboot`，在 GRUB 选 v0.7 再试一次。

**结果分析：**

* 👉 **情况 A：成功进入了 `bash` 的 `#` 提示符**
  * **结论：** 你的动态库拷贝脚本是**成功**的！`ld-linux` 和核心库都在正确的位置。
  * **下一步：** 那么 `error -2` 是 `systemd` 本身的问题（可能是它还需要其他不在 `ldd` 列表里的模块，或者 Ubuntu 的 `/lib` 结构让它迷失了）。
* 👉 **情况 B：依然无情地报 `Requested init /init failed (error -2)`**
  * **结论：** 破案了！就是你的动态加载环境（`lib64` 等）没建好。内核连最基本的 `bash` 都加载不起来。
  * **下一步：** 我们需要修复依赖拷贝脚本，或者手动处理 Ubuntu 的 `/lib` 软链接机制。

**请告诉我执行这两步后的结果（特别是透视 img 文件的输出，或者 bash 测试的结果）**，我们马上就能精准定位并攻克它！嵌入式底层就是这样，跨过这个依赖库的门槛，后面就豁然开朗了。

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/12282bc5-70ec-4a66-b622-ab90ebd5a847)

lsinitramfs /boot/my\_initrd\_v0.7.img | grep -E "^init\$|systemd/systemd\$|ld-linux|libc.so"

init

lib64/ld-linux-x86-64.so.2

lib/x86\_64-linux-gnu/libc.so.6

lib/systemd/systemd

还是有的

然后运行你的步骤二得到这样的结果

## Response:

太棒了！你的 `lsinitramfs` 结果证明了**文件确实被成功打包进去了**。

你现在遇到了操作系统底层开发中最经典的\*\*“薛定谔的文件”问题\*\*：你在外面看，文件明明都在（`ld-linux` 和 `libc` 都有）；但内核在里面执行时，却无情地报 `error -2` (文件或目录不存在)。

既然核心加载器在，内核依然报找不到文件，通常有以下几种极为隐蔽的原因：

1. **软链接断裂（幽灵文件）：** `cp` 命令可能只拷贝了软链接本身，而没有拷贝它指向的真实文件。比如 `/bin/bash` 在某些 Ubuntu 上其实是指向 `/usr/bin/bash` 的软链接，如果你没拷 `/usr/bin/bash`，那 `bash` 就是个死链接。
2. **权限丢失：** 动态加载器（`ld-linux`）虽然在，但如果失去了执行权限（`+x`），内核也会报 `error -2`。
3. **架构隐性依赖：** 某些库隐藏在 `/lib/x86_64-linux-gnu/` 深处，没有被 `ldd` 完全抓取。

不要盲目猜了！我们现在使用高级底层开发中最有效的手段：**“特洛伊木马”调试法。**

### 核心思路：让 BusyBox 带我们进去看现场

既然 v0.5 的静态 `BusyBox` 是绝对不会出错的，我们就**用 BusyBox 作为 `/init` 成功启动系统**，进入纯内存 Shell 后，在现场**手动执行** `/bin/bash`。这样，底层的报错信息就会清清楚楚地打印在屏幕上，而不是内核那句冰冷的 `error -2`。

* * *

### 调试行动步骤

#### 1\. 恢复 BusyBox 作为启动入口

在你的 `~/my_rootfs_v0.7` 工作目录下，把刚才的 `init` 删掉，换成一个基础的调试脚本：

```
cd ~/my_rootfs_v0.7
rm init

# 创建一个专用于调试的 init 脚本 (依赖 v0.5 遗留下来的 busybox)
cat << 'EOF' > init
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

echo "====================================="
echo "  [ V0.7 DEBUG MODE ] "
echo "  Kernel booted via static BusyBox."
echo "  Time to hunt down the dynamic bug!"
echo "====================================="

# 启动 BusyBox 的基础 shell
exec /bin/sh < /dev/console > /dev/console 2>&1
EOF

chmod 777 init
```

#### 2\. 重新打包并替换镜像

```
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7_debug.img
sudo cp ../my_initrd_v0.7_debug.img /boot/my_initrd_v0.7.img
sudo update-grub
sudo reboot
```

#### 3\. 现场审讯（关键！）

这次重启后，你应该能成功看到 `[ V0.7 DEBUG MODE ]` 的欢迎语，并拿到一个 `#` 提示符（这是 BusyBox 提供的）。

现在，请你在虚拟机这个 `#` 提示符下，依次输入以下命令，**看看它们到底报什么错**：

**测试一：查验 bash 真身**

```
ls -l /bin/bash
```

_(留意它到底是个文件，还是一个指向 `/usr/bin/bash` 等不存在路径的软链接 `->`)_

**测试二：查验加载器真身**

```
ls -l /lib64/ld-linux-x86-64.so.2
```

_(留意它是否有 `x` 执行权限)_

**测试三：直接强行运行 bash**

```
/bin/bash
```

**请告诉我执行这三个命令后，屏幕上具体输出了什么！** 特别是执行 `/bin/bash` 时，如果它报了 `not found` 或者 `missing library xxx`，那就是抓到真凶了。拿到线索我们就能一针见血地修复它！

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/4f07adfd-90b8-486c-97c0-136b3042265f)

## Response:

太漂亮了！这张截图简直是教科书级别的底层调试现场。你亲手抓住了这个导致内核崩溃的“幽灵”！

仔细看你的第二条命令 `ls -l /lib64/ld-linux-x86-64.so.2` 的输出： 注意看最后半段：`-> /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2` 是**红色**的（如果你在宿主机上看，或者终端有颜色高亮的话，通常是红色或闪烁的）。

在 Linux 的 `ls -l` 语法中，这代表\*\*“这是一个死链接（Broken Symlink）”\*\*。

**破案了：** 虽然你的 `/bin/bash` 完好无损地躺在那里（1.3MB，有执行权限），但当内核试图运行它时，会去读取它的 ELF 头文件，发现它需要 `/lib64/ld-linux-x86-64.so.2` 这个动态加载器（Loader）来启动。 内核顺着路径找到了这个加载器，结果发现是个软链接，再顺着软链接指向的 `/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2` 找过去——**发现那里空空如也！** 于是内核只能无奈地抛出：`/bin/sh: /bin/bash: not found`（不是找不到 bash，是找不到启动 bash 需要的加载器）。

这往往是因为我们在拷贝库文件时，不小心把软链接“原样”拷过来了，却没有拷贝它真正指向的本体。

### 🛠️ 一针见血的修复方案

既然知道了凶手，修复起来非常简单。我们不需要复杂的软链接寻址，**直接用本体替换掉这个死链接**。

回到你的 **Ubuntu 宿主机（正常桌面系统里）**，打开终端，执行以下命令：

```
cd ~/my_rootfs_v0.7

# 使用 cp -L 参数（大写L代表 dereference），强制跟踪软链接，把真正的实体文件拷过来！
cp -L /lib64/ld-linux-x86-64.so.2 ./lib64/ld-linux-x86-64.so.2

# 验证一下，现在它应该是一个实打实的文件，不再带有 "->" 箭头了
ls -l ./lib64/ld-linux-x86-64.so.2
```

确认它不再是软链接后，我们直接重新打包并测试这个“动态版 bash”：

```
# 重新打包
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7_debug.img
sudo cp ../my_initrd_v0.7_debug.img /boot/my_initrd_v0.7.img

# 重启进入虚拟机测试
sudo reboot
```

再次进入 `[ V0.7 DEBUG MODE ]` 后，在 `#` 提示符下敲下：

```
/bin/bash
```

如果没有任何报错，并且你的提示符可能发生了微小的变化（比如变成了 `bash-5.1#`），那就意味着**动态链接库这座大山被你彻底翻过去了！** 你可以在里面输入 `exit` 退回到 BusyBox。

只要这个 bash 测试成功，你就可以放心地执行 `rm init` 并重新 `ln -s lib/systemd/systemd init`，让 systemd 正式接管系统了（因为 systemd 依赖的也是这套动态链接环境）。去试试看！

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/31abd3e8-063f-45b8-aabf-e8fd4ebbae95)

糟糕，似乎没这么简单

## Response:

哈，欢迎来到 Linux 底层开发最真实的“受苦”环节！这叫“拔出萝卜带出泥”。

这个看似毫不讲理的 `not found` 其实在暗示我们：**主加载器（ld-linux）可能确实在了，但它在试图把 bash 拉起来的瞬间，发现缺少了其他的依赖库，于是它直接“罢工”并向上级（内核）报了 not found。**

在纯正的底层排错中，我们不能靠猜。既然我们现在已经在这个 Debug 终端里了，我教你一个**终极杀手锏**——**绕过系统自动调用，我们直接手执“加载器”来强行“审问” bash！**

请在你现在的这个 `#` 提示符下，输入以下命令：

```
# 终极审问：让加载器直接列出 bash 缺少什么库
/lib64/ld-linux-x86-64.so.2 --list /bin/bash
```

或者，直接用加载器去运行 bash：

```
/lib64/ld-linux-x86-64.so.2 /bin/bash
```

**可能出现的两种情况（请告诉我你看到了哪种）：**

1. **情况 A（吐出具体名字）**：它终于不报 `not found` 了，而是报出类似 `error while loading shared libraries: libtinfo.so.6: cannot open shared object file`。 👉 _这就太棒了！它供出了到底缺哪个小弟，我们回宿主机把它揪过来就行。_
2. **情况 B（连加载器都跑不起来）**：依然固执地报 `/lib64/ld-linux-x86-64.so.2: not found` 或者 `Permission denied`。 👉 _这就说明刚才的 `cp -L` 可能没生效，或者丢了执行权限（我们需要在宿主机 `chmod +x` 它）。你可以顺手再敲个 `ls -l /lib64/ld-linux-x86-64.so.2` 看看它现在的真身是不是依然带有红色的 `->`。_

把你敲完这两个命令后的画面发给我，这只“幽灵”这次绝对无处遁形！

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/8fd7beae-4554-461e-af09-16d7d1241b27)

## Response:

哈哈，典型的“情况 B”出现了！看到这个连加载器自己都报 `not found` 的错误，我们就能百分百确定：**刚才的 `cp -L` 并没有真正把实体文件覆盖过去，系统里那个 `/lib64/ld-linux-x86-64.so.2` 依然是个坏掉的空壳软链接！**（有时候 `cp` 命令如果遇到目标位置已经有一个同名软链接，它的行为会比较诡异，可能没有真正替换掉）。

而且在 Linux 中，动态加载器（ld-linux）本身必须是一个**具备可执行权限（`+x`）的实体二进制文件**。如果它是坏链接，或者没执行权限，内核就会翻脸不认人。

别跟它绕圈子了，我们这次不用优雅的软链接解析，直接用\*\*“暴力硬拷贝”\*\*法则，把这个幽灵揪出来捏碎！

请回到你的 **Ubuntu 宿主机（正常系统）**，打开终端，一步步执行这套“必杀连招”：

### 1\. 物理铲除坏链接，硬核注入真身

我们要直接去 Ubuntu 藏匿这个加载器真身的地方，把它原封不动地复制两份，分别放到你的文件系统中，并强行赋予最高执行权限。

```
cd ~/my_rootfs_v0.7

# 第一步：把之前残留的幽灵文件和软链接彻底删干净！
rm -f ./lib64/ld-linux-x86-64.so.2
rm -f ./lib/x86_64-linux-gnu/ld-linux-x86-64.so.2

# 确保目录结构都在
mkdir -p ./lib64
mkdir -p ./lib/x86_64-linux-gnu

# 第二步：直接从宿主机的绝对真身路径，硬拷贝到你的文件系统中（不搞软链接了，直接放真文件！）
cp /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ./lib64/
cp /lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 ./lib/x86_64-linux-gnu/

# 第三步：赐予它们神圣的执行权限
chmod +x ./lib64/ld-linux-x86-64.so.2
chmod +x ./lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
```

### 2\. 宿主机上的最后一次核验（非常重要）

在打包之前，我们在宿主机上查验一下你的战果，确保这次真的没问题了：

```
ls -l ./lib64/ld-linux-x86-64.so.2
```

**期望结果：** 它必须显示是一个普通文件（比如 `-rwxr-xr-x`），而不是以 `l` 开头的软链接，并且绝不能有 `->` 箭头！

### 3\. 重新打包并审问

如果上面核验通过了，那就进入熟悉的流程：

```
# 重新打包
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7_debug.img
sudo cp ../my_initrd_v0.7_debug.img /boot/my_initrd_v0.7.img

# 重启
sudo reboot
```

再次进入 `[ V0.7 DEBUG MODE ]` 的 `#` 提示符后，再次执行这句终极审问：

```
/lib64/ld-linux-x86-64.so.2 --list /bin/bash
```

这一次，加载器绝对跑得起来了！

* 如果它打印出了一大串库文件列表（比如 `libc.so.6 => /lib/...`），说明加载器成功了！你接着就可以直接敲 `/bin/bash` 看看能不能进去了。
* 如果它打印出了某个具体的库名并带有 `not found`（情况 A），请告诉我缺哪个库，我们马上补齐。

胜利就在眼前，底层系统的门就是这样被一脚一脚踹开的，继续！

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/94f6773a-df3c-455d-9a67-fc166a07c2f5)

> ![Preview Image](blob:https://gemini.google.com/b056bd31-2c6b-400f-a451-a7f6eb474784)

这个是什么情况

## Response:

太棒了！请允许我为你鼓掌——**你成功了！** 看到 `bash-5.1#` 这个提示符，意味着你已经彻底跨越了“动态链接”这座大山。Linux 内核成功找到了动态加载器，加载器成功找到了 `libc` 等依赖库，并完美地把 `bash` 运行了起来。

至于你截图里那两行看起来吓人的报错：

```
bash: cannot set terminal process group (-1): Inappropriate ioctl for device
bash: no job control in this shell
```

**完全不用担心，这是绝对正常的现象！**

**大白话解释：** `bash` 是一个高级的交互式 Shell，它一启动就习惯性地想要接管整个终端（TTY）的“控制权”（也就是 Job Control，用来支持 `Ctrl+Z` 挂起任务、后台运行 `&` 等高级功能）。 但是，你现在是在 `BusyBox` 的基础 Shell 里面**嵌套**运行了 `bash`。当前的控制台 `/dev/console` 的“老大”还是外面那层 `BusyBox`。`bash` 发现自己当不了老大，拿不到终端的完整控制权，所以就抱怨了两句：“我没法设置终端进程组哦，在这里我提供不了任务控制功能”。

这只是个警告，完全不影响 `bash` 执行核心命令。你现在已经在纯动态库环境里了！

* * *

### 🚀 决战 v0.7：让 systemd 正式登基

既然底层的动态运行环境（`ld-linux` 和 `libc`）已经打通，我们现在可以放心地把系统的最高权力（PID 1）从 `BusyBox` 交接给现代 Linux 的大总管——`systemd` 了。

请退回到你的 Ubuntu 宿主机，我们执行最后的移交手续：

**1\. 销毁 Debug 脚本，指定 systemd 为 Init**

```
cd ~/my_rootfs_v0.7

# 删掉刚才用来调试的 busybox 脚本
rm init

# 建立极其神圣的软链接：让内核启动后第一个执行 systemd
ln -s lib/systemd/systemd init
```

**2\. 确保之前的 bash 服务配置还在** 我们之前写过一个 `/etc/systemd/system/bash.service`，确保它在你的文件系统里（如果还在就不用管）： 如果你之前没写或者忘了，这里快速补上：

```
mkdir -p etc/systemd/system
cat << 'EOF' > etc/systemd/system/bash.service
[Unit]
Description=Interactive Bash Shell managed by systemd
DefaultDependencies=no 

[Service]
Type=simple
ExecStart=/bin/bash
StandardInput=tty-force
StandardOutput=inherit
StandardError=inherit
TTYPath=/dev/console
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
EOF

# 建立默认启动目标
ln -sf /etc/systemd/system/bash.service etc/systemd/system/default.target
```

**3\. 打包并见证奇迹**

```
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7.img
sudo cp ../my_initrd_v0.7.img /boot/
sudo update-grub
sudo reboot
```

### 🎯 这次重启你会看到什么？

在 GRUB 选择 v0.7 后：

1. 内核滚完日志后，你应该会看到 `systemd` 特有的启动信息（比如绿色的 `[ OK ]` 或者 `Welcome to systemd...`）。
2. 随后，屏幕上会直接跳出 `bash-5.1#` 的提示符。
3. **终极测试（拔插头测试）**：在这个提示符下输入 `exit` 回车。
   * 如果是以前的 v0.5，输入 exit 系统就死机了（Kernel Panic）。
   * 但现在，由于 `systemd` 在后台做监工（配置了 `Restart=always`），你会发现屏幕闪一下，`bash` **瞬间复活**，再次给你一个 `#` 提示符！

如果成功实现了“退出后秒复活”，你的 v0.7 路线图就完美通关了！去试试吧，期待你的好消息！

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/124732d6-4e4f-4022-b3b3-e8380d6e8c62)

糟糕，好像出了点问题

## Response:

这绝对是**史诗级的胜利**！别看满屏的字，这张图里藏着一个天大的好消息：

你仔细看日志的第三行：`systemd[1]: systemd 249.11-0ubuntu3.16 running in system mode...` **这意味着：Systemd 已经完美复活，成功作为系统的 1 号进程（PID 1）跑起来了！** 你的动态链接库（`ld-linux` 和 `libc`）完全正常运行了，内核再也没有崩溃（没有 Kernel Panic）。

你现在遇到的，只是纯粹的 **systemd 语法配置问题**。

### 🔍 抓出错误真凶

仔细看红色报错上面的那一行： `symlink target "bash.service" has incompatible suffix, rejecting.`

**大白话解释：** Systemd 是个规矩极其森严的管家。它在启动时会去寻找 `default.target`（默认目标）。我们之前偷了个懒，直接把 `default.target` 软链接到了 `bash.service`。 结果管家勃然大怒：“`.target`（目标）类型的软链接，必须指向另一个 `.target` 结尾的文件！你居然让它指向一个 `.service`（服务），后缀不匹配，我拒绝执行！” 找不到目标，管家就直接 `Freezing execution`（罢工冻结）了。

### 🛠️ 终极修复：满足管家的强迫症

既然它规矩严，我们就按标准的 systemd 规范给它写一个真正的 `default.target` 文件，而不是用软链接糊弄它。

请回到宿主机的 `~/my_rootfs_v0.7` 目录下，执行以下 3 步：

**第一步：删掉那个惹管家生气的软链接**

```
cd ~/my_rootfs_v0.7
rm etc/systemd/system/default.target
```

**第二步：创建一个真正的 `default.target` 配置文件** 我们直接写一个目标文件，告诉它：“这个目标唯一的要求，就是把 `bash.service` 拉起来。”

```
cat << 'EOF' > etc/systemd/system/default.target
[Unit]
Description=My Minimal Default Target
# 告诉 systemd，达到这个目标必须启动 bash.service
Requires=bash.service
# 同样去除默认依赖，防止它去瞎找网络和日志服务
DefaultDependencies=no
AllowIsolate=yes
EOF
```

_(注意：你之前的 `etc/systemd/system/bash.service` 文件不用动，保留原样即可。)_

**第三步：重新打包见证奇迹**

```
# 重新打包
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.7.img
sudo cp ../my_initrd_v0.7.img /boot/

# 重启虚拟机
sudo reboot
```

### 🎯 预期结果

这一次，systemd 读取 `default.target` 时不会再报错。它会看到里面写着 `Requires=bash.service`，然后乖乖去启动你的 bash。

屏幕一闪，那个承载着所有底层努力的 `bash-5.1#` 提示符就会出现在你面前。 进去之后，记得执行神圣的拔插头测试（输入 `exit` 回车），享受 systemd 瞬间帮你把 bash 重新拉起的快感！只差这最后半步了，冲！

## Prompt:

成功了，我们可以继续往下走了

## Response:

太棒了！热烈祝贺你完美通关 v0.7！ 看着系统自己把 `bash` 拉起来，并且拥有了“不死之身”（崩溃自动重启），这种掌控底层的感觉是非常让人上瘾的。

既然 `systemd` 已经稳稳地坐在了 PID 1 的宝座上，并且能够完美调度动态链接的程序，我们马上向 **v0.8（验证 login 登录机制）** 进军！

* * *

### 🚀 v0.8 核心原理解析

在 v0.7 中，我们的 `systemd` 是“简单粗暴”地直接启动了 `/bin/bash`，所以你一开机就拿到了 `root` 权限。但这在真实的操作系统中是不安全的。

标准的 Linux 启动流程应该是： **`systemd` → 启动终端守护进程 (`getty`) / `login` 程序 → 提示输入用户名/密码 → 验证通过 → `login` 程序为你生成对应的 `bash`。**

#### ⚠️ 避坑预警：令人胆寒的 PAM (可插拔认证模块)

如果你去翻看 Ubuntu 宿主机里的 `/bin/login`，用 `ldd` 看一下，你会发现它依赖了 `libpam.so`。PAM 是一个极其庞大且复杂的认证框架，它需要 `/etc/pam.d/` 下的一大堆配置文件和 `/lib/x86_64-linux-gnu/security/` 下的一堆动态库。 如果把原生的 `/bin/login` 搬进我们的极简系统，排查依赖会让你怀疑人生。

**我们的破局策略：** 借用 v0.5 留下来的“瑞士军刀” **BusyBox**！BusyBox 内置了一个极简版的 `login` 命令，它完全不依赖 PAM，只认最原始的 `/etc/passwd` 和 `/etc/shadow` 文件。这对于我们在沙盒里验证登录机制来说，简直是完美的平替！

* * *

### 🛠️ v0.8 实操步骤

请在宿主机的 `~/my_rootfs_v0.7` 目录中继续操作（你可以把它重命名为 v0.8，或者直接在原目录改）：

#### 1\. 创建身份验证三剑客（passwd, group, shadow）

`login` 程序需要通过这三个文件来核实你是谁。我们要手动为系统创建 `root` 账户。

```
cd ~/my_rootfs_v0.7

# 1. 配置 /etc/passwd (指定 root 的家目录和默认 Shell 是动态版的 bash)
echo "root:x:0:0:root:/root:/bin/bash" > etc/passwd

# 2. 配置 /etc/group
echo "root:x:0:" > etc/group

# 3. 配置 /etc/shadow (最核心的密码文件)
# 这里第二个字段是空的 (root::)，意味着**不需要密码即可登录**，方便我们初步测试。
echo "root::19000:0:99999:7:::" > etc/shadow

# 权限控制：shadow 文件极其敏感，不能让普通用户读
chmod 600 etc/shadow

# 创建 root 用户的家目录
mkdir -p root
```

#### 2\. 启用 BusyBox 的 login 功能

我们要把 `/bin/login` 指向 BusyBox，让它来接管登录验证。

```
# 建立软链接，指向我们 v0.5 拷进来的静态 busybox
ln -sf busybox ./bin/login
```

#### 3\. 编写 systemd 的登录服务配置

我们要废弃之前的 `bash.service`，写一个新的 `console-login.service`。

```
cat << 'EOF' > etc/systemd/system/console-login.service
[Unit]
Description=Console Login Prompt
DefaultDependencies=no
After=systemd-user-sessions.service

[Service]
Type=simple
# 核心改变：不再直接启动 bash，而是启动 login
ExecStart=-/bin/login
StandardInput=tty-force
StandardOutput=inherit
StandardError=inherit
TTYPath=/dev/console
Restart=always
RestartSec=1

# 告诉 systemd 忽略 login 退出时的某些状态码
SuccessExitStatus=0 1

[Install]
WantedBy=default.target
EOF
```

_(注意：`ExecStart=-/bin/login` 前面的 `-` 号是 systemd 的语法，意思是如果这个命令非正常退出，不要报错，直接重启它即可。)_

#### 4\. 重新指定 systemd 的默认目标

告诉管家，现在的终极目标变成了 `console-login.service`。

```
# 删掉 v0.7 创建的旧 default.target
rm etc/systemd/system/default.target

# 创建新的 default.target 配置
cat << 'EOF' > etc/systemd/system/default.target
[Unit]
Description=Login Target
Requires=console-login.service
DefaultDependencies=no
AllowIsolate=yes
EOF
```

* * *

### 📦 打包与测试验证

```
# 打包生成 v0.8 镜像
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.8.img
sudo cp ../my_initrd_v0.8.img /boot/
```

**更新 GRUB：** 修改 `/etc/grub.d/40_custom`，加一个 v0.8 的启动项（如果你嫌麻烦，直接把 v0.7 里的 `.img` 名字改成 0.8 也行）：

```
menuentry "Test My Minimal Rootfs v0.8 (login prompt)" {
    linux /boot/vmlinuz-5.15.0-xxx-generic rw init=/init
    initrd /boot/my_initrd_v0.8.img
}
```

```
sudo update-grub
sudo reboot
```

### 🎯 期待看到的画面

这一次重启后：

1. 你**不会**直接看到 `#` 提示符了。
2. 屏幕上会出现极其经典的 Linux 登录界面：
   
   ```
   v0.7-systemd login: 
   ```
3. 在这里输入 `root` 并回车。因为我们在 `shadow` 里设置了空密码，它验证通过后，会立刻把你送进动态版 `/bin/bash`，你才会看到 `bash-5.1#`！
4. **验证机制**：如果你输入了错的用户名（比如 `admin`），它会提示 `Login incorrect` 并让你重新输入。如果你进入 bash 后输入 `exit`，它不会死机，而是会**重新回到 `login:` 登录界面**！

去见证属于你的第一个带有账户安全体系的微型 OS 吧！如果遇到任何报错卡住，老规矩，截图发给我。

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/bbb68d08-e696-4509-a298-73b86b7211ff)

开什么玩笑，这个我都没有输入自己就进来了，而且我的输入似乎被吞了，非常的卡，疑似在抢占进程

## Response:

哈哈，你的直觉太敏锐了！你说得完全正确，这就是**典型的“进程抢占控制台（Race Condition）”**！你甚至凭借肉眼就看出了底层系统的调度冲突。

仔细看你截图里的这两行日志：

1. `[ OK ] Started Interactive Bash Shell managed by systemd.` (这是 v0.7 留下的 bash 服务)
2. `[ OK ] Started Console Login Prompt.` (这是 v0.8 新加的 login 服务)

**到底发生了什么？** 系统同时把这两个服务都拉起来了！`bash` 和 `login` 现在就像两个抢方向盘的司机，都在死死盯着 `/dev/console`（你的键盘输入）。 当你输入 `root` 的时候，后台的 `bash` 眼疾手快把输入抢走了，所以它报了 `bash: root: command not found`。而那个可怜的 `login` 进程根本没收到你的按键。

这怪我上一步没让你“斩草除根”。为了彻底解决这个精神分裂的终端，我们只需要**把旧的 `bash.service` 彻底删掉**。

### 🛠️ 一键终结进程战争

回到你的 Ubuntu 宿主机，执行以下命令清理战场：

```
cd ~/my_rootfs_v0.7

# 1. 彻底干掉 v0.7 的直连 bash 服务，拔除隐患
rm etc/systemd/system/bash.service

# 2. 确认你的 default.target 里只剩下了 login
cat etc/systemd/system/default.target
```

请确保终端打印出来的 `default.target` 内容长这样（尤其是 `Requires` 那一行只能有 login）：

```
[Unit]
Description=Login Target
Requires=console-login.service
DefaultDependencies=no
AllowIsolate=yes
```

### 📦 重新打包测试

确认清理干净后，重新打包：

```
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.8.img
sudo cp ../my_initrd_v0.8.img /boot/
sudo reboot
```

这一次重启，`bash` 绝对不会再出来捣乱了。系统会老老实实地停在 `v0.7-systemd login:` 的提示符前，安安静静地等待你输入 `root`。去试试看这次手感是不是完全丝滑了！

## Prompt:

丝滑了，请继续，把密码也加上来吧

## Response:

太棒了！“丝滑”是对底层打磨最好的赞美。看到纯手工打造的系统能够完美按照预期运转，成就感绝对是拉满的。

现在我们来补全这块安全拼图：**给 root 用户加上密码验证**。

### 🧠 核心原理解析：为什么不能直接在虚拟机里敲 `passwd`？

你可能会想：“我直接在现在的虚拟机里运行 `passwd root` 修改密码不就好了？” **千万别！** 别忘了我们现在运行的是 **纯内存文件系统 (initramfs)**。你在虚拟机里用 `passwd` 修改的密码，确实会立即生效，但是它仅仅存在于当时的内存中。一旦重启，系统又会重新从 `initrd.img` 里解压出那个空密码的 `shadow` 文件，一切又白干了。

所以，正确的做法是：**在宿主机（Ubuntu）上把密码提前加密好，直接“烧录”进我们的 `shadow` 文件里，然后再打包。**

* * *

### 🛠️ 实操步骤：生成并烧录密码

请回到你的 **Ubuntu 宿主机终端**，在 `~/my_rootfs_v0.7` 目录下执行以下操作：

#### 1\. 使用 openssl 生成标准的 SHA-512 密码哈希

Linux 的 `/etc/shadow` 文件从不存储明文密码。我们需要用宿主机的工具，把你想设置的密码（这里以最简单的 `123456` 为例）转换成系统能认的哈希值。

```
cd ~/my_rootfs_v0.7

# 使用 openssl 生成 SHA-512 加密（-6 代表 SHA-512 算法）
# 如果你想用别的密码，把双引号里的 123456 换掉即可
HASH=$(openssl passwd -6 "123456")

# 确认一下生成的密文（应该是一长串以 $6$ 开头的字符）
echo $HASH
```

#### 2\. 将密文注入 shadow 文件

在 v0.8 中，我们写入的是 `root::19000...`（第二个字段为空）。现在我们要把刚才生成的 `$HASH` 填进去。

```
# 直接覆盖之前的 shadow 文件配置
echo "root:${HASH}:19000:0:99999:7:::" > etc/shadow

# 检查一下是否写入成功
cat etc/shadow
```

_预期输出类似：`root:$6$vSomeRandomSalt$VeryLongEncryptedString...:19000:0:99999:7:::`_

#### 3\. 重新打包并见证

```
# 重新打包
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.8.img
sudo cp ../my_initrd_v0.8.img /boot/

# 重启虚拟机
sudo reboot
```

* * *

### 🎯 最终验证

重启后，系统会再次停在 `v0.7-systemd login:` 界面。

1. 输入 `root` 回车。
2. 此时，它会要求你输入密码：`Password:`。
3. 输入你刚才设置的密码（比如 `123456`，注意 Linux 下输入密码是没有任何星号显示的，盲敲即可），回车。
4. 验证通过，成功进入 `bash-5.1#`！

如果你故意输错密码，它会毫不留情地提示 `Login incorrect` 并让你重新登录。

**至此，你的 v0.8 路线（验证 login 登录机制）已经获得了完全体级别的成功！** 🎉

根据你的路线图，接下来就是 \*\*v0.9（构建完整的 systemd 运行环境）\*\*了。我们目前的 systemd 还是个“光杆司令”，很多默认的基础设施（比如挂载点、日志服务等）被我们强行砍掉了。 准备好迎接 v0.9 补全系统基础设施的挑战了吗？随时发令！

## Prompt:

准备好了，开始吧

## Response:

太痛快了！你的执行力堪称顶级。我们直接拔剑，向 **v0.9（构建完整的 systemd 运行环境）** 发起总攻！

### 🧠 v0.9 核心作战目标

在你上一张 v0.8 的截图里，有一行不太起眼的日志： `[ 6.685155] systemd[1]: System is tainted: var-run-bad`

现在的 systemd 虽然跑起来了，但它是个名副其实的“光杆司令”。

1. **基础设施缺失**：它抱怨 `var-run-bad`，因为标准的 Linux 必须要有 `/run` 这个内存文件系统供进程间通信（IPC）使用。
2. **耳目闭塞**：它没有 `journald`（日志守护进程），很多报错无处安放。
3. **缺少指挥棒**：我们在里面敲不了 `systemctl`，没法像在真实 Ubuntu 里那样管理服务。

**v0.9 的目标就是：消除 Taint（污染）警告，引入 `systemctl` 和日志系统，让它成为一个真正具有现代管理能力的 OS 底层！**

* * *

### 🛠️ 实操步骤：补全系统基础设施

请在宿主机的 `~/my_rootfs_v0.7`（或者你重命名后的目录）下执行以下连招：

#### 1\. 修复底层目录骨架 (消除 var-run-bad)

systemd 在极早期阶段会自动挂载 `/run` 和 `/tmp`，前提是这些文件夹必须存在。并且它强制要求 `/var/run` 必须是指向 `/run` 的软链接。

```
cd ~/my_rootfs_v0.7

# 创建系统运行时必须的基础目录
mkdir -p run tmp var/log

# 建立 systemd 强依赖的软链接（解决 var-run-bad 警告）
ln -snf ../run var/run
```

#### 2\. 请入 systemd 的“左膀右臂” (systemctl & journalctl)

没有 `systemctl` 的 systemd 是没有灵魂的。我们用之前写好的神器 `cp_libs.sh` 把它们和依赖库拉进来。

```
# 引入系统控制台 systemctl
cp /bin/systemctl ./bin/
./build_tools/cp_libs.sh /bin/systemctl

# 引入日志查询工具 journalctl
cp /bin/journalctl ./bin/
./build_tools/cp_libs.sh /bin/journalctl

# 引入日志记录后台程序 systemd-journald
cp /lib/systemd/systemd-journald ./lib/systemd/
./build_tools/cp_libs.sh /lib/systemd/systemd-journald
```

#### 3\. 编写日志服务配置

我们要让 systemd 在启动 `login` 之前，先把自己的日志系统拉起来。

```
# 创建 systemd-journald 的服务文件
cat << 'EOF' > etc/systemd/system/systemd-journald.service
[Unit]
Description=Journal Service
DefaultDependencies=no

[Service]
ExecStart=/lib/systemd/systemd-journald
Restart=always
EOF
```

#### 4\. 升级大管家的默认目标 (default.target)

告诉 systemd：“现在的目标不仅是启动登录界面，还要先把日志服务跑起来。”

```
# 覆盖之前的 default.target
cat << 'EOF' > etc/systemd/system/default.target
[Unit]
Description=Complete Systemd Target
# 核心变化：同时要求日志服务和登录服务
Requires=systemd-journald.service console-login.service
# 确保日志服务先于登录启动
After=systemd-journald.service
DefaultDependencies=no
AllowIsolate=yes
EOF
```

* * *

### 📦 打包与终极验收

```
# 重新打包成 v0.9 镜像
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v0.9.img
sudo cp ../my_initrd_v0.9.img /boot/
```

**更新 GRUB 启动项：** 在 `/etc/grub.d/40_custom` 中追加 v0.9 的入口：

```
menuentry "Test My Minimal Rootfs v0.9 (Complete systemd)" {
    linux /boot/vmlinuz-5.15.0-xxx-generic rw init=/init
    initrd /boot/my_initrd_v0.9.img
}
```

_(老规矩，别忘了替换你的内核版本号)_

```
sudo update-grub
sudo reboot
```

### 🎯 v0.9 通关见证指南

重启进入 v0.9 后，完成登录（输入 `root` 和你的密码），进入 `bash-5.1#` 提示符。 请在这个提示符下敲入现代 Linux 运维中最神圣的两个命令：

1. **测试系统管理能力：**
   
   ```
   systemctl status
   ```
   
   _预期效果：屏幕上会打印出一棵优美的进程树！你会清晰地看到 `systemd` 作为 PID 1，下面挂着 `systemd-journald` 和正在运行你的 `bash` 的 `login` 进程。_
2. **测试日志追溯能力：**
   
   ```
   journalctl -b
   ```
   
   _预期效果：你会看到从内核启动到 systemd 初始化所有的详细日志，`var-run-bad` 的警告也绝对消失了！_

如果你能成功打出这棵进程树，**你的 v0.9 就彻底大功告成了！** 整个极简系统的骨架已经完全进化为了现代化的架构。去享受敲下 `systemctl` 那一刻的丝滑吧，有任何截图或报错随时砸过来！

## Prompt:

成功了，我们现在需要进入v1.0吗

## Response:

太震撼了！能在纯内存的沙盒里，从零开始手动拉起完整的 `systemd`、日志系统和带密码验证的 `login` 进程，你已经完全看透了现代 Linux 操作系统的核心骨架！这就好比你亲手敲出了一个微型的 Ubuntu 引擎。

按照你最初的路线图，现在我们确实要向最终的 Boss 战进发了：**v1.0 —— 根据标准系统的启动过程，在小系统上复建一套**。

### 🚀 v1.0：终极“交接仪式” (The Grand Unification)

这几天的实操里，我们在 v0.6 学会了**挂载物理硬盘**，在 v0.7~v0.9 学会了**进程管家 systemd 的运转逻辑**。

什么是“标准系统的启动过程”？ 在真实的 Ubuntu 等现代 Linux 发行版中，我们辛辛苦苦构建的这个 `initrd.img`（小系统）其实**并不是终点，而是跳板**。它的终极使命是极其伟大的“自我牺牲”：

1. **侦察兵（类似 v0.6）**：小系统在纯内存中先启动，通过 `mdev` 或 `udev` 快速扫描电脑的物理硬件，加载阵列卡、SCSI 或 SATA 硬盘驱动。
2. **锁死目标**：找到真正的物理根目录（比如你之前实验里的 `/dev/sda3`）并把它挂载到一个临时目录（比如 `/mnt/root`）。
3. **权力交接（终极魔法）**：调用底层的 `switch_root` 命令，一脚踢开纯内存文件系统，把根目录 `/` 瞬间切换到物理硬盘上，并把系统的最高控制权（PID 1）移交给硬盘上那个几百 GB 庞大的真实 `systemd`！

在 v1.0 中，我们要把小系统真正变成一个合格的 `initramfs` 引导程序，让它完成驱动加载后，成功唤醒你硬盘上沉睡的那个原生 Ubuntu 桌面系统。

我们即将废弃掉 v0.9 里为了沙盒测试而写的那些 `default.target`，重头编写一个真正具备实战意义的 `/init` 引导脚本。

准备好挑战这个底层开发中最经典的“金蝉脱壳”机制，编写带有 `switch_root` 的终极引导脚本了吗？

## Prompt:

来吧

## Response:

太燃了！我们这就开启终极的 **v1.0 篇章：金蝉脱壳 (The Grand Unification)**。

在真正的现代 Linux 发行版中，`initramfs`（也就是我们的极简小系统）的终极宿命就是**自我毁灭**。它在纯内存中苏醒，以最快的速度摸清电脑硬件，加载硬盘驱动，找到那块沉睡着庞大 Ubuntu 系统的物理硬盘，将其唤醒，然后把最高指挥权（PID 1）连同整个根目录一起，毫无保留地移交给硬盘上的那个“真命天子”。

完成这个壮举的核心魔法，叫做 **`switch_root`**。

### 🛠️ v1.0 实操步骤：编写终极引导程序

为了完成交接，我们需要废弃 v0.9 中为了测试而让系统停留在内存里的设定，重新编写一个纯粹的 `/init` 引导脚本。

请在宿主机的 `~/my_rootfs_v0.7`（或者你的工作目录）下，执行以下操作：

#### 1\. 斩断旧的羁绊

我们要把之前指向小系统 `systemd` 的 `/init` 软链接删掉，换成一个强大的 Shell 脚本。

```
cd ~/my_rootfs_v0.7
rm init
```

#### 2\. 铸造终极 `/init` 剧本

创建一个全新的 `init` 文件，这段代码融合了我们 v0.5 到 v0.9 所有的心血，并加上了通往真实世界的最后一把钥匙。

```
cat << 'EOF' > init
#!/bin/sh

# 1. 基础环境初始化 (类似 v0.5)
export PATH=/sbin:/usr/sbin:/bin:/usr/bin
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

echo "================================================="
echo "  [ Initramfs v1.0 ] The Grand Unification"
echo "  Booting system and looking for real hardware..."
echo "================================================="

# 2. 硬件侦测与驱动加载 (我们在 v0.6 掌握的核心科技)
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
for alias_file in $(find /sys -name modalias); do
    modprobe $(cat $alias_file) 2>/dev/null
done

# 3. 锁定并挂载真实的物理根分区
# 我们在 v0.6 测出你的 Ubuntu 根目录挂载在 /dev/sda3
mkdir -p /mnt/root

echo "Waiting for physical disk /dev/sda3 to appear..."
# 循环等待，直到内核驱动成功识别出硬盘节点
while [ ! -b /dev/sda3 ]; do
    sleep 0.5
done

echo "Disk found! Mounting real root filesystem..."
mount /dev/sda3 /mnt/root

# 4. 权力交接的准备仪式：搬家
# 硬盘上的真实 systemd 有洁癖，我们需要把当前占用的虚拟文件系统(proc, sys, dev)
# 平滑地“平移”到真实的硬盘目录里，给真皇让位。
mount --move /sys /mnt/root/sys
mount --move /proc /mnt/root/proc
mount --move /dev /mnt/root/dev

# 5. 终极魔法：switch_root
# 语法：exec switch_root <新根目录> <要执行的真实 init>
echo "================================================="
echo "  Mission Accomplished! Handing over to REAL OS"
echo "  Executing switch_root..."
echo "================================================="

exec switch_root /mnt/root /sbin/init

# 如果程序能走到这里，说明 switch_root 失败了
echo "PANIC: switch_root failed!"
exec /bin/sh
EOF

# 赋予至高无上的执行权限
chmod 777 init
```

#### 3\. 打包并配置最后的 GRUB 入口

```
# 重新打包成 v1.0 镜像
find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../my_initrd_v1.0.img
sudo cp ../my_initrd_v1.0.img /boot/
```

编辑 `/etc/grub.d/40_custom`，加入我们这套系列的最终章：

```
menuentry "The Grand Finale v1.0 (Boot to Real Ubuntu)" {
    linux /boot/vmlinuz-5.15.0-xxx-generic rw init=/init
    initrd /boot/my_initrd_v1.0.img
}
```

_(请确保内核版本号替换正确)_

```
sudo update-grub
sudo reboot
```

* * *

### 🎯 见证奇迹的时刻

重启并选择 `The Grand Finale v1.0` 后：

1. 屏幕上会首先飞速闪过你亲手写的 `[ Initramfs v1.0 ]` 日志，它在后台疯狂地遍历 `modalias` 加载驱动。
2. 随后，它挂载了 `/dev/sda3`，并打印出 `Handing over to REAL OS`。
3. **高潮来了**：屏幕画面可能会闪烁，紧接着，你会看到熟悉的 Ubuntu 启动画面（或者直接跳出标准的 Ubuntu 桌面图形登录界面）。

这就意味着，你亲手打造的“引信”，成功引爆并唤醒了庞大的真实操作系统！

你准备好执行这段代码，完成这跨越真实物理世界的一跃了吗？

## Prompt:

> ![Preview Image](blob:https://gemini.google.com/6777cff0-02f6-4778-a9ca-56d0ef991ac7)

> ![Preview Image](blob:https://gemini.google.com/539953bb-9850-4a65-bf78-51ae4b0dd95b)

这个启动是启动了，但是时间为何如此之长，甚至还有红色的报错

## Response:

哈哈，恭喜你！虽然被满屏的报错和漫长的等待折磨了一番，但你必须知道一个事实：**你写的 `switch_root` 成功了！** 系统能走到这一步（开始加载 Network Manager），说明它已经完全脱离了你写的纯内存小系统，成功接管了物理硬盘上那个几百 GB 的真实 Ubuntu 操作系统！

至于为什么会卡这么久，满屏红字，这其实是**底层系统开发中极其经典的“硬件驱动交接后遗症”**，罪魁祸首并不是你的 v1.0 脚本写错了，而是 **VMware 的虚拟网卡遇到了水土不服**。

我们来像法医一样解剖你的截图：

### 🔍 抓出两个真凶

**1\. 第一凶手：`e1000 0000:02:01.0 ens33: Detected Tx Unit Hang`**

* **这是什么：** `e1000` 是 Intel 千兆网卡的驱动程序，在 VMware 里，它默认用这个老旧的驱动来“模拟”一张网卡（也就是你的 `ens33`）。
* **为什么报错：** `Tx Unit Hang` 意思是网卡的发送队列“死机”了。在真实的 Linux 启动中，网卡驱动是由标准 `initramfs` 里的原生程序平滑加载的。而我们的 v1.0 是用粗暴的 `mdev` 强行加载了驱动，随后又通过 `switch_root` 把控制权移交给了真实的 Ubuntu `udev`。这种暴力的“换心脏”手术，让 VMware 模拟的这块老网卡直接精神分裂，驱动卡死了。

**2\. 第二凶手（帮凶）：`A start job is running for Network Manager Wait Online`**

* **这是什么：** 这是真实 Ubuntu 里的一个系统服务（专门用来等网络连通的）。
* **为什么卡这么久：** 因为网卡驱动（第一凶手）已经卡死了，网络根本连不上。但是这个 `Wait Online` 服务是个死脑筋，它会在那里死等（截图里写着 `no limit`，有时候默认是等 90 秒甚至更久），直接阻塞了整个系统的启动进程，导致你感觉极其卡顿。

* * *

### 🛠️ 终极修复方案

要解决这个问题，我们有两种维度的手段。我强烈建议你采用 **“换掉虚拟网卡”** 的降维打击法。

#### 方案一：给虚拟机换上 VMware 原生网卡（最推荐，治本）

既然 `e1000` 模拟网卡在这种底层折腾中容易崩溃，我们就换成 VMware 专门为 Linux 定制的高性能半虚拟化网卡 **`vmxnet3`**，它非常稳定。

1. **彻底关闭这台 Ubuntu 虚拟机**（不要挂起，直接关机）。
2. 在你的物理机（Windows/宿主机）上，找到这台虚拟机的存放目录。
3. 找到那个以 **`.vmx`** 结尾的文件（这是虚拟机的核心配置文件），右键用“记事本”打开它。
4. 在里面按 `Ctrl+F` 搜索：`ethernet0.virtualDev`
5. 你会看到它原本写着：`ethernet0.virtualDev = "e1000"`
6. 把双引号里面的内容**改成 `"vmxnet3"`**。 _(修改后为：`ethernet0.virtualDev = "vmxnet3"`)_
7. 保存文件，关闭记事本。
8. 重新在 VMware 里开启虚拟机，选择 v1.0 启动项。网卡挂起的问题应该会瞬间消失！

#### 方案二：干掉死脑筋的等待服务（治标）

如果你终于等到了它慢吞吞地启动完毕，进入了真正的 Ubuntu 桌面或终端，你可以直接把那个拖慢启动速度的服务禁用掉。

打开真实 Ubuntu 的终端，执行：

```
sudo systemctl disable NetworkManager-wait-online.service
sudo systemctl mask NetworkManager-wait-online.service
```

这样以后再重启，即使网卡抽风，系统也不会再傻傻地等它了。

* * *

### 🏆 终局总结

当你解决掉这个网卡小插曲，看着虚拟机丝滑地从 GRUB 菜单 → 闪过你的 `[ Initramfs v1.0 ]` 日志 → 瞬间进入真实的 Ubuntu 桌面时，**你的这个操作系统底层课设就已经完美大结局了！**

回顾一下你走过的路：

* **v0.5**：在纯内存中利用 BusyBox 建立沙盒。
* **v0.6**：利用 mdev 扫描硬件，挂载物理硬盘。
* **v0.7~v0.9**：手动建立动态链接库，拉起 systemd 和 login 验证，补全日志设施。
* **v1.0**：金蝉脱壳，用 `switch_root` 唤醒真正的物理操作系统。

你已经把 Linux 启动过程中最隐秘、最底层的“黑匣子”彻底拆解并自己重造了一遍。这种硬核的底层 debug 经验，以后在搞任何嵌入式系统（比如你喜欢的机器人底层 OS 部署）时，都会成为你最强的武器。

修改完 `.vmx` 文件再去试一次吧！享受这最后畅快淋漓的开机！

---