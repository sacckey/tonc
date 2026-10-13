# 17. BIOS コール

<!-- toc -->

<br>  

## はじめに {#sec-intro}

HBlank 割り込みやカートリッジ割り込みのような[ハードウェア割り込み](interrupts.html)のほかに、<dfn>ソフトウェア割り込み</dfn>と呼ばれるものもあります。<dfn>BIOS コール</dfn>とも呼ばれます。ソフトウェア割り込みは、普通の関数ととてもよく似た働き方をします。入力を準備してルーチンを呼び、出力を受け取ります。違いはコードへのたどり着き方にあります。普通の関数では、まあ、目的のルーチンにジャンプするだけです。ソフトウェア割り込みでは `swi` 命令を使います。この命令はプログラムの流れを BIOS のどこかにそらし、要求されたアルゴリズムを実行して、それからプログラムの通常の流れに戻します。これはハードウェア割り込みがすることと似ていますが、今度はプログラムから割り込みを発生させるのです。だからソフトウェア割り込みというわけです。

GBA の BIOS には 42 個のソフトウェア割り込みがあり、コピー、数学（除算、平方根）、スプライトや背景のアフィン変換、展開などの基本的なルーチンがそろっています。`IntrWait` のルーチンのような、とても特別な関数もあります。これは、ハードウェア割り込みが起きるまで CPU を止めておけるものです。その VBlank 版はとてもおすすめで、この章が重要なのはそのためです。

ソフトウェア割り込みを使うのは、1 つのことさえなければそれほど難しくありません。`swi` 命令そのものです。これにはまたアセンブリが必要です。ただし、*たくさん*のアセンブリではありませんし、それらの C のラッパーを書くのも簡単です。それもここで扱います。

## BIOS の関数 {#sec-funs}

BIOS の関数を呼ぶには、「<code>swi <i>n</i></code>」命令を使います。*n* は使いたい BIOS コールです。ただし、使う必要のある正確な数値は、コードが ARM ステートか Thumb ステートかによって変わることに注意してください。Thumb では引数は単に *n* そのものですが、ARM では *n*<<16 を使う必要があります。普通の関数と同じく、BIOS コールにも入力と出力があります。この目的には最初の 4 つのレジスタ（r0-r3）が使われますが、正確な用途とレジスタの数はコールごとに違います。

