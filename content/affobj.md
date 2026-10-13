# 11. アフィンスプライト

<!-- toc -->

## アフィンスプライトの紹介 {#sec-intro}

<dfn>アフィンスプライト</dfn>も、本質的にはやはりスプライトです。通常のスプライトとの違いは、オブジェクト属性の適切なビットを設定して **P** 行列を埋めることで、描画の前にアフィン変換（名前の由来です）をかけられる点です。アフィン変換と **P** 行列については[こちら](affine.html)で読めます。この節を読むには、それと、[スプライトと背景の概要](objbg.html)、[通常のスプライト](regobj.html)のページを読んでいることが前提です。

これが本当に独立した節にする価値のあるものなのか、疑問に思うかもしれません。短く答えるなら、はい、です。もう少し長く答えるなら、はい、アフィンスプライトを使うには通常のスプライトよりずっと多くの数学が必要で、えーと、「数学が苦手な」人たちを怖がらせたくなかったからです。[通常のスプライト](regobj.html)の節は単独で成り立っていて、アフィンスプライトに必要な厄介な数学のことなど何も知らないまま、幸せに使えます。

この章では、オブジェクトがアフィン変換を使うように設定する方法を見ていきます。これ自体はかなり簡単です。ほかにも、遅かれ早かれ（実際には、そのうちの 1 つはほぼすぐに）出くわすかもしれないグラフィック上の問題をいくつかと、変換の原点が任意の点にあるように見せるためにスプライトの位置を補正する方法も扱います。そしていつものように、この章で取り上げるさまざまな話題を示すデモのコードもあります。

## アフィンスプライトの初期化 {#sec-init}

通常のスプライトをアフィンスプライトにするには、2 つのことをする必要があります。まず、`OBJ_ATTR.attr0{8}` を設定して、これがアフィンスプライトであることを示します。次に、`OBJ_ATTR.attr1{8-C}` に 0 から 31 の数値を入れます。この数値は、32 個あるオブジェクトのアフィン行列（`OBJ_AFFINE` 構造体）のどれを使うかを示します。忘れてしまった人のために書いておくと、`OBJ_AFFINE` は次のようになっています。

```c
typedef struct OBJ_AFFINE
{
    u16 fill0[3];
    s16 pa;
    u16 fill1[3];
    s16 pb;
    u16 fill2[3];
    s16 pc;
    u16 fill3[3];
    s16 pd;
} ALIGN4 OBJ_AFFINE;
```

*符号付き*の 16 ビットのメンバ `pa`、`pb`、`pc`、`pd` は 8.8 の固定小数点数で、これらが実際の行列を作ります。要素の名前に合わせて、この行列を **P** と呼ぶことにします。この行列について詳しくは、[アフィン行列](affine.html)の節を見てください。まだ読んでいないなら今すぐ読んでください。ここでは繰り返しませんから。単純な「拡大縮小してから回転」の行列が欲しいだけなら、これを使ってください。s<sub>x</sub> と s<sub>y</sub> で拡大縮小したあとに α だけ反時計回りに回転する正しい行列は、次のとおりです。
<!--
\vb{P} =
\begin{bmatrix}
p_{a} & p_{b} \\
p_{c} & p_{d}
\end{bmatrix} =
\begin{bmatrix}
\cos(\alpha)/s_{x} & -\sin(\alpha)/s_{x} \\
-\sin(\alpha)/s_{y} & \cos(\alpha)/s_{y}
\end{bmatrix}
-->
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi mathvariant="bold">P</mi>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <msub>
                      <mi>p</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>a</mi>
                      </mrow>
                    </msub>
                  </mtd>
                  <mtd>
                    <msub>
                      <mi>p</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>b</mi>
                      </mrow>
                    </msub>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <msub>
                      <mi>p</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>c</mi>
                      </mrow>
                    </msub>
                  </mtd>
                  <mtd>
                    <msub>
                      <mi>p</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>d</mi>
                      </mrow>
                    </msub>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
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
                      <mi>&#x3B1;</mi>
                      <mo data-mjx-texclass="CLOSE">)</mo>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <msub>
                      <mi>s</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>x</mi>
                      </mrow>
                    </msub>
                  </mtd>
                  <mtd>
                    <mo>&#x2212;</mo>
                    <mi>sin</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mrow>
                      <mo data-mjx-texclass="OPEN">(</mo>
                      <mi>&#x3B1;</mi>
                      <mo data-mjx-texclass="CLOSE">)</mo>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <msub>
                      <mi>s</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>x</mi>
                      </mrow>
                    </msub>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mo>&#x2212;</mo>
                    <mi>sin</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mrow>
                      <mo data-mjx-texclass="OPEN">(</mo>
                      <mi>&#x3B1;</mi>
                      <mo data-mjx-texclass="CLOSE">)</mo>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <msub>
                      <mi>s</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>y</mi>
                      </mrow>
                    </msub>
                  </mtd>
                  <mtd>
                    <mi>cos</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mrow>
                      <mo data-mjx-texclass="OPEN">(</mo>
                      <mi>&#x3B1;</mi>
                      <mo data-mjx-texclass="CLOSE">)</mo>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mo>/</mo>
                    </mrow>
                    <msub>
                      <mi>s</mi>
                      <mrow data-mjx-texclass="ORD">
                        <mi>y</mi>
                      </mrow>
                    </msub>
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

変換の原点は、左上の角ではなく、スプライトの*中心*であることに注意してください。スプライトをほかのオブジェクトと揃えたいときには、覚えておく価値があります。これはあとでやります。

:::tip アフィンスプライトに欠かせない手順

-   いつもどおりにオブジェクトを準備する: グラフィックとパレットを読み込み、`REG_DISPCNT` を設定し、OAM のエントリを準備する。
-   属性 0 のビット 8 を設定してそのオブジェクトのアフィンを有効にし、使うオブジェクトのアフィン行列を選ぶ（属性 1 のビット 8-12）。
-   そのオブジェクトのアフィン行列を、すべてゼロ以外のもの、たとえば単位行列に設定する。

:::

## グラフィック上のアーティファクト {#sec-artifact}

### クリッピングと離散化のアーティファクト {#ssec-clip}

GBA がスプライトを描く手順は次のとおりです。スプライトは、その大きさで決まる長方形を画面上に作ります。その領域の画面のピクセル（**q**）を塗るには、テクスチャのピクセル **p** を使います。**p** は次の式で計算されます。

