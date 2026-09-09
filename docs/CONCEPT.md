# Framelet の設計背景

## 1. 目的

Minecraft のコマンドは、Minecraft の世界やデータに対して一命令ずつ操作を行うコマンド指向の構文です。`.mcfunction` を利用すれば複数の命令を順番に実行できるため、Datapack としてプログラミングを行うことはできますが、一般的なプログラミング言語で標準的なコードブロック、呼び出しごとのローカル変数、`for` / `while` のようなループ構文、ブロック単位のスコープなどは、そのままの形では備わっていません。

Framelet は、外部コンパイラや独自言語へ変換するのではなく、Minecraft コマンドの構文を維持したまま、可能な限り通常のプログラミング言語に近い構造で `.mcfunction` を記述することを目的としています。

最初の具体的な対象として、次の一連の処理を1つの処理単位として記述することを検討しました。本ドキュメントでは、この一連の流れを **基本フロー** と呼びます。

```text
初期化
  ↓
ループ
  ↓
終了
```

## 2. 基本フローに求める条件

### 2.1 処理固有のコードを1つの frame関数に置く

通常のプログラミング言語では、処理固有の初期化・反復・終了処理を同じ関数やコードブロックにまとめて記述できます。

Minecraft では、複数行を `execute ... run { ... }` のようなコードブロックとして直接まとめることができません。一連の複数行を再実行・分岐させる場合、通常は処理固有のコードを複数の function に分割して記述することになります。

Framelet では、handle の発行、route 遷移、list の next、release などの制御処理を共通ランタイムへ集約し、処理固有の `init / loop / done` を1つの frame関数に記述します。

ここでいう「1 function」は Framelet の内部実装まで1ファイルだけで構成するという意味ではなく、**1 process につき原則1つの処理固有 frame function を追加する**という意味です。

```text
共通ランタイム    → 1度導入し、全frame関数で共有
処理固有のコード  → 1 processにつき原則1 frame function
```

### 2.2 tick 境界を挟まずに完了する

`schedule` を使用すれば、同じ function を後の tick に再実行することでループを構成できます。ただし、tick をまたぐ場合、基本フローは時間方向に分断されます。

```text
初期化
  ↓
loop A

──── tick 境界 ────

loop B

──── tick 境界 ────

loop C
  ↓
終了
```

tick 境界の間では、Minecraft 世界の更新や他の tick 処理、他 Datapack の処理が実行され得ます。そのため、`loop A` と `loop B` の間に、処理対象となる Entity、scoreboard、storage、その他の world state が変化する可能性があります。

たとえば、ある一覧を取得して全要素を順番に処理する場合、途中で tick をまたぐと、一覧を取得した時点と後半の要素を処理する時点で世界の状態が異なる可能性があります。

また、後の tick で処理を続行するには、現在の進行位置、次に処理する対象、初回か継続か、どの呼び出しに属する状態か、といった継続情報を現在の function 実行終了後も保持する必要があります。これは、処理をその場で完了する一連の command chain ではなく、状態を保存して後から再開するステートマシンとして構成する必要があることを意味します。

Framelet では、基本フローの途中に tick 進行による外部更新の介入点を作らず、function を呼び出した tick 内で初期化から終了までを完了させます。

```text
呼び出し
  ↓
初期化
  ↓
ループ
  ↓
終了
  ↓
呼び出し元の次のコマンド
```

この条件は処理速度そのものを目的としたものではありません。大量の反復を行えば1 tick に負荷が集中するため、処理量には別途注意が必要です。また、Framelet 自身が呼び出す function などによる状態変更は発生するため、処理全体が原子的になることを保証するものでもありません。

### 2.3 再入可能である

scoreboard や固定 storage に `running`、`phase`、現在要素などを保持すれば、1つの処理について初回と継続を区別し、基本フローを構成することは可能です。

ただし、状態の保存先が固定されている場合、同じ処理を再帰・ネストして起動すると複数の呼び出しが同じ状態を参照します。

```text
処理A ─┐
       ├─> shared state
処理B ─┘
```

この状態では、処理Bが処理Aの進行状態を上書きする可能性があります。したがって、基本フローを呼び出し単位で独立させるには、**呼び出しごとに分離された可変状態**が必要です。

### 2.4 Entity に依存しない

Entity を状態の identity として利用すれば、個体ごとに異なるデータを保持できます。一方、Framelet が管理する対象はゲーム世界上のオブジェクトではなく、function の一時的な実行状態です。

