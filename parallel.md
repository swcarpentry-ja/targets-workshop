---
title: '並列処理'
teaching: 15
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- `targets` のターゲットを並列でビルドするにはどうすればよいですか？

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- ターゲットを並列でビルドできるようにする

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: instructor

エピソードの概要: 並列処理の使用方法を示す

:::::::::::::::::::::::::::::::::::::



パイプラインに多くのターゲットが含まれ始めたら、並列処理を考えるかもしれません。
これはコンピュータの複数のプロセッサを活用して、同時に複数のターゲットをビルドします。

::::::::::::::::::::::::::::::::::::: {.callout}

## 並列処理を使用するタイミング

並列処理は、ワークフローに独立したタスクがある場合にのみ使用すべきです---ワークフローがターゲットの線形シーケンスのみで構成されている場合、並列化するものはありません。
ブランチングを使用するほとんどのワークフローは並列処理の恩恵を受けることができます。

:::::::::::::::::::::::::::::::::::::

`targets` は高性能コンピューティング、クラウドコンピューティング、およびさまざまな並列バックエンドをサポートしています。
ここでは、この分析をラップトップで実行していると仮定し、比較的シンプルなバックエンドを使用します。
高性能コンピューティングに興味がある場合は、[`targets` マニュアル](https://books.ropensci.org/targets/hpc.html) を参照してください。

### ワークフローのセットアップ

`crew` を使用して並列処理を有効にするには、`crew` パッケージをロードし、`tar_option_set` を使用して `targets` にそれを使用するように指示するだけです。
具体的には、以下の行が `crew` を有効にし、2つの並列ワーカーを使用するように指示します。
より強力なマシンでは、この数を増やすことができます：

```r
library(crew)
tar_option_set(
  controller = crew_controller_local(workers = 2)
)
```

ペンギンの分析にこれらの変更を加えましょう。
現在は次のようになっているはずです：


``` r
source("R/functions.R")
source("R/packages.R")

# Set up parallelization
library(crew)
tar_option_set(
  controller = crew_controller_local(workers = 2)
)

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

このデモの目的のためにまだ1つだけ変更する必要があります：今、分析を並列で実行しても、関数が非常に高速であるため、計算時間の違いに気付かないでしょう。

そこで、`Sys.sleep()` 関数を使用して、`model_glance()` と `model_augment()` の「遅い」バージョンを作成しましょう。これはコンピュータに数秒待つよう指示します。
これにより、長時間実行される計算をシミュレートし、順次実行と並列実行の違いを確認できます。

これらの関数を `functions.R` に追加します（元のものをコピー＆ペーストしてから修正しても構いません）：


``` r
model_glance_slow <- function(penguins_data) {
  Sys.sleep(4)
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
model_augment_slow <- function(penguins_data) {
  Sys.sleep(4)
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

次に、プランを「遅い」バージョンの関数を使用するように変更します：


``` r
source("R/functions.R")
source("R/packages.R")

# Set up parallelization
library(crew)
tar_option_set(
  controller = crew_controller_local(workers = 2)
)

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
  combined_summary = model_glance_slow(penguins_data),
  # Get summary of one model per species
  tar_target(
    species_summary,
    model_glance_slow(penguins_data),
    pattern = map(penguins_data)
  ),
  # Get predictions of combined model with all species together
  combined_predictions = model_augment_slow(penguins_data),
  # Get predictions of one model per species
  tar_target(
    species_predictions,
    model_augment_slow(penguins_data),
    pattern = map(penguins_data)
  )
)
```

最後に、通常どおり `tar_make()` を使用してパイプラインを実行します。


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
+ species_summary declared [3 branches]
+ species_predictions declared [3 branches]
✔ combined_summary completed [4.1s, 371 B]
+ combined_predictions dispatched
✔ species_summary completed [13.2s, 1.11 kB]
✔ combined_predictions completed [4s, 24.80 kB]
✔ species_predictions completed [12s, 26.48 kB]
✔ ended pipeline [19.1s, 8 completed, 3 skipped]
```

各個別ターゲットをビルドするのに約4秒かかるにもかかわらず、ワークフロー全体を実行するのにかかる総時間は、個々のターゲットの合計時間よりも短いことに注目してください！ これはプロセスが並列で実行されており、**時間を節約している** ことの証明です。

`targets` の独自で強力な点は、**並列で実行するためにカスタム関数を変更する必要がなかった** ことです。ワークフローを*調整*しただけです。これは、ワークフローを順次にローカルで実行するか、高性能なコンテキストで並列に実行するようにリファクタリング（修正）するのが比較的簡単であることを意味します。

これがどのように機能するかを実演したので、分析プランを作成した関数の元のバージョンに戻すことができます。

::::::::::::::::::::::::::::::::::::: keypoints 

- 動的ブランチングは単一のコマンドで複数のターゲットを作成します
- ブランチの出力に必要なメタデータを含めるために、通常カスタム関数を書く必要があります
- 並列コンピューティングは関数ではなく、ワークフローのレベルで機能します

::::::::::::::::::::::::::::::::::::::::::::::::