<!--
\vb{p} - \vb{p}_{0} = \vb{P} \cdot (\vb{q} - \vb{q}_0)
-->
<table id="eq:aff-ofs">
<tr>
  <td class="eqnrcell">({!@eq:aff-ofs})
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi mathvariant="bold">p</mi>
            <mo>&#x2212;</mo>
            <msub>
              <mi mathvariant="bold">p</mi>
              <mrow data-mjx-texclass="ORD">
                <mn>0</mn>
              </mrow>
            </msub>
            <mo>=</mo>
            <mi mathvariant="bold">P</mi>
            <mo>&#x22C5;</mo>
            <mo stretchy="false">(</mo>
            <mi mathvariant="bold">q</mi>
            <mo>&#x2212;</mo>
            <msub>
              <mi mathvariant="bold">q</mi>
              <mn>0</mn>
            </msub>
            <mo stretchy="false">)</mo>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

ここで **p**<sub>0</sub> と **q**<sub>0</sub> は、それぞれテクスチャ空間と画面空間でのスプライトの中心です。以下のコードは、本質的にはハードウェアがしていることです。画面の長方形を、半分の幅と半分の高さのプラスからマイナスの範囲で走査し（中心が基準点なので半分の大きさです）、テクスチャのピクセルを計算して、その色を描きます。

```c
// pseudocode for affine objects
hwidth= width/2;   // half-width of object screen canvas
hheight= hheight/2;   // half-height of object screen canvas
for(iy=-hheight; iy<hheight; iy++)
{
    for(ix=-hwidth; ix<hwidth; ix++)
    {
        px= (pa*ix + pb*iy)>>8;    // get x texture coordinate
        py= (pc*ix + pd*iy)>>8;    // get y texture coordinate
        color= GetPixel(px0+px, py0+py);   // get color from (px,py)
        SetPixel(qx0+ix, qy0+iy, color);     // set color to (qx, qy)
    }
}
```

このことから、主に 2 つの結果が生じます。クリッピングのアーティファクトと、離散化のアーティファクトです。

<div class="cpt_fr" style="width:160px;">
<img src="./img/metr/aff_clipped.png" id="fig:metr-clip" 
  alt="defanged metroid"><br>
<b>{*@fig:metr-clip}</b>: 一部の牙を抜かれたメトロイド。
青い四角の外側の部分が切り落とされています。
</div>

<dfn>クリッピングのアーティファクト</dfn>は、**画面上**の長方形だけを走査することで起きます。しかし、ほとんどすべての変換でテクスチャのピクセルはその長方形からはみ出し、長方形の外側のピクセルは描画されません。{*@fig:metr-clip} は、画面上の長方形（灰色で縁が青）と、回転したオブジェクト（赤い縁の内側）を示しています。青い境界線からはみ出した部分は切り落とされます（訳注: 原文は "will not be cut off" ですが、文脈から「切り落とされる」の意味と解釈しました）。

これは明らかな欠点なので、当然回避する方法があります。スプライトのアフィンモードを**倍サイズのアフィン**（`ATTR0_AFF_DBL`、`OBJ_ATTR.attr0{8,9}`）にするのです。これで有効な **q** 座標の画面上の範囲が 2 倍になり、半分の大きさではなく、幅と高さのプラスマイナスの範囲を使えるようになります。この 2 倍（実際には 4 倍）の領域なら、中心からの最大距離は ½√2 ≈ 0.707 なので、スプライトを安全に回転できます。もちろん、2 倍の範囲を超えて拡大すれば、やはりクリッピングのアーティファクトは起きます。また、スプライトの原点がこの長方形の中心に移るので、**q**<sub>0</sub> は左上の角からスプライト 1 つぶんの大きさだけ離れることにも注意してください。

倍サイズのフラグには、もう 1 つの使い道があります。あるいは悪用と言うべきかもしれません。通常のスプライトにこのフラグを設定すると、スプライトは非表示になります。これは、使っていないスプライトを隠す別の方法です。

2 つ目のアーティファクトは、そう呼べるならですが、<dfn>離散化</dfn>のアーティファクトです。これはクリッピングのアーティファクトより微妙な点で、気づくことすらないかもしれません。ここでの問題は、変換が実際にはオブジェクトの中心ではなく、切り上げた**中心のピクセル**で行われることです。例として {@fig:numline} を見てください。0 から 8 までの数直線があり、その間に 0 から 7 までの 8 つのピクセルがあります。中心の数は、もちろん 4 です。中心のピクセルも 4 ですが、その位置は実際には数 4 と 5 のちょうど中間です。そのため、左右のピクセルの数に偏りが生じます。

中心のピクセルは変換アルゴリズムの基準点で、そのインデックスは (ix, iy) = (0, 0) です。これを式に入れてみると、数学的にはそうなるべきではないのに、このピクセルが変換で動かないことがわかります。これはオフセットに影響します。オフセットは位置ではなくピクセルから計算されるからです。{@fig:numline} では、左には 4 ピクセルありますが、右には 3 ピクセルしかありません。ピクセル 4 を中心にして鏡映すると、実質的にスプライトが 1 ピクセル右に動くことになります。

{*@fig:aff-algor} は、これが回転にどう影響するかを示しています。8 ピクセルごとの灰色のグリッド線と、箱の形をした 16×16 のスプライトが表示されています。最初の状態で、左右の辺がグリッド線の上にないことに注意してください。スプライトの幅と高さは 17 ではなく 16 だからです。ほかの図は 90° ずつ回転させたもので、行列にはきれいな切りのいい数が入ります。回転すると、中心のピクセル（真ん中の赤い点）は同じ位置にとどまり、残りがその周りを回ります。この過程で、辺はスプライトに割り当てられた 16×16 の箱（破線）の外に出てしまいます。

<div class="cblock">
<table>
<tbody valign="top">
<tr>
  <td>
  <div class="cpt_fr" style="width:288px;">
  <img src="./img/affine/numline.png" id="fig:numline" width="288"
    alt="Numbers vs pixels"><br>
  <b>{*@fig:numline}</b>: ピクセルは座標の上ではなく、
    座標の間にあります。
  </div></td>
  <td>
  <div class="cpt_fr" style="width:320px;">
  <img src="./img/affine/numalgor.png" id="fig:aff-algor" width="320"
    alt="Rotations"><br>
  <b>{*@fig:aff-algor}</b>: 90&deg; ずつの
    回転。
  </div></td>
</tr>
</tbody>
</table>
</div>

:::warning オフセットは、中心の位置ではなく中心のピクセルからの距離を測る

アフィン行列から計算されるオフセットは、中心の点ではなく、中心のピクセル（*w*/2, *h*/2）からの距離を使います。そのため、数学的な変換とは半ピクセルのずれがあり、スプライト全体が ±1 ピクセルずれたり、テクスチャの端が欠けたりすることがあります。

:::

