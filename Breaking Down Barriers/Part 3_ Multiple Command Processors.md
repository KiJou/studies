---
title: >
    BREAKING DOWN BARRIERS - PART 3: MULTIPLE COMMAND PROCESSORS
---
# BREAKING DOWN BARRIERS - PART 3: MULTIPLE COMMAND PROCESSORS

## FLUSHING PERFORMANCE DOWN THE DRAIN

<Part 1, 2の復習>

より良いアイデアをもたらすため、より現実に則した例に切り替えよう。GPUでグラフィクス処理を行う実際のエンジンでは、それぞれが前の段に依存する描画またはディスパッチの**チェイン**を持つのが非常に一般的である。

![](images/bloom_timeline.png)

これを同じくらいの時間かかる大きなディスパッチとオーバーラップしたいとしよう。ステップひとつとオーバーラップするのは非常に単純である。しかし、そのGPUが、前に起動したシェーダが実行を終了するまで待機する必要があるコマンドに関してバリアを実装する我々の架空のGPUと似たセットアップを持つ場合、理想的ではないかもしれない。その場合、長いディスパッチの後の最初のバリアはオーバーラップのない長い期間を生じさせるかもしれないだろう。

![](images/bloom_timeline_overlap.png)

これはないよりましだが、素晴らしいものではない。関連するキャッシュのフラッシュやその他の行動はシェーダユニットが完全にアイドル状態となる期間であるので、我々は長いディスパッチをバリアのひとつに完全にオーバーラップさせたい。我々はこれを分割バリアで達成できるが、SIGNAL_POST_SHADER命令がすべての以前にキューへ追加されたスレッドが完了した後にシグナルするよう制限される場合、一度しかこれを行うことができないだろう。

![](images/bloom_timeline_overlap_split.png)

## TWO COMMAND PROCESSORS ARE BETTER THAN ONE

全く新しいMJP-4000を設計するとき、ハードウェアエンジニアは複数の独立したコマンドストリームを扱う最も単純な方法が一歩進んでGPUのフロントエンドをコピー＆ペーストすることであると決定した。

![](images/multi_queue_overview.png)

2つの別個のスレッドキューにアタッチされた2つのコマンドプロセッサがある。シェーダコア数はMJP-3000と同じであり、最大スループットの理論値は増えていない。しかし、2つのコマンドプロセッサを用いることで、シェーダコアのアイドル時間を減らし、実行するジョブの全体的なスループットを改善する。

図を見ると、"一体どうやって2つのフロントエンドがシェーダコアを共有するのか？"と(当然)不思議に思うかもしれない。実際のGPUでは、これを行ういくつかの取り得る方法がある。しかし今は、二重のスレッドキューがコアを共有するのに非常に単純なスキームを用いると仮定しよう。

- 1つのキューだけが待機中のスレッドを持ち、いずれのシェーダコアも空ある場合、そのキューは仕事でこれらのコアを埋める
- 両方のキューが仕事を持ち、空のコアがある場合、これらのコアは平等に分割され、両方のスレッドキューからスレッドを割り当てられる(奇数の場合、上のキューが追加のコアを得る)
- スレッドは、ひとたびシェーダコアに割り当てられれば、常に完了するまで実行する。これは待機中のスレッドが割り込みできないことを意味する

では、このすべてが実際にどう動くかを見てみよう。

![](images/multi_queue_0000_layer-1.png)

詳細は少し複雑だが、言うなれば、**2つのコマンドバッファを一度に実行することによって、一度にひとつを実行した場合と比べてより高い総使用率を得た**。しかし、この高使用率は個々のチェインごとのより高い**レイテンシー**のコストに原因があった。別の言い方をすれば、GPUの**全体のスループット**を改善した。

深度のみのレンダリングでは、シェーダコアは頂点シェーダで頂点を変換するために使われるが、ピクセルシェーダは使わない。頂点数はピクセル数と比べてかなり少ない傾向にあるので、固定機能によってボトルネックになってしまうことはこれらのパスでは容易に起こる。そうなれば、その頂点シェーダはシェーダコアの一部しか使わないので、他のコマンドストリームが活用できるような多くのアイドル状態のコアが残ることになる。

このマルチキュー構成をCPU世界で言うなれば、Simultneous Multithreading;SMT(Intelで言う所のHyper-Threading)に当たる。

## SYNCING ACROSS QUEUES

これまで、チェイン1とチェイン2は完全に独立であると考えていた。しかし、単一のアプリケーションでは単一点から開始する2つの異なるチェインを持つことがかなり一般的であり、この最終結果はもうひとつの単一のディスパッチによって必要とされる。より具体的な例として、メインパスをレンダターゲットにレンダリングした後に、ブルームとDOFの2つのポストプロセッシング処理を行い、両方の結果をスクリーン上で合成するとしよう。これらの処理の依存性グラフを作るとすれば、以下のような感じになるだろう。

![](images/bloom_dof_taskgraph.png)

グラフから、ブルームとDOFのチェインは独立であるのは明らかだが、チェインの開始と終了で同期する必要がある。MJP-4000の二重フロントエンドにサブミットされる別個のコマンドバッファとして実行されるとした場合、正しい結果を保証するためのある種のキュー間同期を必要とするだろう。

![](images/bloom_dof_combined2.png)

MJP-4000SIGNAL_POST_SHADERとWAIT_SIGNAL命令を再利用することによる単純なアプローチを取る。