---
title: "いまだにWSL1を使う：Node.js / pnpm を使う際に発生する断続的なファイル操作エラー"
date: 2026-08-16
classes: wide
---

# WSL1 開発ガイド

## このガイドの目的

WSL1 で Node.js / pnpm を使う際に発生する、次のような断続的なファイル操作エラーを診断・回避するためのガイドです。Cygwinよりも本物のWSL1ですが、ファイルシステムがWindowsなのでWSL2程には切り離されておらず、いろいろあるようです。

```text
ERR_PNPM_EACCES EACCES: permission denied, rename
.../node_modules/.pnpm/<package>/node_modules/<scope>/<name>_tmp_<pid>
-> .../node_modules/.pnpm/<package>/node_modules/<scope>/<name>
```

## Why WSL1?

今回の環境は ARM64 Windows on Parallels on macOS で、ネストされた仮想化の制約から WSL2 化が難しく、WSL1 を継続利用する前提です。

なぜピカピカのmacでWindowsなどつかうのかというと。。。VMで用途ごとに厳しく環境分離してセキュリティを高めているのですが、macOS VM on macの構成だとTrackpadがマウスとして認識され、二本指スクロールがホイールのチャタリング扱いされて死ぬからです。Windows11にはこのような問題は存在しません。

Boot Campの初期にはMacBookのTrackpadもWindowsから見るとかなり素朴な入力デバイス扱いだったのですが、その後Windows側にはPrecision Touchpadという標準基盤が整備され、最終的にはAppleもBoot Campでこれに対応。無事Windowsからも「ちゃんとしたTrackpad」として扱われるようになりました。。多分、この伝統が生きているのです！

一方UbuntuのARM64版はまだ動くソフトが少なく。。。（Chromeすら存在していません！）

つまるところ、macに最適なOSはWindows。Microsoftのドライバ拡充とソフトウェア互換性の執念でまたしてもBootcamp以来のこのジンクスが証明されるためには、私はこれを突破せざるを得ませんでした。

## 結論

今回のエラーは、Linux の所有権やパーミッション不足ではなく、WSL1 のファイルシステム互換層と Windows 側のファイル監視が競合したことが原因でした。

主な対策はファイルアクセスの頻度を減らすこと

1. 対象の WSL ディストリビューションを Microsoft Defender の監視対象から除外する
2. VS Code のユーザー設定で `node_modules` をファイル監視対象から除外する

⇒ ただ、Defenderの監視がなくなるのは看過し難いので、Defenderの方は対策としては結局実施していません。

## なぜ EACCES が発生するのか

### pnpm のインストール処理

pnpm はパッケージを直接最終ディレクトリへ書き込まず、概ね次の順序で配置します。

1. `<package>_tmp_<pid>` のような一時ディレクトリへ展開する
2. 展開完了後、一時ディレクトリを最終名へ `rename` する
3. `rename` に成功した時点で完成したパッケージとして扱う

この方法は、途中までしか書かれていないパッケージを利用側から見えなくするための一般的なアトミック配置です。

### WSL1 の制約

通常の Linux では、ディレクトリ内のファイルが他プロセスから開かれていても、そのディレクトリを rename できます。

一方、WSL1 は Windows のファイルシステムやファイルハンドルの制約を受けます。Microsoft Defender、Windows Search、VS Code のファイルウォッチャーなどが、作成直後のファイルを一時的に開いていると、親ディレクトリの rename が `EACCES` になることがあります。

そのため、次の特徴が現れます。

- `chmod` や `chown` をしても直らない
- 同じコマンドを再実行すると、別のパッケージで失敗する
- 再実行するたびにインストール済みパッケージ数が増える
- エラー後に調べると、対象パスの所有者と権限は正常
- 同じディレクトリで手動の `mv` は成功する
- VS Code を閉じると成功することがある

## 原因の切り分け

### 1. WSL バージョンとファイルシステムを確認する

```bash
uname -a
cat /proc/version
findmnt -T "$HOME" -o TARGET,SOURCE,FSTYPE,OPTIONS
```

WSL1 の代表的な特徴は次のとおりです。

- カーネル表示に `4.4.0-...-Microsoft` が含まれる
- ファイルシステムが `wslfs`

Windows 側から確認する場合は、PowerShell または CMD で次を実行します。

```cmd
wsl.exe -l -v
```

### 2. 所有権異常を確認する

```bash
find node_modules -xdev \( ! -user "$(id -u)" -o ! -group "$(id -g)" \) -print
```

何も表示されなければ、異なるユーザーや `sudo` で作成されたファイルはありません。

