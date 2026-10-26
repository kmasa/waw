# デモ・ベンチマーク案内

[README に戻る](../README.md) / [Windows 利用ガイド](WINDOWS.md)

**2026-10-24 に実機動作確認、2026-10-25 に候補タスクで各3回の追加計測・時刻順失敗の切分けを実施しました。静穏基準計測と最終候補の Windows 実演一周は未完了です。** 以下の10月24日の値と、末尾の10月25日の追加確認は別条件です。新しい PC への導入結果ではありません。

## 実施したデモ

ビルド済みの Ubuntu 開発環境を通常の `waw fork` で複製し、子 A のテストと子 B の改修・ビルド・テストを並行実行しました。実験子 B の変更は通常の `waw done` で実験親に統合し、統合後の版を再試験しました。

- 親・子 A・子 B の継承 release binary の checksum 一致と子での実行を確認。
- source / target への書込みが他環境に波及しないことを確認。ただし共有 `/root`、CPU・RAM・OS cache は独立ではありません。
- release 成果物のみを持つ子では、初回 test にコンパイルが必要でした。
- 成果物継承後も再コンパイルが起きた回があります。「再ビルド不要」とは結論しません。

小さなアプリの並列改修、同じポートのサーバーへの Windows アクセスは、この実験では実施していません。

## 対象版・条件

