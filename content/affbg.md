# 12. アフィン背景

<!-- toc -->

## はじめに {#sec-intro}

この節では、<dfn>アフィン背景</dfn>を扱います。**P** 行列によってアフィン変換をかけられる背景です。そして、できるのはそれだけです。[スプライトと背景の概要](objbg.html)と、[通常の背景](regbg.html)と[アフィン変換の行列](affine.html)の節をまだ読んでいない（そして理解していない！）なら、先に進む前に読んでください。

通常の背景の作り方を知っていて、アフィン行列の背後にある考え方を理解していれば、ここで困ることはほとんどないはずです。アフィン背景の背後にある理論は通常の背景と同じですが、実際の使い方はいくつかのとても重要な点で違うことがあります。たとえば、位置の指定には別のレジスタを使いますし、マップの配置と形式もどちらも違います。

GBA にある 4 つの背景のうち、アフィン背景として使えるのは最後の 2 つだけで、それも特定のビデオモードに限られます（{@tbl:bg-types} を見てください）。アフィン背景では大きさも違います。大きさの一覧は {@tbl:bga-size} にあります。

<div class="cblock">
  <table>
    <tr>
      <td>
        <table id="tbl:bg-types" border=1 cellpadding=2 cellspacing=0 width=128 class="table-data">
          <caption align="bottom"><b>{*@tbl:bg-types}</b>: ビデオモードと背景の種類</caption>
          <tbody align="center">
            <tr>
              <th>モード</th>
              <th>0</th>
              <th>1</th>
              <th>2</th>
            </tr>
            <tr>
              <th>bg0</th>
              <td>reg</td>
              <td>reg</td>
              <td>-</td>
            </tr>
            <tr>
              <th>bg1</th>
              <td>reg</td>
              <td>reg</td>
              <td>-</td>
            </tr>
            <tr>
              <th>bg2</th>
              <td>reg</td>
              <td>aff</td>
              <td>aff</td>
            </tr>
            <tr>
              <th>bg3</th>
              <td>reg</td>
              <td>-</td>
              <td>aff</td>
            </tr>
          </tbody>
        </table>
      </td>
      <td width="10%"></td>
      <td style="vertical-align: top;">
        <table id="tbl:bga-size" border=1 cellpadding=2 cellspacing=0 width=144 class="table-data">
          <caption align="bottom"><b>{*@tbl:bga-size}</b>: アフィン背景の大きさ</caption>
          <col>
          <col class="def">
          <tbody align="center">
            <tr>
              <th>Sz</th>
              <th>定義</th>
              <th>（タイル）</th>
              <th>（ピクセル）</th>
            </tr>
            <tr>
              <td>00</td>
              <td>BG_AFF_16x16</td>
              <td>16x16</td>
              <td>128x128</td>
            </tr>
            <tr>
              <td>01</td>
              <td>BG_AFF_32x32</td>
              <td>32x32</td>
              <td>256x256</td>
            </tr>
            <tr>
              <td>10</td>
              <td>BG_AFF_64x64</td>
              <td>64x64</td>
              <td>512x512</td>
            </tr>
            <tr>
              <td>11</td>
              <td>BG_AFF_128x128</td>
              <td>128x128</td>
              <td>1024x1024</td>
            </tr>
          </tbody>
        </table>
      </td>
    </tr>
  </table>
</div>

## アフィン背景のレジスタ {#sec-regs}