### 回り込みのアーティファクト {#ssec-wrap}

クリッピングのアーティファクトのほかに、もう 1 つあるようです。私はどこかで触れられているのを実際に見たことがありません。私が回り込みのアーティファクトと呼んでいるものです。ご存じのとおり、スプライトの位置は 9 ビットの *x* の値と 8 ビットの *y* の値で与えられ、これらの値は画面上で回り込みます。*x* については、単に \[-256, 255\] の範囲だと解釈できます。*y* の値ではそうはいきません。符号付き 8 ビット整数の最大値は 127 なので、それでは画面の下の 32 ラインにスプライトを置けなくなってしまいます。でも、値が回り込むので、結局はすべてうまくいきます。1 つの例外を除いては。

通常のスプライトでは問題はまったく起きませんし、アフィンスプライトでもほとんど起きません。唯一の例外は、倍サイズのフラグを有効にした 64×64 か 32×64 のアフィンスプライトです。このようなスプライトの外接矩形は 128×128 になります。このとき、*y* \> 128 の意味には 3 通りの解釈があります。

1.  完全な回り込み: スプライトの上部が画面の下に表示され、その逆も同様。
2.  正の優先: \[128, 159\] の範囲は画面の下部を表すものとし、回り込みは考えない。
3.  負の優先: *y* の値によってスプライトの一部が画面の上に表示されるなら、それを負の値とみなす。やはり回り込みは考えない。

実際のところ、GBA は 3 番目の解釈を使います。つまり、次のようにしています。

```c
// pseudo code
if(oam.y + bbox_height > 256)
    oam.y -= 256;
```

ちなみに、古いエミュレータの一部（VBA と BoycottAdvance）は、どちらも解釈 2 を使っていることに注意してください。こちらのほうが論理的に見えるかもしれませんが、正しくありません。ご覧のとおり、これは 32×64 か 64×64 の倍サイズのスプライトでしか起きませんし、その場合でも、ごく特定の条件、つまり変換後のスプライトが外接矩形の上の 32 ラインに見えるピクセルを持つ場合にしか気づきません。この問題が起きた場合、私の知る限り、スプライトを画面の下に表示する唯一の方法は、その間だけ高さを 32 に減らすことです。

## とっても（アフィ）ファインなデモ {#sec-demo}

今回は *obj_aff* という、とても面白いデモを用意しました。（箱に入った）普通のメトロイドが登場し、拡大縮小、回転、せん断ができます（訳注: 原文は "scaled, rotated and scaled" ですが、操作の内容からせん断と解釈しました）。これらの変換は行列の*現在の*状態にかけられるので、いろいろな行列をつなげることで、ありとあらゆるアフィン行列を作れます。操作は次のとおりです。

<div class="lblock">
  <table cellspacing=0>
    <col valign="top">
    <tr><th>L,R<td>スプライトをそれぞれ反時計回り、時計回りに回転する。
    <tr><th>D-pad<td>スプライトをせん断する。
    <tr><th>D-pad+Sel<td>スプライトを動かす。
    <tr><th>A,B<td>それぞれ横方向、縦方向に拡大する。
    <tr><th>A,B+Sel<td>それぞれ横方向、縦方向に縮小する。
      （ボタンが足りなくなったので、こうするしかありませんでした）。
    <tr><th>Start<td>倍サイズのフラグを切り替える。a) 回転した
      スプライトの角が切り落とされなくなることと、b) 位置が
      スプライトの大きさの 1/2 だけずれることに注意。
    <tr><th>Start+Sel<td><b>P</b> を通常に戻す。
    <tr><th>Select<td>制御ボタン（A、B、Start を参照）。
  </table>
</div>

変換を続けて見ることの面白い点は、たとえば拡大縮小のあとに回転するもの（**A**=**S**·**R**）と、回転してから拡大縮小するもの（**A**=**R**·**S**）の違いが実際に見られることです。{*@fig:obj-aff-rs} と {@fig:obj-aff-sr} は、45° の回転と縦方向の 2 倍の拡大について、この違いを示しています。また、ここでは角が切り落とされていることにも注意してください。倍サイズのフラグをすでに設定しているのに、クリッピングのアーティファクトが働いています。

<div class="cblock">
<table>
<tr>
<td>
<div class="cpt_fr" style="width:240px;">
<img src="./img/demo/obj_aff_rs.png" id="fig:obj-aff-rs" 
  alt="R*S affine object."><br>
<b>{*@fig:obj-aff-rs}</b>: 
  <tt>obj_aff</tt>、<b>S</b>(1,2) のあとに <b>R</b>(45&deg;)
</div>
<td>
<div class="cpt_fr" style="width:240px;">
<img src="./img/demo/obj_aff_sr.png" id="fig:obj-aff-sr"
  alt="S*R affine object."><br>
<b>{*@fig:obj-aff-sr}</b>: 
  <tt>obj_aff</tt>、<b>R</b>(45&deg;) のあとに <b>S</b>(1,2)
</div>
</table>
</div>

*obj_aff* デモの完全なソースコードを以下に示します。かなり長いですが、その大部分は、かけられるさまざまなアフィンの状態を管理するのに必要なコードです。実際にアフィンスプライトを扱っている関数は、`init_metr()`、`get_aff_new()`、そして `objaff_test()` のゲームループの一部です。残りは基本的に、全体を動かすために必要な飾りのようなものです。

<div id="cd-obj-aff">

