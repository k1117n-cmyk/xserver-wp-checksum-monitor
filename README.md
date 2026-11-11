# Xserver WordPress チェックサム監視

Xserver 上の WordPress を対象に、WordPress コアファイルと公式プラグインの
チェックサム検証を毎日実行するための軽量監視ツールです。

異常がなければ「問題なし」、不一致があれば対象サイトと WP-CLI の検証結果を
メールまたは Webhook に通知します。

## 対象環境

- Xserver のレンタルサーバー
- SSH が利用できること
- サーバー上で `wp` コマンド、つまり WP-CLI が利用できること
- WordPress が `$HOME` 配下に設置されていること

## ファイル構成

配布ファイル:

```text
VERSION
install.sh
uninstall.sh
wp-checksums-monitor.sh
README.md
LICENSE.txt
DISCLAIMER.txt
```

インストール後のサーバー上の構成:

```text
~/checksums/
  VERSION
  wp-checksums-monitor.sh
  uninstall.sh
  sites.conf
  cron.checksums
  checksum-monitor.log
```

## インストール

`install.sh` と `wp-checksums-monitor.sh` を同じディレクトリに置き、SSH で
Xserver にログインして実行します。

まず診断だけ行う場合:

```sh
sh install.sh --check
```

診断モードではインストールは行わず、WP-CLI、crontab、sendmail、
WordPress の検出状況を確認します。

インストール予定内容を確認する場合:

```sh
sh install.sh --dry-run
```

ドライランでは、検出されたWordPress、作成予定ファイル、登録予定Cronを表示します。
ファイル作成や crontab の変更は行いません。

インストールする場合:

```sh
sh install.sh
```

インストーラーは次の内容を行います。

- `$HOME` 配下から WordPress を自動検出
- 検出したサイト一覧を表示し、監視対象を選択
- サブディレクトリやテスト環境の WordPress も検出されることがあるため、監視対象を確認して選択
- 通知先メールアドレスを入力
- 毎日チェックする時刻を入力、初期値は `7`
- メール通知モードを選択
- `~/checksums` を作成
- `sites.conf` を自動生成
- `wp-checksums-monitor.sh` を配置
- 既存の crontab を壊さず、監視用の管理ブロックを追加
- 初回チェックを実行
- 必要に応じて通知テストメールを送信

## インストール時の入力例

```text
通知先メールアドレス: user@example.com
毎日チェックする時刻, 0-23 [7]:

メール通知モード:
  1) 正常時も異常時も毎回メールする
  2) 異常時だけメールする
通知モードを選択してください, 1 or 2 [2]:

検出したWordPress:
  1) example.com: /home/example/example.com/public_html/wp
監視対象を選択してください。all または番号をカンマ区切りで指定 [all]:

この内容で監視を開始しますか？ y/N [N]: y
通知テストメールを送信しますか？ y/N [N]:
```

## 通常の実行

インストール後は、次のコマンドで手動実行できます。

```sh
cd ~/checksums
./wp-checksums-monitor.sh
```

`sites.conf` が同じディレクトリにある場合、自動的に複数サイトモードで実行します。

バージョン確認:

```sh
./wp-checksums-monitor.sh --version
sh install.sh --version
```

## sites.conf

監視対象サイトは `sites.conf` に保存されます。

```text
# site_name|wordpress_path|webhook_url
example.com|/home/example/example.com/public_html/wp|
```

3列目の `webhook_url` は任意です。空欄の場合はメール通知のみで運用できます。

## Cron

インストーラーは crontab に次のような管理ブロックを追加します。

```cron
# BEGIN WordPress checksum monitor
0 7 * * * cd /home/example/checksums && MAIL_TO=user@example.com MAIL_MODE=fail_only ./wp-checksums-monitor.sh >/dev/null 2>&1
# END WordPress checksum monitor
```

再度 `install.sh` を実行した場合、この管理ブロックだけを更新します。
他の cron 設定は削除しません。

メール通知モードは次の3種類です。

```text
always    正常時も異常時も毎回メールする
fail_only 異常時だけメールする
none      メールしない
```

## ログ

実行ログは次のファイルに保存されます。

```text
~/checksums/checksum-monitor.log
```

## 検知できる範囲

このツールは WP-CLI の次のコマンドを使います。

```sh
wp core verify-checksums
wp plugin verify-checksums --all
```

主に検知できるもの:

- WordPress コアファイルの不一致
- WordPress.org 公式プラグインのファイル不一致

検知できない、または弱いもの:

- テーマの改ざん
- 有料プラグインや独自プラグイン
- `uploads` 配下の不正ファイル
- データベース内の改ざん
- 管理者ユーザーの不正追加
- `wp-config.php` や `.htaccess` の変更
- サーバー全体が侵害されているケース

このツールは WordPress の改ざん兆候を確認する補助ツールです。
完全なマルウェア検知や侵入防止を保証するものではありません。

## よくある質問

### 正常時も毎日メールが届きますか？

初期設定では異常時だけメールを送ります。正常時も確認メールがほしい場合は、
インストール時に「正常時も異常時も毎回メールする」を選択してください。

### WordPress 本体やプラグインを書き換えますか？

書き換えません。WP-CLI のチェックサム検証を実行し、結果をログと通知に残します。
監視用ファイルは `~/checksums` に配置されます。

### 有料プラグインも検知できますか？

基本的には対象外です。`wp plugin verify-checksums --all` は WordPress.org 公式
プラグインのチェックサム確認を主な対象にしています。

### テストサイトや古いバックアップサイトも検出されますか？

`$HOME` 配下に WordPress が残っていれば検出されることがあります。サブディレクトリに
設置した別言語サイトや、テストサイト、古いバックアップサイトも検出される場合があります。
インストール時に一覧を確認し、必要なサイトだけを番号で選択してください。

### 異常通知が来たら何をすればよいですか？

通知内の WP-CLI 出力で対象ファイルを確認し、心当たりのない変更であればバックアップ
との比較や復元を検討してください。原因が判断できない場合は、ホスティング会社や
セキュリティに詳しい担当者へ相談してください。

## アンインストール

インストール済みの `~/checksums/uninstall.sh` を実行します。

```sh
cd ~/checksums
sh uninstall.sh
```

アンインストーラーは次の管理ブロックだけを crontab から削除します。

```cron
# BEGIN WordPress checksum monitor
...
# END WordPress checksum monitor
```

ファイルを削除するかどうかは実行時に選択できます。

削除予定内容だけ確認する場合:

```sh
sh uninstall.sh --dry-run
```

## 配布ZIPの作成

開発環境では、次のコマンドで配布用ZIPを作成できます。

```sh
./build-release.sh
```

`build-release.sh` はZIP作成後に配布物を検証します。原本のファイルと
`release/` 配下、ZIP内の内容が一致しない場合や、古い例示文字列が残っている場合は
エラーで停止します。

生成先:

```text
release/xserver-wp-checksum-monitor-バージョン.zip
```

配布物だけを確認する場合:

```sh
./verify-release.sh
```