### 3. 同じ場所で rename できるか確認する

エラーが発生した親ディレクトリで、一時ディレクトリを作成して移動します。

```bash
mkdir .rename-probe
mv .rename-probe .rename-probe-moved
rmdir .rename-probe-moved
```

これが成功するのに pnpm だけが断続的に失敗する場合、恒常的なパーミッション不足ではなく、一時的なファイルハンドル競合が疑われます。

### 4. Windows 側の監視プロセスを確認する

PowerShell で次を実行します。

```powershell
Get-Process |
  Where-Object { $_.ProcessName -match 'MsMpEng|Sense|SearchIndexer|Code|TGitCache|OneDrive' } |
  Select-Object ProcessName, Id, Path
```

代表的な候補は次のとおりです。

- `MsMpEng`: Microsoft Defender
- `SearchIndexer`: Windows Search
- `Code`: VS Code
- `TGitCache`: TortoiseGit
- `OneDrive`: OneDrive 同期

## 解決策1: Microsoft Defender から WSL1 を除外する

> 注意: ディストリビューション全体の除外は保護範囲を広く失います。信頼できる開発環境だけに適用し、WSL 内へ取得するコードやバイナリの安全性は別途管理してください。私は結局この対策候補を採用していません

### 対象ディストリビューションの BasePath を取得する

通常の PowerShell で次を実行します。

```powershell
Get-ChildItem HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss |
  ForEach-Object { Get-ItemProperty $_.PSPath } |
  Select-Object DistributionName, BasePath
```

出力例:

```text
DistributionName BasePath
---------------- --------
Ubuntu-24.04     C:\Users\<user>\AppData\Local\wsl\{GUID}
```

対象ディストリビューションの `BasePath` だけを除外します。複数の Ubuntu がある場合、親の `C:\Users\<user>\AppData\Local\wsl` 全体を除外しないでください。

### 除外を追加する

管理者として PowerShell を開き、次を実行します。

```powershell
Add-MpPreference -ExclusionPath 'C:\Users\<user>\AppData\Local\wsl\{GUID}'
```

管理者 CMD から実行する場合:

```cmd
powershell.exe -NoProfile -Command "Add-MpPreference -ExclusionPath 'C:\Users\<user>\AppData\Local\wsl\{GUID}'"
```

`Add-MpPreference` は PowerShell のコマンドレットなので、CMD へ直接入力すると「認識されていません」と表示されます。

### 除外を確認する

管理者 PowerShell で次を実行します。

```powershell
(Get-MpPreference).ExclusionPath
```

### 除外を削除する

不要になった場合は、管理者 PowerShell で削除します。

```powershell
Remove-MpPreference -ExclusionPath 'C:\Users\<user>\AppData\Local\wsl\{GUID}'
```

### Windows から rootfs を直接編集しない

`BasePath` は除外設定の指定や診断にのみ使います。エクスプローラーや Windows アプリから `rootfs` 内のファイルを直接編集・移動・削除しないでください。

WSL 内のファイル操作は次のいずれかで行います。

- WSL のシェル
- VS Code Remote - WSL
- `\\wsl.localhost\<DistributionName>\...` の公式共有経由

## 解決策2: VS Code のファイル監視を除外する

VS Code のユーザー設定 JSON に次を追加します。

```json
{
  "files.watcherExclude": {
    "**/node_modules/**": true,
    "**/node_modules/.pnpm/**": true
  }
}
```

設定画面から `Preferences: Open User Settings (JSON)` を実行して編集します。

ワークスペース設定ではなくユーザー設定へ入れると、WSL1 で開くすべてのプロジェクトに適用できます。

変更後は `Developer: Reload Window` を実行します。

## pnpm エラーからの復旧

### 基本手順

1. 同時に動いている `pnpm`、`npm`、ビルド、開発サーバーを停止する
2. 対象パッケージの `node_modules` を削除する
3. ビルドする

## 参考情報

- Microsoft WSL Issue: [WSL pins opened directories #1529](https://github.com/microsoft/WSL/issues/1529)
- Microsoft WSL Issue: [EACCES when renaming folder that is being watched from nodejs #3395](https://github.com/microsoft/WSL/issues/3395)
- pnpm Issue: [pnpm install failed on WSL with EACCES #6155](https://github.com/pnpm/pnpm/issues/6155)
- Microsoft Defender: [Configure custom exclusions](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-exclusions-configure)
- Microsoft Defender: [Exclusions overview](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-exclusions-overview)
- Microsoft WSL: [Working across file systems](https://learn.microsoft.com/en-us/windows/wsl/filesystems)