```c
// obj_aff.c

#include <tonc.h>
#include <stdio.h>

#include "metr.h"

OBJ_ATTR obj_buffer[128];
OBJ_AFFINE *obj_aff_buffer= (OBJ_AFFINE*)obj_buffer;


// affine transformation constants and variables
enum eAffState
{
    AFF_NULL=0, AFF_ROTATE, AFF_SCALE_X, AFF_SCALE_Y, 
    AFF_SHEAR_X, AFF_SHEAR_Y, AFF_COUNT
};

// 'speeds' of transformations
const int aff_diffs[AFF_COUNT]= { 0, 128, 4, 4, 4, 4 };
// keys for transformation direction
const int aff_keys[AFF_COUNT]=
{ 0, KEY_L, KEY_SELECT, KEY_SELECT, KEY_RIGHT, KEY_UP };
int aff_state= AFF_NULL, aff_value= 0;


void init_metr()
{
    // Places the tiles of a 4bpp metroid sprite into LOW obj VRAM
    memcpy32(tile_mem[4], metr_boxTiles, metr_boxTilesLen/4);
    memcpy32(pal_obj_mem, metrPal, metrPalLen/4);

    // Set up main metroid
    obj_set_attr(obj_buffer, 
        ATTR0_SQUARE | ATTR0_AFF,          // Square affine sprite
        ATTR1_SIZE_64 | ATTR1_AFF_ID(0),   // 64x64, using obj_aff[0]
        0 | 0);                            // palbank 0, tile 0
    obj_set_pos(obj_buffer, 96, 32);
    obj_aff_identity(&obj_aff_buffer[0]);

    // Set up shadow metroid
    obj_set_attr(&obj_buffer[1], 
        ATTR0_SQUARE | ATTR0_AFF,           // Square affine sprite
        ATTR1_SIZE_64 | ATTR1_AFF_ID(31),   // 64x64, using obj_aff[0]
        ATTR2_PALBANK(1) | 0);              // palbank 1, tile 0
    obj_set_pos(&obj_buffer[1], 96, 32);
    obj_aff_identity(&obj_aff_buffer[31]);

    oam_update_all();
}

int get_aff_state()
{
    if(key_is_down(KEY_L | KEY_R))
        return AFF_ROTATE;
    if(key_is_down(KEY_A))
        return AFF_SCALE_X;
    if(key_is_down(KEY_B))
        return AFF_SCALE_Y;
    if(key_is_down(KEY_LEFT | KEY_RIGHT))
        return AFF_SHEAR_X;
    if(key_is_down(KEY_UP | KEY_DOWN))
        return AFF_SHEAR_Y;
    return AFF_NULL;
}

void get_aff_new(OBJ_AFFINE *oa)
{
    int diff= aff_diffs[aff_state];
    aff_value += (key_is_down(aff_keys[aff_state]) ? diff : -diff);

    switch(aff_state)
    {
    case AFF_ROTATE:    // L rotates left, R rotates right
        aff_value &= SIN_MASK;
        obj_aff_rotate(oa, aff_value);
        break;
    case AFF_SCALE_X:   // A scales x, +SELECT scales down
        obj_aff_scale_inv(oa, (1<<8)-aff_value, 1<<8);
        break;
    case AFF_SCALE_Y:   // B scales y, +SELECT scales down
        obj_aff_scale_inv(oa, 1<<8, (1<<8)-aff_value);
        break;
    case AFF_SHEAR_X:   // shear left and right
        obj_aff_shearx(oa, aff_value);
        break;
    case AFF_SHEAR_Y:   // shear up and down
        obj_aff_sheary(oa, aff_value);
        break;
    default:            // shouldn't happen
        obj_aff_identity(oa);
    }
}

void objaff_test()
{
    OBJ_ATTR *metr= &obj_buffer[0], *shadow= &obj_buffer[1];
    OBJ_AFFINE *oaff_curr= &obj_aff_buffer[0]; 
    OBJ_AFFINE *oaff_base= &obj_aff_buffer[1]; 
    OBJ_AFFINE *oaff_new=  &obj_aff_buffer[2];

    int x=96, y=32;
    int new_state;

    // oaff_curr = oaff_base * oaff_new
    // oaff_base changes when the aff-state changes
    // oaff_new is updated when it doesn't
    obj_aff_identity(oaff_curr);
    obj_aff_identity(oaff_base);
    obj_aff_identity(oaff_new);

    while(1)
    {
        key_poll();

        // move sprite around
        if( key_is_down(KEY_SELECT) && key_is_down(KEY_DIR) )
        {
            // move
            x += 2*key_tri_horz();
            y += 2*key_tri_vert();

            obj_set_pos(metr, x, y);
            obj_set_pos(shadow, x, y);
            new_state= AFF_NULL;
        }
        else    // or do an affine transformation
            new_state= get_aff_state();

        if(new_state != AFF_NULL)   // no change
        {
            if(new_state == aff_state)  // increase current transformation
            {
                get_aff_new(oaff_new);
                obj_aff_copy(oaff_curr, obj_aff_base, 1);
                obj_aff_postmul(oaff_curr, oaff_new);
            }
            else        // switch to different transformation type
            {
                obj_aff_copy(oaff_base, oaff_curr, 1);
                obj_aff_identity(oaff_new);
                aff_value= 0;
            }
            aff_state= new_state;
        }

        // START: toggles double-size flag
        // START+SELECT: resets obj_aff to identity
        if(key_hit(KEY_START))
        {
            if(key_is_down(KEY_SELECT))
            {
                obj_aff_identity(oaff_curr);
                obj_aff_identity(oaff_base);
                obj_aff_identity(oaff_new);
                aff_value= 0;
            }
            else
            {
                metr->attr0 ^= ATTR0_DBL_BIT;
                shadow->attr0 ^= ATTR0_DBL_BIT;
            }
        }

        vid_vsync();

        // we only have one OBJ_ATTR, so update that
        obj_copy(obj_mem, obj_buffer, 2);

        // we have 3 OBJ_AFFINEs, update these separately
        obj_aff_copy(obj_aff_mem, obj_aff_buffer, 3);

        // Display the current matrix
        tte_printf("#{es;P:8,136}P =  "
            "#{y:-7;Ps}| %04X\t%04X#{Pr;x:72}|"
            "#{Pr;y:12}| %04X\t%04X#{Pr;p:72,12}|", 
            (u16)oaff_curr->pa, (u16)oaff_curr->pb, 
            (u16)oaff_curr->pc, (u16)oaff_curr->pd);
    }
}

int main()
{
    REG_DISPCNT= DCNT_BG0 | DCNT_OBJ | DCNT_OBJ_1D;
    oam_init(obj_buffer, 128);
    init_metr();

    tte_init_chr4_b4_default(0, BG_CBB(2)|BG_SBB(28));
    tte_init_con();
    tte_set_margins(8, 128, 232, 160);

    objaff_test();

    return 0;
}
```
</div>

メトロイドをアフィンスプライトにする処理は、すべて `init_metr()` の中で行っています。ビットの設定方法はもう何度も見てきたので、理解できるはずです。とはいえ、（スプライトが使う）最初の `OBJ_AFFINE` を単位行列 **I** にしていることには注意してください。これをすべてゼロのままにしておくと、64×64 ピクセルの単色の長方形になってしまいます。**P** にはピクセルのオフセットが入っていることを思い出してください。それがすべてゼロなら、オフセットがないので、原点の色が全体に使われます。要するに、スプライトが無限大に拡大されるのです。

ただ、正直に言うと、`oam_init()` を呼んだあとなら `obj_aff_identity()` を呼ぶ必要はありません。この関数は行列も初期化するからです。それでも、起こりうる問題は知っておく必要があります。

準備は以上です。次は、このデモがどうやって動いているかです。どの時点でも、何らかの変換行列 **P** があります。ボタンを押すと（あるいは押さないと）、行列の積によって、現在の状態に小さな変換がかけられます。

