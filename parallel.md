---
title: '並列処理'
teaching: 10
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

このデモの目的のためにまだ1つだけ変更する必要があります：今、分析を並列で実行しても、関数が非常に高速であるため、計算時間の違いに気付かないでしょう。

そこで、`Sys.sleep()` 関数を使用して、`glance_with_mod_name()` と `augment_with_mod_name()` の「遅い」バージョンを作成しましょう。これはコンピュータに数秒待つよう指示します。
これにより、長時間実行される計算をシミュレートし、順次実行と並列実行の違いを確認できます。

これらの関数を `functions.R` に追加します（元のものをコピー＆ペーストしてから修正しても構いません）：


``` r
glance_with_mod_name_slow <- function(model_in_list) {
  Sys.sleep(4)
  model_name <- names(model_in_list)
  model <- model_in_list[[1]]
  broom::glance(model) |>
    mutate(model_name = model_name)
}
augment_with_mod_name_slow <- function(model_in_list) {
  Sys.sleep(4)
  model_name <- names(model_in_list)
  model <- model_in_list[[1]]
  broom::augment(model) |>
    mutate(model_name = model_name)
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
    glance_with_mod_name_slow(models),
    pattern = map(models)
  ),
  # Get model predictions
  tar_target(
    model_predictions,
    augment_with_mod_name_slow(models),
    pattern = map(models)
  )
)
```

最後に、通常どおり `tar_make()` を使用してパイプラインを実行します。


``` output
✔ skip target penguins_data_raw_file
✔ skip target penguins_data_raw
✔ skip target penguins_data
✔ skip target models
• start branch model_predictions_5ad4cec5
• start branch model_predictions_c73912d5
• start branch model_predictions_91696941
• start branch model_summaries_5ad4cec5
• start branch model_summaries_c73912d5
• start branch model_summaries_91696941
• built branch model_predictions_5ad4cec5 [4.884 seconds]
• built branch model_predictions_c73912d5 [4.896 seconds]
• built branch model_predictions_91696941 [4.006 seconds]
• built pattern model_predictions
• built branch model_summaries_5ad4cec5 [4.011 seconds]
• built branch model_summaries_c73912d5 [4.011 seconds]
• built branch model_summaries_91696941 [4.011 seconds]
• built pattern model_summaries
• end pipeline [15.153 seconds]
```

各個別ターゲットをビルドするのに約4秒かかるにもかかわらず、ワークフロー全体を実行するのにかかる総時間は、個々のターゲットの合計時間よりも短いことに注目してください！ これはプロセスが並列で実行されており、**時間を節約している** ことの証明です。

`targets` の独自で強力な点は、**並列で実行するためにカスタム関数を変更する必要がなかった** ことです。ワークフローを*調整*しただけです。これは、ワークフローを順次にローカルで実行するか、高性能なコンテキストで並列に実行するようにリファクタリング（修正）するのが比較的簡単であることを意味します。

これがどのように機能するかを実演したので、分析プランを作成した関数の元のバージョンに戻すことができます。

::::::::::::::::::::::::::::::::::::: keypoints 

- 動的ブランチングは単一のコマンドで複数のターゲットを作成します
- ブランチの出力に必要なメタデータを含めるために、通常カスタム関数を書く必要があります
- 並列コンピューティングは関数ではなく、ワークフローのレベルで機能します

::::::::::::::::::::::::::::::::::::::::::::::::
