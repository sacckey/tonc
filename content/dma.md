# 14. ダイレクトメモリアクセス

<!-- toc -->

## DMA って… ケ？ {#sec-intro}

<dfn>ダイレクトメモリアクセス</dfn>（Direct Memory Access、DMA）は、データをある場所から別の場所へ高速にコピーする方法です。というより、データを高速に*転送*する方法です。データのコピーにも使えますが、メモリを埋めることにも使えるからです。DMA を起動すると、いわゆる <dfn>DMA コントローラ</dfn>がハードウェアを引き継ぎ（実際には CPU は停止します）、望む転送を行って、CPU が自分がいなかったことに気づく前に制御を返します。

DMA チャンネルは 4 つあります。チャンネル 0 は優先度が最も高く、時間に厳しい処理に使われ、内部 RAM にしか使えません。チャンネル 1 と 2 は、再生のために音のデータを適切な音声バッファへ転送するのに使われます。優先度が最も低いチャンネル 3 は、汎用のコピー用です。このチャンネルの主な用途の 1 つは、新しいビットマップやタイルのデータを読み込むことです。

## DMA のレジスタ {#sec-regs}

どんな種類の転送処理にも、3 つのものが必要です。転送元、転送先、そしてコピーするデータの量です。*どこから*、*どこへ*、*どれだけ*、です。DMA では、転送元のアドレスを `REG_DMAxSAD` に、転送先のアドレスを `REG_DMAxDAD` に入れます。3 つ目のレジスタ `REG_DMAxCNT` は、転送する量を示すだけでなく、いつ転送を始めるか、チャンクの大きさ、データのチャンクを 1 つ転送するたびに転送元と転送先のアドレスをどう更新するかなど、DMA で使えるほかの機能も制御します。DMA のレジスタはどれも 32 ビットの長さですが、必要なら 2 つの 16 ビットのレジスタに分けることもできます。チャンネル 0 のレジスタは`0400:00B0h` から始まり、続くチャンネルは 12 ずつずれた位置から始まります（{@tbl:dma-regs} を見てください）。

<div class="lblock">
<table id="tbl:dma-regs">
<caption align="bottom">
  <b>{*@tbl:dma-regs}</b>: DMA のレジスタのアドレス
</caption>
<tr><th>レジスタ<th>機能<th>アドレス
<tr><td><code>REG_DMAxSAD</code><td>転送元
	<td><code>0400:00B0h  + 0Ch</code>·x
<tr><td><code>REG_DMAxDAD</code><td>転送先
	<td><code>0400:00B4h + 0Ch</code>·x
<tr><td><code>REG_DMAxCNT</code><td>制御
	<td><code>0400:00B8h + 0Ch</code>·x
</table>
</div>

### DMA の制御 {#ssec-reg-cnt}

転送元と転送先のレジスタの使い方は明らかでしょう。制御レジスタには少し説明が必要です。`REG_DMAxCNT` レジスタそのものは 32 ビットですが、多くの場合、2 つの別々のレジスタに分けられます。1 つは個数用、もう 1 つは実際の制御ビット用です。

<div class="reg">
<table class="table-reg reg-huge" id="tbl:reg-dmaxcnt"
  border=1 frame=void cellPadding=4 cellSpacing=0>
<caption class="reg">
  REG_DMAxCNT @ 0400:00B8+12<i>x</i>
</caption>
<tr class="bits">
	<td>1F<td>1E<td>1D 1C<td>1B<td>1A<td>19<td>18 17
	<td>16 15<td>14 13 12 11 10
	<td>F E D C B A 9 8 7 6 5 4 3 2 1 0
<tr class="bf">
  <td class="rclr1">En
  <td class="rclr6">I
  <td class="rclr5">TM
  <td>-
  <td class="rclr2">CS
  <td class="rclr7">R
  <td class="rclr3">SA
  <td class="rclr4">DA
  <td>-
  <td class="rclr0">N
</table>
<br>

<table class="table-reg-vert">
  <col class="bits" width=56>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">
  <td>00-0F<td class="rclr0">N
  <td> 
  <td>転送の<b>回数</b>。
