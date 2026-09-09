# DSF 内部構造

このドキュメントでは、DSF Framelet の内部ランタイムがどのように handle を発行し、frame を実行し、解放するかを説明します。

## 1. 全体像

DSF の実行ライフサイクルは次の通りです。

```text
issue_with_run
  │
  ├─ issue
  │    └─ free handle を予約
  │
  ├─ call buffer を構築
  │    └─ initial arg + path + handle
  │
  └─ frame function
       │
       ├─ frameへarg/prmを反映
       ├─ init
       ├─ root
       ├─ branch ...
       ├─ done
       └─ release
```

状態の中心は次の2つの storage です。

```text
dsf:handle
  cursor
  used
  cache

dsf:frame
  call
  <handle>.arg
  <handle>.prm
```

---

## 2. `issue`: handle の allocator

`dsf:frame/handle/issue` は、使用中のものと衝突しない handle を発行します。

現在の実装は乱数ではなく、**循環する連番 + 使用中チェック**です。

### cursor

`dsf:handle cursor` は、**今回の `issue` 呼び出しで候補とする handle** を保持します。

ここでは function macro の値が呼び出し時に固定される性質が重要です。`issue` の中で storage 上の `cursor` を更新しても、その呼び出し中の `$(cursor)` は更新前の値のままです。

処理の概略:

```text
呼び出し時の $(cursor) = 今回の候補handle
↓
候補値を一時scoreへ読む
↓
+1（次回候補を計算）
↓
2147483647 以上なら 0
↓
storage上のcursorを次回候補へ更新
↓
used.<今回の候補handle> が存在する？
  yes → 更新済みcursorを引数に issueを再実行
  no  → 今回の候補handleを予約してreturn
```

したがって、初期状態 `cursor = 0` では最初に handle `0` が候補となり、storage 上の cursor は先に `1` へ進みます。

実コードでは `2147483647` 自体は候補として使用せず、`0..2147483646` の範囲を循環します。

### `used`

```text
dsf:handle used.<handle>
```

は現在使用中の handle を示します。

空いている handle が見つかると、

```text
used.<handle> = 1b
```

として予約されます。

release 時にこの値を削除するため、handle は再利用可能です。

### `cache`

`cache` は発行された handle の履歴を保持し、`frame/format` が残留 frame を探索するために利用します。

release は `used` を削除しますが cache は削除しません。

これは異常終了によって frame 本体だけが残った場合でも、format が過去に使用された handle を列挙できるようにするためです。

---

## 3. なぜ循環連番なのか

handle に必要なのは「永続的に一意」であることではなく、**同時に active な frame 間で衝突しないこと**です。

frame は正常終了時に release されるため、解放済みの番号は再利用できます。

循環連番方式では、

- 乱数衝突を考えなくてよい
- 次候補が決定的
- `used` によって active handle だけを回避できる
- 整数上限へ到達しても 0 へ戻せる

という性質があります。

---

## 4. `issue_with_run`: allocate と call の統合

`issue` は handle の発行だけを担当します。

一方、利用者が frame関数を開始するには、

```text
handle発行
+ 初期引数
+ path
+ function実行
```

が必要です。

`dsf:frame/handle/issue_with_run` は、この起動手続きを統合します。

現在の実装:

```text
1. dsf:frame call を削除
2. arg を call へコピー
3. path を call.path へ追加
4. issue の戻り値を call.handle へ格納
5. function <path> with storage dsf:frame call
```

利用側は次の1コマンドだけで済みます。

```mcfunction
/function dsf:frame/handle/issue_with_run {path:"namespace:path", arg:{...}}
```

### `call` は frame ではない

`dsf:frame call` はグローバルですが、永続的な実行状態としては使用しません。

役割は、**issueで生成したhandleと初期引数を、最初の frame関数呼び出しへ渡す直前だけ組み立てる短命なバッファ**です。

