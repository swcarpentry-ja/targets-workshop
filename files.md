---
title: '外部ファイルの取り扱い'
teaching: 10
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- 外部データをどのようにロードできますか？

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- ワークフローに外部データをロードできるようにする
- 外部データの内容が変更された場合にワークフローを再実行するように設定する

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: instructor

エピソードの概要: 外部ファイルの読み書き方法を示す

:::::::::::::::::::::::::::::::::::::



## 外部ファイルを依存関係として扱う

ほとんどすべてのワークフローはデータのインポートから始まります。データは通常、外部ファイルとして保存されています。

簡単な例として、RStudioの「新しいファイル」メニューオプションを使用して外部データファイルを作成しましょう。「Hello World」という一行のテキストを入力し、`_targets/user/data/` に "hello.txt" テキストファイルとして保存します。

次に、このファイルの内容を読み込み、ワークフロー内で `some_data` として保存するために、以下のプランを書いて `tar_make()` を実行します：

::::::::::::::::::::::::::::::::::::: {.callout}

## 進捗の保存

1つのプロジェクト内でアクティブな `_targets.R` ファイルは1つだけです。

新しい `_targets.R` ファイルを作成しようとしていますが、これまで作業してきたもの（ペンギンのくちばし分析）の進捗を失いたくないでしょう。そのファイルを一時的に `_targets_old.R` のような名前に変更することで、以下の新しい例の `_targets.R` ファイルを上書きしないようにできます。再び作業を再開するときに名前を戻してください。

:::::::::::::::::::::::::::::::::::::


``` r
library(targets)
library(tarchetypes)

tar_plan(
  some_data = readLines("_targets/user/data/hello.txt")
)
```


``` output
▶ dispatched target some_data
● completed target some_data [0.001 seconds]
▶ ended pipeline [0.086 seconds]
```

`tar_read(some_data)` を使用して `some_data` の内容を検査すると、期待通り `"Hello World"` という文字列が含まれていることがわかります。

次に、"hello.txt" を編集して、テキストを追加します。例えば、「Hello World. How are you?」としましょう。これをRStudioのテキストエディタで編集して保存します。次にパイプラインを再実行します。


``` r
library(targets)
library(tarchetypes)

tar_plan(
  some_data = readLines("_targets/user/data/hello.txt")
)
```


``` output
✔ skipped target some_data
✔ skipped pipeline [0.086 seconds]
```

ターゲット `some_data` がスキップされましたが、これはファイルの内容が変更されたにもかかわらずです。

これは、現在のところ `targets` がファイルの**名前のみ**を追跡しており、その内容を追跡していないためです。これを行うには、`tarchetypes` パッケージの `tar_file()` 関数を使用する必要があります。`tar_file()` はファイルの「ハッシュ」を計算します。これはファイルの内容によって決定される一意のデジタル署名です。内容が変更されると、ハッシュも変更され、`targets` によって検出されます。


``` r
library(targets)
library(tarchetypes)

tar_plan(
  tar_file(data_file, "_targets/user/data/hello.txt"),
  some_data = readLines(data_file)
)
```


``` output
▶ dispatched target data_file
● completed target data_file [0.001 seconds]
▶ dispatched target some_data
● completed target some_data [0 seconds]
▶ ended pipeline [0.129 seconds]
```

今回は、`targets` が期待通りに `some_data` を再構築するのが確認できます。

## ショートカット（または、ターゲットファクトリーについて）

しかし、これにより、一つのターゲットではなく二つのターゲットを書く必要があることにも気づきます。ファイルの内容を追跡するターゲット（`data_file`）と、ファイルからロードした内容を保存するターゲット（`some_data`）です。

これは `targets` ワークフローでは一般的なパターンであるため、`tarchetypes` はこれをより簡潔に表現するショートカット、`tar_file_read()` を提供しています。


``` r
library(targets)
library(tarchetypes)

tar_plan(
  tar_file_read(
    hello,
    "_targets/user/data/hello.txt",
    readLines(!!.x)
  )
)
```

このプランを `tar_manifest()` で検査してみましょう：


``` r
tar_manifest()
```


``` output
# A tibble: 2 × 2
  name       command                           
  <chr>      <chr>                             
1 hello_file "\"_targets/user/data/hello.txt\""
2 hello      "readLines(hello_file)"           
```

`tar_file_read()` を使用してパイプラインに一つのターゲット（`hello`）のみを指定しましたが、実際には **二つ** のターゲット、`hello_file` と `hello` が含まれていることに気づきます。

これは `tar_file_read()` が **ターゲットファクトリー** と呼ばれる特別な関数だからです。ターゲットファクトリーは一度に**複数**のターゲットを作成します。`tarchetypes` パッケージの主な目的の一つは、パイプラインの記述を容易にし、エラーを減らすためにターゲットファクトリーを提供することです。

## 非標準評価

