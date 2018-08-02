---
title: Morphological Anti-Aliasing [@Reshetov2011]
bibliography: bibliography.bib
numberSections: false
---
# MLAAとは？[What is MLAA?]

http://forums.anandtech.com/showthread.php?t=2126581

> MLAAはかなり古い…ので、特許が取られていません[not patented]。
> MLAAはテキストの文字がより良く見えるようにするために作られたと思いますが、最初にそれをもたらしたのがIMBかAPPLEかは思い出せません。
>
> NVIDIAは彼らが望むなら真似た何かを行うことができるでしょう。これはAAの無い古いゲームでいい感じです。
> 2010年12月10日 --- Arkadrel(Golden Member)

これはAnandTechのフォーラムのひとつからの引用である(原文ママ[orthography preserved])。

この発言にはいくつかの真実が存在する。MLAAはなんと特許取得されていない。
残念ながら、小さなテキストをひどく見えるようにする。そして、明らかに、NVIDIAは欲していたし、似た何かを行った。

Arkadrel、もしそこにいるなら、あなたの誠実な投稿をバカにすることを謝らせて欲しい。法的な理由のため、この投稿を示すたび詫びなければならない。

# MLAA回顧録[This talk: MLAA in retrospect]

- アーティスト回顧展[ARTIST RETROSPECTIVE]
- 前の酷いやつ[Early, Lousy Work]
- 後の良いやつ[Later, Better Work]
- 通路[Streeter]

MLAAとは実際は何なのかについて話していこう。
オリジナルの形態学的アンチエイリアシングアルゴリズムを2年の視点から述べようと思う。


<!-- p.5 -->

これはとても良い画に見えるかもしれないだろうが…

<!-- p.6 -->

だが、ズームインしてみると、実際には醜い[ugly]ことが分かるだろう。これらすべてジグザグなエッジ…現実世界ではまったく以て見ることはないだろう。

<!-- p.7 -->

一方で、依然として目、一般的な卵型の顔、その他の特徴を識別できる。

# 計画[The Plan]

1. 何とかして画像中に輪郭[silhouette]を見つける(そして、現実のオブジェクトと対応することを望む)。
2. 輪郭周りに色をブレンド(aka フィルタ)する。

# 間の有意な類似性…[Meaningful similarities between...]

- ポストプロセッシングのアンチエイリアシング
- 超解像度[super-resolution]
    - [@Fattal2007]
- コンピュータビジョン
- ⇒ 復元された(aka hallucinated)輪郭のエッジは画像の改良[enhancement]/認識[recognition]のために使われる。

輪郭はコンピュータサイエンス分野の至るところで役に立つ[come in handy]。

これらすべての手法では、もっともらしい輪郭はまず *hallucinated* され、画像処理で使われる。

# …と、ある重要な区別[...and one important distinction]

品質

- 3Dモデルデータ($\infty$解像度で利用可能)
- より良い輪郭を推測するために使える。
    - 方向に関して適応的なエッジフィルタ、DEAA、GBAA
- または、ピクセル内部の色以外のスーパーサンプル量
    - SRAA
- または、ピクセルあたりのサンプルをひとつだけ使うことを *選ん* でも良い。
    - 色か深度かその組み合わせのいずれか

単純さ

同時に、レンダリングでは、厳密な輪郭は利用可能な3Dデータから実際に計算される。そのようなアプローチはコースのあとの方に現れるだろう。

この追加データをすべて無視して、ピクセルあたりひとつの色サンプルで行うならば、可能な限り最も単純で最も自由自在[universal]なアルゴリズムを得るだろうが、この代償はクオリティにのしかかる。

# 動作する(ことが期待される)理由[Why (we hope) it will work]

- スーパーサンプリングアンチエイリアシング
    1. 各ピクセルをサンプルする。
    2. 計算した色を平均化する。

最適なクオリティは、もちろん、カメラや人間の目での統合処理を模倣するので、アンチエイリアシングの金字塔であるスーパーサンプリングで達成される可能性がある。

# 単純化[Simplifications]

- 2つの区別できるサンプルされた色を持つピクセルに対して、積分は面積の計算で近似できる。

いくつかのピクセルでは、サンプルされた色は大幅に異なることはなく、そのようなピクセルをスーパーサンプリングする不必要な処理が行われるだろう。

最も単純で非自明なケースは2つの区別できるサンプルされた色を持つひとつのピクセルである。通常は輪郭線がそのピクセルを通るときに起こる。

そのようなピクセルでは、積分が面積の計算で近似できる。

ピクセルあたりひとつの色しか知らないが、なんとか輪郭線を推定することができる場合、これでもなんとか動作するかもしれない。

# それは以前に行われていた…[It was done before...]

