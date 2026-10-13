# 21. モード 7 その 2

<!-- toc -->

## はじめに {#sec-intro}

[モード 7: その 1](mode7.html) では、アフィン背景を 3D の平面のように見せる方法の基本を扱い、そこに含まれる固定小数点の計算のやっかいな部分をいくつか説明しました。3D の平面の基本的な見た目を得るのは、最初の一歩にすぎません。

この章では、3D の世界を作るのに必要な一般的な数学を見て、それをモード 7 で使える形に戻します。これには、前と同じあらゆる方向への平行移動と見回し（ヨー）に加えて、上下を見るためのピッチの角度も含まれます。地平線の扱い方と、地平線より上の背景（バックドロップ）に背景レイヤーを使う方法も見ていきます。地面の遠くの部分を覆い隠すための、ちょっとしたフォグまでおまけに付けます。

3D 空間でのスプライトの扱いについても説明します。3D 空間から 2D の画面への変換だけでなく、カリング、距離に応じた拡大縮小（思ったほど単純ではありません）、アニメーション、ソートもです。この章のこの部分は、3D のスプライトの基本的な理論で、何らかの形でスプライトを使う 3D のゲームに応用できることに注意してください。

この章の理論の部分は、3D の理論がいつもそうであるように、とても数学が多くなります。[線形代数](matrix.html)について少し知っていると、きっと役に立つでしょう。幾何学の話を全部するのは Tonc の範囲を超えますが、ここの内容はかなり一般的なものです。3D プログラミングの本のほとんどには幾何学的な変換の章があるので、ここで少し迷ったらそれらを見るとよいでしょう。

