---
title: 'targetsプロジェクト組織のベストプラクティス'
teaching: 10
exercises: 2
---

:::::::::::::::::::::::::::::::::::::: questions 

- `targets` プロジェクトを整理するためのベストプラクティスは何ですか？
- `targets` のワークフローの組織はスクリプトベースの分析とどのように異なりますか？

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: objectives

- 最大限の再現性のために `targets` プロジェクトをどのように整理するかを説明する
- `targets` の文脈で関数をどのように使用するかを理解する

::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: instructor

Episode summary: プロジェクト組織のベストプラクティスを実演する

:::::::::::::::::::::::::::::::::::::



## ワークフロープランをより簡単に書く方法

プラン内でターゲットを指定するデフォルトの方法は、`tar_target()` 関数を使うことです。
しかし、この書き方は少し冗長に感じるかもしれません。

その代わりに、`targets`の開発者であるWill Landauによって作成された `tarchetypes` パッケージを使う方法があります。

::::::::::::::::::::::::::::::::::::: prereq

## `tarchetypes` のインストール

まだインストールしていない場合は、`install.packages("tarchetypes")` で `tarchetypes` をインストールしてください。

:::::::::::::::::::::::::::::::::::::

`tarchetypes` の目的は、`targets` パイプラインの記述を容易にするさまざまなショートカットを提供することです。

今回はそのうちの一つ、`tar_plan()` を紹介します。これは `_targets.R` スクリプトの最後にある `list()` の代わりに使用されます。

`tar_plan()` を使用することで、`tar_target()` を使用してターゲットを指定する代わりに、`target_name = target_command` のような構文を使用できます。

ペンギンのワークフローを `tar_plan()` 構文を使用するように編集しましょう：

<!-- The chunk below intersperses plan_2b with clean_penguin_data() to avoid writing it manually -->

``` r
library(targets)
library(tarchetypes)
library(palmerpenguins)
library(tidyverse)

clean_penguin_data <- function(penguins_data_raw) {
  penguins_data_raw |>
    select(
      species = Species,
      bill_length_mm = `Culmen Length (mm)`,
      bill_depth_mm = `Culmen Depth (mm)`
    ) |>
    remove_missing(na.rm = TRUE) |>
    # Split "species" apart on spaces, and only keep the first word
    separate(species, into = "species", extra = "drop")
}

tar_plan(
  penguins_csv_file = path_to_file("penguins_raw.csv"),
  penguins_data_raw = read_csv(penguins_csv_file, show_col_types = FALSE),
  penguins_data = clean_penguin_data(penguins_data_raw)
)
```

読みやすくなったと思いませんか？

`tar_plan()` を使用するからといって、すべてのターゲットをこの方法で書かなければならないわけではありません。`tar_plan()` 内で `tar_target()` フォーマットを使用することもできます。

これは、`=` が短く読みやすい一方で、`targets` が提供できるすべてのカスタマイズを提供しないためです。

今のところあまり重要ではありませんが、より高度な `targets` ワークフローを作成し始めると重要になります。

## ファイルとフォルダの整理

これまで、すべてを単一の `_targets.R` ファイルで行ってきました。

これは小規模なワークフローには問題ありませんが、ワークフローが大きくなるとあまりうまく機能しません。

コードを整理するためのより良い方法があります。

まず、`_targets.R` 以外の R コードを保存するために `R` というディレクトリを作成しましょう（`_targets.R` はサブディレクトリではなく、プロジェクト全体のディレクトリに配置する必要があることを覚えておいてください）。

`R/` 内に `functions.R` という新しい R ファイルを作成します。
ここにカスタム関数を配置します。
今すぐ `clean_penguin_data()` をそこに入れて保存しましょう。