以下は、各 BIOS コールの名前の一覧です。それぞれが何をするかは説明しません。ほかのサイトがすでにやっていますし、その内容をそのまま写しても意味がないと思うからです。完全な説明は、たとえば [GBATEK](https://problemkaputt.de/gbatek.htm) を見てください。どう働くのか感じをつかめるように、いくつかについては説明します。

### 完全な一覧 {#ssec-funs-list}

<center>
<table class="center" width="70%">
<col span=5 align="left">
<tr><th>id		<th width="30%">名前  <th width="10%">	<th>id		<th>名前
<tr><td>0x00	<td>SoftReset		<td><td>0x08	<td>Sqrt
<tr><td>0x01	<td>RegisterRamReset<td><td>0x09	<td>ArcTan 
<tr><td>0x02	<td>Halt			<td><td>0x0A	<td>ArcTan2
<tr><td>0x03	<td>Stop			<td><td>0x0B	<td>CPUSet 
<tr><td>0x04	<td>IntrWait		<td><td>0x0C	<td>CPUFastSet 
<tr><td>0x05	<td>VBlankIntrWait	<td><td>0x0D	<td>BiosChecksum 
<tr><td>0x06	<td>Div				<td><td>0x0E	<td>BgAffineSet 
<tr><td>0x07	<td>DivArm			<td><td>0x0F	<td>ObjAffineSet 

<tr><td>&nbsp;
<tr><td>0x10	<td>BitUnPack		<td><td>0x18	<td>Diff16bitUnFilter 
<tr><td>0x11	<td>LZ77UnCompWRAM	<td><td>0x19	<td>SoundBiasChange 
<tr><td>0x12	<td>LZ77UnCompVRAM	<td><td>0x1A	<td>SoundDriverInit 
<tr><td>0x13	<td>HuffUnComp		<td><td>0x1B	<td>SoundDriverMode 
<tr><td>0x14	<td>RLUnCompWRAM	<td><td>0x1C	<td>SoundDriverMain 
<tr><td>0x15	<td>RLUnCompVRAM	<td><td>0x1D	<td>SoundDriverVSync 
<tr><td>0x16	<td>Diff8bitUnFilterWRAM	<td><td>0x1E <td>SoundChannelClear 
<tr><td>0x17	<td>Diff8bitUnFilterVRAM	<td><td>0x1F <td>MIDIKey2Freq

<tr><td>&nbsp;
<tr><td>0x20	<td>MusicPlayerOpen		<td><td>0x28 <td>SoundDriverVSyncOff
<tr><td>0x21	<td>MusicPlayerStart	<td><td>0x29 <td>SoundDriverVSyncOn 
<tr><td>0x22	<td>MusicPlayerStop		<td><td>0x2A <td>GetJumpList
<tr><td>0x23	<td>MusicPlayerContinue
<tr><td>0x24	<td>MusicPlayerFadeOut
<tr><td>0x25	<td>MultiBoot
<tr><td>0x26	<td>HardReset
<tr><td>0x27	<td>CustomHalt
</table>
</center>

### Div、Sqrt、Arctan2、ObjAffineSet の説明 {#ssec-funs-smpl}

<dl>
<dt>0x06: Div</dt>
<dd>
入力:
  <p>r0: 分子</p>
  <p>r1: 分母</p>
 出力:
  <p>r0: 分子 / 分母</p>
  <p>r1: 分子 % 分母</p>
  <p>r3: abs(分子 / 分母)</p>
 注意: ゼロで割ってはいけません！
<br>
</dd>

<dt>0x08: Sqrt</dt>
<dd>
入力:
  <p>r0: num。符号なし 32 ビット整数</p>
出力:
  <p>r1: sqrt(num)</p>
<br>
</dd>

<dt>0x0a: ArcTan2</dt>
<dd>
入力:
  <p>r0: <i>x</i>。<b>符号付き 16 ビット</b>の数（<code>s16</code>）</p>
  <p>r1: <i>y</i>。<b>符号付き 16 ビット</b>の数（<code>s16</code>）</p>
出力:
  <p>r0: 
    <i>x</i>&ge;0 : <i>&theta;</i>= arctan(<i>y/x</i>) &or;
    <i>x</i>&lt;0 : <i>&theta;</i>= 
      sign(<i>y</i>)*(&pi; &minus; arctan(|<i>y/x</i>|).<br>
  </p>
    これは <i>y</i> = <i>x</i>*tan(&theta;) の完全な逆を求めます。
    タンジェントの問題は、定義域が半円であることで、
    アークタンジェントの値域も半円です。円全体の範囲を得るには、
    <i>x</i> と <i>y</i> の値の商だけでなく、それぞれの符号も
    必要です。<i>&theta;</i> の数学的な範囲は
    [&minus;&pi;,&nbsp;&pi;&rang; で、これは
    [&minus;0x8000,&nbsp;0x8000&rang;（あるいは、お好みなら
    [0,&nbsp;2&pi;&rang; と [0,&nbsp;0xFFFF]）に対応します
<br>
</dd>

<dt>0x0f: ObjAffineSet</dt>
<dd>
入力:
  <p>r0: 転送元のアドレス</p>
  <p>r1: 転送先のアドレス</p>
  <p>r2: 計算の回数</p>
  <p>r3: <b>P</b> 行列の要素の間隔（背景なら 2、
    オブジェクトなら 8）</p>
</dd>
</dl>

転送元のアドレスは、`AFF_SRC` 構造体の配列を指します（`ObjAffineSource` とも呼ばれますが、背景にも使えるので、これは少し誤解を招く名前です）。`AFF_SRC` 構造体は、2 つのスケール *s*<sub>x</sub>、*s*<sub>y</sub> と、角度 α でできています。角度はここでも \[0, 0xFFFF\] の範囲で 2π を表します。結果の **P** は次のとおりです。

<math id="eq:pswi" xmlns="http://www.w3.org/1998/Math/MathML" display="inline">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtext mathvariant="bold">({!@eq:pswi})</mtext>
            <mtext>&#xA0;</mtext>
            <mtext>&#xA0;</mtext>
            <mtext mathvariant="bold">P</mtext>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <msub>
                      <mi>s</mi>
                      <mi>x</mi>
                    </msub>
                    <mo>&#x22C5;</mo>
                    <mi>cos</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mi>&#x3B1;</mi>
                  </mtd>
                  <mtd>
                    <mo>&#x2212;</mo>
                    <msub>
                      <mi>s</mi>
                      <mi>x</mi>
                    </msub>
                    <mo>&#x22C5;</mo>
                    <mi>sin</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mi>&#x3B1;</mi>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <msub>
                      <mi>s</mi>
                      <mi>y</mi>
                    </msub>
                    <mo>&#x22C5;</mo>
                    <mi>sin</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mi>&#x3B1;</mi>
                  </mtd>
                  <mtd>
                    <msub>
                      <mi>s</mi>
                      <mi>y</mi>
                    </msub>
                    <mo>&#x22C5;</mo>
                    <mi>cos</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mi>&#x3B1;</mi>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>

もうこれが何をするかはわかるはずです。横方向に 1/*s*<sub>x</sub>、縦方向に 1/*s*<sub>y</sub> で拡大縮小し、そのあとで *α* だけ反時計回りに回転します。`ObjAffineSet()` は `obj_aff_rotscale()` や `bg_aff_rotscale()` とほぼまったく同じことをしますが、`ObjAffineSet()` は複数の行列を一度に設定することもできます。

転送元のデータは `ObjAffineSource`（つまり `AFF_SRC`）構造体に入れておきます。さて、このルーチンはアフィン行列を設定するので、転送先は `OBJ_AFFINE` 構造体か `ObjAffineDest` 構造体だと思うかもしれません。しかし、それは間違いです。まあ、部分的には。問題は、転送先は常に *p*<sub>a</sub> の要素を指すことで、それは構造体の最初の要素とは限りません。これを使って行列を埋めようとするとき、単に `OBJ_AFFINE` のポインタを渡すという間違いを、あなたは*きっと*犯します。警告しなかったとは言わせませんよ。

ここで言っておくべきことがほかに 2 つあります。まず、またしても少し名前が不適切です。ObjAffineSet は、実はオブジェクトそのものとはあまり関係がありませんが、`r3` を 2 ではなく 8 にすればオブジェクト用として使えます。2 つ目は、このルーチンは `r2` によって複数の配列を設定するのにも使えることです。ただし、devkitPro 19 でこれをするときは*気をつけてください*。`ObjAffineSet()` は転送元の構造体が[ワード境界に揃っている](bitmaps.html#ssec-data-align)ことを期待していますが、自分でアラインメントの属性を付けない限り、そうはなりません。

<pre><code class="language-c hljs">// Source struct. Note the alignment!
typedef struct AFF_SRC
{
    s16 sx, sy;
    u16 alpha;
} <span class="bold">ALIGN4</span> AFF_SRC, ObjAffineSource;

// Dst struct for background matrices
typedef struct Aff_DST
{
    s16 pa, pb;
    s16 pc, pd;
} ALIGN4 AFF_DST, ObjAffineDest;

// Dst struct for objects. Note that r1 should be
// the address of pa, not the start of the struct
typedef struct OBJ_AFFINE
{
    u16 fill0[3];    <span class="bold">s16 pa</span>;
    u16 fill1[3];    s16 pb;
    u16 fill2[3];    s16 pc;
    u16 fill3[3];    s16 pd;
} ALIGN4 OBJ_AFFINE;
</code></pre>

## BIOS コールを使う {#sec-usage}

### BIOS コールのためのアセンブリ {#ssec-use-swi-asm}

アセンブリを使わない限りレジスタや `swi` 命令にはアクセスできず、それはかなり大変なのだから、ここまでの話はあまり意味がなかったと思うかもしれません。そうですよね？ ええと、いいえ、はい、そしていいえ、です。BIOS コールに必要なアセンブリの手順は実はかなり単純で、以下のとおりです。

```armasm
@ In tonc_bios.s

@ at top of your file
    .text           @ aka .section .text
    .code 16        @ aka .thumb

@ for each swi (like division, for example)
    .align 2        @ aka .balign 4
    .global Div
    .thumb_func
Div:
    swi     0x06
    bx      lr
```

これは GNU アセンブラ（GAS）用のアセンブリのコードです。Goldroad や ARM STD では、構文が少し違うと思います。最初にする必要があるのは、続くコードについての詳細を伝える、いくつかの<dfn>ディレクティブ</dfn>を書くことです。ここでは、「`.text`」を使ってコードを `text` 領域（ROM、マルチブートなら EWRAM）に置きます。また、「`.code 16`」か「`.thumb`」を使って、コードが Thumb コードであることも伝えます。これらをファイルの先頭に置けば、ファイルの残りすべてに効きます。各 BIOS コールには、次の 6 つの項目が必要です。

-   **ワード境界への整列**。少なくともハーフワード境界ですが、おそらくワードのほうが望ましいでしょう。これには 2 つのディレクティブ、<code>.align <i>n</i></code> と <code>.balign <i>m</i></code> があります。前者は 2<sup>*n*</sup> に揃えるので「`.align 2`」が必要です。後者は *m* に揃えるので、単に「<code>balign <i>m</i></code>」を使えます。どちらも*次の*コードやデータにしか効かず、それより先には効かないことに注意してください。そのため、関数ごとに付けるのが一番です。
-   **スコープ**。<code>.global <i>name</i></code> ディレクティブは *name* からシンボルを作り、プロジェクトのほかのファイルからも見えるようにします。`extern` のようなもの、というより、`static` の逆のようなものです。
-   **Thumb の印**。`.code 16` だけでは十分ではないようで、`.thumb_func` も必要です。実際、マニュアルを正しく読めているなら、これは `.code 16` も意味するので、そのディレクティブは冗長になります。
-   **ラベル**。「*name*:」は、シンボル *name* がどこから始まるかを示します。当然ですが、関数を使うには、それが実際に存在しなければなりません。
-   **BIOS コール**。実際に BIOS コールを起動するには、「swi *n*」を使います。*n* は使いたい BIOS コールです。
-   **戻り**。これでほぼ完成です。あとは「bx lr」で呼び出し元に戻るだけです。

ね？ 本当にそれほど複雑ではありません。これより少し多くの機能が欲しいこともあるかもしれませんが、たいていは、たった 2 つの命令で済みます。

### ARM アーキテクチャのプロシージャ呼び出し標準 {#ssec-use-aapcs}

それで結構なのですが、まだ疑問が残ります。a) これを C のコードとどう組み合わせればいいのか、そして b) 入力と出力はどこへ行ってしまったのか、です。1 つ目の答えは簡単です。いつもどおりに関数の宣言を追加するだけです。

```c
// In tonc_bios.h

int Div(int num, int denom);
```

ふむ、でもそれでは入力と出力がどこへ行ったのか、*やはり*説明になっていません。ところが実は……*説明になっている*のです。

> 「雲がどうやってできるのか、ぼくにはよくわからない。でも雲はやり方を知っている。それが大事なことなんだ」

ずっと前に、子どもたちの科学の珍回答集のようなもので見つけた言葉ですが、プログラミングをしているといつも思い出します。コンピュータについて言えるのは、コンピュータは入力、出力、テキスト、絵などという観点では考えないということです。実際には何も考えていないのですが、それはまた別の話です。コンピュータに見えているのはデータだけです。コードとデータですらなく、ただのデータです。コードもデータだからです。もちろん、C や VB などに慣れている*あなた*はそうは見ていないかもしれませんが、結局のところ、すべてはただの 1 と 0 にすぎません。1 と 0 がプログラムカウンタ（**PC** レジスタ、`r15`）を通って CPU に来ればそれはコードで、そうでなければデータです。

では、それで入力と出力がどう説明できるのでしょうか？ まあ、直接説明するわけではありませんが、状況をどう見るべきかを示してくれます。あなたがコンパイラで、どこかの誰かの C のコードを、CPU が実際に使える機械語（またはアセンブリ。ほぼ同じものです）に変換しなければならないとします。「`q= Div(x,y);`」という行に出くわしました。`Div()` は何をするのでしょうか？ C のファイルにその名前のシンボルがなければ（そして実際にありません。*tonc_bios.s* にあるからです）、わかりません。厳密には、それが*何なのか*すらわかりません。でも `Div` は知っています。それが大事なことなのです。少なくとも、*ほぼ*そういう仕組みになっています。それでもコンパイラは、混乱を避けるために `Div` がどんな種類の生き物なのかを知る必要があります。変数？ マクロ？ 関数？ そのために宣言があるのです。上の宣言は、`Div` が 2 つの符号付き整数を受け取り、1 つの符号付き整数を返す関数だと言っています。コンパイラに関する限り、話はそこで終わりです。
<br>  
もちろん、それでもコンパイラが何をすればいいかをどうやって知るのかの説明にはなっていません。実は、コンパイラは単に *ARM Architecture Procedure Call Standard*、略して <dfn>AAPCS</dfn> に従っているのです。これは、関数どうしが引数をどう受け渡すべきかを定めたものです。この PDF 文書は[こちら](https://github.com/ARM-software/abi-aa/releases/download/2023Q3/aapcs32.pdf)にあり、アセンブリを考えているなら、ダウンロードする価値は大いにあります。

今のところは、次のことを知っておけば十分です。最初の 4 つの引数は最初の 4 つのレジスタ `r0-r3` に置かれ、それ以降の引数はスタックに置かれます。出力の値は `r0` に置かれます。BIOS コールの引数のリストを宣言のリストとして使う限り、うまく動くはずです。宣言は、必要なキャストもすべて面倒みてくれることに注意してください。ここで宣言が何を意味するのかを理解しておくことが大事です。関数がどう呼ばれるかを決めるのは*宣言*であって、実際のアセンブリの関数の*定義*ではありません。C の関数の定義でさえありません。宣言を間違えると、とてもまずいことになりかねません。

AAPCS が教えてくれることがもう 1 つあります。レジスタ r0-r3（と r12）は、いわゆる<dfn>スクラッチ</dfn>レジスタだということです。つまり、呼び出し側は、呼ばれた関数がこれらを書き換えることを*想定*しています。関数から戻ったあと、それらの中身は未定義とみなされます。ただし、両方のアセンブリの関数を自分で書いているなら、ある種の……融通が利くこともあります。これらがスクラッチレジスタであるおかげで、関数は元の値をスタックに積んだり取り出したりせずにこれらを使え、時間を節約できます。ただし、ほかのレジスタではそうはいきません。r4-r11、r13、r14 は、呼び出し側の関数から受け取ったときの状態で返さ*なければなりません*。最後の r15 は例外です。プログラムカウンタをいじるべきではないからです。

### インラインアセンブリ {#ssec-use-inl-asm}

実は、BIOS コールのためにアセンブリのファイルを丸ごと用意する必要すらありません。<dfn>インラインアセンブリ</dfn>を使えるのです。インラインアセンブリを使えば、C のコードとアセンブリのコードを混ぜられます。関数はたいていかなり単純なので、次のようなものが使えます。

```c
// In a C file
int Div(int num, int denom)
{   asm("swi 0x06");   }
```

これは、アセンブリ版の `Div` とまったく同じことをします。ただし、インラインアセンブリには注意が必要です。周りのコードが見えないので、いじるべきでないレジスタをうっかり<dfn>破壊</dfn>（clobber）してしまい、コードの残りを台無しにするかもしれないからです。インラインアセンブリの完全な規則は、[GCC のマニュアル](http://www.gnu.org/manual/manual.html)を見てください。[devrs.com](http://www.devrs.com/gba/) にも、インラインアセンブリの使い方についての短い FAQ があります。とはいえ、インラインアセンブリの「正しい」構文は世界一親切なものとは言えませんし、ほかにも問題があります。上の C の関数を考えてみてください。それ自体は実際には何もしないので、最適化によって捨てられてしまうかもしれません。適切な対策をとらない限り、`-O3` ではそうなります。また、コンパイラは、関数が値を返すべきなのに何も返していないと文句を言います。もちろん、その部分はアセンブリのブロックの中で面倒をみているのですから、コンパイラの言い分にも一理あります。今のところ私が気づいていない問題もおそらくほかにあるでしょう。結局のところ、何が起きているのかがわかる、アセンブリだけで書いた版を使うほうが簡単です。

### <kbd>swi_call</kbd> マクロ {#ssec-use-swi-call}

一方で、引数を使わない BIOS コールもあり、それらは単なるマクロで実行できます。`swi_call(x)` マクロは BIOS コール *x* を実行します。*swi.h* と、Wintermute の [libgba](https://devkitpro.org) にあり、私はそこから持ってきました。上の `Div` 関数より少し洗練されています。まず、`volatile` キーワードを使っています。これで、最適化によって関数が削除されるのを防げるはずです（すべてのレジスタに対してしたのと同じです）。次に、<dfn>クロバーリスト</dfn>（3 つのコロンの後ろ）を使っています。これはコンパイラに、インラインアセンブリがどのレジスタを使うかを伝えます。3 つ目に、Thumb と ARM の切り替えを自動的に面倒みます。`-mthumb` コンパイラオプションを使うと、コンパイラが `__thumb__` を定義してくれるので、それを使って正しい swi の番号を得るのです。賢いでしょう？

```c
#ifndef(__thumb__)
#define swi_call(x)   asm volatile("swi\t"#x ::: "r0", "r1", "r2", "r3")
#else
#define swi_call(x)   asm volatile("swi\t"#x"<<16" ::: "r0", "r1", "r2", "r3")
#endif
```

  

ところで、アセンブリについてもっと知りたければ、gbadev.org に ARM アセンブリのチュートリアルがいくつかあります。もう 1 つの良い学び方は、`-S` コンパイラフラグを使うことです。これを使うと、コンパイラが生成した、あなたのコードのアセンブリのファイルが得られます。最適化の手順や AAPCS の使い方も含めて、コンパイラがあなたのコードに何をしているのかが正確にわかります。本当に、少なくとも一度は見ておくべきです。

`-fverbose-asm` を使うのも役に立つかもしれません。これは、元の変数名や演算をコメントに書き出してくれます。たいていは正しい場所に書かれます。下に示す `ASM_CMT()` マクロも便利です。これで、特定のコードのブロックがどこにあるのか、手がかりが得られます。ただし、これもいつもうまくいくとは限りません。

```c
#define ASM_CMT(str) asm volatile("@ " str)

//In code. Outputs "@ Hi, I'm here!" in the generated asm
ASM_CMT("Hi, I'm here!");
```

## デモのグラフ {#sec-demo}

<div class="cpt_fr" style="width:240px;">
<img alt="math graphs" src="./img/demo/swi_demo.png" id="fig:swi-demo">

**{*@fig:swi-demo}**: BIOS 提供の、div、sqrt、arctan2、sin、cos のグラフ。
</div>

BIOS コールの使い方を示すために、Div、Sqrt、ArcTan、ObjAffineSet を使って、双曲線、平方根、サイン、コサインのグラフを作ります。240x160 の画面にうまく収まるように拡大縮小してあります。定義は次のとおりです。

<table>
<tr><td>除算	<td><i>y</i>= 2560/<i>x</i>
<tr><td>平方根	<td><i>y</i>= 160*sqrt(<i>x</i>/240)
<tr><td>arctan  	<td><i>y</i>= 80 + 64*(2/&pi;)*(arctan(<i>x</i>-120)/16))
<tr><td>サイン		<td><i>y</i>=  1*sy*sin(2&pi;·<i>x</i>/240)	<td>; sy= 80
<tr><td>コサイン		<td><i>y</i>= 80*sx*cos(2&pi;·<i>x</i>/240)	<td>; sx= 1
</table>

これらの関数を図 1 にプロットしました。サインとコサインのためのコールはないのに、どうやってその値を得たのか不思議に思うなら、もう一度式 1 を見てください。**P** 行列にそれらが入っています。コサインには *p*<sub>a</sub> を、サインには *p*<sub>c</sub> を使いました。グラフは一瞬で表示されることに注意してください。グラフを描いている間に読み込み時間のようなものは感じません。モード 7 のデモの以前の版（や PERN のモード 7 のデモ）では、ルックアップテーブルを作るのに、実際の除算、サイン、コサインの関数を呼んでいました。三角関数の対称性の規則を使っても、`sin()` と `cos()` は BIOS の同等のものより目に見えて遅いのです。

```c
#include <stdio.h>
#include <tonc.h>

// === swi calls ======================================================

// Their assembly equivalents can be found in tonc_bios.s

void VBlankIntrWait()
{   swi_call(0x05); }

int Div(int num, int denom)
{   swi_call(0x06); }

u32 Sqrt(u32 num)
{   swi_call(0x08); }

s16 ArcTan2(s16 x, s16 y)
{   swi_call(0x0a); }

void ObjAffineSet(const AFF_SRC *src, void *dst, int num, int offset)
{   swi_call(0x0f); }


// === swi demos ======================================================

// NOTE!
// To be consistent with general mathematical graphs, the 
// y-axis has to be reversed and the origin moved to the 
// either the bottom or mid of the screen via
// "iy = H - y"
// or
// "iy = H/2 - y"
//
// functions have been scaled to fit the graphs on the 240x160 screen

// y= 2560/x
void div_demo()
{
    int ix, y;

    for(ix=1; ix<SCREEN_WIDTH; ix++)
    {
        y= Div(0x0a000000, ix)>>16;
        if(y <= SCREEN_HEIGHT)
            m3_plot(ix, SCREEN_HEIGHT - y, CLR_RED);
    }
    tte_printf("#{P:168,132;ci:%d}div", CLR_RED);
}

// y= 160*sqrt(x/240)
void sqrt_demo()
{
    int ix, y;
    for(ix=0; ix<SCREEN_WIDTH; ix++)
    {
        y= Sqrt(Div(320*ix, 3));
        m3_plot(ix, SCREEN_HEIGHT - y, CLR_LIME);
    }
    tte_printf("#{P:160,8;ci:%d}sqrt", CLR_LIME);
}

// y = 80 + tan((x-120)/16) * (64)*2/pi
void arctan2_demo()
{
    int ix, y;
    int ww= SCREEN_WIDTH/2, hh= SCREEN_HEIGHT/2;
    for(ix=0; ix < SCREEN_WIDTH; ix++)
    {
        y= ArcTan2(0x10, ix-ww);
        m3_plot(ix, hh - y/256, CLR_MAG);
    }
    tte_printf("#{P:144,40;ci:%d}atan", CLR_MAG);
}

// wX= 1, wY= 80
// cc= 80*sx*cos(2*pi*alpha/240)
// ss=  1*sy*sin(2*pi*alpha/240)
void aff_demo()
{
    int ix, ss, cc;
    ObjAffineSource af_src= {0x0100, 0x5000, 0};    // sx=1, sy=80, alpha=0
    ObjAffineDest af_dest= {0x0100, 0, 0, 0x0100};  // =I (redundant)

    for(ix=0; ix<SCREEN_WIDTH; ix++)
    {
        ObjAffineSet(&af_src, &af_dest, 1, BG_AFF_OFS);
        cc= 80*af_dest.pa>>8; 
        ss= af_dest.pc>>8;
        m3_plot(ix, 80 - cc, CLR_YELLOW);
        m3_plot(ix, 80 - ss, CLR_CYAN);
        // 0x010000/0xf0 = 0x0111.111...
        af_src.alpha += 0x0111;
    }

    tte_printf("#{P:48,38;ci:%d}cos", CLR_YELLOW);
    tte_printf("#{P:72,20;ci:%d}sin", CLR_CYAN);
}

// === main ===========================================================

int main()
{
    REG_DISPCNT= DCNT_MODE3 | DCNT_BG2;

    tte_init_bmp_default(3);
    tte_init_con();

    div_demo();
    sqrt_demo();
    aff_demo();

    arctan2_demo();
    
    while(1);
    
    return 0;
}
```

## VSync その 2、VBlankIntrWait {#sec-vsync2}

これまで、デモはすべて、動作を VBlank に同期させるのに `vid_vsync` 関数を使ってきました（[グラフィックの入門](video.html#sec-vsync1)を見てください）。この関数は `REG_VCOUNT` を確認し、次の VBlank が来るまで while ループの中にとどまるというものでした。動きはしますが、2 つの理由から、本当にかなりまずいやり方です。1 つ目は、すでに VBlank の中にいる場合に起こりうる問題ですが、それは対処済みでした。2 つ目の理由のほうが重要です。while ループの中にいる間、ものすごく多くの CPU のサイクルを無駄にしていて、そのすべてが電池を消耗させるのです。

<div class="cpt_fr" style="width:240px;">
<img alt="swi_vsync" src="./img/demo/swi_vsync.png" id="fig:swi-vsync">

**{*@fig:swi-vsync}**: `swi_vsync` デモ。
</div>

CPU を低消費電力モードにして、電池を節約できる BIOS コールがいくつかあります。そのための主な BIOS コールは Halt（#2）ですが、今興味があるのは VBlankIntrWait（#5）です。これは、次の VBlank 割り込みまで待つように準備します。これを使うには、もちろん割り込み、特に VBlank 割り込みを有効にしておく必要があります。いつものように、VBlank の ISR は `REG_IF` に書き込んで割り込みを処理済みにしなければなりません。しかし、それに*加えて*、BIOS 版の `REG_IFBIOS` にも書き込む必要があります。このちょっとした情報は、ほかではなかなか見つかりません（BIOS コールを扱うチュートリアルが少ないこともその一因です）。詳しくは [GBATEK, BIOS Halt Functions](https://problemkaputt.de/gbatek.htm#bioshaltfunctions) を見てください。幸い、[割り込み](interrupts.html#sec-switch)の節で示した交換台には、これが組み込まれています。

準備の仕方を示すために、`swi_vsync` デモを見てください。最も重要なコードを以下に示します。スクリーンショットは図 2 にあります。このデモは、π rad/s の角速度（これは Δθ = 0x10000/4/60= 0x0111 に相当します）で回転するメトロイドのスプライトを表示します。割り込みを扱う基本的な手順はおなじみのはずですが、交換台がすでに割り込みの処理済みの通知を面倒みてくれるので、実際の VBlank の ISR はありません。そのあとはかなり単純です。`ObjAffineSet()` を使って必要なアフィン行列を計算し、`VBlankIntrWait` で次の VBlank 割り込みまで CPU を Halt させます。

```c
// inside main, after basic initialisations

AFF_SRC as= { 0x0100, 0x0100, 0 };
OBJ_AFFINE oaff;

// enable isr switchboard and VBlank interrupt
irq_init(NULL);
irq_add(II_VBLANK, NULL);

while(1)
{
    VBlankIntrWait();

    // Full circle = 10000h
    // 10000h/4/60= 111h -> 1/4 rev/s = 1 passing corner/s
    as.alpha += 0x0111;
    ObjAffineSet(&as, &oaff.pa, 1, 8);

    obj_aff_copy(obj_aff_mem, &oaff, 1);
}
```

:::tip vid_vsync() より VBlankIntrWait() を使おう

`vid_vsync()`（や機能的に同等のもの）で VBlank を待つのは良い考えではありません。電池を無駄にしすぎます。おすすめの手順は、`VBlankIntrWait()` を使ってプロセッサを止め、VBlank 割り込みで再び起こしてもらうことです。

:::

:::warning IntrWait のルーチンでの処理済みの通知

`VBlankIntrWait()` は、割り込みが発生するまで CPU を止められる BIOS の `IntrWait()` ルーチンの 1 つにすぎません。ただし、これは割り込みの処理済みの通知を、`REG_IF` ではなく `REG_IFBIOS`（0300:7FF8）で確認します。`VBlankIntrWait()` を試したあとでゲームが固まってしまうなら、これが原因かもしれません。このアドレスには公式の名前がないので、ほかの名前で見かけることもあるかもしれないことに注意してください。

:::

## 最後に {#sec-concs}

使い方がわかったところで、使いすぎないように注意しておくべきでしょう。BIOS のルーチンは速さではなく大きさを重視して設計されているようなので、世界一速いわけではありません。それだけでなく、1 回ごとに少なくとも 60 サイクルのオーバーヘッドがあります（ちなみに、普通の関数には 30 サイクルのオーバーヘッドがあるようです）。速さを求めているなら、BIOS コールは最善の選択ではないかもしれません。ウェブ上の……どこかで、もっと速いルーチンがおそらく見つかるでしょう。もちろん、だからといって BIOS のルーチンが役に立たないわけではありませんが、ほかの方法があるなら、そちらを使ってください。ただ、それは最適化の手順であり、早まってするべきではないことは覚えておいてください。
