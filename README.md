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

購入したZIPを手元のパソコンで展開し、フォルダー内の配布ファイル一式を
SFTPなどでXserverへアップロードしてください。以下では、アップロード先を
`~/checksum-monitor-package` とします。`~` はSSHでログインするユーザーの
ホームディレクトリです。

アップロード先は、Web公開ディレクトリ（`public_html`）の外にし、
インストール先の `~/checksums` とは別のディレクトリにしてください。
`install.sh`、`wp-checksums-monitor.sh`、`uninstall.sh`、`VERSION` を含む
配布ファイルは、同じディレクトリに置いてください。

SSHでXserverにログインし、アップロード先へ移動します。

```sh
cd ~/checksum-monitor-package
```

以下のインストール用コマンドは、このディレクトリで実行します。

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

この手動実行には、Cronに設定した通知先や通知モードは引き継がれません。
メールも確認する場合は、次のように通知先を指定してください。

```sh
MAIL_TO=user@example.com MAIL_MODE=always ./wp-checksums-monitor.sh
```

`user@example.com` は実際の通知先に置き換えてください。

インストール済みツールのバージョン確認:

```sh
cd ~/checksums
./wp-checksums-monitor.sh --version
```

配布ファイルのバージョン確認は、アップロード先で実行します。

```sh
cd ~/checksum-monitor-package
sh install.sh --version
```

## 導入後の確認

1. インストール時の初回チェックで、選択したサイトが処理されたか確認します。
   初回チェックではメール通知を行いません。警告やエラーが出た場合は、表示された
   WP-CLIの出力を確認してください。
2. 「通知テストメールを送信しますか？」で `y` を選び、実際に受信できるか確認します。
   画面の送信完了表示だけでなく、受信箱で確認してください。
3. 次のコマンドでCron登録を確認します。

   ```sh
   crontab -l
   ```

   `# BEGIN WordPress checksum monitor` から `# END WordPress checksum monitor`
   までのブロックに、指定した実行時刻・通知先・通知モードがあることを確認します。
4. 登録した時刻を過ぎたら、次のコマンドでログの日時と処理結果を確認します。

   ```sh
   tail -n 50 ~/checksums/checksum-monitor.log
   ```

   `Completed multi-site checksum verification: total=... failed=...` の
   `total` が監視対象数と一致するか確認してください。`failed=0` は検証失敗なし、
   `failed` が1以上の場合は同じ実行の警告・エラーを確認します。

異常時だけ通知する設定では、メールが来ないことだけで稼働を判断できません。
導入後は、Cronによる実行結果がログに残るところまで確認してください。

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

上記はCron実行時と、`cd ~/checksums` 後の手動実行時の保存先です。
ログは実行時のカレントディレクトリに作成されるため、インストール時の初回チェックの
ログは、通常 `~/checksum-monitor-package/checksum-monitor.log` に残ります。

## 設定の変更

通知先・実行時刻・メール通知モードを変更する場合は、配布ファイルを置いた
ディレクトリで `sh install.sh` を再実行し、希望する内容を入力してください。
監視用のCron管理ブロックが更新されます。

再実行すると `sites.conf` も選択したサイト一覧で上書きされます。
手動で追加したサイトやWebhook URLは引き継がれないため、事前に設定を控えてください。

監視対象だけを変更する場合は、`~/checksums/sites.conf` をテキストエディターで
編集できます。1サイトにつき1行で追加・削除し、既存の3列の形式を保ってください。
保存後、上記「通常の実行」の手順で対象サイトと結果を確認します。

## 新しいバージョンへの更新

1. `~/checksums` のファイル一式をバックアップし、`crontab -l` の表示内容も保存します。
   バックアップはWeb公開ディレクトリの外で保管してください。
2. 新版のZIPを展開し、配布ファイル一式を `~/checksums` とは別のディレクトリに
   アップロードします。新版のREADMEに更新時の案内がある場合は確認してください。
3. 新版のアップロード先で `sh install.sh --check` を実行し、問題がなければ
   `sh install.sh` を実行します。通知先・実行時刻・通知モード・監視対象を再入力します。
4. `sites.conf` は再生成されるため、必要なWebhook URLや手動設定をバックアップから
   確認して再設定します。スクリプト・`VERSION`・`cron.checksums` も更新されます。
5. バージョンと「導入後の確認」の各項目を確認します。

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

### メールが届かない場合は？

次の順で確認してください。

1. 迷惑メールフォルダーや受信側の振り分け・受信拒否設定を確認します。
2. `crontab -l` で `MAIL_TO` が正しいか確認します。`MAIL_MODE=fail_only` は
   異常時だけ、`none` はメールを送りません。
3. 「通常の実行」の通知先を指定したコマンドで、`MAIL_MODE=always` にして
   手動実行し、受信できるか確認します。
4. ログの日時、検証結果、`sendmail is not available` などのエラーを確認します。
   手動実行は動くのに指定時刻のログが増えない場合は、Cron登録と実行環境を確認します。

環境診断をやり直す場合は、配布ファイルのディレクトリで
`sh install.sh --check` を実行してください。

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

## 利用条件と問い合わせ

利用前に、同梱の [LICENSE.txt](LICENSE.txt)（利用許諾）と
[DISCLAIMER.txt](DISCLAIMER.txt)（免責事項）を確認してください。

問い合わせ窓口やサポート範囲が購入ページ・購入時の案内に記載されている場合は、
そちらを確認してください。問い合わせる際は、ツールのバージョン、実行した操作、
エラー表示や該当時刻のログを用意してください。パスワード、SSH秘密鍵、
Webhook URLなどの認証情報は送らないでください。
