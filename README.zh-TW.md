[简体中文](README.md) · **繁體中文** · [English](README.en.md) · [日本語](README.ja.md)

# Rebocap × Linux × WiVRn 全身追蹤（VRChat）

在 Linux 上把 **Rebocap** 慣性動捕的全身追蹤接到 **VRChat**。不用 SteamVR，走 **WiVRn + xrizer**，頭顯是 Meta Quest。

Rebocap 官方已經出了 Linux 版，但它只附 **SteamVR 驅動**。如果你用 SteamVR，裝官方版就夠了，不需要這個 repo。這個 repo 是給**不用 SteamVR、改用 WiVRn** 的人：用 CLI 把 Rebocap 的追蹤器轉成 SolarXR，送進 WiVRn。

這份紀錄包括實際環境、從零開始的安裝步驟、日常怎麼玩，以及途中遇到的每一個坑和解法。

## 兩種方案

| | **A. 官方 Linux 版（推薦）** | B. Windows 版跑在 Wine（Bottles） |
|---|---|---|
| Rebocap | 官方 Linux 原生程式 | Windows 版裝在 Bottles |
| 接收器 | 官方程式直接用 USB | Wine 的 COM 埠 + setupapi 補丁 |
| 和 CLI 之間 | Rebocap 自己開 TCP `127.0.0.1:20480` | `bridge.exe` 在 Wine 裡把 named pipe 轉成 TCP `127.0.0.1:36850` |
| 需要的東西 | Rebocap、CLI、本 repo | Rebocap、Bottles、rebocap-wine 補丁、bridge.exe、CLI、本 repo |
| 追蹤器韌體 | 可以用官方版更新（我從 v15 升到 v17，感覺更穩） | 不要在 Wine 裡更新 |

方案 A 少了 Wine、補丁和 bridge.exe 三層，啟動和關閉也簡單很多，除非官方版在你的電腦上跑不起來，不然請用 A。方案 B 留著當備案，也記錄當初怎麼接通的。

```
方案 A：
Quest 頭顯 → WiVRn → CLI ⇄ Rebocap（Linux 原生，:20480）
Rebocap 追蹤器 → Rebocap → CLI → SolarXR → WiVRn → xrizer → VRChat

方案 B：
Quest 頭顯 → WiVRn → CLI ⇄ bridge.exe（Wine，:36850）⇄ Rebocap（Wine）
Rebocap 追蹤器 → Rebocap → bridge.exe → CLI → SolarXR → WiVRn → xrizer → VRChat
```

啟動腳本 `rebocap-fbt` 兩種方案都支援，會自動偵測：20480 有在監聽就用方案 A，沒有再找 36850 的 bridge.exe。

## 實測環境

| 項目 | 版本／設定 |
|---|---|
| 系統 | Arch Linux（Omarchy，Hyprland），NVIDIA RTX 3070 |
| 頭顯 | Meta Quest，WiVRn（WiFi 或 USB） |
| WiVRn | wivrn-server / wivrn-dashboard 26.9 |
| xrizer | xrizer-git 0.6.r6（rebocap-wivrn-bridge 作者測試的是 0.5，0.6 也正常） |
| Rebocap（方案 A） | 官方 Linux 版 `rebocap-linux-preview-0.1-1`，追蹤器韌體 v17 |
| Rebocap（方案 B） | Windows 版，在 Flatpak **Bottles** 裡執行，runner 是 soda-11.0-10（有 ntsync），追蹤器韌體 v15 |
| 追蹤器 | 平常 10 點：腰、胸、雙大腿、雙小腿、雙腳、雙上臂（原本綁雙前臂，用不到就改成小腿）。測試時用 6 點：腰、雙大腿、雙小腿、胸 |

## 用到的專案

