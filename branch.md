---
title: 'ブランチング'
teaching: 30
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- すべてを入力せずに多くのターゲットをどのように指定できますか？

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- ブランチングを使用してターゲットを指定できるようにする

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: instructor

エピソードの概要: ブランチングの使用方法を示す

:::::::::::::::::::::::::::::::::::::



## なぜブランチングなのか？

`targets` の大きな強みの一つは、**ブランチング**と呼ばれる、単一のコード行から多くのターゲットを定義できる能力です。
これは入力を省くだけでなく、タイプミスの可能性が減るため、**エラーのリスクも低減**します。

## ブランチングの種類

ブランチングには、**動的ブランチング**と**静的ブランチング**の二種類があります。
「ブランチング」とは、ターゲットを作成する方法（「パターン」）を単一に指定し、`targets` がそれから複数のターゲット（「ブランチ」）を生成するという考え方を指します。
「動的」とは、パターンから生成されるブランチが事前に定義されている必要がなく、コードの結果として動的に生成されることを意味します。

このワークショップでは、**動的ブランチングのみ**を扱います。静的ブランチングは[メタプログラミング](https://books.ropensci.org/targets/static.html#metaprogramming)の使用を必要とするため、これは高度なトピックです。どちらをいつ使用するか（または両方の組み合わせ）についての詳細は、[`targets` パッケージマニュアル](https://books.ropensci.org/targets/dynamic.html)を参照してください。

## ブランチングなしの例

これがどのように機能するかを理解するために、`palmerpenguins` データセットの分析を続けましょう。

**私たちの仮説は、くちばしの深さがくちばしの長さとともに減少するということです。**
この仮説を線形モデルで検証します。

例えば、これはくちばしの長さに依存するくちばしの深さのモデルです：


``` r
lm(bill_depth_mm ~ bill_length_mm, data = penguins_data)
```

これをパイプラインに追加できます。すべての種を区別せずに結合しているため、`combined_model` と呼びます：


``` r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build model
  combined_model = lm(
    bill_depth_mm ~ bill_length_mm,
    data = penguins_data
  )
)
```


``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw
```

``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw

+ combined_model dispatched
✔ combined_model completed [27ms, 11.20 kB]
✔ ended pipeline [231ms, 1 completed, 3 skipped]
```

モデルを見てみましょう。`broom` パッケージの `glance()` 関数を使用します。これは base R の `summary()` とは異なり、出力をティブル（データフレームの tidyverse 相当）として返します。後で見るように、これは下流の分析に非常に便利です。


``` r
library(broom)
tar_load(combined_model)
glance(combined_model)
```

``` output
# A tibble: 1 × 12
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1    0.0552        0.0525  1.92      19.9 0.0000112     1  -708. 1422. 1433.    1256.         340   342
```

小さな *P*-値に注目してください。
これはモデルが非常に有意であることを示しているようです。

しかし、ちょっと待ってください... これは本当に適切なモデルでしょうか？ データセットには3種類のペンギンがいることを思い出してください。くちばしの深さと長さの関係が**種によって異なる**可能性があります。

種ごとに1つずつモデル（合計3つ）を作成して、どうなるか試してみましょう（これは統計的には厳密には正しいアプローチではありませんが、ここでの目的は統計ではなく `targets` を学ぶことです）。

これでワークフローがより複雑になっています。これは**ブランチングなし**でのそのような分析のワークフローの例です（`packages.R` に `library(broom)` を追加することを忘れないでください）：


``` r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Build models
  combined_model = lm(
    bill_depth_mm ~ bill_length_mm,
    data = penguins_data
  ),
  adelie_model = lm(
    bill_depth_mm ~ bill_length_mm,
    data = filter(penguins_data, species == "Adelie")
  ),
  chinstrap_model = lm(
    bill_depth_mm ~ bill_length_mm,
    data = filter(penguins_data, species == "Chinstrap")
  ),
  gentoo_model = lm(
    bill_depth_mm ~ bill_length_mm,
    data = filter(penguins_data, species == "Gentoo")
  ),
  # Get model summaries
  combined_summary = glance(combined_model),
  adelie_summary = glance(adelie_model),
  chinstrap_summary = glance(chinstrap_model),
  gentoo_summary = glance(gentoo_model)
)
```


``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw
```

``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw

+ adelie_model dispatched
✔ adelie_model completed [18ms, 6.48 kB]
+ gentoo_model dispatched
✔ gentoo_model completed [2ms, 5.88 kB]
+ chinstrap_model dispatched
✔ chinstrap_model completed [2ms, 4.54 kB]
+ combined_summary dispatched
✔ combined_summary completed [35ms, 350 B]
+ adelie_summary dispatched
✔ adelie_summary completed [3ms, 351 B]
+ gentoo_summary dispatched
✔ gentoo_summary completed [2ms, 351 B]
+ chinstrap_summary dispatched
✔ chinstrap_summary completed [3ms, 351 B]
✔ ended pipeline [414ms, 7 completed, 4 skipped]
```

モデルの一つのサマリーを見てみましょう：


``` r
tar_read(adelie_summary)
```

``` output
# A tibble: 1 × 12
  r.squared adj.r.squared sigma statistic     p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>       <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1     0.153         0.148  1.12      27.0 0.000000667     1  -231.  468.  477.     188.         149   151
```

この方法でパイプラインを書くと機能しますが、繰り返しが多くなります。各モデルのサマリー統計量を取得するたびに `glance()` を呼び出さなければなりません。
さらに、各サマリーターゲット（`adelie_summary` など）は明示的に名前が付けられ、手動で入力されています。
タイプミスをして間違ったモデルがサマリーされるのはかなり簡単です。

先に進む前に、もう一つ**カスタム関数**を定義しましょう：`model_glance()` です。
`targets` を使用する際にはカスタム関数を頻繁に書く必要があるため、慣れておくと良いでしょう！

`model_glance()` という名前が示すとおり（目的がわかる名前を関数に付けるのは良い習慣です）、この関数はモデルを構築し、すぐにそのモデルに対して `glance()` を実行します。
こうする理由は、結果として**データフレームが得られる**ためで、次のセクションで見るように、これはブランチングに非常に役立ちます。
これを `R/functions.R` に保存してください：


``` r
model_glance_orig <- function(penguins_data) {
  model <- lm(
    bill_depth_mm ~ bill_length_mm,
    data = penguins_data)
  broom::glance(model)
}
```

## ブランチングを使用した例

### 最初の試み

**動的ブランチング**を使用して同じプランを書く方法を見てみましょう（実行した後で、新しいバージョンの各ステップを詳しく見ていきます）：


``` r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Group data
  tar_group_by(
    penguins_data_grouped,
    penguins_data,
    species
  ),
  # Build combined model with all species together
  combined_summary = model_glance(penguins_data),
  # Build one model per species
  tar_target(
    species_summary,
    model_glance(penguins_data_grouped),
    pattern = map(penguins_data_grouped)
  )
)
```

ここで何が起こっているのでしょうか？

まず、`tar_make()` が提供するメッセージを見てみましょう。


``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw
```

``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw

+ combined_summary dispatched
✔ combined_summary completed [38ms, 350 B]
+ penguins_data_grouped dispatched
✔ penguins_data_grouped completed [13ms, 1.53 kB]
+ species_summary declared [3 branches]
✔ species_summary completed [12ms, 1.05 kB]
✔ ended pipeline [389ms, 5 completed, 3 skipped]
```

一連の小さなターゲット（ブランチ）があり、それぞれが species_summary_7fe6634f7c7f6a77 のように名前付けされ、その後に全体の `species_summary` ターゲットがあります。
これはブランチングを使用してターゲットを指定した結果です：小さなターゲットそれぞれが全体のターゲットを構成する「ブランチ」です。
`targets` は、事前にどれだけのブランチが存在するか、またそれらが何を表しているかを知らないため、数字と文字の一連（「ハッシュ」）を使用して各ブランチに名前を付けます。
`targets` は各ブランチを一つずつビルドし、それらを全体のターゲットに結合します。

次に、ワークフローがどのように設定されているか、データの設定から詳しく見てみましょう：


``` r
  # Group data
  tar_group_by(
    penguins_data_grouped,
    penguins_data,
    species
  ),
```

ブランチングなしのバージョンとは異なり、**データをグループ化する**ステップを追加しました。
これは動的ブランチングが、グループ化されたデータフレームに同じ関数を適用する [`tidyverse` のアプローチ](https://dplyr.tidyverse.org/articles/grouping.html)に似ているためです。
そこで、`tar_group_by()` 関数を使用して、入力データのグループ（種ごとに1グループ）を指定します。

次に、ターゲット `species_summary` をビルドするコマンドを見てみましょう。


``` r
  # Build one model per species
  tar_target(
    species_summary,
    model_glance(penguins_data_grouped),
    pattern = map(penguins_data_grouped)
  )
```

以前と同様に、`tar_target()` の最初の引数はビルドするターゲットの名前で、二つ目の引数はそれをビルドするコマンドです。

ここでは、カスタム関数 `model_glance()` を `penguins_data_grouped` の各グループ（つまり各種）に適用しています。

最後に、これまで見たことのない引数 `pattern` があります。これは、このターゲットが動的ブランチングを使用してビルドされるべきことを示します。
`map` は、入力データ（`penguins_data_grouped`）の各グループに対して関数を順次適用することを意味します。

ブランチングワークフローの構築方法を理解したので、出力を検査してみましょう：


``` r
tar_read(species_summary)
```


``` output
# A tibble: 3 × 12
  r.squared adj.r.squared sigma statistic  p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>    <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1     0.153         0.148 1.12       27.0 6.67e- 7     1 -231.   468.  477.    188.          149   151
2     0.427         0.418 0.866      49.2 1.53e- 9     1  -85.7  177.  184.     49.5          66    68
3     0.414         0.409 0.754      85.5 1.02e-15     1 -139.   284.  292.     68.8         121   123
```

モデルのサマリー統計量がすべて一つのデータフレームに含まれています。

しかし、一つ問題があります：**どの行がどの種から来たのか分かりません！** 入力データと同じ順序であると仮定するのは賢明ではありません。

これは動的ブランチングの動作方法によるものです：デフォルトでは、各ターゲットの由来に関する情報が出力に保持されません。

これをどう修正すれば良いでしょうか？

### 第二の試み

ブランチングパイプラインから有用な出力を得るための鍵は、各ブランチの出力に必要な情報を含めることです。
ここでは、モデルサマリーの各行に対応する種を知りたいと考えています。

これは `model_glance` 関数を修正することで実現できます。種を表す列を含めるように修正したら、必ず保存してください：


``` r
model_glance <- function(penguins_data) {
  # Make model
  model <- lm(
    bill_depth_mm ~ bill_length_mm,
    data = penguins_data)
  # Get species name
  species_name <- unique(penguins_data$species)
  # If this is the combined dataset with multiple
  # species, changed name to 'combined'
  if (length(species_name) > 1) {
    species_name <- "combined"
  }
  # Get model summary and add species name
  glance(model) |>
    mutate(species = species_name, .before = 1)
}
```

新しいパイプラインは以前とまったく同じです。修正したのはパイプラインではなく**関数**です。

`targets` はカスタム関数の内容を追跡しているため、`species_summary` を再計算する必要があると判断し、新しく修正した関数を使ってこのターゲットを再び実行します。


``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw

+ combined_summary dispatched
✔ combined_summary completed [65ms, 371 B]
+ species_summary declared [3 branches]
✔ species_summary completed [23ms, 1.11 kB]
✔ ended pipeline [409ms, 4 completed, 4 skipped]
```

今回は、`model_summaries` をロードすると、各行がどのモデルに対応しているかを知ることができます（`mutate()` の `.before = 1` により、他の列より前に表示されます）。


``` r
tar_read(species_summary)
```

``` output
# A tibble: 3 × 13
  species   r.squared adj.r.squared sigma statistic  p.value    df logLik   AIC   BIC deviance df.residual  nobs
  <chr>         <dbl>         <dbl> <dbl>     <dbl>    <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1 Adelie        0.153         0.148 1.12       27.0 6.67e- 7     1 -231.   468.  477.    188.          149   151
2 Chinstrap     0.427         0.418 0.866      49.2 1.53e- 9     1  -85.7  177.  184.     49.5          66    68
3 Gentoo        0.414         0.409 0.754      85.5 1.02e-15     1 -139.   284.  292.     68.8         121   123
```

次に、モデルに基づくくちばしの深さの予測を追加します。これはレポートでモデルをプロットする際に必要になります。
この予測は `broom` パッケージの `augment()` 関数を使用して取得できます。モデルのサマリーの場合と同様に、予測された点をデータフレームとして出力するカスタム関数を作成します。


::::::::::::::::::::::::::::::::::::: {.challenge}

## チャレンジ: ワークフローにモデル予測を追加する

`augment()` を使用してモデル予測を追加できますか？ `glance()` と同様にカスタム関数を定義する必要があります。

:::::::::::::::::::::::::::::::::: {.solution}

新しい関数を `model_augment()` として定義します。これは `model_glance()` と同じですが、`glance()` の代わりに `augment()` を使用します：


``` r
model_augment <- function(penguins_data) {
  # Make model
  model <- lm(
    bill_depth_mm ~ bill_length_mm,
    data = penguins_data)
  # Get species name
  species_name <- unique(penguins_data$species)
  # If this is the combined dataset with multiple
  # species, changed name to 'combined'
  if (length(species_name) > 1) {
    species_name <- "combined"
  }
  # Get model summary and add species name
  augment(model) |>
    mutate(species = species_name, .before = 1)
}
```

ワークフローにステップを追加します：


``` r
source("R/functions.R")
source("R/packages.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean data
  penguins_data = clean_penguin_data(penguins_data_raw),
  # Group data
  tar_group_by(
    penguins_data_grouped,
    penguins_data,
    species
  ),
  # Get summary of combined model with all species together
  combined_summary = model_glance(penguins_data),
  # Get summary of one model per species
  tar_target(
    species_summary,
    model_glance(penguins_data_grouped),
    pattern = map(penguins_data_grouped)
  ),
  # Get predictions of combined model with all species together
  combined_predictions = model_augment(penguins_data_grouped),
  # Get predictions of one model per species
  tar_target(
    species_predictions,
    model_augment(penguins_data_grouped),
    pattern = map(penguins_data_grouped)
  )
)
```

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

### ワークフローをさらに簡素化する

ワークフローをさらに簡素化できることにお気づきかもしれません。`penguins_data` と `penguins_data_grouped` という別々のデータフレームを用意する必要はありません。
一般に、コードを理解しやすくするために、名前付きオブジェクトの数はできるだけ少なくするのが最善です。
クリーニングとグループ化のステップを1つのコマンドにまとめましょう：


``` r
source("R/functions.R")
source("R/packages.R")

tar_plan(
  # Load raw data
  tar_file_read(
    penguins_data_raw,
    path_to_file("penguins_raw.csv"),
    read_csv(!!.x, show_col_types = FALSE)
  ),
  # Clean and group data
  tar_group_by(
    penguins_data,
    clean_penguin_data(penguins_data_raw),
    species
  ),
  # Get summary of combined model with all species together
  combined_summary = model_glance(penguins_data),
  # Get summary of one model per species
  tar_target(
    species_summary,
    model_glance(penguins_data),
    pattern = map(penguins_data)
  ),
  # Get predictions of combined model with all species together
  combined_predictions = model_augment(penguins_data),
  # Get predictions of one model per species
  tar_target(
    species_predictions,
    model_augment(penguins_data),
    pattern = map(penguins_data)
  )
)
```

そして、もう一度実行します：


``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw
```

``` output

Attaching package: ‘palmerpenguins’

The following objects are masked from ‘package:datasets’:

    penguins, penguins_raw

+ penguins_data dispatched
✔ penguins_data completed [37ms, 1.53 kB]
+ combined_summary dispatched
✔ combined_summary completed [112ms, 371 B]
+ species_summary declared [3 branches]
✔ species_summary completed [28ms, 1.11 kB]
+ combined_predictions dispatched
✔ combined_predictions completed [7ms, 24.80 kB]
+ species_predictions declared [3 branches]
✔ species_predictions completed [16ms, 26.48 kB]
✔ ended pipeline [500ms, 9 completed, 2 skipped]
```

::::::::::::::::::::::::::::::::::::: {.callout}

## ブランチングのベストプラクティス

動的ブランチングは**データフレーム**と相性が良いように設計されています（[リスト](https://books.ropensci.org/targets/dynamic.html#list-iteration)も使用できますが、より高度なので、可能な限りデータフレームを使用することをお勧めします）。

カスタム関数は、データフレームを入力として受け取り、データフレームを出力として返すように書き、必要なメタデータを列として常に含めることをお勧めします。

:::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: {.challenge}

## チャレンジ: 他にどんな種類のパターンがありますか？

これまで、`pattern` 引数と組み合わせて `map()` を使用し、入力の各要素に対して関数を順次適用する単一の関数のみを使用しました。

ブランチングパターンを適用する他の方法を考えてみてください。

:::::::::::::::::::::::::::::::::: {.solution}

ブランチングパターンを適用する他の方法には以下のようなものがあります：

- crossing: 要素の組み合わせごとに一つのブランチを作成する（`cross()` 関数）
- slicing: 手動で選択した要素ごとに一つのブランチを作成する（`slice()` 関数）
- sampling: ランダムに選択した要素ごとに一つのブランチを作成する（`sample()` 関数）

ブランチングパターンの詳細については [`targets` マニュアル](https://books.ropensci.org/targets/dynamic.html#patterns) を参照してください。

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints 

- 動的ブランチングは単一のコマンドで複数のターゲットを作成します

- ブランチの出力に必要なメタデータを含めるために、通常カスタム関数を書く必要があります 

::::::::::::::::::::::::::::::::::::::::::::::::
