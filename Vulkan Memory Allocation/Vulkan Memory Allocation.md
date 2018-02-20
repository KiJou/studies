---
title: Vulkan Memory Allocation [@Sellers2016]
bibliography: bibliography.bib
numberSections: false
---
# POOLED OBJECTS

## プールされたオブジェクト(Pooled Objects)

### バッチ生成(Batch Creation)

- 一部のオブジェクトはプールに割り当てられる。
    - コマンドバッファ。
    - デスクリプタ。
    - クエリ。
- 各型には関連するプール型が存在する。
    - `VkCommandPool`
    - `VkDescriptorPool`
    - `VkQueryPool`

### スレッド安全性(Thread Safety)

- プールオブジェクトはスレッドセーフ*ではない*。
    - いくつかプールを生成して、
    - スレッド1つにプール1つを割り当てる。
    - そうすれば、スレッドはロックせずにプールから自由に割り当てを行うことができる。
    - アプリケーションは2つのスレッドが同時に1つのプールを使わないように保証する必要がある。
- プールを解放すれば、そのプールから割り当てたすべてのオブジェクトが解放される。
    - 個々に解放する必要はない。

## コマンドプール(Command Pools)

- コマンドバッファの割り当てに使われる。
- コマンドバッファが成長(grow)すれば、メモリが割り当てられるかもしれない。
    - このメモリはプールに由来する。
    - アプリケーションは2つのスレッドが同じプールからのコマンドバッファを同時に組み立てることがないように保証する。
        - シングルスレッドで同じプールからの複数のコマンドバッファを組み立てることはできる。
- 高いパフォーマンスを得るには、*シングルスレッドな*アロケータを使う。
- スレッドごとにコマンドプールを使う。

## デスクリプタプール(Descriptor Pools)

- デスクリプタセットの割り当てに使われる。
- デスクリプタは一般的には均質な配列(homogenous array)である。
    - 特殊用途(specialized)のアロケータで管理するのは理にかなう。
    - デスクリプタプールはこれらのアロケータである。
    - もう一度言うが、スレッドセーフではない。
        - デスクリプタセットを管理するための専用スレッドを使ったり、
        - デスクリプタプールを各スレッドで生成したり、
        - など。

## クエリプール(Query Pools)

- クエリオブジェクトは一般的にメモリ内では小さい。
    - たいていは単一の64ビット整数か同等なもの。
- これらを個別のオブジェクトとして管理するのは辛い。
    - 効率が悪い(Not performant)。
    - メモリが断片化する。
    - クエリをバッチ処理しづらい。
- 故に、クエリオブジェクトはプールから割り当てられる。
    - 連続的な割り当てを可能にする。
    - バッチ処理結果を楽にする。

# HOST MEMORY

## ホストメモリの管理(Managing Host Memory)

- Vulkanはアプリケーションがシステムメモリを管理することを許可する。
    - コールバック経由でシステムメモリを割り当てる。
    - ほとんどのオブジェクト生成関数は構造体`VkAllocationCallbacks`を受け取る。

~~~c
typedef struct  VkAllocationCallbacks {
    void* pUserData;
    PFN_vkAllocationFunction pfnAllocation;
    PFN_vkReallocationFunction pfnReallocation;
    PFN_vkFreeFunction pfnFree;
    PFN_vkInternalAllocationNotification pfnInternalAllocation;
    PFN_vkInternalFreeNotification pfnInternalFree;
} VkAllocationCallbacks;
~~~

## メモリマネージャーの階層(Memory Manager Hierarchy)

- インスタンスとデバイスはそれぞれ自身のアロケータを持つ。
    - インスタンスを生成するとき、ドライバはインスタンスアロケータを使う。
    - デバイスを生成するとき、ドライバは、
        - デバイスレベルアロケータを使うか、
        - インスタンスレベルアロケータを使う。
    - オブジェクトを生成するとき、ドライバは、
        - オブジェクトレベルアロケータを使うか、
        - デバイスレベルアロケータを使うか、
        - インスタンスレベルアロケータを使う。

## ホストメモリアロケータ(Host Memory Allocator)

### 基本の割り当て(Basic Allocations)

~~~c
typedef void* (VKAPI_PTR *PFN_vkAllocationFunction)(
    void* pUserData,
    size_t size,
    size_t alignment,
    VkSystemAllocationScope allocationScope
);
~~~

- 割り当てのコールバック
    - 割り当て、再割り当て、解放。
    - `pUserData`: ほしいものなら何でも。
    - `allocationScope`: 割り当てのスコープまたは寿命。
        - コマンド、オブジェクト、キャッシュ、デバイス、インスタンス。

### 割り当ての扱い(Handling Allocations)

- ドライバはあなたのスレッドからアロケータを通して呼び出す。
    - アロケータはスレッドセーフである必要はない。
    - しかし、ドライバは階層を登ってくる(walk up)かもしれない。
    - オブジェクトアロケータを渡していても、デバイスやインスタンスのアロケータが呼ばれることもある。
    - デバイスとインスタンスのアロケータはスレッドセーフにすべき。

