# WAW (WSL AI Workspace)

自動試験・AI コーディングの高速並列化

* 1秒で fork
* node_modules やビルド中間ファイルを含む Linux 開発環境を丸ごと複製
* npm install 不要
* ビルド成果物も継承
* Git の squash commit で統合

[English](README.md)

WAW は **Windows + WSL2** 向けの実験的な CLI です。**Incus と Btrfs** で Ubuntu の開発コンテナを複製し、子で commit した変更を大元の環境へ統合します。一つの作業ツリーを取り合わずに変更を試したり、複数の AI コーディングエージェントへ別々の作業を任せたりするために作っています。

**Experimental / Proof of Concept です。** WAW 自身の開発で基本の一周を利用した実績がありますが、本番運用向けの完成品ではありません。インストールも実験段階で、クリーンな WSL2 環境への導入は未検証です。

**WAW は AI オーケストレーターではありません。** 環境の複製と Git の成果統合に責務を絞っています。エージェントの選択・作業の割り当て・レビューは利用者が行います。

## なぜ環境を fork するのか

Git の branch はソースの履歴を分けますが、インストールしたパッケージ、ビルド成果物、ローカルサービスまでは分けません。WAW はコンテナのディスク状態を複製し、準備した開発環境から別々の作業を始められるようにします。

- **環境ごと分岐：** Incus 管理の Btrfs pool 上で、copy-on-write によるコンテナ複製を使います。
- **子で独立して作業：** それぞれの子で編集・ビルド・試験・commit を行えます。子からさらに fork することもできます。
- **持ち帰るのはコードの変更：** `waw done` は対象の Git 差分を大元 `waw` へ提出します。パッケージやデータベースを丸ごと統合する操作ではありません。
- **Windows のツールも利用：** WSL の UNC パスで同じファイルを直接編集でき、コンテナから `waw pwsh` で Windows PowerShell を開けます。