<tr class="bg1">
  <td>15-16<td class="rclr4">DA
  <td>DMA_DST_INC, DMA_DST_DEC, DMA_DST_FIXED, DMA_DST_RELOAD
  <td><b>転送先の調整</b>。
    <ul>
      <li><b>00</b>: 転送ごとに増やす（デフォルト）
      <li><b>01</b>: 転送ごとに減らす
      <li><b>10</b>: なし。アドレスは固定
      <li><b>11</b>: 転送中は転送先を増やし、終わったら
	    元に戻す。これで、繰り返しの DMA が常に
		同じ転送先から始まる。
    </ul>
<tr class="bg0">
  <td>17-18<td class="rclr3">SA
  <td>DMA_SRC_INC, DMA_SRC_DEC, DMA_SRC_FIXED,
  <td><b>転送元の調整</b>。転送先の 2 ビットとまったく同じように
    働きます。<code>DMA_SRC_RESET</code> はないことに注意してください。
    転送元にコード 3 は使えません。
<tr class="bg1">
  <td> 19  <td class="rclr7">R
  <td>DMA_REPEAT
  <td>DMA のタイミングが VBlank か HBlank のモードに設定されていれば、
    そのたびにコピーを繰り返します。
<tr class="bg0">
  <td> 1A  <td class="rclr2">CS
  <td>DMA_16, DMA_32
  <td><b>チャンクの大きさ</b>。DMA がハーフワード単位（クリア時）
    とワード単位（セット時）のどちらでコピーするかを設定します。
<tr class="bg1">
  <td>1C-1D<td class="rclr5">TM
  <td>DMA_NOW, DMA_AT_VBLANK, DMA_AT_HBLANK, DMA_AT_REFRESH
  <td><b>タイミングのモード</b>。転送をいつ始めるかを指定します。
      <ul>
        <li><b>00</b>: すぐに始める。
        <li><b>01</b>: VBlank で始める。
        <li><b>10</b>: HBlank で始める。
        <li><b>11</b>: 今のところ使ったことはありませんが、私が理解している
          動き方は次のとおりです。DMA1 と DMA2 では、FIFO が空になったときに
          補充します。個数と大きさは、それぞれ 1 と 32 ビットに
          固定されます。DMA3 では、各描画ラインの始まりでコピーを
          始めますが、2 スキャンライン遅れます。
      </ul>
<tr class="bg0">
  <td> 1E  <td class="rclr6">I
  <td>DMA_IRQ
  <td><b>割り込み要求</b>。終わったら割り込みを発生させます。
<tr class="bg1">
  <td> 1F  <td class="rclr1">En
  <td>DMA_ENABLE
  <td>このチャンネルの DMA 転送を<b>有効にする</b>。
</tbody>
</table>
</div>

### 転送元と転送先のアドレス {#ssec-reg-adr}

転送元と転送先のアドレスのレジスタは、期待どおりに働きます。正しいアドレスを入れるだけです。そうそう、転送元と転送先のアドレスの大きさは、32 ビットすべてではなく、それぞれ 28 ビットと 27 ビットの幅だということは言っておくべきでしょう。でも、これは気にする必要はありません。どのみち `1000:0000h` より上のアドレスにはアクセスできませんから。転送先のアドレスには、`0800:0000h` より上の領域は使えません。でも、ROM にコピーできたら、それはそれで変ですよね？

### DMA のフラグ {#ssec-reg-flags}

`REG_DMAxCNT` レジスタは 2 つの部分に分けられます。実際のフラグの部分と、コピーする回数の部分です。どちらの使い方でも動きますが、フラグの定義の仕方には注意が必要です。16 ビットのレジスタに 32 ビットの #define を使ったり、その逆をしたりするのは良い考えではありません。

1 つのチャンクを転送し終えたときに、次の転送元と転送先のアドレスをどうするかを制御するオプションがあります。デフォルトでは両方とも増えていくので、コピーとして働きます。しかし、転送元を一定にしておけば、どちらかというとメモリを埋める処理として働くようにもできます。

