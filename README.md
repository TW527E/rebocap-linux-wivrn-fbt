**简体中文** · [繁體中文](README.zh-TW.md) · [English](README.en.md) · [日本語](README.ja.md)

# Rebocap × Linux × WiVRn 全身追踪（VRChat）

在 Linux 上把 **Rebocap** 惯性动捕的全身追踪接到 **VRChat**。不用 SteamVR，走 **WiVRn + xrizer**，头显是 Meta Quest。

Rebocap 官方已经出了 Linux 版，但它只附 **SteamVR 驱动**。如果你用 SteamVR，装官方版就够了，不需要这个仓库。这个仓库是给**不用 SteamVR、改用 WiVRn** 的人：用 CLI 把 Rebocap 的追踪器转成 SolarXR，送进 WiVRn。

这份记录包括实际环境、从零开始的安装步骤、日常怎么玩，以及途中遇到的每一个坑和解法。

## 两种方案

| | **A. 官方 Linux 版（推荐）** | B. Windows 版跑在 Wine（Bottles） |
|---|---|---|
| Rebocap | 官方 Linux 原生程序 | Windows 版装在 Bottles |
| 接收器 | 官方程序直接用 USB | Wine 的 COM 端口 + setupapi 补丁 |
| 和 CLI 之间 | Rebocap 自己开 TCP `127.0.0.1:20480` | `bridge.exe` 在 Wine 里把 named pipe 转成 TCP `127.0.0.1:36850` |
| 需要的东西 | Rebocap、CLI、本仓库 | Rebocap、Bottles、rebocap-wine 补丁、bridge.exe、CLI、本仓库 |
| 追踪器固件 | 可以用官方版更新（我从 v15 升到 v17，感觉更稳） | 不要在 Wine 里更新 |

方案 A 少了 Wine、补丁和 bridge.exe 三层，启动和关闭也简单很多，除非官方版在你的电脑上跑不起来，不然请用 A。方案 B 留着当备案，也记录当初怎么接通的。

```
方案 A：
Quest 头显 → WiVRn → CLI ⇄ Rebocap（Linux 原生，:20480）
Rebocap 追踪器 → Rebocap → CLI → SolarXR → WiVRn → xrizer → VRChat

方案 B：
Quest 头显 → WiVRn → CLI ⇄ bridge.exe（Wine，:36850）⇄ Rebocap（Wine）
Rebocap 追踪器 → Rebocap → bridge.exe → CLI → SolarXR → WiVRn → xrizer → VRChat
```

启动脚本 `rebocap-fbt` 两种方案都支持，会自动检测：20480 在监听就用方案 A，没有再找 36850 的 bridge.exe。

## 实测环境

| 项目 | 版本／设置 |
|---|---|
| 系统 | Arch Linux（Omarchy，Hyprland），NVIDIA RTX 3070 |
| 头显 | Meta Quest，WiVRn（WiFi 或 USB） |
| WiVRn | wivrn-server / wivrn-dashboard 26.9 |
| xrizer | xrizer-git 0.6.r6（rebocap-wivrn-bridge 作者测试的是 0.5，0.6 也正常） |
| Rebocap（方案 A） | 官方 Linux 版 `rebocap-linux-preview-0.1-1`，追踪器固件 v17 |
| Rebocap（方案 B） | Windows 版，在 Flatpak **Bottles** 里运行，runner 是 soda-11.0-10（有 ntsync），追踪器固件 v15 |
| 追踪器 | 平常 10 点：腰、胸、双大腿、双小腿、双脚、双上臂（原本绑双前臂，用不到就改成小腿）。测试时用 6 点：腰、双大腿、双小腿、胸 |

## 用到的项目