<!--
\vb{P}_\text{new} = \vb{P}_{old} \cdot \vb{D}^{-1}
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
              <mi mathvariant="bold">P</mi>
              <mtext>new</mtext>
            </msub>
            <mo>=</mo>
            <msub>
              <mi mathvariant="bold">P</mi>
              <mtext>old</mtext>
            </msub>
            <mo>&#x22C5;</mo>
            <msup>
              <mi mathvariant="bold">D</mi>
              <mrow data-mjx-texclass="ORD">
                <mo>&#x2212;</mo>
                <mn>1</mn>
              </mrow>
            </msup>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

ここで **D** は、小さな回転（**R**）、拡大縮小（**S**）、せん断（**H**）のどれか、あるいは何もしない変換（**I**）です。ただし、ここにはちょっとした問題があります。理論上はうまくいくのですが、*実際には*、固定小数点の行列の積によって、受け入れがたい丸め誤差があっという間にたまってしまうので、うまくいきません。幸い、これらの変換にはどれも次のような便利な性質があります。
<!--
\vb{D}(a)\cdot\vb{D}(b) = \vb{D}(c)
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
            <mi mathvariant="bold">D</mi>
            <mo stretchy="false">(</mo>
            <mi>a</mi>
            <mo stretchy="false">)</mo>
            <mo>&#x22C5;</mo>
            <mi mathvariant="bold">D</mi>
            <mo stretchy="false">(</mo>
            <mi>b</mi>
            <mo stretchy="false">)</mo>
            <mo>=</mo>
            <mi mathvariant="bold">D</mi>
            <mo stretchy="false">(</mo>
            <mi>c</mi>
            <mo stretchy="false">)</mo>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

つまり、複数の小さな変換は 1 つの大きな変換として働くということです。必要なのは、現在選ばれている変換（`get_aff_state()` の中の変数 `aff_state`）を記録しておき、状態の変数（`aff_value`）を変更して、変換行列全体を計算し（`get_aff_new()`）、それをかける（`obj_aff_postmul()` で）ことだけです。別の種類の変換が選ばれたら、現在の行列を保存し、状態の値をリセットして、また別の変換が選ばれるまでその状態で続けます。コードの大部分は、こうした変化を記録するためのものです。きれいではありませんが、仕事はこなします。

## 中心以外の基準点とオブジェクトの組み合わせ {#sec-combo}

<div class="cpt_fr" style="width:240px;">
<img src="./img/metr/rot_ofs.png" id="fig:rot-ofs" 
  alt="Rotation around off-center point"><br>
<b>{*@fig:rot-ofs}</b>: 中心以外の点の周りでの
  オブジェクトの回転。
</div>

前に述べたように、アフィンスプライトは常に中心をアフィンの原点として使います。しかし、何か別のものを中心に回転させたい、つまり別の点を基準点として使いたいときもあります。実際にはそうすることはできませんが、できるように*見せる*ことはできます。そのためには、私がアンカリングと呼んでいるものについて、いくつか説明する必要があります。<dfn>アンカー</dfn>とは、「固定」されたままであるべき位置のことです。テクスチャ（この場合はオブジェクト）が画面に固定される場所です。

アンカリングには、実際には使う座標空間ごとに 1 組の座標が必要です。この場合は 2 つ、テクスチャ空間と画面空間です。これらの点をそれぞれ **p**<sub>0</sub> と **q**<sub>0</sub> と呼ぶことにしましょう。これらが実際にどこ*から*指しているかはほとんど重要ではありませんが、便宜上、画面とテクスチャの原点を使うことにします。これらの点は始まりにすぎません。手順全体では、全部で *7 つ*のベクトルを考える必要があり、それらはすべて {@fig:rot-ofs} に描かれています。それぞれの意味は、下の表で説明します。

<div class="lblock">

<table border=0 cellpadding=1 cellspacing=0>
<colgroup>
    <col span="1" style="width: 14%;">
    <col span="1" style="width: 86%;">
</colgroup>
<tbody valign="top">
<tr align="left"> <th width=48>点</th>		<th>説明</th> </tr>
<tr>
  <td> <b>p</b><sub>0</sub>, <b>q</b><sub>0</sub> </td>
  <td> テクスチャ空間と画面空間でのアンカー。 </td>
</tr>
<tr>
  <td> <b>c</b><sub>p</sub>, <b>c</b><sub>q</sub> </td>
  <td> 
    テクスチャ空間と画面空間でのオブジェクトの中心。
    オブジェクトの大きさを <b>s</b>=(w,h) とすると、
    <b>c</b><sub>p</sub>=&frac12;<b>s</b>、
	<b>c</b><sub>q</sub>=<i>m</i><b>s</b> となります。ここで <i>m</i> は
	  倍サイズのフラグに応じて &frac12; か 1 です。
  </td>
</tr>
<tr>
  <td> <b>r</b><sub>p</sub>, <b>r</b><sub>q</sub> </td>
  <td> オブジェクトの中心とアンカーの間の距離。定義により、
	<b>r</b><sub>p</sub> = <b>P</b>·<b>r</b><sub>q</sub></td>
</tr>
<tr>
  <td> <b>x</b> </td>	<td>求めたいオブジェクトの座標。</td>
</tr>
</tbody>
</table>
</div>

たしかにベクトルがたくさんありますが、面白いことに、そのほとんどはすでにわかっています。中心の点（**c**<sub>p</sub> と **c**<sub>q</sub>）はオブジェクトの大きさと倍サイズの状態から求められますし、アンカーは入力値なので前もってわかっています。そして **r**<sub>p</sub> と **r**<sub>q</sub> はアフィン変換の一般式 {@eq:aff-ex-base} に当てはまるので、これが 2 つの空間を結びつけます。あとは連立方程式を書き出して解くだけです。
<!--
\begin{matrix}
\vb{x} + \vb{c}_{q} + \vb{r}_{q} & = & \vb{q}_{0} \\
\vb{c}_{p} + \vb{r}_{p} & = & \vb{p}_{0} \\
\vb{r}_{p} & = & \vb{P} \cdot\ \vb{r}_{q} \\
\end{matrix}
-->
<table id="eq:aff-ex-base">
<tr>
  <td class="eqnrcell">({!@eq:aff-ex-base})</td>
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
                  <mi mathvariant="bold">x</mi>
                  <mo>+</mo>
                  <msub>
                    <mi mathvariant="bold">c</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>q</mi>
                    </mrow>
                  </msub>
                  <mo>+</mo>
                  <msub>
                    <mi mathvariant="bold">r</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>q</mi>
                    </mrow>
                  </msub>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msub>
                    <mi mathvariant="bold">q</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mn>0</mn>
                    </mrow>
                  </msub>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <msub>
                    <mi mathvariant="bold">c</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>p</mi>
                    </mrow>
                  </msub>
                  <mo>+</mo>
                  <msub>
                    <mi mathvariant="bold">r</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>p</mi>
                    </mrow>
                  </msub>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <msub>
                    <mi mathvariant="bold">p</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mn>0</mn>
                    </mrow>
                  </msub>
                </mtd>
              </mtr>
              <mtr>
                <mtd>
                  <msub>
                    <mi mathvariant="bold">r</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>p</mi>
                    </mrow>
                  </msub>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mi mathvariant="bold">P</mi>
                  <mo>&#x22C5;</mo>
                  <mtext>&#xA0;</mtext>
                  <msub>
                    <mi mathvariant="bold">r</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>q</mi>
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
</tr>
</table>

