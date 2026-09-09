# Frame関数の作成・利用方法

このドキュメントでは、DSF Framelet を利用して frame関数を作成・実行する方法を説明します。

## 1. 初期化

Datapack 導入後、最初に次を実行します。

```mcfunction
/function dsf:init
```

`dsf:init` は DSF が使用する scoreboard と handle 管理用 storage を初期化し、初期化処理の一部として `dsf:frame/format` を frame関数として起動します。

現行実装では、`dsf:init` が `cursor` / `used` / `cache` を先に初期化した後で `dsf:frame/format` を起動します。そのため、`dsf:init` 実行前から残留していた frame を以前の `cache` を使って列挙・削除する動作にはなっていません。既存の `cache` を利用して残留 frame を削除する場合は、handle 管理状態を初期化する前に `dsf:frame/format` を実行する必要があります。

---

## 2. frame関数を起動する

通常は `dsf:frame/handle/issue_with_run` を使用します。

```mcfunction
/function dsf:frame/handle/issue_with_run {path:"dsfsample:sample", arg:{route:"init", elem:0}}
```

引数は大きく2つです。

- `path`: 起動する frame関数
- `arg`: frame関数へ最初に渡す引数 compound

`handle` は自分で指定する必要がありません。`issue_with_run` が未使用の handle を発行し、`arg` に追加したうえで対象 function を実行します。

### なぜ `issue_with_run` を使うのか

手動で行う場合は、

```text
handle発行
↓
初期引数へhandleを追加
↓
frame関数を呼び出す
```

という前処理が必要です。

`issue_with_run` はこれらを1つにまとめ、利用側では handle の発行手順を記述せず、**1コマンドで新しい frame関数の実行を開始**できるようにします。

---

## 3. 「1 process = 1 frame function」

Framelet の「1 function」は、DSF のランタイム内部まで1ファイルだけで実装するという意味ではありません。

`issue`、`issue_with_run`、`goto_root`、`goto_next`、`list/next`、`release` などは、すべての frame関数から再利用する共通ランタイムです。

利用者が新しい処理を追加するときは、その処理固有の、

```text
init
loop
done
```

を1つの frame関数に記述します。

```text
共通ランタイム
  ├─ issue
  ├─ goto_root
  ├─ goto_next
  └─ release
       ↑ 共用

処理A → process_a.mcfunction
処理B → process_b.mcfunction
処理C → process_c.mcfunction
```

この意味で、Framelet の基本単位は **1 process = 1 frame function** です。

---

## 4. template から frame関数を作成する

`data/dsfsample/function/template.mcfunction` をコピーして作成します。

基本構造は次の4領域です。

```text
1. 引数を frame に反映
2. init
3. root / branch loop
4. root done
```

概念的なテンプレートは次の通りです。

```mcfunction
# 固定処理: path / handle / route をframeへ反映
$data merge storage dsf:frame {$(handle):{arg:{path:"$(path)", handle:$(handle)}}}
$data merge storage dsf:frame {$(handle):{prm:{route:"$(route)"}}}

### init ###

# 任意処理: 初期化を記述する

# 固定処理: rootへ移行
$execute if data storage dsf:frame $(handle).prm{route:"init"} run return run function dsf:frame/route/goto_root {path:"$(path)", handle:$(handle)}

### root/branch: loop ###

# 任意処理: 反復処理を記述する

# 固定処理: listを進め、残っていればbranchとして再実行
$function dsf:frame/route/goto_next {path:"$(path)", handle:$(handle), name:"list"}

### root: done ###

# 固定処理: 現在の呼び出しのrouteを復元
$data modify storage dsf:frame $(handle).prm.route set value "$(route)"

# branchは終了処理を行わずreturn
$execute if data storage dsf:frame $(handle).prm{route:"branch"} run return 0

# 任意処理: 最終終了処理を記述する

# 固定処理: frameを解放
$function dsf:frame/release {handle:$(handle)}
```

### 固定処理の扱い

`固定処理` は frame関数の制御フローを成立させるための規約です。原則として削除・順序変更を行わず、`任意処理` の箇所へ処理固有のコードを追加してください。

---

## 5. `route` の意味

frame関数では、同じ `.mcfunction` を複数回呼び直しながら処理を進めます。

その呼び出しがどの経路なのかを `route` で区別します。

| route | 役割 |
| --- | --- |
| `init` | 初回。ローカル状態・listなどを初期化する |
| `root` | ループの最初の本体呼び出し。最終的な done を担当する |
| `branch` | ループ継続のために再帰的に呼び出された本体 |

`root` と `branch` は同じ基本フローを継続するための制御呼び出しであり、新しい handle は発行しません。同じ handle の frame を共有します。

