# DSF Framelet

DSF Framelet は、Minecraft のコマンド構文を維持したまま、`.mcfunction` を可能な限り通常のプログラミング言語に近い構造で記述することを目的とした Datapack ランタイムおよびコード設計です。

中核機構の **DSF（Dynamic Storage Frame / 動的ストレージフレーム）** は、frame関数を新しい処理として起動するごとに一意な storage 上の frame を動的に確保し、その参照となる **handle（参照ID）** を macro 引数として伝播させます。

DSF により、次の構造を Minecraft function 上で実現します。

- frame関数の新規起動ごとに独立した疑似ローカル状態
- 同じ frame関数を独立した処理として再起動・ネスト起動した場合の再入可能性
- handle を介した補助 function への疑似参照渡し
- Entity を実行状態のコンテナとして使用しない状態管理
- tick 境界を挟まず、呼び出した tick 内で完了する基本フロー
- 処理固有の「初期化・ループ・終了」を1つの frame関数にまとめる構造

Framelet は独自言語や外部コンパイラによって Minecraft コマンドを置き換えるものではありません。Minecraft コマンド、function macro、storage、`return` などの既存仕様を組み合わせ、その上に呼び出し単位の実行状態と制御構造を構成します。

## DSF による状態分離

固定された scoreboard や storage を function の状態変数として使用すると、その状態は複数の呼び出しから共有されます。

```text
call A ─┐
        ├─> shared state
call B ─┘
```

同じ function を再帰的・ネストして呼び出した場合、内側の呼び出しが外側の状態を上書きする可能性があります。

DSF では、`issue_with_run` 等によって frame関数を新しい処理として起動するごとに異なる handle を発行し、handle ごとに frame を分離します。

```text
call A ── handle 12 ──> frame 12
call B ── handle 13 ──> frame 13
call C ── handle 14 ──> frame 14
```

frame は storage 上の状態領域です。同じ frame関数を複数の独立した処理として起動した場合、それぞれの起動は異なる handle と frame を持ちます。

## 実引数と仮引数に相当する領域

各 frame は主に `arg` と `prm` の2領域を持ちます。

- `arg (argument)`: 呼び出し側が次の function 呼び出しへ渡す値を構成する領域で、実引数に相当します。
- `prm (parameter)`: 受け側が必要な値を格納・判定する領域で、仮引数に相当します。処理中に保持する list などの作業用データも原則として `prm` に格納します。

Minecraft function 自体に一般的なプログラミング言語の仮引数宣言が追加されるわけではありませんが、Framelet の呼び出し規約として、呼び出し側の実引数と受け側の仮引数に相当する役割を分離しています。

## 1 process = 1 frame function

Minecraft には複数行を直接まとめるコードブロック構文がないため、一連の複数行を再実行・分岐させる場合、通常は処理固有のコードを複数の function に分割して記述することになります。

Framelet は、handle の発行、route 遷移、list の next、release などを共通ランタイムとして提供し、処理固有の次の要素を1つの frame関数に残します。

```text
init
  ↓
loop
  ↓
done
```

「1 function」は、Framelet の内部ランタイムまで1ファイルだけで構成するという意味ではありません。新しい処理を追加するとき、追加する処理固有の function を原則1つにできることを意味します。

```text
Framelet 共通ランタイム
  ├─ issue
  ├─ issue_with_run
  ├─ goto_root
  ├─ goto_next
  ├─ list/next
  └─ release
        ↑ 全frame関数で共用

処理A → process_a.mcfunction
処理B → process_b.mcfunction
処理C → process_c.mcfunction
```

処理を追加するたびに init / loop / next / done 用の function 群を追加するのではなく、共通制御をランタイム側へ集約することで、追加ファイル数を最小限に保ちます。

## handle の発行と `issue_with_run`

frame を新規に使用するには、active な frame と衝突しない handle を確保する必要があります。

`dsf:frame/handle/issue` は、循環する cursor と `used` の状態から未使用の handle を探索・予約し、その handle を返します。

`issue` のみを利用する場合、呼び出し側では次の処理が必要です。

```text
handleを発行
↓
初期引数にhandleを追加
↓
frame関数を実行
```

`dsf:frame/handle/issue_with_run` は、この起動手続きを1回の function 呼び出しに統合します。

```mcfunction
/function dsf:frame/handle/issue_with_run {path:"dsfsample:sample", arg:{route:"init", elem:0}}
```

`issue_with_run` は handle を発行し、初期引数と handle を call buffer に構成したうえで、対象の frame関数を実行します。これにより、handle の確保処理を各 frame関数の呼び出し側へ記述する必要がなくなり、frame関数の起動方法を共通化できます。

## 対象バージョン

- Minecraft `26.2` を前提に作成しています。
- それ以前のバージョンでの動作は未確認です。対象バージョンの詳細は`pack.mcmeta`を参照してください。

## 導入

1. リポジトリをダウンロードします。
2. `pack.mcmeta` が `<world>/datapacks/DSF-Framelet-main/pack.mcmeta` となるように配置します。
3. ワールド上で次を実行します。

```mcfunction
/function dsf:init
```

## サンプル

### `dsfsample:sample`

配列 `[1,2,3,4,5,6]` を作成し、同一 tick 内で順番に処理した後、終了処理を行う frame関数です。

```mcfunction
/function dsf:frame/handle/issue_with_run {path:"dsfsample:sample", arg:{route:"init", elem:0}}
```

frame関数を新規作成する場合は `dsfsample:template` をコピーして利用してください。

## ドキュメント

- [Framelet の設計背景](docs/CONCEPT.md)
- [frame関数の作成・利用方法](docs/USAGE.md)
- [DSF内部構造：handle / issue / route / release](docs/INTERNALS.md)

## 用語

| 用語 | 意味 |
| --- | --- |
| Framelet | 本 Datapack、および Minecraft commands をよりプログラミング言語的な記述へ近づけるための設計 |
| DSF | Dynamic Storage Frame。frame関数の新規起動ごとの局所状態を構成する中核機構 |
| frame | handle ごとに `dsf:frame` storage 上へ確保される、呼び出し固有の実行状態 |
| handle | frame を識別・参照する整数 参照ID |
| frame関数 | DSF の frame を利用し、`init / root / branch / done` の規約に従う function |
| `arg` | 次の function 呼び出しへ渡す実引数領域 |
| `prm` | 受け取った値や作業用データを保持する仮引数・ローカル状態領域 |
| `route` | frame関数内の実行経路。`init` / `root` / `branch` |
| release | frame と handle の使用状態を解放する処理 |
