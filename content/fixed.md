# B. 固定小数点数とルックアップテーブル

<!-- toc -->

## 固定小数点数とは {#sec-intro}

大まかに言うと、数には 2 つの種類があります。整数と浮動小数点数です。本格的な数学のほとんどでは、整数ではどうにもなりません。定義上、整数は分数を許さないからです。そこで 3D のゲームでは浮動小数点の計算を使います。昔、専用の浮動小数点のハードウェアが登場する前は、それはとても遅いものでした！ 少なくとも、整数の算術より遅かったのです。幸い、整数で小数点付きの数をごまかす方法があります。これは<dfn>固定小数点演算</dfn>として知られています。

### 一般的な固定小数点数 {#ssec-fix-gen}

例を挙げます。財布に \$10.78（10 ドル 78 セント）あるとします。この額を整数で書きたいと思うと、問題があります。小数部を切り捨てる（\$10）か、\$11 に丸めるしかないからです。しかし、ドルではなく*セント*で書くこともできます。そうすれば 1078 と書け、これは整数なので、問題は解決です。

固定小数点演算はこうやって働きます。単位を数える代わりに、*分数*を数えるのです。前の例では、セント、つまり 100 分の 1 単位で数えています。固定小数点数には整数部（「10」）と小数部（「78」）があります。小数部に 2 桁あるので、これを <dfn>x.2</dfn> 形式の固定小数点数と呼びます。

PC には 1990 年代半ばから浮動小数点演算ユニット（FPU）があることに注意してください。これで浮動小数点の算術は整数の算術と同じくらい速く（ときにはもっと速く）なったので、固定小数点演算を使う手間は、`float` から `int` への変換がまだ遅いラスタライズなどを除けば、あまり割に合いません。しかし、GBA は浮動小数点の処理が苦手なので、ここでは徹頭徹尾、固定小数点です。

### GBA での固定小数点の使い方 {#ssec-fix-gba}