| 專案 | 用途 | 方案 |
|---|---|---|
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | CLI（`monad2steamvr`）：在 WiVRn 和 Rebocap 之間轉送頭部與追蹤器姿勢，用 SolarXR 發布追蹤器。fork 自 [akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge)，加了 WiVRn 遊玩區移動的修正（libmonado），並拿掉追蹤器轉送時兩段 10 ms 的輪詢等待（平均少約 10 ms 延遲） | A、B |
| 本 repo | 啟動腳本 `rebocap-fbt`、VRChat 按鍵綁定產生器、這份紀錄 | A、B |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | setupapi 補丁，讓 Rebocap 在 Wine 裡認得接收器 | B |
| [TW527E/rebocap-linux-compat](https://github.com/TW527E/rebocap-linux-compat) | `bridge.exe`：在 Wine 裡把 Rebocap 的 named pipe 轉成 TCP。fork 自 [dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat)，加了 Bottles/ntsync 修正 | B |

## 安裝（從零開始）

先完成「共同步驟」，再依你選的方案做 A 或 B。

### 共同 1. WiVRn 與 xrizer

照 WiVRn 的說明裝好 wivrn-server、wivrn-dashboard，以及 xrizer，確認不開全追時 VRChat 能正常用。

### 共同 2. CLI

```sh
sudo pacman -S rust openxr pkgconf
git clone https://github.com/TW527E/rebocap-wivrn-bridge ~/Projects/rebocap-wivrn-bridge
cd ~/Projects/rebocap-wivrn-bridge && cargo build --release
```

### 共同 3. 啟動腳本

```sh
git clone https://github.com/TW527E/rebocap-linux-wivrn-fbt ~/Projects/rebocap-linux-wivrn-fbt
ln -s ~/Projects/rebocap-linux-wivrn-fbt/scripts/rebocap-fbt ~/.local/bin/rebocap-fbt
```

腳本會自動找 CLI（`monad2steamvr`），依序是：`REBOCAP_CLI`、`PATH`、跟這個 repo 放在同一層的 `rebocap-wivrn-bridge`，以及 `~/Projects`、`~/Downloads`、`~/src`、`~/` 底下的 `rebocap-wivrn-bridge`。放在其他地方，或其他路徑不同時，可以用環境變數覆寫：`REBOCAP_CLI`、`REBOCAP_BOTTLE`、`REBOCAP_RECEIVER`、`REBOCAP_XR_RUNTIME_JSON`。埠號可以用 `REBOCAP_NATIVE_PORT`（預設 20480）和 `REBOCAP_WINE_PORT`（預設 36850）改。

### 方案 A：官方 Linux 版（推薦）

1. 依[官方安裝說明](https://doc.rebocap.com/zh-Hans/docs/tutorial/software_install)下載 Linux 版安裝檔並執行：
   ```sh
   chmod +x rebocap-linux-preview-*.run
   ./rebocap-linux-preview-*.run
   ```
   我裝在 `~/.local/opt/rebocap`，從應用程式選單開 Rebocap，或執行 `~/.local/opt/rebocap/rebocap-ui`。
2. 給接收器權限。官方附了腳本，會寫入 `/etc/udev/rules.d/70-rebocap.rules`，並把你加進 `dialout`／`uucp`／`plugdev` 其中一個群組（Arch 是 `uucp`）：
   ```sh
   sudo bash ~/.local/opt/rebocap/linux/install_linux_permissions.sh --install --user "$USER"
   ```
   之後重新登入，並重插接收器。
3. 開 Rebocap，有提示更新追蹤器韌體就照著更新。**更新中不要拔接收器、不要關程式。**
4. **不要**裝官方的 SteamVR 驅動（`data/rebocap_driver`）。我們不用 SteamVR，而且 20480 這個埠同一時間只能連一個程式，SteamVR 驅動連著的話 CLI 就連不上。

Bottles、rebocap-wine、bridge.exe 都不需要。

> 官方版執行時會把接收器從 `cdc_acm` 卸下來，直接用 USB 存取，所以這時 `/dev/ttyACM*` 和 `/dev/serial/by-id/...` 會消失。這是正常的，腳本改成直接找 USB 裝置 248a:8002。

### 方案 B：Windows 版跑在 Wine（Bottles）

#### B1. 接收器權限

使用者要在擁有 `/dev/ttyACM*` 的群組裡（Arch 是 `uucp`），重開機後才會持續有效：

```sh
sudo usermod -aG uucp "$USER"   # 執行後要重新登入
```

我另外加了一條 udev 規則 `/etc/udev/rules.d/70-rebocap.rules`，可加可不加：

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="248a", ATTRS{idProduct}=="8002", MODE="0660", TAG+="uaccess"
```

#### B2. Rebocap 裝進 Bottles

1. 給 Bottles 裝置存取權限，Wine 才能存取序列埠：`flatpak override --user --device=all com.usebottles.bottles`。
2. 建立 Bottle（我取名 `Rebocap`，win64），安裝 Rebocap。
3. 套用 [rebocap-wine](https://github.com/TW527E/rebocap-wine) 的 setupapi 補丁，讓 Rebocap 能列舉到接收器。
4. 在 Bottle 的 `dosdevices` 把接收器對應成 COM 埠，例如 `com33 -> /dev/ttyACM0`。

#### B3. bridge.exe

```sh
git clone https://github.com/TW527E/rebocap-linux-compat
cd rebocap-linux-compat
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32   # 需要 mingw-w64-gcc
cp bridge.exe ~/.var/app/com.usebottles.bottles/data/bottles/bottles/Rebocap/drive_c/Program\ Files\ \(x86\)/Rebocap/
```

不想自己編譯的話，可以從 [Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) 下載 `bridge.exe`。

**不需要**執行它的 `install.py`，也不用註冊 SteamVR 驅動。

建議在 Bottles GUI 裡把 `bridge.exe` 加成這個 Bottle 的程式，方便日常啟動。

### （選用）右搖桿按下 = 跳躍

如果手把 A 鍵壞了，可以讓右搖桿按下也能跳。產生 xrizer 用的綁定檔：

```sh
./xrizer-bindings/make-vrchat-bindings.py      # 寫到 ~/.config/xrizer-bindings/oculustouch.json
```

然後在 VRChat 的 Steam 啟動選項最前面加上：

```
XRIZER_CUSTOM_BINDINGS_DIR=/home/<你>/.config/xrizer-bindings
```

右搖桿原本的功能是「Udon Stick Click」，只有少數世界會用到。A 鍵照樣可以跳。VRChat 更新後重跑一次產生器即可。要恢復原狀，刪掉這個啟動選項就好。

## 日常怎麼玩

**不玩全追**：照平常的方式就好，開 WiVRn → 頭顯連線 → 開 VRChat。不需要 Rebocap，也不需要腳本。

**全身追蹤**，**順序很重要**：

1. 開 **WiVRn**（wivrn-dashboard），確認伺服器在執行。**頭顯先不要連。**
2. 開 **Rebocap**：
   - 方案 A：直接開官方 Linux 版。
   - 方案 B：在 Bottles 開 Rebocap，再開 bridge.exe。
3. 終端機執行：
   ```sh
   rebocap-fbt          # 10 點（預設）：腰、胸、雙大腿、雙小腿、雙腳、雙上臂。自動偵測方案
   rebocap-fbt 12       # 12 點：10 點再加雙前臂
   rebocap-fbt 6        # 6 點
   rebocap-fbt 10 --usb # 準備好後自動用 USB 連頭顯
   rebocap-fbt --wine   # 強制用方案 B（--native 是強制用方案 A）
   ```
   開頭會印出現在用的是哪個方案，例如 `模式：官方 Linux 版 Rebocap（127.0.0.1:20480）`。
4. 看到「**SolarXR 準備好了**」（也會跳桌面通知）之後，**連頭顯**。
5. Rebocap 切到 **VR 模式**，戴著頭顯**校準**。
6. 從 Steam 開 **VRChat**，在快捷選單點 **Calibrate FBT**。

腳本會顯示部位檢查：

```
有資料的部位：0 1 2 3 4 5 6 7 9 10
  ⚠ 已發布但沒有資料：…
  ⚠ 有資料但沒有發布：…
```

出現 ⚠ 就代表部位參數和實際綁的追蹤器不一致，要修正部位參數。

**結束**：

1. 在腳本的終端機按 **Ctrl+C**。腳本會關掉 CLI。方案 B 還會用 `bridge.quit` 讓 bridge 自己退出（加 `--keep-bridge` 就不關）。
2. 關掉 **Rebocap** 視窗。
3. WiVRn 照平常的方式處理。

**重要**：重啟 CLI 或換了部位之後，頭顯要**重新連線**，VRChat 也要**重開**。WiVRn 只在頭顯連線時讀取追蹤器清單。

### 部位編號

| 編號 | 部位 | 編號 | 部位 |
|---|---|---|---|
| 0 | 腰 | 7 | 胸 |
| 1 / 2 | 左／右大腿 | 9 / 10 | 左／右上臂 |
| 3 / 4 | 左／右小腿 | 11 / 12 | 左／右前臂 |
| 5 / 6 | 左／右腳 | | |

VRChat 校準時是依追蹤器的**位置**分配給臀、腳、膝、胸、肘，所以 6 點套組的小腿追蹤器可以當成腳來用。

CLI 預設的 `0,5,6` 只適合有綁腳的套組。6 點套組沒有腳的實體追蹤器，5、6 只是 Rebocap 建出的虛擬節點，只會送短暫的一點資料，用預設值的話 VRChat 會抓不到腳。

### USB 連線

1. 頭顯要開開發者模式，並允許這台電腦用 USB 偵錯。授權視窗會顯示在頭顯裡，要戴上才看得到。
2. 執行 `adb devices`，狀態應該是 `device`。如果是 `unauthorized`，代表頭顯還沒按允許。
3. 用 `rebocap-fbt --usb`，或在 wivrn-dashboard 按「透過 USB 連線」。

手動的話：

```sh
adb reverse tcp:9757 tcp:9757
adb shell am start -a android.intent.action.VIEW -d "wivrn+tcp://127.0.0.1:9757" org.meumeu.wivrn
```

## 更新

```sh
cd ~/Projects/rebocap-wivrn-bridge && git pull && cargo build --release   # CLI
cd ~/Projects/rebocap-linux-wivrn-fbt && git pull                         # 啟動腳本（symlink 會自動指到新版）
```

官方 Linux 版 Rebocap 用安裝資料夾裡的 `maintenancetool` 更新，或下載新的安裝檔重裝。

## 延遲

兩個方案到 CLI 之前的差別很小：方案 A 少了 Wine 的 named pipe、bridge.exe 的輪詢和 Wine 的序列埠模擬，大約省下 1～3 ms。
比較大的差別在 CLI：原版 CLI 裡追蹤器姿態要在兩個 10 ms 的輪詢迴圈裡排隊，平均多約 10 ms、最多約 20 ms，更新間隔也忽長忽短。[TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) 改成一收到就送給 WiVRn。

## 踩過的坑

### 兩種方案都會遇到

| 症狀 | 原因 | 解法 |
|---|---|---|
| CLI：`Failed to connect to monado service process` | WiVRn 伺服器沒在跑（dashboard 開著不代表伺服器開著） | 在 dashboard 按「啟動伺服器」 |
| CLI 找不到 WiVRn | 系統沒有 `active_runtime.json` | 加上 `XR_RUNTIME_JSON=/usr/share/openxr/1/openxr_wivrn.json`（腳本已自動加） |
| CLI 顯示 `invalid SolarXR frame` 加上用法說明 | 部位參數裡混了全形標點（例如 `12。`） | 只用半形數字和逗號 |
| VRChat 抓不到腳 | 部位參數和實際綁的追蹤器不一致 | 看腳本的部位檢查，修正部位參數 |
| 拖動遊玩區後追蹤器偏離身體 | CLI 讀不到 WiVRn 的 STAGE 偏移（log 有 `libmonado unavailable`） | 用 TW527E 的 CLI fork，並確認 WiVRn 的 runtime JSON 裡有 `MND_libmonado_path` |

### 方案 A（官方 Linux 版）

| 症狀 | 原因 | 解法 |
|---|---|---|
| 腳本說找不到接收器，但 Rebocap 認得到 | 官方版把接收器從 `cdc_acm` 卸下來直接用 USB，`/dev/ttyACM*` 會消失 | 腳本已改成找 USB 裝置 248a:8002 |
| CLI 連上了但一直沒有資料 | Rebocap 還沒切 VR 模式，或正在更新韌體 | 等韌體更新完，切 VR 模式並校準 |
| 腳本警告 20480 已經有程式連著 | 官方 SteamVR 驅動（或另一個 CLI）佔著，這個埠只收一個連線 | 關掉 SteamVR／另一個 CLI |

### 方案 B（Wine／Bottles）

詳細的除錯過程在 [docs/troubleshooting.md](docs/troubleshooting.md)。重點如下：

| 症狀 | 原因 | 解法 |
|---|---|---|
| bridge.exe 顯示 listening，但連進去永遠沒有反應 | 這個 Wine（soda-11.0-10 + ntsync）的阻塞式 socket 呼叫是壞的 | bridge 改用非阻塞 socket 加事件等待（fork 已修） |
| 沒有腰的追蹤器、腳的資料斷斷續續、bridge 一直印 `0 bytes` | 讀 pipe 時回報 0 bytes，訊息被丟掉了 | 先用 `PeekNamedPipe` 確認有資料才讀（fork 已修） |
| bridge CPU 很高、Rebocap 完全不回應 | pipe 的讀取一直處於等待，IO 狀態永遠不會更新 | 同上 |
| 從終端機 `kill` bridge 後，Rebocap 當掉 | wineserver 在處理被殺的程序時 segfault | **永遠不要從主機 kill Wine 程序**，改用 `bridge.quit` |
| Rebocap 顯示「網路校驗失敗，10 秒後退出」然後卡住 | wineserver 已經不在了，Wine 裡的網路全部失效 | 結束整個 Bottle 再重開。不是網路本身的問題 |
| Bottles「停止」一直卡住 | bridge 是 console 程式，Rebocap 也不理會 Windows 的關機訊號 | 先 `bridge.quit`，再手動關 Rebocap 視窗 |

## 授權

本 repo 的腳本和文件採用 MIT License。`bridge.exe` 的程式碼來自 dogesoulseller/rebocap-linux-compat（MIT）。CLI 由 akazdayo/rebocap-wivrn-bridge 開發（我的修改在 TW527E/rebocap-wivrn-bridge），本 repo 只放連結，不包含它的程式碼。Rebocap 是 JXQ 的專有軟體，本 repo 不包含它的任何檔案。