ただし、完全に独立した環境ではありません。開発ツールやキャッシュを含む `/root` は共有です。エージェントや不慣れなコードを動かす前に、[制限事項](#制限事項と安全上の注意)を確認してください。

### 操作のイメージ

導入済みの大元 `waw` 内で、clean な Git project から実行します。Git の作成者情報は設定済みとします。

```bash
waw fork --bash
printf 'Hello from a WAW child\n' > hello.txt
git add hello.txt
git commit -m "Add a greeting"
waw done
```

変更が大元へ反映されます。実際の作業では、子で編集・試験してから commit します。入室と project 作成からの手順は [Quick Start](#quick-start) を参照してください。

## AI エージェントで並列開発する

たとえば、機能実装とテスト改善を別々の子に割り当てます。

```text
大元 waw
  ├─ fork → 子 A → AI 1: 機能実装   → 試験 → commit → done ─┐
  └─ fork → 子 B → AI 2: テスト改善 → 試験 → commit → done ─┤
                                                            ↓
                                                         大元 waw
```

大元で `waw fork` を2回実行し、それぞれの ID に別のターミナルから `waw bash <ID>` で入ります。子ごとにエージェントを起動し、変更を確認してから提出します。`fork` が行うのは環境の作成までで、エージェントは起動しません。作業の割り当てやレビューは利用者が行うもので、自律的なオーケストレーション機能ではありません。

孫からの提出先も、直近の複製元ではなく大元 `waw` です。Git の競合解消は必要ですし、競合せず適用できた変更にも統合後の試験が必要です。

## デモと参考計測値

既存の Windows + WSL2 / Incus / Btrfs 環境で、ビルド済みの **ripgrep 14.1.1** を fork し、子 A の試験と子 B の改修・ビルド・試験を並列に実行しました。B の変更を統合し、統合後の版も再試験しています。子への release binary の継承・実行と、source / target への書込みが他環境に波及しないことを確認しました。ただし `/root` とホスト資源は共有で、成果物の継承も「再コンパイル不要」を保証しません。

**2026-10-25 の追加計測**は Ubuntu 24.04、Rust/Cargo 1.85.1、jobs=2 / test threads=2、target/cache 継承済みの条件です。他のエージェントやエディタも稼働しており、**静穏な基準値や速度保証ではなく、参考の動作記録**です。

| 計測区間 | 3回の中央値 |
|---|---:|
| 通常 fork の応答 | 0.945秒 |
| fork 開始 → 子の `rg` 応答（Windows transport 込み） | 2.661秒 |
| Incus copy（別途 backend を計装した計測） | 0.215秒 |
| Incus start（同上） | 0.258秒 |
| 子2個の並列 build → test 操作全体 | 14.542秒 |

copy / start は通常 CLI 全体を加減算できる内訳ではありません。追加計測の単独・並列 test は全12回で各1106件成功しましたが、計測境界や cache 条件が異なるため、並列による速度倍率は主張しません。

**注意・残件：** 別枠の切分けではアクセス時刻順の試験が断続的に失敗し、時計逆行がファイルの作成順を前提とする期待値を崩す機序を確認しました。過去の全失敗の原因を確定したものではありません。Windows の入退室は既設の入口で確認したもので、候補を新規導入した結果ではありません。最終候補での Windows 実演一周と静穏基準計測は未完了です。また、元の統合デモは旧・直近親方式の `done` による記録であり、現行の大元への提出・自動削除の検証結果ではありません。

詳細な条件、各回の値、計測コマンド、失敗と残件は[デモ・ベンチマーク記録](docs/DEMO-BENCHMARK.md)を参照してください。

## 動作環境とインストール

主な対象構成は次のとおりです。

- Windows と **WSL2 Ubuntu**。WSL interop が利用でき、WSL 内で **systemd** が有効であること。
- 動作する Incus daemon と、それを管理できる WSL ユーザー。
- WSL の Ruby と標準ライブラリ。タスク実行に使う生成済み単一ファイル `src/tools/ria.dist.rb` は、このリポジトリに含まれています。
- Ubuntu image、パッケージ、ツール installer を取得できるネットワーク。コンテナ image の既定値は `images:ubuntu/24.04` です。
- Windows で編集する場合は UNC 対応エディタ。PowerShell 連携は `pwsh.exe` を優先し、なければ `powershell.exe` を使います。

native Linux は動作確認済みの対象ではありません。通常の Windows ランチャーに Windows Ruby は不要です。

### ローカル取得物からの初期導入

**この経路は実装・ローカル mock 試験までで、新規 WSL への実導入は未確認です。** 状態を調べ、復旧できる開発環境で試してください。公開 repo は [kmasa/waw](https://github.com/kmasa/waw) です。案内できる公開ワンライナー installer はありません。

信頼できるリポジトリのローカルコピー、または WAW の配布物一式を WSL に用意します。Ruby がなければ、先に導入します。

```bash
sudo apt-get update
sudo apt-get install -y ruby
```

WAW のディレクトリで、**WSL Ubuntu の一般ユーザー（root 以外）**として実行します。

```bash
ruby --disable-gems ./src/tools/ria.dist.rb --riafile src/wsl/Riafile bootstrap
```

bootstrap はホスト変更の承認を求め、Btrfs pool、ネットワーク、共有 `/root` volume、大元 Ubuntu コンテナを用意して、起動用の入口を設定します。pool の初期容量は既定で `8GiB`、変更する場合は `size=20GiB` のように指定します。既存 pool の容量は変更しません。guest の setup では不足する mise、Node.js、Pi、Herdr を導入し、一部に外部 installer を使います。

systemd の有効化と Incus の利用権限は別途準備が必要です。bootstrap は `incus-admin` の付与、サービス起動設定、WSL shutdown を行いません。後段の入口設定でも systemd mount と polkit の設定変更に承認を求めます。未知の同名資源は上書きせず拒否します。失敗時は資源と診断を残し、自動 rollback はしません。

実行前に [初期導入手順](docs/INSTALL.md) を確認してください。構築済みの WAW 親に対する入口設定とホスト変更は [Windows 利用ガイド](docs/WINDOWS.md) に記載しています。`install` 単体では親コンテナを作成しません。詳細ガイドは現在、日本語のみです。

## Quick Start

設定完了後、Windows の新しいターミナルで実行します。

```powershell
waw
waw status
waw bash waw
```

`waw` はその入口で使えるコマンド一覧を表示します。最後のコマンドで大元 Ubuntu 環境に入ります。その Bash で小さな project を作り、分岐します。

```bash
waw new waw-example
cd /waw/waw-example
waw fork --bash
```

子の同じ project パスへ移動します。Git の作成者情報を設定したうえで、変更を作って提出します。

```bash
printf 'Hello from a WAW child\n' > hello.txt
cat hello.txt
# 実際の project では、commit 前にその project の試験を実行します。
git add hello.txt
git commit -m "Add a greeting"
waw done
```

成功すると大元 `waw` へ戻り、project に `hello.txt` が反映されます。`done` は対象の commit 済み差分を統合し、安全に閉じられる場合だけ子を削除します。提出後も子を残したければ `waw done --keep` を使います。単に退室するなら `exit` です。提出・削除はせず、`waw bash <ID>` で再入室できます。

### 主要コマンド

| コマンド | 動作 |
|---|---|
| `waw status` | ホスト・Incus の読み取り専用診断。Windows / WSL のみ。 |
| `waw new <name>` | 大元の `/waw` 配下に Git project を作成。 |
| `waw add <URL>` | 大元に Git repository を clone。 |
| `waw fork [--bash]` | 現在の環境を複製。既定はその場に留まり、`--bash` なら子へ入室。Windows / WSL からは大元を複製。 |
| `waw bash [ID]` | ID 指定、または一覧から選んで入室。 |
| `waw list` | 所有環境の状態、一時環境・保持環境の区分、複製元を表示。 |
| `waw done [--keep]` | 対象の Git 差分を提出して大元へ帰還。子コンテナ専用。 |
| `waw drop [ID]` | **未提出 commit・未コミット変更も、確認なしで破棄して**子を削除。子の外からは ID 必須。 |
| `waw temp <cmd> [args...]` | 一時 fork の同じパスでコマンドを実行し、終了後に子を破棄。 |
| `waw pwsh [command args...]` | 対応する UNC パスで Windows PowerShell を起動。コンテナ内専用。 |
| `waw herdr` | Herdr のタブ作成を要求。接続可能な Herdr サーバーが必要。コンテナ内では現在地を使用。 |

`new` / `add` は子で呼んでも大元に作成・取得します。環境の分岐には `fork` を使い、引数なしの `add` は使いません。WAW は Git remote の管理や push を行いません。

使い捨ての実行例は、project 内で `waw temp bash -c 'pwd; git status --short'`。破棄されるのは子のローカルな変更だけで、共有領域や外部への書込みは巻き戻りません。

## 制限事項と安全上の注意

- **ディスクの複製であり、プロセスの複製ではありません。** RAM や実行中プロセスはコピーしません。稼働中コピーは crash-consistent に留まり、DB 等にはアプリ側で書込みを止めるなどの対応が必要です。子で必要なプロセスは起動し直します。
- **共有状態と強い権限があります。** `/root` のツール・キャッシュは共有で、新しい子は `/waw/waw-root` から大元の `/waw` を読み取れます。Incus 管理権限、特に `incus-admin` はホスト root 相当です。未信頼のエージェントやコードを安全に隔離する sandbox ではありません。
- **提出は Git の変更だけです。** `done` は対象の commit 済み差分を提出し、パッケージや DB は統合しません。競合は手動解消が必要です。自動 rollback はなく、複数 repo の提出も atomic ではありません。
- **環境の削除で失われるデータがあります。** `done` は Git 管理外のファイルごと子を削除する場合があります。`drop` は未提出の作業も確認なしで破棄します。重要なデータはバックアップしてください。SIGKILL やホスト停止後の一時環境の削除は保証しません。
- **連携部分も実験段階です。** 端末・PowerShell 連携には制限があります。並列ジョブはホストの資源を共有します。

対象 repo、squash・再提出、競合解消、保持条件、cleanup の詳細は[成果提出の動作と制限事項](docs/USAGE-LIMITATIONS.ja.md)を参照してください。

### 確認済みの範囲

既存の開発環境では fork → 編集・試験 → commit → done、Windows UNC 編集、PowerShell 往復の実機記録があります。専用 Ubuntu 環境では孫から大元への直接提出、子側での競合解消、再提出、一時環境の操作も確認しています。過去の詳細記録は内部の開発 repo に保持しています。今回の候補については公開用の[既存環境での install・動作確認](docs/VERIFICATION.md)を参照してください。

新規環境の検証ではなく、bootstrap は mock 試験までです。2026-10-26、既存環境での install・Windows の通常操作一周・通常 `done` 後の子の自動削除は実機で確認しました。複数 repo の部分失敗からの再実行と `done --keep` には、実 Incus での一通りの確認が残っています。手軽で確実な初期導入や、対応プラットフォームの拡大を現在の機能として保証するものではありません。

内部の開発 repo では `ria check` / `ria test` を実行します。開発用ルート Riafile と test はこの配布物には含めていません。構文・差分とローカルの CLI・storage・bootstrap・Git 処理を検査しますが、mock を含むため Windows / Incus の実機試験の代わりにはなりません。

## License

[MIT License](LICENSE) — Copyright (c) 2026 kmasa.
