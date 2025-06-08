# Win11系统AOSP环境搭建

## WSL 命令

使用单个命令安装运行 WSL 所需的一切内容。 在管理员模式下打开 PowerShell 或 Windows 命令提示符，输入 wsl --install 命令。

```powershell
wsl --install
```

还可以使用此命令通过运行 `wsl --install <Distribution Name>`来安装其他 Linux 分发版

```powershell
wsl --install Ubuntu-20.04
```

#### 设置默认 WSL 版本

```powershell
wsl --set-default-version 2
```

#### 列出可用的 Linux 分发版

```powershell
wsl -l -o
```

#### 列出已安装的 Linux 版本和状态

```powershell
wsl -l -v
```

#### 设置默认 Linux 分发版

```powershell
wsl --set-default <Distribution Name>
```

#### 检查 WSL 状态

```powershell
wsl --status
```

#### 检查 WSL 版本

```powershell
wsl -v
```

#### WSL 关机

```powershell
wsl --shutdown
```

立即终止所有正在运行的分发版和 WSL 2 轻型实用工具虚拟机。 在需要重启 WSL 2 虚拟机环境的实例中，可能需要此命令，例如 **更改内存使用限制** 或更改 .wslconfig 文件

#### WSL 终止

```powershell
wsl --terminate <Distribution Name>
wsl --terminate Ubuntu-20.04
```

#### 注销或卸载 Linux 分发版

```powershell
wsl --unregister <DistributionName>
wsl --unregister Ubuntu-20.04
```

将 `<DistributionName>` 替换为您目标的 Linux 发行版名称，会使该发行版从 WSL 中注销，以便您可以重新安装或清理它。 **谨慎：** 注销后，与该分发关联的所有数据、设置和软件都将永久丢失。 从应用商店重新安装将安装分发版的干净副本。 例如， `wsl --unregister Ubuntu` 将从 WSL 中可用的分发中删除 Ubuntu。 运行 `wsl --list` 将显示它不再列出。

## 设置 Linux 用户信息

在用户家中通过WSL启动Linux

```powershell
wsl ~
```

`~`可与 wsl 一起使用，以在用户的主目录中启动。 若要从 WSL 命令提示符内从任何目录跳转回主页，可以使用以下命令： `cd ~`

#### 设置 Linux 用户名和密码

使用 WSL 安装 Linux 发行版的过程完成后。 系统将要求你为 Linux 发行版创建“用户名”和“密码”。

- 此**用户名**和**密码**特定于安装的每个单独的 Linux 分发版，与 Windows 用户名无关。

- 创建**用户名**和**密码**后，该帐户将是分发版的默认用户，并将在启动时自动登录。

- 此帐户将被视为 Linux 管理员，能够运行 `sudo` (Super User Do) 管理命令。

- 在 WSL 上运行的每个 Linux 发行版都有其自己的 Linux 用户帐户和密码。 每当添加分发版、重新安装或重置时，都必须配置一个 Linux 用户帐户。

<img title="" src="https://learn.microsoft.com/zh-cn/windows/wsl/media/ubuntuinstall.png" alt="Ubuntu command line enter UNIX username" width="675">

如果忘记了 Linux 分发版的密码：

1. 请打开 PowerShell，并使用以下命令进入默认 WSL 分发版的根目录：`wsl -u root`

2. 在 PowerShell 内的根级别打开 WSL 发行版后，可使用此命令更新密码：`passwd <username>`，其中 `<username>` 是发行版中帐户的用户名，而你忘记了它的密码。

3. 系统将提示你输入新的 UNIX 密码，然后确认该密码。 在您被告知密码已正确更新后，请在 PowerShell 内使用以下命令关闭 WSL：`exit`。

