---
title: 'Instructor Notes'
---

## 全般的な注意事項

例は徐々に、[Palmer Penguins データセット](https://allisonhorst.github.io/palmerpenguins/)の完全な分析へと発展していきます。このデータセットは、R の基本パッケージに `penguins` と `penguins_raw` として含まれています。最終版はこのリポジトリにあります。ワークフロー（`_targets.R`）は [`plan_11.R`](https://github.com/carpentries-incubator/targets-workshop/blob/main/episodes/files/plans/plan_11.R)（`episodes/files/plans/` にある番号付きプランファイルの最後のもの）、それが使用するカスタム関数は [`tar_functions`](https://github.com/carpentries-incubator/targets-workshop/tree/main/episodes/files/tar_functions)、レポートは [`penguin_report.qmd`](https://github.com/carpentries-incubator/targets-workshop/blob/main/episodes/files/penguin_report.qmd) です。

## R のバージョン

penguins データセットは **R バージョン 4.5.0** で基本パッケージに追加されたため、このレッスンには R 4.5.0 以降が必要です。ワークショップの前に、学習者（および使用予定の共有サーバーやクラウドインスタンス）が十分に新しいバージョンの R を使用していることを確認してください。

このレッスンは、ホスト済みの環境を提供していません。Posit Cloud などのクラウドサービスでワークショップを実施したい場合は、R 4.5.0 以降と[セットアップ手順](../learners/setup.md)に記載されているパッケージを備えた環境を、ご自身で用意する必要があります。

## オプションのエピソード

「パッケージの管理」のエピソードは、より発展的なトピック（パッケージを読み込むさまざまな方法、`renv` によるパッケージバージョンの管理、名前空間の衝突）を扱っており、時間が足りない場合は省略できます。レッスンの残りの部分はこのエピソードに依存しません。ワークフローはすでに `library()` でパッケージを読み込んでいます。

「関数の簡単な紹介」のエピソードはオプションのエピソードであり、ワークショップに参加する学習者によって必要かどうかが変わります。
関数を使った経験がある人はいるか、挙手（または付箋）で確認し、経験のない学習者がいる場合にこのエピソードを実施することをお勧めします。

`targets` は関数に大きく依存しているため、関数に不慣れな学習者がいる場合は、少し時間をかける価値があると考えています。関数について簡単な導入を受けないと、そうした学習者はすぐについていけなくなり、ワークショップの終了時に `targets` を使いこなせるようにならないでしょう。
