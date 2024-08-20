# **Learn OS in 3days**

## **メモリ管理**

### **動的メモリ割り当て**

メインメモリには１バイトごとに１つの物理アドレスが割り当てられる。またプロセッサがアクセスできる物理アドレスの範囲を物理アドレス空間と呼ぶ。コンピュータでは、必要に応じてメモリ領域を動的に確保したり解放したりすることが一般的に行われる。このような処理は動的メモリ割り当てと呼ばれる。

### **動的メモリ割り当ての実装**

物理メモリを動的に割り当てられるようにしたい。ただし、`malloc`関数のようにバイト単位で割り当てるのではなく、ページと呼ばれるまとまった単位で割り当てる。ページサイズは4KB（4096B）が一般的である。具体的には、ページ数を引数として渡されると、そのページ分の割り当て可能なメモリ領域の先頭アドレスを返す関数を実装すること。このとき、渡すメモリ領域はゼロクリアしてから返すことが一般的である。`memset`関数が用意されているので、それを使用する。動的に割り当て可能なメモリ領域はリンカスクリプトで定義されている`__free_ram`から`__free_ram_end`までの領域とする。最後に実際に動的にメモリを割り当て、その物理アドレスを表示し、正しく動作することを確かめること。

今回は使い終わったメモリページを解放する処理は実装しない。自作OSを長時間実行することはないと思うので、メモリリークが発生しても問題ないためである。今回実装する割り当てアルゴリズムのことをLinearアロケータ（Bumpアロケータ）と呼び、解放処理が必要ない場面で実際に使われている。

### **仮想メモリ**

仮想メモリは、プロセスが使用するアドレスと実際の物理アドレスを分離することで、メモリを仮想化する技術である。仮想記憶では、プロセスごとに専用の独立したアドレス空間を割り当てる。プロセスから見えるアドレスは物理アドレスではなく、仮想アドレスである。プロセスがメモリにアクセスするときは、仮想アドレスを物理アドレスに変換した上で物理メモリにアクセスする。

仮想メモリを用いることで、他のプロセスに割り当てられたメモリに影響されずに各プロセスに対して連続したアドレス空間を提供でき、また各プロセスのアドレス空間を分離することで互いに隔離し、システムの安定性を高めることができる。

### **ページング方式**

仮想アドレスと物理アドレスとの対応関係（マッピング）をバイト単位で指定できるようにすると、アドレス変換のための情報量が膨大になってしまうため、ある程度の大きさのメモリブロックを単位としてマッピングを行う。現在は固定長のメモリブロックを単位とするページング方式が主流である。

ページング方式では、仮想アドレス空間と物理アドレス空間を固定長（ページサイズ）のブロックに分割する。仮想アドレス空間のブロックを仮想ページ（ページ）、物理アドレス空間のブロックを物理ページ（ページフレーム）と呼ぶ。ページにはページ番号がついており、仮想アドレスや物理アドレスは先頭部分にそのアドレスが属するページ番号を持ち、ページ内オフセットがそれに続く構成になっている。

仮想アドレスから物理アドレスへの変換は、ページを物理ページにマッピングすることで行われる。プロセスに割り当てるメモリは物理アドレス空間上では連続している必要はないため、物理メモリの外部断片化が発生しない。

カーネルは空いている物理ページを連結リストやビットマップなどで管理する。カーネルは新しくプロセスを実行する場合などにプロセスに割り当てるページが必要になると、空いている物理ページを割り当てる。また、プロセスが終了した場合など、ページが不要になれば対応する物理ページを回収する。

### **ページテーブル**

仮想アドレスから物理アドレスへの変換にはページテーブルと呼ばれるデータ構造を使用する。ページテーブルは仮想ページ番号でインデックスする配列であり、物理メモリ上に配置される。ページテーブルの要素をページテーブルエントリと呼び、１つのページテーブルエントリが１つの仮想ページに対応し、マッピングされている物理ページの番号や各種のフラグを格納している。

仮想アドレスを物理アドレスに変換する作業は、MMU（memory management unit）と呼ばれるハードウェアがページテーブルを参照することで動的に行われる（動的アドレス変換）。現代の一般的なプロセッサはMMUを内蔵している。

MMUには、使用するページテーブルの物理アドレスを保持するレジスタ（ページテーブルベースレジスタ）があり、異なるページテーブルを指すように変更することで、プロセッサが使用する仮想アドレス空間を切り替えることができる。

OSはプロセスごとにページテーブルを用意し、コンテキストスイッチによって実行するプロセスを切り替えるとき、ページテーブルベースレジスタの値を、次に実行するプロセスのページテーブルの先頭アドレスに設定し直す。

具体的には下記の流れでアドレス変換が行われる。