Entity を実行状態の管理に使用すると、Entity の存在と chunk の読み込み状態がランタイム管理へ影響します。Minecraft のコマンドから取得・選択できる Entity は読み込まれている範囲に依存するため、対象 Entity を含む chunk が読み込まれていない場合、selector から取得できず、`kill` などによる解放処理もその時点では実行できない可能性があります。

また、Entity と別の storage 等に管理情報を保持する設計では、管理情報だけを削除して Entity 本体を削除できなかった場合、chunk が再度読み込まれた際に管理対象外の Entity が残留する可能性があります。反対に、Entity が想定外に消失した場合には管理情報だけが残る可能性もあり、実体と管理データの整合性を維持する処理が必要になります。

Framelet では、function の実行状態を `storage` 上の frame として管理し、Entity の spawn / kill、Entity selector、chunk の読み込み状態を frame のライフサイクル管理に使用しません。

## 3. 利用可能な Minecraft の状態管理機構

### 3.1 scoreboard

scoreboard は数値計算と比較に適していますが、固定された fake player 名をローカル変数として利用すると、その値は呼び出し単位ではなく共有状態になります。

```text
#i
#phase
#return
```

function ごとに名前を分ければ異なる function 間の衝突は減らせますが、同じ function 自身の再帰には対応できず、処理数に応じて変数名の管理量も増えます。また、NBT の list や compound のような構造化データを直接保持する用途には適していません。

### 3.2 固定 storage

storage は list、compound、string などの構造化データを保持できます。ただし、参照先が固定されている場合、

```text
namespace:runtime arg
namespace:runtime prm
namespace:runtime queue
```

は複数の呼び出しから共有されます。

必要なのは storage そのものではなく、**呼び出しごとに異なる storage 上の状態領域を参照する方法**です。

### 3.3 function macro

function macro は function 呼び出し時に与えられた値を `$(name)` の形式でコマンドへ展開します。この値は呼び出しごとに与えられるため、呼び出し固有の値を function 内へ持ち込む手段として利用できます。

一方、macro 引数そのものは処理途中で自由に更新し続ける可変なローカル状態領域ではありません。現在の呼び出しに与えられた値を変更したい場合は、storage 等に値を書き出し、次の function 呼び出しへ改めて渡す必要があります。

Framelet では、macro の「呼び出し固有の値を伝播できる性質」と、storage の「可変な構造化データを保持できる性質」を組み合わせます。

```text
macro
  → 呼び出し固有の参照情報を伝播する

storage
  → 可変な状態を保持する

handle
  → 参照する frame を識別する
```

## 4. Dynamic Storage Frame

DSF は、frame関数の起動時に active な frame と衝突しない handle（参照ID）を発行し、その handle 配下へ呼び出し固有の状態を保存します。

```text
dsf:frame

12:
  arg: ...
  prm: ...

13:
  arg: ...
  prm: ...

14:
  arg: ...
  prm: ...
```

`issue_with_run` 等によって独立した処理として起動された frame関数は、それぞれ異なる handle を参照するため、同じ frame関数を再起動・ネスト起動しても状態領域を分離できます。

```text
call A ── handle 12 ──> frame 12
call B ── handle 13 ──> frame 13
call C ── handle 14 ──> frame 14
```

### 4.1 frame の位置づけ

frame は storage 上に確保される、1回の frame関数の新規起動に固有の状態領域です。同じ frame関数を複数の独立した処理として起動した場合、それぞれの起動は異なる handle と frame を持ちます。

理解上は、frame を「新規起動ごとに生成される実行 instance に対応する状態」とみなすこともできます。ただし `instance` は Framelet 内部の正式な用語ではなく、class / object の instance を実装するものでもありません。

### 4.2 handle による参照

handle は frame の identity であると同時に、他の function へ渡せる参照 ID です。

```mcfunction
$function example:helper {handle:$(handle)}
```

補助 function は同じ handle を使用して、呼び出し元と同じ frame を読み書きできます。

```mcfunction
$data modify storage dsf:frame $(handle).prm.some_value set value 1
```

このため、処理固有の状態を補助 function へコピーする必要がなく、frame の参照を渡す形で共通処理を実装できます。これは通常のプログラミングにおける参照渡しに近い性質です。

### 4.3 実引数と仮引数に相当する領域

各 frame は主に `arg` と `prm` の2領域を利用します。

