[简体中文](README.md) · [繁體中文](README.zh-TW.md) · [English](README.en.md) · **日本語**

# Rebocap × Linux × WiVRn フルボディトラッキング（VRChat）

Linux で **Rebocap** の慣性式フルボディトラッキングを **VRChat** につなぎます。SteamVR は使わず **WiVRn + xrizer** を通し、
ヘッドセットは Meta Quest です。

Rebocap には公式 Linux 版が出ましたが、付属するのは **SteamVR ドライバー**だけです。SteamVR を使うなら公式版だけで十分で、
このリポジトリは必要ありません。このリポジトリは **SteamVR ではなく WiVRn を使う**人向けです。CLI で Rebocap のトラッカーを
SolarXR のトラッカーに変換し、WiVRn に送ります。

実際の環境、ゼロからのインストール手順、普段の使い方、途中でぶつかった問題とその解決策をまとめています。

## 2 つの構成

| | **A. 公式 Linux 版（おすすめ）** | B. Windows 版を Wine（Bottles）で動かす |
|---|---|---|
| Rebocap | 公式のネイティブ Linux アプリ | Bottles に入れた Windows 版 |
| レシーバー | 公式アプリが USB を直接使う | Wine の COM ポート + setupapi パッチ |
| CLI との接続 | Rebocap 自身が TCP `127.0.0.1:20480` で待ち受け | `bridge.exe` が Wine 内で名前付きパイプを TCP `127.0.0.1:36850` に変換 |
| 必要なもの | Rebocap、CLI、このリポジトリ | Rebocap、Bottles、rebocap-wine パッチ、bridge.exe、CLI、このリポジトリ |
| トラッカーのファームウェア | 公式アプリで更新できる（v15 から v17 にしたら安定した気がします） | Wine 上では更新しない |

構成 A は Wine・パッチ・bridge.exe の 3 層がなくなり、起動も終了もずっと簡単です。公式版があなたの PC で動かない場合を除き、A を使ってください。
構成 B は予備として、また最初にどうつないだかの記録として残しています。

```
構成 A：
Quest ヘッドセット → WiVRn → CLI ⇄ Rebocap（ネイティブ Linux、:20480）
Rebocap トラッカー → Rebocap → CLI → SolarXR → WiVRn → xrizer → VRChat

構成 B：
Quest ヘッドセット → WiVRn → CLI ⇄ bridge.exe（Wine、:36850）⇄ Rebocap（Wine）
Rebocap トラッカー → Rebocap → bridge.exe → CLI → SolarXR → WiVRn → xrizer → VRChat
```

起動スクリプト `rebocap-fbt` はどちらにも対応し、自動で判別します。20480 が待ち受けていれば構成 A、なければ 36850 の bridge.exe を探します。

## 動作確認環境

| 項目 | バージョン／設定 |
|---|---|
| OS | Arch Linux（Omarchy、Hyprland）、NVIDIA RTX 3070 |
| ヘッドセット | Meta Quest、WiVRn（Wi-Fi または USB） |
| WiVRn | wivrn-server / wivrn-dashboard 26.9 |
| xrizer | xrizer-git 0.6.r6（rebocap-wivrn-bridge の作者は 0.5 で確認。0.6 でも問題なし） |
| Rebocap（構成 A） | 公式 Linux 版 `rebocap-linux-preview-0.1-1`、トラッカーのファームウェア v17 |
| Rebocap（構成 B） | Windows 版を Flatpak 版 **Bottles** で実行、runner は soda-11.0-10（ntsync あり）、トラッカーのファームウェア v15 |
| トラッカー | 普段は 10 点：腰、胸、両上腿、両下腿、両足、両上腕（前腕に付けていた分を下腿に回しました）。テストでは 6 点：腰、両上腿、両下腿、胸 |

## 使っているプロジェクト