`REG_DMAxCNT` の下半分に入れるのは、転送の回数です。これはバイト数ではなく、*チャンク*の数です！ ここで `sizeof()` などを使うときは気をつけたほうがよいでしょう。2 倍や 4 倍の係数を書き忘れるのはとても簡単です。チャンクは、ビット 26 によって 16 ビットか 32 ビットになります。

### DMA のタイミングについてもう少し {#ssec-reg-timing}

すぐに始まる DMA が何をするかは、簡単に想像できます。DMA を有効にしたとたんに動きます。まあ、*実際には*動き出すまでに 2 サイクルかかりますが、十分近いです。ほかのタイミングの設定も、考え方としてはそれほど難しくありませんが、混乱しやすい点が 1 つあります。

次のような状況を考えてみてください。普通の背景に何かかっこいいことをしたいとします。具体的には、背景のレジスタをスキャンラインごとに更新する必要のあることをしたいとします。HBlank のたびにデータをコピーできる（`DMA_AT_HBLANK` のタイミングのフラグで）と言ったばかりなので、これはこの仕事にうってつけに思えます。しかし、少し考えてみると、次のような疑問が浮かぶかもしれません。

> タイミングを、たとえば `DMA_AT_HBLANK` に設定したとき、次の HBlank で *N* 回のコピーを*すべて*行うのでしょうか、それとも、リストが終わるまで HBlank ごとに 1 回ずつコピーするのでしょうか？

この 2 つには決定的な違いがあります。1 つ目の場合は、すべてのコピーが一度に行われてしまうので意味がなさそうです。転送先が（背景のレジスタのように）固定されていれば、最後のもの以外のコピーはすべて無駄になります。2 つ目の場合は、HBlank ごとに 2 回以上コピーするにはどうすればいいのでしょうか？ 明らかに、何かがおかしいです。実は、2 つの点でおかしいのです。

念のため言っておくと、これから言うことに 100% の確信があるわけではありませんが、実際に起きていることにかなり近いと思います。理解しておくべき主な点は、チャンネルが有効になっていない（`REG_DMAxCNT`\{1f\} がクリアされている）限り、そのチャンネルは何もしないということです。`REG_DMAxCNT`\{1f\} が設定されて初めて、DMA の処理が開始されます。適切なタイミング（タイミングのビットで決まります）で、DMA は *N* 回のコピーをすべて行い、それから自分で自分を停止させます。

ただし、繰り返しのビット（`REG_DMAxCNT`\{19\}）が設定されている場合は別です。その場合は、自分でチャンネルを無効にするまで、適切なタイミングでコピーを続けます。

## DMA の処理をいくつか {#sec-func}

3 つのレジスタを手で設定するのはそれほど面倒ではありませんが、直接のやり取りはサブルーチンの中に隠したほうがよいでしょう。さて、古いコードでは、次のようなものに出くわすかもしれません。

```c
// Don't do this. Please.
void dma_copy(int ch, void* src, void* dest, uint count, u32 mode)
{
    switch(ch)
    {
    case 0:
        // set DMA 0
    case 1:
        // set DMA 1
... // etc
    }
}
```

これでも動きますが、良いやり方ではありません。switch の各ケースが 1 つの数値しか違わないなら、たいていは単純な表引きで置き換えられます。直す方法はいくつかありますが、一番簡単なのは、タイルのメモリでしたのと同じように、DMA のレジスタに構造体の配列を重ねることです。そうすれば、チャンネルの変数でチャンネルを選び、アドレスとフラグを埋めるだけで済みます。

```c
typedef struct DMA_REC
{
    const void *src;
    void *dst;
    u32 cnt;
} DMA_REC;

#define REG_DMA ((volatile DMA_REC*)0x040000B0)
```

以下は、私の DMA の処理のうち 3 つです。まず `DMA_TRANSER()` マクロ。これは何にでも使える全体的なマクロです。それから、DMA 3 で 32 ビットの転送を使って、汎用のメモリのコピーと埋めを行う 2 つの処理です。

