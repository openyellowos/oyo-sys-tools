# oyo-sys-tools

`oyo-sys-tools` は、**open.Yellow.os における管理・診断を補助するユーティリティ集（Debian パッケージ）**です。  
以下の 3 つのコマンドを含みます：

- `help-hw` : ハードウェア情報の収集ツール  
- `oyo-check` : システム健全性チェックツール  
- `pacup` : パッケージ更新支援ツール  

---

## リポジトリ構成

- `debian/` : Debian パッケージのメタデータ  
- `src/` : 実行スクリプト本体（`help-hw`, `oyo-check`, `pacup`）  
- `man/` : 各コマンドの man ページ  
- `extras/bash-completion/` : Bash 補完スクリプト  
- `extras/zsh-completion/` : Zsh 補完スクリプト  
- `.github/workflows/release.yml` : CI/CD ワークフロー定義  

---

## クライアントPCからの取得方法

oYoのAPTリポジトリからインストール可能です。

```bash
sudo apt update
sudo apt install oyo-sys-tools
```

---

## 使い方

### help-hw
```bash
# ハードウェア情報を収集
help-hw
```
- `journalctl -k`, `lspci`, `lsusb` を実行し、ログを自動でファイルに保存します  
- 出力ファイル名は `Help-HW_コマンド名_日付時刻.log` 形式になります  
- サポート問い合わせ時のログ収集に利用可能  

---

### oyo-check
```bash
# システム健全性をチェック
oyo-check --disk
oyo-check --services
```
- 利用可能なオプション：
  - `--profile` : システムプロファイル確認  
  - `--disk` : ディスク使用量確認  
  - `--security` : セキュリティ関連チェック  
  - `--services` : systemd サービス状態確認  
  - `--apt` : APT パッケージ整合性確認  
  - `--log` : ログ情報確認  
- それぞれ終了コードで成否を返すため、自動化スクリプトにも利用可能  

---

### pacup
```bash
# パッケージを更新
pacup -a   # APT のみ実行
pacup -y   # 対話なしで 'yes' を自動入力
```
- 利用可能なオプション：
  - `-a` : APT のみ実行  
  - `-y` : 対話なしで自動承認  
  - `-h` : ヘルプを表示  

---

## 依存関係

- ランタイム: `bash`, `whiptail`, `pciutils`, `usbutils`  
- 推奨: `apt`, `systemd` など標準ツール  

---

## 仕組みと設定方法（管理者向け）

- `help-hw` は `dmesg`, `lspci`, `lsusb` を呼び出して情報収集を行います。  
- `oyo-check` はシステム状態を複数の観点から検査し、終了コードで成否を返します。  
- `pacup` は APT ラッパーとしてログ保存や安全確認を追加します。  

---

## CI/CD の仕組み（開発者向け）

`oyo-sys-tools` の開発では GitHub Actions を利用した CI/CD を導入しています。  
修正 → tag 作成 → GitHub Actions によるビルド → `apt-repo-infra` によるリポジトリ公開、という流れです。  

```mermaid
flowchart LR
    dev["開発者 (tag vX.Y.Z)"] --> actions["GitHub Actions (oyo-sys-tools)"]
    actions --> release["GitHub Release（成果物添付）"]
    release -->|Run workflow 手動| infra["apt-repo-infra（Run workflow 実行）"]
    infra --> repo["deb.openyellowos.org（APT リポジトリに公開）"]
```

### フロー概要

1. **ソースコード修正**
   ```bash
   git clone https://github.com/openyellowos/oyo-sys-tools.git
   cd oyo-sys-tools
   ```

2. **プログラム修正**
   - `src/help-hw`, `src/oyo-check`, `src/pacup` を編集する。  
   - 必要に応じて README.md も更新する。  

3. **changelog 更新**
   ```bash
   debchange -i
   ```
   - changelog に修正内容を記入する。  

   例:
   ```text
   oyo-sys-tools (1.1-1) kerria; urgency=medium

     * pacup の更新確認処理を改善
     * README.md の利用方法を更新

    -- 開発者名 <you@example.com>  Sun, 31 Aug 2025 20:00:00 +0900
   ```

4. **コミット & push**
   ```bash
   git add .
   git commit -m "修正内容を記述"
   git push origin main
   ```

5. **タグ付与**
   ```bash
   git tag v1.1-1
   git push origin v1.1-1
   ```

6. **GitHub Actions による自動ビルド**
   - タグ push を検知してワークフローが起動。  
   - `.deb` がビルドされ、GitHub Release に添付される。  

7. **APT リポジトリ公開**
   - `apt-repo-infra` の GitHub Actions を **手動で Run workflow** する。  
   - 実際の入力例：  
     - Target environment: `production`  

   - 実行すると apt リポジトリに反映される。  
   - 利用者は以下で最新を取得可能：  
     ```bash
     sudo apt update
     sudo apt install oyo-sys-tools
     ```

---

## 開発環境に必要なパッケージ & ローカルでビルドする手順

### 必要なツールのインストール
```bash
sudo apt update
sudo apt install -y devscripts build-essential debhelper lintian
```

### deb-src を有効にする
1. `/etc/apt/sources.list` を編集し、`deb-src` 行を有効化。  
2. 更新：
   ```bash
   sudo apt update
   ```

### ビルド依存の導入
```bash
sudo apt-get build-dep -y ./
```

### ローカルビルド
```bash
dpkg-buildpackage -us -uc -b
```
- 生成物: `../oyo-sys-tools_*_all.deb`

### テストインストール / アンインストール
```bash
sudo apt install ./../oyo-sys-tools_*_all.deb
sudo apt remove oyo-sys-tools
```

### クリーン
```bash
fakeroot debian/rules clean
```

---

## 注意事項

- **必ず changelog を更新すること**  
- **バージョン番号は changelog, git tag, GitHub Release を揃えること**  
- **依存関係を追加・削除した場合は debian/control を修正すること**  

