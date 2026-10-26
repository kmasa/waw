# 初期導入（WSL2 Ubuntu）

WSL2 Ubuntu 導入済みの環境から、ローカル取得物で親 `waw` を構築し、通常の `waw bash` へつなぐ手順です。

**新規 WSL への bootstrap は未確認です。** 2026-10-26、既存 WSL 環境で公開候補の install を実行し、Windows の通常入口から fork・編集・commit・通常 done・大元帰還と子の自動削除を確認しました。[確認範囲と制限](VERIFICATION.md)を参照してください。下記はワンライナーの公開インストーラーではありません。

## 取得物と最初の起動

信頼できる WAW 公開取得物を WSL 内の作業 directory に展開します。**生成済み `src/tools/ria.dist.rb` は同梱しています。** ria の別途取得・生成、klib の配置は不要です。Ruby と標準ライブラリは必要です。公開物は `README.md` / `LICENSE` / `.gitignore` / `docs` と、実行コード一式の `src` で構成します。ルートの開発用 `Riafile` や `test` は含みません。

公開 repo は `https://github.com/kmasa/waw` です。取得物全体を対象 WSL に置いてください。内部の開発 repo や ria repo は不要です。OS のパッケージ・Ubuntu image・共有ツール等は導入時に外部から取得します。

WSL Ubuntu の一般ユーザーで Ruby がない場合は、変更内容を確認の上、まず実行します。

```bash
sudo apt-get update
sudo apt-get install -y ruby
```

WAW の展開先で実行します（`/path/to` は実際の取得物の場所に置換）。

```bash
cd /path/to/waw
ruby --disable-gems ./src/tools/ria.dist.rb --riafile src/wsl/Riafile bootstrap
```

既定値は Btrfs pool `images` の初期容量 `8GiB`、親 image `images:ubuntu/24.04` です。初回の image 取得にはネットワークが必要です。取得済み image を使う場合は `image=local:<alias>`、初期容量は `size=20GiB` のように渡せます。既存 pool の容量は変更しません。

## 確認・承認と変更内容

- WSL Ubuntu の一般ユーザー、systemd が PID 1 であることを確認します。
- 変更内容の表示に `yes` で同意した後、不足するホストツールを setup の apt 導入で補います（sudo が必要なら対話端末が必要）。既存ツールの無条件更新はしません。
- Incus daemon に接続できない場合は診断して停止します。サービス設定や `incus-admin` の付与は自動実行しません。
- 独立 runtime `~/.waw/runtime/ria.dist.rb` と `~/.local/bin/ria` を配置します。同じ内容なら再利用し、異なる既存ファイルや symlink は拒否します。WSL の起動用 Ruby は apt の Ruby を利用でき、mise は必須にしません。
- WAW 所有の `images` Btrfs pool、専用 bridge `wawbr0`、共有 `/root` 用 custom volume `images/waw-root` を Incus に作成させます。未知の同名資源は採用・上書き・削除しません。
- default profile を継承せず、非特権 Ubuntu 親 `waw` を作成します。guest root の UID/GID は実行した WSL ユーザーへ対応させます。Incus 管理権限はホスト root 相当です。未信頼コードを隔離する sandbox として扱わないでください。
- guest の apt で Git / Ruby 等を導入し、`/opt/waw/runtime/ria.dist.rb` を配置します。取得物の `src/` を `/waw/waw/src/` にコピーし、`/waw/waw` に通常の Git repo を初期作成します。元取得物の `.git` はコピーせず、remote 登録・push はしません。
- 共有 `/root` に既存 setup の mise / Node / 公式 Pi / Herdr を不足時だけ導入します。Herdr は `curl -fsSL https://herdr.dev/install.sh | sh` を使います。外部 installer を実行するため、信頼性・配布条件の確認が必要です。`waw herdr` は Incus 内で現在地のタブ作成を要求し、バイナリ不在や Herdr サーバーへの接続失敗はそのままエラーになります。
- 既存 install へ進み、親コードの bind mount、必要時起動の systemd mount unit、対象 unit の start のみを許可する polkit rule、WSL/Windows の入口を設定します。この段階にも独立した対話確認・sudo があります。

systemd が無効なら、`/etc/wsl.conf` の `[boot] systemd=true` と Windows 側の `wsl --shutdown` が別途必要です。設定変更・shutdown は利用者が影響を確認して明示的に行い、再実行してください。`incus-admin` の付与、サービスの起動設定、Startup keepalive の登録もこのコマンドでは行いません。

## 起動と失敗時

導入後、Windows の新しいターミナルで実行します。

```powershell
waw
waw bash waw
```

WSL からは `~/.local/bin/waw bash waw` でも起動できます。一般の `waw` / `ria` を直接呼ぶには `~/.local/bin` を PATH に含めてください。通常入口は Windows Ruby や、元の取得物 directory に依存しません。

失敗時は非0で止まり、作成済み資源を保持します。自動 rollback / delete はしません。親構築途中の `user.waw.bootstrap=pending` は再実行で上書きしません。診断して状態を確認してください。構築完了済みの同じ所有者の親は再作成せず install へ進みます。親の再作成・置換後の mount 更新、既存開発環境への移行は別作業です。

## 確認範囲

`ria check` / `ria test` に bootstrap のローカル試験を含めています。dist の採用、コマンドの接続、所有印つき親の作成要求、再実行、未知資源拒否、確認への不同意、取得失敗・部分失敗の保持を mock で確認します。実際の apt / Incus / bridge 通信 / UID mapping / 共有 volume / installer / Windows 起動の成功を意味しません。

新規 WSL 実機では、別途承認した検証環境でこの既定手順を通し、Windows の `waw bash`、親 `/waw/waw` の Git、共有 `/root`、通常 fork / done を確認する必要があります。
