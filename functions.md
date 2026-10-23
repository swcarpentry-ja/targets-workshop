---
title: '関数の簡単な紹介'
teaching: 20
exercises: 10
---

:::::::::::::::::::::::::::::::::::::: questions 

- 関数とは何ですか？
- なぜ関数の書き方を知っておくべきですか？
- 関数の主な構成要素は何ですか？

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- カスタム関数の有用性を理解する
- 関数を書くための基本的な概念を理解する

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: {.instructor}

Episode summary: 関数の使用経験がない学習者がいる場合のための、関数のごく簡単な紹介。

::::::::::::::::::::::::::::::::::::::::::::::::



## 関数について

Rの関数は、パッケージから提供されるものだと考えるのに慣れているかもしれません。パッケージから専門的な関数を探してインストールし、作業を進めるために使用します。

しかし、自分で関数を書くこともできますし、書くべきだとも言えるでしょう！
関数は、同じ操作を異なる設定で簡単に繰り返すための優れた方法です。
スクリプト内でまったく同じコードをコピー＆ペーストし、いくつかの部分（変数や入力など）だけを変更して再び実行したことが何度ありますか？
その後でコードにエラーがあることに気づき、修正するときには、そのコードをコピーしたすべての場所で修正することを覚えておかなければなりません。

関数を書くことで、こうした手戻りを減らし、より効率的なワークフローを作ることができます。
バグを見つけたときは、自分で作った関数という1か所を修正するだけで、その後のその関数の呼び出しはすべて修正されたものになります。

さらに、`targets` はカスタム関数を多用するため、関数がどのように動作するかについての基本的な理解は、`targets` をうまく使うためにとても重要です。

### 関数を書く

自分で関数を書くことと、Rで他のコードを書くことの間に大きな違いはありません。あなたはこれまでどおりRでコーディングしているのです！
ペンギンのデータにあるミリメートルの測定値を、センチメートルに変換したいとしましょう（`head()` を使って最初の数行だけを表示します）。


``` r
library(tidyverse)

penguins |>
  mutate(
    bill_len_cm = bill_len / 10,
    bill_dep_cm = bill_dep / 10
  ) |>
  head()
```

``` output
  species    island bill_len bill_dep flipper_len body_mass    sex year
1  Adelie Torgersen     39.1     18.7         181      3750   male 2007
2  Adelie Torgersen     39.5     17.4         186      3800 female 2007
3  Adelie Torgersen     40.3     18.0         195      3250 female 2007
4  Adelie Torgersen       NA       NA          NA        NA   <NA> 2007
5  Adelie Torgersen     36.7     19.3         193      3450 female 2007
6  Adelie Torgersen     39.3     20.6         190      3650   male 2007
  bill_len_cm bill_dep_cm
1        3.91        1.87
2        3.95        1.74
3        4.03        1.80
4          NA          NA
5        3.67        1.93
6        3.93        2.06
```

これは複雑な操作ではありませんが、それでもこの変換を行ってくれる便利なカスタム関数を作りたいかもしれません。

関数を書くには、`function()` 関数を使用します。
この関数では、関数の入力引数となるものを括弧の中に、そしてそれらの入力引数を使って関数が行う処理を、括弧の後ろの波括弧 `{}` の中に記述します。
これを代入するオブジェクトの名前が、そのまま関数の名前になります。


``` r
my_function <- function(argument1, argument2) {
  # the things the function will do
}
# call the function
my_function(1, "something")
```

mm から cm への変換を行う関数は次のようになります：


``` r
mm2cm <- function(x) {
  x / 10
}
```

このカスタム関数は、任意の数値の入力を10で割って変換します。

試してみましょう：


``` r
penguins |>
  mutate(
    bill_len_cm = mm2cm(bill_len),
    bill_dep_cm = mm2cm(bill_dep)
  ) |>
  head()
```

``` output
  species    island bill_len bill_dep flipper_len body_mass    sex year
1  Adelie Torgersen     39.1     18.7         181      3750   male 2007
2  Adelie Torgersen     39.5     17.4         186      3800 female 2007
3  Adelie Torgersen     40.3     18.0         195      3250 female 2007
4  Adelie Torgersen       NA       NA          NA        NA   <NA> 2007
5  Adelie Torgersen     36.7     19.3         193      3450 female 2007
6  Adelie Torgersen     39.3     20.6         190      3650   male 2007
  bill_len_cm bill_dep_cm
1        3.91        1.87
2        3.95        1.74
3        4.03        1.80
4          NA          NA
5        3.67        1.93
6        3.93        2.06
```

おめでとうございます、最初のカスタム関数を作成して使用しました！

### 既存のコードから関数を作る

すでに手元にあるコードを使って関数を作りたい場合も多いでしょう。
たとえば、あるコードを何度もコピー＆ペーストしていて、そのコードが繰り返しになっていることに気づいたとき、関数にするのが適切です。
あるいは、ワークフローを `targets` に変換する際に、スクリプトを `targets` が呼び出す一連の関数に変更する必要がある場合もあります。

