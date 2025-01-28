---
title: 'ブランチング'
teaching: 10
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
✔ skipped target penguins_data_raw_file
✔ skipped target penguins_data_raw
✔ skipped target penguins_data
▶ dispatched target combined_model
● completed target combined_model [0.049 seconds]
▶ ended pipeline [0.169 seconds]
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

おそらく、種に対するパラメータを追加するモデルや、種とくちばしの長さの相互作用効果を追加するモデルなど、いくつかの代替モデルをテストする必要があります。

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
  species_model = lm(
    bill_depth_mm ~ bill_length_mm + species,
    data = penguins_data
  ),
  interaction_model = lm(
    bill_depth_mm ~ bill_length_mm * species,
    data = penguins_data
  ),
  # Get model summaries
  combined_summary = glance(combined_model),
  species_summary = glance(species_model),
  interaction_summary = glance(interaction_model)
)
```


``` output
✔ skipped target penguins_data_raw_file
✔ skipped target penguins_data_raw
✔ skipped target penguins_data
✔ skipped target combined_model
▶ dispatched target interaction_model
● completed target interaction_model [0.003 seconds]
▶ dispatched target species_model
● completed target species_model [0.001 seconds]
▶ dispatched target combined_summary
● completed target combined_summary [0.007 seconds]
▶ dispatched target interaction_summary
● completed target interaction_summary [0.003 seconds]
▶ dispatched target species_summary
● completed target species_summary [0.003 seconds]
▶ ended pipeline [0.153 seconds]
```

モデルの一つのサマリーを見てみましょう：


``` r
tar_read(species_summary)
```

``` output
# A tibble: 1 × 12
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1     0.769         0.767 0.953      375. 3.65e-107     3  -467.  944.  963.     307.         338   342
```

この方法でパイプラインを書くと機能しますが、繰り返しが多くなります。各モデルのサマリー統計量を取得するたびに `glance()` を呼び出さなければなりません。
さらに、各サマリータゲット（`combined_summary` など）は明示的に名前が付けられ、手動で入力されています。
タイプミスをして間違ったモデルがサマリーされるのはかなり簡単です。

## ブランチングを使用した例

### 最初の試み

**動的ブランチング**を使用して同じプランを書く方法を見てみましょう：


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
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
  # Get model summaries
  tar_target(
    model_summaries,
    glance(models[[1]]),
    pattern = map(models)
  )
)
```

ここで何が起こっているのでしょうか？

まず、`tar_make()` が提供するメッセージを見てみましょう。


``` output
✔ skipped target penguins_data_raw_file
✔ skipped target penguins_data_raw
✔ skipped target penguins_data
▶ dispatched target models
● completed target models [0.014 seconds]
▶ dispatched branch model_summaries_812e3af782bee03f
● completed branch model_summaries_812e3af782bee03f [0.007 seconds]
▶ dispatched branch model_summaries_2b8108839427c135
● completed branch model_summaries_2b8108839427c135 [0.002 seconds]
▶ dispatched branch model_summaries_533cd9a636c3e05b
● completed branch model_summaries_533cd9a636c3e05b [0.003 seconds]
● completed pattern model_summaries
▶ ended pipeline [0.158 seconds]
```

一連の小さなターゲット（ブランチ）があり、それぞれが model_summaries_812e3af782bee03f のように名前付けされ、その後に全体の `model_summaries` ターゲットがあります。
これはブランチングを使用してターゲットを指定した結果です：小さなターゲットそれぞれが全体のターゲットを構成する「ブランチ」です。
`targets` は、事前にどれだけのブランチが存在するか、またそれらが何を表しているかを知らないため、数字と文字の一連（「ハッシュ」）を使用して各ブランチに名前を付けます。
`targets` は各ブランチを一つずつビルドし、それらを全体のターゲットに結合します。

次に、ワークフローがどのように設定されているか、モデルの定義から詳しく見てみましょう：