- とても単純なコンテンツでは、*ピクセルアートスケーリング* アルゴリズムが動作するかも。
- 元の低解像度のコンピュータゲームをより良いハードウェアで動作できるようにするために80年代に開発された。
- ([@Kopf2011]も参照)

それは以前に行われていた。80年代の初めにはピクセルアートスケーリングアルゴリズムが高解像度ディスプレイで昔のコンピュータゲームを動かすために開発された。

これらのアルゴリズムは高解像度のピクセルを計算するために現在のピクセルの小さな近傍を用いる。昔のゲームでは可能性のあるパターン数が限定されるので正常に動作する。

最近では、@Kopf2011 が高次の曲線を用いる巧み[clever]なデピクセル化アルゴリズムをもたらした。

# 必要なもの[What we need]

(どのピクセルが異なるかの)二値データ
↓
連続的な輪郭線

これらのアルゴリズムはリアルタイムには高価すぎる。

我々はピクセルアートアルゴリズムに刺激を受けつつも、一般のコンテンツが非ローカルなパターンを考慮することができるようにそれらを拡張する。

どのピクセルが異なるかを説明する二値データのみを用いて輪郭を再生成する。

これは唯一可能なアプローチではないが、その長所は単純さにある。この二値オラクルは色や深度のようないずれの入力を使うことができるが、その出力は与えられた2ピクセルに対して真か偽のいずれかである。

# ピクセルが異なるかを決める方法[How to decide if pixels are different]

- 色チャンネルごとのしきい値
    - ≠ 人間の視覚
    - 輪郭近くで照明が変化すると問題になる。
- 光度[luminosity] [ITU-R BT.709]
    - フォールスネガティブ
- 非線形しきい値処理(GOW)
    - 範囲全体で良好な検出
    - アーティストの調整が必要
- 深度のみ
    - 尺度の選択が難しい。
    - コーナーで問題になる。
- 深度＋色＋オブジェクトID＋…
    - 恐らく、最良のもの(データが使えれば)

最も単純な方法はチャンネルごと個別に色データのみを使うことである。この手法では、人間の視覚が非線形であるので、しきい値を選択するのが難しい。
小さな値は結果として必要のない輪郭となり、大きな値はすべての差異を見つけられなくなる。

他の可能性として、人間の視覚に合うよう設計されたITUの勧告[recommendation]に基づいて(例えば、10%で)光度を定量化することがある。光度だけを用いると、フォールスネガティブの可能性がある。

*God of War* が行ったように、非線形変換は色範囲全体で良好な検出を提供するために使われるかもしれない。

深度のみが用いられる場合、望ましくない特性を示す可能性もある。しかし、深度、色、オブジェクトIDの組み合わせが最適な選択のように思えるが、それはアプリケーションによる。JiminezのMLAAはこのアプローチを用いる。

<!-- p.16 -->

ピクセルの不連続性のデータはすべての場合で簡単に追跡可能な輪郭に必ずしも適していない。

if we go back to our fella, 彼の目は簡単に識別可能である。

同時に、彼は、恐らく不健康なダイエットにより、歯に問題を抱えている。

一般的な法則は、ピクセル間の差異が大きくなれば輪郭抽出がより信頼できるということである。
これは、最も目立つエイリアシングアーティファクトを排除することができるので、良いことである。

# MLAAのルールその1(2つのうち)[MLAA rule #1 (out of 2)]

- 輪郭は、水平および垂直の分割線が交差する所のピクセルのエッジで始まり/終わりを分割[segment]する。

輪郭を抽出するためのたった2つのルールだけがある。この画像において、黄色の線は異なるピクセルを分割する。

いくつかのピクセルは水平および垂直の分割線に接する(そのようなピクセルすべてはstripped shadingで示される)。

MLAAにおいて、輪郭はそのようなピクセルのエッジでのみ開始および終了する可能性がある。

最も単純な方法はハーフエッジを用いることであるだろうが、2009年のHPGの論文では、カラーバランスを用いて妨害[intercept]を見つける手の込んだ方法が提示された。

# MLAAのルール #2[MLAA rule #2]

- 分割線ごとに
    - 隣接する直交する線ですべての始点/終点を調べる。
    - 最長の線分を選ぶ。

輪郭線の終点を見つければ、それらをつながなければならない。

分割線ごとに、できるだけ最長の輪郭線分を選ぶ。

これはオリジナルのMLAAと、最初に見つけた輪郭が使われるJimenezのMLAAとで異なる。

# 論拠: オブジェクト交差[Rationale: object intersection]

- 鼻の上にメガネがあるけど、鼻の輪郭線を保存したい。

このための論拠は、このEdgarの画像に示されるように、他のオブジェクトと交差するにも関わらず、オブジェクトの輪郭を保存することである。

# オーバーブラーの回避[Avoiding over-blurring]