frame関数は受け取った macro 引数のうち、後続処理で必要な値を handle 配下の `arg` / `prm` に反映します。

その後 `call` が別の起動によって上書きされても、既に動作中の frame は `<handle>` 配下の状態を利用します。

### グローバルな一時レジスタについて

DSF は `#temp` や `#return`（objective: `dsf.control`）のような固定 scoreboard 値も使用しますが、これらは frame の状態保存には使用しません。

- `#temp`: `issue` 内で次回 cursor を計算する短命な作業レジスタ
- `#return`: `list/next` の戻り値を直後の分岐へ渡す短命な作業レジスタ

どちらも、値を保持したまま別の非同期処理を待つ用途ではありません。値を必要とするコマンドが直後に評価され、再帰へ入る時点では外側の処理がその値を再利用しない構造になっています。

この「**グローバル領域を使う場合でも、呼び出し固有の長寿命な状態を置かない**」ことも、再入可能性を維持するための重要な規約です。

---

## 5. frame の構造

例:

```text
dsf:frame
  12:
    arg:
      path: "example:foo"
      handle: 12
      route: "branch"
      elem: ...
    prm:
      route: "root"
      list: [...]
      ...
```

### `arg`

呼び出し側が次の function 呼び出しへ渡す値を構成する領域で、実引数に相当します。

自己再帰の前に `route` や `elem` を更新し、

```mcfunction
function <path> with storage dsf:frame <handle>.arg
```

として再実行します。

### `prm`

受け側が必要な値を格納・判定する領域で、仮引数に相当します。また、処理全体で保持する可変なローカル状態も置きます。

macro 引数は呼び出しごとに固有ですが可変 storage ではないため、必要な値を `prm` へ投影することで条件判定や更新を行います。


---

## 6. route による自己再帰

### `goto_root`

init 完了後、

```text
arg.route = "root"
↓
same frame function
```

として再実行します。

init 側では `return run` を使うため、初期化用の呼び出しはその時点で終了します。

`goto_root` と `goto_next` が行う自己再帰は、同じ frame の基本フローを継続するための制御呼び出しです。これらの呼び出しでは handle を新規発行せず、`dsf:frame <handle>.arg` を引数として同じ handle を伝播します。

### `goto_next`

loop 終了時、

```text
arg.route = "branch"
↓
list/next
↓
次要素あり？
  yes → same frame function
  no  → return to caller
```

として反復します。

### 再帰復帰時の終了制御

最深の branch から戻ると、各呼び出しは file の done 部分へ進みます。

そこで macro 引数として保持されていた現在呼び出しの `route` を `prm.route` へ復元します。

```text
route == branch → return 0
route == root   → doneへ進む
```

branch は `return 0` で終了するため、終了処理と release は root で1回だけ実行されます。

---

## 7. `list/next` と handle による参照

`dsf:data/list/next` は frame関数自身ではありません。

handle と list 名だけを受け取り、

```text
dsf:frame <handle>.prm.<name>
```

を直接操作します。

この構造は、handle が単なる「起動番号」ではなく、**frameへの参照として機能している**ことを示しています。

```text
frame function ──┐
                 ├─ handle 12 → frame 12
list/next ───────┘
```

補助 function は frame関数と同じ storage 状態を共有できます。

これが DSF における疑似参照渡しの基礎です。

---

## 8. `release`

正常終了時、root が次を実行します。

```mcfunction
/function dsf:frame/release {handle:n}
```

release は、

```text
dsf:frame <handle>
dsf:handle used.<handle>
```

を削除します。

`cursor` はリセットしません。

現在位置から探索を続けることで、直前に利用した領域を毎回先頭から探し直すことを避けています。

---

## 9. `format`

`dsf:frame/format` は、`dsf:handle cache` に記録されている handle を基に、残留している frame を一括削除し、handle 管理領域を初期化するための保守用 frame関数です。