![Windows Terminal screenshot](https://learn.microsoft.com/zh-cn/windows/wsl/media/terminal.png)

#### 更新和升级软件包

建议使用发行版的首选包管理器定期更新和升级包。 对于 Ubuntu 或 Debian，请使用以下命令：

```bash
sudo apt update && sudo apt upgrade
```

Windows 不会自动更新或升级 Linux 分发版。 大多数 Linux 用户往往倾向于自行控制此任务。

#### 文件存储

- 若想获得最快的性能速度，请将文件存储在 WSL 文件系统中，前提是使用 Linux 工具在 Linux 命令行（Ubuntu、OpenSUSE 等）中处理这些文件。 如果是使用 Windows 工具在 Windows 命令行（PowerShell、命令提示符）中工作，会将文件存储在 Windows 文件系统中。 可以跨操作系统访问文件，但这可能会显著降低性能。

例如，在存储 WSL 项目文件时：

- 使用 Linux 文件系统根目录：`\\wsl$\<DistroName>\home\<UserName>\Project`
- 不是 Windows 文件系统的根目录：`C:\Users\<UserName>\Project` 或 `/mnt/c/Users/<UserName>/Project$`

![Windows 文件资源管理器显示 Linux 存储](https://learn.microsoft.com/zh-cn/windows/wsl/media/windows-file-explorer.png)

#### 设置你最喜欢的代码编辑器

建议使用 Visual Studio Code 或 Visual Studio，因为它们直接支持使用 WSL 进行远程开发和调试。 Visual Studio Code 使你能够将 WSL 用作功能完备的开发环境。 Visual Studio 提供了对 C++ 跨平台开发的原生 WSL 支持。

##### 使用 Visual Studio Code

按照此分步指南开始在 WSL 中使用 Visual Studio Code，其中包括安装远程开发扩展包。 使用此扩展，能够运行 WSL、SSH 或开发容器，以使用整套 Visual Studio Code 功能进行编辑和调试。 在不同的独立开发环境之间快速切换并进行更新，而无需担心会影响本地计算机。

## 适用于 Linux 的 Windows 子系统使用 Visual Studio Code

#### 安装 VS Code 和 WSL 扩展

- 当在安装过程中系统提示“选择其他任务”时，请务必选中“添加到 PATH”选项，以便可以使用代码命令在 WSL 中轻松打开文件夹。

- 安装[远程开发扩展包](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)。 除了 Remote - SSH 和 Dev Containers 扩展之外，此扩展包还包含 WSL 扩展，使你能够打开容器中、远程计算机上或 WSL 中的任何文件夹。

**重要**：若要安装 WSL 扩展，需要 VS Code 的 [1.35 5 月发行版](https://code.visualstudio.com/updates/v1_35)版本或更高版本。 建议不要在没有 WSL 扩展的情况下在 VS Code 中使用 WSL，因为会失去对自动完成、调试、linting 等的支持。

```
此 WSL 扩展安装在 $HOME/.vscode/extensions 中
（在 PowerShell 中输入命令 `ls $HOME\.vscode\extensions\`）
```

#### 在 Visual Studio Code 中打开 WSL 项目

若要从 WSL 发行版打开项目，请打开发行版的命令行并输入：`code .`

![Open WSL project with VS Code remote server](https://learn.microsoft.com/zh-cn/windows/wsl/media/wsl-open-vs-code.gif)

还可以通过使用 VS Code 中的快捷方式 `CTRL+SHIFT+P` 调出命令面板，以访问更多 VS Code WSL 选项。 如果随后键入 `WSL`，你将看到可用的选项列表，你可以在 WSL 会话中重新打开文件夹，指定要在哪个发行版中打开，等等。

![VS Code's command palette](https://learn.microsoft.com/zh-cn/windows/wsl/media/vscode-remote-command-palette.png)

## WSL2 Ubuntu AOSP开发环境配置

#### 1. 优化 WSL2 性能，设置内存，网络等参数

创建或编辑`.wslconfig`文件：

`.wslconfig` 文件是创建在 **Windows 用户目录** 下，而不是 WSL 内部的 Ubuntu 系统中。具体路径为：C:\Users\你的用户名\\.wslconfig

ini

```ini
# Settings apply across all Linux distros running on WSL2 
[wsl2]  
# Limits VM memory to use more than 4 GB
memory=32GB  
# Sets the VM to use two virtual processors 
processors=4
# Sets amount of swap storage space to 32GB, default is 25% of available RAM 
swap=32GB
localhostForwarding=true
autoMemoryReclaim=gradual
networkingMode=mirrored
dnsTunneling=true
firewall=false
autoProxy=true
```

#### 2. 配置网络代理

在 WSL2 中配置 Windows 代理服务器：

- 代理软件已占用的端口号（不能随意设置）
  
  如果您使用的是 Windows 上已运行的代理软件（如 Clash、Shadowsocks、VPN 客户端等），端口号由该软件**自动分配或预先配置**，不能随意修改。例如：
  
  - **Clash for Windows** 默认使用 `7890`（HTTP 代理）或 `7891`（SOCKS5 代理）。
  - **Shadowsocks** 通常使用 `10808`（SOCKS5）或 `10809`（HTTP 代理）。
  - **VPN 客户端** 可能使用随机或固定端口（如 `443`、`1194` 等）。

正常情况WSL2设置了 `networkingMode=mirrored`，不再需要Linux设置代理，只要保证Windows代理可以正常使用就行。

验证网络连接情况

bash

```bash
# 测试代理是否工作
curl -v https://www.google.com
# 如果失败，检查代理配置和Windows防火墙设置
```

###### 常见问题

- 防火墙阻止连接：确保 Windows 防火墙允许 Clash 应用通过。在 **Windows 安全中心 > 防火墙和网络保护 > 允许应用通过防火墙** 中添加 Clash.exe，并勾选 **专用** 和 **公用** 网络。

- **端口被占用**：若 Clash 无法绑定到 `0.0.0.0:7890`，尝试在 Clash 界面修改端口（如改为 `7892`）。

- **WSL2 无法访问**：检查 WSL2 与 Windows 是否在同一网络（默认情况下是），或尝试重启 WSL2：
  
  bash
  
  ```powershell
  wsl --shutdown  # 关闭 WSL2
  wsl ~ # 重新启动
  ```

#### 3. 安装 AOSP 编译依赖

bash

```bash
# 更新包管理器
sudo apt update && sudo apt upgrade -y

# 安装必备工具
sudo apt install -y bc bison build-essential ccache curl flex g++-multilib gcc-multilib git gnupg gperf imagemagick lib32ncurses5-dev lib32readline-dev lib32z1-dev liblz4-tool libncurses5 libncurses5-dev libsdl1.2-dev libssl-dev libxml2 libxml2-utils lzop pngcrush rsync schedtool squashfs-tools xsltproc zip zlib1g-dev

# 安装Java开发工具包
sudo apt install -y openjdk-11-jdk

# 设置JDK环境变量
echo 'export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# 配置ccache加速编译
# 1.创建目录，(/home/hao/ 是Home下用户路径，根据实际路径修改)
mkdir /home/hao/.android_ccache
sudo mount --bind /home/hao/.ccache /home/hao/.android_ccache
# 2.配置ccache使能，路径 (/home/hao/ 是Home下用户路径，根据实际路径修改)
echo 'export USE_CCACHE=1' >> ~/.bashrc
echo 'export CCACHE_EXEC=/usr/bin/ccache' >> ~/.bashrc
echo 'export CCACHE_DIR=/home/hao/.android_ccache' >> ~/.bashrc
source ~/.bashrc

# 设置ccache大小（根据磁盘空间调整）
ccache -M 50G
```

#### 4. 下载 AOSP 源代码

bash

```bash
# 创建工作目录
mkdir -p ~/aosp

# 下载repo工具
mkdir -p ~/bin
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo

# 配置repo环境变量
echo 'export PATH=~/bin:$PATH'>> ~/.bashrc
source ~/.bashrc

# 设置git config配置
git config --global user.name "you name"
git config --global user.email "you email"
git config --global credential.helper store

# 初始化repo（选择合适的版本分支）
cd ~/aosp
repo init -u https://android.googlesource.com/platform/manifest -b android-12.0.0_r1

# 同步代码（使用多线程加速）
repo sync -j4 --force-sync
```

#### 5. 编译 AOSP

bash

```bash
# 进入AOSP目录
cd ~/aosp

# 设置编译环境
source build/envsetup.sh

# 选择目标设备（例如Pixel 3 XL）
lunch aosp_crosshatch-userdebug 
lunch aosp_x86_64-eng # AVD img 版本

# 开始编译（根据CPU核心数调整-j参数）
make -j2

# 编译emulator可以使用的版本
source build/envsetup.sh
lunch sdk_phone_x86_64
make - j1 emulator
```

 运行结果 :

![](C:\Users\zheng\AppData\Roaming\marktext\images\2025-06-05-19-39-57-c9cff9520f9eb4411add773d5553d45.png)

#### 6. AVD 运行 AOSP Img

###### 准备工作

1. **确保 AOSP 编译成功**：成功编译 AOSP 后，在相应的输出目录（如`out/target/product/<product_name>` ，`<product_name>` 常见的有 `generic_x86_64` 等）中，会生成一系列镜像文件，关键的文件包括`system-qemu.img` 、`ramdisk.img` 、`vendor-qemu.img` 、`kernel-ranchu` 以及`system/build.prop` 等 。其中，`system-qemu.img`是专门供模拟器使用的系统镜像文件，注意不要使用普通的`system.img` 。
2. **创建合适的 AVD**：打开 Android Studio 中的 AVD Manager，创建一个新的虚拟设备。在选择系统镜像步骤时，要选择与你编译的 AOSP 镜像架构一致的镜像（比如编译的是 x86_64 架构的 AOSP 镜像，这里就选择 x86_64 架构的 Android 系统镜像） 。其他参数可根据需求设置，创建完成后先不要启动。

###### 替换镜像文件

1. **找到 AVD 镜像存储目录**：不同操作系统下，AVD 镜像存储目录有所不同。
   - **Windows**：通常在`C:\Users\<你的用户名>\.android\avd\<虚拟设备名>.avd` 。
   - **macOS**：一般在`/Users/<你的用户名>/.android/avd/<虚拟设备名>.avd` 。
   - **Linux**：多在`~/.android/avd/<虚拟设备名>.avd` 。
2. **备份原镜像文件（可选但推荐）**：进入上述目录后，对原有的`system.img` 、`ramdisk.img` 、`vendor.img` 等文件进行备份，以防操作失误。
3. **复制并重命名编译生成的镜像文件**：将编译 AOSP 生成的`system-qemu.img` 、`ramdisk.img` 、`vendor-qemu.img` 以及`system/build.prop` 等文件复制到 AVD 镜像存储目录。然后将`system-qemu.img`重命名为`system.img` ，`vendor-qemu.img`重命名为`vendor.img` 。如果同时存在`system.img`和`system-qemu.img` ，保留`system-qemu.img`并重命名为`system.img` ，因为`system.img`通常是用于刷机，而`system-qemu.img`才是供模拟器使用的 。同时，将`system/build.prop`文件也复制到该目录下。

###### 启动 AVD 并验证

1. **启动 AVD**：回到 Android Studio 的 AVD Manager，选择刚才修改镜像文件的虚拟设备，点击 “启动” 按钮。
2. **验证**：等待 AVD 启动完成后，检查系统是否正常运行，测试一些基本功能（如打开应用、设置等） ，看是否加载的是自己编译的 AOSP 镜像。

###### 其他方法（命令行方式）

如果你更倾向于使用命令行，也可以按照以下步骤操作：

1. **设置环境变量**：确保你的系统环境变量中已经配置好了 Android SDK 的`tools`目录路径，以便能直接使用`emulator`命令。
2. **执行命令**：打开命令行窗口，进入到 Android SDK 的`emulator`目录（例如`cd <android_sdk_path>/emulator` ），然后执行类似以下命令启动 AVD 并加载自己编译的镜像：

bash

```bash
emulator -avd <虚拟设备名> -sysdir <编译镜像所在目录> -datadir <数据存储目录> -kernel <kernel文件路径> -ramdisk <ramdisk文件路径> -system <system镜像文件路径> -data <userdata文件路径> -cache <cache文件路径> -vendor <vendor镜像文件路径>
```

其中，`<虚拟设备名>` 是你在 AVD Manager 中创建的虚拟设备名称；`<编译镜像所在目录>` 是编译生成的 AOSP 镜像文件所在目录；后面的参数根据实际镜像文件位置填写 。例如：

bash

```bash
emulator -avd MyAVD -sysdir /path/to/aosp/out/target/product/generic_x86_64 -datadir /path/to/data -kernel /path/to/aosp/out/target/product/generic_x86_64/kernel-ranchu -ramdisk /path/to/aosp/out/target/product/generic_x86_64/ramdisk.img -system /path/to/aosp/out/target/product/generic_x86_64/system-qemu.img -data /path/to/aosp/out/target/product/generic_x86_64/userdata.img -cache /path/to/aosp/out/target/product/generic_x86_64/cache.img -vendor /path/to/aosp/out/target/product/generic_x86_64/vendor-qemu.img
```

如果在加载过程中遇到问题，比如启动失败、报错等，可以通过查看 AVD 启动日志来排查。在命令行启动 AVD 时添加`-verbose`参数（如`emulator -avd <虚拟设备名> -verbose` ），能获取更详细的日志信息，帮助定位问题所在。