| プロジェクト | 用途 | 構成 |
|---|---|---|
| [TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) | CLI（`monad2steamvr`）：WiVRn と Rebocap の間で頭とトラッカーの姿勢を中継し、SolarXR でトラッカーを公開します。[akazdayo/rebocap-wivrn-bridge](https://github.com/akazdayo/rebocap-wivrn-bridge) のフォークで、WiVRn のプレイエリア移動の補正（libmonado）を加え、トラッカー姿勢の 10 ms ポーリング待ち 2 段をなくしました（平均で約 10 ms の遅延削減） | A、B |
| このリポジトリ | 起動スクリプト `rebocap-fbt`、VRChat のボタン割り当て生成スクリプト、この手順書 | A、B |
| [TW527E/rebocap-wine](https://github.com/TW527E/rebocap-wine) | Wine 上の Rebocap がレシーバーを認識できるようにする setupapi パッチ | B |
| [TW527E/rebocap-linux-compat](https://github.com/TW527E/rebocap-linux-compat) | `bridge.exe`：Wine 内で Rebocap の名前付きパイプを TCP に変換します。[dogesoulseller/rebocap-linux-compat](https://github.com/dogesoulseller/rebocap-linux-compat) のフォークで、Bottles/ntsync 向けの修正入り | B |

## インストール（ゼロから）

先に「共通」の手順を済ませてから、選んだ構成 A か B に進んでください。

### 共通 1. WiVRn と xrizer

WiVRn の説明に従って wivrn-server、wivrn-dashboard、xrizer をインストールし、フルボディトラッキングなしで VRChat が普通に動くことを確認します。

### 共通 2. CLI

```sh
sudo pacman -S rust openxr pkgconf
git clone https://github.com/TW527E/rebocap-wivrn-bridge ~/Projects/rebocap-wivrn-bridge
cd ~/Projects/rebocap-wivrn-bridge && cargo build --release
```

### 共通 3. 起動スクリプト

```sh
git clone https://github.com/TW527E/rebocap-linux-wivrn-fbt ~/Projects/rebocap-linux-wivrn-fbt
ln -s ~/Projects/rebocap-linux-wivrn-fbt/scripts/rebocap-fbt ~/.local/bin/rebocap-fbt
```

スクリプトは CLI（`monad2steamvr`）を次の順で探します：`REBOCAP_CLI`、`PATH`、このリポジトリと同じ階層の `rebocap-wivrn-bridge`、
そして `~/Projects`、`~/Downloads`、`~/src`、`~/` の下の `rebocap-wivrn-bridge`。別の場所に置いた場合やほかのパスが違う場合は、
環境変数 `REBOCAP_CLI`、`REBOCAP_BOTTLE`、`REBOCAP_RECEIVER`、`REBOCAP_XR_RUNTIME_JSON` で上書きできます。ポートは
`REBOCAP_NATIVE_PORT`（既定 20480）と `REBOCAP_WINE_PORT`（既定 36850）で変更できます。

### 構成 A：公式 Linux 版（おすすめ）

1. [公式のインストール手順](https://doc.rebocap.com/zh-Hans/docs/tutorial/software_install)に従って Linux 版のインストーラーをダウンロードし、実行します。
   ```sh
   chmod +x rebocap-linux-preview-*.run
   ./rebocap-linux-preview-*.run
   ```
   私は `~/.local/opt/rebocap` に入れました。アプリメニューから Rebocap を開くか、`~/.local/opt/rebocap/rebocap-ui` を実行します。
2. レシーバーへのアクセス権を設定します。公式のスクリプトが `/etc/udev/rules.d/70-rebocap.rules` を書き込み、
   ユーザーを `dialout`／`uucp`／`plugdev` のどれかのグループ（Arch では `uucp`）に追加します。
   ```sh
   sudo bash ~/.local/opt/rebocap/linux/install_linux_permissions.sh --install --user "$USER"
   ```
   そのあと再ログインし、レシーバーを挿し直してください。
3. Rebocap を開き、トラッカーのファームウェア更新を求められたら更新します。**更新中はレシーバーを抜かず、アプリも閉じないでください。**
4. 公式の SteamVR ドライバー（`data/rebocap_driver`）は**入れないでください**。SteamVR は使いませんし、ポート 20480 は同時に
   1 つしか接続を受け付けないため、SteamVR ドライバーがつながっていると CLI が接続できません。

Bottles、rebocap-wine、bridge.exe はどれも不要です。

> 公式版は実行中にレシーバーを `cdc_acm` から外して USB で直接アクセスするので、その間 `/dev/ttyACM*` と
> `/dev/serial/by-id/...` は消えます。これは正常な動作で、スクリプトは USB デバイス 248a:8002 を直接探すようにしています。

### 構成 B：Windows 版を Wine（Bottles）で動かす

#### B1. レシーバーの権限

再起動後も有効にするには、ユーザーが `/dev/ttyACM*` を所有するグループ（Arch では `uucp`）に入っている必要があります。

```sh
sudo usermod -aG uucp "$USER"   # 実行後に再ログイン
```

私は udev ルール `/etc/udev/rules.d/70-rebocap.rules` も追加しましたが、なくてもかまいません。

```
SUBSYSTEM=="tty", ATTRS{idVendor}=="248a", ATTRS{idProduct}=="8002", MODE="0660", TAG+="uaccess"
```

#### B2. Rebocap を Bottles に入れる

1. Wine がシリアルポートにアクセスできるよう、Bottles にデバイスへのアクセス権を与えます：`flatpak override --user --device=all com.usebottles.bottles`
2. ボトルを作成し（私の名前は `Rebocap`、win64）、Rebocap をインストールします。
3. [rebocap-wine](https://github.com/TW527E/rebocap-wine) の setupapi パッチを当て、Rebocap がレシーバーを列挙できるようにします。
4. ボトルの `dosdevices` でレシーバーを COM ポートに割り当てます。例：`com33 -> /dev/ttyACM0`

#### B3. bridge.exe

```sh
git clone https://github.com/TW527E/rebocap-linux-compat
cd rebocap-linux-compat
x86_64-w64-mingw32-gcc -O2 -static -o bridge.exe bridge/bridge.c -lws2_32   # mingw-w64-gcc が必要
cp bridge.exe ~/.var/app/com.usebottles.bottles/data/bottles/bottles/Rebocap/drive_c/Program\ Files\ \(x86\)/Rebocap/
```

自分でビルドしたくない場合は、[Releases](https://github.com/TW527E/rebocap-linux-compat/releases/latest) から `bridge.exe` をダウンロードできます。

`install.py` の実行や SteamVR ドライバーの登録は**不要**です。

Bottles の GUI で `bridge.exe` をそのボトルのプログラムとして追加しておくと、毎回の起動が楽です。

### （任意）右スティック押し込み＝ジャンプ

コントローラーの A ボタンが壊れている場合、右スティックの押し込みでもジャンプできるようにできます。xrizer 用の割り当てファイルを生成します。

```sh
./xrizer-bindings/make-vrchat-bindings.py      # ~/.config/xrizer-bindings/oculustouch.json に書き込み
```

そして VRChat の Steam 起動オプションの先頭に次を追加します。

```
XRIZER_CUSTOM_BINDINGS_DIR=/home/<あなた>/.config/xrizer-bindings
```

右スティック押し込みの本来の機能は「Udon Stick Click」で、使うワールドはごく一部です。A ボタンでも今までどおりジャンプできます。
VRChat を更新したら生成スクリプトをもう一度実行してください。元に戻すには、この起動オプションを消すだけです。

## 普段の使い方

**フルトラなしで遊ぶ**：いつもどおりです。WiVRn を起動 → ヘッドセットを接続 → VRChat を起動。Rebocap もスクリプトも不要です。

**フルボディトラッキング**では**順番が大事**です。

1. **WiVRn**（wivrn-dashboard）を起動し、サーバーが動いていることを確認します。**ヘッドセットはまだ接続しません。**
2. **Rebocap** を起動します。
   - 構成 A：公式 Linux 版を開くだけ。
   - 構成 B：Bottles で Rebocap を起動し、続けて bridge.exe を起動。
3. 端末で実行します。
   ```sh
   rebocap-fbt          # 10 点（既定）：腰、胸、両上腿、両下腿、両足、両上腕。構成は自動判別
   rebocap-fbt 12       # 12 点：10 点に両前腕を追加
   rebocap-fbt 6        # 6 点
   rebocap-fbt 10 --usb # 準備ができたら USB でヘッドセットを接続
   rebocap-fbt --wine   # 構成 B を強制（--native は構成 A を強制）
   ```
   スクリプトのメッセージは繁体字中国語です。最初に、使っている構成が表示されます。例：
   `模式：官方 Linux 版 Rebocap（127.0.0.1:20480）`（モード：公式 Linux 版 Rebocap）
4. 「**SolarXR 準備好了**」（SolarXR の準備完了。デスクトップ通知も出ます）が表示されたら、**ヘッドセットを接続**します。
5. Rebocap を **VR モード**に切り替え、ヘッドセットを着けて**キャリブレーション**します。
6. Steam から **VRChat** を起動し、クイックメニューで **Calibrate FBT** を押します。

スクリプトは部位のチェックを表示します。

```
有資料的部位：0 1 2 3 4 5 6 7 9 10
  ⚠ 已發布但沒有資料：…
  ⚠ 有資料但沒有發布：…
```

1 行目はデータが届いている部位です。⚠ の行は「公開しているのにデータがない」「データがあるのに公開していない」部位で、
部位の指定が実際に着けているトラッカーと合っていません。部位の指定を直してください。

**終了**：

1. スクリプトの端末で **Ctrl+C** を押します。スクリプトが CLI を止めます。構成 B では `bridge.quit` で bridge も終了させます（`--keep-bridge` を付けると終了させません）。
2. **Rebocap** のウィンドウを閉じます。
3. WiVRn はいつもどおりに扱います。

**重要**：CLI を再起動したり部位を変えたりしたら、ヘッドセットを**再接続**し、VRChat も**再起動**してください。WiVRn はヘッドセット接続時にだけトラッカー一覧を読み込みます。

### 部位番号

| 番号 | 部位 | 番号 | 部位 |
|---|---|---|---|
| 0 | 腰 | 7 | 胸 |
| 1 / 2 | 左／右上腿 | 9 / 10 | 左／右上腕 |
| 3 / 4 | 左／右下腿 | 11 / 12 | 左／右前腕 |
| 5 / 6 | 左／右足 | | |

VRChat はキャリブレーション時、トラッカーの**位置**で腰・足・膝・胸・肘に割り当てるので、6 点セットの下腿トラッカーを足として使えます。

CLI の既定値 `0,5,6` は足にトラッカーを着けるセット向けです。6 点セットには足の実機トラッカーがなく、5 と 6 は Rebocap が作る
仮想ノードで、データは短時間しか届きません。既定値のままだと VRChat は足を認識できません。

### USB 接続

1. ヘッドセットで開発者モードを有効にし、この PC からの USB デバッグを許可します。許可のダイアログはヘッドセットの中に出るので、着けないと見えません。
2. `adb devices` を実行し、状態が `device` になっていることを確認します。`unauthorized` なら、まだヘッドセットで許可していません。
3. `rebocap-fbt --usb` を使うか、wivrn-dashboard で「USB で接続」を押します。

手動の場合：

```sh
adb reverse tcp:9757 tcp:9757
adb shell am start -a android.intent.action.VIEW -d "wivrn+tcp://127.0.0.1:9757" org.meumeu.wivrn
```

## 更新

```sh
cd ~/Projects/rebocap-wivrn-bridge && git pull && cargo build --release   # CLI
cd ~/Projects/rebocap-linux-wivrn-fbt && git pull                         # 起動スクリプト（シンボリックリンクが新しい版を指します）
```

公式 Linux 版 Rebocap は、インストール先フォルダの `maintenancetool` で更新するか、新しいインストーラーで入れ直してください。

## 遅延

CLI に届くまでの差は 2 つの構成でごくわずかです。構成 A では Wine の名前付きパイプ、bridge.exe のポーリング、Wine のシリアルエミュレーションがなくなり、
おおよそ 1〜3 ms 短くなります。大きな差は CLI 側にあります。元の CLI ではトラッカーの姿勢が 10 ms 間隔のポーリングループを 2 つ通り、
平均で約 10 ms（最大で約 20 ms）遅れ、更新間隔も不揃いでした。[TW527E/rebocap-wivrn-bridge](https://github.com/TW527E/rebocap-wivrn-bridge) では
受け取ったらすぐ WiVRn に送ります。

## ハマったところ

### どちらの構成でも起こるもの

| 症状 | 原因 | 対処 |
|---|---|---|
| CLI：`Failed to connect to monado service process` | WiVRn サーバーが動いていない（ダッシュボードを開いただけではサーバーは起動しません） | ダッシュボードで「サーバーを起動」を押す |
| CLI が WiVRn を見つけられない | システムに `active_runtime.json` がない | `XR_RUNTIME_JSON=/usr/share/openxr/1/openxr_wivrn.json` を付ける（スクリプトは自動で付けます） |
| CLI が `invalid SolarXR frame` と使い方を表示する | 部位の指定に全角の句読点が混じっている（例：`12。`） | 半角の数字とカンマだけを使う |
| VRChat が足を認識しない | 部位の指定が実際に着けているトラッカーと合っていない | スクリプトの部位チェックを見て、部位の指定を直す |
| プレイエリアを動かすとトラッカーが体からずれる | CLI が WiVRn の STAGE オフセットを読めない（ログに `libmonado unavailable`） | TW527E の CLI フォークを使い、WiVRn のランタイム JSON に `MND_libmonado_path` があるか確認する |

### 構成 A（公式 Linux 版）

| 症状 | 原因 | 対処 |
|---|---|---|
| スクリプトがレシーバーが見つからないと言うが、Rebocap は認識している | 公式版がレシーバーを `cdc_acm` から外して USB を直接使うため、`/dev/ttyACM*` が消える | スクリプトは USB デバイス 248a:8002 を探すように対応済み |
| CLI はつながるがデータが来ない | Rebocap がまだ VR モードでない、またはファームウェア更新中 | 更新が終わるのを待ち、VR モードにしてキャリブレーションする |
| スクリプトが 20480 に別のプログラムが接続中だと警告する | 公式の SteamVR ドライバー（または別の CLI）がポートを使っている。このポートは 1 接続のみ | SteamVR／別の CLI を閉じる |

### 構成 B（Wine／Bottles）

詳しいデバッグの記録は [docs/troubleshooting.md](docs/troubleshooting.md)（繁体字中国語）にあります。要点は次のとおりです。

| 症状 | 原因 | 対処 |
|---|---|---|
| bridge.exe は listening と表示するが、接続しても反応がない | この Wine（soda-11.0-10 + ntsync）ではブロッキングのソケット呼び出しが壊れている | bridge をノンブロッキングのソケットとイベント待ちに変更（フォークで修正済み） |
| 腰のトラッカーがない、足のデータが途切れる、bridge が `0 bytes` と出し続ける | パイプの読み取りが 0 バイトを返し、メッセージが失われる | `PeekNamedPipe` でデータを確認してから読む（フォークで修正済み） |
| bridge の CPU 使用率が高く、Rebocap が反応しない | パイプの読み取りが保留のままで、I/O ステータスが更新されない | 同上 |
| 端末から bridge を `kill` したら Rebocap が落ちた | 終了させたプロセスの処理中に wineserver がセグフォルトする | **ホストから Wine のプロセスを kill しない**。`bridge.quit` を使う |
| Rebocap が「ネットワーク認証に失敗、10 秒後に終了」と表示して固まる | wineserver がもういないため、Wine 内のネットワークがすべて使えない | ボトルごと終了して起動し直す。ネットワーク自体の問題ではない |
| Bottles の「停止」が終わらない | bridge はコンソールプログラムで、Rebocap も Windows の終了メッセージを無視する | 先に `bridge.quit`、そのあと Rebocap のウィンドウを手で閉じる |

## ライセンス

このリポジトリのスクリプトと文書は MIT License です。`bridge.exe` のコードは dogesoulseller/rebocap-linux-compat（MIT）由来です。
CLI は akazdayo/rebocap-wivrn-bridge が開発したもので（私の変更は TW527E/rebocap-wivrn-bridge にあります）、このリポジトリはリンクを
置いているだけで、そのコードは含みません。Rebocap は JXQ のプロプライエタリソフトウェアで、このリポジトリにはそのファイルを一切含みません。