`!!.x` の意味は何でしょうか？これはRの使用に慣れていても馴染みがないかもしれません。これは「非標準評価」として知られ、特定のコンテキストで使用されます。詳細については時間がありませんが、`tar_file_read()` を使用する際にはこの特別な記法を使用する必要があることを覚えておいてください。書き方を忘れた場合（これは頻繁に起こります！）、`?tar_file_read` を実行してヘルプファイルの例を参照してください。

## 他のデータ読み込み関数

ここでは `readLines()` を例として使用しましたが、`readr::read_csv()`、`xlsx::read_excel()` など、外部ファイルからデータを読み込む他の関数でも同じパターンを使用できます（例えば、`read_csv(!!.x)`、`read_excel(!!.x)` など）。
    
これは一般的に推奨されます。そうすることで、入力データとパイプラインが同期し、常に最新の状態に保たれます。

::::::::::::::::::::::::::::::::::::: {.challenge}

## Challenge: penguins の例で `tar_file_read()` を使用する

ペンギンのくちばし分析を開始したとき、まだ `tar_file_read()` を知りませんでした。

`tar_file_read()` を使用してCSVファイルを読み込み、その内容を追跡するにはどうすればよいですか？

:::::::::::::::::::::::::::::::::: {.solution}


``` r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  penguins_data = clean_penguin_data(penguins_data_raw)
)
```


``` output
▶ dispatched target penguins_data_raw_file
● completed target penguins_data_raw_file [0.001 seconds]
▶ dispatched target penguins_data_raw
● completed target penguins_data_raw [0.102 seconds]
▶ dispatched target penguins_data
● completed target penguins_data [0.015 seconds]
▶ ended pipeline [0.382 seconds]
```

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

## データの書き出し

ファイルへの書き出しは、ファイルの読み込みと似ています。`tar_file()` 関数を使用します。ただし、重要な注意点があります：この場合、`tar_file()` の第二引数（ターゲットをビルドするためのコマンド）は**ファイルへのパスを返さなければなりません**。すべてのファイルを書き出す関数がこれを行うわけではありません（一部は何も返さず、ファイルの出力を関数の副作用として扱います）。そのため、ファイルを書き出し、そのパスを返すカスタム関数を定義する必要があるかもしれません。


``` r
x <- writeLines("some text", "test.txt")
x
```


``` output
NULL
```

ここでは、文字データをファイルに書き出し、そのファイル名を返す修正済み関数を作成します（`...` は「これらの引数の残りを `writeLines()` に渡す」を意味します）：


``` r
write_lines_file <- function(text, file, ...) {
  writeLines(text = text, con = file, ...)
  file
}
```

これを試してみましょう：


``` r
x <- write_lines_file("some text", "test.txt")
x
```


``` output
[1] "test.txt"
```

これで、この関数をパイプラインで使用できます。例えば、テキストを大文字に変換して再度書き出してみましょう：


``` r
library(targets)
library(tarchetypes)

source("R/functions.R")

tar_plan(
  tar_file_read(
    hello,
    "_targets/user/data/hello.txt",
    readLines(!!.x)
  ),
  hello_caps = toupper(hello),
  tar_file(
    hello_caps_out,
    write_lines_file(hello_caps, "_targets/user/results/hello_caps.txt")
  )
)
```


``` output
▶ dispatched target hello_file
● completed target hello_file [0 seconds]
▶ dispatched target hello
● completed target hello [0 seconds]
▶ dispatched target hello_caps
● completed target hello_caps [0 seconds]
▶ dispatched target hello_caps_out
● completed target hello_caps_out [0 seconds]
▶ ended pipeline [0.109 seconds]
```

`results` フォルダ内の `hello_caps.txt` を見て、期待通りであることを確認してください。

::::::::::::::::::::::::::::::::::::: {.challenge}

## Challenge: What happens to file output if its modified?

Delete or change the contents of `hello_caps.txt` in the `results` folder.
What do you think will happen when you run `tar_make()` again?
Try it and see.

:::::::::::::::::::::::::::::::::: {.solution}

`targets` は `hello_caps_out` が変更された（「無効化された」）ことを検出し、再構築のためにコードを再実行します。これにより、`hello_caps.txt` が再度 `results` に書き出されます。

この方法で結果を出力することで、パイプラインがより堅牢になります。つまり、`results` 内のファイルの内容がプラン内のコードによってのみ生成されることが保証されます。

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints 

- `tarchetypes::tar_file()` はファイルの内容を追跡します
- `tarchetypes::tar_file_read()` をデータ読み込み関数（例えば `read_csv()`）と組み合わせて使用し、入力データとパイプラインを同期させる
- `tarchetypes::tar_file()` をファイルに書き出す関数（ファイルに書き込んでパスを返す関数）と組み合わせてデータを書き出す

::::::::::::::::::::::::::::::::::::::::::::::::