---
title: '外部ファイルの取り扱い'
teaching: 25
exercises: 10
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

ここまでのペンギンのワークフローは、プロジェクトの `data/` フォルダにある CSV ファイルからデータを読み込んできました。
ここでは、自分たちで作成する、もう1つのずっと小さなデータファイルを追加しましょう。ペンギンが測定された島に関する情報です。
これは後でレポートで使用します。

RStudio の「New File」メニューオプションで新しいテキストファイルを作成します。
次の行を入力（コピー＆ペーストでも構いません）し、プロジェクトの `data/` フォルダに "islands.csv" として保存してください（このフォルダは[最初のワークフローのエピソード](episodes/basic-targets.Rmd)で作成しました。まだない場合は、今プロジェクトフォルダ内に作成してください）。
これらの座標はおおよその値です。

```
island,latitude,longitude
Biscoe,-65.43,-65.50
Dream,-64.73,-64.23
Torgersen,-64.77,-64.08

```

このファイルの内容を読み込み、ワークフロー内で `island_data` として保存します。
`_targets.R` のプランの最後に新しいターゲットを追加し（直前のターゲットの後にカンマが必要になることに注意してください）、`tar_make()` を実行します：


``` r
  # Get predictions of one model per species
  tar_target(
    species_predictions,
    penguins_data |> dplyr::filter(species == species_name) |> model_augment(),
    pattern = map(species_name)
  ),
  # Read island data
  island_data = read_csv("data/islands.csv", show_col_types = FALSE)
)
```


``` output
+ island_data dispatched
✔ island_data completed [80ms, 365 B]
+ species_summary declared [3 branches]
+ species_predictions declared [3 branches]
✔ ended pipeline [303ms, 1 completed, 12 skipped]
```

`tar_load(island_data)` で `island_data` をロードして確認すると、期待通り3つの島が含まれています。

ここで、トルガーセン島のより正確な緯度が分かり、"islands.csv" を編集して `-64.77` を `-64.80` に変更したとしましょう。RStudio のテキストエディタで編集して保存してください。そして、もう一度パイプラインを実行します。


``` r
tar_make()
```

``` output
+ species_summary declared [3 branches]
+ species_predictions declared [3 branches]
✔ skipped pipeline [139ms, 13 skipped]
```

ファイルの内容が変更されたにもかかわらず、ターゲット `island_data` はスキップされました。

これは、現時点では targets がファイルの**名前**だけを追跡しており、その内容を追跡していないためです。そのためには特別な関数、`tarchetypes` パッケージの `tar_file()` を使用する必要があります。`tar_file()` はファイルの「ハッシュ」を計算します---これはファイルの内容によって決まる一意のデジタル署名です。内容が変更されるとハッシュも変わり、`targets` がそれを検出します。

`_targets.R` のプランの最後を、島のファイルに `tar_file()` を使用するように変更し、`tar_make()` を実行します：


``` r
  # Get predictions of one model per species
  tar_target(
    species_predictions,
    penguins_data |> dplyr::filter(species == species_name) |> model_augment(),
    pattern = map(species_name)
  ),
  # Read island data
  tar_file(island_file, "data/islands.csv"),
  island_data = read_csv(island_file, show_col_types = FALSE)
)
```

その後、もう一度 "islands.csv" を編集します。たとえば、ドリーム島の経度を `-64.23` から `-64.25` に変更してみましょう。保存して、もう一度パイプラインを実行します。


``` output
+ island_file dispatched
✔ island_file completed [0ms, 91 B]
+ island_data dispatched
✔ island_data completed [67ms, 358 B]
+ species_summary declared [3 branches]
+ species_predictions declared [3 branches]
✔ ended pipeline [337ms, 2 completed, 12 skipped]
```

今回は、`targets` が期待通り `island_data` を再構築することが分かります。

## 他のデータ読み込み関数

これには2つのターゲットが必要であることに注意してください。1つはファイルの内容を追跡するもの（`tar_file()` で作成する `island_file`）、もう1つはファイルから読み込んだものを保存するもの（`island_data`）です。

ここでは例として `read_csv()` を使用しましたが、外部ファイルからデータを読み込む他の関数、たとえば `readxl::read_excel()` や `readLines()` などにも同じパターンを使用できます（たとえば、`tar_file(excel_file, "data.xlsx")` の後に `my_data = read_excel(excel_file)` とします）。

パイプラインを入力データと同期させるために、一般的にこれが推奨されます。

::::::::::::::::::::::::::::::::::::: {.challenge}

## チャレンジ: penguins の例で `tar_file()` を使用する

penguins のくちばし分析を始めた時点では `tar_file()` についてまだ知りませんでした。そのため、`targets` はペンギンの CSV ファイルの**名前**だけを追跡しており、その内容は追跡していません。

`tar_file()` を使用して、内容を追跡しながら CSV ファイルを読み込むにはどうすればよいでしょうか？

