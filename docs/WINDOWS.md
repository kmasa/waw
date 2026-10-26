# Windows 利用ガイド

[README に戻る](../README.md) / [デモ・ベンチマーク案内](DEMO-BENCHMARK.md)

## 必要環境と未対応範囲

主対象は Windows + WSL2 Ubuntu / Incus Ubuntu です。WSL2 Ubuntu から初期構築する場合は [初期導入手順](INSTALL.md) を参照してください（実装・ローカル mock 試験まで、新規 WSL 実機は未確認）。以下は既存環境から install 単体を使う場合の前提です。

| 層 | 前提 |
|---|---|
| Windows | WSL2 Ubuntu、WSL interop、UNC 対応のエディタ。PowerShell 連携には `pwsh.exe` または `powershell.exe` |
| WSL ホスト | systemd、Incus、Incus を操作できるユーザー、既存の Ruby / ria / mise。WAW のホスト用 setup が扱う不足ツール |
| Incus | WAW 所有印・構成の確認が通る非特権 Ubuntu 親 `waw`、Incus 管理の Btrfs pool（標準構成では `images`）、ネットワークと共有 mount |
| 親コンテナ | `/waw/waw` に WAW コード、各 project に通常の `.git`、開発に必要なツール |

Ubuntu コンテナを一つ作るだけでは、WAW 所有情報・共有 mount 等が揃うとは限りません。初期導入用 bootstrap は上記の別手順へ分離しています。

親再作成・置換後の mount 自動更新は対象外です。親の UUID 変更や rootfs 不在で診断停止することは、自動修復できるという意味ではありません。native Linux は今回の実機確認対象ではありません。