``` r
  # Build models
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
```

ブランチングなしのバージョンとは異なり、モデルを**リスト内**で定義しました（モデルごとに一つのターゲットではなく）。
これは動的ブランチングが `base::apply()` や [`purrrr::map()`](https://purrr.tidyverse.org/reference/map.html) のループ方法に似ているためです：リストの各要素に関数を適用します。
したがって、ループの入力としてリストを準備する必要があります。

次に、ターゲット `model_summaries` をビルドするコマンドを見てみましょう。


``` r
  # Get model summaries
  tar_target(
    model_summaries,
    glance(models[[1]]),
    pattern = map(models)
  )
```

以前と同様に、最初の引数はビルドするターゲットの名前で、二つ目の引数はそれをビルドするコマンドです。

ここでは、`glance()` 関数を `models` の各要素に適用しています（`[[1]]` が必要なのは、関数が適用されるとき、各要素が実際にはネストされたリストであり、ネストを一層取り除く必要があるためです）。

最後に、これまで見たことのない引数 `pattern` があります。これは、このターゲットが動的ブランチングを使用してビルドされるべきことを示します。
`map` は、入力リスト（`models`）の各要素に対して関数を順次適用することを意味します。

ブランチングワークフローの構築方法を理解したので、出力を検査してみましょう：


``` r
tar_read(model_summaries)
```


``` output
# A tibble: 3 × 12
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int>
1    0.0552        0.0525 1.92       19.9 1.12e-  5     1  -708. 1422. 1433.    1256.         340   342
2    0.769         0.767  0.953     375.  3.65e-107     3  -467.  944.  963.     307.         338   342
3    0.770         0.766  0.955     225.  8.52e-105     5  -466.  947.  974.     306.         336   342
```

モデルのサマリー統計量がすべて一つのデータフレームに含まれています。

しかし、一つ問題があります：**どの行がどのモデルから来たのか分かりません！** モデルのリストと同じ順序であると仮定するのは賢明ではありません。

これは動的ブランチングの動作方法によるものです：デフォルトでは、各ターゲットの由来に関する情報が出力に保持されません。

これをどう修正すれば良いでしょうか？

### 第二の試み

ブランチングパイプラインから有用な出力を得るための鍵は、各ブランチの出力に必要な情報を含めることです。
ここでは、モデルサマリーの各行に対応するモデルの種類を知りたいと考えています。
これを行うために、**カスタム関数**を書く必要があります。
`targets` を使用する際にはカスタム関数を頻繁に書く必要があるため、慣れておくと良いでしょう！

以下がその関数です。これを `R/functions.R` に保存してください：


``` r
glance_with_mod_name <- function(model_in_list) {
  model_name <- names(model_in_list)
  model <- model_in_list[[1]]
  glance(model) |>
    mutate(model_name = model_name)
}
```

新しいパイプラインは以前とほぼ同じですが、今回は `glance()` の代わりにカスタム関数を使用しています。


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
  # Build models
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
  # Get model summaries
  tar_target(
    model_summaries,
    glance_with_mod_name(models),
    pattern = map(models)
  )
)
```


``` output
✔ skipped target penguins_data_raw_file
✔ skipped target penguins_data_raw
✔ skipped target penguins_data
✔ skipped target models
▶ dispatched branch model_summaries_812e3af782bee03f
● completed branch model_summaries_812e3af782bee03f [0.013 seconds]
▶ dispatched branch model_summaries_2b8108839427c135
● completed branch model_summaries_2b8108839427c135 [0.006 seconds]
▶ dispatched branch model_summaries_533cd9a636c3e05b
● completed branch model_summaries_533cd9a636c3e05b [0.003 seconds]
● completed pattern model_summaries
▶ ended pipeline [0.161 seconds]
```

今回は、`model_summaries` をロードすると、各行がどのモデルに対応しているかを知ることができます（右にスクロールする必要があるかもしれません）。


``` r
tar_read(model_summaries)
```

``` output
# A tibble: 3 × 13
  r.squared adj.r.squared sigma statistic   p.value    df logLik   AIC   BIC deviance df.residual  nobs model_name       
      <dbl>         <dbl> <dbl>     <dbl>     <dbl> <dbl>  <dbl> <dbl> <dbl>    <dbl>       <int> <int> <chr>            