この章は、これまでに扱ったほとんどすべての話題に触れます。[アフィンオブジェクト](affobj.html)、背景（[通常](regbg.html)と[アフィン](affbg.html)の両方）、[割り込み](interrupts.html)、[色の効果](gfx.html#blend)などを使います。どれかの理解が足りないと、ここでは苦労するかもしれません。

<div class="cblock">
    <table id="fig:img-m7-ex">
      <tbody align="center">
        <tr>
          <td><img src="img/mode7/m7_ex_00.png" alt="full mode 7"></td>
          <td><img src="img/mode7/m7_ex_01.png" alt="full mode 7"></td>
        </tr>
        <tr>
          <td colspan=2>
            <b>*@fig:img-m7-ex</b>: <tt>m7_ex</tt>。地平線、スプライト、可変のピッチの角度、距離に応じたフォグ付き。
          </td>
        </tr>
      </tbody>
    </table>
</div>

これからやろうとしているのは、スーパーファミコンのマリオカートの一場面を再現することです（{@fig:img-m7-ex} を見てください。グラフィックを使ってしまって任天堂さんごめんなさい、でもここではあまり選択肢がないのです <kbd>:\\</kbd>）。これはゲームの一場面を止めたもので、実際のゲームプレイはありませんが、目指す目標としてはちょうどよいでしょう。コードはいくつかのファイルに分かれています。単純なモード 7 の関数は `mode7.c` に、それほど単純ではないモード 7 の関数と割り込みルーチンは `mode7.iwram.c` にあります。デモ固有のコードは `m7_ex.c` にあり、準備、操作、メインループを行います。基本の操作は次のとおりです。

<div class="lblock">
  <table cellpadding=2 cellspacing=0>
    <col span=2 align="left" valign="top">
      <tr><th>D-pad	<td>見回す
      <tr><th>A/B		<td>後退/前進
      <tr><th>L/R		<td>横移動（ストレイフ）
      <tr><th>Select+A/B	<td>上昇/下降
      <tr><th>Start	<td>メニュー
  </table>
</div>

移動と見回しは FPS や飛行機の動きに従っています。少なくとも、使えるボタンの数で期待できる程度にはです。いくつかの追加のオプションはメニューに入れてあります。1 つ目は_移動の制御_で、移動の方法を切り替えます。「local」はフライトの操作のようにカメラの軸に従い、「level」は FPS がたいていそうであるように地面と平行に移動し、「global」は移動にワールドの軸を使います。ほかに、フォグのオンオフの切り替えと、デモのリセットがあります。

## モード 7 の基本の理論 {#sec-theory}

{\*@fig:img-crd-overview} は、私たちが立ち向かう相手を示しています。**a**<sub>cw</sub> にあるカメラがあり、それはワールドの座標系に対して何らかの向きを持っています。すべきことは、画面の点 **x**<sub>s</sub> とワールドの点 **x**<sub>w</sub> を結びつける変換を見つけることです。これにはいくつかの方法があります。1 つは[モード 7 の最初の章](mode7.html)ですでに見ました。そこでは最初から GBA のハードウェアを念頭に置いていました。少し手間をかければ、これを一般のモード 7 の場合（ゼロでないピッチ）に拡張することもできます。純粋に三角法を使うこともできますが、それはマイナスの符号の地雷原で、サインとコサインを取り違える危険もあります。それでも可能ではあります。ただ、ここで使うのは[線形代数](matrix.html)です。これを選んだ理由はいくつかあります。まず、線形代数はとても簡潔な表記を持つので、最終的な解をほんの数行で書き表せます（実際、定義さえ済めば、すべての場合を網羅する解は 2 行で書けます）。さらに、式がよく整理されていて見た目も統一されているので、デバッグが楽になります。それから、全体の逆を求めるのがとても簡単だということもあります。そして最後に、本物の 3D システムも線形代数を使っているので、この理論はモード 7 の外でも応用できます。逆に、3D の基本の理論を知っていれば、ここではすっかりくつろげるでしょう。

<div class="lblock">
  <div class="cpt" style="width:378px;">
    <img src="img/mode7/crd_overview.png" id="fig:img-crd-overview" alt="overview">
    <br>
    <b>*@fig:img-crd-overview</b>: 3D の基本の状況。コツは、カメラの位置 <b>a</b><sub>cw</sub> とその向きを考慮に入れて、画面の点 <b>x</b><sub>s</sub> をワールドの点 <b>x</b><sub>w</sub> に結びつけることです。
  </div>
</div>

### 定義 {#ssec-try-defs}

ただし、何かをする前に、何を使うのかを_正確に_知っておく必要があります。最初に注意すべきなのは、主な座標系が 2 つあることです。<dfn>ワールド</dfn>座標系 _S_<sub>w</sub> と<dfn>カメラ</dfn>座標系 _S_<sub>c</sub> です。カメラ座標系の中には、さらに 2 つの小さな座標系があります。<dfn>投影</dfn>空間 _S_<sub>p</sub> と<dfn>画面</dfn>空間 _S_<sub>s</sub> です。さて、座標系 _S_<sub>i</sub> と _S_<sub>j</sub> の間のすべての変換について、次の関係が成り立ちます。

<table id="eq:crd-transf">
  <tr>
    <td class="eqnrcell">(!@eq:crd-transf)
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <msub>
                    <mi>M</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>i</mi>
                      <mi>j</mi>
                    </mrow>
                  </msub>
                  <mtext>&#xA0;</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext>&#xA0;</mtext>
                  <msub>
                    <mi>x</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>j</mi>
                    </mrow>
                  </msub>
                  <mo>=</mo>
                  <msub>
                    <mi>x</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>i</mi>
                    </mrow>
                  </msub>
                  <mo>&#x2212;</mo>
                  <msub>
                    <mi>a</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>j</mi>
                      <mi>i</mi>
                    </mrow>
                  </msub>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
</table>

ここで

<div class="lblock">
  <table cellpadding=0>
    <tr>
      <td><b>x</b><sub>i</sub>
      <td>座標系 <i>S</i><sub>i</sub> での座標ベクトル。
    <tr>
      <td><b>x</b><sub>j</sub>
      <td>座標系 <i>S</i><sub>j</sub> での座標ベクトル。
    <tr>
      <td><b>a</b><sub>ji</sub>
      <td>座標系 <i>S</i><sub>j</sub> の原点を、座標系 <i>S</i><sub>i</sub> の座標で表したもの。
    <tr>
      <td><b>M</b><sub>ij</sub>
      <td>変換行列。基本的には、<i>S</i><sub>j</sub> の主軸のベクトルを <i>S</i><sub>i</sub> で表して並べた行列。
  </table>
</div>

たくさんの添字に最初はぎょっとするかもしれませんが（まあ、一般相対性理論にはリーマンテンソルというものがあって、添字が _4 つ_もあるんですよ）、それを乗り越えれば、この式が筋の通ったものだとわかるでしょう。すぐにはわからなければ、配列と行列だと考えてください。目ざとい読者なら、アフィンのマップのところで見た[画面↔マップの変換](affbg.html#sec-aff-ofs) **P·q** = **p − dx** と同じ構造に気づくでしょう。ちなみに {\*@eq:crd-transf} はとても一般的な式で、あらゆる種類の線形の座標変換で成り立ちます。実際、座標系 _S_<sub>i</sub> と _S_<sub>j</sub> の次元の数が同じである必要すらありません！

先ほど言ったように、全部で 4 つの座標系があるので、<dfn>w</dfn>(orld、ワールド)、<dfn>c</dfn>(amera、カメラ)、<dfn>p</dfn>(rojection、投影)、<dfn>s</dfn>(creen、画面) の 4 つの添字があります。これらは本当にしょっちゅう出てくるので、覚えておいてください。行列と原点の最終的な形は、これらの座標系の正確な定義にとても強く依存するので、それぞれが何を意味するのかを正確に把握しておいてください。

### ワールド座標系 {#ssec-try-world}

1 つ目のワールド座標系 _S_<sub>w</sub> は簡単に扱えます。これは単に、主軸 **i**、**j**、**k** を持つ右手系のデカルト座標系で、それぞれが x 軸、y 軸、z 軸です。コンピュータグラフィックスで使われる右手系では、x 軸（**i**）は右を、y 軸（**j**）は上を、そして z 軸（**k**）は_後ろ_を指します！ つまり負の _z_ 方向を見ていることになり、最初は変に思えるかもしれません。どうしても前を向いた **k** が欲しければ、左手系を使うこともできます。私の 3D の直感は完全に崩壊しますが、そうしたければどうぞ。ただしその前に、マップはワールド空間の床を表していて、右手系ならテクスチャの座標がワールドの座標ときれいに一致することを思い出してください。

### カメラの座標系 {#ssec-try-cam}

<div class="cpt_fr" style="width:260px;">
  <img src="img/mode7/crd_w2c.png" id="fig:img-w2c" alt="word to camera transformation">
  <br>
  <b>{*@fig:img-w2c}</b>: ワールド空間 {<b>i, j, k</b>} でのカメラの向き {<b>u, v, w</b>}。角度 &theta; と &phi; で与えられます
</div>

カメラ座標系への変換は、おそらく全体の中で一番の難関です。少なくとも、行列がなければそうだったでしょう。@eq:crd-transf を書き直すと、カメラ空間とワールド空間の間の変換は次のようになります。

<table id="eq:w2c">
<tr>
   <td class="eqnrcell">({!@eq:w2c})
   <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mi>C</mi>
                <mtext>&#xA0;</mtext>
                <mo>&#x22C5;</mo>
                <mtext>&#xA0;</mtext>
                <msub>
                  <mi>x</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>c</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <msub>
                  <mi>x</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>w</mi>
                  </mrow>
                </msub>
                <mo>&#x2212;</mo>
                <msub>
                  <mi>a</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>c</mi>
                    <mi>w</mi>
                  </mrow>
                </msub>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
</table>

予想どおり、カメラ空間の原点はカメラの位置 **a**<sub>cw</sub> です。カメラ行列 **C** はカメラ空間の主軸でできていて、それはそれぞれローカルの x 軸、y 軸、z 軸である **u、v**、**w** です。つまり、カメラ行列は **C** = \[**u v w**\] です。

ワールド空間に対するカメラの向きは、3 つの角度で定義されます。<dfn>ピッチ</dfn>（x 軸周りの回転）、<dfn>ヨー</dfn>（y 軸周りの回転）、<dfn>ロール</dfn>（z 軸周り）です。これらを組み合わせると **C** になります。伝統的には、これらの回転の向きは、その軸を見下ろしたときに正の角度で座標系が反時計回りに回るように決められています。しかし私はまったく逆にします。そのほうがいくつかのことが楽になるからです。さらに、使う角度はピッチとヨーの 2 つだけです。モード 7 にロールを組み込むのは不可能です。なぜでしょうか？ こう考えてみてください。横に傾いていたら、地面は画面の右か左に来ることになり、縦方向の透視除算が必要になります。HBlank でしかアフィンパラメータを変えられないので、それは実現不可能です。そのため、ピッチ（θ）とヨー（φ）しか使えません。正の θ と φ で、それぞれ下と右を向くようにしたいので、次の回転行列が必要です。

<table id="eq:mat-rot">
  <tr>
    <td class="eqnrcell">({!@eq:mat-rot}a)</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <msub>
                    <mtext mathvariant="bold">R</mtext>
                    <mrow data-mjx-texclass="ORD">
                      <mi>x</mi>
                    </mrow>
                  </msub>
                  <mo stretchy="false">(</mo>
                  <mi>&#x3B8;</mi>
                  <mo stretchy="false">)</mo>
                  <mo>=</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mn>1</mn>
                        </mtd>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mo>&#x2212;</mo>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
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
    </td>
  </tr>
  <tr>
    <td class="eqnrcell">({!@eq:mat-rot}b)</td>
    <td>
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <msub>
                    <mtext mathvariant="bold">R</mtext>
                    <mrow data-mjx-texclass="ORD">
                      <mi>y</mi>
                    </mrow>
                  </msub>
                  <mo stretchy="false">(</mo>
                  <mi>&#x3C6;</mi>
                  <mo stretchy="false">)</mo>
                  <mo>=</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mo>&#x2212;</mo>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mn>1</mn>
                        </mtd>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
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
    </td>
  </tr>
</table>

ところが、ここで次の問題が出てきます。ピッチとヨーのどちらを先にするのか？ それは、どんな効果を出したいのか、_そして_どの座標系に対して回転を行うのかに本当によります。実は、ロールが使えなかったのと同じ理由で、可能な順序は 1 つしかありません。縦方向の遠近法は使えないのです。結局のところ、**u**（カメラの座標系の x 軸）は地面と平行で_なければならない_、つまり _u_<sub>y</sub> はゼロでなければならないということです。そうするには、ピッチを先にして、それからヨーをする必要があります。これを {@fig:img-w2c} に示します。感覚をつかむために、立ち上がって、頭を下に傾け（ピッチ θ\>0）、それから右を向いて（ヨー φ\>0）みてください。カメラ行列全体は次のようになります。

<table id="eq:rotxy">
  <tr>
    <td class="eqnrcell">({!@eq:rotxy})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">C</mtext>
                  <mo stretchy="false">(</mo>
                  <mi>&#x3B8;</mi>
                  <mo>,</mo>
                  <mi>&#x3C6;</mi>
                  <mo stretchy="false">)</mo>
                  <mo>=</mo>
                  <msub>
                    <mtext mathvariant="bold">R</mtext>
                    <mrow data-mjx-texclass="ORD">
                      <mi>y</mi>
                    </mrow>
                  </msub>
                  <mo stretchy="false">(</mo>
                  <mi>&#x3C6;</mi>
                  <mo stretchy="false">)</mo>
                  <mtext>&#xA0;</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext>&#xA0;</mtext>
                  <msub>
                    <mtext mathvariant="bold">R</mtext>
                    <mrow data-mjx-texclass="ORD">
                      <mi>x</mi>
                    </mrow>
                  </msub>
                  <mo stretchy="false">(</mo>
                  <mi>&#x3B8;</mi>
                  <mo stretchy="false">)</mo>
                  <mo>=</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                          <mo>&#x22C5;</mo>
                          <mi>s</mi>
                          <mi>i</mi>
                          <mi>n</mi>
                          <mo stretchy="false">(</mo>
                          <mi>&#x3B8;</mi>
                          <mo stretchy="false">)</mo>
                        </mtd>
                        <mtd>
                          <mo>&#x2212;</mo>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                          <mo>&#x22C5;</mo>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mn>0</mn>
                        </mtd>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mo>&#x2212;</mo>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                          <mo>&#x22C5;</mo>
                          <mi>sin</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mtd>
                        <mtd>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3C6;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                          <mo>&#x22C5;</mo>
                          <mi>cos</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mi>&#x3B8;</mi>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
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
    </td>
</table>

正しいことに加えて、この行列には 2 つの素敵な性質があります。1 つ目は、列ベクトルの長さが 1 であること。2 つ目は、成分のベクトルが互いに垂直であることです。つまり **C** は<dfn>直交行列</dfn>で、**C**<sup>−1</sup> = **C**<sup>T</sup> というとても素敵な特徴を持っています。おかげで、ワールド→カメラの変換は比較的単純な操作になります。

ここで最後に 1 つ。カメラ座標系を **i** の周りに 180° 回転させると、**w** は前を、**v** は下を向くことになります。どちらもあとの計算での厄介なマイナスの符号を減らしますが、その代わりにカメラの座標系が厄介なものになります。そうするかどうかはお任せします。

:::note 行列の変換と、それが行われる座標系

**C** の回転をまねるには、まず頭を傾け（θ）、それから体を回す（φ）と言いました。逆の順序、つまり先に向きを変えてから見下ろしても同じ効果が得られると思うかもしれません。しかし、それは間違いです。

同じように_感じる_かもしれませんが、2 つ目の場合は、傾けるのに実際には **R**<sub>x</sub>(θ) を使っていないことになります。行列はそれ自体で存在するものではなく、ある空間の中に「住んで」います。この場合、**R**<sub>x</sub>(θ) と **R**<sub>y</sub>(φ) はどちらも_ワールド_座標系で定義されていて、それらをかけるとき、方向はワールドの軸に従います。向きを変えてから傾ける順序では、**R**<sub>x</sub>(θ) をローカルの座標系で使うことになります。それは正当な操作ですが、数学が必要としている操作ではありません。

微妙な点なのはわかっていますが、本当に重要な違いがあるのです。両方の順序で 90° 回転させるところを思い描いてみてください。それで助けになるかもしれません。

:::

### 投影面 {#ssec-try-proj}

<div class="cpt_fr" style="width:256px;">
  <img src="img/mode7/crd_c2p.png" id="fig:img-c2p" alt="">
  <br>
  <b>{*@fig:img-c2p}</b>: 透視投影。
</div>

奥行きの錯覚を作るには、<dfn>透視図</dfn>が必要です。そのためには、<dfn>投影の中心</dfn>（COP）と<dfn>投影面</dfn>が必要です。当然、どちらもカメラ空間にある必要があります。これらは好きなように選んでかまいませんが、投影の中心をカメラ空間の原点に置き、投影面をカメラの前方の距離 _D_ に置けば、話が簡単になります。そうすると平面は **x**<sub>p</sub> = (_x_<sub>p</sub>, *y*<sub>p</sub>, −*D*) で表されます。そう、負の _z_<sub>p</sub> です。負の z 方向を見ているからです。投影された座標は、COP と **x**<sub>c</sub> を結ぶ直線と投影面との交点です。COP は原点にあるので、**x**<sub>c</sub> と **x**<sub>p</sub> の関係は次のようになります。

<table id="eq:c2p">
<tr>
  <td class="eqnrcell">({!@eq:c2p})</td>
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mi>&#x3BB;</mi>
                <mtext>&#xA0;</mtext>
                <msub>
                  <mi>x</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>p</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <msub>
                  <mi>x</mi>
                  <mi>c</mi>
                </msub>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
    </td>
</table>

ここで λ は単純なスケール係数で、その値は、導出のその時点で使える情報に応じて、いろいろな方法で求められます。たとえば、定義により _z_<sub>p</sub> = −*D* なので、λ = −*z*<sub>c</sub> / _D_ です。あとで別の式も見ます。この式の面白いところは、λ がカメラ空間での距離に比例していて、それがカメラの位置をどれだけ_縮小_、つまりズームすればいいかを教えてくれることです。アフィン行列の拡大縮小のパラメータも縮小する方向に効くので、これは便利です。また、距離 _D_ は拡大縮小を弱めるので、<dfn>焦点距離</dfn>として働きます。_z_<sub>c</sub> = −*D* のときスケールが 1 になる、つまりこの距離にある物体は普通の大きさに見えることに注意してください。

### ビューポートと視野の空間 {#ssec-try-view}

<div class="cpt_fr" style="width:308px;">
  <img src="img/mode7/viewport.png" id="fig:img-viewport" alt="">
  <br>
  <b>{*@fig:img-viewport}</b>: カメラ空間での視錐台。緑の長方形が投影面の見える部分（つまり画面）です。
</div>

画面への変換の最後の段階を示す前に、ビューポートと視野の空間について少し話しておかなければなりません。想像がつくでしょうが、見えるのは世界のある部分だけです。<dfn>ビューポート</dfn>と呼ばれる領域を通して世界を見ます。これは投影面上の領域で、普通は長方形をしていて、見えるものの左右と上下の境界を決めます。具体的には、左辺（_L_）、右辺（_R_）、上辺（_T_）、下辺（_B_）があります。軸がこのように定義されていて、原点が普通は中央にある（@fig:img-viewport の差し込み図を見てください）ので、_R\>0\>L_ と _T\>0\>B_ になります。そう、この場合は _L_ が負で、_T_ が正です！

ビューポートの幅と高さは、それぞれ *W* = \|_R−L_\| と *H* = \|_B−T_\| です。投影の中心と合わせて、ビューポートが<dfn>視野の空間</dfn>を決めます（@fig:img-viewport を見てください）。長方形のビューポートなら、これは角錐になります。

たいていは奥行きの境界も欲しくなります。近すぎるものはほかのすべてを視界から遮ってしまいます（それに、0 で割るのは決して良いことではありません）し、とても遠くの物体は小さくなりすぎてほとんど気づかないので、ほんの一握りのピクセルのためにたくさんの計算を無駄にすることはありません。これらの奥行きの境界は<dfn>ニア</dfn>（_N_）面と<dfn>ファー</dfn>（_F_）面と呼ばれ、視野の空間を錐台に変えます。これらの距離の数値は好みの問題です。何を使うにせよ、z の値は実際には負であることに気をつけてください。私は _N_ と _F_ の値を正にしておき、距離の順序が 0\>−*N*\>−*F* になるようにするのが好みです。

もう 1 つは<dfn>視野角</dfn>（FOV）という考え方です。これは見ることのできる横方向の角度 α で、次のようになります。

<table id="eq:fov-cam">
  <tr>
    <td class="eqnrcell">({!@eq:fov-cam})
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>tan</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mrow>
                    <mo data-mjx-texclass="OPEN">(</mo>
                    <mi>&frac12; &#x3B1;</mi>
                    <mo data-mjx-texclass="CLOSE">)</mo>
                  </mrow>
                  <mo>=</mo>
                  <mfrac>
                    <mi>&frac12; W</mi>
                    <mi>D</mi>
                  </mfrac>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
</table>

よく使われる FOV は約 90° だと聞いています。それには *D* = ½*W* が必要です。*D* = 128 なら、この条件に十分近く、しかも 2 の累乗だという利点もありますが、もちろんそれは実装の細部です。_しかし_、*D* = 256 のほうが一般的なようなので、代わりにそれを使います。

### 画面 {#ssec-try-scr}

<div class="cpt_fr" style="width:192px;">
  <img src="img/mode7/crd_p2s.png" id="fig:img-p2s" alt="">
  <br>
  <b>{*@fig:img-p2s}</b>: 画面空間とカメラ空間
</div>

最後の段階は、投影面から画面への変換です。この段階はほとんど自明ですが、その「ほとんど」が、注意していないと大きな問題の原因になりかねません。状況を @fig:img-p2s に示します。カメラを通して見ているところです。軸 **u** と **v** はカメラ座標系の上と右の軸（訳注: 順序どおりなら **u** が右、**v** が上です）で、緑の矢印は画面空間の x 軸と y 軸を表しています。そして、どれかのチュートリアルに注意を払っていれば、画面の y 軸は_下_を向いていることを知っているはずです。これがバグ祭りその 1 です。また、カメラ空間と画面空間の原点は違います。画面はビューポートに対応するので、カメラ/投影空間での画面の原点は **a**<sub>sp</sub> = (_L, T, −D_) です。ここで符号を逆にしないように気をつけてください。それがバグ祭りその 2 になります。また、これはカメラ空間でのことなので、_L_ は負で _T_ は正であることも忘れないでください。縦の軸が反転していることと、画面空間の原点の両方を考慮に入れると、次のようになります。

<table id="eq:p2s">
  <tr>
    <td class="eqnrcell">({!@eq:p2s})
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>S</mi>
                  <mo stretchy="false">(</mo>
                  <mn>1</mn>
                  <mo>,</mo>
                  <mo>&#x2212;</mo>
                  <mn>1</mn>
                  <mo>,</mo>
                  <mn>1</mn>
                  <mo stretchy="false">)</mo>
                  <mo>&#x22C5;</mo>
                  <msub>
                    <mi>x</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>s</mi>
                    </mrow>
                  </msub>
                  <mo>=</mo>
                  <msub>
                    <mi>x</mi>
                    <mi>p</mi>
                  </msub>
                  <mo>&#x2212;</mo>
                  <msub>
                    <mi>a</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>s</mi>
                      <mi>p</mi>
                    </mrow>
                  </msub>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
</table>

拡大縮小の行列は y 軸の符号を反転させます。カメラの座標系をさらに 180° 回転させていれば、この余分な行列は避けられました。その場合、**v** は下を、**w** は前を向いていたはずです。でもそうしなかったので、ここではこれで我慢しなければなりません。また、カメラ空間での画面の原点は **a**<sub>sp</sub> = (_L, T, −D_) なので、画面の位置は **x**<sub>s</sub> = (_x_<sub>s</sub>, *y*<sub>s</sub>, 0)、言い換えると _z_<sub>s</sub> は常にゼロです。すべてが正しいかどうかを確かめたければ、ビューポートの角が正しい画面の座標になるかを見てみてください。

### 理論のまとめ {#ssec-try-sum}

そして、基本的にはこれで全部です。ふう。ここまで来るのに 3 ページかかったので、最も重要なことを繰り返しておきます。まず、必要な主な式は次のとおりです。

<table id="eq:m7-main">
<col span=2 align="right">
<tr>
  <td class="eqnrcell">({!@eq:m7-main}a)</td>
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mi>&#x3BB;</mi>
                <mi>C</mi>
                <mtext>&#xA0;</mtext>
                <mo>&#x22C5;</mo>
                <mtext>&#xA0;</mtext>
                <msub>
                  <mi>x</mi>
                  <mi>p</mi>
                </msub>
                <mo>=</mo>
                <msub>
                  <mi>x</mi>
                  <mi>w</mi>
                </msub>
                <mo>&#x2212;</mo>
                <msub>
                  <mi>a</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>c</mi>
                    <mi>w</mi>
                  </mrow>
                </msub>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
  </td>
</tr>
<tr>
  <td class="eqnrcell">({!@eq:m7-main}b)</td>
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mi>S</mi>
                <mo stretchy="false">(</mo>
                <mn>1</mn>
                <mo>,</mo>
                <mo>&#x2212;</mo>
                <mn>1</mn>
                <mo>,</mo>
                <mn>1</mn>
                <mo stretchy="false">)</mo>
                <mo>&#x22C5;</mo>
                <msub>
                  <mi>x</mi>
                  <mi>s</mi>
                </msub>
                <mo>=</mo>
                <mo stretchy="false">(</mo>
                <msub>
                  <mi>x</mi>
                  <mi>p</mi>
                </msub>
                <mo>&#x2212;</mo>
                <msub>
                  <mi>a</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>s</mi>
                    <mi>p</mi>
                  </mrow>
                </msub>
                <mo stretchy="false">)</mo>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
  </td>
</table>

ここで

<div class="lblock">
  <table cellpadding=0>
    <tr>
      <td width="8%"><b>x</b><sub>w</sub>
      <td>ワールド空間での座標。
    <tr>
      <td><b>x</b><sub>p</sub>
      <td> 投影面上の座標。<b>x</b><sub>p</sub>=
      (<i>x</i><sub>p</sub>, <i>y</i><sub>p</sub>, &minus;<i>D</i>)。
    <tr>
      <td><b>x</b><sub>s</sub>
      <td>画面上の座標。<b>x</b><sub>s</sub>=
      (<i>x</i><sub>s</sub>, <i>y</i><sub>s</sub>, 0)。
    <tr>
      <td><b>a</b><sub>cw</sub>
      <td>ワールド空間でのカメラの位置。
    <tr>
      <td><b>a</b><sub>sp</sub>
      <td>カメラ空間での画面の原点の位置。
      <b>a</b><sub>sp</sub>&nbsp;=&nbsp;(<i>L, T, &minus;D</i>)。
    <tr>
      <td><b>C</b>
      <td>ピッチ &theta; とヨー &phi; の関数としてのカメラ行列。
      <b>C</b> =
      <b>R</b><sub>y</sub>(&phi;)· <b>R</b><sub>x</sub>(&theta;)。
    <tr>
      <td>&lambda;
      <td>スケール係数。その値は
      境界条件で決まります。
  </table>
</div>

これらの式と用語はしょっちゅう参照するので、覚えておいてください。{@eq:m7-main}a と {@eq:m7-main}b を分けているのは意図的なものです。本当の情報はすべて {@eq:m7-main}a にあり、{@eq:m7-main}b は変換を完了させるために必要な最後の段階にすぎません。本文の残りでは、{@eq:m7-main}a を頻繁に使い、必要でない限り {@eq:m7-main}b は省きます。ほかに知っておくと面白いこと:

- ワールド座標系 _S_<sub>w</sub> = {**i**,**j**,**k**} とカメラ座標系 _S_<sub>c</sub> = {**u**, **v**, **w**} は右手系のデカルト座標系です。予想どおり、カメラ行列 **C** の列は _S_<sub>c</sub> の主軸です。**C** = \[**u v w**\]。
- ビューポートと視錐台はカメラ空間にあり、その境界もそうです。つまり次のようになります。
  <table>
    <tr><td><i>R</i> &gt; 0 &gt; <i>L</i> <td>（横方向）
    <tr><td><i>T</i> &gt; 0 &gt; <i>B</i> <td>（縦方向）
    <tr><td>0 &gt; &minus;<i>N</i> &gt; &minus;<i>F</i> <td>（奥行き）
  </table>
- GBA の画面の大きさ（*W* = 240、*H* = 160）と *D* = 256 を基準にするなら、視錐台の境界の妥当な値は次のとおりですが、望むならほかの値を選んでもかまいません（訳注: 表の R は 120 の誤りと思われます）。
  <table>
    <col span=1>
      <tr><td><i>L</i> = &minus;120  <td><i>R</i> = &minus;120
      <tr><td><i>T</i> =  80   <td><i>B</i> = &minus;80
      <tr><td><i>N</i> =  24   <td><i>F</i> = 1024
  </table>

## 地平線とバックドロップ {#sec-horz}

モード 7 の本質的な場合、つまり遠近法の床を考えてください。透視除算のせいで、床の遠くの部分は 1 本の線に近づいていきます。<dfn>地平線</dfn>です。マップは本当に床にすぎないので、地平線も本当にそれだけ、つまり 1 本の水平な線になります。その上の空間は普通は空っぽですが、少しでも味気なさを減らすために、<dfn>バックドロップ</dfn>を使います。遠くの環境のパノラマの眺めで、カメラの回転に合わせて動くものです。

### 地平線を求める {#sec-horz-find}

大まかに言えば、地平線は z = −∞ のところです。床に線が引いてあれば、地平線はすべての平行線が交わるように見えるところ、つまり消失線です。当然、床しかないなら、床は地平線より下にだけ描くべきで、その上のグラフィックはスカイボックスの一部にするべきです。初代のマリオカートやほかのモード 7 のレースゲームで、これを見たことがあるはずです。ロールのないカメラに限っているので、地平線は常に水平な線、つまり 1 本のスキャンライン _y_<sub>s,h</sub> になります。それを求めるには、{@eq:m7-main}a の _y_ 成分を取り出して項を並べ替えるだけです。

<table id="eq:horz-line">
<tr>
  <td class="eqnrcell">({!@eq:horz-line}a)
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                  <mtr>
                    <mtd>
                      <mi>&#x3BB;</mi>
                      <mo stretchy="false">(</mo>
                      <msub>
                        <mi>v</mi>
                        <mi>y</mi>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <msub>
                        <mi>y</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>p</mi>
                          <mo>,</mo>
                          <mi>h</mi>
                        </mrow>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>w</mi>
                        <mi>y</mi>
                      </msub>
                      <mi>D</mi>
                      <mo stretchy="false">)</mo>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                          <mo>,</mo>
                          <mi>y</mi>
                        </mrow>
                      </msub>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd style="text-align: left;">
                      <msub>
                        <mi>y</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>p</mi>
                          <mo>,</mo>
                          <mi>h</mi>
                        </mrow>
                      </msub>
                    </mtd>
                    <mtd>
                      <mi></mi>
                      <mo>=</mo>
                      <mo stretchy="false">(</mo>
                      <msub>
                        <mi>w</mi>
                        <mi>y</mi>
                      </msub>
                      <mi>D</mi>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                          <mo>,</mo>
                          <mi>y</mi>
                        </mrow>
                      </msub>
                      <mrow data-mjx-texclass="ORD">
                        <mo>/</mo>
                      </mrow>
                      <mi>&#x3BB;</mi>
                      <mo stretchy="false">)</mo>
                      <mrow data-mjx-texclass="ORD">
                        <mo>/</mo>
                      </mrow>
                      <msub>
                        <mi>v</mi>
                        <mi>y</mi>
                      </msub>
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

そして、地平線を無限遠に取るなら λ = −∞ となり、@eq:horz-line は次のように簡単になります。

<table>
<tr>
  <td class="eqnrcell">({!@eq:horz-line}b)
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <msub>
                  <mi>y</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>p</mi>
                    <mo>,</mo>
                    <mi>h</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <mi>D</mi>
                <mtext>&#xA0;</mtext>
                <msub>
                  <mi>w</mi>
                  <mi>y</mi>
                </msub>
                <mrow data-mjx-texclass="ORD">
                  <mo>/</mo>
                </mrow>
                <msub>
                  <mi>v</mi>
                  <mi>y</mi>
                </msub>
                <mo>=</mo>
                <mi>D</mi>
                <mi>tan</mi>
                <mo data-mjx-texclass="NONE">&#x2061;</mo>
                <mrow>
                  <mo data-mjx-texclass="OPEN">(</mo>
                  <mi>&#x3B8;</mi>
                  <mo data-mjx-texclass="CLOSE">)</mo>
                </mrow>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
</table>

しかし、この簡単にした式を使いたいかどうかはよく考える必要があります。λ がとても大きいと、表示されるマップの点の隙間が大きくなりすぎて、実質的にノイズを表示していることになり、それは本当にとても見苦しくなりえます。もっと良い方法は、_z_<sub>c</sub> = −*F* にあるファーのクリッピング面を使うことです。その場合は λ = *F/D* となり、@eq:horz-line を使って地平線を計算できます。それは次のようなものになります。

<table>
<tr>
  <td class="eqnrcell">({!@eq:horz-line}c)
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <msub>
                  <mi>y</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>p</mi>
                    <mo>,</mo>
                    <mi>h</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <mi>D</mi>
                <mrow data-mjx-texclass="ORD">
                  <mo>/</mo>
                </mrow>
                <mi>F</mi>
                <mtext>&#xA0;</mtext>
                <mo>&#x22C5;</mo>
                <mtext>&#xA0;</mtext>
                <mo stretchy="false">(</mo>
                <mi>F</mi>
                <msub>
                  <mi>w</mi>
                  <mi>y</mi>
                </msub>
                <mtext>&#xA0;</mtext>
                <mo>&#x2212;</mo>
                <mtext>&#xA0;</mtext>
                <msub>
                  <mi>a</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>c</mi>
                    <mi>w</mi>
                    <mo>,</mo>
                    <mi>y</mi>
                  </mrow>
                </msub>
                <mo stretchy="false">)</mo>
                <mrow data-mjx-texclass="ORD">
                  <mo>/</mo>
                </mrow>
                <msub>
                  <mi>v</mi>
                  <mi>y</mi>
                </msub>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
</table>

予想どおり、*F* = −∞ なら {@eq:horz-line}c は {@eq:horz-line}b になります。_z_<sub>c</sub> を有限と無限のどちらにしても、地平線はスキャンライン _y_<sub>s,h</sub> = *T − y*<sub>p,h</sub> にあります。

### 地平線を使う {#ssec-horz-use}

地平線は、マップと「はるか遠く」の境目、つまり床とバックドロップの境目を示します。床は当然アフィン背景にするべきです。バックドロップには通常の背景を使いますが、それは必須ではありません。必要なのは、地平線のスキャンラインで 2 つを切り替える方法です。一番簡単なのは HBlank の割り込みです。地平線のスキャンラインに達したら、背景の制御レジスタで床とバックドロップの設定を切り替え、アフィンパラメータの転送に DMA を使うことにしたなら、HDMA を開始します。

バックドロップと床の背景を切り替えるのは、実は思ったよりやっかいです。たとえば、それぞれに別の背景を用意して、必要に応じて有効、無効にすることもできます。問題は、背景がハードウェアで完全に準備されるまでに約 3 スキャンラインかかるらしい（[forum:1303](https://gbadev.net/forum-archive/thread/4/1303.html) を見てください）ことで、その間はゴミが表示されてしまいます。つまり、この解決策はだめです。

別の方法は、両方に 1 つの背景を使い、ビデオモードを 0 から 1 か 2 に切り替えることです。これなら 3 ラインのゴミは出ませんが、今度は別の問題が出てきます。バックドロップと床とでは、タイルやマップの属性がまったく違う可能性がとても高いのです。でも、これは簡単に解決できます。`REG_BGxCNT` のスクリーン（とキャラ）のベースブロックを変えるだけです。

<div class="lblock">
  <div class="cpt" style="width:336px;">
    <img src="img/mode7/bg_switch.png" id="fig:img-bg-switch" alt="bg-switch">
    <br>
    <b>{*@fig:img-bg-switch}</b>: 地平線でビデオモードと背景のパラメータを切り替える。
  </div>

  <div class="cpt" style="width:450px;">
    <img src="img/mode7/panorama.png" id="fig:img-pan" alt="panorama" width=450>
    <br>
    <b>{*@fig:img-pan}</b>: 円筒からパノラマの眺めをはがす。
  </div>
</div>

### バックドロップを作って配置する {#ssec-horz-backdrop}

地平線のすぐ上の空間はバックドロップのためのものです。おそらく、ただの退屈な空っぽの空ではなく、遠くの町や並木の素敵な絵をそこに置きたいでしょう。バックドロップはパノラマの眺めを提供します。これは、円筒の内側に描いたマップを、普通の 2D の面にはがしたものと考えられます（@fig:img-pan を見てください）。考え方は、その面を背景に置いてスクロールさせるというものです。

縦方向では、背景の下端が地平線につながるようにするべきです。通常の背景は回り込む座標を使うので、これは実はとても簡単です。バックドロップの地面の高さをスクリーンブロックの一番下に置き、縦のオフセットを −*y*<sub>s,h</sub> にするだけです。

横方向では、気をつけるべき問題がいくつかあります。1 つ目はマップの幅で、これは単に円筒の周の長さ _P_ です。360° 回転したらマップの幅全体をスクロールしているはずなので、単位角度あたりの正しいスクロールの比率は、単に _P_/2π = *R*、つまり半径です。原理的には _R_ は任意ですが、パノラマの角度で作られる視野（α<sub>p</sub> = *W*/_R_）が、@eq:fov-cam のカメラの視野角 α<sub>c</sub> と等しいときに最良の結果が得られます。すべてが正しければ、α<sub>p</sub> = α<sub>c</sub> = α となるはずです。

<table id="eq:fov">
<tr>
  <td class="eqnrcell">({!@eq:fov})
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                  <mtr>
                    <mtd>
                      <mi>&#x3B1;</mi>
                    </mtd>
                    <mtd>
                      <mi></mi>
                      <mo>=</mo>
                      <mn>2</mn>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x22C5;</mo>
                      <mtext>&#xA0;</mtext>
                      <mi>arctan</mi>
                      <mo data-mjx-texclass="NONE">&#x2061;</mo>
                      <mrow>
                        <mo data-mjx-texclass="OPEN">(</mo>
                        <mfrac>
                          <mi>&frac12; W</mi>
                          <mrow>
                            <mi>D</mi>
                          </mrow>
                        </mfrac>
                        <mo data-mjx-texclass="CLOSE">)</mo>
                      </mrow>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>&#x3B1;</mi>
                    </mtd>
                    <mtd style="text-align: left">
                      <mi></mi>
                      <mo>=</mo>
                      <mfrac>
                        <mi>W</mi>
                        <mi>R</mi>
                      </mfrac>
                      <!-- <mrow data-mjx-texclass="ORD">
                        <mo>/</mo>
                      </mrow>
                      <mi>R</mi> -->
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>R</mi>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mfrac>
                        <mi>&frac12; W</mi>
                        <mrow>
                          <mi>arctan</mi>
                          <mo data-mjx-texclass="NONE">&#x2061;</mo>
                          <mrow>
                            <mo data-mjx-texclass="OPEN">(</mo>
                            <mfrac>
                              <mi>&frac12; W</mi>
                              <mi>D</mi>
                            </mfrac>
                            <mo data-mjx-texclass="CLOSE">)</mo>
                          </mrow>
                        </mrow>
                      </mfrac>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd></mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>&#x2248;</mo>
                      <mfrac>
                        <mi>D</mi>
                        <mrow>
                          <mn>1</mn>
                          <mo>&#x2212;</mo>
                          <mo stretchy="false">(</mo>
                          <mfrac>
                            <mi>&frac12; W</mi>
                            <mi>D</mi>
                          </mfrac>
                          <msup>
                            <mo stretchy="false">)</mo>
                            <mn>2</mn>
                          </msup>
                          <mtext>&#xA0;</mtext>
                          <mrow data-mjx-texclass="ORD">
                            <mo>/</mo>
                          </mrow>
                          <mtext>&#xA0;</mtext>
                          <mn>3</mn>
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

最後の近似は、アークタンジェントの[テイラー級数](https://en.wikipedia.org/wiki/Taylor's_theorem)の最初の数項から来ています。面白いことに、_R_ ≈ _D_ でもまずまず十分なようです。ともかく、*W* = 240 と *D* = 256 を入れると *P* = 1720 になりますが、これはあまり都合の良いマップの大きさではありませんね。さて、どんな大きさのマップでも作れて、スクリーンブロックの境界の外に出たら VRAM を更新することもできます（市販のゲームはいつもそうしています）が、そうすると今の話題から気がそれてしまうので、どうするかって？ 規則を少し曲げて、*P* = 1024 に決めてしまいます。

「ちょっと待って……そんなことしちゃだめでしょ！」 いや、実はできるのです。そうする_べき_ではないのですが、それは別の問題です。実際のところ、バックドロップを正しくスクロールしているモード 7 のゲームは_ひとつも_ないと思います！ たとえばマリオカートのシリーズは、バックドロップにスクロールの速さの違う複数の背景をよく使っていますが、見回しても相対的な視線は変わらないので、数学的に言えばまったくばかげています。でも、誰も気づかなかったか、少なくとも誰も気にしていないのでしょう。私が言いたいのは、私たちは良い仲間に囲まれているということです <kbd>:P</kbd>

そこで、周の長さの値と、それによってバックドロップのマップの幅を自分で決めてしまいます。この場合は *P* = 1024 を使います。これはきりのいい数で、幅 512 px のタイルマップを使えば、実質的に 180° の回転対称性を持つパノラマになります。円の分割 2π ⇔ 10000h を考慮に入れると、スクロールの値は単に φ\*_P_/10000h = φ/64 です。φ = 0 を真北に対応させたいので、これを _L_ だけずらす必要もあります。バックドロップの最終的な位置は {!@eq:bd-pos} で与えられます。

<table id="eq:bd-pos">
<tr>
  <td class="eqnrcell">({!@eq:bd-pos})
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                  <mtr>
                    <mtd>
                      <mi>d</mi>
                      <mi>x</mi>
                    </mtd>
                    <mtd>
                      <mi></mi>
                      <mo>=</mo>
                      <mi>&#x3C6;</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mo>/</mo>
                      </mrow>
                      <mn>64</mn>
                      <mo>+</mo>
                      <mi>L</mi>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>d</mi>
                      <mi>y</mi>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>y</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>s</mi>
                          <mo>,</mo>
                          <mi>h</mi>
                        </mrow>
                      </msub>
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

## 床 {#sec-flr}

### 床のアフィンパラメータ {#ssec-flr-parms}

{\*@eq:m7-main} はワールド↔画面の変換を表していますが、その情報は 3D のベクトルを使っています。一方 GBA で使えるのは、2×2 のアフィン行列 **P** と 2D の変位ベクトル **dx** だけです。ですから、少し書き直す必要があります。さて、2D↔3D の変換なども含めて完全な導出を示すこともできますが、本当は見たくないだろうという気がします。そこで代わりに、解く必要のある連立方程式と、その解き方のヒントを示します。

<table id="eq:m7-set">
<tr>
  <td class="eqnrcell">({!@eq:m7-set})
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                  <mtr>
                    <mtd>
                      <mi>&#x3BB;</mi>
                      <mi>C</mi>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x22C5;</mo>
                      <mtext>&#xA0;</mtext>
                      <msub>
                        <mi>x</mi>
                        <mi>p</mi>
                      </msub>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <msub>
                        <mi>x</mi>
                        <mi>w</mi>
                      </msub>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                        </mrow>
                      </msub>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>S</mi>
                      <mo stretchy="false">(</mo>
                      <mn>1</mn>
                      <mo>,</mo>
                      <mo>&#x2212;</mo>
                      <mn>1</mn>
                      <mo>,</mo>
                      <mn>1</mn>
                      <mo stretchy="false">)</mo>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x22C5;</mo>
                      <mtext>&#xA0;</mtext>
                      <msub>
                        <mi>x</mi>
                        <mi>s</mi>
                      </msub>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mo stretchy="false">(</mo>
                      <msub>
                        <mi>x</mi>
                        <mi>p</mi>
                      </msub>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>s</mi>
                          <mi>p</mi>
                        </mrow>
                      </msub>
                      <mo stretchy="false">)</mo>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>P</mi>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x22C5;</mo>
                      <mtext>&#xA0;</mtext>
                      <mi>q</mi>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mi>p</mi>
                      <mo>&#x2212;</mo>
                      <mi>d</mi>
                      <mi>x</mi>
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

最初の 2 つの式は {@eq:m7-main} をもう一度書いたもので、完全を期して挙げているだけです。最後の式は、アフィンのマップでの画面の点 **q** とマップの点 **p** の関係で、もうおなじみの式のはずです。さて、マップは床にあること、言い換えると **p** = (_x_<sub>w</sub>, *z*<sub>w</sub>) であることを思い出してください。2D の画面の点 **q** は、もちろん 3D の画面のベクトル **x**<sub>s</sub> に似たものです。覚えておく必要があるのは、`REG_BGxY` に書き込むとき、現在のスキャンラインの左端が原点として扱われるので、実質的に **q** = (_x_<sub>s</sub>, 0) となり、そのため _p_<sub>b</sub> と _p_<sub>d</sub> は何の影響も持たないということだけです。**P** のほかの要素の値は、単にスケールしたカメラの x 軸 λ**u** の _x_ 成分と _z_ 成分です。これらの値を使うと、最終的には次のようにまとめるのが一番よい式にたどり着くことがわかるでしょう。

<table id="eq:m7-ofs">
<tr>
  <td class="eqnrcell">({!@eq:m7-ofs})
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                  <mtr>
                    <mtd>
                      <mi>d</mi>
                      <msup>
                        <mi>x</mi>
                        <mo data-mjx-alternate="1">&#x2032;</mo>
                      </msup>
                      <mo>=</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                        </mrow>
                      </msub>
                      <mo>+</mo>
                      <mi>&#x3BB;</mi>
                      <mi>C</mi>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x22C5;</mo>
                      <mtext>&#xA0;</mtext>
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
</table>

ここで

<div class="lblock">
  <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
    <mstyle displaystyle="true" scriptlevel="0">
      <mrow data-mjx-texclass="ORD">
        <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
          <mtr>
            <mtd>
              <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                <mtr>
                  <mtd>
                    <mi>d</mi>
                    <msup>
                      <mi>x</mi>
                      <mo data-mjx-alternate="1">&#x2032;</mo>
                    </msup>
                  </mtd>
                  <mtd style="text-align: left;">
                    <mi></mi>
                    <mo>=</mo>
                    <mo stretchy="false">(</mo>
                    <mi>d</mi>
                    <mi>x</mi>
                    <mo>,</mo>
                    <mtext>&#xA0;</mtext>
                    <mn>0</mn>
                    <mo>,</mo>
                    <mtext>&#xA0;</mtext>
                    <mi>d</mi>
                    <mi>y</mi>
                    <mo stretchy="false">)</mo>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mi>b</mi>
                  </mtd>
                  <mtd>
                    <mi></mi>
                    <mo>=</mo>
                    <mo stretchy="false">(</mo>
                    <mi>L</mi>
                    <mo>,</mo>
                    <mtext>&#xA0;</mtext>
                    <mi>T</mi>
                    <mo>&#x2212;</mo>
                    <msub>
                      <mi>y</mi>
                      <mrow data-mjx-texclass="ORD">
                        <msup>
                          <mi>s</mi>
                          <mo data-mjx-alternate="1">&#x2032;</mo>
                        </msup>
                      </mrow>
                    </msub>
                    <mo>,</mo>
                    <mo>&#x2212;</mo>
                    <mi>D</mi>
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
</div>

変位に必要なものはすべて、この 1 つの式にきれいに詰め込まれています。次はこれを分解して、アルゴリズムを組み立てる必要があります。まず、**dx'** の _y_ 成分を使って λ を計算できます。それが得られれば、それを使って残りの 2 つの要素、つまり実際のアフィンのオフセットを計算できます。アフィン行列はすでに先ほど示しました。

{\*@eq:m7-sum} はすべての関係を明示的に示しています。もっとも、私自身は {@eq:m7-ofs} の簡潔さのほうが好きなのを許してもらえるとうれしいです。

<table id="eq:m7-sum">
<col span=2 align="right">
<tr>
  <td rowspan=5 class="eqnrcell">({!@eq:m7-sum})
  <td>
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                  <mtr>
                    <mtd>
                      <mi>&#x3BB;</mi>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                          <mo>,</mo>
                          <mi>y</mi>
                        </mrow>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mrow data-mjx-texclass="ORD">
                        <mo>/</mo>
                      </mrow>
                      <mtext>&#xA0;</mtext>
                      <mo stretchy="false">(</mo>
                      <mo stretchy="false">(</mo>
                      <msub>
                        <mi>y</mi>
                        <mi>s</mi>
                      </msub>
                      <mo>&#x2212;</mo>
                      <mi>T</mi>
                      <mo stretchy="false">)</mo>
                      <msub>
                        <mi>v</mi>
                        <mi>y</mi>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>+</mo>
                      <mi>D</mi>
                      <msub>
                        <mi>w</mi>
                        <mi>y</mi>
                      </msub>
                      <mo stretchy="false">)</mo>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <msub>
                        <mi>p</mi>
                        <mi>a</mi>
                      </msub>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mi>&#x3BB;</mi>
                      <mtext>&#xA0;</mtext>
                      <msub>
                        <mi>u</mi>
                        <mi>x</mi>
                      </msub>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <msub>
                        <mi>p</mi>
                        <mi>c</mi>
                      </msub>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <mi>&#x3BB;</mi>
                      <mtext>&#xA0;</mtext>
                      <msub>
                        <mi>u</mi>
                        <mi>z</mi>
                      </msub>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>d</mi>
                      <mi>x</mi>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                          <mo>,</mo>
                          <mi>x</mi>
                        </mrow>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>+</mo>
                      <mtext>&#xA0;</mtext>
                      <mi>&#x3BB;</mi>
                      <mo stretchy="false">(</mo>
                      <mi>L</mi>
                      <msub>
                        <mi>u</mi>
                        <mi>x</mi>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>+</mo>
                      <mtext>&#xA0;</mtext>
                      <mo stretchy="false">(</mo>
                      <mi>T</mi>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>y</mi>
                        <mi>s</mi>
                      </msub>
                      <mo stretchy="false">)</mo>
                      <msub>
                        <mi>v</mi>
                        <mi>x</mi>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>&#x2212;</mo>
                      <mi>D</mi>
                      <msub>
                        <mi>w</mi>
                        <mi>x</mi>
                      </msub>
                      <mo stretchy="false">)</mo>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd>
                      <mi>d</mi>
                      <mi>y</mi>
                    </mtd>
                    <mtd style="text-align: left;">
                      <mi></mi>
                      <mo>=</mo>
                      <msub>
                        <mi>a</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>c</mi>
                          <mi>w</mi>
                          <mo>,</mo>
                          <mi>z</mi>
                        </mrow>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>+</mo>
                      <mtext>&#xA0;</mtext>
                      <mi>&#x3BB;</mi>
                      <mo stretchy="false">(</mo>
                      <mi>L</mi>
                      <msub>
                        <mi>u</mi>
                        <mi>z</mi>
                      </msub>
                      <mtext>&#xA0;</mtext>
                      <mo>+</mo>
                      <mtext>&#xA0;</mtext>
                      <mo stretchy="false">(</mo>
                      <mi>T</mi>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>y</mi>
                        <mi>s</mi>
                      </msub>
                      <mo stretchy="false">)</mo>
                      <msub>
                        <mi>v</mi>
                        <mi>z</mi>
                      </msub>
                      <mo>&#x2212;</mo>
                      <mi>D</mi>
                      <msub>
                        <mi>w</mi>
                        <mi>z</mi>
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

上端を 0 にしてピッチをなくす（_T_=0 かつ θ=0）と、最初のモード 7 の章とまったく同じ結果になることに注意してください。また、真下を見る（θ=90°）と、全体が点 (−*L, T*) を中心とした単純な拡大縮小と回転になり、まさにそうなるべき形になります。{\*@eq:m7-sum} はモード 7 の一般の式です。実装では、計算を速くする近道がいくつも使えることがよくありますが、それは[あとで](#eq-aff-calc)扱います。

### 距離に応じたフォグ {#ssec-flr-fog}

現実の世界では、遠くの物体から来る光は大気の中を通らなければならず、大気が光子を散乱させて光線を弱めます。最終的に見えるのは、一部は物体そのもの、一部は周囲の色で、元の物体が遠いほど、その寄与は小さくなります。この効果は霧の状態で一番見えやすいので、これを<dfn>フォグ</dfn>と呼ぶことにします。

フォグは距離の手がかりになり、これを入れると奥行きの感覚が増します。また、読み込まれて視界に急に現れる物体を隠すこともできます。GBA では、スキャンラインごとに違うアルファブレンドを使うことで実装できます。

その基本の式は、次の微分方程式です。

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
              <mtr>
                <mtd>
                  <mi>d</mi>
                  <mi>I</mi>
                  <mo>=</mo>
                  <mo>&#x2212;</mo>
                  <mi>I</mi>
                  <mtext>&#xA0;</mtext>
                  <mi>k</mi>
                  <mo stretchy="false">(</mo>
                  <mi>&#x3BD;</mi>
                  <mo stretchy="false">)</mo>
                  <mtext>&#xA0;</mtext>
                  <mi>&#x3C1;</mi>
                  <mtext>&#xA0;</mtext>
                  <mi>d</mi>
                  <mi>z</mi>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>

ここで _I_ は強度、_k_(ν) は媒質の吸収係数で、光の周波数 ν と、場合によっては位置に依存します。ρ は密度、_z_ は距離です。これを解くと、距離に対する指数関数的な減衰になります。それも、2 乗やら平方根やらが全部そろった、本当の距離での話です。

幸い、そんなに複雑なものを使う必要はありません。本当に必要なのは、無限遠で 0、近くで 1 になる何らかの関数の関係だけです。面白いことに、そういうものはすでにあります。スキャンラインの関数としての λ です（{!@eq:m7-sum} を見てください）。これは基本的に双曲線なので、あとはスケールやオフセットを少しいじって見栄えの良いものにするだけです。私の場合は、λ\*6/16 で十分うまくいくようです。

<div class="cblock">
  <table class="bdr" id="fig:img-fog" border=0 cellpadding=2 cellspacing=2>
    <tr>
      <td> <img src="img/mode7/fog_off.png" alt="fog off"> </td>
      <td> <img src="img/mode7/fog_on.png" alt="fog on"> </td>
    </tr>
    <tr>
      <td>
        <b>{*@fig:img-fog}</b>: フォグなし（左）とあり（右）。
      </td>
    </tr>
  </table>
</div>

{\*@fig:img-fog} は、かなり高い高度から見た、フォグの効果ありとなしのスクリーンショットです。画面の下のほうでは床までの距離が比較的近いので、まだはっきり見えます。地平線では、床はオレンジ色のフォグに完全に覆われます。地平線の近くのラインはどのみちたいして見栄えのするものではないので、これは実は良いことです。

ところで、_オレンジ色の_フォグと言ったことに注意してください。[グラフィック効果](gfx.html#sec-blend)の章に注意を払っていたなら、GBA には白と黒へのフェードのモードしかないことを知っているはずです。それでも、任意の色へのフェードは十分に可能です。それについては実装のところで説明します。どうやってできるのか考えてもらっている間に、3D のスプライトに進みます。

## スプライト {#sec-objs}

スプライトと 3D は奇妙な組み合わせです。本来、スプライトは 2D の物体で、ビューポート（つまり画面）に貼り付けたステッカーのようなものです。それを 3D の世界の一部に見せるには、世界と一緒に動いているように見えるように画面上を動かし、距離に応じて拡大縮小しなければなりません。ここでもその基本は @eq:m7-main ですが、話はそれだけではありません。

ここでは 4 つの話題を扱う必要があります。1 つ目はスプライトの**位置合わせ**です。{\*@eq:m7-main} は点/ピクセルの単位で働きますが、スプライトは単純な長方形です。それを回避するようにスプライトのピクセルを書き換えることもできますが、それではそもそもスプライトを使う意味がなくなってしまいます。代わりに、物体の 1 つの点をスプライトのワールド座標に結びつけ、それに合わせて OAM の位置と行列を設定します。これは基本的に、アフィンオブジェクトの章で説明した[アンカリング](affobj.html#sec-combo)の理論です。

次は、スプライトの**カリング**です。正しい OAM の位置が得られても、それをそのまま使うことはできません。スプライトが実際にビューポートの中に見えている場合にだけ有効になるようにしなければなりません。見えていなければ、無効にするべきです。

それから、スプライトの**アニメーション**の問題があります。{@fig:img-m7-obj} のキノピオのカートを考えてみてください。アンカーの位置は正しいのですが、どの角度から見ても、いつも同じ側が見えてしまいます。物体の周りを実際に動けるように見せるため、アニメーションの別々のフレームを使って違う側を見せます。

最後に、スプライトの**ソート**です。デフォルトでは、オブジェクトはオブジェクトの番号の順に並びます。obj 0 が obj 1 の上に、obj 1 が obj 2 の上に、という具合です。スプライトを常に同じオブジェクトに結びつけていると、反対側から見たときに順序が間違ってしまうので、距離でソートする必要があります。

以上が扱うべき主な問題です。ほかにも、影を置くことや、アフィン行列が 32 個までというハードウェアの制限を回避するために前もって拡大縮小したオブジェクトを使うことなどがありますが、ほかの点がすでに片づいていれば、これらはかなり簡単です。もう 1 つ説明するのは、私がオブジェクトの**正規化**と呼んでいるものです。オブジェクトがクリッピングの長方形に対して大きくなりすぎないように、追加の拡大縮小をかけることです。

<table id="fig:img-m7-obj" class="bdr" style="width:512px; margin:10px auto;" border=0 cellpadding=2 cellspacing=0>
  <tr>
    <td><img src="img/mode7/obj_back.png" alt="normal view"></td>
    <td><img src="img/mode7/obj_down.png" alt="looking down"></td>
    <td><img src="img/mode7/obj_right.png" alt="looking down and left"></td>
  </tr>
  <tr>
    <td colspan=3>
      <b>{*@fig:img-m7-obj}</b>: アンカーで固定したスプライト。位置は良いのですが、どう回り込んでも、キノピオはいつもそっぽを向いています。帽子のせいかもしれません。
    </td>
  </tr>
</table>

### 位置合わせとアンカリング {#ssec-obj-pos}

スプライトの位置合わせには 2 つの側面があります。1 つ目は、スプライトのワールドでの位置 **x**<sub>w</sub> を画面上の位置 **x**<sub>s</sub> に変換することです。そのあとで、その点を使って最も適切な OAM の座標を決める必要があります。

最初の部分は、{@eq:m7-main} のまた別の応用にすぎません。ただし逆向きです。普通、3D の行列の逆を求めるのは特に楽しくない作業ですが、カメラ行列はたまたま正規直交行列です。<dfn>正規直交行列</dfn>とは、成分のベクトルが直交していて（互いに垂直で）、長さが 1 の行列のことです。正規直交行列の素敵なところは、その逆行列が単に転置行列になることです。**C**<sup>−1</sup> = **C**<sup>T</sup> です。これで次の式が得られます。

<table id="eq:obj-w2s">
  <tr>
    <td class="eqnrcell">({!@eq:obj-w2s})
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                    <mtr>
                      <mtd>
                        <msub>
                          <mi>x</mi>
                          <mi>p</mi>
                        </msub>
                      </mtd>
                      <mtd style="text-align: left;">
                        <mi></mi>
                        <mo>=</mo>
                        <msup>
                          <mi>C</mi>
                          <mi>T</mi>
                        </msup>
                        <mtext>&#xA0;</mtext>
                        <mo>&#x22C5;</mo>
                        <mtext>&#xA0;</mtext>
                        <mo stretchy="false">(</mo>
                        <msub>
                          <mi>x</mi>
                          <mi>w</mi>
                        </msub>
                        <mo>&#x2212;</mo>
                        <msub>
                          <mi>a</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mi>c</mi>
                            <mi>w</mi>
                          </mrow>
                        </msub>
                        <mo stretchy="false">)</mo>
                        <mrow data-mjx-texclass="ORD">
                          <mo>/</mo>
                        </mrow>
                        <mi>&#x3BB;</mi>
                      </mtd>
                    </mtr>
                    <mtr>
                      <mtd>
                        <msub>
                          <mi>x</mi>
                          <mi>s</mi>
                        </msub>
                      </mtd>
                      <mtd>
                        <mi></mi>
                        <mo>=</mo>
                        <mi>S</mi>
                        <mo stretchy="false">(</mo>
                        <mn>1</mn>
                        <mo>,</mo>
                        <mo>&#x2212;</mo>
                        <mn>1</mn>
                        <mo>,</mo>
                        <mn>1</mn>
                        <mo stretchy="false">)</mo>
                        <mtext>&#xA0;</mtext>
                        <mo>&#x22C5;</mo>
                        <mtext>&#xA0;</mtext>
                        <mo stretchy="false">(</mo>
                        <msub>
                          <mi>x</mi>
                          <mi>p</mi>
                        </msub>
                        <mo>&#x2212;</mo>
                        <msub>
                          <mi>a</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mi>s</mi>
                            <mi>p</mi>
                          </mrow>
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

ここで本当にわからないのは λ だけで、これは _z_<sub>p</sub> = −*D* という事実を使って計算できます。さて、カメラとスプライトの間の距離を **r** = **x**<sub>w</sub> − **a**<sub>cw</sub> とします。**C** = \[**u v w**\] を使うと、次のようになります。

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
              <mtr>
                <mtd>
                  <mi>&#x3BB;</mi>
                </mtd>
                <mtd>
                  <mi></mi>
                  <mo>=</mo>
                  <mo>&#x2212;</mo>
                  <mi>w</mi>
                  <mo>&#x22C5;</mo>
                  <mi>r</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>D</mi>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <msub>
                    <mi>x</mi>
                    <mi>p</mi>
                  </msub>
                </mtd>
                <mtd style="text-align: left;">
                  <mi></mi>
                  <mo>=</mo>
                  <mi>u</mi>
                  <mo>&#x22C5;</mo>
                  <mi>r</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>&#x3BB;</mi>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <msub>
                    <mi>y</mi>
                    <mi>p</mi>
                  </msub>
                </mtd>
                <mtd style="text-align: left;">
                  <mi></mi>
                  <mo>=</mo>
                  <mi>v</mi>
                  <mo>&#x22C5;</mo>
                  <mi>r</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>&#x3BB;</mi>
                </mtd>
              </mtr>
            </mtable>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>

<div class="cpt_fr" style="width:88px;">
  <img src="img/mode7/anchor.png" id="fig:img-anchor" alt="a sprite, with anchor">
  <br>
  <b>{*@fig:img-anchor}</b>: 32&times;32 のスプライト。アンカー <b>p</b><sub>0</sub> は左上からの相対位置です。
</div>

そのあとで **x**<sub>w</sub> の画面上の位置を求めるのは簡単です。そして、ここからがアンカリングの部分です。ステッカーの代わりに、物体をボード（つまり画面）に画鋲で留める紙切れだと考えてください。画鋲は物体の 1 か所を貫き、その場所がボードに固定されます。その場所が<dfn>アンカー</dfn>です。アフィンオブジェクトではそれほど単純ではありません。アンカーの座標ではなく OAM の座標を指定しなければならないので、OAM の座標 **x** をテクスチャのアンカー **p**<sub>0</sub> と画面のアンカー **q**<sub>0</sub> でどう表すかに、少し数学がからみます。この理論は[アフィンオブジェクト](affobj.html#sec-combo)の章で扱い、@eq:anchor にたどり着きました。そこにあるほかの量は、オブジェクトの大きさ **s** = (_w_, *h*) と、通常のアフィンオブジェクトでは ½、倍サイズのアフィンオブジェクトでは 1 になる _m_ です。

<table id="eq:anchor">
  <tr>
    <td class="eqnrcell">({!@eq:anchor})
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                    <mtr>
                      <mtd>
                        <mi>x</mi>
                        <mo>=</mo>
                        <msub>
                          <mi>q</mi>
                          <mn>0</mn>
                        </msub>
                        <mo>&#x2212;</mo>
                        <mi>m</mi>
                        <mi>s</mi>
                        <mo>&#x2212;</mo>
                        <msup>
                          <mi>P</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mo>&#x2212;</mo>
                            <mn>1</mn>
                          </mrow>
                        </msup>
                        <mtext>&#xA0;</mtext>
                        <mo>&#x22C5;</mo>
                        <mtext>&#xA0;</mtext>
                        <mo stretchy="false">(</mo>
                        <msub>
                          <mi>p</mi>
                          <mn>0</mn>
                        </msub>
                        <mo>&#x2212;</mo>
                        <mi>&frac12; s</mi>
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

さて、することは、手元のデータをこの式に結びつけることです。画面のアンカー **q**<sub>0</sub> は単に **x**<sub>s</sub> です。テクスチャのアンカー **p**<sub>0</sub> は、テクスチャ空間で固定しておきたいピクセルで、自分で選べます。カートのスプライトでは、@fig:img-anchor に描かれているように、カートの下のほうに置くのが理にかなっています。「ベクトル」**s** はオブジェクトの大きさで決まり、この場合は (32, 32) です。そして、ここでは常に倍サイズのオブジェクトを使うことにしているので、_m_=1 です。**P** 行列は、ほかのものも加えたいのでなければ、単なる λ による拡大縮小です。あとは数値を埋めるだけです。

### スプライトのカリング {#ssec-obj-cull}

<div class="cpt_fr" style="width:308px;">
  <img src="img/mode7/viewport_obj.png" id="fig:img-vp-obj" alt="">
  <br>
  <b>{*@fig:img-vp-obj}</b>: スプライト <i>a</i>、<i>b</i>、<i>c</i> がある視錐台。<i>b</i> と <i>c</i> は見え、<i>a</i> は見えません。
</div>

<dfn>カリング</dfn>とは、世界のうち見えない部分をすべて取り除く処理です。この場合は、視野の空間に入らないスプライトを取り除くことを意味します。これはとても賢いやり方で、スプライトではさらに理にかなっています。そうしないと、OAM は **x**<sub>s</sub> のありうる範囲に対応できないので、ひどくおかしなことになってしまうからです。

最初にすべきことは距離の確認でしょう。物体が遠すぎれば、見えるべきではありません。ニア面の距離の確認もしておくのが良い考えです。それから、ビューポートとの交差を調べる必要があります。各スプライトは投影面上のある長方形で囲まれていて、それが完全にビューポートの外にあれば、その物体は描画するべきではありません。

{\*@fig:img-vp-obj} にいくつかの例を示します。物体 _a_ と _b_ はすでに投影面に投影されています。物体 _a_ はビューポートの外にあるので、無効にするべきです。物体 _b_ は一部が見えているので、描画するべきです。物体 _c_ はまだ投影されていませんが、ニア面とファー面の間にあるので、少なくとも調べるべきです（そして、完全に見えていることがわかります）。

視野の空間の確認は、2D の投影空間よりも、3D のカメラ空間で行うほうが実は簡単です。物体の長方形は、**x**<sub>c</sub> = **C**<sup>T</sup>·**r**、アンカー **p**<sub>0</sub>、大きさ **s** から簡単に計算できます。ビューポートは λ で拡大縮小する必要があり、これで次の判定をすることになります。

<div class="lblock">
  <table id="tbl:culltest" class="table-data">
    <caption align="bottom">
      <b>{*@tbl:culltest}</b>: カメラ空間での物体の長方形とカリングの判定。符号に注意！
    </caption>
    <tbody>
      <tr>
        <th>&nbsp;</th><th>物体の位置</th><th>見える条件</th>
      </tr>
      <tr>
        <th> 奥行き </th>
        <td> 
          <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
            <mstyle displaystyle="true" scriptlevel="0">
              <mrow data-mjx-texclass="ORD">
                <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                  <mtr>
                    <mtd>
                      <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                        <mtr>
                          <mtd>
                            <mi>d</mi>
                            <mo>=</mo>
                            <mo>&#x2212;</mo>
                            <msub>
                              <mi>z</mi>
                              <mi>c</mi>
                            </msub>
                            <mo>=</mo>
                            <mi>w</mi>
                            <mo>&#x22C5;</mo>
                            <mi>r</mi>
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
        <td>
          <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
            <mstyle displaystyle="true" scriptlevel="0">
              <mrow data-mjx-texclass="ORD">
                <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                  <mtr>
                    <mtd>
                      <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                        <mtr>
                          <mtd>
                            <mi>N</mi>
                            <mo>&#x2264;</mo>
                            <mi>d</mi>
                            <mtext>&#xA0;</mtext>
                            <mi mathvariant="normal">&amp;</mi>
                            <mi mathvariant="normal">&amp;</mi>
                            <mtext>&#xA0;</mtext>
                            <mi>d</mi>
                            <mo>&lt;</mo>
                            <mi>F</mi>
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
      <tr>
        <th> 横方向 </th>
        <td> 
          <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
            <mstyle displaystyle="true" scriptlevel="0">
              <mrow data-mjx-texclass="ORD">
                <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                  <mtr>
                    <mtd>
                      <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                        <mtr>
                          <mtd>
                            <mi>l</mi>
                            <mo>=</mo>
                            <msub>
                              <mi>x</mi>
                              <mi>c</mi>
                            </msub>
                            <mo>&#x2212;</mo>
                            <msub>
                              <mi>p</mi>
                              <mrow data-mjx-texclass="ORD">
                                <mn>0</mn>
                                <mo>,</mo>
                                <mi>x</mi>
                              </mrow>
                            </msub>
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
        <td>
          <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
            <mstyle displaystyle="true" scriptlevel="0">
              <mrow data-mjx-texclass="ORD">
                <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                  <mtr>
                    <mtd>
                      <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                        <mtr>
                          <mtd>
                            <mi>&#x3BB;</mi>
                            <mi>L</mi>
                            <mo>&#x2264;</mo>
                            <mi>l</mi>
                            <mo>+</mo>
                            <mi>w</mi>
                            <mtext>&#xA0;</mtext>
                            <mi mathvariant="normal">&amp;</mi>
                            <mi mathvariant="normal">&amp;</mi>
                            <mtext>&#xA0;</mtext>
                            <mi>l</mi>
                            <mo>&lt;</mo>
                            <mi>&#x3BB;</mi>
                            <mi>R</mi>
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
      <tr>
        <th> 縦方向 </th>
        <td>
          <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
            <mstyle displaystyle="true" scriptlevel="0">
              <mrow data-mjx-texclass="ORD">
                <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                  <mtr>
                    <mtd>
                      <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                        <mtr>
                          <mtd>
                            <mi>t</mi>
                            <mo>=</mo>
                            <mo>&#x2212;</mo>
                            <msub>
                              <mi>y</mi>
                              <mi>c</mi>
                            </msub>
                            <mo>&#x2212;</mo>
                            <msub>
                              <mi>p</mi>
                              <mrow data-mjx-texclass="ORD">
                                <mn>0</mn>
                                <mo>,</mo>
                                <mi>y</mi>
                              </mrow>
                            </msub>
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
        <td>
          <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
            <mstyle displaystyle="true" scriptlevel="0">
              <mrow data-mjx-texclass="ORD">
                <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                  <mtr>
                    <mtd>
                      <mtable displaystyle="true" columnalign="right" columnspacing="" rowspacing="3pt">
                        <mtr>
                          <mtd>
                            <mi>&#x3BB;</mi>
                            <mi>T</mi>
                            <mtext>&#xA0;</mtext>
                            <mo>&#x2264;</mo>
                            <mtext>&#xA0;</mtext>
                            <mi>t</mi>
                            <mo>+</mo>
                            <mi>h</mi>
                            <mtext>&#xA0;</mtext>
                            <mi mathvariant="normal">&amp;</mi>
                            <mi mathvariant="normal">&amp;</mi>
                            <mtext>&#xA0;</mtext>
                            <mi>t</mi>
                            <mo>&lt;</mo>
                            <mo>&#x2212;</mo>
                            <mi>&#x3BB;</mi>
                            <mi>B</mi>
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
    </tbody>
  </table>
</div>

これらの条件がすべて真なら、その物体は見えるはずです。さて、判定の_符号_、特に縦方向の確認の符号に注意してください。

### アニメーション {#ssec-obj-ani}

<div class="cpt_fr" style="width:200px;">
  <img src="img/mode7/psi_def.png" id="fig:img-psi-def" alt="view angle">
  <br>
  <b>{*@fig:img-psi-def}</b>: 視線の角度 &psi; を求める。
</div>

正確には、回転のアニメーションです。{@fig:img-m7-obj} で見たように、スプライトはどこから見ても同じ側を見せます。スプライトは実際には 3D の存在ではないので、これは当然のことです。もう少し 3D らしく_見せる_には、違うカメラの角度から撮ったスプライトの画像を用意して、どの角度から見ているかに応じて必要なものを選ぶ必要があります。

まず、正しい視線の角度 ψ を求めます。{\*@fig:img-psi-def} は一般的な状況を示しています。必要な角度は、カメラと物体を結ぶベクトル（赤い破線）と、物体のグローバルな向きの間の角度です。図には、カメラと物体のグローバルな向きの角度、それぞれ φ<sub>c</sub> と φ<sub>o</sub> が描かれています。カメラの向きとスプライトの間の角度 α も示されています。これらの角度をよく見ると、φ<sub>c</sub> + α + ψ = φ<sub>o</sub> であることがわかるでしょう。言い換えると次のようになります。

<table id="eq:psi">
  <tr>
    <td class="eqnrcell">({!@eq:psi}) </td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                    <mtr>
                      <mtd>
                        <mi>&#x3C8;</mi>
                      </mtd>
                      <mtd style="text-align: left;">
                        <mi></mi>
                        <mo>=</mo>
                        <msub>
                          <mi>&#x3C6;</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mn>0</mn>
                          </mrow>
                        </msub>
                        <mo>&#x2212;</mo>
                        <msub>
                          <mi>&#x3C6;</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mi>c</mi>
                          </mrow>
                        </msub>
                        <mo>&#x2212;</mo>
                        <mi>&#x3B1;</mi>
                      </mtd>
                    </mtr>
                    <mtr>
                      <mtd></mtd>
                      <mtd>
                        <mi></mi>
                        <mo>=</mo>
                        <msub>
                          <mi>&#x3C6;</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mn>0</mn>
                          </mrow>
                        </msub>
                        <mo>&#x2212;</mo>
                        <msub>
                          <mi>&#x3C6;</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mi>c</mi>
                          </mrow>
                        </msub>
                        <mo>&#x2212;</mo>
                        <mi>arctan</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <msub>
                            <mi>x</mi>
                            <mi>c</mi>
                          </msub>
                          <mrow data-mjx-texclass="ORD">
                            <mo>/</mo>
                          </mrow>
                          <mo>&#x2212;</mo>
                          <msub>
                            <mi>z</mi>
                            <mi>c</mi>
                          </msub>
                          <mo data-mjx-texclass="CLOSE">)</mo>
                        </mrow>
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

arctan() の中のマイナスの符号が必要かどうかは、すべての項をどう定義するかによります。{\*@eq:psi} は完全に正しい版ですが、arctan が気に入らなければ、ほとんどの場合 α の項は誰にも気づかれずに安全に無視できると知ってうれしく思うでしょう。

視線の角度が得られたので、それを何らかの形で使う必要があります。回転のフレームが _N_ 枚あるとすると、円は 2π/_N_ ラジアンずつの等しい部分に分けられます。ψ がどの切れ端に入るかを得るには、各切れ端の角度で割るだけです。*i* = ψ/(2π/_N_) = *N*·ψ/(2π) です。円を 2 の累乗で分割して定義している（私たちはそうしています）なら、この部分はばかばかしいほど簡単です。右シフトを使うだけです。フレームのインデックスが得られれば、残りは簡単なはずです。だいたいは。話をわかりにくくする込み入った点もいくつかありますが、それらは実装に依存するので、あとに取っておきます。

### スプライトのソート {#ssec-obj-sort}

優先順位のビットはひとまず置いておくと、画面上のオブジェクトの順序はオブジェクトの番号で決まります。小さい番号が大きい番号の手前に来ます。2D のゲームでは、スプライトは同じレイヤーにあるので、これはたいてい無視できます。3D のゲームでは、本当に、本当に無視できません。たとえば {@fig:img-obj-sort} を見てください。ここにいる 4 体のドッスンには決まったオブジェクトの順序があります。左の絵では、たまたま一番近いドッスンが一番小さいオブジェクトの番号を持っていて、見た目の順序は正しくなっています。しかし、反対側から見ると（真ん中の絵）、事情は少し違います。奥行きの視覚的な手がかりは 2 つあります。拡大縮小（遠いものほど小さい）と遮蔽（遠くの物体は近くの物体に隠される）です。真ん中の絵では、一番近い物体が_一番大きい_番号を持っているので、この 2 つが矛盾し、全体の絵が少し落ち着かないものになっています。右の絵では、正しいオブジェクトの順序になるように手を打ったので、すべてがまた正しく見えます。

<table id="fig:img-obj-sort" class="bdr" style="width:512px; margin:10px auto;" border=0 cellpadding=2 cellspacing=2>
  <tr>
    <td><img src="img/mode7/order_dflt.png" alt="default order"></td>
    <td><img src="img/mode7/order_nosort.png" alt="non-sorted"></td>
    <td><img src="img/mode7/order_sorted.png" alt="sorted"></td>
  </tr>
  <tr>
    <td colspan=3>
      <b>{*@fig:img-obj-sort}</b>。ソートしていないオブジェクトは、ある角度からは問題なく見えます（左）が、反対から見るとそうではありません（真ん中）。正しい順序にするにはソートする必要があります（右）。
    </td>
  </tr>
</table>

必要なのは、OAM の中のオブジェクトを奥行きでソートすることです。オブジェクトのための一種の [Z バッファ](https://en.wikipedia.org/wiki/Z-buffering)です。スプライトの奥行きは単に _z_<sub>c</sub> で、_z_<sub>c</sub> の昇順に、スプライトのオブジェクト属性で OAM を埋める必要があります。念のため、隠れているオブジェクトには可能な最大の奥行きの値を与えるか、ソートの処理から完全に外しておくのがよいでしょう。

オブジェクトのソートの方法はたくさん考えられます。今の私の選択は、スプライトやオブジェクトを直接ソートするのではなく、スプライトの属性を OAM に入れる順序を示す**インデックスの表**を作ることです。その擬似コードを下に示します。キーのソートにどのアルゴリズムを使うかは、仕事をこなしさえすれば、今のところあまり重要ではありません。もっと速い方法もきっと見つかるでしょうが、おそらくコードが増えるという代償があり、私は話を比較的単純にしておきたいのです。

```c
// Pseudo code for sorting sprites for OAM
void spr_sort()
{
    int ids[N];     // Index table
    int keys[N];    // Sort keys

    // Create initial index and sort-key table
    for ii=0; ii<N; ii++)
    {
        ids[ii]= ii;
        keys[ii]= is_visible(sprite[ii]) ? sprite[ii].depth : DEPTH_MAX;
    }

    // Sort keys (i.e., fill ids)
    id_sort(ids, keys);

    // Fill OAM according to
    for(ii=0; ii<N; ii++)
        oam_mem[ii]= sprite[ids[ii]].obj;
}
```

### 再正規化 {#ssec-obj-norm}

この言葉をこれまで聞いたことがなくても驚きません。<dfn>正規化</dfn>とは、量を使いやすい値、普通は 1 に合わせて拡大縮小することです。スプライトはすでに係数 λ で拡大縮小していますが、それでは足りません。ほとんどの場合、さらに拡大縮小する、つまり_再正規化_する必要があります。その理由は次のとおりです。

定義により、スケール係数 λ は _z_<sub>c</sub> = −*D* のときに 1 になります。では、もっと近い物体、たとえば _z_<sub>c</sub> = −½*D* にある物体を見たらどうなるか考えてみてください。この場合 λ は ½ になり、物体は 2 倍に拡大されます。言い換えると、それだけで倍サイズのキャンバスをいっぱいに使ってしまいます。そして、もっと大きな拡大率も起こりえます。提案した値 *D* = 256 と *N* = 24 では、拡大率が 10 になることもあります！ これではだめです。

ニア面をもっと遠くに動かせば、これを回避できます。しかしそうすると、まだかなり遠くにある物体が消えてしまうのが見えることになり、切り取られるのを見るのと同じくらい変に見えます。もっと良い解決策は、物体に追加のスケール係数を与えることです。`m7_ex` では物体をさらに ¼ 倍に拡大縮小していて、32x32 のスプライトが実際には「ワールド」のピクセルで 8x8 の大きさしかないようにしています。これはかなりうまくいくようです。

この再正規化は、実際には_2 つの別々の_スケール係数を扱っていることを意味します。1 つは座標変換のため、もう 1 つは見た目の効果のためです。スプライトの位置合わせとカリングで使う必要があるのは、変換のスケールではなく_見た目の_スケールです。変換のスケールの影響は、アンカーの画面上の位置を求めたところで終わります。

この処理にはおそらく公式の名前があるのでしょうが、私は知りません。再正規化の処理は物理学で知っていて（数年前に、オランダの教授が何人か、この題材でノーベル賞を受けました）、ぴったりだと思ったのです。公式の名前を知っていたら、ぜひ教えてください。

<div class="cblock">
  <table id="fig:img-obj-norm" class="bdr" width=512 border=0 cellpadding=4 cellspacing=0>
    <tr>
      <td><img src="img/mode7/norm_1x.png" alt="norm 1x"></td>
      <td><img src="img/mode7/norm_2x.png" alt="norm 2x"></td>
      <td><img src="img/mode7/norm_4x.png" alt="norm 4x"></td>
    </tr>
    <tr>
      <td colspan=3>
        <b>{*@fig:img-obj-norm}</b>。オブジェクトの再正規化。左: 通常（うわっ、だめ！）。真ん中: &times;&frac12;（うーん、だめ）。右: &times;&frac14;（そう、これだ）。
      </td>
    </tr>
  </table>
</div>

これで理論は終わりです。次は、これを全部実際に実装します。

## 実装 {#sec-code}

### 設計の検討 {#ssec-code-design}

ここでの私の目的は、モード 7 を実現できる関数をいくつか出すだけでなく、必要なら簡単に変更できるものを提供することでもあります。`m7_ex` デモのコードは 4 つのファイルに分かれています。デモそのものに固有のもののための `m7_ex.c` と、モード 7 に固有のもののための 3 つ、`mode7.h`、`mode7.c`、`mode7.iwram.c` です。そう、IWRAM の関数もあります。これらの中には計算量の多いものがあり、最初からできるだけ速くしておきたいのです。オブジェクトのソートは[優先順位のデモ](lab.html##ssec-prio-objsort)から借りています。

主な関心事は 3 つあります。**カメラ**、**背景まわり**、**スプライト**です。それぞれについて、データを保持するために構造体や配列を使うので、オブジェクト指向っぽくて良い感じです。モード 7 全体のための、一種の管理用の構造体もあります。そしてもちろん、視野の空間、焦点距離、そのほかいくつかの項目のための定数が必要です。それから、一握りの関数がこれらの項目を操作して、必要なものを提供します。

#### 定数

定数はそれほど多くありません。ほとんどはビューポートに関するもので、残りは焦点と再正規化に関するものです。

```c
#define M7_D        256     //!< Focal length
#define M7_D_SHIFT    8     //!< Focal shift
#define M7O_NORM      2     //!< Object renormalization shift (by /4)

// View frustum limits
#define M7_LEFT     (-120)      //!< Viewport left
#define M7_RIGHT     120        //!< Viewport right
#define M7_TOP        80        //!< Viewport top (y-axis up)
#define M7_BOTTOM   (-80)       //!< Viewport bottom (y-axis up!)
#define M7_NEAR       24        //!< Near plane (objects)
#define M7_FAR       512        //!< Far plane (objects)

#define M7_FAR_BG    768        //!< Far plane (floor)
```

#### 構造体と変数 {#ssec-code-class}

モード 7 はクラスを使うのにうってつけの場所でしょうが、私は C++ ではなく C で書いているので、構造体にこだわります。[アフィン背景](affbg.html)のページで示した `BG_AFFINE` 構造体のほかに、カメラのための構造体と、モード 7 のオブジェクトのための構造体が必要です。また、モード 7 の機能に関わるすべての部品を管理するために、モード 7 の入れ物の構造体も使っています。そうすれば、グローバル変数があちこちに転がっていることもありません。

これらのために自分の構造体を作ってもかまいませんが、私が使うものを以下に示します。ここまで注意を払ってきたなら、メンバのほとんどはおなじみのはずです。そうそう、`POINT` 構造体と `VECTOR` 構造体は、もちろん 2D と 3D のベクトルです。

```c
//! 3D sprite struct
typedef struct M7_SPRITE
{
    VECTOR pos;     //!< World position.
    POINT anchor;   //!< Sprite anchor.
    OBJ_ATTR obj;   //!< Object attributes.
    s16 phi;        //!< Azimuth angle.
    u8 obj_id;      //!< Object index.
    u8 aff_id;      //!< OBJ_AFFINE index.
    TILE *tiles;    //!< Gfx pointer.
    VECTOR pos2;    //!< Position in cam space (subject to change)
} M7_SPRITE;

//! 3D camera struct
typedef struct M7_CAM
{
    VECTOR pos;     //!< World position.
    int theta;      //!< Polar angle.
    int phi;        //!< Azimuth angle.
    VECTOR u;       //!< local x-axis (right)
    VECTOR v;       //!< local y-axis (up)
    VECTOR w;       //!< local z-axis (back)
} M7_CAM;


//! One struct to bind them all
typedef struct M7_LEVEL
{
    M7_CAM *camera;         //!< Camera variables
    BG_AFFINE *bgaff;       //!< Affine parameter array
    M7_SPRITE *sprites;     //!< 3D sprites
    int horizon;            //!< Horizon scanline (sorta)
    u16 bgcnt_sky;          //!< BGxCNT for backdrop
    u16 bgcnt_floor;        //!< BGxCNT for floor
} M7_LEVEL;
```

これらの構造体について言うべきことは、それほど多くありません。`M7_SPRITE` は、何かのバッファへのインデックスやポインタではなく、オブジェクトの属性そのものをメンバとして持っています。その理由は、基本的には「別にいいじゃないか」です。どのみちオブジェクトをソートしなければならないので、余分なバッファを使う価値はあまりないかもしれず、こうすることにしました。また、カメラ空間での位置も保持しています。何度か必要になるからです。そしてグラフィックのための TILE のポインタもあります。その理由は、アニメーションを実装するときに明らかになります。

`M7_LEVEL` は、モード 7 の主な変数（カメラ、アフィンの配列、スプライト）へのポインタと、バックドロップから床に切り替えるのに必要な地平線のスキャンライン、そして背景の制御レジスタのデータを入れた 2 つの変数を持っています。背景の制御レジスタはバックドロップと床とで違うからです。

さて、これらの構造体を使った 4 つの変数が必要です。これらは厳密にはデモそのものの一部なので、モード 7 の主なコードではなく `m7_ex.c` に置きました。ただし、そのコードは HBlank の割り込みのために、実際の `m7_level` 変数が存在することを必要とします。`SPR_COUNT` はスプライトの数で、これは_間違いなく_デモ固有のものです。`m7_bgaffs` のエントリが 160 ではなく 161 個あるのは、[DMA のデモ](dma.html#sec-demo)と同じ理由です。HBlank は現在のラインではなく次のラインを準備するので、こうしておくほうが、if/else ブロックを使うほかの方法より良い（そして速い）のです。

```c
M7_CAM m7_cam;
BG_AFFINE m7_bgaffs[SCREEN_HEIGHT+1];
M7_SPRITE m7_sprites[SPR_COUNT];

M7_LEVEL m7_level;
```

:::tip 構造体のメンバの型と順序

私はいつも、データ型には int を使うようにすすめていますが、構造体では、それが常に最善とは限りません。ローカル変数はメモリを使わないかもしれませんが、構造体は使います。そして構造体の配列があると、ワードの大きさのメンバにかかる余分な領域はすぐにかさみます。ですからその場合は、遠慮なく int 以外を使ってください。

とはいえ、それらのメンバを使うときには、すべての計算にバイトやハーフワードのメンバを使うのではなく、そのデータを 32 ビットのローカル変数にコピーしたほうが得をすることがあります。

また、そしてこれは_とても_重要なのですが、メンバの順序に注意を払わなければ、領域はまったく節約できません。int は、バイトのメンバのすぐあとに来ても、やはりワード境界への整列が必要です。コンパイラは、次のメンバが正しく整列するように、バイトやハーフワードのあとに詰め物を加えることがあります。詰め物ができるだけ少なくなるようにメンバを並べるのが一番です。

:::

### 背景の関数 {#ssec-code-bg}

背景の主な関数は次の 4 つです。

- `void m7_prep_horizon(M7_LEVEL *level)`。地平線のスキャンラインを計算します。
- `IWRAM_CODE void m7_prep_affines(M7_LEVEL *level)`。カメラの位置と向きに基づいて、床のアフィンパラメータを計算します。
- `void m7_update_sky(const M7_LEVEL *level)`。バックドロップを配置します。
- `IWRAM_CODE void m7_hbl_floor()`。HBlank の割り込みルーチン。必要に応じてモード 2 に切り替え、アフィンパラメータをコピーし、フォグの効果を作ります。

`m7_prep_horizon()` と ` m7_update_sky()` は、それぞれ {@eq:horz-line} と {@eq:psi} の単純な実装なので（訳注: 2 つ目の関数が実装しているのは、バックドロップの位置の式のほうと思われます）、これらは手短に済ませられます。

```c
//! Calculate the horizon scanline
void m7_prep_horizon(M7_LEVEL *level)
{
    int horz;
    M7_CAM *cam= level->camera;

    if(cam->v.y != 0)
    {
        horz= M7_FAR_BG*cam->w.y - cam->pos.y;
        horz= M7_TOP - Div(horz*M7_D, M7_FAR_BG*cam->v.y);
    }
    else    // looking straight down (w.y > 0) means horizon at -inf scanline
        horz= cam->w.y > 0 ? INT_MIN : INT_MAX;

    level->horizon= horz;
}

//! Update sky-bg position
void m7_update_sky(const M7_LEVEL *level)
{
    REG_BG2HOFS= (level->camera->phi>>6)+M7_LEFT;
    REG_BG2VOFS= -clamp(level->horizon, 0, 228)-1;
}
```

地平線の計算ではクリッピングのファー面を使っていますが、厳密には必要ありません。地平線を無限遠にしたければ、カメラの高さを引くのをやめて、`M7_FAR_BG` = 1 を使ってください。_v_<sub>y</sub> = 0 の確認に注意してください。_v_<sub>y</sub> = cos(θ) なので、これは真上か真下を見ているときに真になります。一方では空だけが見え（アフィン背景なし）、もう一方では床だけが見える（バックドロップなし）ので、この区別は大事です。厳密には ±無限大にするべきですが、これは固定小数点なので、`INT_MIN/MAX` で我慢するしかありません。

バックドロップの配置については、ここでは_たくさん_手を抜いています。数学的に正しいバックドロップなら、幅 1720 ピクセルの背景のマップを使うことになります。できなくはありませんが、たいていはただ面倒なだけです。代わりに、512x256p の通常の背景を使い、角度→スクロールのオフセットの変換で *P* = 1024 を使っています。つまり、360° の 1 回転でマップが 2 回表示され、_dx_ は単に φ/64 になります。たしかに、床とバックドロップの視野は少しずれますが、何を探せばいいか知っていなければ気づかないので、問題ありません。

厳密に言えば、縦のオフセットは「背景の高さ − 地平線」になるべきですが、回り込みのおかげで背景の高さは無視できます。地平線をビューポートの大きさに制限しているのは、地平線のスキャンラインがとても大きくなりうるからです。上を見ると、その中の tan(θ) は無限大に近づくのでしたね？ 制限しないと、上を向いていくときにバックドロップのマップ全体を何周もスクロールしてしまい、とにかくひどい見た目になります。

#### アフィンパラメータの表を準備する

アフィンパラメータの計算は `m7_prep_affines()` で行います。HBlank の ISR の中でやってみることもできますが、除算が必要なので、単純に時間がかかりすぎます。また、1 か所でまとめてするほうが効率的です。変数の準備が 1 回で済むからです。このルーチンは {@eq:m7-sum} の計算を行います。スキャンラインごとに除算を含むかなりの数の計算をしなければならないので、かなり高くつくと予想できます。そのため、最初から IWRAM に置いています。

さて、すべてのスキャンラインについて計算する必要はありません。地平線より下のものだけです。{@eq:m7-sum} そのものの実装については、カメラ行列の 9 つの要素で計算するよりも、カメラ行列をまたばらして、θ と φ のサインとコサインで計算したほうがずっとうまくいくことがわかりました。次の段落でその方法を説明しますが、読み飛ばしてコードに進んでもかまいません。

カメラ行列は **C** = **R**<sub>y</sub>(φ)·**R**<sub>x</sub>(θ) であり、λ と **dx** は {@eq:m7-ofs}、つまり **dx**′ = **a**<sub>cw</sub> + λ·**C**·**b** で計算されることを思い出してください。**C** を分解して **b** と組み合わせれば、**b**′ = **R**<sub>x</sub>(θ)·**b** を作れます。この新しいベクトルがピッチを完全に引き受けます。縦の軸の周りの回転しかない場合、つまり前の章で説明した場合と同じようになるのです。この事前の回転のおかげで、コードはより単純で速くなります。

```c
IWRAM_CODE void m7_prep_affines(M7_LEVEL *level)
{
    if(level->horizon >= SCREEN_HEIGHT)
        return;

    int ii, ii0= (level->horizon>=0 ? level->horizon : 0);

    M7_CAM *cam= level->camera;
    FIXED xc= cam->pos.x, yc= cam->pos.y, zc=cam->pos.z;

    BG_AFFINE *bga= &level->bgaff[ii0];

    FIXED yb, zb;           // b' = Rx(theta) *  (L, ys, -D)
    FIXED cf, sf, ct, st;   // sines and cosines
    FIXED lam, lcf, lsf;    // scale and scaled (co)sine(phi)
    cf= cam->u.x;      sf= cam->u.z;
    ct= cam->v.y;      st= cam->w.y;
    for(ii= ii0; ii<SCREEN_HEIGHT; ii++)
    {
        yb= (ii-M7_TOP)*ct + M7_D*st;
        lam= DivSafe( yc<<12,  yb);     // .12f

        lcf= lam*cf>>8;                 // .12f
        lsf= lam*sf>>8;                 // .12f

        bga->pa= lcf>>4;                // .8f
        bga->pc= lsf>>4;                // .8f

        // lambda·Rx·b
        zb= (ii-M7_TOP)*st - M7_D*ct;   // .8f
        bga->dx= xc + (lcf>>4)*M7_LEFT - (lsf*zb>>12);  // .8f
        bga->dy= zc + (lsf>>4)*M7_LEFT + (lcf*zb>>12);  // .8f

        // hack that I need for fog. pb and pd are unused anyway
        bga->pb= lam;
        bga++;
    }
    level->bgaff[SCREEN_HEIGHT]= level->bgaff[0];
}
```

まず、計算を始めるスキャンライン（何もないこともあります）を求め、一時変数を_たくさん_定義します。一時変数のすべてが必要なわけではありませんが、コードが読みやすくなります。名前はさておき、ループの中のコードは[最初のモード 7 のデモ](mode7.html#ssec-order-code)の `hbl_mode7_c` ととてもよく似ています。違うのは、λ の計算で回転させた _y_<sub>s</sub> の値を使い、オフセットの計算で回転させた _z_<sub>s</sub>（= −*D*）の値を使うことです。でもって、それで全部です。

計算の後ろのコメントは結果の固定小数点の桁数を示していて、この場合は .8f か .12f のどちらかです。さあ、よく聞いてください。φ の拡大縮小した（コ）サイン、`lcf` と `lsf` は、12 ビット以上の精度を使うことが**とても**大事です。8 ビットで試しましたが、ひどいものでした。近距離で変位が全部ずれてしまいます。次に、変位での乗算とシフトの順序に注意してください。これらが今のままであることも、とても大事です。特に _L_ のあるものは、`M7_LEFT` との乗算をシフトのあとにしなければ**なりません**。これは信じてください。

最後に面白い点は、ループのあとの行です。HBlank の割り込みの 1 つずれのエラーを補うために、スキャンライン 0 のパラメータを配列の最後にコピーしています。

この関数は、おそらく C で作れる限り速いもので、コンパイラもかなりうまく仕事をしているので、手書きのアセンブリにしても得るものはほとんどありません。だからといって、まだかなりの時間がかからないわけではありません。除算だけで 100 から 400 サイクルほどかかります（BIOS の除算のサイクル数は、おおよそ 90 + 有効ビットあたり 13 です）。スキャンラインごとに除算が 1 回あると、これは本当にかさみます。これに対処する最善の戦略は、必要がなければ_しない_ことです。ピッチの角度を固定にするなら、すべての除算を前もって計算しておき、表を引くだけにできます。どうしても可変のピッチが必要なら、三角関数の方法もあります。{@fig:img-obj-sort} を振り返ってください（訳注: 参照すべきなのは投影の図のほうと思われます）。β を (0, *y*<sub>p</sub>, −*D*) と (0, 0, −*D*) の間の角度とすると、tan(β) = *y*<sub>p</sub>/_D_ です。三角法をたっぷり使えば、λ の式を次のように書き直せます。

<table id="eq:lambda-alt">
  <tr>
    <td class="eqnrcell">({!@eq:lambda-alt})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtable displaystyle="true" columnalign="right left" columnspacing="0em" rowspacing="3pt">
                    <mtr>
                      <mtd>
                        <mi>&#x3BB;</mi>
                      </mtd>
                      <mtd>
                        <mi></mi>
                        <mo>=</mo>
                        <msub>
                          <mi>a</mi>
                          <mrow data-mjx-texclass="ORD">
                            <mi>c</mi>
                            <mi>w</mi>
                            <mo>,</mo>
                            <mi>y</mi>
                          </mrow>
                        </msub>
                        <mrow data-mjx-texclass="ORD">
                          <mo>/</mo>
                        </mrow>
                        <mi>D</mi>
                        <mtext>&#xA0;</mtext>
                        <mo>&#x22C5;</mo>
                        <mi>cos</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <mi>&#x3B2;</mi>
                          <mo data-mjx-texclass="CLOSE">)</mo>
                        </mrow>
                        <mrow data-mjx-texclass="ORD">
                          <mo>/</mo>
                        </mrow>
                        <mi>sin</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <mi>&#x3B8;</mi>
                          <mo>+</mo>
                          <mi>&#x3B2;</mi>
                          <mo data-mjx-texclass="CLOSE">)</mo>
                        </mrow>
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

β は、スキャンラインごとに 1 つ、160 エントリのアークタンジェントのルックアップテーブルで得られます（そうだ、それを _p_<sub>d</sub> に入れておくことだってできます！）。それから 1/サインのルックアップテーブルを使います。ただし、十分に大きなルックアップテーブルを使うように気をつけなければなりません。ルックアップテーブルの引数は整数なので、β は切り捨てられ、それによって_大きく_精度を失います。特に地平線の近くではそうです。さて、三角関数の方法はまだ実際に試していませんが、Excel で簡単なテストをしてみたところ、1 周 512 の 1/サインのルックアップテーブルでは、地平線の近くで λ の誤差が 10% を大きく超え、それ以外のところでも誤差は 1% 前後になるようです。それを踏まえると、少なくとも 1 周 1024 をおすすめします。あるいは、ルックアップテーブルのエントリの間を補間することです。それは libtonc の `lu_lerp16()` 関数と `lu_lerp32()` 関数でできます。

三角関数の方法を使う以外にも、おそらくいろいろな方法で除算を速くできるでしょう。でも、これを最適化しに行く前に、本当にそれが必要なのかをまず自問してください。結局のところ、早すぎる最適化は諸悪の根源なのですから。

:::note アフィンの計算の高速化

最近、3 つの最適化を試しました。まず ARM/IWRAM で、これで 23k-58k サイクルにまで下がります。次に、sgeos との議論で見つかったちょっとしたリファクタリングです。カメラのベクトルをより少ない変数の集まりにまとめられ、10-20% 節約できます。それから三角関数の方法で、cos(β) と 1/sin(θ+β) を大きなルックアップテーブルで得るか、小さなルックアップテーブルと線形補間で得るかによって、全体を最大でも 10-20k、あるいは 7k-14k にまで下げられます。数学、シフト、符号をきちんと整えれば、魔法のようにうまくいきます。

:::

#### モード 7 の HBlank の割り込みルーチン

話を単純にするため、VDraw の間に起きる必要のあることは、ほぼすべて `m7_hbl_floor()` という 1 つの HBlank の ISR の中で行います。このデモの以前の版では VCount/HBlank の割り込みの仕組みを使っていましたが、それは手間に見合わないことがわかりました。これも IWRAM のルーチンです。本当にできるだけ速くする必要があるからです。この割り込みサービスルーチンは次のことをします。

1.  **vcount が床の範囲にあるか確認する**。このスキャンラインが床の一部でなければ、戻ります。
2.  **vcount が地平線か確認する**。地平線のスキャンラインに達したら、ビデオモードを変えて、`REG_BG2CNT` を床の設定にします。
3.  **アフィンパラメータを `REG_BG_AFFINE[2]` にコピーする**。現在のスキャンラインはもう過ぎているので、_次の_スキャンラインのパラメータを `REG_BG_AFFINE[2]` にコピーします。
4.  **フォグ**。この場合はオレンジ色にフェードします。

```c
// from tonc_core.h
//! Range check; true if xmin<=x<xmax
#define IN_RANGE(x, min, max)  ( (x) >= (min) && (x) < (max) )
```

```c
IWRAM_CODE void m7_hbl_floor()
{
    int vc= REG_VCOUNT;
    int horz= m7_level.horizon;

    // (1) Not in floor range: quit
    if(!IN_RANGE(vc, horz, SCREEN_HEIGHT) )
        return;

    // (2) Horizon: switch to mode 1; set-up bg control for floor
    if(vc == horz)
    {
        BF_SET(REG_DISPCNT, DCNT_MODE1, DCNT_MODE);
        REG_BG2CNT= m7_level.bgcnt_floor;
    }

    // (3) Looking at floor: copy affine params
    BG_AFFINE *bga= &m7_level.bgaff[vc+1];
    REG_BG_AFFINE[2] = *bga;

    // (4) A distance fogging with high marks for hack-value
    u32 ey= bga->pb*6>>12;
    if(ey>16)
        ey= 16;

    REG_BLDALPHA= BLDA_BUILD(16-ey, ey);
}
```

ポイント (3) と (4) はもう少し説明があったほうがよいでしょう。もう何度も言ったように、スキャンライン _vc_ の ISR は_次の_スキャンラインのパラメータを準備するべきです。だから単なる `[vc]` ではなく `level.bgaff[vc+1]` からコピーしているのです。スキャンライン 0 では *vc* = 160 の組を使いますが、0 のデータを配列の最後の要素にコピーしてあるので問題ありません。いつものように、構造体のコピー最高。

フォグには、まさにこのために `m7_prep_affines()` で λ を入れておいた _p_<sub>b</sub> を使っています。λ を拡大縮小したものはフォグのモデルとして最も正確ではありませんが、効果は十分に良く見えます。ブレンドのレジスタは 16 で頭打ちになるので、大きな値で回り込まないようにする必要があります。

それでも_まだ_、実際に何とブレンドしているのかという疑問が残ります。オレンジ色は GBA のフェードの持ち技にはないからです。少なくとも、_直接_にはありません。しかし、バックドロップとブレンドすることは十分に可能で、バックドロップは背景の色 0 を表示しているだけです。この色はオレンジ色も含めて何にでもできます。

### スプライトとオブジェクト {#ssec-code-spr}

スプライトとオブジェクトの処理は、次の 3 つの関数に分けてあります。

- `void update_sprites()`。主なスプライトの処理で、ほかの関数を呼んで位置合わせ、ソート、アニメーションを行います。
- `IWRAM_CODE void m7_prep_sprite(M7_LEVEL *level, M7_SPRITE *spr)`。スプライトの正しい位置と拡大率を計算します。
- `void kart_animate(M7_SPRITE *spr, const M7_CAM *cam)`。カートの周りを回るための正しいフレームを選びます。

実際にモード 7 の関数の一部なのは `m7_prep_sprite()` だけです。ほかのものは、考えているモード 7 のゲームごとに大きく違ってもおかしくありません。主なスプライトの処理 `update_sprites()` はかなり単純です。各スプライトについて `m7_prep_sprite()` を呼んでスプライトのソートのキーを作り、すべてのスプライトをソートし、ソートした属性を OAM にコピーする必要があります。アニメーションのために、各カートのスプライトについて `kart_animate()` も呼びます。ドッスンやほかのスプライトにもアニメーションがあれば、おそらくそれもここに入るでしょう。

```c
void update_sprites()
{
    int ii;

    M7_SPRITE *spr= m7_level.sprites;
    for(ii=0; ii<SPR_COUNT; ii++)
    {
        m7_prep_sprite(&m7_level, &spr[ii]);

        // Create sort key
        if(BF_GET2(spr[ii].obj.attr0, ATTR0_MODE) != ATTR0_HIDE)
            sort_keys[ii]= spr[ii].pos2.z;
        else
            sort_keys[ii]= INT_MAX;
    }

    // Sort the sprites
    id_sort_shell(sort_keys, sort_ids, SPR_COUNT);

    // Animate karts
    for(ii=0; ii<8; ii++)
        kart_animate(&spr[ii], m7_level.camera);

    // Update real OAM
    for(ii=0; ii<SPR_COUNT; ii++)
        obj_copy(&oam_mem[ii], &spr[sort_ids[ii]].obj, 1);
}
```

コードのほとんどはスプライトのソートに関するもので、それは理論のところですでに説明しました。スプライトの `pos2` メンバは、`m7_prep_sprite()` がスプライトのカメラ空間での位置を入れるように設定します。ソートのルーチン `id_sort_shell()` は、[優先順位の節](lab.html#sec-prio)で説明したインデックスの表のソート処理です。

もっと高度なアニメーションやスプライトの処理が欲しかったなら、それもここに入れていたでしょう。でも欲しくなかったので、入れていません。

#### スプライトの位置合わせと拡大縮小

関数 `m7_prep_sprite()` は、スプライトの画面上の正しい位置を計算し、正しい（再正規化した）拡大率でアフィン行列を準備し、スプライトが視野の空間の外にあれば隠します。

最初の段階は、{@eq:obj-w2s} の最初の部分 **x**<sub>c</sub> = **C**<sup>T</sup>·**r** を使って、スプライトのワールドでの位置をカメラ空間のベクトルに変換することです。ここで **r** はカメラに対するスプライトの位置 **r** = **x**<sub>w</sub>−**a**<sub>cw</sub> です。これを変数 `vc` に入れますが、_y_ と _z_ の符号を入れ替えています！ これで、続く計算が少し楽になります。このベクトルは `spr->pos2` にも格納され、別のところでソートに使われます。

2 つ目の段階は、{@tbl:culltest} の条件を使って、スプライトが実際に見えるかどうかを確認することです。ただし 1 つ例外があり、今度の確認ではスプライトの_再正規化した_長方形を使います。その部分を省くと、向きによってはおかしな表示が出ることがあります。スプライトの長方形を計算するのに、オブジェクトの長方形の大きさを使っています。オブジェクトの枠の中で見えるピクセルを示すスプライトの長方形も定義すれば、もっとぴったり合わせられますが、ここではそれは少しやりすぎかもしれません。

範囲の確認から先のコードのほとんどが `do-while(0)` ループの中にあることに注意してください。このパターンは、いわば貧乏人の `try/catch` ブロックです。ここで `goto` を使うことも_できました_が、有害とされているのでやめておきました。ともかく、ここで範囲外の「例外」が起きたら、スプライトを隠すべきだということで、それは手順 (5) で行います。

範囲の確認を通過したら、アフィン行列を準備し、{@eq:anchor} のアンカリングの式でオブジェクトの位置を計算する必要があります。

```c
//! Setup an object's attr/affine with the right attributes
/*! \param level    Mode 7 level data
*   \param spr      3D sprite to calculate for
*/
IWRAM_CODE void m7_prep_sprite(M7_LEVEL *level, M7_SPRITE *spr)
{
    M7_CAM *cam= level->camera;
    VECTOR vr, vc;      // Difference and inverted-cam vector
    int sx, sy;         // Object size
    RECT rect;          // Object rectangle

    // (1) Convert to camera frame
    vec_sub(&vr, &spr->pos, &cam->pos);
    vc.x=  vec_dot(&vr, &cam->u);
    vc.y= -vec_dot(&vr, &cam->v);
    vc.z= -vec_dot(&vr, &cam->w);
    spr->pos2= vc;

    OBJ_ATTR *obj= &spr->obj;
    sx= obj_get_width(obj);
    sy= obj_get_height(obj);

    // --- Check with viewbox ---
    do
    {
        // (2a) check distance
        if(M7_NEAR*256 > vc.z || vc.z > M7_FAR*256)
            break;

        // (2b) check horizontal
        rect.l= vc.x - spr->anchor.x*(256>>M7O_NORM);
        rect.r= rect.l + sx*(256>>M7O_NORM);
        if(M7_LEFT*vc.z > rect.r*M7_D || rect.l*M7_D > M7_RIGHT*vc.z)
            break;

        // (2c) check vertical
        rect.t= vc.y - spr->anchor.y*(256>>M7O_NORM);
        rect.b= rect.t + sy*(256>>M7O_NORM);
        if(-M7_TOP*vc.z > rect.b*M7_D || rect.t*M7_D > -M7_BOTTOM*vc.z)
            break;

        // (3) Set-up affine matrix
        OBJ_AFFINE *oa= &obj_aff_mem[spr->aff_id];
        oa->pa= oa->pd= vc.z>>(M7_D_SHIFT-M7O_NORM);    // normalized lambda
        oa->pb= oa->pb= 0;

        FIXED scale= DivSafe(M7_D<<16, vc.z);   // (.16 / .8) = .8

        // (4) anchoring
        // Base anchoring equation:
        // x = q0 - s - A(p0 - s/2)
        // In this case A = 1/lam; and q0 = xc/lam
        // -> x = (xc - p0 + s/2)/lam - s + screen/2
        int xscr, yscr;
        xscr  = spr->anchor.x*256 - sx*128;             // .8
        xscr  = (vc.x - (xscr>>M7O_NORM))*scale>>16;    // .0
        xscr += -sx - M7_LEFT;

        yscr  = spr->anchor.y*256 - sy*128;             // .8
        yscr  = (vc.y - (yscr>>M7O_NORM))*scale>>16;    // .0
        yscr += -sy + M7_TOP;
        obj_unhide(obj, ATTR0_AFF_DBL);
        obj_set_pos(obj, xscr, yscr);

        return;
    }
    while(0);

    // (5) If we're here, we have an invisible sprite
    obj_hide(obj);
}
```

#### カートのアニメーション

スプライトの周りを回るアニメーションの基本の理論は単純で、{@eq:psi} です。視線の角度 ψ は、スプライトのグローバルな角度 φ<sub>o</sub> と、カメラの角度 φ<sub>c</sub> およびカメラ空間でのスプライトへの角度 α との差、ψ = φ<sub>o</sub>−φ<sub>c</sub>−α です。角度を使うべきアニメーションのフレームに変換すれば完成です。

理論上は。

実際には、特にスーパーマリオカートのやり方には、いくつも落とし穴があります。まず {@fig:img-obj-frames} を見てください。これらの 12 枚のフレームは、スーパーマリオカートがキノピオに使っているものです。最初の厄介な点は、これが回転の右側だけだということです。左側は鏡映で作ります。それは簡単です。視線の角度が負なら、_p_<sub>a</sub> の符号を変えるだけです。

2 つ目の問題はタイルの数です。半円で 12 フレームということは、1 回転では 24 フレーム（実際には、前と後ろのフレームを重複させる必要はないので 22）になります。それぞれ 4x4=16 タイルなので、キノピオだけで 384 タイルになります（しかも回転のアニメーションだけで！）。キャラクター全員分で 8 倍すれば、VRAM を_はるかに_超えてしまいます。つまり、すべてのフレームを一度に VRAM に読み込んで、オブジェクトのタイルのインデックスでアニメーションさせることはできません。必要なフレームを動的に読み込まなければなりません。スプライトの構造体に、ROM にあるスプライトシート全体を指す `tiles` メンバがあったのはこのためです。

3 つ目の厄介な点は、フレームが円の上に均等に分けられていないことです。よく見ると、最初の 8 フレームは 0° から 90° の角度用で、残りの 4 つが 90°-180° 用です。その理由は、ほとんどの時間はカートを後ろから見ることになるので、そちらにフレームを多く割いたほうが得だからです。さて、理論ではアニメーションのフレームをとてもうまく計算できました。_N_·ψ/2<sup>16</sup> です。しかしそれは _N_ 個の等しい切れ端があることを前提にしていて、それはもうありません。本当にそうでしょうか？

たしかに、等しい切れ端はもうありません。でも、一種の対応づけを使って、等しい切れ端を_作る_ことはできます。{\*@fig:img-psi-lut} は、その原理がどう働くかを示しています。図には 12 個の主な区画（内側の円）があり、0、1、10、11 は 2-9 より広い角度を占めています。しかし、円を 16 個の部分（外側の円）に分けて、複数のエントリに同じフレームを使うこともできます。たとえば、主な並びの切れ端 0 は、新しい並びの切れ端 0 と切れ端 1 で受け持ちます。if/else ブロックで対応づけることもできますが、ルックアップテーブルを使うほうが誰にとっても楽です。実はこれで、前に触れなかったほかの 2 つの問題も片づきます。鏡映には通常の並びを何らかの形で逆にする必要があることと、たとえば真正面や真後ろを見ているときに切れ端が切り替わらないように、切れ端を実際には半分ずらさなければならないことです。ルックアップテーブルなら、これらの問題を一度にすべて解決できます。

<div class="cblock">
  <table cellspacing=4>
    <tr valign="top">
      <td>
        <div class="cpt" style="width:392px;">
          <img src="img/mode7/toad_frames.png" id="fig:img-obj-frames" alt="animation frames">
          <br>
          <b>{*@fig:img-obj-frames}</b>: いろいろな角度から見たキノピオのフレーム。
        </div>
      </td>
      <td>
        <div class="cpt" style="width:160px;">
          <img src="img/mode7/psi_lut.png" id="fig:img-psi-lut" alt="view LUT">
          <br>
          <b>{*@fig:img-psi-lut}</b>: 12 個の不均等な区画の代わりに、&psi; を 16 要素のルックアップテーブルのエントリに使う。
        </div>
      </td>
    </tr>
  </table>
</div>

```c
const u8 cKartFrames[32]=
{
     0,  1,  2, 3, 4, 5, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11,
    11, 10, 10, 9, 9, 8, 8, 7, 7, 6, 5, 4, 3,  2,  1,  0,
};

//! Animate kart sprite
void kart_animate(M7_SPRITE *spr, const M7_CAM *cam)
{
    OBJ_ATTR *obj= &spr->obj;

    if(BF_GET2(obj->attr0,ATTR0_MODE) == ATTR0_HIDE)
        return;

    TILE *dst= &tile_mem[4][BF_GET(obj->attr2, ATTR2_ID)];
    s16 psi= spr->phi - cam->phi;

    // Extra arctan angle for correctness
    if(g_state & STATE_OBJ_VIEW_ATAN)
        psi -= ArcTan2(spr->pos2.z>>8, spr->pos2.x>>8);

    memcpy32(dst, &spr->tiles[cKartFrames[(psi>>11)&31]*16], 16*8);

    OBJ_AFFINE *oa= &obj_aff_mem[spr->aff_id];
    if(psi < 0)
        oa->pa= -oa->pa;
}
```

上のコードは、カートの角度のルックアップテーブルとアニメーションのルーチンです。ルックアップテーブルには 32 個のエントリがあり、最初と最後の 7 つは 1 つずつの区画を使い、残りは 2 つずつになっています。また、ルックアップテーブルが対称であることにも注意してください。これは鏡映のために必要です。

ルーチンそのものはきれいとは言えませんが、仕事はこなします。まずスプライトが見えるかどうかを確認し、見えなければ抜けます。結果が見えないなら作業する意味はありません。スプライトのカメラでの角度 α には arctan が必要です。α の補正ありとなしの結果を見られるように、メニューにスイッチを付けておきました。違いはかなり小さいとわかると思います。各スプライトには常に同じ VRAM を使っているので、タイルのコピー先を求めるのは簡単です。コピー元のフレームを求めるのは少し見苦しく見えますが、実際には ψ→切れ端の変換と表引きだけです。

### 仕上げ: メインループとそのほかの飾り {#ssec-code-misc}

モード 7 の難しい部分は、これでほぼ扱いました。例外があるとすればメインループで、それはすぐあとで扱います。もちろん、レジスタ、スプライト、モード 7 の変数の初期化や、VRAM の読み込み、入力もありますが、それらはかなり簡単ですし、どのみちゲームごとに違いがちです。それらについては、実際のコードを見てください。

#### プログラムの主な流れ

下のコードに、`main()` 関数とその主な分岐があります。`init_main()` は、モード 7 の主な変数 `m7_level` を `m7_init()` で準備し、VBlank と HBlank の割り込みや、そのほかいろいろなものを初期化します。メインループはかなり短いです。関数 `input()` がカメラの移動とメニューの両方を処理します。

そのあとに実際のモード 7 の関数が来ます。`m7_prep_horizon()` は最初に来る必要がありますが、残りの順序はかなり任意です。ただ、`m7_prep_affines()` は最後に呼ぶことをおすすめします。ここで最も高くつく関数ですが、VDraw の時間にはみ出しても問題ないからです。ここでそうなるわけではありません（メインループはスキャンライン 170-210 あたりで終わることを計測しました）が、そうなっても大丈夫です。

```c
int main()
{
    init_main();

    while(1)
    {
        VBlankIntrWait();
        input();

        m7_prep_horizon(&m7_level);
        // Switch to backdrop display.
        if(m7_level.horizon > 0)
        {
            BF_SET(REG_DISPCNT, DCNT_MODE0, DCNT_MODE);
            REG_BG2CNT= m7_level.bgcnt_sky;
            REG_BLDALPHA= 16;
        }
        m7_update_sky(&m7_level);

        update_sprites();
        m7_prep_affines(&m7_level);
    }

    return 0;
}
```

#### 3D での移動

これが最後に扱いたいことです。3D でものを動かす方法です。正確に言うと、3D でのいろいろな移動の方法の実現の仕方です。知りたい人はきっといると思います。

3D での移動は、次元が 1 つ増えることを除けば、実は 2D での移動とほとんど同じです。人がときどき難しいと感じるのは、_本当は_ベクトルで考えるべきなのに、角度で考えてしまうからです。ベクトルに基づく移動（あるいは、ベクトルに基づく何でも）は、たいてい角度と三角法を使うよりずっと話を簡単にします。この章の理論でベクトルと行列を使ってきたのもそのためです。

ここでは、カメラの移動の 3 つの異なるモードを見ていきます。ワールドの座標系を使うもの、カメラの座標系を使うもの、そして地面と平行なままにするための、その中間のものです。でもまず、ある方向に動くということが実際に何を_意味する_のかを見てみましょう。

3D 空間のすべての物体は、自分だけの小さな座標空間、<dfn>ローカル座標系</dfn>を持っています。これは、ローカルの _x_、_y_、_z_ の方向を表す 3 つのベクトルの組として定義されます。カメラの場合は、これらをそれぞれ **u**、**v**、**w** と名付けました。ローカルの行列は、このベクトルの組を書き表す別の方法にすぎません。移動は普通、これらのベクトルに沿った歩みとして定義されます。

この例として、自分の頭をカメラだと考えて、ローカルの軸を矢印で表してみてください。**u** は右耳から、**v** は頭のてっぺんから、**w** は後頭部から突き出ています。右に 1 歩なら **u** の方向に、前に 1 歩なら −**w** の方向に進みます。一般の移動は、右に _x_ 歩、上に _y_ 歩、後ろに _z_ 歩と書けます。_x_、_y_、_z_ は方向ベクトルの_倍率_として使われ、グローバルな空間での最終的な変位は **dx** = _x_·**u** + _y_·**v** + _z_·**w** になります。

そしてここで行列が出てきます。この 3 つの倍率はベクトル **r** = (_x_, *y*, *z*) として書けます。これは_ローカル_空間での距離のベクトルです。3 つの方向は行列 **C** を作っていました。上で示した **dx** の定義は、**dx** = **C**·**r** を長々と書いたものにほかなりません。行列の積は、「**C** のベクトルを **r** の要素で拡大縮小して、全部足し合わせる」の略記にすぎないのです。この手順は、どんな向きのどんな物体にも使えることに注意してください。必要なのは、正しいローカルの行列を見つけることだけです。

私の場合は、`input()` の中で、いろいろなボタンに基づいてベクトル **r** を組み立てています。この時点では、それはまだ何も意味していません。移動の方法にはそれぞれ独自の方向の組、したがって **r** にかけるべき独自の行列があります。それらをかけて結果をカメラの位置に加える関数を用意してあります。これらはすべて {@tbl:motion} とその下のコードにあります。

「level」（つまり地面と水平な）方式は、地上の物体のカメラの仕組みとしてはおそらく最も一般的です。ただ、飛んでいる物体ならローカルの座標系を使うのが理にかなっているかもしれません。いろいろ試して、好きなものを見つけてください。

<div class="cblock">
  <table id="tbl:motion" class="table-data">
    <caption align="bottom">
      <b>{*@tbl:motion}</b>: 移動の方法と、それに対応するワールド空間への変換。物体の新しい位置は <b>x</b><sub>w</sub> += <b>v</b><sub>w</sub> で与えられます。
    </caption>
    <tbody>
      <tr>
        <th>方法</th>
        <th>関数</th>
        <th>変換</th>
      </tr>
      <tr>
        <td>グローバル座標系</td> 
        <td><code>m7_translate_global()</code></td>
        <td><b>dx</b> = <b>I</b> · <b>r</b> = <b>r</b>
      </tr>
      <tr>
        <td>ローカル（カメラ）座標系</td> 
        <td><code>m7_translate_local()</code></td>
        <td><b>dx</b> = <b>C</b>(&theta;, &phi;) · <b>r</b>
      </tr>
      <tr>
        <td>Level: ローカルだが地面と平行</td> 
        <td><code>m7_translate_level()</code></td>
        <td><b>dx</b> = <b>R</b><sub>y</sub>(&phi;) · <b>r</b>
      </tr>
    </tbody>
  </table>
</div>

```c
//! Translate by \a dir in global frame
void m7_translate_global(M7_CAM *cam, const VECTOR *dir)
{
    vec_add_eq(&cam->pos, dir);
}

//! Translate by \a dir in local frame
void m7_translate_local(M7_CAM *cam, const VECTOR *dir)
{
    cam->pos.x += (cam->u.x * dir->x + cam->v.x * dir->y + cam->w.x * dir->z) >> 8;
    cam->pos.y += ( 0                + cam->v.y * dir->y + cam->w.y * dir->z) >> 8;
    cam->pos.z += (cam->u.z * dir->x + cam->v.z * dir->y + cam->w.z * dir->z) >> 8;
}

//! Translate by \a dir using local frame for x/y, but global z
void m7_translate_level(M7_CAM *cam, const VECTOR *dir)
{
    cam->pos.x += (cam->u.x * dir->x - cam->u.z * dir->z)>>8;
    cam->pos.y += dir->y;
    cam->pos.z += (cam->u.z * dir->x + cam->u.x * dir->z)>>8;
}
```

行列にあまりなじみがなければ、まぶしくて恐ろしく見えるかもしれませんが、少し慣れれば命の恩人になりえます。大規模な 3D システムがひっきりなしに行列を使っているのには_理由が_あるのです。何もかも生の三角法でやるのは難しい、とても難しいのです。行列を使えば、目の前の仕事に一番自然な座標系の中で作業し、最後に必要な座標系に変換できます。幾何学に関わる仕事をするなら、線形代数の基本（ベクトルと行列の使い方の規則）をもっと学ぶ努力は、十分に報われます。

:::note 余談: スプライトを中心に据える

行列でどれほど楽ができるかの例として、カメラを特定のスプライトに向けて中心に据え、その周りを回る問題を考えてみてください。手元にあるのは、カメラ行列 **C**、見る距離 _Z_、そしておそらくスプライトの位置 **x**<sub>w</sub> です。すべきことは、カメラをスプライトの位置に動かし、それから後ろに _Z_ 歩下がることです。言い換えると **a**<sub>cw</sub> = **x**<sub>w</sub>+**C**·(0, 0, _Z_) で、これは結局 **a**<sub>cw</sub> = **x**<sub>w</sub>+Z**w** になります。

カメラ行列がわかっていれば、位置合わせはほとんど自動的に書けてしまいます。

:::

## まとめ {#sec-conc}

ついに終わりました！ この章の本文では、モード 7 のゲームの最も重要な要素を説明しました。アフィンパラメータの計算、地平線とバックドロップの追加、3D のスプライトの位置合わせ、ソート、_そして_アニメーション、さらにおまけとして距離に応じたフォグの効果の作り方です。ここまでの本文では、Tonc のほかの部分で説明したほぼすべての題材を、しかも簡単な部分だけでなく使いました。ここまで来て、上のことのすべてかほとんどを理解できたなら、おめでとうございます。

それでも、もう少し良くするためのいくつかのことは省きました。たとえば、λ の計算にあるあの除算をすべてなくすこと。あるいは、アフィンオブジェクトが 32 個までという制限を回避することや、床にスプライトの影を置くこと。ピッチを真上か真下で止める代わりに、宙返りをきちんとできるようにする方法も示していません。これらは概念的には比較的わかりやすいのですが、実装するにはずっと多くのコードが必要になります。この本文を理解できたなら、きっと自分で解決できるでしょう。