実行時には `dsf:handle cache` を自身の `prm.list` にコピーし、通常の Framelet の route / list loop を使用して、記録されている handle の frame を順番に削除します。

処理完了時には、handle 管理領域を次の状態へ初期化します。

```text
cursor = 0
used = {}
cache = remove
```

最後に `dsf:frame/release` を実行し、format 自身が使用した frame と handle も解放します。

このため、`format` 自体も Framelet の基本フローを利用した実例になっています。

### `dsf:init` から呼び出される場合

`dsf:init` では、`cursor`、`used`、`cache` を明示的に初期化した後、`dsf:frame/format` を起動します。

この場合、format が持つ既存 `cache` を利用した残留 frame の削除機能は使用されません。これは、`dsf:init` 自身が DSF の初期状態と使用領域を明示的に定義するためです。

一方、`format` を init 以外から実行する場合は、その時点で `dsf:handle cache` に残っている handle を利用して frame を削除し、その後 handle 管理領域を初期化できます。

したがって、`cache` の削除は次の2箇所に存在しますが、役割が異なります。

* `dsf:init`: DSF の初期状態を明示的に構成するために `cache` を初期化する
* `dsf:frame/format`: format 単体で実行された場合を含め、処理後の handle 管理領域を初期状態へ戻すために `cache` を削除する

---

## 10. 再入可能性が成立する条件

Framelet の再入可能性は「Minecraft が自動で local scope を提供する」ことによって成立するわけではありません。

次の規約によって成立します。

1. frame関数を新しい独立した処理として起動する際に、active handle と衝突しない handle を発行する
2. その処理に固有の可変状態を `dsf:frame <handle>` 配下へ置く
3. `root / branch` の制御再帰や補助 function など、同じ処理に属する呼び出しへは同じ handle を伝播する
4. 同じ frame関数を別の独立した処理として再起動・ネスト起動する場合は、新しい handle を発行する
5. 固定グローバル領域を長寿命のローカル状態として利用しない
6. 正常終了時に frame と handle を release する

この規約を崩す固定変数を独自に追加すると、Framelet を利用していても再入可能性を失う可能性があります。

---

## 11. Stack方式との違い

ローカル実行状態を実装する別の考え方として、1つの list に frame を push / pop し、末尾を現在 frame とする stack 方式があります。

```text
stack
  frame A
  frame B
  frame C <- current
```

DSF は stack 上の位置ではなく、handle によって frame を識別します。

```text
handle 12 -> frame A
handle 27 -> frame B
handle 81 -> frame C
```

どちらも呼び出し状態を分離する方法として成立し得ますが、性質が異なります。

DSF の handle 方式では、現在の stack position に依存せず、handle を知っている任意の補助 function が対象 frame を直接参照できます。

そのため、

- frame参照を引数として明示的に伝播する
- 共通functionから呼び出し元frameを操作する
- frameのidentityをstack位置から分離する

といった用途と相性があります。

これは「常に stack より優れている」という意味ではありません。LIFO の call stack をそのまま模倣する用途では stack 方式が自然な場合もあります。

---

## 12. Entity を利用する状態分離との違い

Entity を状態の identity として利用する方式では、Entity ごとに異なるデータを関連付けられます。

DSF は対象が異なります。

```text
Entity-based
  Entity identity -> private data

DSF
  frame function launch -> handle -> frame
```

DSF は function の一時的な実行状態を管理するため、Entity の生成・選択・生存を必要とせず、実行状態をゲーム世界上の実体から分離します。

Entity を実行状態の identity とする場合、対象 Entity を含む chunk が読み込まれていないと selector から取得できず、解放のための `kill` 等をその時点で実行できない場合があります。Entity と別領域の管理データを併用する設計では、管理データだけを削除した後に Entity 本体が残留する、または Entity 本体だけが消失して管理データが残るといった不整合も管理対象になります。DSF の frame は storage 上だけで生成・参照・解放されるため、この種の chunk / Entity lifecycle 依存を持ちません。