```c
// in tonc_core.h

//! General DMA transfer macro
#define DMA_TRANSFER(_dst, _src, count, ch, mode)   \
do {                                            \
    REG_DMA[ch].cnt= 0;                         \
    REG_DMA[ch].src= (const void*)(_src);       \
    REG_DMA[ch].dst= (void*)(_dst);             \
    REG_DMA[ch].cnt= (count) | (mode);          \
} while(0)

//! General DMA copier
INLINE void dma_cpy(void *dst, const void *src, uint count, int ch, u32 mode)
{
    REG_DMA[3].cnt = 0; // shut off any previous transfer
    REG_DMA[3].src = src;
    REG_DMA[3].dst = dst;
    REG_DMA[3].cnt = count;
}

//! General DMA full routine
INLINE void dma_fill(void *dst, volatile u32 src, uint count, int ch, u32 mode)
{
    REG_DMA[3].cnt = 0; // shut off any previous transfer
    REG_DMA[3].src = (const void*)&src;
    REG_DMA[3].dst = dst;
    REG_DMA[3].cnt = count | DMA_SRC_FIXED;
}

//! Word copy using DMA 3
INLINE void dma3_cpy(void *dst, const void *src, u32 size)
{   dma_cpy(dst, src, size/4, 3, DMA_CPY32);  }

//! Word fill using DMA 3
INLINE void dma3_fill(void *dst, const void *src, u32 size)
{   dma_fill(dst, src, size/4, 3, DMA_CPY32);  }
```

どの場合も、まず、それまで動いていた転送をすべて無効にしています。DMA が CPU を止めるなら冗長に思えるかもしれませんが、DMA の転送はタイミングを指定することもできることを思い出してください。レジスタを設定している途中で転送が始まってほしくはないはずです。そのあとは、レジスタを埋めるだけです。さて、実は転送が本当に始まるまでに 2 サイクルの遅延があります。つまり、立て続けに転送を要求すると、転送が失われることがあるということです。ただ、これがよく起きるかどうかはわかりません。メモリのウェイトステート自体がすでにそれくらいの時間をとるので、安全な*はず*です。