:::::::::::::::::::::::::::::::::: {.solution}

プランの最初にある `penguins_csv_file` ターゲットに `tar_file()` を使用します：


``` r
tar_plan(
  # Load raw data
  tar_file(penguins_csv_file, "data/penguins_raw.csv"),
  penguins_data_raw = read_csv(penguins_csv_file, show_col_types = FALSE),
```


``` output
+ penguins_csv_file dispatched
✔ penguins_csv_file completed [0ms, 57.98 kB]
+ penguins_data_raw dispatched
✔ penguins_data_raw completed [73ms, 10.40 kB]
+ species_summary declared [3 branches]
+ species_predictions declared [3 branches]
✔ ended pipeline [356ms, 2 completed, 12 skipped]
```

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

## データの書き出し

ファイルへの書き出しは、ファイルの読み込みと似ています。`tar_file()` 関数を使用します。これが機能するのは、`tar_file()` がファイルが入力か出力かを区別しないためです。どちらの場合も、ターゲットの値はファイルへの**パス**であり、`targets` はファイルの名前だけでなく内容（ハッシュ）も追跡します。ファイルが最初からそこにあったもの（"islands.csv" のように）か、ターゲットをビルドするコマンドによって作成されるものかは関係ありません。ただし、重要な注意点があります：この場合、`tar_file()` の第二引数（ターゲットをビルドするためのコマンド）は**ファイルへのパスを返さなければなりません**。すべてのファイルを書き出す関数がこれを行うわけではありません（一部は書き出したデータを返したり、何も返さなかったりして、ファイルの出力を関数の副作用として扱います）。そのため、ファイルを書き出し、そのパスを返すカスタム関数を定義する必要があるかもしれません。

ここでは、データフレームを CSV ファイルに書き出す `readr` の関数 `write_csv()` で試してみましょう。この関数の出力は、書き出されたデータであり、ファイルへのパスではありません。次のように確認できます：


``` r
x <- readr::write_csv(head(penguins, 3), "test.csv")
x
```


``` output
  species    island bill_len bill_dep flipper_len body_mass    sex year
1  Adelie Torgersen     39.1     18.7         181      3750   male 2007
2  Adelie Torgersen     39.5     17.4         186      3800 female 2007
3  Adelie Torgersen     40.3     18.0         195      3250 female 2007
```

ここでは、データフレームを CSV ファイルに書き出し、そのファイル名を返す修正済み関数を示します（`...` は「これらの引数の残りを `write_csv()` に渡す」を意味します）。これを `R/functions.R` に追加してください：


``` r
write_csv_file <- function(data, file, ...) {
  readr::write_csv(x = data, file = file, ...)
  file
}
```

これを試してみましょう：


``` r
x <- write_csv_file(head(penguins, 3), "test.csv")
x
```


``` output
[1] "test.csv"
```

これでワークフローで使用できます。例えば、プランの最後にターゲットをもう1つ追加して、各種のモデルの結果（`species_summary`）を書き出してみましょう：


``` r
  # Read island data
  tar_file(island_file, "data/islands.csv"),
  island_data = read_csv(island_file, show_col_types = FALSE),
  # Write out the summary of one model per species
  tar_file(
    species_summary_file,
    write_csv_file(species_summary, "results/species_summary.csv")
  )
)
```


``` output
+ species_summary declared [3 branches]
+ species_predictions declared [3 branches]
+ species_summary_file dispatched
✔ species_summary_file completed [64ms, 664 B]
✔ ended pipeline [344ms, 1 completed, 14 skipped]
```

`results` フォルダ内の `species_summary.csv` を見て、期待通りであることを確認してください。

::::::::::::::::::::::::::::::::::::: {.challenge}

## チャレンジ: ファイル出力が変更された場合はどうなりますか？

`results` フォルダ内の `species_summary.csv` を削除するか、その内容を変更してください。
もう一度 `tar_make()` を実行すると、どうなると思いますか？
実際に試してみましょう。

:::::::::::::::::::::::::::::::::: {.solution}

`targets` は `species_summary_file` が変更された（「無効化された」）ことを検出し、再構築のためにコードを再実行します。これにより、`species_summary.csv` が再度 `results` に書き出されます。

この方法で結果を出力することで、パイプラインがより堅牢になります。つまり、`results` 内のファイルの内容がプラン内のコードによってのみ生成されることが保証されます。

::::::::::::::::::::::::::::::::::

:::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::: keypoints 

- `tarchetypes::tar_file()` はファイルの内容を追跡します
- `tarchetypes::tar_file()` をデータ読み込み関数（例えば `read_csv()`）と組み合わせて使用し、入力データとパイプラインを同期させる
- `tarchetypes::tar_file()` をファイルに書き出す関数（ファイルに書き込んでパスを返す関数）と組み合わせてデータを書き出す

::::::::::::::::::::::::::::::::::::::::::::::::