| 项目 | 用途 | 方案 |
|---|---|---|
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | CLI（`monad2steamvr`）：在 WiVRn 和 Rebocap 之间转发头部与追踪器姿势，用 SolarXR 发布追踪器。fork 自 [akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge)，加了 WiVRn 游玩区移动的修正（libmonado），并拿掉追踪器转发时两段 10 ms 的轮询等待（平均少约 10 ms 延迟） | A、B |
| 本仓库 | 启动脚本 `rebocap-fbt`、VRChat 按键绑定生成器、这份记录 | A、B |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | setupapi 补丁，让 Rebocap 在 Wine 里识别接收器 | B |
| [TW527E/rebocap-linux-compat](https://github.com/TW527E/rebocap-linux-compat) | `bridge.exe`：在 Wine 里把 Rebocap 的 named pipe 转成 TCP。fork 自 [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat)，加了 Bottles/ntsync 修正 | B |

## 安装（从零开始）

先完成「共同步骤」，再按你选的方案做 A 或 B。

### 共同 1. WiVRn 与 xrizer

按 WiVRn 的说明装好 wivrn-server、wivrn-dashboard，以及 xrizer，确认不开全追时 VRChat 能正常用。

### 共同 2. CLI

```sh
sudo pacman -S rust openxr pkgconf
git clone https://github.com/TW527E/rebocap-wivrn-bridge ~/Projects/rebocap-wivrn-bridge
cd ~/Projects/rebocap-wivrn-bridge && cargo build --release
```

### 共同 3. 启动脚本

```sh
git clone https://github.com/TW527E/rebocap-linux-wivrn-fbt ~/Projects/rebocap-linux-wivrn-fbt
ln -s ~/Projects/rebocap-linux-wivrn-fbt/scripts/rebocap-fbt ~/.local/bin/rebocap-fbt
```

脚本会自动找 CLI（`monad2steamvr`），依次是：`REBOCAP_CLI`、`PATH`、跟这个仓库放在同一层的 `rebocap-wivrn-bridge`，以及 `~/Projects`、`~/Downloads`、`~/src`、`~/` 底下的 `rebocap-wivrn-bridge`。放在其他地方，或其他路径不同时，可以用环境变量覆盖：`REBOCAP_CLI`、`REBOCAP_BOTTLE`、`REBOCAP_RECEIVER`、`REBOCAP_XR_RUNTIME_JSON`。端口号可以用 `REBOCAP_NATIVE_PORT`（默认 20480）和 `REBOCAP_WINE_PORT`（默认 36850）改。

### 方案 A：官方 Linux 版（推荐）

1. 依[官方安装说明](https://doc.rebocap.com/zh-Hans/docs/tutorial/software_install)下载 Linux 版安装包并运行：
   ```sh
   chmod +x rebocap-linux-preview-*.run
   ./rebocap-linux-preview-*.run
   ```
   我装在 `~/.local/opt/rebocap`，从应用程序菜单开 Rebocap，或运行 `~/.local/opt/rebocap/rebocap-ui`。
2. 给接收器权限。官方附了脚本，会写入 `/etc/udev/rules.d/70-rebocap.rules`，并把你加进 `dialout`／`uucp`／`plugdev` 其中一个组（Arch 是 `uucp`）：
   ```sh
   sudo bash ~/.local/opt/rebocap/linux/install_linux_permissions.sh --install --user "$USER"
   ```
   之后重新登录，并重插接收器。
3. 开 Rebocap，有提示更新追踪器固件就照着更新。**更新中不要拔接收器、不要关程序。**
4. **不要**装官方的 SteamVR 驱动（`data/rebocap_driver`）。我们不用 SteamVR，而且 20480 这个端口同一时间只能连一个程序，SteamVR 驱动连着的话 CLI 就连不上。

Bottles、rebocap-wine、bridge.exe 都不需要。

> 官方版运行时会把接收器从 `cdc_acm` 卸下来，直接用 USB 访问，所以这时 `/dev/ttyACM*` 和 `/dev/serial/by-id/...` 会消失。这是正常的，脚本改成直接找 USB 设备 248a:8002。

### 方案 B：Windows 版跑在 Wine（Bottles）

#### B1. 接收器权限

用户要在拥有 `/dev/ttyACM*` 的组里（Arch 是 `uucp`），重启后才会持续有效：

```sh
sudo usermod -aG uucp "$USER"   # 运行后要重新登录
```

我另外加了一条 udev 规则 `/etc/udev/rules.d/70-rebocap.rules`，可加可不加：

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="248a", ATTRS{idProduct}=="8002", MODE="0660", TAG+="uaccess"
```

#### B2. Rebocap 装进 Bottles

1. 给 Bottles 设备访问权限，Wine 才能访问串口：`flatpak override --user --device=all com.usebottles.bottles`。
2. 创建 Bottle（我取名 `Rebocap`，win64），安装 Rebocap。
3. 套用 [rebocap-wine](https://github.com/TW527E/rebocap-wine) 的 setupapi 补丁，让 Rebocap 能枚举到接收器。
4. 在 Bottle 的 `dosdevices` 把接收器映射成 COM 端口，例如 `com33 -> /dev/ttyACM0`。

#### B3. bridge.exe

```sh
git clone https://github.com/TW527E/rebocap-linux-compat
cd rebocap-linux-compat
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32   # 需要 mingw-w64-gcc
cp bridge.exe ~/.var/app/com.usebottles.bottles/data/bottles/bottles/Rebocap/drive_c/Program\ Files\ \(x86\)/Rebocap/
```

不想自己编译的话，可以从 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) 下载 `bridge.exe`。

**不需要**运行它的 `install.py`，也不用注册 SteamVR 驱动。

建议在 Bottles GUI 里把 `bridge.exe` 加成这个 Bottle 的程序，方便日常启动。

### （选用）右摇杆按下 = 跳跃

如果手柄 A 键坏了，可以让右摇杆按下也能跳。产生 xrizer 用的绑定文件：

```sh
./xrizer-bindings/make-vrchat-bindings.py      # 写到 ~/.config/xrizer-bindings/oculustouch.json
```

然后在 VRChat 的 Steam 启动选项最前面加上：

```
XRIZER_CUSTOM_BINDINGS_DIR=/home/<你>/.config/xrizer-bindings
```

右摇杆原本的功能是「Udon Stick Click」，只有少数世界会用到。A 键照样可以跳。VRChat 更新后重跑一次生成器即可。要恢复原状，删掉这个启动选项就好。

## 日常怎么玩

**不玩全追**：照平常的方式就好，开 WiVRn → 头显连接 → 开 VRChat。不需要 Rebocap，也不需要脚本。

**全身追踪**，**顺序很重要**：

1. 开 **WiVRn**（wivrn-dashboard），确认服务器在运行。**头显先不要连。**
2. 开 **Rebocap**：
   - 方案 A：直接开官方 Linux 版。
   - 方案 B：在 Bottles 开 Rebocap，再开 bridge.exe。
3. 终端运行：
   ```sh
   rebocap-fbt          # 10 点（默认）：腰、胸、双大腿、双小腿、双脚、双上臂。自动检测方案
   rebocap-fbt 12       # 12 点：10 点再加双前臂
   rebocap-fbt 6        # 6 点
   rebocap-fbt 10 --usb # 准备好后自动用 USB 连头显
   rebocap-fbt --wine   # 强制用方案 B（--native 是强制用方案 A）
   ```
   开头会输出现在用的是哪个方案，例如 `模式：官方 Linux 版 Rebocap（127.0.0.1:20480）`（脚本的输出是繁体中文）。
4. 看到「**SolarXR 準備好了**」（也会跳桌面通知）之后，**连头显**。
5. Rebocap 切到 **VR 模式**，戴着头显**校准**。
6. 从 Steam 开 **VRChat**，在快捷菜单点 **Calibrate FBT**。

脚本会显示部位检查：

```
有資料的部位：0 1 2 3 4 5 6 7 9 10
  ⚠ 已發布但沒有資料：…
  ⚠ 有資料但沒有發布：…
```

出现 ⚠ 就代表部位参数和实际绑的追踪器不一致，要修正部位参数。

**结束**：

1. 在脚本的终端按 **Ctrl+C**。脚本会关掉 CLI。方案 B 还会用 `bridge.quit` 让 bridge 自己退出（加 `--keep-bridge` 就不关）。
2. 关掉 **Rebocap** 窗口。
3. WiVRn 照平常的方式处理。

**重要**：重启 CLI 或换了部位之后，头显要**重新连接**，VRChat 也要**重开**。WiVRn 只在头显连接时读取追踪器列表。

### 部位编号

| 编号 | 部位 | 编号 | 部位 |
|---|---|---|---|
| 0 | 腰 | 7 | 胸 |
| 1 / 2 | 左／右大腿 | 9 / 10 | 左／右上臂 |
| 3 / 4 | 左／右小腿 | 11 / 12 | 左／右前臂 |
| 5 / 6 | 左／右脚 | | |

VRChat 校准时是依追踪器的**位置**分配给臀、脚、膝、胸、肘，所以 6 点套件的小腿追踪器可以当成脚来用。

CLI 默认的 `0,5,6` 只适合有绑脚的套件。6 点套件没有脚的实体追踪器，5、6 只是 Rebocap 建出的虚拟节点，只会送短暂的一点数据，用默认值的话 VRChat 会抓不到脚。

### USB 连接

1. 头显要开开发者模式，并允许这台电脑用 USB 调试。授权窗口会显示在头显里，要戴上才看得到。
2. 运行 `adb devices`，状态应该是 `device`。如果是 `unauthorized`，代表头显还没按允许。
3. 用 `rebocap-fbt --usb`，或在 wivrn-dashboard 按「通过 USB 连接」。

手动的话：

```sh
adb reverse tcp:9757 tcp:9757
adb shell am start -a android.intent.action.VIEW -d "wivrn+tcp://127.0.0.1:9757" org.meumeu.wivrn
```

## 更新

```sh
cd ~/Projects/rebocap-wivrn-bridge && git pull && cargo build --release   # CLI
cd ~/Projects/rebocap-linux-wivrn-fbt && git pull                         # 启动脚本（软链接会自动指向新版）
```

官方 Linux 版 Rebocap 用安装目录里的 `maintenancetool` 更新，或下载新的安装包重装。

## 延迟

两个方案到 CLI 之前的差别很小：方案 A 少了 Wine 的 named pipe、bridge.exe 的轮询和 Wine 的串口模拟，大约省下 1～3 ms。
比较大的差别在 CLI：原版 CLI 里追踪器姿态要在两个 10 ms 的轮询循环里排队，平均多约 10 ms、最多约 20 ms，更新间隔也忽长忽短。[TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) 改成一收到就送给 WiVRn。

## 踩过的坑

### 两种方案都会遇到

| 症状 | 原因 | 解法 |
|---|---|---|
| CLI：`Failed to connect to monado service process` | WiVRn 服务器没在跑（dashboard 开着不代表服务器开着） | 在 dashboard 按「启动服务器」 |
| CLI 找不到 WiVRn | 系统没有 `active_runtime.json` | 加上 `XR_RUNTIME_JSON=/usr/share/openxr/1/openxr_wivrn.json`（脚本已自动加） |
| CLI 显示 `invalid SolarXR frame` 加上用法说明 | 部位参数里混了全角标点（例如 `12。`） | 只用半角数字和逗号 |
| VRChat 抓不到脚 | 部位参数和实际绑的追踪器不一致 | 看脚本的部位检查，修正部位参数 |
| 拖动游玩区后追踪器偏离身体 | CLI 读不到 WiVRn 的 STAGE 偏移（log 有 `libmonado unavailable`） | 用 TW527E 的 CLI fork，并确认 WiVRn 的 runtime JSON 里有 `MND_libmonado_path` |

### 方案 A（官方 Linux 版）

| 症状 | 原因 | 解法 |
|---|---|---|
| 脚本说找不到接收器，但 Rebocap 能识别 | 官方版把接收器从 `cdc_acm` 卸下来直接用 USB，`/dev/ttyACM*` 会消失 | 脚本已改成找 USB 设备 248a:8002 |
| CLI 连上了但一直没有数据 | Rebocap 还没切 VR 模式，或正在更新固件 | 等固件更新完，切 VR 模式并校准 |
| 脚本警告 20480 已经有程序连着 | 官方 SteamVR 驱动（或另一个 CLI）占着，这个端口只收一个连接 | 关掉 SteamVR／另一个 CLI |

### 方案 B（Wine／Bottles）

详细的调试过程在 [docs/troubleshooting.md](docs/troubleshooting.md)（繁体中文）。重点如下：

| 症状 | 原因 | 解法 |
|---|---|---|
| bridge.exe 显示 listening，但连进去永远没有反应 | 这个 Wine（soda-11.0-10 + ntsync）的阻塞式 socket 调用是坏的 | bridge 改用非阻塞 socket 加事件等待（fork 已修） |
| 没有腰的追踪器、脚的数据断断续续、bridge 一直输出 `0 bytes` | 读 pipe 时返回 0 bytes，消息被丢掉了 | 先用 `PeekNamedPipe` 确认有数据才读（fork 已修） |
| bridge CPU 很高、Rebocap 完全没反应 | pipe 的读取一直处于等待，IO 状态永远不会更新 | 同上 |
| 从终端 `kill` bridge 后，Rebocap 崩溃 | wineserver 在处理被杀的进程时 segfault | **永远不要从主机 kill Wine 进程**，改用 `bridge.quit` |
| Rebocap 显示「网络校验失败，10 秒后退出」然后卡住 | wineserver 已经不在了，Wine 里的网络全部失效 | 结束整个 Bottle 再重开。不是网络本身的问题 |
| Bottles「停止」一直卡住 | bridge 是 console 程序，Rebocap 也不理会 Windows 的关机信号 | 先 `bridge.quit`，再手动关 Rebocap 窗口 |

## 许可

本仓库的脚本和文档采用 MIT License。`bridge.exe` 的代码来自 dogesoulseller/rebocap-linux-compat（MIT）。CLI 由 akazdayo/rebocap-wivrn-bridge 开发（我的修改在 TW527E/rebocap-wivrn-bridge），本仓库只放链接，不包含它的代码。Rebocap 是 JXQ 的专有软件，本仓库不包含它的任何文件。