1. 仮想アドレスから仮想ページ番号とページ内オフセットを求める
2. ページテーブルベースレジスタからページテーブルを参照し、仮想ページ番号をもとに、ページテーブルエントリを取得する
3. ページテーブルエントリから物理ページ番号を取得する
4. 物理ページ番号とページ内オフセット（１で取得したもの）を組み合わせて物理アドレスを求める（物理ページ番号 * ページサイズ + ページ内オフセット）

### **多段ページテーブル**

仮想アドレス空間が広くなると、その分、必要なページテーブルのサイズも大きくなる。例えば仮想アドレス空間が256TB、ページサイズが4KB、ページテーブルエントリが8Bの場合、ページテーブルとして、256 / 4 * 8 = 512GB必要になる。しかし、一般的にプロセスは仮想アドレス空間のごく一部しか使用しないため、ほとんどのページテーブルエントリは無駄となる。

そこで考え出されたのが、ページテーブルを階層化した多段ページテーブルである。多段ページテーブルでは、最下位のページテーブルは１段しかない場合のページテーブルと同様のページテーブルエントリを保持するが、最下位以外のページテーブルでは、各エントリは１つ下のページテーブルの物理アドレスを保持する。

多段ページテーブルでは、実際に使用する仮想アドレスの範囲に対応するページテーブルのみを用意すれば良いため、物理メモリを節約することができる。

### TLB

ページテーブルを用いたアドレス変換では、メモリアクセスのたびにページテーブルの段数分のメモリ参照が追加で必要となるため、単純な実装ではただでさえ遅いメモリへのアクセスが非常に遅くなってしまう。このため、MMUは仮想アドレスから物理アドレスへのアドレス変換を高速化するためのTLB（トランスレーションルックアサイドバッファ）と呼ばれるハードウェアを備えている。

TLBはページテーブルエントリ専用のキャッシュメモリであり、連想メモリ（CAM: content addressable memory）と呼ばれる、高速に任意のエントリを検索できる特殊なメモリで構成されている。MMUは、仮想アドレスに対応するページテーブルエントリがTLB内に存在（TLBヒット）すれば、TLB内のページテーブルエントリを使用する。このときはページテーブルを参照する必要がないため、アドレス変換を高速に実行できる。一方、存在しなければ（TLBミス）、ページテーブルを参照してページテーブルエントリを取得し、取得したエントリをTLBに格納する。

一般的なプロセッサではTLBのエントリ数は数千個程度であり、TLBは仮想アドレスをキーとしてキャッシュされたページテーブルエントリを検索する。プロセスが異なれば同じ仮想アドレスでもページテーブルエントリは異なるため、プロセス間でコンテキストスイッチする場合、OSはTLBをクリアしなければならない。これをTLBフラッシュと呼び、そのための機械語命令がある。

### **仮想アドレス空間**

仮想アドレス空間の構成はOSによって異なるが、低位領域をユーザープロセス用とし、高位領域をカーネル用とするのが一般的である。前者をユーザー空間、後者をカーネル空間と呼ぶ。

## **プロセス管理**

### **プロセスとは？**

OSでは実行中のプログラムのことをプロセスと呼ぶ。OSはプロセス単位で保護、リソース割り当て、権限管理を行う。

ユーザーは信頼度の低いプログラムを動かす可能性があり、そのような場合でもシステムを安定動作させるために、OSはそれぞれのプロセスが他のプロセスやOSそのものに影響を与えないように隔離し、保護する。これにより、バグがあるプログラムを動かしてもたいていの場合はそのプロセスだけがクラッシュ（異常終了）するだけですむ。

また、プロセスが動作するには機械語を実行するためのプロセッサと、プログラムやデータを配置するためのメモリが必要であり、OSは各プロセスに対して適切にこれらのリソースを割り当てる。

また、OSにはさまざまなオブジェクト（ファイル、デバイス、ソケット、共有メモリなど）が存在するが、OSはそれぞれのオブジェクトに対して操作を行う権限をプロセスごとに管理し、プロセスが許可されていない操作を行おうとすると、OSはプロセスの実行を停止させたり、ユーザーに許可を求めたりする。

### **PCB**

OSはプロセスを管理するためにそれぞれのプロセスに対してカーネル内にPCB（プロセス制御ブロック）と呼ばれるデータ構造を割り当てる。プロセッサごとに現在実行中のプロセスのPCBへのポインタを持つ。PCBには一般的に下記の内容が含まれる。

- PID（プロセスID）
- プロセスを実行しているユーザに関する情報
- プロセスが使用しているメモリ領域に関する情報
- プロセスが使用しているリソース（オープンしているファイルなど）に関する情報
- プロセスが保持する権限に関する情報
- アカウンティング情報（使用したCPU時間やメモリ使用量、入出力の回数などに関する統計情報）