通常の背景と同じく、アフィン背景を主に制御するのは `REG_BGxCNT` です。何をするものか忘れてしまったなら、[こちら](regbg.html#tbl-reg-bgxcnt)で説明を読めます。通常の背景との違いは大きさと、`BG_WRAP` が今度は実際に働くことです。ほかの重要なレジスタは、<dfn>変位ベクトル</dfn> **dx**（`REG_BGxX` と `REG_BGxY`）と、<dfn>アフィン行列</dfn> **P**（`REG_BGxPA`-`REG_BGxPD`）です。それらのアドレスは {@tbl:aff-regs} にあります。

<div class="lblock">
  <table id="tbl:aff-regs" class="table-data">
    <caption align="bottom"><b>{*@tbl:aff-regs}</b>: アフィン背景のレジスタのアドレス。<i>x</i> は 2 か 3 だけであることに注意！</caption>
    <col span=2 align="right">
    <tr>
      <th>レジスタ</th>
      <th>長さ</th>
      <th>アドレス</th>
    </tr>
    <tr>
      <th>REG_BGxCNT</th>
      <td>2</td>
      <td>0400:0008h + 2·x</td>
    </tr>
    <tr>
      <th>REG_BGxPA-PD</th>
      <td>2</td>
      <td>0400:0020h + 10h·(x-2)</td>
    </tr>
    <tr>
      <th>REG_BGxX</th>
      <td>4</td>
      <td>0400:0028h + 10h·(x-2)</td>
    </tr>
    <tr>
      <th>REG_BGxY</th>
      <td>4</td>
      <td>0400:002ch + 10h·(x-2)</td>
    </tr>
  </table>
</div>

アフィン背景の変位と変換については、注意すべき点がいくつかあります。まず、変位 *dx* には通常の背景とは別のレジスタを使います。`REG_BGxHOFS` と `REG_BGxVOFS` ではなく、`REG_BGxX` と `REG_BGxY` です。次に、これらはピクセルのオフセットではなく、24.8 の固定小数点数です。（実際には 20.8 の固定小数点数ですが、今それは重要ではありません。）

私はたいてい、`REG_BGxPA` などではなく、BG_AFFINE 構造体を通してアフィンパラメータを使います。*tonc_memmap.h* のメモリマップには、この目的のための `REG_BG_AFFINE` があります。この方法でレジスタを設定すると便利なことがあります。たいていはすでに BG_AFFINE 構造体を用意してあるので、それを 1 回の代入でレジスタにコピーできるからです。この例を下に示します。

アフィン変換の行列 **P** の要素は、アフィンスプライトの場合とまったく同じように働きます。画面空間からテクスチャ空間への変換を表す 8.8 の固定小数点数です。ただし、アフィン背景では要素が連続して（2 バイト間隔で）格納されるのに対し、スプライトの要素は 8 バイト間隔で並んでいます。*tonc_bg_affine.c* の `bg_aff_foo` 関数を使えば、望む変換にこれらを設定できます。

<div id="cd-bga-types">

```c
typedef struct tagBG_AFFINE
{
    s16 pa, pb;
    s16 pc, pd;
    s32 dx, dy
} ALIGN4 BG_AFFINE;

//! BG affine params array
#define REG_BG_AFFINE   ((BG_AFFINE*)(REG_BASE+0x0000))
```
</div>

```c
// Default BG_AFFINE data (tonc_core.c)
const BG_AFFINE bg_aff_default= { 256, 0, 0, 256, 0, 0 };

// Initialize affine registers for bg 2
REG_BG_AFFINE[2] = bg_aff_default;
```

:::warning 通常のタイルマップとアフィンのタイルマップのスクロール

  アフィンのタイルマップは**別の**スクロールレジスタを使います！ REG_BG*x*HOFS と REG_BG*x*VOFS ではなく、REG_BG*x*X と REG_BG*x*Y を使います。また、これらはハーフワードではなく、32 ビットの固定小数点数です。

:::

## アフィン背景の位置合わせと変換 {#sec-aff-ofs}

変位と変換のレジスタがわかったので、次はそれらが何をするのかを見てみましょう。これは思ったよりずっとややこしい話なので、よく注意してください。警告: また数学っぽくなります。

変位ベクトル **dx** は、通常の背景と同じように働きます。**dx** には、画面の原点に写される背景の座標が入ります。（その逆では*ありません*！）ただし、今回の **dx** は固定小数点表記です。同様に、アフィン変換の行列 **P** はアフィンスプライトと同じように働きます。**P** は画面空間からテクスチャ空間への変換を表します。数学的に書くと、次のように定義すれば

<table id="eq:defs">
  <tr>
    <td class="eqnrcell">({!@eq:defs}a)</td>
    <td class="eqcell">
      <i>T</i>(<b>dx</b>)<b>p</b> <b>:=</b> <b>p</b> + <b>dx</b><br>
      <i>T</i><sup>&minus;1</sup>(<b>dx</b>) = <i>T</i>(&minus;<b>dx</b>)
    </td>
  </tr>
  <tr>
    <td class="eqnrcell">({!@eq:defs}b)</td>
    <td class="eqcell"><b>P = A</b><sup>&minus;1</sup></td>
  </tr>
</table>

次のようになります。

<table id="eq:aff-defs">
  <tr>
    <td class="eqnrcell">({!@eq:aff-defs}a)</td>
    <td class="eqcell"><i>T</i>(<b>dx</b>)<b>q</b> = <b>p</b></td>
  </tr>
  <tr>
    <td class="eqnrcell">({!@eq:aff-defs}b)</td>
    <td class="eqcell"><b>P &middot; q</b> = <b>p</b></td>
  </tr>
</table>

ここで、

<table cellpadding=1 cellspacing=0>
  <tr>
    <th>p</th>
    <td>はテクスチャ空間の点、</td>
  </tr>
  <tr>
    <th>q</th>
    <td>は画面空間の点、</td>
  </tr>
  <tr>
    <th>dx</th>
    <td>は変位ベクトル（<code>REG_BGxX</code> と <code>REG_BGxY</code>）、</td>
  </tr>
  <tr>
    <th>A</th>
    <td>はテクスチャ空間から画面空間への変換、</td>
  </tr>
  <tr>
    <th>P</th>
    <td>は画面空間からテクスチャ空間への変換（<code>REG_BGxPA</code>-<code>REG_BGxPD</code>）です。</td>
  </tr>
</table>
<br>

{@eq:aff-defs} の問題は、変位と変換のどちらか一方だけを使った場合に何が起きるかしか表していないことです。では、両方を使いたい場合はどうなるのでしょうか？ これは重要な問題です。（[アフィンスプライトのデモ](affobj.html#sec-demo)で見たように）変換の順番は結果を左右しますし、それは変換と変位の順番についても同じだからです。実際には、平行移動が先に行われます。

<table id="eq:ofs">
  <tr>
    <td class="eqnrcell">({!@eq:ofs})</td>
    <td class="eqcell">
      <table class="eqtbl" cellpadding=2 cellspacing=0>
        <col align="right">
        <col align="center">
        <col align="left">
        <tr>
          <td><b>q</b></td>
          <td>=</td>
          <td><b>A</b> &middot; T(&minus;<b>dx</b>) <b>p</b></td>
        </tr>
        <tr>
          <td>T(<b>dx</b>) <b>P</b> &middot; <b>q</b></td>
          <td>=</td>
          <td><b>p</b></td>
        </tr>
        <tr>
          <td><b>dx</b> + <b>P</b> &middot; <b>q</b></td>
          <td>=</td>
          <td><b>p</b></td>
        </tr>
      </table>
    </td>
  </tr>
</table>

別の言い方をすると、変換は常に画面の左上を原点として使い、変位はそこにどの背景のピクセルが置かれるかを決めます。もちろん、たとえば画面上の別の点を中心に回転させたい場合、この仕組みはあまり役に立ちません。そうするには、ちょっとした技を使う必要があります。それらをまとめて扱うために、{@eq:ofs} と一般的な座標変換の式を組み合わせます。

<table id="eq:aff-ofs">
  <tr>
    <td class="eqnrcell">({!@eq:aff-ofs})</td>
    <td class="eqcell">
      <table class="eqtbl" cellpadding=2 cellspacing=0>
        <col align="right">
        <col align="center">
        <col align="left">
        <tr>
          <td><b>dx + P &middot; q</b></td>
          <td>=</td>
          <td><b>p</b></td>
        </tr>
        <tr>
          <td class="bdrB"><b>P</b> &middot; (<b>q &minus; q</b><sub>0</sub>)</td>
          <td class="bdrB">=</td>
          <td class="bdrB"><b>p &minus; p</b><sub>0</sub></td>
          <td class="bdrB">&minus;</td>
        </tr>
        <tr>
          <td><b>dx + P &middot; q</b><sub>0</sub></td>
          <td>=</td>
          <td><b>p</b><sub>0</sub></td>
        </tr>
        <tr>
          <td><b>dx</b></td>
          <td>=</td>
          <td><b>p</b><sub>0</sub> &minus; <b>P &middot; q</b><sub>0</sub></td>
        </tr>
      </table>
    </td>
  </tr>
</table>

で、*これは*いったい何を意味するのでしょうか？ この **dx** を変位ベクトルに使えば、テクスチャの点 **p**<sub>0</sub> を中心に変換が行われ、その点は画面の点 **q**<sub>0</sub> に来る、ということです。**P**·**q**<sub>0</sub> の項は、回転の中心を (0,0) ではなく **q**<sub>0</sub> にするために、テクスチャ空間で行う必要のある補正です。で、*これは*いったい何を意味するのでしょうか？ これを使おうとする前に、自分が実際にどんな効果を出そうとしているのかをよく考え、扱う座標系が 1 つではなく *2 つ*あることを意識するべきだ、ということです。そうすれば、{@eq:aff-ofs} の意味が見えてきます。いずれにせよ、私がこのために使っている関数は `bg_rotscale_ex()` で、基本的には次のようになっています。

```c
typedef struct tagAFF_SRC_EX
{
    s32 tex_x, tex_y;   // vector p0: origin in texture space (24.8f)
    s16 scr_x, scr_y;   // vector q0: origin in screen space (16.0f)
    s16 sx, sy;         // scales (8.8f)
    u16 alpha;          // CCW angle ( integer in [0,0xFFFF] )
} ALIGN4 AFF_SRC_EX;

void bg_rotscale_ex(BG_AFFINE *bgaff, const AFF_SRC_EX *asx)
{
    int sx= asx->sx, sy= asx->sy;
    int sina= lu_sin(asx->alpha), cosa= lu_cos(asx->alpha);

    FIXED pa, pb, pc, pd;
    pa=  sx*cosa>>12;   pb=-sx*sina>>12;    // .8f
    pc=  sy*sina>>12;   pd= sy*cosa>>12;    // .8f

    bgaff->pa= pa;  bgaff->pb= pb;
    bgaff->pc= pc;  bgaff->pd= pd;

    bgaff->dx= asx->tex_x - (pa*asx->scr_x + pb*asx->scr_y);
    bgaff->dy= asx->tex_y - (pc*asx->scr_x + pd*asx->scr_y);
}
```

これは、[中心以外を基準にしたオブジェクトの変換](affobj.html#sec-combo)の節で扱った `obj_rotscale_ex()` 関数とよく似ています。数学は同じですが、項を少し並べ替えています。背景版のほうが実は単純です。アフィンのオフセットの補正を画面空間ではなくテクスチャ空間で行えるので、**P** の逆行列をいじる必要がないからです。スプライトの大きさの補正も要りません。IPU に感謝です。念のため書いておくと、そう、この関数は `REG_BG_AFFINE` に直接使えます。

### 内部の基準点レジスタ {#ssec-bga-refpts}

変位と変換のレジスタについて、もう 1 つ重要なことが残っています。[GBATEK](https://problemkaputt.de/gbatek.htm#lcdiobgrotationscaling) から直接引用します（角かっこの部分を除く）。

> 上記の基準点 \[変位レジスタ\] は、VBlank のたびに自動的に内部レジスタへコピーされ、最初のスキャンラインの原点を指定します。その後、内部レジスタは各スキャンラインの後に dmx \[`REG_BGxPB`\] と dmy \[`REG_BGxPD`\] だけ加算されます。
> 注意: VBlank 期間外にソフトウェアから基準点レジスタに書き込むと、新しい値はただちに対応する内部レジスタにコピーされます。つまり、現在のフレームでは、新しい値は（一番上のスキャンラインではなく）*現在の*スキャンラインの原点を指定します。

普段はこれが問題になることはありませんが、HBlank 中に `REG_BGxY` に書き込もうとすると、期待どおりに動かないかもしれません。私は Mode 7 のものを動かそうとしたときに、これを痛い目にあって学びました。ただし、これが影響するのはアフィン背景だけです。通常の背景は別のレジスタを使います。

## マッピングの形式 {#sec-map}

アフィン背景のマップの配置とスクリーンエントリは、どちらも通常の背景のものとは大きく違います。皮肉なことに、ずっと単純でもあります。通常の背景はマップ全体を区画に分けます（それぞれがスクリーンブロック 1 つを丸ごと使います）が、アフィン背景は平らなマップを使います。つまり、スクリーンエントリの番号 *n* を求める普通の式がそのまま使えるので、ずっと楽になります。

<table id="eq:aff-sid">
  <tr>
    <td class="eqnrcell">({!@eq:aff-sid})</td>
    <td class="eqcell"><i>n</i> = <i>tx</i> + <i>ty</i>·<i>tw</i></td>
  </tr>
</table>

スクリーンエントリそのものも、通常の背景のものとは違います。アフィンのマップでは、スクリーンエントリは *1 バイトの長さ*で、使うタイルのインデックスしか入っていません。さらに、256 色のタイルしか使え*ません*。これで、ベースのキャラブロックにあるすべてのタイルにアクセスできますが、その後ろのキャラブロックのタイルにはアクセスできません。

本当に、それでだいたい全部です。いや、待ってください。もう 1 つ問題があります。*VRAM には一度に 16 ビットか 32 ビット単位でしかアクセスできない*ので、マップを埋めたり変更したりするときには注意が必要です。マップをバイトの配列に格納しているなら、まず `u16` か `u32` にキャストする必要があります。あるいは [DMA](dma.html) を使いましょう。よし、これで終わりです。

:::warning 通常のタイルマップとアフィンのタイルマップのマッピングの違い

  通常のマップとアフィンのマップの形式には、重要な違いが 2 つあります。1 つ目は、アフィンのスクリーンエントリが 1 バイトのタイルのインデックスにすぎないこと。2 つ目は、大きな通常のマップのように 32x32t のマップに分割するのではなく、マップが線形の配置を使うことです。

:::

## *sbb_aff* デモ {#sec-demo}

<div class="cpt_fr" style="width:240px">
  <img id="fig:sbb-aff" src="./img/demo/sbb_aff.png" alt="sbb_aff demo">
  <b>{*@fig:sbb-aff}</b>: <i>sbb_aff</i> デモ。
</div>

*sbb_aff* は、通常の背景に対する *sbb_reg* のアフィン背景版で、おまけもいくつか付いています。このデモは、{@fig:sbb-aff} に示す 64x64 タイルのアフィン背景を使います。これは 256 バイトずつ 16 の部分に分かれていて、それぞれが 1 色のタイルと、その部分の番号で埋められています。さて、アフィン背景のマップの配置が通常の背景と同じなら、各部分は 16x16t の正方形になるはずです。平らなメモリ配置なら、各部分は 64x16t の帯になるはずです。{@fig:sbb-aff} でわかるとおり、後者です。また、通常の背景とは違って、このマップは端で自動的に回り込まないこともわかります。

このデモで一番面白いのは、小さな黒と白の照準マークです。白い照準マークは回転の中心（アンカー）を示しています。前に言ったように、マップ上の点 **p**<sub>0</sub> を選んで、それを「その」回転の中心だと言うことは単純にはできません。まあ、できなくはないのですが、望む効果にはなりません。マップ上の点を使うだけだと、その点の周りでマップは回転しますが、画面上では常に左上の角にあります。マップのアンカーを画面上の特定の位置に動かすには、そこにもアンカーが必要です。これが **q**<sub>0</sub> です。両方を {@eq:aff-ofs} に入れれば、必要な変位ベクトルが求まります。**dx** = **p**<sub>0</sub>−**P·q**<sub>0</sub> です。この **dx** は、**p**<sub>0</sub> とも **q**<sub>0</sub> ともかなり違ったものになります。その軌跡は黒い照準マークで示されています。

このデモでは、**p**<sub>0</sub> と **q**<sub>0</sub> の両方を操作できます。もちろん、回転と拡大縮小もできます。操作の完全な一覧は次のとおりです。

<div class="lblock">
  <table cellpadding=1 cellspacing=0>
    <tr>
      <th>D-pad</th>
      <td>マップの回転の中心 <b>p</b><sub>0</sub> を動かす</td>
    </tr>
    <tr>
      <th>D-pad + A</th>
      <td>画面の回転の中心 <b>q</b><sub>0</sub> を動かす</td>
    </tr>
    <tr>
      <th>L,R</th>
      <td>背景を回転する。</td>
    </tr>
    <tr>
      <th>B(+Se)</th>
      <td>拡大と縮小。</td>
    </tr>
    <tr>
      <th>St</th>
      <td>回り込みのフラグを切り替える。</td>
    </tr>
    <tr>
      <th>St+Se</th>
      <td>アンカーと <b>P</b> をリセットする</td>
    </tr>
  </table>
</div>

<div id="cd=sbb-aff">

```c
#include <stdio.h>
#include <tonc.h>
#include "nums.h"

#define MAP_AFF_SIZE 0x0100

// --------------------------------------------------------------------
// GLOBALS
// --------------------------------------------------------------------

OBJ_ATTR *obj_cross= &oam_mem[0];
OBJ_ATTR *obj_disp= &oam_mem[1];

BG_AFFINE bgaff;

// --------------------------------------------------------------------
// FUNCTIONS
// --------------------------------------------------------------------

void win_textbox(int bgnr, int left, int top, int right, int bottom, int bldy)
{
    REG_WIN0H= left<<8 | right;
    REG_WIN0V=  top<<8 | bottom;
    REG_WIN0CNT= WIN_ALL | WIN_BLD;
    REG_WINOUTCNT= WIN_ALL;

    REG_BLDCNT= (BLD_ALL&~BIT(bgnr)) | BLD_BLACK;
    REG_BLDY= bldy;

    REG_DISPCNT |= DCNT_WIN0;

    tte_set_margins(left, top, right, bottom);
}

void init_cross()
{
    TILE cross=
    {{
        0x00011100, 0x00100010, 0x01022201, 0x01021201,
        0x01022201, 0x00100010, 0x00011100, 0x00000000,
    }};
    tile_mem[4][1]= cross;

    pal_obj_mem[0x01]= pal_obj_mem[0x12]= CLR_WHITE;
    pal_obj_mem[0x02]= pal_obj_mem[0x11]= CLR_BLACK;

    obj_cross->attr2= 0x0001;
    obj_disp->attr2= 0x1001;
}

void init_map()
{
    int ii;

    memcpy32(&tile8_mem[0][1], nums8Tiles, nums8TilesLen/4);
    memcpy32(pal_bg_mem, numsPal, numsPalLen/4);

    REG_BG2CNT= BG_CBB(0) | BG_SBB(8) | BG_AFF_64x64;
    bgaff= bg_aff_default;

    // fill per 256 screen entries (=32x4 bands)
    u32 *pse= (u32*)se_mem[8];
    u32 ses= 0x01010101;
    for(ii=0; ii<16; ii++)
    {
        memset32(pse, ses, MAP_AFF_SIZE/4);
        pse += MAP_AFF_SIZE/4;
        ses += 0x01010101;
    }
}

void sbb_aff()
{
    AFF_SRC_EX asx=
    {
        32<<8, 64<<8,           // Map coords.
        120, 80,                // Screen coords.
        0x0100, 0x0100, 0       // Scales and angle.
    };

    const int DX=256;
    FIXED ss= 0x0100;

    while(1)
    {
        vid_vsync();
        key_poll();

        // dir + A : move map in screen coords
        if(key_is_down(KEY_A))
        {
            asx.scr_x += key_tri_horz();
            asx.scr_y += key_tri_vert();
        }
        else    // dir : move map in map coords
        {
            asx.tex_x -= DX*key_tri_horz();
            asx.tex_y -= DX*key_tri_vert();
        }
        // rotate
        asx.alpha -= 128*key_tri_shoulder();


        // B: scale up ; B+Se : scale down
        if(key_is_down(KEY_B))
            ss += (key_is_down(KEY_SELECT) ? -1 : 1);

        // St+Se : reset
        // St : toggle wrapping flag.
        if(key_hit(KEY_START))
        {
            if(key_is_down(KEY_SELECT))
            {
                asx.tex_x= asx.tex_y= 0;
                asx.scr_x= asx.scr_y= 0;
                asx.alpha= 0;
                ss= 1<<8;
            }
            else
                REG_BG2CNT ^= BG_WRAP;
        }

        asx.sx= asx.sy= (1<<16)/ss;

        bg_rotscale_ex(&bgaff, &asx);
        REG_BG_AFFINE[2]= bgaff;

        // the cross indicates the rotation point
        // (== p in map-space; q in screen-space)
        obj_set_pos(obj_cross, asx.scr_x-3, (asx.scr_y-3));
        obj_set_pos(obj_disp, (bgaff.dx>>8)-3, (bgaff.dy>>8)-3);

        tte_printf("#{es;P}p0\t: (%d, %d)\nq0\t: (%d, %d)\ndx\t: (%d, %d)",
            asx.tex_x>>8, asx.tex_y>>8, asx.scr_x, asx.scr_y,
            bgaff.dx>>8, bgaff.dy>>8);
    }
}

int main()
{
    init_map();
    init_cross();

    REG_DISPCNT= DCNT_MODE1 | DCNT_BG0 | DCNT_BG2 | DCNT_OBJ;

    tte_init_chr4_b4_default(0, BG_CBB(2)|BG_SBB(28));
    tte_init_con();
    win_textbox(0, 8, 120, 232, 156, 8);

    sbb_aff();

    return 0;
}
```
</div>