## 内部メモリ割り当て(Internal Memory Allocations)

- 時々、ドライバは内部割り当てを行う。
    - いくつかの理由によりホストのアロケータを使うことは出来ない。
        - 実行可能なコードにためにメモリを割り当てる必要があるのか？
        - 特別なアライメント、キャッシングやプラットフォーム固有の制約。
- ドライバは通知関数を呼び出す。
    - `pfnInternalAllocation`、`pfnInternalFree`。
    - 情報の提供であり、オプションである。
        - デバッグビルドでこれらをフックする。

## ホストメモリの解放(Freeing Host Memory)

- オブジェクト破棄関数もアロケータを受け取る。
    - `vkDestroy*`に渡すアロケータは`vkCreate*`に渡したものに対応していなければならない。
- もう一度言うが、ドライバはあなたのアロケータを使わないかもしれない。
    - オブジェクトは内部的にプールに割り当てられるのか？
    - オブジェクトは内部的にまた使用中なのか？
    - など。

# DEVICE MEMORY

## デバイスメモリ(Device Memory)

- リソースはGPUメモリを必要とする。
- リソースとメモリはオブジェクトが分かれている。
- `vkAllocateMemory`と`vkFreeMemory`でメモリの割り当てと解放を行う。
- 以下でメモリをリソースにバインドする。
    - イメージは`vkBindImageMemory`。
    - バッファは`vkBindBufferMemory`。
- 他のオブジェクトはGPUメモリを必要とするかもしれない。
    - ドライバが管理する。

## デバイスメモリの特性(Device Memory Properties)

- メモリはヒープから割り当てられる。
- 各ヒープの中には、複数のメモリの"タイプ"が存在してもよい。
- ヒープ特性にはサイズとデバイスローカルかどうかが含まれる。
- タイプ特性にはキャッシングとコヒーレンシーのオプションが含まれる。
- メモリ特性は`vkGetPhysicalDeviceMemoryProperties`で特定する。

## オブジェクトの必須条件の問い合わせ(Querying Object Requirements)

- イメージの必須条件を特定するには`vkGetImageMemoryRequirements`を使う。
- `VkMemoryRequirements`が返る。
- `VkMemoryRequirements::memoryTypeBits`を報告されたタイプとマッチングすることでメモリタイプを選ぶ。

## デバイスメモリを割り当てる(Allocate Device Memory)

- デバイスメモリを割り当てるには、`vkAllocateMemory`を呼び出す。
- `VkMemoryAllocateInfo`を渡す。
- `VkMemoryAllocateInfo::memoryTypeIndex`に生息するヒープから割り当てる。

## メモリをオブジェクトにバインドする(Bind Memory To Objects)

- メモリとオブジェクトを手に入れたら、それらを`vkBind[Image|Buffer]Memory`でバインドする。
- `memoryOffset`から始まるところにメモリをバインドする。
    - オブジェクトはメモリオブジェクトから必要な分だけ消費する。
    - オブジェクトはメモリをオーバーラップできる。--- ハザードを管理する必要がある。

## デバイスメモリの解放(Freeing Device Memory)

- デバイスメモリを解放するには、`vkFreeMemory`を呼び出す。
- ハザードを解決するのはあなたの責任である。
    - デバイスが使用している可能性のあるメモリは解放しない。
    - これはメモリの解放する前にストールすることを意味するかもしれない。
    - 割り当てのリサイクルを検討する。

# BE SMART

## オブジェクトプールを使う(Use Object Pools)

- オブジェクトプールは大量の似たオブジェクトを集約する。
    - クエリやデスクリプタのような均質なオブジェクトに対して使われる。
    - コマンドバッファのような高頻度な割り当てを必要とするものに対して使われる。
- プールはスレッドセーフ*ではない*。
    - スレッドごとにプールを使う。
    - 独自のミューテクスを実装する。
- プールを解放すると、中のすべてのオブジェクトが解放される。
    - 解放する前に処理が完了していることを確かめる。

## システムメモリを管理する(Manage System Memory)

- システムメモリアロケータの使用はオプション。
    - ドライバより良い仕事ができると考えるならやればいい。
    - ドライバはあなたが思ったようにアロケータを使ってくれるとは限らない。
- もしシステムメモリアロケータを書くとしたら、
    - アライメントを適切にサポートしていることを確かめる。
    - `malloc`に接続するだけにしない。
        - "バカなドライバ"でもそれより賢いことをやっている。
    - 割り当ての中身を詮索(poke around)しない。
        - そこに秘伝のタレは存在しない。

## GPUメモリを管理する(Manage GPU Memory)

- メモリアロケータを書く必要がある。
- リソースをメモリオブジェクトと1対1に対応させない。
    - 各メモリオブジェクトはドライバやOSによって追跡されている。
    - 各サブミッションはアロケータレベルでメモリアクセスを取り扱う。
    - これは主要なパフォーマンス問題である。
- 少数の大きなメモリオブジェクトを作り、そこから副割り当てを行う。
    - リソースがメモリを共有できる。--- あなたがハザードを管理する。
    - リソースを一緒にパックする。

# References