これらの処理についてのほかの注意点です。`DMA_TRANSFER()` マクロのコードは `do {} while(0);` ループの中に置いてあります。マクロの問題は、展開されたときに複数の文が入れ子のブロックを壊すことがある点です。たとえば、中かっこで囲まずに `if` の本体の中で呼ぶと、最初の行だけが本体になり、残りはその外に出てしまいます。この種のループは、それを防ぐ方法の 1 つです。マクロのもう 1 つの問題は、引数の名前がマクロのコードのほかの部分を隠してしまうことがある点です。`DMA_REC` 構造体の `src` メンバと `dst` メンバのようにです。引数にアンダースコアが付いているのはこのためです。埋める処理にも注目すべき点があり、それについては[次の小節](#ssec-func-fill)で読めます。最後に、`dma3` のインライン関数はワード単位の転送を使い、最後の引数としてバイト単位の大きさを取るので、標準の `memcpy()` や `memset()` にとてもよく似ています。
<br>  
以前は、転送に次のマクロを使っていました。これは、プリプロセッサのやや風変わりな機能の 1 つである連結演算子「##」を使っています。これを使うと、コンパイル時にシンボル名を作れます。恐ろしく、まったく安全ではなく、一般に手に負えないものですが、ちゃんと動きます。先ほどのマクロのほうが良いのですが、私はこれもまだ気に入っています。

```c
#define DMA_TRANSFER(_dst, _src, _count, _ch, _mode)  \
    REG_DMA##_ch##SAD = (u32)(_src),                  \
    REG_DMA##_ch##DAD = (u32)(_dst),                  \
    REG_DMA##_ch##CNT = (_count) | (_mode)            \
```

`_ch` にリテラルの数値を使っている限り、正しいレジスタ名ができます。そして、そう、文の間のカンマ演算子は実際にちゃんと働きます。文を分けておくだけでなく、`do{} while(0)` の構文と同じように、誤った入れ子を防ぐ役目も果たします。

### DMA による埋めについて {#ssec-func-fill}

DMA はメモリを埋めるのに使えますが、試す前に知っておくべき問題が 2 つあります。1 つ目は、注意していれば防げます。DMA による埋めは、`memset()` と*まったく*同じようには働きません。`REG_DMAxSAD` に入れるのは、埋めたい値ではなく、その*アドレス*なのです！

「わかった、値を変数に入れて、そのアドレスを使えばいいんだね。」 そう、そしてそれが 2 つ目の問題につながります。見つけるのがほぼ不可能なバグです。これを試してみると、うまくいかないことがわかります。まあ、*何か*では埋められるのですが、たいていは埋めたかったものではありません。完全な説明は少し技術的ですが、基本的には、おそらく変数のアドレスしか使っておらず*値*を使っていないので、最適化によって変数が一度も初期化されないのです。簡単な解決策があります。前にも見たものですが、volatile にすることです。あるいは、`dma_fill()` のような（インライン）関数を使うこともできます。この関数では転送元の引数が volatile になっているので、期待どおりに数値をそのまま入れられます。そこから volatile のキーワードを取り除くと、また失敗することに注意してください。
<br>  
要するに、DMA による埋めには、値を直接ではなくアドレスを渡す必要があるということです。グローバル変数なら常に動きますが、ローカル変数や引数を使う場合は volatile にする必要があります。同じことが BIOS コールの CpuFastSet() にも当てはまることに注意してください。

### DMA、使いすぎにご用心 {#ssec-func-use}

DMA は速いです。それは疑いようがありません。配列のコピーより[最大 10 倍速く](text.html#ssec-demo-se2)なることもあります。しかし、すべてのコピーに使うかどうかはよく考えてください。速いとはいえ、ほかのすべての転送処理を圧倒するほどではありません。CpuFastSet() はコピーでは 10% 以内の差に迫り、埋めでは実際に 10% 速いです。速度の向上はそれほど大きなものではありません。もう 1 つの問題は、DMA が CPU を止めることです。これは[割り込み](interrupts.html)をめちゃくちゃにして、一見でたらめなバグを引き起こすことがあります。DMA にはたいていタイマーや割り込みと組み合わせた特定の使い道がありますが、汎用のコピーなら、ほかのものも検討するとよいでしょう。CpuFastSet() は良い処理ですが、libtonc にはそれより安全で制約も少ない `memcpy16()/32()` と `memset16()/32()` の処理もあります。ただしこれらはアセンブリの処理なので、アセンブルの仕方かライブラリの使い方を知っている必要があります。

## DMA のデモ: 円形のウィンドウ {#sec-demo}

<div class="cpt_fr" style="width:240px;">
<img alt="dma_demo short" src="./img/demo/dma_demo.png" id="fig:dma-demo">

**{*@fig:dma-demo}**: `dma_demo` の画面。（訳注: 原文は "palette for" ですが、図の内容に合わせました）
</div>

この章のデモは少し複雑に見えるかもしれませんが、その効果にはそれだけの価値があります。DMA 転送の基本的な使い方はとても簡単なので、デモにするまでもありません。*トリガー付き*の DMA を使うのは別の話です。ここでは、HBlank をトリガーにした DMA、略して HDMA を見ていきます。これを使って、HBlank の中で[ウィンドウ](gfx.html#sec-win)の大きさを更新し、円形のウィンドウの効果を出します。

もちろん、言うは易く行うは難しです。設計の最初の一歩は、そもそもこのために HDMA をどう使うかです。HBlank のたびに `REG_WIN0H` へコピーする必要があるので、転送先は固定しておく必要があります。厳密には、元の転送先に*戻す*必要がありますが、コピーするのがハーフワード 1 つだけなら、同じことです。転送元については、そこにコピーする必要のあるデータを、スキャンラインごとに 1 つのエントリを持つ配列で管理し、1 スキャンラインずつ配列を進めていきます（つまり、転送元を増やしていきます）。そしてもちろん、転送は*各*スキャンラインで行う必要があるので、繰り返しに設定します。つまり、基本的には次のものが必要です。

```c
#define DMA_HDMA    (DMA_ENABLE | DMA_REPEAT | DMA_AT_HBLANK | DMA_DST_RELOAD)
```

円については、円の左右の端を計算できる処理が必要です。円を描けるアルゴリズムはいくつかあります。たとえば [Bresenham](http://www.gamedev.net/reference/articles/article767.asp) のものです。そこにピクセルを描く代わりに左右の点を保存するだけでよいので、それに手を加えた版を使います。なぜ上下ではなく左右なのでしょうか？ 配列はスキャンライン単位なので、*y* の値はすでに配列が表しているからです。

実際には、端を求められさえすれば、何を使ってもかまいません。求められたら、あとは VBlank で DMA を準備するだけで完成です。
<br>  
最終的な結果は {@fig:dma-demo} のようになります。ウィンドウの内側には（またしても）ブリンスタの背景があり、外側には縞模様の背景があります。文字はウィンドウの位置と半径を示しています。ウィンドウは十字キーで動かせ、A と B で大きさを変えられます。

<pre><code class="language-c hljs">
#include <stdio.h>
#include <tonc.h>

#include "brin.h"

// From tonc_math.h
//#define IN_RANGE(x, min, max) ( (x) >= (min) && (x) < (max) )


// The source array
<span class="bold">u16 g_winh[SCREEN_HEIGHT+1];</span>

//! Create an array of horizontal offsets for a circular window.
/*! The offsets are to be copied to REG_WINxH each HBlank, either 
*     by HDMA or HBlank isr. Offsets provided by modified 
*     Bresenham's circle routine (of course); the clipping code is not
*     optional.
*   \param winh Pointer to array to receive the offsets.
*   \param x0   X-coord of circle origin.
*   \param y0   Y-coord of circle origin.
*   \param rr   Circle radius.
*/
void win_circle(u16 winh[], int x0, int y0, int rr)
{
    int x=0, y= rr, d= 1-rr;
    u32 tmp;

    // clear the whole array first.
    memset16(winh, 0, SCREEN_HEIGHT+1);

    while(y >= x)
    {
        // Side octs
        tmp  = clamp(x0+y, 0, SCREEN_WIDTH);
        tmp += clamp(x0-y, 0, SCREEN_WIDTH)<<8;
        
        if(IN_RANGE(y0-x, 0, SCREEN_HEIGHT))       // o4, o7
            winh[y0-x]= tmp;
        if(IN_RANGE(y0+x, 0, SCREEN_HEIGHT))       // o0, o3
            winh[y0+x]= tmp;

        // Change in y: top/bottom octs
        if(d >= 0)      
        {
            tmp  = clamp(x0+x, 0, SCREEN_WIDTH);
            tmp += clamp(x0-x, 0, SCREEN_WIDTH)<<8;
            
            if(IN_RANGE(y0-y, 0, SCREEN_HEIGHT))   // o5, o6
                winh[y0-y]= tmp;
            if(IN_RANGE(y0+y, 0, SCREEN_HEIGHT))   // o1, o2
                winh[y0+y]= tmp;

            d -= 2*(--y);
        }
        d += 2*(x++)+3;
    }
    winh[SCREEN_HEIGHT]= winh[0];
}

void init_main()
{
    // Init BG 2 (basic bg)
    <span class="bold">dma3_cpy(pal_bg_mem, brinPal, brinPalLen);</span>
    dma3_cpy(tile_mem[0], brinTiles, brinTilesLen);
    dma3_cpy(se_mem[30], brinMap, brinMapLen);

    REG_BG2CNT= BG_CBB(0)|BG_SBB(30);

    // Init BG 1 (mask)
    const TILE tile= 
    {{
        0xF2F3F2F3, 0x3F2F3F2F, 0xF3F2F3F2, 0x2F3F2F3F, 
        0xF2F3F2F3, 0x3F2F3F2F, 0xF3F2F3F2, 0x2F3F2F3F
    }};
    tile_mem[0][32]= tile;
    pal_bg_bank[4][ 2]= RGB15(12,12,12);
    pal_bg_bank[4][ 3]= RGB15( 8, 8, 8);
    pal_bg_bank[4][15]= RGB15( 0, 0, 0);
    se_fill(se_mem[29], 0x4020);

    REG_BG1CNT= BG_CBB(0)|BG_SBB(29);

    tte_init_chr4_b4_default(0, BG_CBB(2)|BG_SBB(28));
    tte_init_con();
    tte_set_margins(8, 8, 232, 40);

    // Init window
    REG_WIN0H= SCREEN_WIDTH;
    REG_WIN0V= SCREEN_HEIGHT;

    // Enable stuff
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0 | DCNT_BG1 | DCNT_BG2 | DCNT_WIN0;
    REG_WININ= WIN_BUILD(WIN_BG0|WIN_BG2, 0);
    REG_WINOUT= WIN_BUILD(WIN_BG0|WIN_BG1, 0);
}

int main()
{
    int rr=40, x0=128, y0=120;

    init_main();

    while(1)
    {
        vid_vsync();
        key_poll();

        rr += key_tri_shoulder();   // size with B/A
        x0 += key_tri_horz();       // move left/right
        y0 += key_tri_vert();       // move up/down

        if(rr<0)
            rr= 0;

        // Fill circle array
        <span class="bold">win_circle(g_winh, x0, y0, rr);</span>
    
        // Init win-circle HDMA
        <span class="bold">DMA_TRANSFER(&REG_WIN0H, &g_winh[1], 1, 3, DMA_HDMA);</span>

        tte_printf("#{es;P}(%d,%d) | %d", x0, y0, rr);
    }
    
    return 0;
} 
</code></pre>

初期化関数は、ほとんどがただの飾りです。「ほとんど」と言ったのは、1 つ注目すべき点があるからです。ブリンスタのパレット、タイル、マップをコピーするための `dma_cpy` の呼び出しです。それ以外は、見るべきものはありません。

main 関数そのものも、かなり標準的なものです。ここで注目すべきなのは、転送元の配列を準備する `win_circle()` の呼び出しと、HDMA を初期化する `DMA_TRANSFER()` の呼び出しです。実際には `g_winh[0]` ではなく `g_winh[1]` から始めていることに注意してください。その理由は、HBlank はあるスキャンラインの前ではなく*後*に起きるので、そうしないと 1 ラインずれてしまうからです。`g_winh` 配列は実際には 160+1 の長さで、エントリ 0 と 160 はどちらもスキャンライン 0 のデータを表します。ここでははっきり見えませんが、もう 1 つ重要なのは、HDMA は*表示される* HBlank でのみ起き、VBlank 中の HBlank では起きないということです。おかげで、準備するときに何スキャンラインぶん補正すればいいかを決める手間が大いに省けます。

そして `win_circle()` です。Bresenham の円のアルゴリズムの仕組みを知っていれば、1 つの八分円のオフセットを計算し、対称性の規則を使ってそれを残りの 7 つに使うことはご存じでしょう。ここでも同じことをしています。おそらく元のアルゴリズムにないのは、すべてのクリッピング（`clamp()` や `IN_RANGE()s`）です。しかし、ここではこれらの手順が絶対に欠かせません。横方向に範囲外に出ると、ウィンドウのオフセットが間違ってしまい、ウィンドウが裏返ってしまいます。縦方向に範囲外に出ると、`g_winh` の範囲外にアクセスすることになり、あらゆる恐ろしいことが起きます。信じてください。これらは必要です。

また、最初に配列全体をきれいに消していることにも注目してください。これはループの中でもできますが、先に全体を埋めてから必要な部分だけを更新するほうが速いこともあります。最後に、前に述べたように、HBlank の起き方に合わせて、最初のスキャンラインのデータを配列の最後のエントリにコピーしています。
<br>  
これで DMA の章は終わりです。HDMA をこのように使うのは、円形のウィンドウだけでなく、あらゆる効果にとても役立ちます。必要なのは、スキャンラインのデータを入れた配列と、それを前もって準備する関数だけです。ただし、チャンネルを取り違えないように気をつけてください。

DMA は最も速いコピーの方法ですが、割り込みを妨げてしまうので、`memcpy32()` を使うほうがおそらく安全です。どのみち速度の差は 10% しかありません。DMA は、タイマーと組み合わせて音の FIFO にも使われます。音に使う方法はここではお見せできませんが、タイマーの仕組みは説明できます。それは次の章で説明します。