ペンギンのデータをクリーニングするために使ったコードを思い出してください：


``` r
penguins_data_raw |>
  select(
    species = Species,
    bill_length_mm = `Culmen Length (mm)`,
    bill_depth_mm = `Culmen Depth (mm)`
  ) |>
  drop_na()
```

このコードを関数にするために調整する必要があります。この関数には、クリーニング対象のデータセットという引数が1つ必要です。

次のようになります：

``` r
clean_penguin_data <- function(penguins_data_raw) {
  penguins_data_raw |>
    select(
      species = Species,
      bill_length_mm = `Culmen Length (mm)`,
      bill_depth_mm = `Culmen Depth (mm)`
    ) |>
    drop_na()
}
```

この関数を、`_targets.R` の `library()` でパッケージを読み込む部分の後、末尾のリストの前に追加してください。

::::::::::::::::: callout

# RStudioの関数抽出機能

RStudioには、コードの一部から関数を抽出してくれる便利な機能もあります。
関数に少し慣れてくれば、コードを関数にする際に必要な入力を見極めるのに役立つでしょう。

使うには、関数にしたいコードの部分をハイライトします。
今回の場合は、`penguins_data_raw` から `drop_na()` 文までのパイプライン全体です。
ハイライトしたら、RStudioの上部バーの「Code」から「Extract function」を選択します。
Enterキーを押すよう求めるプロンプトが開き、カーソルがあった位置に次のようなコードがスクリプトに挿入されます。

ただし、この関数は引数として必要以上のものを含んでいるため、そのままでは動作しません。
これは、tidyverse が非標準評価を使用しており、`select()` の中で引用符なしの列名を書けるためです。
関数抽出機能は、コード内の引用符なし（またはバッククォートで囲まれた）テキストはすべてオブジェクトへの参照だと判断してしまいます。
関数を動作させるには手作業での修正が必要になるため、すでに関数に少し慣れていたほうが便利です。

::::::::::::::::::

::::::::::::::::::::::::::::::::::::: {.challenge}

## チャレンジ: 数値ベクトルを受け取り、その平均を10で割った値を返す関数を書いてください。

:::::::::::::::::::::::::::::::::: {.solution}


``` r
vecmean <- function(x) {
  mean(x) / 10
}
```

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

## ワークフローで関数を使う

カスタムのデータクリーニング関数を定義したので、ワークフローで使ってみましょう。

どのようにすればよいか、わかりますか？

最後の `tar_target()` から対応するコードを削除し、新しい関数の呼び出しに置き換える必要があります。

ワークフローを次のように修正してください：


``` r
library(targets)
library(tidyverse)

clean_penguin_data <- function(penguins_data_raw) {
  penguins_data_raw |>
    select(
      species = Species,
      bill_length_mm = `Culmen Length (mm)`,
      bill_depth_mm = `Culmen Depth (mm)`
    ) |>
    drop_na()
}

list(
  tar_target(penguins_csv_file, "data/penguins_raw.csv"),
  tar_target(penguins_data_raw, read_csv(
    penguins_csv_file, show_col_types = FALSE)),
  tar_target(penguins_data, clean_penguin_data(penguins_data_raw))
)
```

ワークフローが最新の状態であることを確認するため、`tar_make()` でもう一度実行しましょう：


``` r
tar_make()
```

``` output
+ penguins_data dispatched
✔ penguins_data completed [11ms, 1.61 kB]
✔ ended pipeline [138ms, 1 completed, 2 skipped]
```

`targets()` が出力するメッセージについては、すぐ後で詳しく学びます。

## 関数によってコードの理解が容易になる

末尾のターゲットのリストが、分析の高レベルな要約のように見え始めていることに注目してください。

これはカスタム関数を使うもう一つの利点です：**関数を使うことで、各ワークフローステップの詳細を、ワークフロー全体から切り離すことができます**。

ワークフロー全体を理解するために、データがどのようにクリーニングされたかの詳細をすべて知る必要はありません。クリーニングのステップがあったことさえわかれば十分です。
一方、データクリーニングの具体的な内容に立ち返って詳しく調べる必要がある場合は、その関数の内部で何が行われているかだけに注目すればよく、ワークフローの残りの部分は無視できます。
**これによりコードの理解が容易になり**、バグが減り、最終的に時間と労力の節約につながります。

ここでは関数のほんの入り口に触れただけですので、関数について学ぶにはさらに助けが必要になるでしょう。
詳しくは、Carpentries の R Novice レッスンにある、[関数についてのエピソード](https://swcarpentry.github.io/r-novice-gapminder/10-functions.html)を読むことをお勧めします。

::::::::::::::::::::::::::::::::::::: keypoints 

- 同じコードを、わずかな違いで何度も繰り返す場合に、関数は欠かせません
- RStudioの「Extract function」ツールは、コードを関数に変換する作業を始める助けになります
- 関数は `targets` の仕組みに欠かせない要素です

::::::::::::::::::::::::::::::::::::::::::::::::