`arg (argument)` は、呼び出し側が次の function 呼び出しへ渡す値を構成する領域で、実引数に相当します。`path`、`handle`、`route`、`elem` など、次回呼び出しへ渡す値を格納します。

`prm (parameter)` は、受け側が必要な値を格納・判定する領域で、仮引数に相当します。また、list など処理中に保持する作業用データも原則として `prm` に格納します。

Minecraft function 自体に仮引数宣言が追加されるわけではありませんが、Framelet の呼び出し規約として、呼び出し側の実引数と受け側の仮引数に相当する役割を分離しています。

### 4.4 `issue`

動的 frame を使用するには、各呼び出しに active なものと衝突しない handle を割り当てる必要があります。

`dsf:frame/handle/issue` は、循環する cursor と `used` の情報から未使用 handle を探索し、使用中として予約します。frame の終了時には `dsf:frame/release` が frame と `used.<handle>` を削除し、handle を再利用可能な状態に戻します。

### 4.5 `issue_with_run`

`issue` のみを利用する場合、利用側では handle の発行、初期引数への handle の追加、対象 frame関数の実行を個別に記述する必要があります。

```text
issue
  ↓
initial arguments + handle
  ↓
frame function
```

`dsf:frame/handle/issue_with_run` はこの起動手続きを統合し、次の1コマンドで新しい frame関数の実行を開始します。

```mcfunction
/function dsf:frame/handle/issue_with_run {path:"namespace:path", arg:{route:"init", elem:0}}
```

DSF が呼び出し固有の可変状態を提供し、`issue_with_run` が handle の確保から frame関数の起動までを共通化することで、処理固有の基本フローを1つの frame関数へまとめる記述形式を体系化しています。

## 5. frame関数の基本フロー

frame関数は `route` によって `init`、`root`、`branch` の実行経路を区別します。

```text
init
  │
  └─> root
        │
        ├─> branch
        │     │
        │     └─> branch
        │           │
        │           └─> ...
        │
        └─> done → release
```

`init` は frame の初期状態を構成し、`root` へ移行します。`root` と `branch` がループ本体を実行し、次要素が存在する場合は同じ frame関数を `branch` として再帰的に呼び出します。

この `root / branch` の自己再帰は、同じ基本フローを継続するための制御呼び出しであり、新しい frame は発行しません。`path`、`handle`、`route`、`elem` を `arg` 経由で引き継ぎ、同じ handle の frame を共有します。これとは別に、同じ frame関数を新しい独立した処理として再度起動する場合は `issue_with_run` 等から新しい handle を発行します。

最深の `branch` から戻った後、各 `branch` 呼び出しは終了処理を実行せず `return` します。最初の `root` 呼び出しのみが `done` へ進み、処理固有の終了処理と frame の `release` を実行します。

## 6. Framelet が提供する構造

Framelet の目的は、Minecraft function に単独の「ローカル変数機能」を追加することではありません。基本フローを1 tick・1 frame関数・Entity非依存・再入可能な形で記述するために必要な実行状態を DSF として構成した結果、次の性質を同時に利用できるようになります。

- frame関数の新規起動ごとに分離された疑似ローカル変数
- 再帰・ネストに対する再入可能性
- handle による疑似参照渡し
- frame関数の新規起動ごとに独立した frame
- handle の明示的な issue / release
- Entity 非依存の実行状態
- tick 境界を挟まない基本フロー
- 1 process = 1 frame function の記述形式

Framelet は Minecraft に新しい構文を追加するものではなく、Minecraft の既存コマンド仕様を組み合わせ、その上に小規模な実行環境を構成する設計です。

## 7. 現行実装の規模

現行の DSF コアは、次の8つの `.mcfunction` で構成されています。

```text
dsf:init
dsf:data/list/next
dsf:frame/format
dsf:frame/release
dsf:frame/handle/issue
dsf:frame/handle/issue_with_run
dsf:frame/route/goto_next
dsf:frame/route/goto_root
```

GitHub 上の現行ファイルサイズを合計すると約 5.5 KB です（sample / template / pack.mcmeta 等を除く）。

Framelet は、この小規模な共通ランタイムで handle の発行・参照・解放、再入可能な frame 管理、基本フローの制御を実現しています。処理固有の function は原則として1 processにつき1つ追加すればよいため、Framelet を利用する処理が増えても、制御機構自体を処理ごとに複製する必要はありません。