最深の branch から戻った後、各 branch は `return 0` で終了し、最初の root だけが終了処理と release まで進みます。

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

---

## 6. `arg` と `prm`

各 frame は主に `arg` と `prm` の2領域を利用します。Framelet の呼び出し規約では、`arg` を実引数、`prm` を仮引数に相当する領域として扱います。

### `arg` — argument / 実引数

呼び出し側が、次の function 呼び出しへ渡す値を組み立てる領域です。

代表的な値:

- `path`
- `handle`
- `route`
- `elem`

たとえば `goto_next` は `arg.route` を `"branch"` に変更し、更新された `arg` を使って同じ frame関数を再実行します。

### `prm` — parameter / 仮引数・ローカル状態

受け側が必要な値を格納・判定する領域です。また、frame関数内で継続して使用する list や処理固有の作業データも原則として `prm` に格納します。

代表例:

- `route`
- `list`
- 処理固有の作業変数

Minecraft function 自体に仮引数宣言が追加されるわけではありませんが、`arg` と `prm` の役割分離により、一般的な関数呼び出しにおける実引数／仮引数に近い構造を扱えます。

---

## 7. list を使ったループ

`dsf:data/list/next` は handle と list 名を受け取り、指定 frame の list を1つ進める汎用処理です。

```mcfunction
/function dsf:data/list/next {handle:n, name:"list"}
```

内部では概ね次を行います。

```text
prm.<name>[0] を削除
↓
新しい prm.<name>[0] を arg.elem に設定
↓
次要素があれば 1、なければ 0 を返す
```

`dsf:frame/route/goto_next` はこの戻り値を利用し、次要素が存在する場合だけ現在の `path` を branch として再実行します。

### サンプル

```mcfunction
# init
$execute if data storage dsf:frame $(handle).prm{route:"init"} run data modify storage dsf:frame $(handle).prm.list set value [1,2,3,4,5,6]
$execute if data storage dsf:frame $(handle).prm{route:"init"} run data modify storage dsf:frame $(handle).arg.elem set from storage dsf:frame $(handle).prm.list[0]

# loop
$say @a $(elem)

# next
$function dsf:frame/route/goto_next {path:"$(path)", handle:$(handle), name:"list"}
```

---

## 8. handle を補助 function に渡す

handle は frame への参照として使えます。

```mcfunction
$function example:helper {handle:$(handle)}
```

補助 function 側では、

```mcfunction
$data modify storage dsf:frame $(handle).prm.some_value set value 1
```

のように、呼び出し元と同じ frame を操作できます。

これにより、処理を共通 function として切り出してもローカル状態を共有できます。

ただし、handle を受け取る補助 function はその frame を直接変更できるため、参照渡しと同様に副作用を意識してください。

---

## 9. frame の解放

frame関数の正常終了時は必ず、

```mcfunction
$function dsf:frame/release {handle:$(handle)}
```

を実行します。

release は、

- `dsf:frame <handle>`
- `dsf:handle used.<handle>`

を削除します。

これにより、handle は将来の呼び出しで再利用可能になります。

### 異常終了時

コマンドエラーや開発途中の処理などで release まで到達しなかった場合、frame が残留する可能性があります。

`dsf:frame/format` は、実行時点の `dsf:handle cache` に記録されている handle を基に frame を強制削除し、handle 管理状態を初期化します。

```mcfunction
/function dsf:frame/handle/issue_with_run {path:"dsf:frame/format", arg:{route:"init", elem:0}}
```

使用中の frame も対象になるため、通常処理中ではなくデバッグ・初期化用途で使用してください。

---

## 10. 記述規約

引数の順序が意味を持たない場合、Framelet では原則として次の順序を使用します。

1. `path`
2. `handle`
3. `route`
4. `elem`
5. その他の処理固有引数

例:

```mcfunction
/function dsfsample:sample {path:"dsfsample:sample", handle:1, route:"root", elem:1}
```

通常の利用では handle を手入力せず、`issue_with_run` から起動してください。

---

## 11. 注意点

### 1 tick に処理を集中させる

Framelet は schedule でループを分散しません。

そのため、大量の反復を行うと1 tickの負荷が増えます。Framelet は「任意の大規模処理を1 tickで安全に処理する仕組み」ではありません。

### frame は必ず release する

正常終了パスでは `release` に到達するようにしてください。

### 固定グローバル領域をローカル状態として追加しない

Framelet の再入可能性は、呼び出し固有の状態を handle 配下へ分離することで成立しています。

独自処理の途中で固定 scoreboard / 固定 storage を「その呼び出し専用の状態」として追加すると、再入可能性を崩す可能性があります。

共有レジスタを使用する場合は、値の寿命と再入時の上書きを明確にしてください。