未知数が 3 つで式が 3 つなので、解けます。導出はそれほど難しくないので全部は載せません。{@eq:aff-ex} は、最も使いやすい形にした最終結果です。
<!--
\vb{x} = \vb{q}_{0} - m\vb{s} - \vb{P}^{-1} \cdot (\vb{p}_{0} - \tfrac{1}{2} s)
-->
<table id="eq:aff-ex">
<tr>
  <td class="eqnrcell">({!@eq:aff-ex})
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mi mathvariant="bold">x</mi>
            <mo>=</mo>
            <msub>
              <mi mathvariant="bold">q</mi>
              <mrow data-mjx-texclass="ORD">
                <mn>0</mn>
              </mrow>
            </msub>
            <mo>&#x2212;</mo>
            <mi>m</mi>
            <mi mathvariant="bold">s</mi>
            <mo>&#x2212;</mo>
            <msup>
              <mi mathvariant="bold">P</mi>
              <mrow data-mjx-texclass="ORD">
                <mo>&#x2212;</mo>
                <mn>1</mn>
              </mrow>
            </msup>
            <mo>&#x22C5;</mo>
            <mo stretchy="false">(</mo>
            <msub>
              <mi mathvariant="bold">p</mi>
              <mrow data-mjx-texclass="ORD">
                <mn>0</mn>
              </mrow>
            </msub>
            <mo>&#x2212;</mo>
            <mstyle displaystyle="false" scriptlevel="0">
              <mfrac>
                <mn>1</mn>
                <mn>2</mn>
              </mfrac>
            </mstyle>
            <mi>s</mi>
            <mo stretchy="false">)</mo>
          </mtd>
        </mtr>
      </mtable>
    </mrow>
  </mstyle>
</math>
</table>

この右辺には 3 つの別々のベクトルがあり、そのうち 2 つは入力の一部です。それに倍サイズモードのためのスケールのフラグと、アフィン行列の逆行列があります。そう、逆行列と言いました。これが出てくるのは、オブジェクトを正しい位置に置くための平行移動が、主に画面空間で行われるからです。逆行列を使う項全体は、単に **r**<sub>q</sub>、つまりテクスチャ空間でのアンカーと中心の差を変換したもので、最後の補正にはこれが必要です。

さて、この逆行列には 2 つの意味があります。1 つ目は、おそらく行列を *2 つ*、つまりアフィン行列そのものとその逆行列を用意しなければならないということです。一般の行列では、これには時間がかかるかもしれません。特に拡大縮小をしたいなら、どこかで割り算をしなければならないことを考えるとなおさらです。2 つ目は、行列の要素には 16 ビットしかないので、逆行列は*正確な*逆行列にはならず、オブジェクトをぴったり揃えるのは、不可能とまでは言わないまでも難しいということです。これはハードウェアそのものによってほぼ確実にそうなるもので、この点にはあとでまた戻ります。今は、2 方向の拡大縮小のあとに回転する場合について、{@eq:aff-ex} を実装した関数を見てみましょう。

<div id="cd-oe-rs-ex">

```c
// === in tonc_types.h ===

// This is the same struct that's used in BgAffineSet, 
// where it is called BGAffineSource, even though its uses go 
// beyond just backgrounds.
typedef struct tagAFF_SRC_EX
{
    s32 tex_x, tex_y;   // vector p0: anchor in texture space (.8f)
    s16 scr_x, src_y;   // vector q0: anchor in screen space (.0f)
    s16 sx, sy;         // scales (Q.8)
    u16 alpha;          // CCW angle ( integer in [0,0xFFFF] )
} AFF_SRC_EX;

// === in tonc_core.c === 
// Usage: oam_sizes[shape][size] is (w,h)
const u8 oam_sizes[3][4][2]=
{
    { { 8, 8}, {16,16}, {32,32}, {64,64} }, 
    { {16, 8}, {32, 8}, {32,16}, {64,32} },
    { { 8,16}, { 8,32}, {16,32}, {32,64} },
};

// === in tonc_obj_affine.c ===
void obj_rotscale_ex(OBJ_ATTR *obj, OBJ_AFFINE *oa, AFF_SRC_EX *asx)
{
    int sx= asx->sx, sy= asx->sy;
    int sina= lu_sin(asx->alpha)>>4, cosa= lu_cos(asx->alpha)>>4;

    // (1) calculate P
    oa->pa= sx*cosa>>8;     oa->pb= -sx*sina>>8;
    oa->pc= sy*sina>>8;     oa->pd=  sy*cosa>>8;

    // (2) set-up and calculate A= P^-1
    // sx = 1/sx, sy = 1/sy (.12f)
    sx= Div(1<<20, sx);
    if(sx != sy)
        sy= Div(1<<20, sy);
    else
        sy= sx;
    FIXED aa, ab, ac, ad;   // .8f
    aa=  sx*cosa>>12;   ab= sy*sina>>12;
    ac= -sx*sina>>12;   ad= sy*cosa>>12;

    // (3) get object size
    sx= oam_sizes[obj->attr0>>14][obj->attr1>>14][0];
    sy= oam_sizes[obj->attr0>>14][obj->attr1>>14][1];

    // (4) calculate dx =  q0 - ms - A*(p0-s/2)
    int dx= asx->src_x, dy= asx->src_y;   // .0f
    if(obj->attr0&ATTR0_DBL_BIT)
    {   dx -= sx;       dy -=sy;        }
    else
    {   dx -= sx>>1;    dy -= sy>>1;    }

    sx= asx->tex_x - (sx<<7);      // .8f
    sy= asx->tex_y - (sy<<7);      // .8f
    dx -= (aa*sx + ab*sy)>>16;      // .0 - (.8f*.8f/.16f)
    dy -= (ac*sx + ad*sy)>>16;      // .0 - (.8f*.8f/.16f)

    // (5) update OBJ_ATTR
    obj_set_pos(obj, dx, dy);
}
```
</div>