- 対象：[ripgrep 14.1.1](https://github.com/BurntSushi/ripgrep/tree/4649aa9700619f94cf9c66876e9549d83420e16c)、固定 SHA `4649aa9700619f94cf9c66876e9549d83420e16c`。
- [公式 README](https://github.com/BurntSushi/ripgrep/blob/4649aa9700619f94cf9c66876e9549d83420e16c/README.md)：Rust 1.72.0 以上、`cargo build --release`、`cargo test --all`。実験では Rust / Cargo **1.85.1**、Ubuntu GCC 13.3.0 を使用。
- [ライセンス](https://github.com/BurntSushi/ripgrep/blob/4649aa9700619f94cf9c66876e9549d83420e16c/COPYING)：Unlicense または MIT。取得した repo 内の表示を保持してください。OSS 本体・binary・cache は WAW の公開候補に収録しません。
- Windows build 10.0.26200.9711、WSL 3.0.3.0、Incus 6.0.0、Ubuntu 24.04 guest、Btrfs の 32 GiB loop image。
- AMD Ryzen 5 5500（6 cores / 12 threads）、guest 表示 RAM 23 GiB / swap 6 GiB。WSL の resource 制限詳細は未確認。
- target：`/waw/ripgrep/target`、Cargo cache：`/waw/.ripgrep-cargo-cache`（どちらも環境コピー対象）。toolchain は共有 `/root`。
- release / test は既定 feature。Cargo jobs=2、test threads=2。test は通常の test/debug profile。TMPDIR は初期 `/tmp`、後半は `/dev/shm/waw-ripgrep-test`。

## 参考所要時間と結果

**他のエージェント等が稼働しており、静穏は未保証です。異なる条件の成功回を抜粋した動作記録で、同条件の速度比較・統計値ではありません。**

| 区間 | 参考結果 |
|---|---:|
| 初回依存取得 | 2.664秒 |
| 初回 release build | 40.862秒 |
| 固定版 test の成功回 | 1106件成功 |
| 子 A：継承 target の release build | 0.176秒、その回は再コンパイルなし |
| 子 A / B の並列 build→test | 両方成功、操作全体18.370秒 |
| 通常 fork の応答 | 1.842秒 |
| fork 開始→新規子の rg 応答まで | 3.542秒 |
| 子 B の変更統合後の test 成功回 | 1107件成功 |

並列回では A は固定版、B はリテラル検索の回帰テスト1件を追加した版です。統合後の試験結果は統合版から取得し、旧版の兄弟環境の結果を流用していません。

### 断続失敗の記録

アクセス時刻順のテストが断続的に失敗しました。`/tmp` と tmpfs の両方で発生し、tmpfs で解決したとは言えません。成功回はテストを skip していませんが、成功回だけで安定性を判断できません。当時は原因未確定でした。その後の切分けで、時計逆行に伴って新規ファイルの atime が逆転し、作成順を前提とする期待値が崩れる例を確認しました（末尾参照）。過去の失敗すべてをこの機序と確定したものではありません。

### 時計に関する別の観測

既存 WAW 試験ログに負の経過秒（`elapsed=-5s` / `-1s`）や終了表示が開始より前になる例があり、実時間時計の逆行として記録しています。当時は時計自体の独立検証や因果関係の切分けは未実施でした。10月25日に単調時計との比較で実時間の逆行を独立に確認しましたが、Cargo 再コンパイルとの関係は未切分けです。計測タスクは単調時計を使いますが、それだけで参考値が静穏な基準値になるわけではありません。時計設定は変更していません。

### その他の未確認・制限
- Cargo の `Blocking` 行は今回のログで観測なし。lock 待ちがない保証ではありません。
- 純粋な copy / start の個別時間、Windows の対話入室完了までの時間は未計測です。rg probe は PowerShell / WSL / Incus 呼出しの待ち時間を含みます。
- 静穏条件での複数回測定・要約、同条件の単独／並列比較は未完了。「並列なら高速」「一定秒数以内」とは保証しません。

## 計測コマンド（開発用）

以下の `ria ripgrep-*` は、開発 repo のルート `Riafile` にある作業用タスクです。現在の GitHub 公開物にはルート `Riafile` を含めないため、公開物だけでは実行できません。以下は開発環境での再計測手順・過去の実行記録として残します。
作成済み WAW 環境と、**固定 SHA の clean な `/waw/ripgrep`**、guest の C compiler、共有 mise の Rust 1.85.1 を前提とします。自動導入・既存 cache の削除はしません。不足ツールの導入は guest / 共有領域への変更として別途判断してください。

新規取得は大元の親で `waw add https://github.com/BurntSushi/ripgrep.git` を使います。既存同名 repo を上書きせず、その状態を確認してください。取得時の HEAD は固定版とは限りません。新規取得直後の clean な repo で上記 SHA を確認・checkout してから計測します。子からの `add` も大元に取得するため、現在の子に追加されると考えないでください。

Ubuntu guest の `/waw/waw` で実行：

```bash
mise exec rust@1.85.1 -- ria ripgrep-env
mise exec rust@1.85.1 -- ria ripgrep-bench fetch
mise exec rust@1.85.1 -- ria ripgrep-bench build
mkdir -p /dev/shm/waw-ripgrep-test
TMPDIR=/dev/shm/waw-ripgrep-test mise exec rust@1.85.1 -- ria ripgrep-bench test
mise exec rust@1.85.1 -- ria ripgrep-bench version
```

`fetch` は `cargo fetch --locked`、build / test は `--frozen --verbose` を加えて依存取得とコンパイルを分離します。各処理を再実行して初回と分けて記録します。`BENCH_RESULT` は preflight を除いた Cargo / rg の起動→終了を測り、SHA・終了コードと秒数を出します。失敗も記録し、成功値に混ぜません。

書込み終了後、`ria ripgrep-fork` で通常 fork と rg probe を計測できます。**新しい子を作成し、削除せず保持する操作です。** `READY_RESULT` の transport 終了コードだけでなく、実際の rg version 出力も確認してください。

作成済み専用子2個に対する並列実行は `ria ripgrep-parallel a=<子AのID> b=<子BのID> bsha=<子Bの完全SHA>`。A は上記固定版、B は指定版を clean にし、両子でツール・成果物の配置を確認します。各子の build / test 時間、両子完了までの操作全体時間、`host_success` とログを確認します。ログは各子と実行元の `/var/tmp` に残ります。

## 現行 done の注意

現行 done の統合先・帰還先は**大元 waw**です。大元に存在し、その子の開始 ref もある repo だけを提出し、子で後から取得した実験 repo は自動追加しません。子孫・対象外 repo・旧 mount 依存があれば統合後も環境を保持します。この計測の実験子 B の done 記録は旧・直近親方式の結果であり、新仕様や任意の入れ子環境の自動削除を確認した意味ではありません。

実験担当環境そのものから大元への合流は WAW repo の Git merge で行い、その後、明示承認で実験環境を削除しました。これは通常 done の成功例ではありません。大元へ直接提出する新仕様はその後に実装され、別の専用 Ubuntu 実機で孫からの直接提出・帰還と保持・再提出・追加提出を確認しました。その後、大元との同一ファイル競合を子側で解消し、解消中に進んだ大元へ通常 done で提出・帰還する経路も専用 Ubuntu 実機で確認しました。保持理由のない子の自動削除と、複数 repo の部分成功後の再実行の実機確認は残っています。この ripgrep 実験をそれらの検証結果には読み替えません。

保存したいログ・成果物を確認し、[Windows ガイドの統合と削除の注意](WINDOWS.md#権限副作用制限)に従ってください。基準計測中は同一ホストの重い処理を避け、並列負荷の計測とは分けます。この候補からの Windows 実演・再計測は引き続き未確認です。

## 2026-10-25の追加確認

ripgrep を採用し、WAW 開発 commit `28e7d8261c3b` から生成した17ファイルの候補の計測タスクを実行しました。初回準備は直前の `ebc39050ef1a` 候補の同じ Cargo タスクです。最初の fork 計測では CLI 出力変更により、子作成成功後に ID 取得だけが失敗しました。`28e7d8261c3b` で計測側の解析を修正し、再実行しました。失敗時の資源・ログも保持しました。

- Ubuntu 24.04、Incus 6.0.0、WSL kernel 6.18.54.1、Btrfs 32GiB。12 logical CPUs、今回の表示 RAM 15GiB / swap 4GiB。旧実験の23/6GiBと混ぜません。
- ripgrep 固定 SHA・Rust/Cargo 1.85.1、jobs=2 / threads=2、既定 feature、TMPDIR=/tmp。各子の target/cache はコピー対象、共有 toolchain は更新していません。
- 他の Pi・エディタ等が稼働中。別の重いビルドは確認時に見られませんでしたが、継続監視・静穏保証はありません。**以下は参考値です。**

| 区間（秒） | 回1 | 回2 | 回3 | 中央値 |
|---|---:|---:|---:|---:|
| 通常 fork 応答 | 0.932645 | 0.945012 | 1.022458 | 0.945012 |
| fork開始→子のrg応答（Windows transport込み） | 2.592015 | 2.661228 | 2.778302 | 2.661228 |
| Incus copy（別のbackend区間計測） | 0.200663 | 0.279192 | 0.215122 | 0.215122 |
| Incus start（同上） | 0.251894 | 0.273308 | 0.257878 | 0.257878 |
| 子A/Bの並列build→test操作全体 | 14.510562 | 14.541728 | 15.003516 | 14.541728 |

copy/start は候補の backend を外付け計装した別の3回です。start は内部 mount 等を含み、通常 CLI 全体の厳密な内訳として加減算しません。計装準備の fork 成功・区間値取得失敗の6回は集計外です。

同一固定版・継承済み target の A/B を各3回ずつ単独で試験した test 区間は、A が11.612648 / 11.933228 / 11.497823秒、B が11.859598 / 12.684764 / 11.494791秒。並列の各 test は A が12.415050 / 12.369251 / 13.146614秒、B が12.527084 / 12.343471 / 13.162297秒でした。全12回の test は exit 0 / 1106 passed、上流既定の ignored 以外に独自 skip はありません。並列操作全体は transport・preflight 等も含み、単独 Cargo 区間の和と境界が異なるため速度倍率は主張しません。単独→並列の順で、OS cache cold の比較でもありません。

初回の fetch / release build / コンパイル込み test は2.592378 / 35.800736 / 47.265051秒、すべて exit 0。既存 cache を消して条件を作った値ではありません。

**Windows は既設の通常 waw.cmd から子へ入室し、hostname / cwd / 継承 rg の版と exit を確認しました。** stdin は regular file で渡し、試験側 PTY は使いませんでした。ホスト側の cmd.exe 起動→退室終了は1回0.697919秒 / exit 0。fork・人の入力・編集・commit・done を含みません。入口は既設の親コードを使い、公開候補を親へ導入し直したものではありません。

### 基準計測とは別枠の切分け

上記を終えてから専用子で、時刻順3テストを /tmp と tmpfs で各10回、計60回実行。/tmp の `sortr_accessed` が1回失敗しました。同時に単調時計と実時間時計を10ms周期で比較し、実時間の約1.265秒の逆行を確認しました。この試行だけでは失敗瞬間との厳密な一致は確定していません。

さらに4ファイルを100ms間隔で作り、各 atime と両時計を記録してから rg で並べる別probeを100回実行しました。/tmp の1回で、単調時計は進むのに実時間と atime が戻り、作成順の逆なら `f3,f2,f1,f0` のはずが、実際は `f1,f0,f3,f2` になりました。**実際の atime の降順も後者と一致し、時計逆行でテストの作成順前提が崩れる機序を確認しました。** tmpfs の成功回を対策の証明とはしません。時計設定・mount・上流テストは変更していません。過去の全失敗、時計逆行自体の発生原因、Cargo 再コンパイルとの関係は確定していません。

## 最終実演・静穏基準値の追記欄（残件）

- **対象版・条件：未記入。** 最終公開候補、採用デモ、ホスト・guest・storage・データ量、他の負荷、cache / target の状態と並列数を記録します。
- **Windows 実演：未確認。** 実行コマンド、開始・終了条件、成否と制限を追記します。
- **静穏基準計測：未実施。** copy / start / 利用可能まで、Windows 操作全体、同条件の単独／並列を区別し、各回の値・終了コード・要約を追記します。失敗・未完了を成功値に含めません。

断続失敗の原因調査は基準計測とは別枠です。調査完了を先行条件にせず、発生した失敗と未解明であることを結果に残します。未計測の速度、並列による短縮、新規環境での導入容易性は主張しません。
