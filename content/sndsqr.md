# 18. ピッ！ GBA の音の入門

<!-- toc -->

## GBA の音の入門 {#sec-intro}

グラフィックと操作のほかに、ゲームにとって大事な感覚がもう 1 つあります。音です。グラフィックが場面を作るとすれば、音は雰囲気を作ります。それはグラフィック以上に大事なこともあります。たとえば "Weird Al" Yankovic を流しながら *バイオハザード* を遊んでみてください。まるで合いません。雰囲気が台無しです。

GBA には 6 つのサウンドチャンネルがあります。最初の 4 つは、初代ゲームボーイにあったものとほぼ同じです。2 つの矩形波の生成器（チャンネル 1 と 2）、サンプルの再生器（チャンネル 3）、ノイズの生成器（チャンネル 4）です。これらは、ゲームボーイのコードネーム "Dot Matrix Game" にちなんで、DMG チャンネルとも呼ばれます。新しく加わったのが、2 つの Direct Sound チャンネル A と B です（DirectX の構成要素である Microsoft の DirectSound と混同しないでください）。これらは 8 ビットのデジタルのパルス符号変調（PCM）チャンネルです。

先に言っておくべきでしょうが、私は音のプログラミングについては本当にほとんど知りません。主な理由は、実際に曲を組み立てることができないからです（すでに頭の中で音楽が鳴っていると、それはなかなか難しいのです）。音のプログラミングを本格的に学びたいなら、ほとんど誰もがそこから情報を得ている [Belogic.com](http://www.belogic.com) と、サウンドミキサーの作り方を説明している [deku.gbadev.org](https://stuij.github.io/deku-sound-tutorial/) を見てください。どちらも素晴らしいサイトです。
<!-- as of 2023-09, belogic.com uses a self-signed certificate -->

音の作り方やプログラミングについてはあまり知らないかもしれませんが、音の本質は物質の中の波です。波は数学的な生き物で、数学のことなら私も少しは*わかります*。というわけで、矩形波の生成器について、ここではそういう方向の話をします。

## 音と波 {#sec-sndwav}

粒子の巨大な海を想像してみてください。すべての粒子は、小さなばねで隣の粒子とつながっています。さて、そのうちの 1 つを少し押してみましょう。押した方向でばねが縮んでから戻り、元の粒子を通常の位置に押し戻しつつ、押す力を隣の粒子に伝えます。それが次のばねを縮め、押す力を*その*隣の粒子に伝え、ということが延々と続きます。

これは波のふるまいの典型的な例です。あらゆる場合を網羅する正確な波の定義を与えるのは難しいのですが、本質的には、<dfn>波</dfn>とは<dfn>伝わっていく乱れ</dfn>のことです。波には多くの種類があります。主な 2 つの種類は、進む方向に振動する<dfn>縦</dfn>波と、進む方向に垂直に振動する<dfn>横</dfn>波です。周期的な（時間や空間で同じパターンを繰り返す）波もあれば、そうでない波もあります。進む波もあれば、進まない波もあります。

### 波 {#ssec-harmonic}

標準的な波は<dfn>調和波</dfn>です。これは、{@eq:wave} の解となるすべての関数 ψ(*x*) のことです。変数の名前は実はどうでもいいのですが、普通は空間（*x*、*y*、*z*）か時間（*t*）、あるいはその全部です。一般解は {@eq:wave-sols} にあります。というより解**たち**と言うべきかもしれません。書き表し方はたくさんあるからです。ただし、どれも同等で、今は関係のない技を使えば、ある形から別の形へ移れます。

<!--
\dv[2]{x}\psi(x) + k^2\psi(x) = 0
-->
<table id="eq:wave">
<tbody valign="middle">
<tr>
  <td class="eqnrcell">({!@eq:wave})
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mfrac>
              <msup>
                <mrow data-mjx-texclass="ORD">
                  <mi mathvariant="normal">d</mi>
                </mrow>
                <mrow data-mjx-texclass="ORD">
                  <mn>2</mn>
                </mrow>
              </msup>
              <mrow>
                <mrow data-mjx-texclass="ORD">
                  <mi mathvariant="normal">d</mi>
                </mrow>
                <msup>
                  <mi>x</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mn>2</mn>
                  </mrow>
                </msup>
              </mrow>
            </mfrac>
            <mi>&#x3C8;</mi>
            <mo stretchy="false">(</mo>
            <mi>x</mi>
            <mo stretchy="false">)</mo>
            <mo>+</mo>
            <msup>
              <mi>k</mi>
              <mn>2</mn>
            </msup>
            <mi>&#x3C8;</mi>
            <mo stretchy="false">(</mo>
            <mi>x</mi>
            <mo stretchy="false">)</mo>
            <mo>=</mo>
            <mn>0</mn>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</tbody>
</table>

一般解:

<!--
\begin{matrix}
\psi(x) & = & A \cdot cos(kx) + B \cdot sin(kx) \\
 & = & C \cdot e^{ikx} + D \cdot e^{-ikx} \\
 & = & E \cdot sin(kx + \varphi_0)
\end{matrix}
-->

<table id="eq:wave-sols">
<tr>
  <td class="eqnrcell">({!@eq:wave-sols})
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
                  <mi>&#x3C8;</mi>
                  <mo stretchy="false">(</mo>
                  <mi>x</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>A</mi>
                  <mo>&#x22C5;</mo>
                  <mi>c</mi>
                  <mi>o</mi>
                  <mi>s</mi>
                  <mo stretchy="false">(</mo>
                  <mi>k</mi>
                  <mi>x</mi>
                  <mo stretchy="false">)</mo>
                  <mo>+</mo>
                  <mi>B</mi>
                  <mo>&#x22C5;</mo>
                  <mi>s</mi>
                  <mi>i</mi>
                  <mi>n</mi>
                  <mo stretchy="false">(</mo>
                  <mi>k</mi>
                  <mi>x</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>C</mi>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mi>e</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>i</mi>
                      <mi>k</mi>
                      <mi>x</mi>
                    </mrow>
                  </msup>
                  <mo>+</mo>
                  <mi>D</mi>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mi>e</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mo>&#x2212;</mo>
                      <mi>i</mi>
                      <mi>k</mi>
                      <mi>x</mi>
                    </mrow>
                  </msup>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>E</mi>
                  <mo>&#x22C5;</mo>
                  <mi>s</mi>
                  <mi>i</mi>
                  <mi>n</mi>
                  <mo stretchy="false">(</mo>
                  <mi>k</mi>
                  <mi>x</mi>
                  <mo>+</mo>
                  <msub>
                    <mi>&#x3C6;</mi>
                    <mn>0</mn>
                  </msub>
                  <mo stretchy="false">)</mo>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

<div class="cpt_fr" style="width:212px;">
<img src="./img/wave.png" id="fig:wave" alt="it's a sine wave">

**{*@fig:wave}**: 調和波
</div>

完全な波は 3 つのもので表せます。1 つ目は<dfn>振幅</dfn> *A* で、最小値と最大値の間の距離の半分です。2 つ目は<dfn>波長</dfn> λ で、波が繰り返すまでの長さです（これは波数 *k*= 2π/λ と結びついています）。それから、始まりの点を決める<dfn>初期位相</dfn> φ<sub>0</sub> があります。時間についての波なら、波長の代わりに<dfn>周期</dfn> *T* と<dfn>周波数</dfn> *f*=1/*T*（と角周波数 ω= 2π*f*= 2π/*T*）を使います。これらのパラメータがそれぞれ何を表すかは {@fig:wave} で見られます。

波動方程式の面白い点の 1 つは、それが ψ に対する線形の演算だということです。つまり、解をどう組み合わせても、やはり解になるということです。これが<dfn>重ね合わせの原理</dfn>です。たとえば、2 つの波 ψ<sub>1</sub> と ψ<sub>2</sub> があれば、Ψ = *a*ψ<sub>1</sub> + *b*ψ<sub>2</sub> も波です。当たり前のことに聞こえるかもしれませんが、そうではありません。非線形の方程式（そういうものもあります）には科学者たちが少し身をすくめがちだという事実が、線形の方程式の価値について何かを物語っているはずです。

### 音波 {#ssec-wave-sound}

音も波です。実際、音は物質の中の縦方向の圧力の波で、先ほどのばねでつながった粒子の仕組みとほぼ同じように、分子の集まり全体が行ったり来たりして伝わります。原理的には空間と時間の両方の構造を持っていて、すべてを扱おうとするとおそろしく複雑になります。でもここでは簡単にして、2 つの部分だけを考えます。振幅 *A* と、周期と周波数 *T* と *f* です。ご存じかもしれませんが、音の高さは周波数に関係しています。人間の耳は 20 Hz から 20 kHz の範囲を聞くことができ、周波数が高いほど（つまり、波が詰まっているほど）、音は高くなります。ほとんどの音は、実際には振幅と周波数の異なるさまざまな波の寄せ集めです。重ね合わせの原理が働いているのです。面白いのは、それらの成分をすべて 1 つの関数に足し合わせてプロットすると、もうまったくサイン波には見えないことです。さらに面白いのは、その過程を逆にたどることもできるということです。関数（*どんな*関数でも）を取ってきて、サイン波とコサイン波の重ね合わせに分解し、その音にどんな周波数が含まれているかを見られるのです。これはフーリエ変換と呼ばれ、すぐあとで扱います。

### 音階 {#ssec-notes}

20 Hz から 20 kHz の範囲全体が聞こえるとはいえ、音楽に使われるのは飛び飛びの周波数の集まりだけです。ここで<dfn>音階</dfn>という考え方が出てきます。その中心にあるのが<dfn>オクターブ</dfn>で、周波数が 2 倍になることを表します。各オクターブは、いくつかの異なる音に分けられます。西洋の体系では 12 で、A から G までの名前が付いていますが、なぜかオクターブの番号は C から始まります。オクターブ 0 は<dfn>中央の C</dfn>（ド）から始まり、その周波数は約 262 Hz です（{@tbl:oct0} も見てください）。そして、ええ、A から G までには 7 つの文字しかないことは知っています。ほかの音は、それらの音の間にあるフラットやシャープです。「12」は 1 オクターブの中の半音の数を指しています。音階は**対数的**で、半音ごとに 2<sup>1/12</sup> 倍ずつ離れています。まあ、ほぼそうなのですが、なぜか*ぴったり*当てはまらない音もあります。

<div class="cblock">
<table id="tbl:oct0" class="table-data">
<caption align="bottom">
  <b>{*@tbl:oct0}</b>: オクターブ 0 の
  音と周波数
</caption>
<tbody align="center">
<tr>
  <th> 半音
  <th> 0 <th> 1 <th> 2 <th> 3 <th> 4 <th> 5
  <th> 6 <th> 7 <th> 8 <th> 9 <th>10 <th>11 
  <th> (12) 
<tr>
  <th> 名前
  <td> C <td> C# <td> D <td> D# <td> E <td> F 
  <td> F# <td> G <td> G# <td> A <td> A# <td> B 
  <td> (C)
<tr>
  <th> 周波数 (Hz)
  <td> 261.7 <td> 277.2 <td> 293.7 <td> 311.2 <td> 329.7 <td> 349.3 
  <td> 370.0 <td> 392.0 <td> 415.3 <td> 440.0 <td> 466.2 <td> 493.9
  <td> (523.3)
</tbody>
</table>
</div>

### フーリエ変換と矩形波 {#ssec-fourier}

フーリエ変換は、時間領域の関数を、<dfn>スペクトル</dfn>と呼ばれる周波数の分布として表す方法です。また、教授たちが自然科学を学ぶ若い学生を震え上がらせる、数多くの方法の 1 つでもあります。心配いりません。きっとこの節は無傷で切り抜けられますよ <kbd>\>:)</kbd>。ほどほどに、あるいはそこそこに素直な関数なら、多項式や指数関数、そして波のような、*とても*素直な関数の級数として書き直せます。たとえば、フーリエ級数として書くと、関数は {@eq:fser} のようになります。

<!--
f(x) = \frac{1}{2}A_0 + \sum_{n>0}A_m\cos(m{\omega}t) + \sum_{n>0}B_m\sin(m{\omega}t)
-->
<table id="eq:fser">
<tr>
  <td class="eqnrcell">({!@eq:fser})
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>f</mi>
            <mo stretchy="false">(</mo>
            <mi>x</mi>
            <mo stretchy="false">)</mo>
            <mo>=</mo>
            <mfrac>
              <mn>1</mn>
              <mn>2</mn>
            </mfrac>
            <msub>
              <mi>A</mi>
              <mn>0</mn>
            </msub>
            <mo>+</mo>
            <munder>
              <mo data-mjx-texclass="OP">&#x2211;</mo>
              <mrow data-mjx-texclass="ORD">
                <mi>n</mi>
                <mo>&gt;</mo>
                <mn>0</mn>
              </mrow>
            </munder>
            <msub>
              <mi>A</mi>
              <mi>m</mi>
            </msub>
            <mi>cos</mi>
            <mo data-mjx-texclass="NONE">&#x2061;</mo>
            <mrow>
              <mo data-mjx-texclass="OPEN">(</mo>
              <mi>m</mi>
              <mrow data-mjx-texclass="ORD">
                <mi>&#x3C9;</mi>
              </mrow>
              <mi>t</mi>
              <mo data-mjx-texclass="CLOSE">)</mo>
            </mrow>
            <mo>+</mo>
            <munder>
              <mo data-mjx-texclass="OP">&#x2211;</mo>
              <mrow data-mjx-texclass="ORD">
                <mi>n</mi>
                <mo>&gt;</mo>
                <mn>0</mn>
              </mrow>
            </munder>
            <msub>
              <mi>B</mi>
              <mi>m</mi>
            </msub>
            <mi>sin</mi>
            <mo data-mjx-texclass="NONE">&#x2061;</mo>
            <mrow>
              <mo data-mjx-texclass="OPEN">(</mo>
              <mi>m</mi>
              <mrow data-mjx-texclass="ORD">
                <mi>&#x3C9;</mi>
              </mrow>
              <mi>t</mi>
              <mo data-mjx-texclass="CLOSE">)</mo>
            </mrow>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

もちろん、これはすべて係数 *A*<sub>m</sub> と *B*<sub>m</sub> を求められるかどうかにかかっています。それらの式を導くのはかなり素直なのですが、それは読者への演習として残しておき、結果だけを {@eq:ftrans} の形で示します。実は、フーリエ変換の定義の仕方はいくつかあることも言っておくべきでしょう。たとえば、\[0,*T*\] ではなく \[−½*T*, ½*T*\] で積分するものや、サインとコサインの代わりに複素指数関数を使うものがありますが、結局はどれも同じことをしています。

<!--
\begin{matrix}
A_m & = & \frac{2}{T}\int_{0}^{T} f(t)\cos(m{\omega}t) dt \\
B_m & = & \frac{2}{T}\int_{0}^{T} f(t)\sin(m{\omega}t) dt
\end{matrix}
-->
<table id="eq:ftrans">
<tr>
  <td class="eqnrcell">({!@eq:ftrans})
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
                  <msub>
                    <mi>A</mi>
                    <mi>m</mi>
                  </msub>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mfrac>
                    <mn>2</mn>
                    <mi>T</mi>
                  </mfrac>
                  <msubsup>
                    <mo data-mjx-texclass="OP">&#x222B;</mo>
                    <mrow data-mjx-texclass="ORD">
                      <mn>0</mn>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mi>T</mi>
                    </mrow>
                  </msubsup>
                  <mi>f</mi>
                  <mo stretchy="false">(</mo>
                  <mi>t</mi>
                  <mo stretchy="false">)</mo>
                  <mi>cos</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mrow>
                    <mo data-mjx-texclass="OPEN">(</mo>
                    <mi>m</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>&#x3C9;</mi>
                    </mrow>
                    <mi>t</mi>
                    <mo data-mjx-texclass="CLOSE">)</mo>
                  </mrow>
                  <mi>d</mi>
                  <mi>t</mi>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <msub>
                    <mi>B</mi>
                    <mi>m</mi>
                  </msub>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mfrac>
                    <mn>2</mn>
                    <mi>T</mi>
                  </mfrac>
                  <msubsup>
                    <mo data-mjx-texclass="OP">&#x222B;</mo>
                    <mrow data-mjx-texclass="ORD">
                      <mn>0</mn>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mi>T</mi>
                    </mrow>
                  </msubsup>
                  <mi>f</mi>
                  <mo stretchy="false">(</mo>
                  <mi>t</mi>
                  <mo stretchy="false">)</mo>
                  <mi>sin</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mrow>
                    <mo data-mjx-texclass="OPEN">(</mo>
                    <mi>m</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>&#x3C9;</mi>
                    </mrow>
                    <mi>t</mi>
                    <mo data-mjx-texclass="CLOSE">)</mo>
                  </mrow>
                  <mi>d</mi>
                  <mi>t</mi>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </tbody>
  </table>
</table>

<div class="cpt_fr" style="width:212px;">
<img src="./img/sqrwave.png" id="fig:sqrwave" alt="sharp transitions between flat crest and flat trough"><br>

**{*@fig:sqrwave}**: 矩形波
</div>

例として、{@fig:sqrwave} に示す矩形波を見てみましょう。矩形波は、ある時間（パラメータ *h*）だけオン（1）になり、残りの周期の間はオフ（0）になります。周期的な波であることに変わりはないので、*t* 軸のどこに置くかは実はどうでもいいことです。便宜上、山を中心に置きました。こうすると対称な波になり、反対称なサイン波が*すべて*消えるという素敵な性質が得られます。*A*<sub>0</sub>=*h*/*T* です。これは関数の平均だからです。残りの *A*<sub>m</sub> は {@eq:ftrans} から求まります。

<!--
A_m = \frac{2}{\pi} \cdot \frac{\sin({\pi}mh/T)}{m} = \frac{2T}{h} \cdot \frac{\sin({\pi}h/T \cdot m)}{{\pi}h/T \cdot m}
-->
<table id="eq:fsqr">
<tr>
  <td class="eqnrcell">({!@eq:fsqr})
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <msub>
              <mi>A</mi>
              <mi>m</mi>
            </msub>
            <mo>=</mo>
            <mfrac>
              <mn>2</mn>
              <mi>&#x3C0;</mi>
            </mfrac>
            <mo>&#x22C5;</mo>
            <mfrac>
              <mrow>
                <mi>sin</mi>
                <mo data-mjx-texclass="NONE">&#x2061;</mo>
                <mrow>
                  <mo data-mjx-texclass="OPEN">(</mo>
                  <mrow data-mjx-texclass="ORD">
                    <mi>&#x3C0;</mi>
                  </mrow>
                  <mi>m</mi>
                  <mi>h</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>T</mi>
                  <mo data-mjx-texclass="CLOSE">)</mo>
                </mrow>
              </mrow>
              <mi>m</mi>
            </mfrac>
            <mo>=</mo>
            <mfrac>
              <mrow>
                <mn>2</mn>
                <mi>T</mi>
              </mrow>
              <mi>h</mi>
            </mfrac>
            <mo>&#x22C5;</mo>
            <mfrac>
              <mrow>
                <mi>sin</mi>
                <mo data-mjx-texclass="NONE">&#x2061;</mo>
                <mrow>
                  <mo data-mjx-texclass="OPEN">(</mo>
                  <mrow data-mjx-texclass="ORD">
                    <mi>&#x3C0;</mi>
                  </mrow>
                  <mi>h</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>T</mi>
                  <mo>&#x22C5;</mo>
                  <mi>m</mi>
                  <mo data-mjx-texclass="CLOSE">)</mo>
                </mrow>
              </mrow>
              <mrow>
                <mrow data-mjx-texclass="ORD">
                  <mi>&#x3C0;</mi>
                </mrow>
                <mi>h</mi>
                <mrow data-mjx-texclass="ORD">
                  <mo>/</mo>
                </mrow>
                <mi>T</mi>
                <mo>&#x22C5;</mo>
                <mi>m</mi>
              </mrow>
            </mfrac>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </tbody>
  </table>
</table>

*A*<sub>m</sub> は <dfn>sinc</dfn> 関数、つまり sin(*x*)/*x* です。*m* が大きいとゼロに近づきます（高次の項は相対的に重要でなくなるはずなので、当然です）。しかし面白いのは、高次の項の中には、サインのせいで*消えてしまう*ものもあることです。これは、*m* が *T/h* の倍数になるたびに起きます。

## GBA の音 {#sec-gbasnd}

### サウンドのレジスタ {#ssec-snd-regs}

グラフィックでは、結果を得るのに 2 つのレジスタ（`REG_DISPCNT` と `REG_BGxCNT`）を扱うだけで済みました。音では、*何か*を得るまでに、たくさんのレジスタを扱わなければなりません。DMG のチャンネルにはそれぞれ 2 つか 3 つのレジスタがあり、似た機能を持つものもあれば、そうでないものもあります。それとは別に、全体を制御するレジスタが 4 つあります。

レジスタの名前は、音に関しては特に混乱しているようです。見かける名前には基本的に 2 つの系統があります。1 つは `REG_SOUNDxCNT` のあとに、かなり行き当たりばったりに `_L`、`_H`、`_X` が付くもの。もう 1 つは `REG_SGxy` と `REG_SGCNTy` という構成のもの（*x*=1、2、3、4、*y*=0、1）です。前者のほうが新しいのだと思いますが、古いほうが一貫性があるのはおかしな話です。まあいいでしょう。いずれにせよ、私はどちらもあまりわかりやすいとは思えず、L/H/X や 0/1 のどれが何をするのかをいつも忘れてしまうので、[tepples](https://pineight.com/gba/) の pin8gba.h にある名前をもとにした*3 つ目*の名前の系統を使っています。私見ですが、ほかの 2 つより筋が通っています。

<div class="cblock">
<table id="tbl:snd-names" class="table-data">
<caption align="bottom">
  <b>{*@tbl:snd-names}</b>: サウンドのレジスタの名前。
</caption>
<tr align="center">
  <th> オフセット	<th> 機能 
  <th> 旧		<th> 新			<th> tonc
<tr>
  <th> 60h	<td> チャンネル 1（矩形波）スイープ
  <td rowspan=2> REG_SG10	<td> SOUND1CNT_L	<td> REG_SND1SWEEP	
<tr>
  <th> 62h	<td> チャンネル 1（矩形波）長さ、デューティ、エンベロープ
  <!-- -->		<td> SOUND1CNT_H	<td> REG_SND1CNT
<tr>
  <th> 64h	<td> チャンネル 1（矩形波）周波数、オン
  <td> REG_SG11	<td> SOUND1CNT_X	<td> REG_SND1FREQ
<tr>
  <th> 68h		<td> チャンネル 2（矩形波）長さ、デューティ、エンベロープ
  <td> REG_SG20	<td> SOUND2CNT_L	<td> REG_SND2CNT
<tr>
  <th> 6Ch		<td> チャンネル 2（矩形波）周波数、オン
  <td> REG_SG21	<td> SOUND2CNT_H	<td> REG_SND2FREQ
<tr>
  <th> 70h		<td> チャンネル 3（波形）モード
  <td rowspan=2> REG_SG30	<td> SOUND3CNT_L	<td> REG_SND3SEL
<tr>
  <th> 72h		<td> チャンネル 3（波形）長さ、音量
  <!-- -->		<td> SOUND3CNT_H	<td> REG_SND3CNT
<tr>
  <th> 74h		<td> チャンネル 3（波形）周波数、オン
  <td> REG_SG31	<td> SOUND3CNT_X	<td> REG_SND3FREQ
<tr>
  <th> 78h		<td> チャンネル 4（ノイズ）長さ、音量、エンベロープ
  <td> REG_SG40	<td> SOUND4CNT_L	<td> REG_SND4CNT
<tr>
  <th> 7Ch		<td> チャンネル 4（ノイズ）周波数、オン
  <td> REG_SG41	<td> SOUND4CNT_H	<td> REG_SND4FREQ	
<tr>
  <th> 80h	<td> DMG のマスター制御
  <td rowspan=2> REG_SGCNT0	<td> SOUNDCNT_L	<td> REG_SNDDMGCNT
<tr>
  <th> 82h	<td> DSound のマスター制御	
  <!-- -->		<td> SOUNDCNT_H	<td> REG_SNDDSCNT
<tr>
  <th> 84h	<td> サウンドの状態	
  <td> REG_SGCNT1	<td> SOUNDCNT_X	<td> REG_SNDSTAT
<tr>
  <th> 88h	<td> バイアスの制御
  <td> REG_SGBIAS	<td> SOUNDBIAS	<td> REG_SNDBIAS
</table>
</div>

「あーあ。これって例の『tegel』みたいなやつでしょ？ *自分*では素敵で一味違うものを手に入れたつもりでいて、あとになって世間に合わせて標準の用語に戻すっていう。でしょ？」

いいえ、この名前を使い続けます。たぶん。そうだといいのですが。……正直に言うと、本当にわかりません <kbd>:P</kbd>。でも、これは大したことではありません。いくつかの define や検索と置換で、名前は簡単に切り替えられます。ともかく、`REG_SNDxFREQ` には周波数の情報が、`REG_SNDxCNT` には音量やエンベロープの設定などが入っています。場合によっては、ビットの配置がまったく同じこともあります。チャンネル 1 は、スイープの機能を除けば、チャンネル 2 とまったく同じです。

### マスターサウンドのレジスタ {#ssec-snd-mstr}

`REG_SNDDMGCNT`、`REG_SNDDSCNT`、`REG_SNDSTAT` は、マスターのサウンド制御です。何かを動かすには、それぞれに少なくともいくつかのビットを設定する必要があります。

<div class="reg">
<table class="table-reg">
<caption class="reg">
  REG_SNDDMGCNT (SOUNDCNT_L / SGCNT0_L ) @ 0400:0080h
</caption>
<tr class="bits">
  <td>F<td>E<td>D<td>C<td>B<td>A<td>9<td>8
  <td>7<td>6 5 4<td>3<td>2 1 0
<tr class="bf">
  <td class="rclr1">R4
  <td class="rclr1">R3
  <td class="rclr1">R2
  <td class="rclr1">R1
  <td class="rclr0">L4
  <td class="rclr0">L3
  <td class="rclr0">L2
  <td class="rclr0">L1
  <td> -
  <td class="rclr3">RV
  <td> -
  <td class="rclr2">LV
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width=128>
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-2<td class="rclr2">LV
  <td> &nbsp;
  <td> 左の音量
<tr class="bg1">	
  <td>4-6<td class="rclr3">RV
  <td> &nbsp;
  <td> 右の音量
<tr class="bg0">	
  <td>8-B<td class="rclr0">L1-L4
  <td>SDMG_LSQR1, SDMG_LSQR2, SDMG_LWAVE, SDMG_LNOISE
  <td>左のチャンネル 1-4
<tr class="bg1">	
  <td>C-F<td class="rclr1">R1-R4
  <td>SDMG_RSQR1, SDMG_RSQR2, SDMG_RWAVE, SDMG_RNOISE
  <td>右のチャンネル 1-4
</tbody>
</table>
</div>

`REG_SNDDMGCNT` は、DMG チャンネルの主音量と、どのチャンネルを有効にするかを制御します。これらの制御は左右のスピーカーで別々です。以下は、このレジスタを扱いやすくする 2 つのマクロです。これらは実際にはレジスタを設定*せず*、フラグを組み合わせるだけであることに注意してください。

```c
#define SDMG_SQR1    0x01
#define SDMG_SQR2    0x02
#define SDMG_WAVE    0x04
#define SDMG_NOISE   0x08

#define SDMG_BUILD(_lmode, _rmode, _lvol, _rvol)    \
    ( ((_lvol)&7) | (((_rvol)&7)<<4) | ((_lmode)<<8) | ((_rmode)<<12) )

#define SDMG_BUILD_LR(_mode, _vol) SDMG_BUILD(_mode, _mode, _vol, _vol)
```

<div class="reg">
<table class="table-reg">
<caption class="reg">
  REG_SNDDSCNT (SOUNDCNT_H / SGCNT0_H) @ 0400:0082h
</caption>
<tr class="bits">
  <td>F<td>E<td>D<td>C<td>B<td>A<td>9<td>8
  <td>7 6 5 4<td>3<td>2<td> 1 0
<tr class="bf">
  <td class="rclr4">BF
  <td class="rclr3">BT
  <td class="rclr2">BL
  <td class="rclr2">BR
  <td class="rclr4">AF
  <td class="rclr3">AT
  <td class="rclr2">AL
  <td class="rclr2">AR
  <td> - 
  <td class="rclr1">BV
  <td class="rclr1">AV
  <td class="rclr0">DMGV
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width=128>
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-1<td class="rclr0">DMGV
  <td>SDS_DMG25, SDS_DMG50, SDS_DMG100
  <td>DMG の音量の比率。
    <ul>
      <li><b>00</b>: 25%
      <li><b>01</b>: 50%
      <li><b>10</b>: 100%
      <li><b>11</b>: 使用禁止
    </ul>
<tr class="bg1">	
  <td> 2 <td class="rclr1">AV
  <td>SDS_A50, SDS_A100
  <td>DSound A の音量の比率。クリアなら 50%、セットなら 100%
<tr class="bg0">	
  <td> 3 <td class="rclr1">BV
  <td>SDS_B50, SDS_B100
  <td>DSound B の音量の比率。クリアなら 50%、セットなら 100%
<tr class="bg1">	
  <td>8-9<td class="rclr2">AR, AL
  <td>SDS_AR, SDS_AL
  <td><B>DSound A の有効化</b>。右と左のスピーカーで DS A を有効にする
<tr class="bg0">	
  <td> A <td class="rclr3">AT
  <td>SDS_ATMR0, SDS_ATMR1
  <td><b>Dsound A のタイマー</B>。DS A にタイマー 0（クリア時）か
    1（セット時）を使う
<tr class="bg1">	
  <td> B <td class="rclr4">AF
  <td>SDS_ARESET
  <td><b>Dsound A の FIFO のリセット</b>。Direct Sound に DMA を使うとき、
    これを使うと、DMA は FIFO バッファを使ったあとでリセットします。
<tr class="bg0">	
  <td>C-F
  <td>
    <span class="rclr2">BR, BL</span>, 
    <span class="rclr3">BT</span>, 
    <span class="rclr4">BF</span>
  <td>SDS_BR, SDS_BL, SDS_BTMR0, SDS_BTMR1, SDS_BRESET
  <td>ビット 8-B と同じだが、DSound B 用
</tbody>
</table>
</div>

`REG_SNDDSCNT` については、PCM の音を制御するものの、なぜか DMG の音のビットもいくつか持っているということ以外、あまり知りません。`REG_SNDSTAT` は DMG チャンネルの状態を示し、*さらに*すべての音を有効にします。何か音を出したいなら、そこでビット 7 を設定する必要があります。

<div class="reg">
<table class="table-reg">
<caption class="reg">
  REG_SNDSTAT (SOUNDCNT_X / SGCNT1) @ 0400:0084h
</caption>
<tr class="bits">
  <td>F E D C B A 9 8
  <td>7<td>6 5 4
  <td class="rof">3<td class="rof">2<td class="rof">1<td class="rof">0
<tr class="bf">
  <td> -
  <td class="rclr0">MSE
  <td> -
  <td class="rclr1">4A
  <td class="rclr1">3A
  <td class="rclr1">2A
  <td class="rclr1">1A
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width=128>
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td class="rof">0-3<td class="rclr1">1A-4A
  <td>SSTAT_SQR1, SSTAT_SQR2, SSTAT_WAVE, SSTAT_NOISE
  <td><b>動作中のチャンネル</b>。どの DMG チャンネルが現在
    鳴っているかを示します。チャンネルを有効にするものでは<i>ありません</i>。
    それは <code>REG_SNDDMGCNT</code> の役目です。
<tr class="bg1">	
  <td> 7 <td class="rclr0">MSE
  <td>SSTAT_DISABLE, SSTAT_ENABLE
  <td><b>マスターサウンドの有効化</b>。何か音を聞きたいなら、
    必ず設定しなければなりません。ほかの何よりも<b>先に</b>設定してください。
    そうしないと、ほかのレジスタにアクセスできません。詳しくは GBATEK を
    見てください。
</tbody>
</table>
</div>

:::warning サウンドのレジスタへのアクセス

エミュレータでは、音が無効（`REG_SNDSTAT`\{7\} がクリア）でもサウンドのレジスタにアクセスできることがありますが、実機ではできません。使う前に必ず音を有効にしてください。

:::

### GBA の矩形波の生成器 {#ssec-snd-sqr}

GBA には、矩形波の音の生成器が 2 つあります。チャンネル 1 と 2 です。両者の唯一の違いは、チャンネル 1 の<dfn>周波数スイープ</dfn>です。これを使うと、鳴っている間に周波数を指数関数的に上げたり下げたりできます。それはすべて `REG_SND1SWEEP` で行います。`REG_SNDxCNT` は、波の長さ、エンベロープ、デューティ比を制御します。長さは明らかでしょう。<dfn>エンベロープ</dfn>は、基本的には時間の関数としての振幅です。フェードインさせたり（<dfn>アタック</dfn>）、同じレベルに保ったり（<dfn>サステイン</dfn>）、またフェードアウトさせたり（<dfn>ディケイ</dfn>）できます。エンベロープには 16 段階の音量があり、開始時の音量、エンベロープの方向、次の変化までの時間を制御できます。音量は線形で、12 は 6 の 2 倍の振幅になります。<dfn>デューティ</dfn>は、「オン」の時間と周期の比、言い換えると *D* = *h/T* のことです。

もちろん、周波数も制御できます。`REG_SNDxFREQ` を使います。ただし、このフィールドに入れるのは周波数ではありません。周期でもありません。私が<dfn>レート</dfn> *R* と呼ぶものです。この 3 つの量は関係していますが、微妙に違っていて、混同すると大混乱になります。そして資料ではよく混同*されている*ので、気をつけてください。周波数 *f* とレート *R* の関係は {@eq:fvsr} で表されます。レートが上がれば、周波数も上がります。*R* ∈ \[0, 2047\] なので、周波数の範囲は \[64 Hz, 131 kHz\] です。これは 10 オクターブにわたりますが、一番高いあたりは、周波数の刻みが大きくなりすぎる（{@eq:fvsr} の式の分母が 0 に近づく）ので、あまり役に立ちません。

<!--
f(R) = \frac{2^{17}}{2048-R}
R(f) = 2048 - \frac{2^{17}}{f}
-->

<table id="eq:fvsr">
<tr>
  <td class="eqnrcell">({!@eq:fvsr}a)
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>f</mi>
            <mo stretchy="false">(</mo>
            <mi>R</mi>
            <mo stretchy="false">)</mo>
            <mo>=</mo>
            <mfrac>
              <msup>
                <mn>2</mn>
                <mrow data-mjx-texclass="ORD">
                  <mn>17</mn>
                </mrow>
              </msup>
              <mrow>
                <mn>2048</mn>
                <mo>&#x2212;</mo>
                <mi>R</mi>
              </mrow>
            </mfrac>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math><tr>
  <td class="eqnrcell">({!@eq:fvsr}b)
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>R</mi>
            <mo stretchy="false">(</mo>
            <mi>f</mi>
            <mo stretchy="false">)</mo>
            <mo>=</mo>
            <mn>2048</mn>
            <mo>&#x2212;</mo>
            <mfrac>
              <msup>
                <mn>2</mn>
                <mrow data-mjx-texclass="ORD">
                  <mn>17</mn>
                </mrow>
              </msup>
              <mi>f</mi>
            </mfrac>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

### 矩形波のサウンドのレジスタ {#ssec-snd-sqrreg}

どちらの矩形波の生成器にも、エンベロープ、長さ、デューティを制御する `REG_SNDxCNT` と、周波数を制御する `REG_SNDxFREQ` のレジスタがあります。サウンド 1 には、さらに `REG_SND1SWEEP` というスイープの制御もあります。伝統的な名前は {@tbl:snd-names} を見てください。伝統的な名前では、制御と周波数のレジスタの接尾辞が、チャンネル 1 と 2 でまったく同じ機能なのに*違っている*ことに注意してください。

<div class="reg">
<table class="table-reg" id="tbl-reg-snd1cnt" width=420>
<caption class="reg">
<span class="nobr">
  REG_SND1CNT (SOUND1CNT_H / SG10_H) @ 0400:0062h</span>
  <br> and <br>
<span class="nobr">
  REG_SND2CNT (SOUND2CNT_L / SG20_L) @ 0400:0068h</span>
</caption>
<tr class="bits">
  <td>F E D C<td>B<td>A 9 8<td>7 6<td class="wof">5 4 3 2 1 0
<tr class="bf">
  <td class="rclr0">EIV
  <td class="rclr1">ED
  <td class="rclr2">EST
  <td class="rclr3">D
  <td class="rclr4">L
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td class="wof">0-5<td class="rclr4">L
  <td>SSQR_LEN#
  <td>音の<b>長さ</b>。これは<i>書き込み専用</i>のフィールドで、
    チャンネルが時間指定になっている（<code>REG_SNDxFREQ{E}</code>）
    ときにだけ働きます。長さそのものは、実際には (64&minus;<i>L</i>)/256 秒で、
    範囲は [3.9, 250] ms です。
<tr class="bg1">	
  <td>6-7<td class="rclr3">D
  <td>SSQR_DUTY1_8, SSQR_DUTY1_4, SSQR_DUTY1_2, SSQR_DUTY3_4, 
    SSQR_DUTY#
  <td>波の<b>デューティ比</b>。矩形波のオンとオフの時間の比です。
    式&nbsp;18.2 を振り返ると、
	これは <i>D=h/T</i> になります。使えるデューティ比は
	12.5%、25%、50%、75%（8 分の 1、4 分の 1、2 分の 1、
	4 分の 3）です。
<tr class="bg0">	
  <td>8-A<td class="rclr2">EST
  <td>SSQR_TIME#
  <td>エンベロープの<b>ステップ時間</b>。エンベロープが変化する間隔:
    &Delta;t = <i>EST</i>/64 s。
<tr class="bg1">	
  <td> B <td class="rclr1">ED
  <td>SSQR_DEC, SSQR_INC
  <td>エンベロープの<b>方向</b>。エンベロープが各ステップで
    減る（デフォルト）か増えるかを示します。
<tr class="bg0">	
  <td>C-F<td class="rclr0">EIV
  <td>SSQR_IVOL#
  <td>エンベロープの<b>初期値</b>。一種の<b>音量</b>の設定と
    考えられます。0 は無音で、15 は最大音量です。方向と
    組み合わせれば、フェードインやフェードアウトができます。
    音を持続させるには、初期音量を 15 にして、方向を増加に
    します。<i>実際の</i>音量を変えるには、
    <code>REG_SNDDMGCNT</code> を思い出してください。
</tbody>
</table>
</div>

<div class="cpt_fr" style="width:312px;">
<img src="./img/sqrfour.png" alt="Fourier transform of square wave" id="fig:sqrf"><br>
<b>{*@fig:sqrf}</b>: 矩形波のスペクトル。
  （整数の <i>m</i> のみ）
</div>

<!--
A_m = \frac{2}{\pi} \cdot \frac{sin({\pi}Dm)}{m}
-->
<table>
<tr>
  <td class="fill">&nbsp;
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <msub>
              <mi>A</mi>
              <mi>m</mi>
            </msub>
            <mo>=</mo>
            <mfrac>
              <mn>2</mn>
              <mi>&#x3C0;</mi>
            </mfrac>
            <mo>&#x22C5;</mo>
            <mfrac>
              <mrow>
                <mi>s</mi>
                <mi>i</mi>
                <mi>n</mi>
                <mo stretchy="false">(</mo>
                <mrow data-mjx-texclass="ORD">
                  <mi>&#x3C0;</mi>
                </mrow>
                <mi>D</mi>
                <mi>m</mi>
                <mo stretchy="false">)</mo>
              </mrow>
              <mi>m</mi>
            </mfrac>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

デューティ比についてもう少し説明します。矩形波をフーリエ解析して、そこに含まれる周波数を求めたことを思い出してください。**基本周波数**のほかに、周波数 *m·f* の**倍音**もあります。スペクトル（{@fig:sqrf} を見てください）は、これらすべての周波数の振幅を表します。図には線が描かれていますが、*m* には整数の値しか使えないことに注意してください。*m*=1 の基本周波数が最も重要で、残りは 1/*m* で小さくなっていきます。面白いのは、サインが効いてくるところです。*m·D* が整数になるたびに、その成分が消えるのです！ （ここで使えるもののような）分数のデューティ比では、これは *m* が分母と等しくなるたびに起きます。50% のデューティでは 1 つおきに倍音が消え、かなりなめらかな音になります。12.5% では 8 つごとにしか消えず、実際に、よりノイズっぽい音になります。¼ と ¾ の*両方*のデューティで 4 つごとに消えるので、両者は区別がつかないはずであることに注意してください。この結果には少し驚きましたが、確かめてみると、たしかに私には同じに聞こえました。

<div class="reg">
<table class="table-reg" id="tbl-reg-snd1freq" width=420>
<caption class="reg">
<span class="nobr">
  REG_SND1FREQ (SOUND1CNT_X / SG11) @ 0400:0062h</span>
  <br> and <br>
<span class="nobr">
  REG_SND2FREQ (SOUND2CNT_H / SG21) @ 0400:006Ch</span>
</caption>
<tr class="bits">
  <td class="wof">F<td>E<td>D C B
  <td class="wof">A 9 8 7 6 5 4 3 2 1 0
<tr class="bf">
  <td class="rclr1">Re
  <td class="rclr2">T
  <td> - 
  <td class="rclr0">R
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td class="wof">0-A<td class="rclr0">R
  <td>SFREQ_RATE#
  <td>音の<b>レート</b>。正確には初期レートです。<i>レート</i>であって、
    周波数ではありません。周期でもありません。レートと周波数の関係は
    <span class="nobr"><i>f</i> = 
    2<sup>17</sup><big>/</big>(2048-<i>R</i>)</span> です。書き込み専用の
    フィールドです。
<tr class="bg1">	
  <td> E <td class="rclr2">T
  <td>SFREQ_HOLD, SFREQ_TIMED
  <td><b>時間指定</b>のフラグ。設定すると、音は長さのフィールド
    （<code>REG_SNDxCNT</code>{0-5}）が示す間だけ鳴ります。
    クリアすると、音はずっと鳴り続けます。減衰する
    エンベロープが 0 に達していても、無音であるにもかかわらず、
    音自体はまだオンとみなされることに注意してください。
<tr class="bg0">	
  <td class="wof"> F <td class="rclr1">Re
  <td>SFREQ_RESET
  <td>音の<b>リセット</b>。音を初期音量（とスイープ）の設定に
    リセットします。レートのフィールドもこのレジスタにあり、
    それが書き込み専用であるため、単純な
    &lsquo;<code>|= SFREQ_RESET</code>&rsquo; では十分では<i>ない</i>ことを
    覚えておいてください（エミュレータではそれで動くかもしれませんが）。
</tbody>
</table>
</div><br>

<div class="reg">
<table class="table-reg" id="tbl-reg-snd1sweep" width=420>
<caption class="reg">
  REG_SND1SWEEP (SOUND1CNT_L / SG10_L) @ 0400:0060h
</caption>
<tr class="bits">
  <td>F E D C B A 9 8 7<td>6 5 4<td>3<td>2 1 0
<tr class="bf">
  <td> - 
  <td class="rclr2">T
  <td class="rclr1">M
  <td class="rclr0">N
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-2<td class="rclr0">N
  <td>SSW_SHIFT#
  <td>スイープの<b>番号</b>。スイープの回数では<i>ありません</i>。
    下の説明を見てください。
<tr class="bg1">	
  <td> 3 <td class="rclr1">M
  <td>SSW_INC, SSW_DEC
  <td>スイープの<b>モード</b>。スイープはレートを上げる
    （デフォルト）か、下げる（セット時）かのどちらかです。
<tr class="bg0">	
  <td>4-6<td class="rclr2">T
  <td>SSW_TIME#
  <td>スイープの<b>ステップ時間</b>。スイープの間隔は
    128 Hz（kHz ではありません！）単位で測ります: &Delta;t = <i>T</i>/128 ms &asymp; 7.8<i>T</i> 
    ms。<i>T</i>=0 ならスイープは無効です。
</tbody>
</table>
</div>

シフトの*正確な*働き方は、ほとんどの資料で十分な注意を払って説明されていないと、かなりの確信を持って言えます。そこで、それについてもう少し説明しておきます。たしかに、スイープはビット 3 で制御される方向に音の高さを上げ下げ*します*し、ステップ時間はその時間が経ったあとで音の高さを変え*ます*。しかし、スイープのシフトが正確に何をするのかは、よく言っても曖昧です。情報は書いてあるのですが、何を探せばいいかを知っていなければわかりません。よく示される式は、次のようなものです。

<!--
T = T \pm T\cdot2^{-n}
-->
<table>
<tr>
  <td class="fill">
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi>T</mi>
            <mo>=</mo>
            <mi>T</mi>
            <mo>&#xB1;</mo>
            <mi>T</mi>
            <mo>&#x22C5;</mo>
            <msup>
              <mn>2</mn>
              <mrow data-mjx-texclass="ORD">
                <mo>&#x2212;</mo>
                <mi>n</mi>
              </mrow>
            </msup>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

これは belogic が示している式で、各項が何なのかを知っていれば問題ありません。あちこちで読むかもしれないこととは違って、スイープは周波数（*f*）にかかるのでは*ありません*。周期（*T*、上を見てください）にかかるのでも*ありません*。**レート**（*R*）にかかるのです。エミュレータで見てみると、実際にレートの値が変わるのを*見る*ことができます。

2 つ目に、指数の *n* は、スイープのシフトの回数まで進んでいく現在のスイープのインデックスでは*ありません*。実際には単に**スイープのシフト数**で、スイープはレートが 0 か最大値の 2047 に達するまで続きます。

見かける式もそう言ってはいるのですが、読み違えやすいのです。私も読み違えました。{*@eq:sweep} に、正しい関係をいくつか示します。*R* はレート、*n* はスイープのシフト（{!@eq:sweep}c を見れば、なぜそれが（複数形ではなく単数形の）*シフト*と呼ばれるのかがわかります）、*j* は現在のスイープのインデックスです。いろいろな見方ができますが、結局はどれも指数関数になります。結局のところ、「d*y*(*x*) = *a·y*(*x*)d*x*」とはそういう意味です。たとえば *n*=1 なら、増加のスイープでは 1½<sup>j</sup>、減少のスイープでは ½<sup>j</sup> のふるまいになります。*n*=2 なら 1¼<sup>j</sup> と ¾<sup>j</sup>、という具合です。シフトが大きいほど、スイープは遅くなります。

<!--
{\Delta}R = 2^{-n} \cdot R
-->
<table id="eq:sweep">
<tr>
  <td class="eqnrcell">({!@eq:sweep}a)
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mrow data-mjx-texclass="ORD">
              <mi mathvariant="normal">&#x394;</mi>
            </mrow>
            <mi>R</mi>
            <mo>=</mo>
            <msup>
              <mn>2</mn>
              <mrow data-mjx-texclass="ORD">
                <mo>&#x2212;</mo>
                <mi>n</mi>
              </mrow>
            </msup>
            <mo>&#x22C5;</mo>
            <mi>R</mi>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

<!--
\begin{matrix}
R_j & = & R_{j-1} \pm R_{j-1} \cdot 2^{-n} \\
    & = & R_{j-1} \cdot (1 \pm 2^{-n}) \\
    & = & R_0 \cdot (1 \pm 2^{-n})^j
\end{matrix}
-->
<table>
<tr>
  <td class="eqnrcell">({!@eq:sweep}b)
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
                  <msub>
                    <mi>R</mi>
                    <mi>j</mi>
                  </msub>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msub>
                    <mi>R</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>j</mi>
                      <mo>&#x2212;</mo>
                      <mn>1</mn>
                    </mrow>
                  </msub>
                  <mo>&#xB1;</mo>
                  <msub>
                    <mi>R</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>j</mi>
                      <mo>&#x2212;</mo>
                      <mn>1</mn>
                    </mrow>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mo>&#x2212;</mo>
                      <mi>n</mi>
                    </mrow>
                  </msup>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msub>
                    <mi>R</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>j</mi>
                      <mo>&#x2212;</mo>
                      <mn>1</mn>
                    </mrow>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <mo stretchy="false">(</mo>
                  <mn>1</mn>
                  <mo>&#xB1;</mo>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mo>&#x2212;</mo>
                      <mi>n</mi>
                    </mrow>
                  </msup>
                  <mo stretchy="false">)</mo>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msub>
                    <mi>R</mi>
                    <mn>0</mn>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <mo stretchy="false">(</mo>
                  <mn>1</mn>
                  <mo>&#xB1;</mo>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mo>&#x2212;</mo>
                      <mi>n</mi>
                    </mrow>
                  </msup>
                  <msup>
                    <mo stretchy="false">)</mo>
                    <mi>j</mi>
                  </msup>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

<table>
<tr>
  <td class="eqnrcell">({!@eq:sweep}c)
  <td class="eqcell">
    <code>R += R &gt;&gt; n;</code>
</table>

### 音を鳴らす {#ssec-snd-notes}

すべてのレートは平等ですが、中にはほかより平等なものもあります。オクターブ 0 の標準的な音の周波数の表（{@tbl:oct0}）はすでに示しました。もちろん、それらを {@eq:fvsr}b でレートに変換して、そのまま使うこともできます。しかし、*すべて*のオクターブの音を鳴らす方法を考えておくと役に立つかもしれません。

そのために、18.2.3 節で触れた音階の成り立ちについての事実をいくつか使います。続く音どうしの対数的な関係（Δ*f*=2<sup>1/12</sup>·*f*）を使うことも*できます*が、ここでは、オクターブ間の音は 2 倍違うという事実だけに絞ります。（当然ながら）レートと周波数の関係も必要です。必要な基本の情報はこれだけです。数学をすべて片づけたら、もっと説明します。そう、また数学ですが、このページではこれが最後です。約束します。

出発点にする式は、一般的な周波数の式と、レートと周波数の関係です。これらには、レート *R*、周波数 *f*、オクターブ *c* があります。さらに、基準のオクターブ *C* と、その基準のオクターブでの周波数 *F* もあります。

<!--
\begin{matrix}
f(F, c) & = & F \cdot 2^{c - C} \\
R(F, c) & = & 2^{11} - \frac{2^{17}}{f(F, c)}
\end{matrix}
-->
<table>
<tr>
  <td class="fill">&nbsp;
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
                  <mo stretchy="false">(</mo>
                  <mi>F</mi>
                  <mo>,</mo>
                  <mi>c</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi>F</mi>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mi>c</mi>
                      <mo>&#x2212;</mo>
                      <mi>C</mi>
                    </mrow>
                  </msup>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <mi>R</mi>
                  <mo stretchy="false">(</mo>
                  <mi>F</mi>
                  <mo>,</mo>
                  <mi>c</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>11</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2212;</mo>
                  <mfrac>
                    <msup>
                      <mn>2</mn>
                      <mrow data-mjx-texclass="ORD">
                        <mn>17</mn>
                      </mrow>
                    </msup>
                    <mrow>
                      <mi>f</mi>
                      <mo stretchy="false">(</mo>
                      <mi>F</mi>
                      <mo>,</mo>
                      <mi>c</mi>
                      <mo stretchy="false">)</mo>
                    </mrow>
                  </mfrac>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
  </table>
</table>

さて、ここからが魔法です。そして、これは理解できると*期待されています*。

<!--
\begin{matrix}
R(F, c) & = & 2^{11} - \frac{2^{17}}{f(F, c)} \\
        & = & 2^{11} - \frac{2^{17}}{F \cdot 2^{c - C}} \\
        & = & 2^{11} - \frac{2^{17 + C - c}}{F} \\
        & = & 2^{11} - \frac{1}{F} \cdot 2^{17 + C + m - (c + m)}  \\
        & = & 2^{11} - \frac{2^{17 + C + m}}{F} \cdot 2^{ - (c + m)}
\end{matrix}
-->
<table id="eq:noterate">
<tr>
  <td class="eqnrcell">({!@eq:noterate})
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
                  <mi>R</mi>
                  <mo stretchy="false">(</mo>
                  <mi>F</mi>
                  <mo>,</mo>
                  <mi>c</mi>
                  <mo stretchy="false">)</mo>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>11</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2212;</mo>
                  <mfrac>
                    <msup>
                      <mn>2</mn>
                      <mrow data-mjx-texclass="ORD">
                        <mn>17</mn>
                      </mrow>
                    </msup>
                    <mrow>
                      <mi>f</mi>
                      <mo stretchy="false">(</mo>
                      <mi>F</mi>
                      <mo>,</mo>
                      <mi>c</mi>
                      <mo stretchy="false">)</mo>
                    </mrow>
                  </mfrac>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>11</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2212;</mo>
                  <mfrac>
                    <msup>
                      <mn>2</mn>
                      <mrow data-mjx-texclass="ORD">
                        <mn>17</mn>
                      </mrow>
                    </msup>
                    <mrow>
                      <mi>F</mi>
                      <mo>&#x22C5;</mo>
                      <msup>
                        <mn>2</mn>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mo>&#x2212;</mo>
                          <mi>C</mi>
                        </mrow>
                      </msup>
                    </mrow>
                  </mfrac>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>11</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2212;</mo>
                  <mfrac>
                    <msup>
                      <mn>2</mn>
                      <mrow data-mjx-texclass="ORD">
                        <mn>17</mn>
                        <mo>+</mo>
                        <mi>C</mi>
                        <mo>&#x2212;</mo>
                        <mi>c</mi>
                      </mrow>
                    </msup>
                    <mi>F</mi>
                  </mfrac>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>11</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2212;</mo>
                  <mfrac>
                    <mn>1</mn>
                    <mi>F</mi>
                  </mfrac>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>17</mn>
                      <mo>+</mo>
                      <mi>C</mi>
                      <mo>+</mo>
                      <mi>m</mi>
                      <mo>&#x2212;</mo>
                      <mo stretchy="false">(</mo>
                      <mi>c</mi>
                      <mo>+</mo>
                      <mi>m</mi>
                      <mo stretchy="false">)</mo>
                    </mrow>
                  </msup>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mn>11</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2212;</mo>
                  <mfrac>
                    <msup>
                      <mn>2</mn>
                      <mrow data-mjx-texclass="ORD">
                        <mn>17</mn>
                        <mo>+</mo>
                        <mi>C</mi>
                        <mo>+</mo>
                        <mi>m</mi>
                      </mrow>
                    </msup>
                    <mi>F</mi>
                  </mfrac>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mn>2</mn>
                    <mrow data-mjx-texclass="ORD">
                      <mo>&#x2212;</mo>
                      <mo stretchy="false">(</mo>
                      <mi>c</mi>
                      <mo>+</mo>
                      <mi>m</mi>
                      <mo stretchy="false">)</mo>
                    </mrow>
                  </msup>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

では、*なぜ*これが役に立つのかです。GBA にはハードウェアの除算も浮動小数点のサポートもないので、残されているのは整数と（可能なら）シフトだということを思い出してください。{@eq:noterate} の最後の段階で最後の項を分けたのはそのためです。*F* の項は基準のオクターブでのレートのオフセットを与え、それを各オクターブのオクターブのオフセットの項で割る（つまりシフトする）必要があります。整数の除算は切り捨てられるので、精度を最大にするには大きな分子が必要です。それは、大きな *C* を使い、余分な項 *m* を加えることで実現できます。基本的に、これで *m*f の固定小数点の除算になります。使えるオクターブの範囲は −2 から 5 なので、*C*=5 にします。*m* の値は*ほぼ*任意ですが、最小のオクターブが −2 で、シフトは決して負にできないので、2 より大きくする必要があります。*m*=4 で十分です。

それでも*まだ*除算が残っていることに注意してください。幸い、*F* に使える値は 12 個しかないので、項全体をルックアップテーブルに保存してしまえばよいでしょう。最終的な結果が、下のリスト 18.1 です。

<div id="cd-snd-rate">

```c
// Listing 18.1: a sound-rate macro and friends

typedef enum 
{
    NOTE_C=0, NOTE_CIS, NOTE_D,   NOTE_DIS, 
    NOTE_E,   NOTE_F,   NOTE_FIS, NOTE_G, 
    NOTE_GIS, NOTE_A,   NOTE_BES, NOTE_B
} eSndNoteId;

// Rates for equal temperament notes in octave +5
const u32 __snd_rates[12]=
{
    8013, 7566, 7144, 6742, // C , C#, D , D#
    6362, 6005, 5666, 5346, // E , F , F#, G
    5048, 4766, 4499, 4246  // G#, A , A#, B
};

#define SND_RATE(note, oct) ( 2048-(__snd_rates[note]>>(4+(oct))) )

// sample use: note A, octave 0
    REG_SND1FREQ= SFREQ_RESET | SND_RATE(NOTE_A, 0);
```
</div>

ここには、音のインデックスのための定数がいくつかと、レートのオフセットのルックアップテーブル `__snd_rates`、そして欲しいものを返してくれる単純なマクロがあります。ここでは `__snd_rates` は定数ですが、調律できるように定数でない版にすることも考えられます。矩形波に調律する価値があるわけではありませんが、言ってみただけです……ね。

1 つ面倒かもしれないのは、音を音名の部分とオクターブの部分に分けなければならないことで、それを動的に行うには 12 による除算と剰余が必要になります。本当にそうでしょうか？ [定数による除算はその逆数による乗算である](fixed.html#sec-rmdiv)ことについて少し知っていれば、どうすればいいかわかるはずです。（<span class="small">ヒント: *c*=(*N*\*43\>\>9)−2。ここで *N* は 0 から 95 まで（オクターブ −2 から +5 まで）の通しの音のインデックスです。</span>）

## デモの時間 {#sec-demo}

今日のところは理論はもう十分だと思います。そうでしょう、読者の皆さん？

「 \@\_@ 」

はい、ということにしておきます。今回のデモでは、この章のいろいろなマクロ、特に `SND_RATE` の使い方を示します。また、矩形波の生成器でちょっとした曲（という言葉は軽い意味で使っています）を演奏する方法も示します。どの曲かわかってもらえるといいのですが。

<pre><code class="language-c hljs">#include &lt;stdio.h&gt;
#include &lt;tonc.h&gt;

u8 txt_scrolly= 8;

const char *names[]=
{   "C ", "C#", "D ", "D#", "E ", "F ", "F#", "G ", "G#", "A ", "A#", "B "  };

// === FUNCTIONS ======================================================

// Show the octave the next note will be in
void note_prep(int octave)
{
    char str[32];
    siprintf(str, "[  %+2d]", octave);
    se_puts(8, txt_scrolly, str, 0x1000);
}


// Play a note and show which one was played
void note_play(int note, int octave)
{
    char str[32];

    // Clear next top and current rows
    SBB_CLEAR_ROW(31, (txt_scrolly/8-2)&31);
    SBB_CLEAR_ROW(31, txt_scrolly/8);   

    // Display note and scroll
    siprintf(str, "%02s%+2d", names[note], octave);
    se_puts(16, txt_scrolly, str, 0);

    txt_scrolly -= 8;
    REG_BG0VOFS= txt_scrolly-8;

    // Play the actual note
    <span class="bold">REG_SND1FREQ = SFREQ_RESET | SND_RATE(note, octave);</span>
}


// Play a little ditty
void sos()
{
    const u8 lens[6]= { 1,1,4, 1,1,4 };
    const u8 notes[6]= { 0x02, 0x05, 0x12,  0x02, 0x05, 0x12 };
    int ii;
    for(ii=0; ii&lt;6; ii++)
    {
        note_play(notes[ii]&15, notes[ii]&gt;&gt;4);
        VBlankIntrDelay(8*lens[ii]);
    }
}

int main()
{
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    irq_init(NULL);
    irq_add(II_VBLANK, NULL);

    txt_init_std();
    txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0, CLR_ORANGE, 0);
    pal_bg_mem[0x11]= CLR_GREEN;

    int octave= 0;

    // turn sound on
    <span class="bold">REG_SNDSTAT= SSTAT_ENABLE;</span>
    // snd1 on left/right ; both full volume
    <span class="bold">REG_SNDDMGCNT = SDMG_BUILD_LR(SDMG_SQR1, 7);</span>
    // DMG ratio to 100%
    <span class="bold">REG_SNDDSCNT= SDS_DMG100;</span>

    // no sweep
    <span class="bold">REG_SND1SWEEP= SSW_OFF;</span>
    // envelope: vol=12, decay, max step time (7) ; 50% duty
    <span class="bold">REG_SND1CNT= SSQR_ENV_BUILD(12, 0, 7) | SSQR_DUTY1_2;</span>
    REG_SND1FREQ= 0;

    sos();

    while(1)
    {
        VBlankIntrWait();
        key_poll();

        // Change octave:
        octave += bit_tribool(key_hit(-1), KI_R, KI_L);
        octave= wrap(octave, -2, 6);
        note_prep(octave);

        // Play note
        if(key_hit(KEY_DIR|KEY_A))
        {
            if(key_hit(KEY_UP))
                note_play(NOTE_D, octave+1);
            if(key_hit(KEY_LEFT))
                note_play(NOTE_B, octave);
            if(key_hit(KEY_RIGHT))
                note_play(NOTE_A, octave);
            if(key_hit(KEY_DOWN))
                note_play(NOTE_F, octave);
            if(key_hit(KEY_A))
                note_play(NOTE_D, octave);
        }

        // Play ditty
        if(key_hit(KEY_B))
            sos();      
    }
    return 0;
}
</code></pre>

`main()` の太字のコードは、サウンドのレジスタを初期化しています。凝ったことは何もしていませんが、何か音が聞こえるようになる前に、これをしておかなければなりません。まず `REG_SNDSTAT` のビット 7（`SSTAT_ENABLE`）、つまりマスターサウンドの有効化から始めることが大事です。これがないと、ほかのレジスタにアクセスすることすらできません。もちろん、音量をゼロでない値にしておくのも良い考えです。それからスイープの機能を切り、サウンド 1 がデューティ 50% で減衰するエンベロープを使うように設定します。ここからが楽しいところです。

`sos()` が何なのかは少しあとで説明しますが、まずはデモの操作について。十字キーと A で音を鳴らせます（ふむ、この配置には何か見覚えがありますね）。作業中のオクターブ *c* は L と R で変えられ、それに合わせて背景の色も変わります。B でもう一度 `sos()` を演奏します。

<div class="lblock">
  <table>
    <tbody valign="top">
      <tr>
        <th>A / D-pad <td>音を鳴らす
          <tr><td><td>&uarr; : D（次のオクターブ）
          <tr><td><td>&larr; : B 
          <tr><td><td>&rarr; : A 
          <tr><td><td>&darr; : F 
          <tr><td><td> A     : D 
      <tr>
        <th>L / R <td> 現在のオクターブを下げる / 上げる（[-2, 5]、
        端で回り込む）
      <tr>
        <th> B    <td>ちょっとした曲を演奏する。
    </tbody>
  </table>
</div>

十字キーと A で鳴らす音を選び、それを `note_play()` で処理しています。そこの太字の行が実際に音を鳴らしていて、残りは、今鳴らした音を画面に書いてスクロールさせ、何を鳴らしたかの履歴が見えるようにするための追加の処理です。このためのコードはちょっと汚いのですが、話の本筋ではないので問題ありません。

### ちょっとした曲を演奏する {#ssec-demo-ditty}

では、`sos()` とはいったい何なのでしょうか？ もう一度見てみましょう。

```c
void sos()
{
    const u8 lens[6]= { 1,1,4, 1,1,4 };
    const u8 notes[6]= { 0x02, 0x05, 0x12,  0x02, 0x05, 0x12 };
    int ii;
    for(ii=0; ii<6; ii++)
    {
        note_play(notes[ii]&15, notes[ii]>>4);
        VBlankIntrDelay(8*lens[ii]);
    }
}
```

ここには `notes` と `lens` の 2 つの配列と、すべての要素を回るループがあります。`notes` から 1 バイトを取り出し、その 4 ビットずつをオクターブと音の情報として使って音を鳴らし、それからしばらく待って（長さは `lens` 配列で示されます）、次の音を鳴らします。基本的に、これは音楽を演奏しているのです。ほら、*Schnappi* や *Crazy Frog* のようなものがトップ 10 に入れるのなら、*これ*も音楽と呼んでいいと思いますよね？ いいですよね。

私が言いたいのは、音の生成器だけでも曲を演奏することは十分にできるということです。厳密には、何かを演奏するのに、デジタル化した音楽などは必要ありません。もちろん、あったほうが良い音になりますが、ちょっとしたジングルが必要なだけなら、音の生成器だけで十分かもしれません。音の生成器だけを使った 12 年分のゲームボーイのゲームが、それを証明しています。音（オクターブと音名の 4 ビット形式で十分です）と長さをいくつか定義すれば、基本はもうできあがりです。いろいろな効果のために、複数のチャンネルを使うことさえできます。

それが理解できたなら、次のことも理解してください。音＋長さ＋チャンネルという考え方は、トラッカーの音楽（mod、it、xm など）がしていることとほぼ同じなのです。違うのは、矩形波より洗練された波形を使うことだけです。でも原理は同じです。それを動かすにはもう少し手間がかかりますが、そのために Deku の[サウンドミキシングのチュートリアル](https://stuij.github.io/deku-sound-tutorial/)があるのです。
