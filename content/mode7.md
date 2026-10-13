# 20. モード 7 その 1

<!-- toc -->

さて、ここからはかっこいいもの、モード 7 です。GBA での実装方法だけでなく、その背後にある数学も扱います。[タイル背景](regbg.html)（特に[変形できる](affbg.html)もの）と[割り込み](interrupts.html)についてよく知っている必要があります。そうでなければ、それらの話題を読んでおいてください。ここにあるのはモード 7 の基本の説明です。[発展編のページ](mode7ex.html)もありますが、まずはこちらを読むことを強くすすめます。そちらで使う数学に比べれば、ここの数学はまだかなり簡単だからです。

## はじめに {#sec-intro}

ずっとずっと昔の 1990 年に、スーパーファミコンがありました。ファミコンの 16 ビットの後継機です。新しい技術につきものの普通の改良に加えて、スーパーファミコンは、背景やスプライトに線形変換（回転や拡大縮小など）をかけられるグラフィックの技のための専用ハードウェアを持った、最初のゲーム機でした。<dfn>モード 7</dfn> はこれをさらに一歩進めました。背景を回転・拡大縮小するだけでなく、遠近法の段階を加えて 3D のような見た目を作り出したのです。

モード 7 を、またひとつのきれいな小技などとはとても呼べません。たとえば、モード 7 はレースゲームのジャンルを根本から変えました。昔のレースゲーム（ポールポジションやアウトランなど）は、単純な左右のカーブに限られていました。モード 7 では、視界がすぐ目の前のコースに限られなくなったので、もっと面白いコースが作れるようになりました。最初にこれを使ったゲームは F-ZERO で、それ以前のものをすべて吹き飛ばしました（初代のファイアフィールドは、ヘアピンカーブ、マグネットビーム、地雷があって、今でも最も凶悪なコースの 1 つです）。すぐに、スーパーマリオカート（うーん、レインボーロード。150cc、ずっとアクセル全開で *ごぼごぼ*）やパイロットウイングスといった輝かしいゲームが続きました。

GBA は基本的に小さなスーパーファミコンなので、GBA でもモード 7 のグラフィックができると考えるのは筋が通っています。そして、それは正しいのですが、GBA のモード 7 はスーパーファミコンのものとは少し違うと聞いています。スーパーファミコンでは、ビデオモードは本当に #7 までありました（[SNESdev Wiki](https://snes.nesdev.org/wiki/Backgrounds) を見てください）。GBA にはモード 0-5 しかありません。ですから、厳密には「GBA のモード 7」もまた名前として不適切です。しかし、スーパーファミコンのプログラマでない人（私も含めて、ほぼ全員が_そう_です）にとって、この言葉はそれが有名にしたグラフィックの効果、つまり遠近法の視点と同じ意味です。そして GBA でも遠近法の視点は作れるので、その意味ではこの言葉はやはり正しいのです。

スーパーファミコンについてはよくわかりませんが、GBA のモード 7 は、OpenGL や Direct3D のような本物の 3D API とはまったく違います。それらのシステムでは、適切な透視投影の行列を渡して、パイプラインに置くだけです。しかし GBA では、使えるのは一般的な 2D の変換行列 **P** と変位 **dx** だけで、遠近法の計算はすべて自分でしなければなりません。つまり基本的には、HBlank の DMA か HBlank の割り込みを使って、スキャンラインごとに拡大縮小と平行移動を変えなければならないということです。