### **PCBの実装**

PID、プロセスの状態、コンテキストスイッチ時のスタックポインタ、カーネルスタックを持つPCBを定義する。プロセッサがカーネル内のコードを実行しているときは、ユーザープロセスのスタック（ユーザースタック）とは別のスタック（カーネルスタック）を使用する。カーネルスタックはカーネル領域に存在する。カーネルスタックにはコンテキストスイッチ時のCPUレジスタ、どこから呼ばれたのか（関数の戻り先）、各関数でのローカル変数などが入っており、カーネルスタックをプロセスごとに用意することで、別の実行コンテキストを持ち、コンテキストスイッチで状態の保存と復元ができるようになる。

ちなみにプロセスの状態は`os/kernel.h`で、`PROC_UNUSED`（未使用）と`PROC_RUNNABLE`（実行可能）が定義されている。

### **コンテキストスイッチの実装**

プロセスを作成する関数を実装し、また複数のプロセス間でコンテキストスイッチをできるようにもしたい。プロセスの最大数は`os/kernel.h`で`PROCS_MAX`が定義されており、この数以上のプロセスは作成できないようにすること。実装したらプロセスを複数作成し、コンテキストスイッチを行い、正しく動作することを確認すること。

コンテキストスイッチの際に、グローバル変数、スレッドは特にプロセス内で使用しないため、`gp`と`tp`は保存する必要はない。またRISC-Vの呼び出し規約（calling convention）により、`t0`~`t6`と`a0`~`a7`は関数呼び出し間で値が保持されることが保証されないレジスタであるため、これらのレジスタも保存する必要はない。`s0`~`s11`は関数内で使用するレジスタであるが、関数呼び出し間で値が保持されることが保証されるため、これらのレジスタの値を保存する必要がある。また関数の実行を正しく再開するために、`ra`も保存する必要がある。

### **実行ファイルの構造**

実行ファイルはヘッダと複数のセクションから構成される。ヘッダはマジックナンバ（ファイル形式を識別するための値）から始まり、実行ファイルのメタデータ（各セクションのサイズ、プログラムの実行開始アドレス、サポートするOSのバージョン、プロセッサの情報など）を格納している。セクションには下記がある。

- テキスト：プログラムの機械語命令の部分を格納する
- データ：プログラムが使用するデータのうち、初期値が与えられているものを格納する
- BSS：初期値が与えられていないデータのための領域。実行ファイルにはBSSセクションのサイズ情報のみが格納される
- 共有ライブラリ情報：動的リンクする共有（動的）ライブラリに関する情報を格納する
- リロケーションテーブル：プログラムを任意のアドレスに配置するための情報を格納する
- シンボルテーブル：関数やグローバル変数などのシンボルとアドレスの対応表を格納する。動的リンクでは、共有ライブラリからプログラムのアドレスを参照するために用いる。デバッガはこの情報を使用することでシンボル名を使用してデバッグできるようにする
- デバッグ情報：デバッグに必要な情報（ソースコードの行番号とアドレスの対応表など）が格納される。デバッガはこの情報も参照する。プログラムの開発段階でのみ使用される
- その他：フォーマットによっては実行ファイルのアイコンの画像などを格納できるものもある

### **Cソースコードが実行ファイルになるまで**

1. プリプロセス：マクロ展開、ファイルのインクルード、条件付きコンパイルが処理される
2. コンパイル：コンパイラはプリプロセスされたソースコードを受け取り、アセンブリ言語に変換する
   1. 字句解析：トークンに分割
   2. 構文解析：構文木に変換
   3. 中間表現生成：ClangのようなLLVMを基盤とするコンパイラはLLVM IRを生成する
   4. 最適化：LLVM IRはLLVMのバックエンドで最適化される。ループ最適化、デッドコード除去、インライン展開などが行われる
   5. アセンブリコード生成：最適化された中間表現をターゲットアーキテクチャに応じたアセンブリコードに変換する
3. アセンブル：アセンブラはアセンブルコードをマシンコード（オブジェクトファイル）に変換する
4. リンク：リンカは全てのオブジェクトファイルと静的ライブラリをリンクし、未解決の外部シンボルを解決して１つの実行可能ファイルを生成する

### **プログラムのロード**

プログラムを実行するには、ストレージ上に実行ファイルとして格納されている機械語プログラムを、プロセッサのアドレス空間上にロードする必要がある。またスタックやヒープ領域などの実行中に必要なメモリ領域を確保し、初期化する必要もある。これらを行うプログラムはローダと呼ばれ、カーネルに組み込まれている。ロードにあたって、OSはプロセスにアドレス空間上の適当なメモリ領域を割り当てる。一般的なOSではプロセスのメモリ空間は下記の領域から構成される。