`AFF_SRC_EX` 構造体と `oam_sizes` 配列は、位置合わせをする関数 `obj_rotscale_ex()` を支えるものです。この関数はアフィン行列（`pa-pd`）を作り、{@eq:aff-ex} に必要な手順をすべて実行します。つまり、逆行列 **A**（`aa-ad`）を作り、すべてのオフセットを計算して大きさを補正し、最後に `OBJ_ATTR` を更新します。固定小数点の精度があちこちで大きく変わるので、こまめにコメントを書いておくことが大事です。

先ほど言ったように、これは特に速い関数ではありません。だいたいスキャンライン 1 本ぶんのサイクルがかかります。もっと速さが必要なら、約 40% 速い Thumb アセンブリ版も用意してあります。

### アフィンオブジェクトの組み合わせのデモ {#ssec-combo-demo}

<div class="cpt_fr" style="width:64px;">
<img src="./img/demo/oac_orb.png" id="fig:oac-orb"
  width=64 alt=""><br>
<b>{*@fig:oac-orb}</b>: <tt>oacombo</tt> のオブジェクト。
</div>

この節のデモ *oacombo* は、本質的には同じオブジェクト、つまり {@fig:oac-orb} の円を 3 通り表示します。違いは、それぞれの組み立て方です。

0.  32×32p のオブジェクト 1 つ。完全な円。
1.  32×16p のオブジェクト 2 つ。2 つの半円。
2.  16×16p のオブジェクト 4 つ。4 つの四分円。

このデモの目的は、それらを回転させ、組み合わせたスプライト（<dfn>オブジェクトの組み合わせ</dfn>）の部品を、あたかも 1 つのスプライトであるかのように配置することです。これには中心以外のアンカーが必要なので、この節の話題とうまく結びつきます。組み合わせを管理するために、次の構造体を使います。

```c
typedef struct OACOMBO
{
    OBJ_ATTR *sub_obj; // obj pointer for sub-objects
    POINT *sub_pos;     // Local sub-object coords (.8f)
    int sub_count;      // Number of sub-objects
    POINT pos;          // Global position (.8f)
    POINT anchor;       // Local anchor (.8f)
    s16 sx, sy;         // scales (.8f)
    u16 alpha;          // CCW angle
} OACOMBO;
```

それぞれの組み合わせは `sub_count` 個のオブジェクトでできています。`sub_oe` はこれらのオブジェクトを格納する配列へのポインタで、`sub_pos` は、スプライト全体の左上を基準にした、これらのオブジェクトの（左上の）座標のリストへのポインタです。この全体の位置は `pos` に入っています。アンカー（`anchor` に入っています）も、この位置を基準にしたものです。画面全体でのアンカーは `pos+anchor` に、サブオブジェクト *ii* のテクスチャでのアンカーは `anchor-sub_pos[ii]` になります。

回転は円の中心の周りで行うので、アンカーは (16,16) です。正確には .8 の固定小数点数なので (16,16)\*256 ですが、今それは重要ではありません。完全な円では、これはオブジェクトの中心になりますが、それでも倍サイズのフラグに合わせて補正する必要があります。ほかの組み合わせでは、アンカーはサブオブジェクトの中心には*ありません*。

サブオブジェクトは同じ **P** 行列を共有するので、毎回それを計算し直すのは無駄です。そこで、`OACOMBO` 構造体専用に手を加えた `oac_rotscale()` という版を使っています。とはいえ、コードは基本的に同じです。`oacs[]` 配列が 3 つの組み合わせで、定義時に初期化しています。そのほうがずっと楽だからです。完全な円は (16,20)、半円は (80,20)、四分円でできたものは (48,60) にあります。`obj_data[]` 配列には 7 つのオブジェクトのデータが入っていて、初期化関数で `obj_buffer` にコピーされます。一般に、マジックナンバー（OAM の属性に 16 進数を使うなど）は悪だというのは本当ですが、この話の本筋ではありませんし、それらすべてを「正しい」やり方で初期化するのに多くの紙幅を割くのは、かえって害になるかもしれません…今回は。ただし、アンカーと基準点にはやはり `#define` を使っています。コードの残りの部分で何度も出てくるからです。

<div id="cd-oacombo">