- 水平および垂直の両方の輪郭線が同じピクセルと交差する場合
    - 最長の輪郭線を選択する(これらのピクセルに対して垂直)。
    - または、いずれかひとつ(両方の長さが1の場合)。

水平および垂直の両方の輪郭線が同じピクセルと交差する場合、オーバーブラーを回避するために最長の輪郭線を選択する。

# 2つの形状の種類[Two type of shapes]

そして、最後に、輪郭の終点が分割線の反対側にある場合、いわゆるZ型[Z-shape]を得る。そうでなければ、2つの線分から構成されるU型[U-shape]が生成される。

# これが得るもの[This is what we will get]

すべてをまとめると、左から右に: 元画像、分割線、再構築された輪郭、アンチエイリアスした画像

# MLAAを一言で[MLAA in a one sentence]

1. 近傍から異なるピクセルすべてを検出する。
2. 輪郭を近似する。
3. この輪郭の周囲で色をフィルタする。

- 手順1と2は革新と差別化を可能にする。
- 手順3は(ガンマなしの)RGB空間で大丈夫なように見える。

一言で言えば、MLAAは、輪郭の周囲の色をフィルタするために、不連続性データからhallucinateされた輪郭を用いる。

他の規則でもまったくもって可能である。ほとんどは結果として良い状況では似た輪郭となり、悪い状況ではいずれにせよ十分な情報を持ち合わせていない。

# 当時(2009年)と現在(2011年)[Then (2009) and now (2011)]

|MLAAの落とし穴|できること|
|---------------------------------|-----------------------------------|
|非ローカルでCPUフレンドリーなフィルタは概念実証[proof-of-concept]と見なされる|GPU、PS3、Xbox、CPUに対する効率的な実装、それと同様の代替アルゴリズム|
|Nyquist限界でのアンダーサンプリング|SRAA、方向に関する適応的なエッジフィルタ|
|変化するライティングが静的なシーンで輪郭変化を誘発する|不連続性バッファ(JimenezのMLAA)|
|時間的なアーティファクト|時空間的アップサンプリング|
|潜在的な1フレームのレイテンシー|他のポストプロセッシング効果と並行に行う(*God Of War*)|

HPG2009で提示されたときの、オリジナルのMLAAアプローチは概念実証の側面が強かった。

実際に、ディファードレンダリングに言及さえせず、Leonardo da Vinci、Kazimir Malevich、Georges Seuratについて述べた。

今や、オリジナルのMLAAの欠点のほとんどはあれやこれやで対処されている。

IntelのソリューショングループのAlexandre De Pereyraにより実装された、約6msで実行される新しいCPUバージョンさえもある。

彼は徹底的[thorough]なジョブ最適化とコードのコメント付けを行った。あなたが理解できるコードを眺めたいと思うなら、このバージョンをダウンロードしよう。

# 2020年へのタイムライン？[Timeline for 2020?]

- AnandTechでのAA命名ガイド: 主なバリエーションに対して27エントリ
- 歴史的な視点: Zバッファは *ほか* のすべての不可視表面の除去アルゴリズムをコロした…
- ハードウェアAAはそうできなかった(まだ？)

AnandTechにあるアンチエイリアシング命名ガイドは27エントリがある。(きょう提示されたいくつかのテクニックでさえ未だに含まれていない)

それは問題と機会の両方を綴る[spell]。

ポストプロセッシングのアンチエイリアシングは長らく使われるのか、巧みなハードウェアマルチサンプリングが隆盛するのかは定かではない。

# じゃあ、質問なんだけど…[So the question is...]

- Retinaディスプレイ(~300dpi)はすべてのAAを殺すのだろうか？
    - (ワクワクさせてくれるよね)
- 最終結果[bottom line]: ポストプロセッシングAAアルゴリズムは以下になればやがて成熟し終わる。
    - 解像度があるアーティファクトを緩和するために十分良好であるとき
    - ただし、AAについて忘れてしまうほど大きくはなりすぎない程度に

主な疑問のひとつは、どれくらいのディスプレイ解像度が良いか、ということである。
これは私の元のスライドで、これらの問題を議論しており、それはカンファレンスのDVDに収録されている。しかし、。いくつかの理由により、complimentaryのiPadを受け取れなかった。

# じゃあ、質問なんだけど…(改定)[So the question is... (amended)]

- 300dpiでもAAのことを忘れるには十分ではない。
    - 人々は目の解像度より高周波での不連続性に気付くよう進化している(超視力[hyperacuity])。
- 更に読むなら (コースのウェブサイトを参照)
    - John Hableのブログ
    - Daved LuebkeのThe Ultimate Display

これが改訂された[revised]スライドである。見ての通り、300dpiでは十分でない。これは魅力的なトピックである。コースのウェブサイトでこれについてさらに読むことができる。

# 参考文献[References]