このチュートリアルでは、[sbb_aff](affbg.html#sec-demo) デモの 64x64t のアフィン背景（@fig:img-m7-map に少し似ています）を使い、モード 7 の魔法をかけて、@fig:img-m7-persp に描かれているようなものに変えます。重点は、その魔法がどう働くのかを詳しく示すことです。最終的な結果は HBlank の割り込み関数として示しますが、HBlank の DMA の場合に書き換えるのはそれほど難しくないはずです。

<div class="lblock">
  <table>
    <tr valign="top">
    <td>
      <div class="cpt" style="width:256px;">
        <img src="img/mode7/m7_map.png" id="fig:img-m7-map" alt="This is your map.">
        <br>
        <b>*@fig:img-m7-map</b>: これがあなたのマップです（まあ、だいたいは）
      </div>
    <td>
      <div class="cpt" style="width:240px;">
        <img src="img/mode7/m7_persp.png" id="fig:img-m7-persp" alt="This is your map in mode7.">
        <br>
        <b>*@fig:img-m7-persp</b>: これがモード 7 でのあなたのマップです。
      </div>
  </table>
</div>

## 遠近法の感覚をつかむ {#sec-persp}

（遠近法の基本を知っているなら、この節は流し読みしてかまいません。）  
世界地図や 3D のゲームを見たことがあれば、3D から 2D に写すときには何かを犠牲にしなければならないことを知っているでしょう。これを技術的には<dfn>投影</dfn>と言います。投影には多くの種類がありますが、ここで扱うのは<dfn>透視投影</dfn>で、遠くにあるものほど小さく見えるようにします。

@fig:img-3dview のような 3D 空間から始めます。コンピュータグラフィックスでは、_x_ 軸を右向き、_y_ 軸を上向きにするのが慣例です。_z_ 軸の向きは空間の利き手で決まります。<dfn>右手系</dfn>の座標系では後ろ（画面から手前に出る向き）を指し、左手系では前を指します。私は右手系を使います。左手系だと、回転や法線の計算になると頭がどうしようもなく混乱するからです。もう 1 つの理由は、このほうが画面の座標が (_x_, _z_) の値に対応するからです。また、見る人を原点に置くのも慣例です（見る人の位置を変えたければ、世界を逆方向に平行移動するだけです）。右手系では、これは負の _z_ 軸の方向を見ていることになります。

もちろん、すべてが見えるわけではありません。<dfn>視野の空間</dfn>の内側にある物体だけが見えます。透視投影では、これは見る人の位置（この場合は原点）と、見る人の前方の距離 _D_ にある<dfn>投影面</dfn>で決まります。投影面は画面だと考えてください。投影面は幅 _W_ と高さ _H_ を持ちます。ですから視野の空間は実際には<dfn>視野の角錐</dfn>ですが、実際には知覚できる距離にも最小と最大があるので、たいていは視野の_錐台_（頭を切り落とした角錐）になります。

{\*@fig:img-side1} は、透視投影が実際に何をするかを示しています。点 (_y_, _z_) が、投影面上の点 (_y_<sub>p</sub>, −*D*) に投影されます。投影された _z_ 座標は、定義により −*D* です。投影された _y_ 座標は、投影面と、見る人と元の点を通る直線との交点です。

<table id="eq:persp">
  <tr>
    <td class="eqnrcell">(!@eq:persp)
    <td class="eqcell">
      <math display="block" class="tml-display" style="display:block math;">
        <mrow>
          <msub>
            <mi>y</mi>
            <mi>p</mi>
          </msub>
          <mo>=</mo>
          <mi>y</mi>
          <mo>·</mo>
          <mi>D</mi>
          <mo lspace="0em" rspace="0em">⁄</mo>
          <mi>z</mi>
        </mrow>
      </math>
</table>

基本的には、
<math style="display: inline-block math;">
<mrow>
<mi>z</mi>
<mo lspace="0em" rspace="0em">/</mo>
<mi>D</mi>
</mrow>
</math> で割ります。
とても重要な係数なので、専用の変数があります。<dfn>ズーム係数</dfn> λ です。

<table id="eq:lambda">
  <tr>
    <td class="eqnrcell">(!@eq:lambda)
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>&#x3BB;</mi>
                  <mo>=</mo>
                  <mi>z</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <mi>D</mi>
                  <mo>=</mo>
                  <mi>y</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mo>/</mo>
                  </mrow>
                  <msub>
                    <mi>y</mi>
                    <mrow data-mjx-texclass="ORD">
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

原則として、投影面より手前にあるもの（λ\<1）はすべて拡大され、奥にあるもの（λ\>1）はすべて縮小されます。

<div class="cblock">
  <table>
    <tr valign="top">
      <td>
        <div class="cpt" style="width:256px;">
          <img src="img/mode7/m7_3dview.png" id="fig:img-3dview" alt="3D coordinate system">
          <br>
          <b>*@fig:img-3dview</b>:
          原点と、<i>z</i>= &minus;<i>D</i> にある画面の長方形（<i>W</i>&times;<i>H</i>）で決まる視野の角錐を示した 3D の座標系
        </div>
      <td>
        <div class="cpt" style="width:320px;">
          <img src="img/mode7/m7_side1.png" id="fig:img-side1" alt="projection step, side view">
          <br>
          <b>*@fig:img-side1</b>:
          横から見た図。点 (<i>y, z</i>) が (<i>z</i> = &minus;<i>D</i> の面に投影されます。投影された点は <i>y</i><sub>p</sub> =  <i>y·D/z</i> です
        </div>
  </table>
</div>

## モード 7 の登場 {#sec-m7-math}

{\*@fig:img-3dview} と @fig:img-side1 は、たくさんの物体と見る人のいろいろな向きがある 3D の世界での、透視投影の一般の場合を表しています。モード 7 の場合は、それよりかなり単純です。

- **物体**。扱う物体は 2 つだけです。見る人（点 **a** = (_a_<sub>x</sub>, _a_<sub>y</sub>, _a_<sub>z</sub>) にいます）と床（定義により _y_=0 にあります）です。
- **見る人の向き**。完全な 3D の世界では、見る人の向きは 3 つの角度で与えられます。ヨー（y 軸）、ピッチ（x 軸）、ロール（z 軸）です。話を簡単にするため、ヨーだけに絞ります。
- **地平線の問題**。視線の方向を床と平行に保つので、地平線は画面の中央に来るはずです。そうすると画面の上半分が空になり、少しもったいないです。これを解決するため、視野の空間の下半分だけを使い、地平線が画面の一番上に来るようにします。上と下の視線は少し見下ろした場合と同じになりますが、投影面は依然として垂直なので、2 つの場合は同じでは_ない_ことに注意してください。この違いを理解しておくことが大事です。

<div class="lblock">
  <div class="cpt" style="width:320px;">
    <img src="img/mode7/m7_side2.png" id="fig:img-side2" alt="Side view of Mode7 perspective">
    <b>*@fig:img-side2</b>: モード 7 の遠近法を横から見た図
  </div>
</div>

{\*@fig:img-side2} は状況全体を示しています。*y* = *a*<sub>y</sub> にいる見る人が、負の z 方向を見ています。見る人の前方の距離 _D_ に投影面があり、その下半分が高さ _H_（=160）の GBA の画面に表示されます。さて、ここからが楽しいところです。GBA には本当の 3D のハードウェアの機能はありませんが、スキャンラインごとに拡大縮小と平行移動 `REG_BGxX-REG_BGxPD` を巧みに操作すれば、それらしく見せられます。床のどのラインの部分がどのスキャンラインに、どのズームの度合いで入るのかを求めるだけです。実質的には、とても単純なレイキャスターを作っているのです。

### 数学 {#ssec-math-math}

概念的には、モード 7 には {@fig:img-steps}a-d に示す 4 つの段階があります。緑の図形は元のマップ、赤はその操作のあとのマップです。スキャンライン _h_ について、することは次のとおりです。

1. **事前の平行移動**。**a**= (_a_<sub>x</sub>, _a_<sub>z</sub>) だけ平行移動します。これで見る人が原点に来ます。段階 b と c ではそこにいる必要があります。
2. **回転**。α だけ回転します。これでヨーの角度を扱います。ここまでの段階は通常の変形できる背景と同じなので、理解するのに苦労はないはずです。
3. **透視除算**。次に、全体を 1/λ で拡大縮小します。@eq:lambda から λ = *a*<sub>y</sub>/_h_ です。直線 *z* = *z*<sub>h</sub> は、スキャンライン _h_ に属する直線です。拡大縮小したあとのこの直線の新しい位置は _z_ = −*D* です。それこそが透視除算の目的だったのですから。
4. **事後の平行移動**。(−**x**<sub>s</sub>) だけ平行移動します。マイナスの符号に注意してください。透視除算のあとに残っているのは、完全に変換されたマップを画面上の正しい位置（ベージュの領域）に戻すことだけです。明らかな理由から、横方向の成分は画面の幅の半分になるはずです。縦方向の移動は床のラインをスキャンラインに動かすものなので、ベクトルは次のようになります。

<table id="eq:post-ofs">
<tr>
  <td class="eqnrcell">(!@eq:post-ofs)
  <td class="eqcell">
  <table class="eqtbl" cellpadding=2 cellspacing=0>
  <col align="right">
  <col align="center">
  <col align="left">
  <tr>
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <msub>
                  <mi>x</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>s</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <mi>W</mi>
                <mrow data-mjx-texclass="ORD">
                  <mo>/</mo>
                </mrow>
                <mn>2</mn>
                <mo>=</mo>
                <mn>120</mn>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
  <tr>
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <msub>
                  <mi>y</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>s</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <mo stretchy="false">(</mo>
                <mi>D</mi>
                <mo>+</mo>
                <mi>h</mi>
                <mo stretchy="false">)</mo>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
  </table>
</table>

<div class="cblock">
  <table id="fig:img-steps" border=1 cellpadding=4 cellspacing=0>
    <caption align="bottom">
      {*@fig:img-steps}a-d: モード 7 の 4 つの段階
    </caption>
    <tr>
      <td><img src="img/mode7/m7_step1.png" alt="pre-translate">
      <td><img src="img/mode7/m7_step2.png" alt="rotate">
      <td><img src="img/mode7/m7_step3.png" alt="scale">
      <td><img src="img/mode7/m7_step4.png" alt="post-translate">
    <tr>
      <td>{*@fig:img-steps}a: (<i>a</i><sub>x</sub>, <i>a</i><sub>z</sub>) だけ事前に平行移動
      <td>{*@fig:img-steps}b: &alpha; だけ回転
      <td>{*@fig:img-steps}c: 1/&lambda; で拡大縮小
      <td>{*@fig:img-steps}d: (<i>x</i><sub>s</sub>, <i>y</i><sub>s</sub>) だけ事後に平行移動
  </table>
</div>

### すべてをまとめる {#ssec-math-combine}

上で説明した段階はたしかに手順の全体ですが、まだ片づけておくべき細かい点がいくつかあります。まず、GBA の変換行列 **P** は画面空間から背景の空間へ写すもので、それは実際にはやろうとしていることの逆だということを思い出してください。ですから、使うべきなのは次のものです。

<table id="eq:prs">
<tr>
  <td class="eqnrcell">(!@eq:prs)
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtext mathvariant="bold">P</mtext>
                <mo>=</mo>
                <mtext mathvariant="bold">S</mtext>
                <mo stretchy="false">(</mo>
                <mi>&#x3BB;</mi>
                <mo stretchy="false">)</mo>
                <mo>&#x22C5;</mo>
                <mtext mathvariant="bold">R</mtext>
                <mo stretchy="false">(</mo>
                <mi>a</mi>
                <mo stretchy="false">)</mo>
                <mo>=</mo>
                <mrow data-mjx-texclass="INNER">
                  <mo data-mjx-texclass="OPEN">[</mo>
                  <mtable columnspacing="1em" rowspacing="4pt">
                    <mtr>
                      <mtd>
                        <mi>&#x3BB;</mi>
                        <mo>&#x22C5;</mo>
                        <mi>cos</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <mi>a</mi>
                          <mo data-mjx-texclass="CLOSE">)</mo>
                        </mrow>
                      </mtd>
                      <mtd>
                        <mo>&#x2212;</mo>
                        <mi>&#x3BB;</mi>
                        <mo>&#x22C5;</mo>
                        <mi>sin</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <mi>a</mi>
                          <mo data-mjx-texclass="CLOSE">)</mo>
                        </mrow>
                      </mtd>
                    </mtr>
                    <mtr>
                      <mtd>
                        <mi>&#x3BB;</mi>
                        <mo>&#x22C5;</mo>
                        <mi>sin</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <mi>a</mi>
                          <mo data-mjx-texclass="CLOSE">)</mo>
                        </mrow>
                      </mtd>
                      <mtd>
                        <mi>&#x3BB;</mi>
                        <mo>&#x22C5;</mo>
                        <mi>cos</mi>
                        <mo data-mjx-texclass="NONE">&#x2061;</mo>
                        <mrow>
                          <mo data-mjx-texclass="OPEN">(</mo>
                          <mi>a</mi>
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
  </table>
</table>

そう、反時計回りの回転にはマイナスの符号で正しいのです（**R** は時計回りの回転として定義されています）。また、GBA は画面の点 **q** と背景の点 **p** の間に次の関係を使うことも思い出してください。

<table id="eq-ofs-base">
  <tr>
    <td class="eqnrcell">(20.5)
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>d</mi>
                  <mi>x</mi>
                  <mo>+</mo>
                  <mi>P</mi>
                  <mo>&#x22C5;</mo>
                  <mi>q</mi>
                  <mo>=</mo>
                  <mi>p</mi>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
</table>

つまり、平行移動が<u>1 つ</u>と変換が<u>1 つ</u>です。うまくいかせるには、事前と事後の平行移動を組み合わせなければなりません。これは以前、[アフィン背景のページ](affbg.html#sec-aff-ofs)の式 4 で、名前は違いますが見ました。ともかく、必要なのは次のものです。

<table id="eq-aff-ofs">
<tr>
  <td class="eqnrcell">(20.6)
  <td class="eqcell">
    <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
      <mstyle displaystyle="true" scriptlevel="0">
        <mrow data-mjx-texclass="ORD">
          <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
            <mtr>
              <mtd>
                <mtable displaystyle="true" columnalign="right left" columnspacing="0em" columnwidth="auto auto" rowspacing="3pt" data-width-includes-label="true">
                  <mtr>
                    <mtd></mtd>
                    <mtd>
                      <mi>d</mi>
                      <mi>x</mi>
                      <mo>+</mo>
                      <mi>P</mi>
                      <mo>&#x22C5;</mo>
                      <mi>q</mi>
                      <mo>=</mo>
                      <mi>p</mi>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd></mtd>
                    <mtd>
                      <mi>P</mi>
                      <mo>&#x22C5;</mo>
                      <mo stretchy="false">(</mo>
                      <mi>q</mi>
                      <mo>&#x2212;</mo>
                      <msub>
                        <mi>x</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>s</mi>
                        </mrow>
                      </msub>
                      <mo stretchy="false">)</mo>
                      <mo>=</mo>
                      <mi>p</mi>
                      <mo>&#x2212;</mo>
                      <mi>a</mi>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd></mtd>
                    <mtd>
                      <mspace width="5cm" height="1pt" style="background: var(--fg);"></mspace>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd></mtd>
                    <mtd>
                      <mi>d</mi>
                      <mi>x</mi>
                      <mo>+</mo>
                      <mi>P</mi>
                      <mo>&#x22C5;</mo>
                      <msub>
                        <mi>x</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>s</mi>
                        </mrow>
                      </msub>
                      <mo>=</mo>
                      <mi>a</mi>
                    </mtd>
                  </mtr>
                  <mtr>
                    <mtd></mtd>
                    <mtd>
                      <mi>d</mi>
                      <mi>x</mi>
                      <mo>=</mo>
                      <mi>a</mi>
                      <mo>&#x2212;</mo>
                      <mi>P</mi>
                      <mo>&#x22C5;</mo>
                      <msub>
                        <mi>x</mi>
                        <mrow data-mjx-texclass="ORD">
                          <mi>s</mi>
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

ですから、スキャンラインごとにズームの計算をし、@eq:prs の **P** 行列を `REG_BGxPA-REG_BGxPD` に、**a−P·x**<sub>s</sub> を `REG_BGxX` と `REG_BGxY` に入れれば、はい完成！ 即席のモード 7 です。

まあ、ほぼ完成です。HBlank の割り込みの中で `REG_BGxY` に書き込むと何が起きるかを[思い出してください](affbg.html#ssec-ao-refpts)。_現在の_スキャンラインが、画面の原点のゼロのラインとみなされます。言い換えると、_y<sub>s</sub>_ の _+h_ の部分は自動的に行われるのです。本当の _y_<sub>s</sub> を _y_<sub>s0</sub> と呼び直すと、使う_べき_なのは次のものです。

<table id="eq-yofs">
<tr>
  <td class="eqnrcell">(20.7)
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
                    <mi>s</mi>
                  </mrow>
                </msub>
                <mo>=</mo>
                <msub>
                  <mi>y</mi>
                  <mrow data-mjx-texclass="ORD">
                    <mi>s</mi>
                    <mn>0</mn>
                  </mrow>
                </msub>
                <mo>&#x2212;</mo>
                <mi>h</mi>
                <mo>=</mo>
                <mi>D</mi>
              </mtd>
            </mtr>
          </mtable>
        </mrow>
      </mstyle>
    </math>
</table>

さて、理論上は必要なものはすべてそろいました。しかし実際には、うまくいかなくなりうることがいくつもあります。その話に入る前に、素敵な（それほど小さくない）デモをお見せします。

## 3 通りのデモ {#sec-demo}

いつものように、デモがあります。実はモード 7 のデモはいくつかあるのですが、今それは重要ではありません。デモの名前は `m7_demo` で、操作は次のとおりです。

<table>
  <col valign="top">
  <tr><th>D-pad<td>平行移動（ストレイフ）する。
  <tr><th>L, R<td>左右に向きを変える（つまり、マップをそれぞれ右と左に回転する）
  <tr><th>A, B<td>上下に移動する。どちらがどちらかは忘れました。
  <tr><th>Select<td>3 つの異なるモード 7 の種類（A、B、C）を切り替える
  <tr><th>Start<td>すべての値をリセットする（<b>a</b>= (256, 32, 256)、&alpha;= 0）
</table>

「3 つの異なるモード 7 の種類を切り替える」？ ええ、そう言いました。3 つの種類すべてで動き回ってみてください。お願いします。左上の隅に、現在の種類を示すラベルがあります。

<div class="cblock">
  <table id="img-types">
    <tr>
      <td>
        <div class="cpt" style="width:240px;">
          <img src="img/demo/m7_demo_a.png" alt="blocky"><br>
          <b>図 20.7a</b>: 種類 A: ブロック状。
        </div>
      <td>
        <div class="cpt" style="width:240px;">
          <img src="img/demo/m7_demo_b.png" alt="sawtooth"><br>
          <b>図 20.7b</b>: 種類 B: のこぎり状。
        </div>
    <tr>
      <td colspan=2 align="center">
        <div class="cpt" style="width:240px;">
          <img src="img/demo/m7_demo_c.png" alt="smooth"><br>
          <b>図 20.7c</b>: 種類 C: なめらか。
        </div>
  </table>
</div>

## 順序、順序！ {#sec-order}

デモを少しいじってみましたか？ 結構です。3 つの種類の違いに気づきましたか？ さらに結構です！ 参考までに、それぞれの種類に対応する図 20.7a-c を見てください。違いがよくわかります。

- 種類 A はひどくブロック状です。赤いタイルの中の数字は「8」のはずです。へっ、数字？ 何の数字？
- 種類 B はましです。左側はなめらかですが、右側にはまだ少し問題があります。でも少なくとも、少し想像力を働かせれば 8 に見えます。
- 種類 C。これですよ！ 中央の線ははっきりしています。ほとんどの時間はそこを見ているので、これは大事です。でも両側でも、かなりまともに見えます。

というわけで、まったく違うモード 7 の結果が 3 つありますが、すべて同じ数学に基づいていることは保証します。では、なぜある方法はひどく見え、別の方法は素晴らしく見えるのでしょうか？

### コード {#ssec-order-code}

以下は、これらの種類を作る 2 つの HBlank の ISR です。種類 A と B は、1 点を除いてほとんど同じです。種類 C はほかとはまったく違います。自分を痛めつけるのが好きなら、コードだけから違いを説明してみてください。種類 C がなぜうまくいくのかを突き止めるのに昨夜のほとんどを費やしたので、あなたを宙ぶらりんのままにしておこうかと半分思っています。幸い、その半分は今眠っています。

```c
#define M7_D   128

extern VECTOR cam_pos;          // Camera position
extern FIXED g_cosf, g_sinf;    // cos(phi) and sin(phi), .8f
```

<pre><code class="language-c hljs">// --- Type A ---
<span class="rem">// (offset * zoom) * rotation
// All .8 fixed</span>
void m7_hbl_a()
{
    FIXED lam, xs, ys;

    lam= cam_pos.y*lu_div(REG_VCOUNT)&gt;&gt;16;  // .8*.16/.16 = .8

    // Calculate offsets (.8)
    xs= 120*lam;
    ys= M7_D*lam;

    REG_BG2PA= (g_cosf*lam)&gt;&gt;8;
    REG_BG2PC= (g_sinf*lam)&gt;&gt;8;

    REG_BG2X = cam_pos.x - ( (xs*g_cosf-ys*g_sinf)&gt;&gt;8 );
    REG_BG2Y = cam_pos.z - ( (xs*g_sinf+ys*g_cosf)&gt;&gt;8 );  
}
</code></pre>

<pre><code class="language-c hljs">// --- Type B ---
<span class="rem">// (offset * zoom) * rotation
// Mixed fixed point: lam, xs, ys use .12</span>
void m7_hbl_b()
{
    FIXED lam, xs, ys;

    lam= cam_pos.y*lu_div(REG_VCOUNT)&gt;&gt;12;  // .8*.16/.12 = .12

    // Calculate offsets (.12f)
    xs= 120*lam;
    ys= M7_D*lam;

    REG_BG2PA= (g_cosf*lam)&gt;&gt;12;
    REG_BG2PC= (g_sinf*lam)&gt;&gt;12;

    REG_BG2X = cam_pos.x - ( (xs*g_cosf-ys*g_sinf)&gt;&gt;12 );
    REG_BG2Y = cam_pos.z - ( (xs*g_sinf+ys*g_cosf)&gt;&gt;12 );   
}
</code></pre>

<pre><code class="language-c hljs">// --- Type C ---
<span class="rem">// offset * (zoom * rotation)
// Mixed fixed point: lam, lcf, lsf use .12
// lxr and lyr have different calculation methods</span>
void m7_hbl_c()
{
    FIXED lam, lcf, lsf, lxr, lyr;

    lam= cam_pos.y*lu_div(REG_VCOUNT)&gt;&gt;12;  // .8*.16 /.12 = 20.12
    lcf= lam*g_cosf&gt;&gt;8;                     // .12*.8 /.8 = .12
    lsf= lam*g_sinf&gt;&gt;8;                     // .12*.8 /.8 = .12
    
    REG_BG2PA= lcf&gt;&gt;4;
    REG_BG2PC= lsf&gt;&gt;4;

    // Offsets
    // Note that the lxr shifts down first! 

    // horizontal offset
    lxr= 120*(lcf&gt;&gt;4);      lyr= (M7_D*lsf)&gt;&gt;4;
    REG_BG2X= cam_pos.x - lxr + lyr;

    // vertical offset
    lxr= 120*(lsf&gt;&gt;4);      lyr= (M7_D*lcf)&gt;&gt;4; 
    REG_BG2Y= cam_pos.z - lxr - lyr;
}
</code></pre>

### 考察（技術的な話） {#ssec-order-disc}

3 つの版はどれも次のことをしています。式 2 と除算のルックアップテーブルを使ってズーム係数 λ を計算し、λ と保存しておいた cos(φ) と sin(φ) を使ってアフィン行列を計算し、アフィンのオフセットを計算します。実際に計算しているのは _p_<sub>a</sub> と _p_<sub>c</sub> だけであることに注意してください。スキャンラインのオフセットは実質的に常にゼロなので、_p_<sub>b</sub> と _p_<sub>d</sub> には効果がなく、無視できます。以上が共通点ですが、もっと面白いのは違いのほうです。

1.  **固定小数点**。種類 A は全体で .8 の固定小数点の計算を使いますが、B と C は .12 と .8 の固定小数点数を組み合わせて使います。
2.  **アフィンのオフセットの計算順序**。アフィンの変位 **dx** は、拡大縮小、回転、オフセットの 3 つの部分の組み合わせです。種類 A と B は **dx** = (オフセット\*拡大縮小)\*回転 を使いますが、C は **dx** = オフセット\*(拡大縮小\*回転) を使います。種類 C はオフセットを最後に処理するので、オフセットに別の固定小数点を使うこともできます。

<div class="cpt_fr">
<table id="tbl:divs" class="table-data">
  <caption align= bottom>
      <b>*@tbl:divs</b>: 除算のテーブルとズーム係数。<i>a</i><sub>y</sub>=32
  </caption>
  <tr><th>h	<th>1/h			<th>&lambda;（真の値）	<th>&lambda;(.8)
  <tr><td>157	<td>0.01a16d..h	<td>0.342da7h	<td>0.34h
  <tr><td>158	<td>0.019ec8..h	<td>0.33d91dh	<td>0.33h
  <tr><td>159	<td>0.019c2d..h	<td>0.3385a2h	<td>0.33h
  <tr><td>160	<td>0.019999..h	<td>0.333333h	<td>0.33h
</table>
</div>

この 2 つ（というより、実際には 2 つ半）の違いで、結果の違いを十分に説明できます。コードの違いはかなり微妙だということを忘れないでください。固定小数点数はゲーム機以外ではめったに使われませんし、計算の順序によって結果が変わるというのは、おそらくもっとまれでしょう。それでも、ここではこの 2 つの項目がすべてを左右しているのです。

まず種類 A と B から始めましょう。この 2 つは `lam` の固定小数点だけが違います。λ はカメラの高さとスキャンラインの比で、多くの場合かなり小さく、いずれにせよ 1 より小さくなります。@tbl:divs にいくつかの数値を示します。小数部が 8 ビットしかない λ を使うと、複数のスキャンラインで同じ数になることがよくあり、それがあとの計算にまで持ち越されることに注意してください。種類 A はお行儀よく規則に従って一定の固定小数点を使っているのに、低い高度でひどくブロック状になるのはこのためです。種類 B の余分な 4 ビットで、ずっと良い結果が得られます。規則は結構なものですが、結果を得るために破らなければならないこともあります。

さて、種類 B にもまだ少し歪みがあることに気づくでしょう。では、なぜ種類 B では .12 の固定小数点にとどめ、16 にしないのでしょうか？ 16 にすると、整数のオーバーフローで問題が起きることがあるからです。`xs` と `ys` を計算するだけなら大丈夫でしょうが、これらの値はあとで回転もさせなければなりません。よし、では 64 ビットの計算を使いましょう。そうすれば 32 ビットのオーバーフローは問題にならず、固定小数点のビットを_もっと_使えます！ 結局、多いほうが良いのですから、そうでしょう？

いいえ、違います。大きい/強い/多いが、いつも良いとは限りません（DS 対 PSP を見てください）。残っている歪みは、固定小数点のビット数の問題ではありません。少なくとも、正確にはそうではありません。128 ビットの計算と .32f の除算や三角関数のテーブルを使おうが、私の知ったことではありません。ここではそれは関係ありません。問題はそこにはないからです。

問題は、少なくともその一部は、種類 A と B で使っている基本のアルゴリズムにあります。理論を振り返ると、まずアフィン行列を計算し、それからオフセットを計算していることがわかるでしょう。言い換えると、まず拡大縮小と回転を組み合わせ、それからオフセットの補正 **P**·**x**<sub>s</sub> を計算します。いずれにせよ、GBA のアフィンパラメータはこうやって働きます。しかし、これは実は最初の段階にすぎません。その手順に従っても、やはりギザギザの結果になります。このギザギザの_本当の_原因は、`lxr` の計算の順序なのです。

```c
// Multiply, then shift to .8 (A and B)
    lxr= (120*lcf)>>4;

// Shift to .8 first, then multiply (C)
    lxr= 120*(lcf>>4);
```

`lxr` = *p*<sub>a/c</sub>·_x_<sub>s</sub> を得るには 2 つの部分が必要です。**P** の要素との乗算と、.8 の固定小数点への下向きのシフトです。シフトを最後にするほうが精度が高いので良いだろうと思うかもしれません。面白いことに、そうは**ならない**のです！ 乗算の前に _p_<sub>a</sub> や _p_<sub>c</sub> を小数部 8 ビットに落とすことで残りの歪みが消えるのであって、演算の順序を逆にしても消えません。

なぜなのかは 100% の確信はありませんが、推測はできます。アフィン変換は画面の原点を中心に行われ、原点を別の場所に置くには **x**<sub>s</sub> だけ事後の平行移動をする必要があります。決定的な点は、**x**<sub>s</sub> が、固定小数点ではなく普通の整数を使う画面空間の点だということだと思います。ただし、これが当てはまるのは _x_<sub>s</sub> だけです。それは_本当に_画面上のオフセットを表しているからです。_y_<sub>s</sub> は実際には画面上の点ではなく、カメラの焦点距離です。あるいは、変位のための内部レジスタと何か関係があるのかもしれません。

### 結論 {#ssec-order-verdict}

明らかに、使いたいのは種類 C です。自分で思いつかなかったのが本当に腹立たしいです。しかも、拡大縮小と回転の乗算を実際に_使っていた_のに、投影距離 _D_ との乗算でへまをしてそれを捨ててしまったという事実も、何の慰めにもなりません（ええ、この文はちゃんと意味が通っています）。上に示した `m7_hbl_c` のコードは、32 ビットの計算しか使っていないのにうまく動きます。拡大縮小と回転の乗算を先に行い、`wxr` の計算で 120 を掛ける前に .8 の固定小数点に落としさえすれば、すべてうまくいくはずです。

## 最後に {#sec-final}

これは、プログラミング（特に低水準のプログラミング）が科学であると同時に芸術でもあることを示す出来事の 1 つでした。3 つのモード 7 の版の理論は同じなのに、実装での計算の順序と精度のわずかな違いが、最終的な結果にとても目立つ違いを生みました。モード 7 では、補正のオフセットより先にアフィン行列を計算しましょう。そして何より大事なのは、画面の _x_ のオフセットを固定小数点で処理するべきではないということです。

次に、これはモード 7 のグラフィックの背後にある基本の理論にすぎませんでした。スプライトもピッチの角度も地平線もなく、最初から GBA のハードウェアに合わせて作ったものです。次の章では、線形代数を使った標準的な 3D の理論に従って、理論をもっと幅広く導きます。その章では、スプライトを 3D で配置する方法や、回転のためのアニメーションやソートなど、スプライトでほかのことをする方法も示し、可変のピッチと地平線も扱います。複雑そうに聞こえるなら、まあ、そのとおりだと思います。でも、見る価値は間違いなくあります。