```c
// oacombo.c

#include <stdio.h>
#include <tonc.h>


#include "oac_gfx.h"

#define AX   (16<<8)     // X-anchor
#define AY   (16<<8)     // Y-anchor
#define X0    120        // base X
#define Y0    36         // base Y

// === GLOBALS ========================================================

OBJ_ATTR obj_buffer[128];
OBJ_AFFINE *obj_aff_buffer= (OBJ_AFFINE*)obj_buffer;

// Obj templates
const OBJ_ATTR obj_data[7]=
{
    // obj[0]  , oaff[0]: 1 full 32x32p double-affine circle
    { 0x0300, 0x8200, 0x0000, 0x0000 }, 
    // obj[1-2], oaff[1]: 2 32x16p double-affine semi-circles 
    { 0x4300, 0x8200, 0x0000, 0x0000 }, 
    { 0x4300, 0x8200, 0x0008, 0x0000 }, 
    // obj[3-7], oaff[1]: 4 16x16p double-affine quarter-circles
    { 0x0300, 0x4400, 0x0010, 0x0000 }, 
    { 0x0300, 0x4400, 0x0014, 0x0000 }, 
    { 0x0300, 0x4400, 0x0018, 0x0000 }, 
    { 0x0300, 0x4400, 0x001C, 0x0000 }, 
};

POINT sub_pos[7]=
{
    {0,0},
    {0,0},{0,AY}, 
    {0,0},{AX,0}, {0,AY},{AX,AY}, 
};

OACOMBO oacs[3]=
{
    // full 32x32p double-affine circle
    { &obj_buffer[0], &sub_pos[0], 1, 
        {(X0-48)<<8, Y0<<8}, {AX, AY}, 256, 256, 0 },
    // 2 32x16p double-affine semi-circles
    { &obj_buffer[1], &sub_pos[1], 2, 
        {(X0+16)<<8, Y0<<8}, {AX, AY}, 256, 256, 0 },
    // 4 16x16p double-affine quarter-circles
    { &obj_buffer[3], &sub_pos[3], 4, 
        {(X0-16)<<8, (Y0+40)<<8}, {AX, AY}, 256, 256, 0 },
};

void oac_rotscale(OACOMBO *oac)
{
    int alpha= oac->alpha;
    int sx= oac->sx, sy= oac->sy;
    int sina= lu_sin(alpha)>>4, cosa= lu_cos(alpha)>>4;

    // --- create P ---
    OBJ_AFFINE *oaff= 
        &obj_aff_buffer[BF_GET(oac->sub_obj->attr1, ATTR1_AFF_ID)];
    oaff->pa=  cosa*sx>>8;    oaff->pb= -sina*sx>>8;
    oaff->pc=  sina*sy>>8;    oaff->pd=  cosa*sy>>8;

    // --- create A ---
    // sx = 1/sx, sy = 1/sy (.12f)
    sx= Div(1<<20, sx);
    if(sx != sy)
        sy= Div(1<<20, sy);
    else
        sy= sx;
    FIXED aa, ab, ac, ad;
    aa=  sx*cosa>>12;   ab= sy*sina>>12;    // .8f
    ac= -sx*sina>>12;   ad= sy*cosa>>12;    // .8f

    int ii;
    OBJ_ATTR *obj= oac->sub_obj;
    POINT *pt= oac->sub_pos;
    // --- place each sub-object ---
    for(ii=0; ii<oac->sub_count; ii++)
    {
        int dx, dy;     // all .8f
        sx= oam_sizes[obj->attr0>>14][obj->attr1>>14][0]<<7;
        sy= oam_sizes[obj->attr0>>14][obj->attr1>>14][1]<<7;

        dx= oac->pos.x+oac->anchor.x - sx;  // .8f
        dy= oac->pos.y+oac->anchor.y - sy;  // .8f

        if(obj->attr0&ATTR0_DBL_BIT)
        {   dx -= sx;   dy -= sy;   }

        sx= oac->anchor.x - pt->x - sx;
        sy= oac->anchor.y - pt->y - sy;

        dx -= (aa*sx + ab*sy)>>8;       // .8f
        dy -= (ac*sx + ad*sy)>>8;       // .8f      
        BF_SET(obj->attr0, dy>>8, ATTR0_Y);
        BF_SET(obj->attr1, dx>>8, ATTR1_X);

        obj++;   pt++;
    }
}

void init_main()
{
    memcpy32(pal_obj_mem, oac_gfxPal, oac_gfxPalLen/4);
    memcpy32(tile_mem[4], oac_gfxTiles, oac_gfxTilesLen/4);

    // init objs and obj combos
    oam_init();
    memcpy32(obj_buffer, obj_data, sizeof(obj_data)/4);

    REG_DISPCNT= DCNT_BG0 | DCNT_OBJ | DCNT_OBJ_1D;

    tte_init_chr4_b4_default(0, BG_CBB(2)|BG_SBB(28));
    tte_init_con();

    // Some labels
    tte_printf("#{P:%d,%d}1 full #{P:%d,%d}2 semi #{P:%d,%d}4 quarts", 
        X0-48, Y0-16, X0+20, Y0-16, X0-20, Y0+74);
}

int main()
{
    init_main();

    int ii, alpha=0;
    while(1)
    {
        vid_vsync();
        key_poll();
        alpha -= 128*key_tri_shoulder();

        for(ii=0; ii<3; ii++)
        {
            oacs[ii].alpha= alpha;
            oac_rotscale(&oacs[ii]);
        }
        oam_copy(oam_mem, obj_buffer, 128);
    }
    return 0;
}
```
</div>

<div class="cpt_fr" style="width:272px;">
<img src="./img/demo/oacombo.png" id="fig:oacombo" width=272
  alt=""><br>
<b>{*@fig:oacombo}</b>: 動作中の <tt>oacombo</tt>。
隙間に注目してください。
</div>

右の {*@fig:oacombo} は、このデモのスクリーンショットです。ここで指摘しておくべき点が主に 3 つあります。1 つ目は、3 つのオブジェクトがどれもたしかにほぼ同じ形をしていること、つまり関数がきちんと動いていることです。もっとも、数学のとおりにしているだけなので、これはそもそもあまり疑う余地のないことでした。2 つ目は、半円と四分円の組み合わせに隙間ができているように見えることです。しばらく自分でデモを触ってみると、これらの隙間が一見でたらめに現れたり消えたりするのがわかるでしょう。一方、完全な円のオブジェクトは、ずっと問題なく見えます。まあ、だいたいは。

この原因は 3 つ目の点に関係しています。3 つの円すべてのピクセルの集まり、特にそれぞれの中にある小さな円を比べてみてください。**まったく**同じ **P** 行列を使っているのに、形が違うことに注目してください！ これは、サブオブジェクトを並べて大きなオブジェクトを形作ったとしても、それぞれのピクセルの写像は*やはり*それぞれの中心から始まるからです。そのため、画面のあるピクセルにどの元のピクセルを使うかを決める累積のオフセットが変わり、違う絵になってしまうのです。これは特に継ぎ目で目立ちます。

これが少し思い描きにくければ、次のことを試してみてください。ビットマップエディタを開いて、幅 1 ピクセルの斜めの線を引きます。次に、それを (1, 1) ピクセルずらして複製します。すると、1 本の太い線ではなく、間に細い隙間のある 2 本の細い線になります。ここでも同じことが起きているのです。

要するに、アフィンオブジェクトを継ぎ目でぴったり揃えるのは、ほぼ不可能だということです。まあ、簡単な場合ならなんとかなるかもしれませんし、テクスチャを補正して正しく揃えるコードを書くのに時間をかけることもできますが、一般的には、ハードウェアによる約 1 ピクセルの不確かさがあると考えておくべきです。これは、中心以外の基準点では少しぐらつくという形で、アフィンオブジェクトの組み合わせの継ぎ目では隙間という形で、目に見える影響になります。前者の簡単な解決策は、基準点が中心から外れないようにオブジェクトのタイルを並べ替えるか（安直に聞こえるのはわかっていますが、見事にうまくいきます）、そこに透明なピクセルを置くことです。結局のところ、見えないものがぐらついても気づけませんから。これは組み合わせにも使えますし、組み合わせではオブジェクトを少し重ねるのも効果があるかもしれません。私はまだ試していませんが。計算に丸めの項を加えることで精度を少し上げられる*かもしれません*が、それほど効果はないだろうという気がします。とはいえ、自由に試してみてください。

アフィンオブジェクトの落とし穴の話ばかりで、あまり気を落とさないでください。期待していたほど単純ではないかもしれない、と指摘しているだけです。いくつか条件が付くとはいえ、やはりかなりかっこいい効果です。これを使うゲームを設計するときは、この章で取り上げた問題をしっかり心に留めて、数学をきちんと整理しておきましょう。あとで多くの手間を省けるかもしれません。