コンピュータは [2 進法](numbers.html#ssec-num-basen)を使うので、固定小数点の基礎に 10 進数を使うのはばかげています。幸い、固定小数点演算は 2 進数を含めどの基数でもできます。基本の形式は *i*.*f* で、*i* は整数部のビット数、*f* は小数部のビット数です。小数部だけ知っていれば十分なことが多いので、「.*f*」という表示だけを見かけることもあるでしょう。

GBA はいくつかの場面で固定小数点演算を使います。たとえば、[アフィンパラメータ](affine.html)はどれも .8 の固定小数点数（略して「<dfn>固定小数点</dfn>」（fixeds））です。実質的に、これは 1/2<sup>8</sup> = 1/256 単位で数えていることを意味し、0.004 の精度が得られます。ですから、REG_BG2PA のようなレジスタに 256 を書き込むと、これは実際には 256/256=1.00 と解釈されます。REG_BG2PA=512 なら 2.00、640 なら 2.50、という具合です。もちろん 10 進数ではわかりにくいですが、電卓を手に取れば、それが正しいとわかるでしょう。このため、16 進数で書くほうが便利なことがよくあります。256=0x100→1.00、512=0x200→2.00、640=0x280→2.50 です（8 は 16/2、つまり 2 分の 1 であることを思い出してください）。

```c
// .8 fixed point examples : counting in fractions of 256

int a= 256;         // 256/256 = 1.00
int a= 1 << 8;      // Ditto
int a= 0x100;       // Ditto

int b= 0x200;       // 0x200/256 = 512/256 = 2.00
int c= 0x080;       // 0x080/256 = 128/256 = 0.50
int d= 0x280;       // 0x280/256 = 640/256 = 2.50
```

固定小数点が使われているのはアフィンのレジスタだけではありませんが、一番わかりやすいのはそこです。[ブレンドの重み](gfx.html#sec-blend)も本質的には固定小数点数で、ただ .8 ではなく 1.4 の固定小数点です。実はこれは重要な点です。小数点を置く位置は任意で、途中で位置を変えることさえできます。さて、数そのものは小数点がどこにあるかを教えてくれないので、自分で覚えておくか、もっと良いのはコメントに書いておくことが大事です。信じてください。長いアルゴリズムの途中で、小数点の位置を推測したくはないはずです。

:::tip 固定小数点の位置をコメントに書こう

固定小数点の変数を使うときは、その固定小数点の形式を示すようにしてください。使う演算によって小数点の位置が動くことのある、長い計算でそれらが必要なときは特にそうです。

:::

### 固定小数点と符号 {#ssec-fix-sign}

固定小数点数は、浮動小数点数の貧乏人向けの代用品のはずで、浮動小数点数には負の数も含まれます。つまり、固定小数点数は*符号付き*であるべきです。少なくとも、普通は。たとえば、アフィンのレジスタは符号付きの 8.8 の固定小数点を使いますが、ブレンドの重みは符号なしの 1.4 の固定小数点です。大したことではないと思うかもしれませんが、気をつけないと[符号](numbers.html#bits-int-sign)は本当に物事をめちゃくちゃにしかねません。位置と速度に固定小数点を使っているとします。位置が常に正だとしても、速度はそうではないので、符号付きの数のほうがふさわしいでしょう。さらに、固定小数点数がハーフワード、たとえば 8.8 の固定小数点なら、符号付きの「−1」は `0xFFFFFFFF`、つまり正しい「−1」として使われますが、符号なしの「−1」は `0x0000FFFF` で、これは実際には正の数です。これにつまずくのはあなたが最初ではありませんし、最後でもないでしょう。ですから、符号付きの固定小数点を使ってください。

もう 1 つの注意点は、符号付きの固定小数点がよく表される書き方です。「1.*n*.*f*」の形のものを見かけるかもしれません。これは、符号ビットが 1 つ、整数部が *n* ビット、小数部が *f* ビットであることを示すつもりのものです。厳密に言えば、これは**誤り**です。固定小数点数は分数として解釈されるだけの、ただの整数です。つまり、[2 の補数](numbers.html#bits-int-neg)に従い、最上位ビットが設定されていれば負の数を示すとはいえ、それは*独立した*符号ビットではありません。先ほど言ったように、2 の補数での「−1」は `0xFFFFFFFF` で、符号と絶対値の表現の場合のような `0x80000001` ではありません。この区別は大したことではなく、やはり 2 の補数なのは明らかだと思うかもしれませんが、浮動小数点の形式には独立した符号ビットが*ある*ことを考えると、覚えておく価値があると思います。

:::warning 符号付きの固定小数点の形式の表記

符号付きの固定小数点の形式は、「1.*n*.*f*」と表されることがあります。そこから、浮動小数点の形式のように独立した符号ビットがあると思うかもしれませんが、それは**正しくありません**。これらはやはり普通の整数で、負の数には 2 の補数を使います。

:::

## 固定小数点演算 {#sec-fmath}

固定小数点数が何かを知ることと、それを何らかの形で使うことは別です。ここで関係するのは 3 つのことです。

-   普通の整数や浮動小数点数と、固定小数点数の間の変換。
-   算術演算。
-   オーバーフロー。

どれも理解するのは難しくありませんが、それぞれにやっかいな問題があります。実際、オーバーフローは単に問題であって、項目とは言えません。この節では、24.8 の符号付きの固定小数点に焦点を当て、そのために「FIXED」と typedef した int を使います。この固定小数点の形式しか使いませんが、ここで扱う話題はほかの形式にも簡単に応用できます。

### 固定小数点への変換と固定小数点からの変換 {#ssec-fmath-conv}

ここで「変換」という言葉が正しいのかどうかさえよくわかりません。固定小数点数と普通の数の唯一の違いは、スケール係数 *M* です。FIXED から int や float に移るのに必要なのは、そのスケールを掛け算か割り算で考慮することだけです。そう、本当にそれだけ単純なのです。スケールに 2 の累乗を使っているので、整数↔FIXED の変換はシフトでもできます。シフトを自分でコードに書いてもかまいませんが、コンパイラは賢いので、2 の累乗の掛け算や割り算を自分でシフトに変換してくれます。

```c
typedef s32 FIXED;         //! 32bit FIXED in 24.8 format

// For other fixed formats, change FIX_SHIFT and the rest goes with it.

//! Convert an integer to fixed-point
INLINE FIXED int2fx(int d)
{   return d<<FIX_SHIFT;    }

//! Convert a float to fixed-point
INLINE FIXED float2fx(float f)
{   return (FIXED)(f*FIX_SCALEF);   }


//! Convert a fixed point value to an unsigned integer.
INLINE u32 fx2uint(FIXED fx)    
{   return fx>>FIX_SHIFT;   }

//! Get the unsigned fractional part of a fixed point value (orly?).
INLINE u32 fx2ufrac(FIXED fx)
{   return fx&FIX_MASK; }

//! Convert a FIXED point value to an signed integer.
INLINE int fx2int(FIXED fx)
{   return fx/FIX_SCALE;    }

//! Convert a fixed point value to floating point.
INLINE float fx2float(FIXED fx)
{   return fx/FIX_SCALEF;   }
```

#### 丸めと負の数の食い違い

変換は上で説明したのとほぼ同じくらい単純です。問題になりうるのは 2 か所で、丸めの食い違いと負の分数です。問題に*なりうる*と言ったことに注意してください。何を考えていたかによります。ここでは細かいことをすべて説明するつもりはありません。たいていはあまり問題にならないからですが、知っておく必要はあります。

プログラミングが初めてでなければ、浮動小数点数から整数への丸めの問題は間違いなく知っているでしょう。単純なキャストによる変換は数を切り捨てるだけで、本当に丸めるわけではありません。たとえば、「(int)1.7」の結果は 2 ではなく 1 です。先ほどのマクロにも同じ問題があります（それを問題と呼べるなら）。浮動小数点数から整数への丸めは、丸める前に浮動小数点数に 2 分の 1（0.5）を足すことで行い、これは固定小数点の変換にも適用できます。もちろんこの場合、2 分の 1 の値は固定小数点のビット数によります。たとえば .8 の固定小数点なら ½ は 0x80=128（256/2）で、.16 の固定小数点なら 0x8000=32768 です。下にシフトする前にこれを足せば、正しく丸められます。実は丸めの方法はいくつもあり、それについては ["An Introduction to Fixed Point Math" by Brian Hook](https://web.archive.org/web/20060204155500/http://www.bookofhook.com/Article/GameDevelopment/AnIntroductiontoFixedPoin.html) で読めます。

それから負の数です。率直に言って、負の整数の割り算はいつも厄介です。ここでの基本的な問題は、常にゼロに向かって丸められることです。+3/4 も −3/4 も 0 になります。ある意味ではこれは筋が通っていますが、一点では通っていません。ゼロの周りで出力の並びが途切れてしまうのです。これだけでも腹立たしいのですが、もっと悪いのは、右シフトはこのふるまいに従わ*ない*ことです。右シフトは常に負の無限大に向かってシフトします。言い換えると、負の整数の割り算では、割り算と右シフトの演算子は同じでは*ない*のです。どちらの方法を選ぶかは、あなたの設計上の検討事項です。個人的には、より一貫した結果が得られるので、シフトを選びたくなります。

<div class="cblock">
<table id="tbl:neg-div" class="table-data">
<caption align="bottom">
  <b>*@tbl:neg-div</b>:
  ゼロの周りでの割り算と右シフト。
</caption>
<col span=19 width=20>
<tbody align="center">
<tr>
  <th><i>x</i> 
  <td> -8<td> -7<td> -6<td> -5<td> -4<td> -3<td> -2<td> -1<td>  0
  <td>  1<td>  2<td>  3<td>  4<td>  5<td>  6<td>  7<td>  8<td>  9
<tr>
  <th><i>x</i>/4 
  <td> -2<td> <b>-1</b><td> <b>-1</b><td> <b>-1</b><td> -1
  <td>  <b>0</b><td>  <b>0</b><td>  <b>0</b><td>  0
  <td>  0<td>  0<td>  0<td>  1<td>  1<td>  1<td>  1<td>  2<td>  2

<tr>
  <th><i>x</i>&gt;&gt;2 
  <td> -2<td> <b>-2</b><td> <b>-2</b><td> <b>-2</b><td> -1
  <td> <b>-1</b><td> <b>-1</b><td> <b>-1</b><td>  0
  <td>  0<td>  0<td>  0<td>  1<td>  1<td>  1<td>  1<td>  2<td>  2
</tbody>
</table>
</div>

負の割り算の厄介さは、小数部を扱おうとするとさらにひどくなります。AND でマスクすると、実質的に数の符号が壊れます。たとえば、8.8 の −2¼ は −0x0240 = 0xFDC0 です。これを 0xFF でマスクすると 0xC0 = ¾ になり、正の数で、しかも小数部も間違っています。一方で、0xFDC0\>\>8 は良くも悪くも −3 で、−3 + ¾ はたしかに −2¼ なので、その意味ではうまくいっています。それが*あなた*にとってうまくいくかどうかは、自分で決める必要があります。固定小数点数を何らかの形で（たとえばこの場合なら -2.40 のように）表示したいなら、単なるシフトとマスクより少し工夫が必要です。今のところ、それには触れるつもりもありません。

:::warning 負の固定小数点数の変換

負の固定小数点数から整数への変換は特にややこしいもので、同じくらい妥当な解決策が複数あるという事実がそれをさらに複雑にしています。どれを選ぶべきかはあなた次第です。できるなら、その可能性を避けてください。固定小数点→整数の変換はたいてい算術の最終段階のためのもので、何とかしてそれらの数が正になるようにできるなら、そうしてください。

:::

### 算術演算 {#ssec-fmath-ops}

固定小数点数もやはり整数なので、整数と同じ算術演算を使います。ただし、ときには小数点を正しい位置に保つために、多少の注意が必要です。手順は 10 進数の小数の算術と同じです。たとえば、0.01+0.02 = 0.03 です。この和を求めるときは普通、小数点を取り除いて 1 と 2 を残し、それらを足して 3 にして、小数点を戻します。固定小数点も基本的にそうやって働きます。しかし、たとえば 0.1 と 0.02 を足すとき、固定小数点の小数部は 1 と 2 ではなく、**10** と 2 です。ここでの鍵は、足し算（と引き算）では小数点が同じ位置にあるべきだということです。

掛け算と割り算でも同じようなことが起きます。0.2×0.3 という掛け算を考えてみてください。2×3 は 6 で、小数点を戻すと 0.6、ですよね？ まあ、小学校でちゃんと宿題をしていれば、結果は実際には 0.06 になるべきだとわかるでしょう。小数部が掛け合わされるだけでなく、*スケール*も掛け合わされるのです。

これらはどちらも固定小数点の算術にも当てはまります。常に同じ固定小数点を使えば、足し算と引き算は問題になりません。掛け算と割り算では、余分なスケール係数も考慮する必要があります。固定小数点どうしの掛け算では、あとでスケールで割る必要があり、固定小数点どうしの割り算では、割り算の*前に*スケールを掛ける必要があります。どちらの場合も、スケールの補正をその位置で行う理由は、最も高い精度を保つためです。式 1 と 2 は、これをもっと数学的な形で示しています。固定小数点数は常に、定数掛ける固定のスケール *M* で与えられます。足し算と引き算はスケールを保ちますが、掛け算と割り算は保たないので、それぞれスケール係数を取り除くか加える必要があります。

<!--
\begin{matrix}
fa & = & a \cdot M \\
fb & = & b \cdot M
\end{matrix}
-->
<table id="eq:fix-add">
<tr>
  <td class="eqnrcell">(!@eq:fix-add)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtable columnspacing="1em" rowspacing="4pt">
              <mtr>
                <mtd>
                  <mi>f</mi>
                  <mi>a</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>a</mi>
                  <mo>&#x22C5;</mo>
                  <mi>M</mi>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>f</mi>
                  <mi>b</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>b</mi>
                  <mo>&#x22C5;</mo>
                  <mi>M</mi>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </td>
</tr>
</table>

<!--
\begin{matrix}
fc & = & fa+fb & = & (a+b) \cdot M \\
fd & = & fa-fb & = & (a-b) \cdot M \\
fe & = & fa \cdot fb & = & (a \cdot b) \cdot M^2 \\
ff & = & fa / fb & = & a/b
\end{matrix}
-->
<table id="eq:fix-mul">
<tr>
  <td class="eqnrcell">(!@eq:fix-mul)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtable columnspacing="1em" rowspacing="4pt">
              <mtr>
                <mtd>
                  <mi>f</mi>
                  <mi>c</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>f</mi>
                  <mi>a</mi>
                  <mo>+</mo>
                  <mi>f</mi>
                  <mi>b</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mo stretchy="false">(</mo>
                  <mi>a</mi>
                  <mo>+</mo>
                  <mi>b</mi>
                  <mo stretchy="false">)</mo>
                  <mo>&#x22C5;</mo>
                  <mi>M</mi>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>f</mi>
                  <mi>d</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>f</mi>
                  <mi>a</mi>
                  <mo>&#x2212;</mo>
                  <mi>f</mi>
                  <mi>b</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mo stretchy="false">(</mo>
                  <mi>a</mi>
                  <mo>&#x2212;</mo>
                  <mi>b</mi>
                  <mo stretchy="false">)</mo>
                  <mo>&#x22C5;</mo>
                  <mi>M</mi>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>f</mi>
                  <mi>e</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>f</mi>
                  <mi>a</mi>
                  <mo>&#x22C5;</mo>
                  <mi>f</mi>
                  <mi>b</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mo stretchy="false">(</mo>
                  <mi>a</mi>
                  <mo>&#x22C5;</mo>
                  <mi>b</mi>
                  <mo stretchy="false">)</mo>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mi>M</mi>
                    <mn>2</mn>
                  </msup>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>f</mi>
                  <mi>f</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>f</mi>
                  <mi>a</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>f</mi>
                  <mi>b</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>a</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>b</mi>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</tr>
</table>

```c
//! Add two fixed point values
INLINE FIXED fxadd(FIXED fa, FIXED fb)
{   return fa + fb;         }

//! Subtract two fixed point values
INLINE FIXED fxsub(FIXED fa, FIXED fb)
{   return fa - fb;         }

//! Multiply two fixed point values
INLINE FIXED fxmul(FIXED fa, FIXED fb)
{   return (fa*fb)>>FIX_SHIFT;              }

//! Divide two fixed point values.
INLINE FIXED fxdiv(FIXED fa, FIXED fb)
{   return ((fa)*FIX_SCALE)/(fb);           }
```

### オーバーフローとアンダーフロー {#ssec-fmath-flow}

これは実は、掛け算と割り算のスケールの問題の一部です。オーバーフローとは、演算の結果が、それを格納するビット数より大きくなることです。これはどんな整数の掛け算でも起こりうる問題ですが、固定小数点演算ではずっと頻繁に起きます。固定小数点数はスケールが上がっているだけでなく、固定小数点数どうしを掛けるとスケールが*2 回*上がるからです。.8 の固定小数点の掛け算では「1」が 2<sup>16</sup> になり、これはすでにハーフワードの範囲外です。

余分なスケールに対処する 1 つの方法は、掛け算のあとではなく前に補正することです。ただし、その過程で精度が少し失われます。良い妥協案は、両方のオペランドを、全体のシフト量の半分ずつ右シフトすることでしょう。

固定小数点の割り算には、アンダーフローと呼ばれる似た問題があります。簡単な例として、整数の割り算 *a*/*b* で *b*\>*a* のときに何が起きるかを考えてみてください。そうです。分数が欲しいのに、結果はゼロになってしまいます。このふるまいを直すには、まず分子を *M* だけスケールアップします（それがオーバーフローの問題につながるかどうかはわかりませんが <kbd>:P</kbd>）。

ご覧のとおり、固定小数点演算の原理はそれほど難しくも魔法のようでもありません。でも、頭をはっきりさせておく必要があります。シフトを 1 つ忘れたり間違った場所に置いたりすれば、全体が崩れます。新しいアルゴリズムに取り組んでいるなら、まず浮動小数点数で（できれば PC で）やってみて、アルゴリズム自体が動くと確信してから固定小数点に変換することを考えてください。

## 割り算をごまかす（任意） {#sec-rmdiv}

:::warning 数学が重く、任意

この節は、ときどき役に立つ最適化の技法についてです。技法を紹介するだけでなく、その使い方と安全の限界も導きます。そのため、途中にいくらか厄介な数学があります。この節を詳しく知らなくてもまったく問題ない可能性が高いですが、必要なら遅い割り算を取り除くのに役立ちます。

:::

「定数による割り算はその逆数による掛け算である」という言葉を聞いたことがあるかもしれません。この技法を使えば、割り算を取り除いて、ずっと速い掛け算に置き換えられます。たとえば *x*/3 = *x*·(1/3) = *x*·0.333333 です。一見すると、これは役に立たないように思えます。1/*y* の整数の形は定義上常にゼロですし、代わりの浮動小数点もあまり良いものではありませんし、そこにたどり着くのにさえ*やはり*割り算が必要です！ これはすべて本当ですが、重要なのは、これらの問題は避けられるということです。整数/浮動小数点の問題は、代わりに固定小数点を使えば解決できます。割り算については、話しているのは*定数*による割り算で、定数の算術は実行時ではなくコンパイル時に行われることを思い出してください。というわけで問題は解決、ですよね？ うーん、ええ。たしかに。*表面的な*問題は解決しましたが、今度はオーバーフローと丸めという昔からの 2 つの問題が、また醜い頭をもたげてきます。

下は「*x*/12」を評価するコードです。ARM でコンパイルしたコードは 1/12 の .33 の固定小数点を作り、それから割り算のために 64 ビットの掛け算を使います。一方、Thumb 版はこれをしない（そして実際にできない）ので、標準の遅い割り算のルーチンを使います。この時間のかかる割り算を取り除きたければ、自分で面倒をみなければなりません。念のため言っておくと、ええ、ARM アセンブリを知っていても、それがなぜそうしているのかを追うのは難しいかもしれないことはわかっています。この節はそのためにあります。

```armasm
@ Calculating y= x/12

@ === Thumb version ===
    ldr     r0, .L0     @ load numerator
    ldr     r0, [r0]
    mov     r1, #12     @ set denominator
    bl      __divsi3    @ call the division routine
    ldr     r1, .L0+4
    str     r0, [r1]
.L0:
    .align  2
    .word   x
    .word   y

@ === ARM version ===
    ldr     r1, .L1         @ Load M=2^33/12
    ldr     r3, .L1+4
    ldr     r3, [r3]        @ Load x
    smull   r2, r0, r1, r3  @ r0,r2= x*M (64bit)
    mov     r3, r3, asr #31     @ s = x>=0 ? 0 : -1 (for sign correction)
    rsb     r3, r3, r0, asr #1  @ y= (x*M)/2 - s = x/12
    ldr     r1, .L1+8
    str     r3, [r1]        @  store y
.L1:
    .align  2
    .word   715827883   @ 0x2AAAAAAB (≈ 2^33/12 )
    .word   x
    .word   y
```

この節の残りは、これらの問題を見分けて対処すること、そしてこの技法を安全に使うための指針を導くことについてです。でもまず、いくつかの定義が必要です。

整数の割り算。正の整数 *p*、*q*、*r*

<!--
r = \left\lfloor p/q \right\rfloor \iff p = r \cdot q + p \text{%}q
-->
<table id="eq:int-div">
<tr>
  <td class="eqnrcell">(!@eq:int-div)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>r</mi>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">&#x230A;</mo>
              <mi>p</mi>
              <mrow data-mjx-texclass="ORD">
                <mo>/</mo>
              </mrow>
              <mi>q</mi>
              <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
            </mrow>
            <mstyle scriptlevel="0">
              <mspace width="0.278em"></mspace>
            </mstyle>
            <mo stretchy="false">&#x27FA;</mo>
            <mstyle scriptlevel="0">
              <mspace width="0.278em"></mspace>
            </mstyle>
            <mi>p</mi>
            <mo>=</mo>
            <mi>r</mi>
            <mo>&#x22C5;</mo>
            <mi>q</mi>
            <mo>+</mo>
            <mi>p</mi>
            <mo>%</mo>
            <mi>q</mi>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </td>
</tr>
</table>

近似。正の整数 *x*、*y*、*a*、*m*、*n* と、実数の誤差項 δ
<!--
y = \left\lfloor x/a \right\rfloor = \left\lfloor (x \cdot m)/n \right\rfloor + \delta
-->
<table id="eq:div-aprx">
<tr>
  <td class="eqnrcell">(!@eq:div-aprx)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>y</mi>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">&#x230A;</mo>
              <mi>x</mi>
              <mrow data-mjx-texclass="ORD">
                <mo>/</mo>
              </mrow>
              <mi>a</mi>
              <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
            </mrow>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">&#x230A;</mo>
              <mo stretchy="false">(</mo>
              <mi>x</mi>
              <mo>&#x22C5;</mo>
              <mi>m</mi>
              <mo stretchy="false">)</mo>
              <mrow data-mjx-texclass="ORD">
                <mo>/</mo>
              </mrow>
              <mi>n</mi>
              <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
            </mrow>
            <mo>+</mo>
            <mi>&#x3B4;</mi>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </td>
</tr>
</table>

整数の割り算を示すのに床関数（&#x230A;*p*/*q*&#x230B;）を使っています。これは基本的に、実数の割り算の切り捨て版です。いつものように、剰余は余りで、普通は *p* − *r·q* で計算します。1/*a* の近似の鍵は *m* と *n* の項にあります。この場合 *n* は、シフトを使えるように 2 の累乗 *n*=2<sup>F</sup> にしますが、そうである必要はありません。δ はどんな近似にもつきものの誤差項です。ここでは正の整数しか使っていないことに注意してください。負の数では、「本当の」割り算をまねたければ、結果に 1 を足す必要があります。（あるいは符号ビットを引きます。上の ARM アセンブリでわかるように、これも同じようにうまくいきます。）

:::note 負の割り算のごまかしと丸め

この節は正の数についてのものです。標準の整数の割り算の結果（ゼロへの丸め）が欲しければ、分子が負なら 1 を足す必要があります。これは符号ビットを引くことですばやくできます。

```c
// pseudo-code for division by constant M
int x, y;
y= fake_div(x, M);  // shift-like div
y -= y>>31;         // convert to /-like division
```

負の無限大へ丸めたいなら、何か別のことをする必要があります。でも正直に言うと、何をすればいいのかはよくわかりません。

:::

### 理論 {#ssec-rmdiv-try}

成功するには 2 つのものが必要です。まず、*m* を求める方法。次に、近似がいつ失敗するかを判断する方法です。後者は @eq:div-aprx から導けます。近似の誤差は &#x230A;ε/*n*&#x230B; で与えられるので、これがゼロである限り安全です。

<!--
x \cdot m - n \cdot \left\lfloor x/A \right\rfloor = \varepsilon \\
\text{Fail if: } \varepsilon \ge n
-->
<table id="eq:aprx-fail">
<tr>
  <td class="eqnrcell">(!@eq:aprx-fail)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>x</mi>
            <mo>&#x22C5;</mo>
            <mi>m</mi>
            <mo>&#x2212;</mo>
            <mi>n</mi>
            <mo>&#x22C5;</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">&#x230A;</mo>
              <mi>x</mi>
              <mrow data-mjx-texclass="ORD">
                <mo>/</mo>
              </mrow>
              <mi>A</mi>
              <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
            </mrow>
            <mo>=</mo>
            <mi>&#x3B5;</mi>
          </mtd>
        </mtr>
        <mtr>
          <mtd>
            <mtext>失敗する条件:&#xA0;</mtext>
            <mi>&#x3B5;</mi>
            <mo>&#x2265;</mo>
            <mi>n</mi>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </td>
</tr>
</table>

*m* を求めることについてです。&#x230A;1/*A*&#x230B; = &#x230A;(*n·A*)/*n*&#x230B; であることを思い出すと、*m* = &#x230A;*n*/*A*&#x230B; を使うのが良い値に思えるでしょう。しかし、そうではありません。

ここで少し例を挙げるのがよいでしょう。最初と同じく、*A* = 3 の場合を考えます。ここでは .8 の固定小数点数、つまり *k* = 8 と *n*=256 を使います。試しの *m* は *m* = &#x230A;*n*/*A*&#x230B; = 85 = 0x55 で、余りは 1 です。

別の見方は、16 進数の浮動小数点に移って、最初の *F* ビットを取ることです。これは思ったほど難しくありません。分数の浮動小数点数を求めるやり方は、基数を掛けて整数部を書き留め、余りに基数を掛けて整数部を書き留め、ということを繰り返すことです。下の表は 1/7 の 16 進数版です（1/3 はかなり単調なので使っていません）。ご覧のとおり、1/7 は 16 進数で 0.249249…h です。3 分の 1 でこれをすると 0.5555…h になります。

<div class="lblock">
<table id="tbl:hexfloat" class="table-data">
<caption align="bottom">
  <b>*@tbl:hexfloat</b>: 
  基数 <i>B</i>=16 での 1/7 の浮動小数点表現
</caption>
<tbody align="center">
<tr><th> x <th> x&middot;B <th> x&middot;&#x230A;B/7&#x230B; <th> x&middot;B%7
<tr><th width=16> 1 <td> 16  <td> 2     <td> 2
<tr><th> 2 <td> 32  <td> 4     <td> 4
<tr><th> 4 <td> 64  <td> 9     <td> 1
<tr><th> 1 <td> 16  <td> 2     <td> 2
<tr><th> 2 <td> 32  <td> 4     <td> 4
</tbody>
</table>
</div>

というわけで、1/3 は 16 進数ではゼロのあとに 5 が続くもので、切り捨てた .8 の固定小数点の表記では単に *m*=0x55 です。さて、逆数を掛けるやり方をすると何が起きるか見てみましょう。ここでは 16 進数の浮動小数点数と、@eq:div-aprx のとおり *y*=&#x230A;(*x·m*)/*n*&#x230B; を使います。実際に得られる結果は整数部だけです。（16 進の）小数部は無視してください。

<div class="cblock">
<table id="tbl:rmdiv-bad" class="table-data">
<caption align="bottom">
  <b>*@tbl:rmdiv-bad</b>: 
  <i>m</i>= &#x230A;256/3&#x230B; = 0x55 を使った <i>x</i>/3。3、6、&hellip; でだめ
</caption>
<tbody align="center">
<tr><th>x
  <th> 0 <th> 1 <th> 2 <th> 3 <th> 4 <th> 5 <th> 6 <th> 7
<tr><th>y=(x&middot;m)&gt;&gt;F
  <td> 0.00h        <td> 0.55h <td> 0.AAh 
  <td> <b>0.FFh</b> <td> 1.54h <td> 1.A9h
  <td> <b>1.FEh</b> <td> 2.53h
<tr><th>本当の <i>x</i>/3
  <td> 0 <td> 0 <td> 0 <td> 1 <td> 1 <td> 1 <td> 2 <td> 2
</tbody>
</table>
</div>

ご覧のとおり、問題は*ほぼすぐに*起きます！ *x*=*A* に達することすらできずに困ったことになります。これは精度の問題では*ありません*。.128 の固定小数点数を使っても、やはりずれます。これは純粋に<dfn>丸め誤差</dfn>の結果で、浮動小数点数でもまったく同じように起きます。逆数による割り算を使うときは、*m* を切り捨てではなく切り*上げ*るべきです。ここではアラインメントの技が使えます。*A*−1 を足してから割るのです。これで *m*=0x56 になり、安全になります。少なくとも、しばらくの間は。

<!--
m = \left\lfloor (n+A-1)/A \right\rfloor
-->
<table id="eq:reci-m">
<tr>
  <td class="eqnrcell">(!@eq:reci-m)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>m</mi>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">&#x230A;</mo>
              <mo stretchy="false">(</mo>
              <mi>n</mi>
              <mo>+</mo>
              <mi>A</mi>
              <mo>&#x2212;</mo>
              <mn>1</mn>
              <mo stretchy="false">)</mo>
              <mrow data-mjx-texclass="ORD">
                <mo>/</mo>
              </mrow>
              <mi>A</mi>
              <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
            </mrow>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </td>
</tr>
</table>

<div class="cblock">
<table id="tbl:rmdiv-good" class="table-data">
<caption align="bottom">
  <b>*@tbl:rmdiv-good</b>: 
  <i>m</i>= &#x230A;(256+2)/3&#x230B; = 0x56 を使った <i>x</i>/3。3、6、&hellip; でも正しい
</caption>
<tbody align="center">
<tr><th>x
  <th> 0 <th> 1 <th> 2 <th> 3 <th> 4 <th> 5 <th> 6 <th> 7
<tr><th>y=(x&middot;m)&gt;&gt;F
  <td> 0.00h <td> 0.56h <td> 0.ACh 
  <td> 1.02h <td> 1.58h <td> 1.AEh
  <td> 2.04h <td> 2.5Ah
<tr><th>本当の <i>x</i>/3
  <td> 0 <td> 0 <td> 0 <td> 1 <td> 1 <td> 1 <td> 2 <td> 2
</tbody>
</table>
</div>

そう、安全です。でも、いつまででしょうか？ いずれ、問題が起きる *x* の値に達します。今度は精度が関係します。幸い、*x* と *n* について、いつまずいことになりうるかをはっきり示す安全の限界を導けます。@eq:aprx-fail の誤差の条件の飛び跳ね方のせいで、本当の範囲は少し広いかもしれませんが、用心するに越したことはありません。導出は @eq:aprx-fail から始まり、@eq:int-div と、剰余についての技、つまり *p*%*q* ∈ \[0, *q*⟩ を使います。

<!--
\begin{matrix}
x \cdot m - n \left\lfloor x/A \right\rfloor & < & n &  \\
x \cdot m \cdot a - n \left\lfloor x/A \right\rfloor A & < & n \cdot A & \text{[insert } \left\lfloor x/A \right\rfloor A=x-x \text{%} A \text{]} \\
x \cdot m \cdot A - n \cdot x + n(x \text{%} A) & < & n \cdot A & \text{[insert } \max(x \text{%} A) = A - 1 \text{]} \\
x(m \cdot A - n) + n(A - 1) & < & n \cdot A &  \\
x(m \cdot A - n) & < & n & 
\end{matrix}
-->
<table id="eq:lims">
<tr>
  <td class="eqnrcell" rowspan=6>(!@eq:lims)</td>
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtable columnspacing="1em" rowspacing="4pt">
              <mtr>
                <mtd>
                  <mi>x</mi>
                  <mo>&#x22C5;</mo>
                  <mi>m</mi>
                  <mo>&#x2212;</mo>
                  <mi>n</mi>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">&#x230A;</mo>
                    <mi>x</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <mi>A</mi>
                    <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
                  </mrow>
                </mtd>
                <mtd>
                  <mo>&lt;</mo>
                </mtd>
                <mtd>
                  <mi>n</mi>
                </mtd>
                <mtd></mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>x</mi>
                  <mo>&#x22C5;</mo>
                  <mi>m</mi>
                  <mo>&#x22C5;</mo>
                  <mi>a</mi>
                  <mo>&#x2212;</mo>
                  <mi>n</mi>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">&#x230A;</mo>
                    <mi>x</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <mi>A</mi>
                    <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
                  </mrow>
                  <mi>A</mi>
                </mtd>
                <mtd>
                  <mo>&lt;</mo>
                </mtd>
                <mtd>
                  <mi>n</mi>
                  <mo>&#x22C5;</mo>
                  <mi>A</mi>
                </mtd>
                <mtd>
                  <mtext>[代入:&#xA0;</mtext>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">&#x230A;</mo>
                    <mi>x</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <mi>A</mi>
                    <mo data-mjx-texclass="CLOSE">&#x230B;</mo>
                  </mrow>
                  <mi>A</mi>
                  <mo>=</mo>
                  <mi>x</mi>
                  <mo>&#x2212;</mo>
                  <mi>x</mi>
                  <mo>%</mo>
                  <mi>A</mi>
                  <mtext>]</mtext>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>x</mi>
                  <mo>&#x22C5;</mo>
                  <mi>m</mi>
                  <mo>&#x22C5;</mo>
                  <mi>A</mi>
                  <mo>&#x2212;</mo>
                  <mi>n</mi>
                  <mo>&#x22C5;</mo>
                  <mi>x</mi>
                  <mo>+</mo>
                  <mi>n</mi>
                  <mo stretchy="false">(</mo>
                  <mi>x</mi>
                  <mo>%</mo>
                  <mi>A</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>&lt;</mo>
                </mtd>
                <mtd>
                  <mi>n</mi>
                  <mo>&#x22C5;</mo>
                  <mi>A</mi>
                </mtd>
                <mtd>
                  <mtext>[代入:&#xA0;</mtext>
                  <mo data-mjx-texclass="OP" movablelimits="true">max</mo>
                  <mo stretchy="false">(</mo>
                  <mi>x</mi>
                  <mo>%</mo>
                  <mi>A</mi>
                  <mo stretchy="false">)</mo>
                  <mo>=</mo>
                  <mi>A</mi>
                  <mo>&#x2212;</mo>
                  <mn>1</mn>
                  <mtext>]</mtext>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>x</mi>
                  <mo stretchy="false">(</mo>
                  <mi>m</mi>
                  <mo>&#x22C5;</mo>
                  <mi>A</mi>
                  <mo>&#x2212;</mo>
                  <mi>n</mi>
                  <mo stretchy="false">)</mo>
                  <mo>+</mo>
                  <mi>n</mi>
                  <mo stretchy="false">(</mo>
                  <mi>A</mi>
                  <mo>&#x2212;</mo>
                  <mn>1</mn>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>&lt;</mo>
                </mtd>
                <mtd>
                  <mi>n</mi>
                  <mo>&#x22C5;</mo>
                  <mi>A</mi>
                </mtd>
                <mtd></mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>x</mi>
                  <mo stretchy="false">(</mo>
                  <mi>m</mi>
                  <mo>&#x22C5;</mo>
                  <mi>A</mi>
                  <mo>&#x2212;</mo>
                  <mi>n</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>&lt;</mo>
                </mtd>
                <mtd>
                  <mi>n</mi>
                </mtd>
                <mtd></mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </td>
</tr>
</table>

この結果から、与えられた *A* と *n* に対する有効な *x* の最大値を簡単に計算できます。

<table id="eq:lim-x">
<tr>
  <td class="eqnrcell">(!@eq:lim-x)
  <td class="eqcell"><i>x</i> &lt; <i>n</i> / 
    (<i>m&middot;A</i> &minus; <i>n</i>)
</table>

*n* の下限は、(6) により max(*m·A*) = *n*+*A*−1 であることから導かれるので、次のようになります。

<table id="eq:lim-n">
<tr>
  <td class="eqnrcell">(!@eq:lim-n)
  <td class="eqcell"><i>n</i> &gt; <i>x</i>(<i>A</i>&minus;1)
</table>

基本的にはこれで全部です。もちろん、もう少し話はあります。掛け算をするので、積 *m·A* は変数に収まらなければなりません。そのため、実用的な数の限界は 16 ビットくらいになります。*A* の下位のゼロのビットをシフトで追い出すことで、この制限を少し緩められることもあります。たとえば *A*=10=5·2 なら、計算全体をする前に *x* を 1 回右シフトできます。360 でさえ 45·8 なので、この方法で 3 ビット節約できます。また、限界を超えても、まだ正しいか、わずかにずれるだけの可能性が十分にあることに注意してください（@eq:aprx-fail を確認してください）。そうなれば、比較的すぐに本当の答えを見つけられるはずです。

:::tip ARM の「int/const int」の割り算は常に安全

これで、GCC がなぜ 32 ビットの割り算を常に安全に最適化できるのかがわかります。32 ビットの *x* と *A* の最大値は、もちろん 2<sup>32</sup> です。これに対する安全の限界は 2<sup>64</sup>−2<sup>32</sup> で、これは `smull` の 64 ビットの結果に常に収まります。

:::

もちろん、これを毎回入力したくはないでしょう。そこで、仕事をしてくれる 2 つのマクロを示します。見た目はひどいですが、プリプロセッサとコンパイラはこれらの扱い方を知っています。これらをインライン関数に変換するのはおすすめしません。どういうわけか、コードがもたらすはずの利点をすべて失ってしまう可能性が高いからです。

```c
// Division by reciprocal multiplication
// a, and fp _must_ be constants

//! Get the reciprocal of \a a with \a fp fractional bits
#define FX_RECIPROCAL(a, fp)    ( ((1<<(fp))+(a)-1)/(a) )

//! Division of x/a by reciprocal multiplication
#define FX_RECIMUL(x, a, fp)    ( ((x)*((1<<(fp))+(a)-1)/(a))>>(fp) )
```

### まとめ {#ssec-rmdiv-sum}

これはちょっとした小技であり、*A* が定数のときに**だけ**うまくいくことを決して忘れないでください。そもそもの要点は割り算を実行時ではなくコンパイル時に行うことで、それは *A* が定数のときにしかできません。定数の良いところの 1 つは、定義上、前もってわかっていることです。必要なら、負の値や 2 の累乗もコンパイル時に解決できます。

丸め誤差の理由から、逆数の倍率 *m* は単なる &#x230A;*n*/*A*&#x230B; では*ありません*。常に切り上げてください。言い換えると次のとおりです。

> *m* = &#x230A;(*n*+*A*−1) / *A*&#x230B;

それから、割り算の失敗、つまり近似が「本当の」&#x230A;*x*/*A*&#x230B; と違う場合の問題があります。正確な条件は実はあまり重要ではありませんが、*x* の安全な範囲と、逆に与えられた *x* の範囲に必要な *n* を知っておくと役立ちます。ここでも、重要な項は定数なので、前もって求められます。下に示す関係は*ひとつの*限界を表すもので、*唯一の*限界ではないことに注意してください。実際に失敗する数はもう少し緩いかもしれませんが、状況によるので、そのための関係はもっと複雑になります。

> *x* \< *n* / (*m·a* − *n*)

> *n* \> *x*(*A*−1)

最後に、この節全体が何の話だったのかまったくわからなければ、この方法は使わないことをおすすめします。これは部分的に安全な割り算の最適化の技法で、普通の割り算よりかなり速くなりえますが、重要でない部分では使う価値がないかもしれません。とにかく、自分で判断してください。

:::note 別の方法

逆数による掛け算には別の方法もあります。*n*/*A* を切り上げる代わりに、*x* に 1 を足すこともできます。

> *y* = &#x230A;*x* / *A*&#x230B; ≈ (*x*+1) × &#x230A;*N* / *A* / *N*&#x230B;

これでも @tbl:rmdiv-bad で示した問題はなくなります。安全の条件は前とほぼ同じですが、負の *x* の割り算については少し違いがあります。*どうしても*知りたければ、ご要望に応じて詳細をお教えします。

:::

## ルックアップテーブル {#sec-lut}

<dfn>ルックアップテーブル</dfn>（<dfn>LUT</dfn>）とは、まあ、物を調べる（look up）のに使う表です。かなり明らかでしたね。重要なのは、それを本当に*すばやく*できるということです。簡単な例として、2<sup>5</sup> はいくつで、3<sup>5</sup> はいくつでしょうか？ 真のプログラマなら 1 つ目の答えは即座にわかる（はず）ですが、もう 1 つは見つけるのに少し時間がかかるかもしれません。なぜでしょうか？ 自尊心のあるプログラマなら誰でも、少なくとも 2<sup>10</sup> までは 2 の累乗を暗記しているからです。2 の累乗はプログラミングでしょっちゅう出てくるので、答えを覚えてしまっているのです。「2<sup>5</sup>」という問いを見たとたん、掛け算を繰り返して答えを計算するのではなく、頭が単に記憶からそれを**調べて**、ほとんど考えずに即座に答えます。（10 進数の）九九も同じです。7×8？ 56、一瞬です。でも、たとえば 3 の累乗や 16 進数の掛け算になると、この過程はうまくいかず、難しく長いやり方でしなければなりません。私が言いたいのは、きちんと計算しなければならないより、単に調べられるほうがずっと速くなりうるということです。

ルックアップテーブルの考え方はコンピュータにも使えます。そうでなければ話題にしませんでした。この場合、ルックアップテーブルは単に、調べる必要があると思うものを何でも詰め込んだ配列です。

### 例: サイン/コサインのルックアップテーブル {#ssec-lut-sin}

古典的な例は三角関数のルックアップテーブルです。サインとコサインは、特に GBA では高くつく演算なので、それらの表を作っておくのが一番です。そうすれば、2 回の（高くつく）型変換と浮動小数点の関数を経る代わりに、メモリアクセス 1 回で済みます。簡単なやり方は、それぞれたとえば 360 要素（1 度ごとに 1 つ）の FIXED の配列を 2 つ作り、ゲームの開始時に埋めることです。

```c
#define PI 3.14159265
#define DEGREES    360      // Full circle

FIXED sin_lut[DEGREES], cos_lut[DEGREES];

// A really simple (and slow and wasteful) LUT builder
void sincos_init()
{
    const double conv= 2*PI/DEGREES;
    for(int ii=0; ii<DEGREES; ii++)
    {
        sin_lut[ii]= (FIXED)(sin(conv*ii)*FIX_SCALEF);
        cos_lut[ii]= (FIXED)(cos(conv*ii)*FIX_SCALEF);
    }
}
```

しかし、この方法には深刻な欠陥があります。そう、動きはします。そう、簡単です。でも、改善の余地は間違いなくあります。この関数を使えばすぐにわかる問題から始めると、これは終わるまでに実際に数*秒*かかります。そう、標準の三角関数のルーチンはそれほど遅いのです。1 回呼ぶだけでいいので、これはかなり軽い問題ですが、それでも問題です。さらに、配列が定数でないので、IWRAM に置かれます。初期化のとき以外は実際には決して変わらないもののために、IWRAM の 10% が基本的に無駄になるのです。サインとコサインの対称性を使って計算時間を減らしたり、表を重ねたりと、これら 2 つの点を改善する方法はいくつかありますが、そもそもなぜゲームの中で計算するのでしょうか？ PC で表を前もって計算し、そのデータを配列に書き出すのも同じくらい簡単です。そうすれば表は定数になり（つまり IWRAM を独占しない）、GBA はその初期化に 1 サイクルも使わずに済みます。

2 つ目の改善は、より大きな固定小数点の小数部を使うことです。サインとコサインの値域は \[−1, +1\] です。つまり、ルックアップテーブルに 8.8 の固定小数点を使うと、より高い精度に使えたはずの 6 ビットを実は無駄にしていることになります。そこで、4.12 の固定小数点を使うことにします。そう、.14 の固定小数点まで上げることもできますが、12 のほうがきりのいい数です。

そして最後の改善として、円に 360 単位ではなく 2 の累乗、この場合は 512 を使います。これには 2 つの利点があります。

-   回り込み（α\<0 や α\>2π）に、if 文や、\*あえぎ\* 剰余の代わりにビットマスクを使えます。
-   コサインは単にずらしたサインで、1 つ目の点のおかげで、両方の波に表が 1 つあればよくなり、角度のオフセットとマスクによる回り込みを使って、一方の波からもう一方を得られます。

どちらの点も、物事をずっと楽にしてくれます。

念のため言っておくと、こうしてもまったく問題ありません。サインとコサインの形は、単位円の円周に沿って進むことから生じるもので、その道のりをいくつに分けるかは任意です。360 という数には歴史的な意味がありますが、それだけです。正直なところ、1 度がどれくらいかなんてどのみちわからないでしょう。重要なのは円の分割です。360° は 1 周、90° は 4 分の 1 周、という具合です。それが今度は、1 周が 512、4 分の 1 周が 128（512/4）、などになるのです。手っ取り早く雑なサインのルックアップテーブルの生成器は、次のようなものになるでしょう。まとめると次のとおりです。

-   ルックアップテーブルは GBA の外で前もって計算し、普通の const の配列のようにリンクする。
-   4.8 ではなく 4.12 の固定小数点を使う。
-   ルックアップテーブルは 360 ではなく 2 の累乗（512 など）に分割する。

```c
// Example sine LUT generator
#include <stdio.h>
#include <math.h>

#define M_PI 3.1415926535f
#define SIN_SIZE 512
#define SIN_FP 12

int main()
{
    int ii;
    FILE *fp= fopen("sinlut.c", "w");
    unsigned short hw;

    fprintf(fp, "//\n// Sine LUT; %d entries, %d fixeds\n//\n\n", 
        SIN_SIZE, SIN_FP);
    fprintf(fp, "const short sin_lut[%d]=\n{", SIN_SIZE);
    for(ii=0; ii<SIN_SIZE; ii++)
    {
        hw= (unsigned short)(sin(ii*2*M_PI/SIN_SIZE)*(1<<SIN_FP));
        if(ii%8 == 0)
            fputs("\n\t", fp);
        fprintf(fp, "0x%04X, ", hw);
    }
    fputs("\n};\n", fp);

    fclose(fp);
    return 0;
}
```

これは、`sin_lut` という 512 個のハーフワードの配列を含む sinlut.c というファイルを作ります。ここでは C のファイルを作っていますが、アセンブリのファイルに表を作ってもいいですし、あるいは何らかの方法でプロジェクトにリンクする単なるバイナリファイルとして作ることもできます。実際にサインとコサインの値を求めるには、`lu_sin()` 関数と `lu_cos()` 関数を通します。

```c
// Sine/cosine lookups. 
// NOTE: theta's range is [0, 0xFFFF] for [0,2π⟩, just like the 
// BIOS functions

//! Look-up a sine value
INLINE s32 lu_sin(u32 theta)
{   return sin_lut[(theta>>7)&0x1FF];   }

//! Look-up a cosine value
INLINE s32 lu_cos(u32 theta)
{   return sin_lut[((theta>>7)+128)&0x1FF]; }
```

#### excellut の紹介

libtonc のルックアップテーブルには、実は上に示した生成器は使っていません。代わりに、私自身の [excellut](http://www.coranac.com/projects/#excellut) を使っています。これはプログラムではなく、Excel のファイルです。そう、Excel と言いました。ルックアップテーブルを作るのに表計算のプログラムを使うことの良いところは、*どんな*種類の数学的な表でも作れ、望む精度があるかどうかを確かめ、グラフにしたりなんでもできることです。満足したら、表の一部を好きな方法で書き出すだけです。なんと柔軟でしょう？

#### 精度と分解能

これらは、ルックアップテーブルを作るときに考えるべき 2 つの主なことです。<dfn>精度</dfn>は各エントリの有効ビット数に関するもので、<dfn>分解能</dfn>は引数の空間で各エントリがどれだけ離れているかです。どちらも大きいほうが良いのですが、もちろん領域とのトレードオフがあります。妥協が必要で、ここでもまた、それで何をするつもりかに大きく左右されます。

精度については、関数の値域を考える必要があります。先ほど言ったように、サインの値域は \[−1, +1\] で、8.8 の固定小数点を使うと、より多くの有効ビットに使えたはずの 6 ビットを無駄にします。[最初のモード 7 の章](mode7.html)で使っているような割り算のルックアップテーブルでは、1/1 から 1/160 までが必要で、これは .8 の固定小数点ではうまく*いかない*ので、そこでは .16 の固定小数点を使っています。それでもまだ足りないかもしれませんが、もっと増やすとオーバーフローの問題が出るかもしれません。

2 つ目の問題、分解能は、エントリをいくつ持つかに結びついています。世界中の精度があっても、エントリがまばらに散らばりすぎていれば、あまり役に立ちません。画面の解像度と似ています。32 ビットカラーでも、320×240 の解像度の 17 インチのモニタしかなければ、ものすごく醜く見えるでしょう。一方で、それを支える精度がなければ、高すぎる分解能も役に立ちません。ルックアップテーブルにする価値のある関数のほとんどはなめらかな曲線で、どんな精度でも、分解能を上げてもルックアップテーブルに同じ値が加わるだけになる点に達します。それは領域の無駄です。それから、どうしても必要なら、いつでも補間できることを忘れないでください。

たとえば、私の 512 エントリ、8.8 の固定小数点のサインのルックアップテーブルの最初の数個の値は「0x0000、0x0003、0x0006、0x0009」です。ここは導関数が最大になるところなので、これが隣り合うエントリの間で見つかる最大の差です。分解能を 4 倍にすると、差は最後のビットになります。それ以上進めても、精度も上げない限り無意味です。

というわけで、これは実は三つ巴の妥協です。精度と分解能の間でバランスを取る必要があり（これを見つけるには関数の導関数が役立つでしょう）、その 2 つと対立するのが、ルックアップテーブルに割り当てたい ROM の領域の量です。ここでも、正しいバランスを判断できるのはあなただけです。

### ルックアップテーブルの線形補間 {#ssec-lut-lerp}

ルックアップテーブルは本質的に、関数から標本を取った点の集まりです。常にその点で配列にアクセスするならそれで問題ありませんが、点と点の間の値を取り出したい場合はどうでしょうか？ その例は、（コ）サインのインライン関数の `theta` のような固定小数点の角度です。普通は、固定小数点数の下位のビットは単に切り捨てられ、その前の点が使われます。速いのですが、その結果として精度が多少失われます。

<div class="cpt_fr" style="width:256px;">
  <img src="img/math/lutlerp.png" id="fig:lerp" 
    alt="lut lerp" width=256><br>
  <b>*@fig:lerp</b>: 直接の表引きと線形補間による
  サインの近似。
</div>

もっと正確な解決策は、周りの点を使って目的の点に補間することです。その中で最も簡単なのは<dfn>線形補間</dfn>（<dfn>lerp</dfn>）です。点 *x*<sub>a</sub> と *x*<sub>b</sub> があり、関数の値がそれぞれ *y*<sub>a</sub> と *y*<sub>b</sub> だとします。これで直線を定義できます。すると、点 *x* の関数の値は次の式で補間できます。

<table id="eq:lerp">
<tr>
  <td class="eqnrcell">(!@eq:lerp)
  <td class="eqcell">
    <i>y</i> = 
  <td class="eqcell">
	<table>
	  <tr><td class="bdrB"><i>y</i><sub>b</sub> &minus; <i>y</i><sub>a</sub>
	  <tr><td><i>x</i><sub>b</sub> &minus; <i>x</i><sub>a</sub>
	</table>
  <td class="eqcell">
    (<i>x</i> &minus; <i>x</i><sub>a</sub>) + <i>y</i><sub>a</sub>
</table>

*@fig:lerp は、線形補間がもたらしうる違いの例です。ここでは 16 点で標本を取った、.12f の精度のサイン関数があります。青い線は実際のサイン関数を表しています。マゼンタの線は前の点を使った直接の表引きで、線形補間は黄色い線で示しています。青と黄色の線はほとんど同じですが、マゼンタの線はかなりずれうることに注意してください。赤で示した *x* = 4.5 を考えてみてください。ルックアップテーブルの値は 8.5% ずれていますが、線形補間の値は 0.5% しかずれていません。16 倍良いのです！ たしかにこれは誇張した例ですが、線形補間は大きな違いを生みえます。

では、どう実装すればいいのでしょうか？ 基本的には @eq:lerp を使います。その中の割り算は厄介に見えるかもしれませんが、続く点どうしの差は常に 1、固定小数点数なら 2 の累乗であることを思い出してください。効率的な実装は次のようになるでしょう。

```c
//! Linear interpolator for 32bit LUTs.
/*! A LUT is essentially the discrete form of a function, f(\i x).
*   You can get values for non-integer \i x via (linear) 
*   interpolation between f(x) and f(x+1).
*   \param lut  The LUT to interpolate from.
*   \param x    Fixed point number to interpolate at.
*   \param shift    Number of fixed-point bits of \a x.
*/
INLINE int lu_lerp32(const s32 lut[], int x, const int shift)
{
    int xa, ya, yb;
    xa=x>>shift;
    ya= lut[xa]; yb= lut[xa+1];
    return ya + ((yb-ya)*(x-(xa<<shift))>>shift);
}
```

これは 32 ビットのルックアップテーブル用の版で、`lu_lerp16()` という 16 ビット版もあります。本体は同じで、宣言が違います。C++ なら、これは素敵なテンプレート関数になるでしょう。

これらの関数はどんな種類のルックアップテーブルにも使えますが、上端に小さな落とし穴があります。*N* エントリのルックアップテーブルがあるとします。関数は *x*+1 を使いますが、*N*−1 と *N* の間の最後の区間では、それはおそらく存在しません。そのため、その点で補間が大きく狂いかねません。それを特別な場合として扱う代わりに、ルックアップテーブルに点を 1 つ追加してください。`sinlut` は実際には 512 点ではなく 513 点あります。（実はワード境界に揃えるために 514 点ありますが、それは本題ではありません。）

:::warning 上端での線形補間

線形補間には *x* の上と下の標本点が必要で、これが上端で問題を起こすことがあります。いわば「柵を仕上げる」ために、そこに標本点を 1 つ追加してください。

:::

直接の表引きは 0 次の補間としても知られていて、線形補間は 1 次です。もっと高い次数もありますが、より多くの周りの点と、より多くの複雑な計算が必要になります。本当に、本当に必要な場合だけ試してください。

### 数学以外のルックアップテーブル {#ssec-lut-nomath}

ルックアップテーブルの最も明らかな使い道は、前もって計算した数学の関数ですが、ルックアップテーブルは数学に限りません。たとえば私の[テキストシステム](text.html)では、文字→タイルのインデックスの変換にルックアップテーブルを使っています。これで、そうでなければできなかったほど幅広くタイルを配置できます。デフォルトのフォントは ASCII 文字の 32-127 を使い、それらのタイルは普通タイル 0 から 95 にあります。でも、何らかの理由で数字のタイルだけが必要なら、文字のルックアップテーブルを数字だけに設定でき、あとはテキストシステムがそれに従います。残りのタイルはほかの目的に使えるようになります。

もう 1 つの使い道はフラグの表引きです。Visual C++ に付属するライブラリは、`isalpha()`、`isnum()` などの文字の種類のルーチンにルックアップテーブルを使っています。ビットフラグの表があり、ルーチンを使うと、適切な要素を取ってきてビットを調べるだけです。ゲームプログラミングでも似たようなものが見つかります。背景、歩ける、罠、などのタイルの種類のビットフラグの表です。巨大な switch ブロックの代わりに、配列の表引きだけで済むかもしれず、そのほうがずっと速いのです。
