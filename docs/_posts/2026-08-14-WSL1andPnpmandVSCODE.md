---
title: "WSL1 で Node.js / pnpm を使う際に発生する、次のような断続的なファイル操作エラー"
date: 2026-08-16
classes: wide
---

# WSL1 開発ガイド

## このガイドの目的

WSL1 で Node.js / pnpm を使う際に発生する、次のような断続的なファイル操作エラーを診断・回避するためのガイドです。

```text
ERR_PNPM_EACCES EACCES: permission denied, rename
.../node_modules/.pnpm/<package>/node_modules/<scope>/<name>_tmp_<pid>
-> .../node_modules/.pnpm/<package>/node_modules/<scope>/<name>
```

今回の環境は ARM64 Windows on Parallels on macOS で、ネストされた仮想化の制約から WSL2 化が難しく、WSL1 を継続利用する前提です。

## 結論

今回のエラーは、Linux の所有権やパーミッション不足ではなく、WSL1 のファイルシステム互換層と Windows 側のファイル監視が競合したことが原因でした。

主な対策は次の2点です。

1. 対象の WSL ディストリビューションを Microsoft Defender の監視対象から除外する
2. VS Code のユーザー設定で `node_modules` をファイル監視対象から除外する

対策後、VS Code を起動した状態で `functions` のクリーンインストール（926パッケージ）と TypeScript ビルドを2回連続で実行し、どちらも成功しました。

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

> 注意: ディストリビューション全体の除外は保護範囲を広く失います。信頼できる開発環境だけに適用し、WSL 内へ取得するコードやバイナリの安全性は別途管理してください。

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

### まだ再発する場合の診断設定

一時的にワークスペース全体の監視を止めると、VS Code watcher が原因か切り分けられます。

```json
{
  "files.watcherExclude": {
    "**/*": true
  }
}
```

これは通常運用には広すぎる設定です。診断後は `node_modules` の除外へ戻します。

## pnpm エラーからの復旧

### 基本手順

1. 同時に動いている `pnpm`、`npm`、ビルド、開発サーバーを停止する
2. 対象パッケージの `node_modules` を削除する
3. GHA と同じ Node.js と pnpm を選択する
4. frozen lockfile で再インストールする
5. ビルドする

例:

```bash
cd /path/to/parent-repository
pwd

source "$HOME/.nvm/nvm.sh"
nvm use 22

cd path/to/functions
pwd

rm -rf node_modules
corepack pnpm install --frozen-lockfile
corepack pnpm build
```

### やらないほうがよいこと

- `sudo pnpm install` を実行する
- 原因確認なしに再帰的な `chmod 777` を行う
- pnpm のバージョンをロックファイルや `packageManager` と無関係に更新する
- エラーが出るたびに pnpm store 全体を削除する
- 複数の対象を一括インストールし、どこで失敗したか分からなくする

`sudo` でインストールすると、本当のWSL1競合に加えて root 所有ファイルの問題まで作ってしまいます。

## 複数Nodeバージョンで個別ビルドするコツ

### 毎回、現在位置を確認する

長い検証ではターミナルの現在位置を仮定しないようにします。

```bash
cd /absolute/path/to/parent-repository
pwd
```

対象へ移動した後も確認します。

```bash
cd path/to/package
pwd
```

### Node.js と pnpm の実体を確認する

```bash
node --version
corepack pnpm --version
```

`package.json` に `packageManager: "pnpm@..."` がある場合、`corepack pnpm` を使うと指定バージョンを利用できます。

### ワークスペースではフィルターを使う

pnpm workspace のルートで単純に `pnpm install` を実行すると、全パッケージが対象になります。個別確認が必要な場合はフィルターを使います。

```bash
corepack pnpm --filter @scope/api install --frozen-lockfile
corepack pnpm --filter @scope/api build
```

対象を切り替える前に共有 `node_modules` を削除すると、前のNodeバージョンや別パッケージの依存関係が混ざるのを防げます。

```bash
rm -rf node_modules apps/api/node_modules apps/web/node_modules
```

## 日常運用のチェックリスト

### セットアップ時

- [ ] `wsl.exe -l -v` でWSL1であることを把握する
- [ ] nvmで必要なNode.jsバージョンを用意する
- [ ] Corepackでリポジトリ指定のpnpmを使う
- [ ] Defender除外は対象ディストリビューションのBasePathだけに限定する
- [ ] VS Codeユーザー設定で `node_modules` をwatcher除外する
- [ ] `sudo` でNode.js依存関係をインストールしない

### ビルド時

- [ ] 親リポジトリへ絶対パスで移動する
- [ ] `pwd` を確認する
- [ ] GHAのNode.jsバージョンを確認する
- [ ] `node --version` と `corepack pnpm --version` を確認する
- [ ] 対象ごとに依存関係をインストールする
- [ ] `--frozen-lockfile` を使う
- [ ] 対象ごとにビルド結果を記録する

### EACCES 再発時

- [ ] 失敗対象が毎回変わるか確認する
- [ ] 所有者不一致がないか確認する
- [ ] 同じ場所の手動renameを試す
- [ ] Defender除外が残っているか確認する
- [ ] VS CodeをReload Windowする
- [ ] VS Codeを完全終了し、通常のUbuntuターミナルで再現するか確認する
- [ ] Windows Search、TortoiseGit、OneDriveなど他の監視プロセスを確認する

## 参考情報

- Microsoft WSL Issue: [WSL pins opened directories #1529](https://github.com/microsoft/WSL/issues/1529)
- Microsoft WSL Issue: [EACCES when renaming folder that is being watched from nodejs #3395](https://github.com/microsoft/WSL/issues/3395)
- pnpm Issue: [pnpm install failed on WSL with EACCES #6155](https://github.com/pnpm/pnpm/issues/6155)
- Microsoft Defender: [Configure custom exclusions](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-exclusions-configure)
- Microsoft Defender: [Exclusions overview](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-exclusions-overview)
- Microsoft WSL: [Working across file systems](https://learn.microsoft.com/en-us/windows/wsl/filesystems)