同様に、`library()` 呼び出しを `R/` 内の `packages.R` という独自のスクリプトに配置しましょう（ただし、これは唯一の方法ではありません。["パッケージの管理" エピソード](https://joelnitta.github.io/targets-workshop/packages.html) を参照してください）。

また、`_targets.R` スクリプトをこれらのスクリプトを `source` で呼び出すように修正する必要があります：


``` r
source("R/packages.R")
source("R/functions.R")

tar_plan(
  penguins_csv_file = path_to_file("penguins_raw.csv"),
  penguins_data_raw = read_csv(penguins_csv_file, show_col_types = FALSE),
  penguins_data = clean_penguin_data(penguins_data_raw)
)
```

これで `_targets.R` はずっとスリムになりました：ワークフローに集中し、各ステップで何が起こるかをすぐに教えてくれます。

最後に、データや出力など、コードではないファイルを保存するためのディレクトリを作成しましょう。
ターゲットキャッシュ内に `user` という新しいディレクトリを作成します：`_targets/user`。
`user` 内にさらに `data` と `results` の2つのディレクトリを作成します。
（バージョン管理を使用している場合は、`_targets` ディレクトリを無視することをおそらく望むでしょう）。

## 関数についての一言

このレッスンの前半でカスタム関数について触れましたが、これはさらに明確化が必要な重要なトピックです。
`targets` のような単一のワークフローではなく、複数のスクリプトを使用して R でデータを分析することに慣れている場合、多くの関数（`function()` 関数を使用）を書かないかもしれません。

これは `targets` との大きな違いです。
カスタム関数を使用せずに効率的な `targets` パイプラインを書くのは非常に難しいでしょう。なぜなら、ビルドする各ターゲットが単一のコマンドの出力でなければならないからです。

このカリキュラムでは R での関数の書き方をカバーする時間がありませんが、このトピックを復習するためには [Software Carpentry のレッスン](https://swcarpentry.github.io/r-novice-gapminder/10-functions) をお勧めします。

もう一つの大きな違いは、**各ターゲットが一意の名前を持たなければならない** ということです。
以下のようなコードを書くことに慣れているかもしれません：


``` r
# 人の身長をcmで保存し、インチに変換する
height <- 160
height <- height / 2.54
```

同等の `targets` パイプラインを実行しようとするとエラーが発生します：


``` r
tar_plan(
    height = 160,
    height = height / 2.54
)
```


``` error
Error:
! Error running targets::tar_make()
Error messages: targets::tar_meta(fields = error, complete_only = TRUE)
Debugging guide: https://books.ropensci.org/targets/debugging.html
How to ask for help: https://books.ropensci.org/targets/help.html
Last error message:
    duplicated target names: height
Last error traceback:
    base::tryCatch(base::withCallingHandlers({ NULL base::saveRDS(base::do.c...
    tryCatchList(expr, classes, parentenv, handlers)
    tryCatchOne(tryCatchList(expr, names[-nh], parentenv, handlers[-nh]), na...
    doTryCatch(return(expr), name, parentenv, handler)
    tryCatchList(expr, names[-nh], parentenv, handlers[-nh])
    tryCatchOne(expr, names, parentenv, handlers[[1L]])
    doTryCatch(return(expr), name, parentenv, handler)
    base::withCallingHandlers({ NULL base::saveRDS(base::do.call(base::do.ca...
    base::saveRDS(base::do.call(base::do.call, base::c(base::readRDS("/tmp/R...
    base::do.call(base::do.call, base::c(base::readRDS("/tmp/Rtmp8yhWqG/call...
    (function (what, args, quote = FALSE, envir = parent.frame()) { if (!is....
    (function (targets_function, targets_arguments, options, envir = NULL, s...
    tryCatch(out <- withCallingHandlers(targets::tar_callr_inner_try(targets...
    tryCatchList(expr, classes, parentenv, handlers)
    tryCatchOne(expr, names, parentenv, handlers[[1L]])
    doTryCatch(return(expr), name, parentenv, handler)
    withCallingHandlers(targets::tar_callr_inner_try(targets_function = targ...
    targets::tar_callr_inner_try(targets_function = targets_function, target...
    pipeline_from_list(targets)
    pipeline_from_list.default(targets)
    pipeline_init(out)
    pipeline_targets_init(targets, clone_targets)
    tar_assert_unique_targets(names)
    tar_throw_validate(message)
    tar_error(message = paste0(...), class = c("tar_condition_validate", "ta...
    rlang::abort(message = message, class = class, call = tar_empty_envir)
    signal_abort(cnd, .file)
```

**`targets` パイプラインで作業する大部分は、適切なサイズのカスタム関数を書くことです。**
それらは単一行のコードだけになるほど小さくてはいけません；そうするとパイプラインが理解しにくくなり、維持管理が難しくなります。
一方で、変更に過度に敏感になるほど大きくしてはいけません。

このバランスを取ることは科学というよりもアートであり、練習を通じてしか習得できません。私が見つけた良い経験則は、ターゲットごとに3つを超える入力を持たないことです。

::::::::::::::::::::::::::::::::::::: keypoints 

- コードを `R/` フォルダに配置する
- 関数を `R/functions.R` に配置する
- パッケージを `R/packages.R` に指定する
- その他の雑多なファイルを `_targets/user` に配置する
- 関数を書くことは `targets` パイプラインの重要なスキルである

::::::::::::::::::::::::::::::::::::::::::::::::