1    0.0552        0.0525 1.92       19.9 1.12e-  5     1  -708. 1422. 1433.    1256.         340   342 combined_model   
2    0.769         0.767  0.953     375.  3.65e-107     3  -467.  944.  963.     307.         338   342 species_model    
3    0.770         0.766  0.955     225.  8.52e-105     5  -466.  947.  974.     306.         336   342 interaction_model
```

次に、モデルに基づくくちばしの深さの予測を追加します。これはレポートでモデルをプロットする際に必要になります。
この予測は `broom` パッケージの `augment()` 関数を使用して取得できます。


``` r
tar_load(models)
augment(models[[1]])
```

``` output
# A tibble: 342 × 8
   bill_depth_mm bill_length_mm .fitted .resid    .hat .sigma   .cooksd .std.resid
           <dbl>          <dbl>   <dbl>  <dbl>   <dbl>  <dbl>     <dbl>      <dbl>
 1          18.7           39.1    17.6  1.14  0.00521   1.92 0.000924      0.594 
 2          17.4           39.5    17.5 -0.127 0.00485   1.93 0.0000107    -0.0663
 3          18             40.3    17.5  0.541 0.00421   1.92 0.000168      0.282 
 4          19.3           36.7    17.8  1.53  0.00806   1.92 0.00261       0.802 
 5          20.6           39.3    17.5  3.06  0.00503   1.92 0.00641       1.59  
 6          17.8           38.9    17.6  0.222 0.00541   1.93 0.0000364     0.116 
 7          19.6           39.2    17.6  2.05  0.00512   1.92 0.00293       1.07  
 8          18.1           34.1    18.0  0.114 0.0124    1.93 0.0000223     0.0595
 9          20.2           42      17.3  2.89  0.00329   1.92 0.00373       1.50  
10          17.1           37.8    17.7 -0.572 0.00661   1.92 0.000296     -0.298 
# ℹ 332 more rows
```

::::::::::::::::::::::::::::::::::::: {.challenge}

## チャレンジ: ワークフローにモデル予測を追加する

`augment()` を使用してモデル予測を追加できますか？ `glance()` と同様にカスタム関数を定義する必要があります。

:::::::::::::::::::::::::::::::::: {.solution}

新しい関数を `augment_with_mod_name()` として定義します。これは `glance_with_mod_name()` と同じですが、`glance()` の代わりに `augment()` を使用します：


``` r
augment_with_mod_name <- function(model_in_list) {
  model_name <- names(model_in_list)
  model <- model_in_list[[1]]
  augment(model) |>
    mutate(model_name = model_name)
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
  # Build models
  models = list(
    combined_model = lm(
      bill_depth_mm ~ bill_length_mm, data = penguins_data),
    species_model = lm(
      bill_depth_mm ~ bill_length_mm + species, data = penguins_data),
    interaction_model = lm(
      bill_depth_mm ~ bill_length_mm * species, data = penguins_data)
  ),
  # Get model summaries
  tar_target(
    model_summaries,
    glance_with_mod_name(models),
    pattern = map(models)
  ),
  # Get model predictions
  tar_target(
    model_predictions,
    augment_with_mod_name(models),
    pattern = map(models)
  )
)
```

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: {.callout}

## ブランチングのベストプラクティス

動的ブランチングは**データフレーム**（ティブル）と相性が良いように設計されています。

可能であれば、カスタム関数をデータフレームを入力として受け取り、データフレームを出力として返すように書き、必要なメタデータを列として常に含めるようにしてください。

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