外部依存の概要は [WSL](https://learn.microsoft.com/windows/wsl/)、[Incus](https://linuxcontainers.org/incus/docs/main/)、[Ubuntu](https://ubuntu.com/)、[mise](https://mise.jdx.dev/)、[Ruby](https://www.ruby-lang.org/)、[PowerShell](https://learn.microsoft.com/powershell/) の公式情報を参照してください。これらのインストールだけで WAW の親構築が完了するわけではありません。ria を含む依存の配布元・採用版・再配布表示は、公開候補の依存案内で確定が必要です。

## ria の配布形式

ria は klib を内包した生成済みの単一ファイル `ria.dist.rb` を使う構成に切替済みです。配置名が `ria.rb` でも、中身が生成済み dist なら隣の klib は不要です。Ruby と標準ライブラリは引き続き必要です。WAW の公開物に `src/tools/ria.dist.rb` を同梱しています。初期導入時に別の ria repo や dist の生成作業は不要です。

これは依存ファイルの配置方式の説明であり、新規環境への導入確認を意味しません。bootstrap は実装・ローカル mock 試験までです。ria 本体と内包 klib の正式配布元・採用版・ライセンス・必要表示・取得 URL は公開前の確認事項です。未確定のダウンロードコマンドは掲載しません。

## 作成済み親への入口設定

**空の PC 用インストーラーではありません。** 作成済み親・Ruby / ria と、親コードを参照できる状態を前提にします。WSL の一般ユーザーで、親コードが既に `~/.waw/waw` から見えている場合の入口は次のとおりです。

```bash
ria --riafile "$HOME/.waw/waw/src/wsl/Riafile" install
```

まだ親コードを参照できない環境で、このコマンドだけから開始することはできません。見えない場合は停止し、既存構成を確認してください。公開先に別 checkout や未知のファイルがあっても、削除・上書きして進めないでください。

install は所有印・pool・公開先を確認し、必要に応じて以下を設定します。

- 親 `/waw/waw` を WSL `~/.waw/waw` から参照する bind mount。
- 必要時起動の systemd mount unit と、対象ユーザーにその unit の start のみを許す polkit rule。変更が必要な場合は対話確認・sudo が必要です。
- WSL の独立した `~/.local/bin/waw` と Windows の薄い `%LOCALAPPDATA%/Microsoft/WindowsApps/waw.cmd`。
- WSL の不足ホストツールの導入。既存ツールの無条件更新を目的とする処理ではありません。

systemd 全体の設定、`incus-admin` 付与、WSL shutdown、Startup keepalive 登録は別のホスト変更です。この手順を根拠に一括で行わないでください。

設定済み環境では Windows のターミナルから次を実行します。

```powershell
waw
waw status
waw bash waw
```

通常の起動は Windows Ruby や起動前の UNC 読み取りに依存せず、設定済み環境では sudo 不要です。WSL の起動用 sh が Incus・親 rootfs・公開 mount を確認し、必要時に mount を復旧して親コードを実行します。rootfs が見えない場合は診断停止し、親の新規作成や置換はしません。

## 子で作業して戻す

以下は設定済み環境での通常操作の例です。公開デモの実行済み記録ではありません。

1. Windows で `waw bash waw` を実行して親へ入る。
2. Incus Bash で対象 project に移動し、`waw fork --bash` で子を作って入る。
3. 子で編集・アプリの試験・Git commit を行う。WAW 自体を変更する場合は `ria check` / `ria test` を使う。
4. 子を閉じてよいことを確認したうえで、子の Incus Bash から `waw done` を実行する。

`waw fork` だけなら子を作って現在の環境に留まります。表示された ID を使い、別の Windows ターミナルから `waw bash <ID>` でその子に入れます。子からさらに fork できます。孫の done も大元 waw へ直接提出・帰還し、複製元の先行 commit は自動提出しません。

`waw new <name>` と `waw add <URL>` の作成・clone 先は親です。子内で呼び出しても、現在の子だけに作成するコマンドではありません。remote の登録・書換え・push は通常の Git の責務であり、WAW は外部同期を行いません。

退室だけなら `exit` を使います。子は残り、再入室できます。`done` は保存用の exit ではなく、対象成果を統合して大元へ帰還し、安全に閉じられる子だけを削除する操作です。

## Windows から直接編集する

Incus Bash で次を実行すると、現在地に対応する UNC を cwd にして Windows PowerShell を開きます。

```bash
waw pwsh
# PowerShell 内で exit → 元の Incus Bash へ戻る
```

その場所を UNC 対応エディタで開いて編集します。Windows 用の別 checkout へコピーする必要はありません。親を開けば親を直接変更するため、子で作業したいときは子の UNC であることを確認してください。

UNC は WSL 名・pool・instance・共有 mount に依存します。他人のストレージ内部パスをそのまま貼り付けず、対象環境の場所を確認してください。CMD は UNC を cwd にできず、UNC 非対応ツールまで動くとは保証しません。参照用に見える Incus 内部ストレージを、手動のコピー・削除・subvolume 操作に使わないでください。

引数ありの例（Incus Bash）：

```bash
waw pwsh Write-Output 'hello world'
```

`pwsh.exe` がなければ `powershell.exe` を使用します。cmdlet の名前付き引数、PowerShell 内 native コマンドの終了コード伝播には未対応部分があります。外側の終了コードだけで成功と判定せず、実際の出力・成果も確認してください。

## 権限・副作用・制限

- **管理権限：** Incus 管理権限が必要です。`incus-admin` はホスト root 相当であり、sudo 不要でも低権限とは限りません。コンテナ内から WSL への要求や Windows PowerShell 起動も行います。信頼できないコード向けの安全な sandbox と説明するものではありません。
- **共有状態：** `/root` の mise ランタイム・ツール・キャッシュ等は共有です。子での更新・キャッシュ削除は他環境にも影響し得ます。OS・systemd・PostgreSQL・Git/Ruby は Ubuntu guest の apt、言語ランタイムは共有 `/root` の mise、Pi は公式 installer / `pi update` で管理する方針です。共有ツールの更新は分岐で隔離されると考えないでください。
- **コピー：** fork は Incus 管理のディスク copy です。RAM・プロセスは複製しません。実行中 DB 等の整合性が必要なら書込みを止めるなど、アプリ側の対応が必要です。子のプロセスは新たに起動します。
- **容量と負荷：** copy-on-write でも書込みに応じて容量を使い、並列実行は CPU・RAM・I/O を共有します。総処理時間の短縮や固定の速度は保証しません。
- **親参照：** 新規子孫の `/waw/waw-root` は大元 waw の `/waw` の読み取り専用参照です。既存環境の旧 mount は自動変更しません。秘密を含む大元データが子から見えない保証にはなりません。
- **統合と削除：** done はローカル Git により大元 waw の履歴・作業ツリーを変更して大元へ帰還します。大元に存在し、その子の開始 ref もある repo だけが対象です。保持後の再提出は前回成功 HEAD 以降の差分を提出します。子孫・旧 mount 依存・対象外 repo があれば環境を保持し、安全に閉じられる場合だけ削除します。Git が追跡せず無視しているビルド成果物や、project 外の子ローカルデータも削除され得るため、保存したいものは事前に確認してください（通常 repo の未追跡ファイルは dirty 検査の対象です）。競合時は子の提出準備 branch で内容を解消し、`git add` → `git cherry-pick --continue` → 通常 `waw done` を実行します。大元には競合状態を作らず、最終反映は fast-forward のみです。準備中は元の作業 branch の追加開発と混ぜないでください。Git 競合をなくす機能ではありません。
- **失敗：** 親での done、統合対象 repo の dirty・Git 操作中、fork 時の repo が大元から消えた場合、統合失敗等は拒否・停止します。子で後から追加した実験 repo 等は理由を表示して対象外とし、その repo の dirty は統合を妨げず変更もしません。資源と診断を残し、自動 reset / stash / rollback はしません。途中の変更がすべて取り消されるとは考えず、状態を確認してください。
- **既存資源：** 未知の同名資源は自動採用・上書き・削除しません。失敗後の資源や旧試験環境の削除は別判断です。
- **端末：** TTY 表示、エコー、リサイズ、Ctrl+C、終了コード等には未確認・未対応部分があります。通常往復の過去の成功は、すべての端末操作の保証ではありません。

## 検証の読み方

WAW コードのある場所で `ria check` / `ria test` を実行できます。通常 test は CLI / storage / done / bootstrap の既存試験です。CLI / storage / bootstrap は隔離 HOME の mock を含み、done は隔離した実 Git repo で本番の Git 処理を実行します。開発 repo では集約処理の試験も実行します。実 Incus の分岐・統合、Windows PowerShell、空の PC への導入確認とは別です。

`ria testincus` は旧 NixOS 基盤専用の opt-in 試験であり、Ubuntu の通常 CLI や公開デモの確認には使いません。

## 公開前に追記する項目

以下は未確定・未確認の記入待ちです。上の作成済み親向け手順を、新規導入の確認結果として扱わないでください。

- **導入手順：実装・ローカル試験まで。** ローカル取得物からの bootstrap・入口設定・ホスト変更の承認箇所は [初期導入手順](INSTALL.md) に記載済みです。GitHub 所有者・repo・ref と取得 URL、ria dist の正式取得元・採用版・権利条件の確定が残ります。
- **新規環境での確認結果：未確認。** 対象の公開候補、Windows / WSL / Incus の版、開始状態、実行コマンド、`waw bash` 到達の成否と制限を実機確認後に追記します。実装・ローカル試験だけでは完了にしません。
- **候補タスクの計測・既設 Windows 入口の入室：確認済み（2026-10-25）。** [デモ案内](DEMO-BENCHMARK.md#2026-10-25の追加確認)を参照してください。候補を親へ導入し直したものではなく、fork → 編集・試験・commit → done と UNC / PowerShell 往復を含む最終候補の実演一周は未確認です。過去の成功記録を流用しません。