- テキスト領域：プログラムを構成する機械語命令を格納する。実行ファイルのテキストセクションに対応する
- データ領域：初期値が与えられているデータを格納する。実行ファイルのデータセクションに対応する
- BSS領域：初期値が与えられていないデータのための領域。実行ファイルのBSSセクションに対応する
- ヒープ領域：C言語のmalloc関数やC++言語のnew演算子のようにプログラムの実行中に動的に割り当てるメモリのための領域。通常アドレスの大きくなる方向に拡大する
- スタック領域：機械語プログラム実行中に使用されるスタックのための領域。通常アドレスの小さくなる方向に拡大する

実行中に必要なデータを一時的に保持するために、スタック領域を使用する。一般的なプロセッサはスタックトップ（最後にpushしたデータ）のアドレスを持つスタックポインタ（SP）というレジスタを持っている。スタックはレジスタに格納できない関数の引数やリターンアドレス、ローカル変数などを保持する。

下記の流れでプログラムはロードされる。

1. 実行ファイルのヘッダ部から各セグメントの大きさを読み取る
2. 必要なメモリ領域を確保する
3. 実行ファイルからテキストセクションとデータセクションの内容をメモリ上に読み込む
4. BSS領域を初期化する（一般的には0クリアする）
5. スタック領域を初期化する（一般的には0クリアする）
6. プログラムが動的リンクされる場合、動的リンクの準備を行う（メモリ上に共有ライブラリを配置するなど）

### **システムコール**

ユーザープログラムから自由にプロセッサの設定を変更したり、ハードウェアを直接制御したりできてしまうと、システムの安定性やセキュリティ上重大な問題となるため、カーネルは実行ファイルからロードした機械語プログラムを実行している間、プロセッサをユーザーモードに設定して、ユーザープログラムが問題がある動作を行おうとすると特権違反例外が発生し、カーネルはそのプロセスを強制終了するなどの処置をとる。

しかし、ユーザープログラムもファイルの読み書きやメモリの割り当てなど、リソースを操作したい場面が存在する。そのため、それらの操作をC言語などから呼び出し可能なAPIとして提供する。このAPIをシステムコールと呼ぶ。

ユーザープログラムからシステムコールを呼び出すと、プロセッサのモードをユーザーモードからカーネルモードに切り替え、システムコールの処理を実行した後、カーネルモードからユーザーモードに切り替えてからユーザープログラムに戻る。

システムコールの実行は下記のように行われる。

1. アプリケーションプログラムがシステムコールのラッパー関数を呼び出す
2. ラッパー関数はシステムコール番号とシステムコールへの引数をカーネルに渡すために、レジスタやスタックなどに書き込む
3. ラッパー関数はシステムコールを実行するための特別な機械語命令を実行する。この命令はソフトウェア割り込みを引き起こす
   1. プロセッサをカーネルモードに遷移させる
   2. スタックをカーネルスタックに切り替える
   3. ユーザープロセスの現在のコンテキストをレジスタやカーネルスタックなどに退避させる
   4. カーネル内にあるシステムコールを処理するためのルーチンに制御を移す
4. ２で渡されたシステムコール番号を読み取り、そのシステムコールを処理する関数を呼び出す
5. 処理が終了したらシステムコールの返り値をレジスタなどに書き込み、ユーザーモードに戻るための機械語命令を実行する
6. プロセッサはユーザーモードに遷移し、ユーザープロセスのコンテキストを復元する
7. ラッパー関数はシステムコールの返り値を返す

### **マルチタスク**

現代のOSは複数のプログラムを同時に実行できるマルチタスク（マルチプロセス）をサポートしている。マルチタスクには協調的マルチタスクとプリエンティブマルチタスクの２つがある。

協調的マルチタスクではそれぞれのプログラムは自発的に他のプログラムにプロセッサを譲る（これをyieldという）操作を行うまで実行を継続する。協調的マルチタスクは単純なので、プリエンプティブマルチタスクに比べて実装が容易であるが、各プログラムは長時間プロセッサを占有しないように定期的にyieldする必要がある。

一方、プリエンプティブマルチタスクでは、プログラムが明示的にyieldしなくても、実行するプログラムがタイムスライス（プリエンプションされずに連続して実行できる時間）を使い切るとカーネルが強制的に切り替える。この強制的な切り替えのことをプリエンプション（横取り）と呼ぶ。プリエンプションされたプログラムはいずれ中断した処理の続きを実行する機会が与えられる。

実行するプロセスやスレッドを切り替えることをコンテキストスイッチと呼ぶ。どのプロセスにプロセッサを割り当てて実行するのかを決める処理のことをスケジューリングといい、スケジューラと呼ばれるプログラムがこれを行う。
