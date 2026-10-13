# 13. グラフィック効果

<!-- toc -->

さて、スプライトや背景を画面に出す方法はもうわかりましたね？ では、場を盛り上げるおまけの効果はどうでしょうか？ スプライトと背景を説明したとき、いくつかのフラグには触れずにおきました。[モザイク](#sec-mos)と[半透明合成（ブレンド）](#sec-blend)のフラグです。ここではそれらを扱います。また、背景やスプライトを隠すための領域を作れる[ウィンドウ](#sec-win)についても見ていきます。

## モザイク {#sec-mos}

モザイクを一番うまく説明するなら、スプライトやタイルをブロック状に見せるもの、です。モザイクは *w<sub>m</sub>* と *h<sub>m</sub>* というパラメータで 2 次元に働きます。これらの数値で、スプライトや背景を *w<sub>m</sub>* × *h<sub>m</sub>* ピクセルのブロックに分けます。各ブロックの左上のピクセルがそのブロックの残りを埋めるのに使われ、それでブロック状になるのです。{*@fig:metr-mos} は、メトロイドのスプライトに 1x4 のモザイクをかけたものです。青い線は縦方向のブロックの境界を示しています。言ったとおり、各ブロックの最初のラインがブロックの残りにコピーされています。モザイク効果のほかの例としては、ゼルダの伝説 神々のトライフォースで電気を帯びた敵を攻撃したときや、メトロイドフュージョンで X が姿を変えるときなどがあります。

<div class="lblock">
  <div class="cpt" style="width:400px;">
    <img id="fig:metr-mos" src="./img/metr/mos.png" alt="a 1x4 mosaiced metroid">
    <b>{*@fig:metr-mos}</b>: 1×4 のモザイクをかけたメトロイド。
  </div>
</div>

### モザイクの使い方: スプライトと背景のフラグ、REG_MOSAIC {#ssec-mos-use}

モザイクを使うには、2 つのことをする必要があります。まず、モザイクを有効にします。個々のスプライトなら `OBJ_ATTR.attr0`\{C\} を設定します。背景なら `REG_BGxCNT`\{7\} を設定します。次に、`REG_MOSAIC` でモザイクの度合いを設定します。このレジスタは次のようになっています。

<div class="reg">
  <table class="table-reg" id="tbl:reg-mosaic">
    <caption class="reg">REG_MOSAIC @ 0400:004Ch</caption>
    <tr class="bits">
      <td>F E D C</td>
      <td>B A 9 8</td>
      <td>7 6 5 4</td>
      <td>3 2 1 0</td>
    </tr>
    <tr class="bf">
      <td class="rclr3">Ov</td>
      <td class="rclr2">Oh</td>
      <td class="rclr1">Bv</td>
      <td class="rclr0">Bh</td>
    </tr>
  </table>
  <br>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width="8%">
    <col class="def" width="12%">
    <tr align="left">
      <th>ビット</th>
      <th>名前</th>
      <th>定義</th>
      <th>説明</th>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-3</td>
        <td class="rclr0">Bh</td>
        <td><i>MOS_BH#</i></td>
        <td>背景の横方向の引き伸ばし。</td>
      </tr>
      <tr class="bg1">
        <td>4-7</td>
        <td class="rclr1">Bv</td>
        <td><i>MOS_BV#</i></td>
        <td>背景の縦方向の引き伸ばし。</td>
      </tr>
      <tr class="bg0">
        <td>8-B</td>
        <td class="rclr2">Oh</td>
        <td><i>MOS_OH#</i></td>
        <td>オブジェクトの横方向の引き伸ばし。</td>
      </tr>
      <tr class="bg1">
        <td>C-F</td>
        <td class="rclr3">Ov</td>
        <td><i>MOS_OV#</i></td>
        <td>オブジェクトの縦方向の引き伸ばし。</td>
      </tr>
    </tbody>
  </table>
</div>

<dfn>引き伸ばし</dfn>（stretch）とは、基準のピクセルを何ピクセルぶん引き伸ばすかです。これは *w<sub>m</sub>*−1 や *h<sub>m</sub>*−1 に相当します。各効果に 4 ビットずつあるので、引き伸ばしは 0 から 15 まで、つまりモザイクの幅と高さは 1 から 16 までになります。

:::tip モザイクを有効にするには

背景なら REG_BGxCNT のビット 7 を設定します。スプライトなら属性 0 のビット 12 を設定します。それから REG_MOSAIC でモザイクの度合いを設定します。

:::

### 小さなモザイクのデモ {#ssec-mos-demo}

オブジェクトと背景の両方でのモザイクの使い方を示す、*mos_demo* というデモがあります。

<div id="cd-mos-demo">

```c
// mos_demo.c
//   bg 0, cbb  0, sbb 31, pb 0: text
//   bg 1, cbb  1, sbb 30, pb 1: bg metroid
//   oam 0: tile 0-63: obj metroid

#include <stdio.h>
#include <tonc.h>
#include "metr.h"

void test_mosaic()
{
    tte_printf("#{P:48,8}obj#{P:168,8}bg");
    tte_set_margins(4, 130, 128, 156);

    POINT pt_obj={0,0}, pt_bg={0,0};
    POINT *ppt= &pt_obj;
    while(1)
    {
        vid_vsync();

        // control the mosaic
        key_poll();

        // switch between bg or obj mosaic
        ppt= key_is_down(KEY_A) ? &pt_bg : &pt_obj;
        ppt->x += key_tri_horz();       // inc/dec h-mosaic
        ppt->y -= key_tri_vert();       // inc/dec v-mosaic

        ppt->x= clamp(ppt->x, 0, 0x80);
        ppt->y= clamp(ppt->y, 0, 0x80);

        REG_MOSAIC= MOS_BUILD(pt_bg.x>>3, pt_bg.y>>3, pt_obj.x>>3, pt_obj.y>>3);

        tte_printf("#{es;P}obj h,v: %2d,%2d\n bg h,v: %2d,%2d",
            pt_obj.x>>3, pt_obj.y>>3, pt_bg.x>>3, pt_bg.y>>3);
    }
}

void load_metr()
{
    int ix, iy;

    memcpy32(&tile_mem[1][0], metrTiles, metrTilesLen/4);
    memcpy32(&tile_mem[4][0], metrTiles, metrTilesLen/4);
    memcpy32(pal_obj_mem, metrPal, metrPalLen/4);

    // create object: oe0
    OBJ_ATTR *metr= &oam_mem[0];
    obj_set_attr(metr, ATTR0_SQUARE | ATTR0_MOSAIC, ATTR1_SIZE_64, 0);
    obj_set_pos(metr, 32, 24);              // left-center

    // create bg map: bg1, cbb1, sbb 31

    for(ix=1; ix<16; ix++)
        pal_bg_mem[ix+16]= pal_obj_mem[ix] ^ CLR_WHITE;

    SCR_ENTRY *pse= &se_mem[30][3*32+18];    // right-center
    for(iy=0; iy<8; iy++)
        for(ix=0; ix<8; ix++)
            pse[iy*32+ix]= (iy*8+ix) | SE_PALBANK(1);

    REG_BG1CNT= BG_CBB(1) | BG_SBB(30) | BG_MOSAIC;
}

int main()
{
    // setup sprite
    oam_init(oam_mem, 128);
    load_metr();
    REG_DISPCNT= DCNT_BG0 | DCNT_BG1 | DCNT_OBJ | DCNT_OBJ_1D;

    // set-up text: bg0, cbb0, sbb31
    tte_init_chr4_b4_default(0, BG_CBB(2)|BG_SBB(31));
    tte_init_con();

    test_mosaic();
    return 0;
}
```
</div>

<div class="cpt_fr" style="width:240px;">
  <img id="fig:mos-demo" src="./img/demo/mos_demo.png">
  <b>{*@fig:mos-demo}</b>: <i>mos_demo</i>。
</div>

このデモでは 2 匹のメトロイドを使います。左がスプライトのメトロイドで、右が色を反転させた背景のメトロイドです。スプライトと背景の準備の仕方は前に見せたので、ここでの手順は何も新しいことではなく、追えるはずです。まあ、`OBJ_ATTR.attr0` と REG_BG0CNT にモザイクのフラグを設定するところは別ですが、そこはここで太字にしてあります。

モザイク効果は `test_mosaic()` の中で調整しています。モザイクの現在の度合いを記録するのに、2 つの 2D の点を使っています。十字キーでモザイクの度合いを増減します。十字キーだけならオブジェクトのモザイクを、A を押しながらなら背景のモザイクを設定します。

コードの設計について言えば、ここではオブジェクト用と背景用の 2 つの if ブロックを使うこともできましたが、ポインタでモザイクの対象を切り替えることもでき、そのほうがコードを少し節約できます。ポインタ万歳。また、座標は .3 の固定小数点形式になっていて、これでモザイクの度合いの変化を遅くしています。ここでも、タイマー用の変数を使って、しきい値に達したかどうかをさらに確認することもできましたが、固定小数点のタイマーのほうがずっと簡単で、私の考えではすっきりしています。

ところで、このデモはぜひ実機で見てください。どういうわけか、VBA も no$gba も、モザイクについてはどちらも欠陥があります。VBA 1.7.2 以降では、スプライトの横方向のモザイクに問題があります。実機とスクロールするモザイクの背景の間で食い違いを見たことがあると思うのですが、どこで見たのか思い出せません。no$gba については、縦方向のモザイクがスプライトでも背景でも無効になっているようです。

:::warning エミュレータとモザイク

最も人気のある GBA エミュレータである VBA と no$gba は、どちらもモザイクに問題を抱えています。気をつけてください。

:::

## 半透明合成（ブレンド） {#sec-blend}

ゲームやグラフィックがまったくの初めてでなければ、<dfn>アルファブレンド</dfn>という言葉を聞いたことがあるかもしれません。重なった 2 つのレイヤーの色の値を組み合わせて、透明さを作り出すものです（半透明とも呼ばれます。*完全に*透明なものは見えないからです）。ビットマップの種類によっては、アルファチャンネルを持つものもあります。これは、そのピクセルの透明度か不透明度のどちらかを表します。

ブレンドの基本的な考え方は次のとおりです。互いに重なる 2 つのレイヤー A と B があるとします。A が B の上にあると考えてください。この領域のピクセルの色の値は、次のように定義されます。

<table id="eq:blend">
  <tr>
    <td class="eqnrcell">({!@eq:blend})</td>
    <td class="eqcell">C = w<sub>A</sub>·A + w<sub>B</sub>·B,</td>
  </tr>
</table>

ここで *w<sub>A</sub>* と *w<sub>B</sub>* は、レイヤーの<dfn>重み</dfn>です。重みは一般に（0 から 1 の間に）正規化されていて、0 が完全に透明、1 が完全に見える状態です。色の成分もこのように考えると便利です。これを使ってできることをいくつか挙げます。

<div class="lblock">
  <table>
    <col span=3 align="left">
    <tr>
      <th width="8%">w<sub>A</sub></th>
      <th width="10%">w<sub>B</sub></th>
      <th>効果</th>
    </tr>
    <tr>
      <td>1</td>
      <td>0</td>
      <td>レイヤー A が完全に見える（B を隠す。標準）</td>
    </tr>
    <tr>
      <td>0</td>
      <td>1</td>
      <td>レイヤー B が完全に見える（または A が見えない）</td>
    </tr>
    <tr>
      <td>&alpha;</td>
      <td>1&minus;&alpha;</td>
      <td>アルファブレンド。この場合 &alpha; は不透明度。</td>
    </tr>
  </table>
</div>

これらの例では重みの合計が 1 なので、最終的な色 *C* も 0（黒）と 1（白）の間に収まることに注意してください。あとで見るように、この範囲から外れる場合もあります。そうなると、値は標準の範囲に切り詰められます。

### GBA のブレンド {#ssec-bld-gba}

背景は常にブレンドが有効になっています。スプライトのブレンドを有効にするには、`OBJ_ATTR.attr0`\{a\} を設定します。ブレンドを制御するレジスタは 3 つありますが、残念ながらいろいろな名前で呼ばれています。私が使っているのは `REG_BLDCNT`、`REG_BLDALPHA`、`REG_BLDY` です。ほかに `REG_BLDMOD`、`REG_COLEV`、`REG_COLEY` という名前もあり、最後の 2 つは「E」が取れていることもあります。気をつけてください。ともかく、1 つ目はブレンドをどのように、どのレイヤーに行うかを指定し、残りの 2 つは重みを持っています。そうそう、GBA は浮動小数点を扱えないので、重みは 1.4 形式の[固定小数点](fixed.html)数です。もちろん 0 と 1 の範囲に制限されているので、ブレンドの段階は 17 段階です。

<div class="reg">
  <table class="table-reg reg-huge" id="tbl:reg-bldcnt">
    <caption class="reg">REG_BLDCNT (REG_BLDMOD) @ 0400:0050h</caption>
    <tr class="bits">
      <td>F E</td>
      <td>D</td>
      <td>C</td>
      <td>B</td>
      <td>A</td>
      <td>9</td>
      <td>8</td>
      <td>7 6</td>
      <td>5</td>
      <td>4</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr class="bf">
      <td>-</td>
      <td class="rclr1">bBD</td>
      <td class="rclr1">bOBJ</td>
      <td class="rclr1">bBG3</td>
      <td class="rclr1">bBG2</td>
      <td class="rclr1">bBG1</td>
      <td class="rclr1">bBG0</td>
      <td class="rclr2">BM</td>
      <td class="rclr0">aBD</td>
      <td class="rclr0">aObj</td>
      <td class="rclr0">aBG3</td>
      <td class="rclr0">aBG2</td>
      <td class="rclr0">aBG1</td>
      <td class="rclr0">aBG0</td>
    </tr>
  </table>
  <br>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width="8%">
    <col class="def" width="12%">
    <tr align="left">
      <th>ビット</th>
      <th>名前</th>
      <th>定義</th>
      <th>説明</th>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-5</td>
        <td class="rclr0">aBG0-aBD</td>
        <td><i>BLD_TOP#</i></td>
        <td>A（上）のレイヤー。ちなみに <b>BD</b> は<dfn>バックドロップ</dfn>（back drop）で、色 0 で塗りつぶされた面です。ビットを設定すると、そのレイヤーが A の重みを使います。これらのレイヤーは実際に B のレイヤーより手前になければならず、そうでなければブレンドは失敗することに注意してください。</td>
      </tr>
      <tr class="bg1">
        <td>6-7
        <td class="rclr2">BM
        <td>BLD_OFF, BLD_STD, BLD_WHITE, BLD_BLACK, <i>BLD_MODE#</i>
        <td>
          <b>ブレンドのモード</b>。
          <ul>
            <li><b>00</b>: ブレンドしない。</li>
            <li><b>01</b>: <code>REG_ALPHA</code> の重みを使う通常のブレンド。</li>
            <li><b>10</b>: <code>REG_BLDY</code> の重みを使って A を白とブレンドする（白へのフェード）</li>
            <li><b>11</b>: <code>REG_BLDY</code> の重みを使って A を黒とブレンドする（黒へのフェード）</li>
          </ul>
        </td>
      </tr>
      <tr class="bg0">
        <td>8-D</td>
        <td class="rclr1">bBG0-bBD</td>
        <td><i>BLD_BOT#</i></td>
        <td>B（下）のレイヤー。B の重みを使います。これらのレイヤーは実際に A のレイヤーより奥になければならず、そうでなければブレンドは働かないことに注意してください。</td>
      </tr>
    </tbody>
  </table>
</div>

`REG_BLDALPHA` レジスタと `REG_BLDY` レジスタは、ブレンドの重みを <span class="rclr0"><b>eva</b></span>、<span class="rclr1"><b>evb</b></span>、<span class="rclr2"><b>ey</b></span> の形で持っています。どれも 1.4 の固定小数点形式です。それから、いいえ、なぜそう呼ばれるのかは知りません。単にそういう名前なのです。

<div class="reg">
  <table class="table-reg" id="tbl:reg-bldalpha">
    <caption class="reg">REG_BLDALPHA (REG_COLEV) @ 0400:0052h</caption>
    <tr class="bits">
      <td>F E D</td>
      <td>C B A 9 8</td>
      <td>7 6 5</td>
      <td>4 3 2 1 0</td>
    </tr>
    <tr class="bf">
      <td>-</td>
      <td class="rclr1">evb</td>
      <td>-</td>
      <td class="rclr0">eva</td>
    </tr>
  </table>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width="8%">
    <col class="def" width="12%">
    <tr align="left">
      <th>ビット</th>
      <th>名前</th>
      <th>定義</th>
      <th>説明</th>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-4</td>
        <td class="rclr0">eva</td>
        <td><i>BLD_EVA#</i></td>
        <td><b>上のブレンドの重み</b>。通常のブレンドでのみ使われます</td>
      </tr>
      <tr class="bg1">
        <td>8-C</td>
        <td class="rclr1">evb</td>
        <td><i>BLD_EVB#</i></td>
        <td><b>下のブレンドの重み</b>。通常のブレンドでのみ使われます</td>
      </tr>
    </tbody>
  </table>
</div>
<br>
<div class="reg">
  <table class="table-reg" id="tbl:reg-bldy">
    <caption class="reg">REG_BLDY (REG_COLEY) @ 0400:0054h</caption>
    <tr class="bits">
      <td>F E D C B A 9 8 7 6 5</td>
      <td>4 3 2 1 0</td>
    </tr>
    <tr class="bf">
      <td>-</td>
      <td class="rclr2">ey</td>
    </tr>
  </table>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width="8%">
    <col class="def" width="12%">
    <tr align="left">
      <th>ビット</th>
      <th>名前</th>
      <th>定義</th>
      <th>説明</th>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-4</td>
        <td class="rclr2">ey</td>
        <td><i>BLDY#</i></td>
        <td><b>上のブレンドのフェード</b>。白と黒へのフェードに使われます。</td>
      </tr>
    </tbody>
  </table>
</div>

### ブレンドの注意点 {#ssec-bld-cav}

ブレンドはあると便利な機能ですが、次の点に気をつけてください。

- A のレイヤーは B のレイヤーより手前に*なければなりません*。そうなっているときにだけ、実際にブレンドが起きます。ですから、優先順位に気をつけてください。
- アルファブレンドのモード（モード 1）では、ブレンドはレイヤー A とレイヤー B の**重なっている、透明でない**ピクセルでのみ行われます。重なっていないピクセルは通常の色のままです。
- スプライトは背景とは違う影響を受けます。特に、`REG_BLDCNT`\{6,7\} で指定したブレンドのモードは、重なって*いない*部分にだけ適用されます（そのため、実質的にはフェードしか働きません）。重なっているピクセルには、現在のブレンドのモードにかかわらず、*常に*標準のブレンドがかかります。
- [ウィンドウ](#sec-win)を使っている場合は、ブレンドが働くように、REG_WININ や REG_WINOUT のビット 5 や 13 を設定する必要があります。

### お約束のデモ {#ssec-bld-demo}

<div id="cd-bld-demo">

```c
// bld_demo.c

//   bg 0, cbb  0, sbb 31, pb 15: text
//   bg 1, cbb  2, sbb 30, pb 1: metroid
//   bg 2, cbb  2, sbb 29, pb 0: fence
//   oam 0: tile 0-63: obj metroid

#include <stdio.h>
#include <tonc.h>
#include "../gfx/metr.h"

void test_blend()
{
    tte_printf("#{P:48,8}obj#{P:168,8}bg");
    tte_set_margins(16, SCR_H-4-4*12, SCR_W-4, SCR_H-4);

    u32 mode=0;
    // eva, evb and ey are .4 fixeds
    // eva is full, evb and ey are empty
    u32 eva=0x80, evb= 0, ey=0;

    REG_BLDCNT= BLD_BUILD(
        BLD_OBJ | BLD_BG0,  // Top layers
        BLD_BG1,            // Bottom layers
        mode);              // Mode

    while(1)
    {
        vid_vsync();
        key_poll();

        // Interactive blend weights
        eva += key_tri_horz();
        evb -= key_tri_vert();
        ey  += key_tri_fire();

        mode += bit_tribool(key_hit(-1), KI_R, KI_L);

        // Clamp to allowable ranges
        eva = clamp(eva, 0, 0x81);
        evb = clamp(evb, 0, 0x81);
        ey  = clamp(ey, 0, 0x81);
        mode= clamp(mode, 0, 4);

        tte_printf("#{es;P}mode :\t%2d\neva :\t%2d\nevb :\t%2d\ney :\t%2d",
            mode, eva/8, evb/8, ey/8);

        // Update blend mode
        BFN_SET(REG_BLDCNT, mode, BLD_MODE);

        // Update blend weights
        REG_BLDALPHA= BLDA_BUILD(eva/8, evb/8);
        REG_BLDY= BLDY_BUILD(ey/8);
    }
}

void load_metr()
{
    // copy sprite and bg tiles, and the sprite palette
    memcpy32(&tile_mem[2][0], metrTiles, metrTilesLen/4);
    memcpy32(&tile_mem[4][0], metrTiles, metrTilesLen/4);
    memcpy32(pal_obj_mem, metrPal, metrPalLen/4);

    // set the metroid sprite
    OBJ_ATTR *metr= &oam_mem[0]; // use the first sprite
    obj_set_attr(metr, ATTR0_SQUARE | ATTR0_BLEND, ATTR1_SIZE_64, 0);
    obj_set_pos(metr, 32, 24);       // mid-center

    // create the metroid bg
    // using inverted palette for bg-metroid
    int ix, iy;
    for(ix=0; ix<16; ix++)
        pal_bg_mem[ix+16]= pal_obj_mem[ix] ^ CLR_WHITE;

    SCR_ENTRY *pse= &se_mem[30][3*32+18]; // right-center
    for(iy=0; iy<8; iy++)
        for(ix=0; ix<8; ix++)
            pse[iy*32+ix]= iy*8+ix + SE_PALBANK(1);

    REG_BG0CNT= BG_CBB(0) | BG_SBB(30);
}

// set-up the fence background
void load_fence()
{

    // tile 0 / ' ' will be a fence tile
    const TILE fence=
    {{
        0x00012000, 0x00012000, 0x00022200, 0x22220222,
        0x11122211, 0x00112000, 0x00012000, 0x00012000,
    }};
    tile_mem[2][64]= fence;
    se_fill(se_mem[29], 64);

    pal_bg_mem[0]= RGB15(16, 10, 20);
    pal_bg_mem[1]= RGB15( 0,  0, 31);
    pal_bg_mem[2]= RGB15(16, 16, 16);

    REG_BG2CNT= BG_CBB(2) | BG_SBB(29);
}

int main()
{
    oam_init(oam_mem, 128);
    load_metr();
    load_fence();

    tte_init_chr4_b4_default(0, BG_CBB(0)|BG_SBB(31));
    tte_init_con();

    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0 | DCNT_BG1 | DCNT_BG2 |
        DCNT_OBJ | DCNT_OBJ_1D;

    test_blend();

    return 0;
}
```
</div>

<div class="cpt_fr" style="width:240px;">
  <img id="fig:bld-demo" src="./img/demo/bld_demo.png" alt="bld_demo">
  <b>{*@fig:bld-demo}</b>: ブレンドのデモ。mode=2、eva=0、evb=0、ey=10。
</div>

いつものように、これらすべてに対応するデモがあります。*bld_demo* には、フェンスのような背景（正確には背景 1）の上に 2 匹のメトロイド（左はスプライト、右の（パレットを反転させた）ものは背景 0 にいます）が登場し、モードと 3 つの重みを個別に変更できます。ちなみに、モードは左上の隅に表示されます。操作は次のとおりです。

<div class="lblock">
  <table>
    <col valign="top">
    <tr>
      <th>left, right</th>
      <td><code>eva</code> を変更する。<code>eva</code> は最初は最大値になっていることに注意。</td>
    </tr>
    <tr>
      <th>down,up</th>
      <td><code>evb</code> を変更する。</td>
    </tr>
    <tr>
      <th>B,A</th>
      <td><code>ey</code> を変更する</td>
    </tr>
    <tr>
      <th>L,R</th>
      <td>モードを変更する。</td>
    </tr>
  </table>
</div>

注目すべき関数は `test_blend()` です。ここでボタンの処理をして、ブレンドの設定を変更しています。*mos_demo* と同じように、ブレンドの重みの変数には .3 の固定小数点数を使い、変化の速さをもっと扱いやすい程度に落としています。ブレンドのレジスタそのものを設定するには、`BUILD()` マクロと `BF_SET()` を使っています。この目的には十分うまく働きます。もちろん、ここでラッパー関数を書くのはごく簡単でしょう。コードのほとんどはかなり標準的なものです。ブレンドのモードと重みをいろいろ変えて、何が起きるか見てみてください。

前に言ったように、スプライトのメトロイドが背景のメトロイドとは違う影響を受けることに、ぜひ注目してください。背景どうしのブレンドは、モードのとおりにふるまいます。一方スプライトは、フェンスのピクセルと重なっていれば常にブレンドされ、残りの部分がモードに従います。注意点で説明したとおりです。

## ウィンドウ {#sec-win}

ウィンドウを使うと、画面を長方形の領域、つまり、その、ウィンドウに分けられます。基本的なウィンドウは 2 つあります。<dfn>win0</dfn> と <dfn>win1</dfn> です。3 つ目の種類のウィンドウとして、<dfn>オブジェクト</dfn>ウィンドウもあります。これは、スプライトの見えているピクセルからウィンドウを作ります。ウィンドウは、それぞれ `REG_DISPCNT`\{d,e,f\} を設定して有効にします。

長方形のウィンドウは、<dfn>左</dfn>、<dfn>右</dfn>、<dfn>上</dfn>、<dfn>下</dfn>の辺で定義されます。「長方形には内側と外側の 2 つの辺しかない」と言っておもしろがる*あの手の*人でなければ、ですが。実は、これは思ったより当たっています。win0 と win1 の和集合が<dfn>内側</dfn>のウィンドウです。そして<dfn>外側</dfn>のウィンドウもあり、それ以外のすべてがこれに当たります。言い換えると、次のようになります。

<div class="lblock">
  <table>
    <tr>
      <td class="fill"></td>
      <td class="eqcell">winIn = win0 | win1<br>winOut = ~(winIn)</td>
    </tr>
  </table>
  <table id="fig:win-dgrm">
    <tbody valign="top">
      <tr>
        <td>
          <div class="cpt" style="width:240px;">
            <img src="./img/demo/win_demo.png" alt="Showing win0, win1, and win_out windows">
            <b>{*@fig:win-dgrm}a</b>: win0、win1、win_out のウィンドウを表示したところ。
          </div>
        </td>
        <td>
          <div class="cpt" style="width:180px;">
            <img src="./img/demo/win_demo_x.png" alt="diagram.">
            <b>{*@fig:win-dgrm}b</b>: win0 は赤、win1 は緑、winIn は win0 | win1（青い縁）、winOut は灰色。
          </div>
        </td>
      </tr>
    </tbody>
  </table>
</div>

### ウィンドウの境界 {#ssec-win-bound}

win0 と win1 には、それぞれ境界を定める 2 つのレジスタがあります。順に `REG_WIN0H`（`0400:0040h`）、`REG_WIN1H`（`0400:0042h`）、`REG_WIN0V`（`0400:0044h`）、`REG_WIN1V`（`0400:0046h`）で、次のような配置になっています。

<div class="reg">
  <table class="table-reg" id="tbl:reg-winxy">
    <caption class="reg">REG_WINxH and REG_WINxV @ 0400:0040-0400:0047h</caption>
    <tr class="bits">
      <th>レジスタ</th>
      <td>F E D C B A 9 8</td>
      <td>7 6 5 4 3 2 1 0</td>
    </tr>
    <tr class="bf">
      <td><code>REG_WINxH</code></td>
      <td class="rclr0">left</td>
      <td class="rclr1">right</td>
    </tr>
    <tr class="bf">
      <td><code>REG_WINxV</code></td>
      <td class="rclr2">top</td>
      <td class="rclr3">bottom</td>
    </tr>
  </table>
  <br>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width=40>
    <col class="def">
    <tr align="left">
      <th>ビット</th>
      <th>名前</th>
      <th></th>
      <th>説明</th>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-7</td>
        <td class="rclr1">right</td>
        <td></td>
        <td>ウィンドウの右辺（含まない）</td>
      </tr>
      <tr class="bg1">
        <td>8-F</td>
        <td class="rclr0">left</td>
        <td></td>
        <td>ウィンドウの左辺（含む）</td>
      </tr>
      <tr>
        <td colspan=4>-</td>
      </tr>
      <tr class="bg0">
        <td>0-7</td>
        <td class="rclr3">bottom</td>
        <td></td>
        <td>ウィンドウの下辺（含まない）</td>
      </tr>
      <tr class="bg1">
        <td>8-F</td>
        <td class="rclr2">top</td>
        <td></td>
        <td>ウィンドウの上辺（含む）</td>
      </tr>
    </tbody>
  </table>
</div>

つまり、それぞれの値に 1 バイトずつあります。*符号なし*の char という意味でのバイトです。ウィンドウの中身は、左上から始まって、右下の手前まで（右下は含みません）描かれます。理解しておくべきなのは、たとえば右の値が左の値より小さい場合にもこれが成り立つということです。その場合は回り込みが起き、そのラインでは R と L の間のピクセルを除くすべてがウィンドウの内側になります。*R* < *L* と *B* < *T* の両方が成り立つなら、十字の形をしたウィンドウになります。

### ウィンドウの中身 {#ssec-win-content}

ウィンドウの中身にできるのは、背景 0-3 とオブジェクトです。意外なことは何もありませんね？ 全部で、win0、win1、winOut、winObj の領域があります。`REG_WININ`（`0400:0048h`）が win0 と win1 を制御し、`REG_WINOUT`（`0400:004ah`）が winOut と winObj を受け持ちます。中身の種類ごとに 1 ビットずつあり、さらにブレンド用のビットが 1 つあります。特定のウィンドウの中身にブレンドを使うつもりなら、このビットが必要です。

<div class="reg">
  <table class="table-reg reg-huge" id="tbl:reg-winio">
    <tr class="bits">
      <th>レジスタ</th>
      <td>F E</td>
      <td>D</td>
      <td>C</td>
      <td>B</td>
      <td>A</td>
      <td>9</td>
      <td>8</td>
      <td>7 6</td>
      <td>5</td>
      <td>4</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr class="bf">
      <th>ビット</th>
      <td>-</td>
      <td class="rclr1">Bld</td>
      <td class="rclr1">Obj</td>
      <td class="rclr1">BG3</td>
      <td class="rclr1">BG2</td>
      <td class="rclr1">BG1</td>
      <td class="rclr1">BG0</td>
      <td>-</td>
      <td class="rclr0">Bld</td>
      <td class="rclr0">Obj</td>
      <td class="rclr0">BG3</td>
      <td class="rclr0">BG2</td>
      <td class="rclr0">BG1</td>
      <td class="rclr0">BG0</td>
    </tr>
    <tr align="center">
      <th><code>REG_WININ</code></th>
      <td>-</td>
      <td colspan=6>win1</td>
      <td>-</td>
      <td colspan=6>win0</td>
    </tr>
    <tr align="center">
      <th><code>REG_WINOUT</code></th>
      <td>-</td>
      <td colspan=6>winObj</td>
      <td>-</td>
      <td colspan=6>winOut</td>
    </tr>
  </table>
  <br>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width="8%">
    <col class="def" width="12%">
    <tr align="left">
      <th>ビット</th>
      <th>名前</th>
      <th>定義</th>
      <th>説明</th>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-5</td>
        <td class="rclr0">BGx, Obj, Bld</td>
        <td>WIN_BGx, WIN_OBJ, WIN_BLD, <i>WIN_LAYER#</i></td>
        <td><b>ウィンドウのフラグ</b>。REG_WININ と REG_WINOUT のすべてのバイトで使います。</td>
      </tr>
    </tbody>
  </table>
</div>

ここでは、マクロやビットの定義はほとんどありません。あまり必要ないからです。とはいえ、*tonc_memdef.h* には次のものがあります。

```c
#define WIN_BUILD(low, high)    \
    ( ((high)<<8) | (low) )

#define WININ_BUILD(win0, win1)     WIN_BUILD(win0, win1)

#define WINOUT_BUILD(out, obj)      WIN_BUILD(out, obj)
```

ウィンドウについて知っておくべきことが、まだいくつかあります。まず、`REG_DISPCNT` でウィンドウを有効にすると、何も表示されなくなります。これには 2 つの理由があります。1 つ目は、境界のレジスタがすべて 0 なので、基本的に画面全体が winOut になることです。2 つ目は、これは本当に重要なのですが、背景やオブジェクトは、それが有効になっているウィンドウにしか表示されないということです！ つまり、`REG_WININ` か `REG_WINOUT` に*何らかの*ビットが設定されていない限り、何も表示されません。デモで見るように、これはものを隠すための効果的な方法になります。もう 1 つ覚えておかなければならないのは、win0 が win1 より優先され、win1 が winOut より優先されることです。winObj がこの中でどこに入るのかは、まだよくわかりません。

:::tip ウィンドウに必要なこと

ウィンドウを使うには、次のことをする必要があります。

- `REG_DISPCNT` でウィンドウを有効にする
- `REG_WININ` と `REG_WINOUT` の適切なビットを設定して、どのウィンドウに何を表示したいかを指定する。ウィンドウを有効にしているなら、ここで少なくとも何らかのビットを設定**しなければなりません**。そうしないと、何も表示されません！
- `REG_WINxH/V` で望むウィンドウの大きさを設定する。設定しないと、すべてが Out のウィンドウにあるとみなされます。

:::

### 注意点 {#ssec-win-caveat}

上か下のどちらかが画面の外にあると、本当に奇妙なことが起きます。実際には奇妙なことがいくつも起きるので、詳しくは*実機で！*デモを見てください。

-   上が \[-29, 0⟩ の範囲（つまり \[227, 255\]）にあると、ウィンドウはまったく描画*されません*。同様に、下がこの範囲にあると、ウィンドウは画面の高さ全体を覆います。原因が何なのかははっきり言えませんが、VCount も 227 で止まるので、それと何か関係があるのかもしれません。
-   また、下を 161 から 160 に動かすと、ウィンドウはやはり全体を覆いますが、それは 1 フレームほどの間だけです。
-   ここまでの点は *T*<*B* を前提にしています。上のほうが大きければ、効果は逆になります。

:::warning エミュレータでは起きないウィンドウの奇妙なふるまい

このふるまいは、私が試したエミュレータでは起き*ません*。

VBA は、常識から考えるとおりに、ウィンドウを切り詰めます。（もちろん、常識は、太陽が地球の周りを回っているとか、星は大きな黒いキャンバスに開いた針の穴だとかも教えてくれます。常識というのは、なかなか常識的ではないものです。）

MappyVM と BoycottAdvance は、境界のどれかが画面の外に出ると、単にウィンドウを消してしまいます。

:::

### デモ: ポケットの中にロケットが {#ssec-win-demo}

まだ気づいていなかったなら言っておくと、私はメトロイドシリーズが好きです。本当に大好きです。スーパーメトロイドを遊んだことがあれば、おそらく X 線スコープを使ったことがあるでしょう。これを使うと、レイヤーを透かして見ることができ、アイテムや隠し通路をずっと簡単に見つけられます。これがどうやって実現されていたと思いますか？ そう、ウィンドウです。ウィンドウのデモ *win_demo* も、基本的には同じことをします。背景のレイヤーの後ろにロケットのアイテムが隠されていて、X 線の長方形を画面上で動かして、それを見つけるのです。

操作は簡単です。十字キーでウィンドウを動かし、START でロケットの位置を変えます。細かく動かす操作（A + 十字キー）も加えたので、特定の位置でウィンドウが見せる奇妙なふるまいを確かめられます。

<div class="lblock">
  <table>
    <tr>
      <th width="10%">dir</th>
      <td>長方形を動かす。</td>
    </tr>
    <tr>
      <th>A + dir</th>
      <td>押すたびに長方形を動かす。細かい制御用。</td>
    </tr>
    <tr>
      <th>start</th>
      <td>ロケットの位置をランダムに変える。</td>
    </tr>
  </table>
</div>

以下は、デモのコードの大部分です。背景とスプライトを準備する関数は、これまでに見たことのないものが何もないので省きました。先ほどの {@fig:win-dgrm}a は、動作中のデモのスクリーンショットです。

<div id="cd-win-demo">

```c
// win_demo.c

//   bg 0, cbb  0, sbb  2, pb 0: numbered forground
//   bg 1, cbb  0, sbb  3, pb 0: fenced background
//   oam 0: tile 0-3: rocket

//   win 0: objects
//   win 1: bg 0
//   win out : bg 1

#include <tonc.h>
#include "nums.h"
#include "rocket.h"

typedef struct tagRECT_U8 { u8 ll, tt, rr, bb; } ALIGN4 RECT_U8;

// window rectangle regs are write only, so buffers are necessary
// Objects in win0, BG 0 in win1
RECT_U8 win[2]=
{
    { 36, 20,  76,  60 },   // win0: 40x40 rect
    { 12, 12 ,228, 148 }    // win1: screen minus 12 margin.
};

// gfx loaders omitted for clarity
void init_front_map();  // numbers tiles
void init_back_map();   // fence
void init_rocket();     // rocket

void win_copy()
{
    REG_WIN0H= win[0].ll<<8 | win[0].rr;
    REG_WIN1H= win[1].ll<<8 | win[1].rr;
    REG_WIN0V= win[0].tt<<8 | win[0].bb;
    REG_WIN1V= win[1].tt<<8 | win[1].bb;
}

void test_win()
{
    win_copy();
    while(1)
    {
        key_poll();
        vid_vsync();

        // key_hit() or key_is_down() 'switch'
        // A depressed: move on direction press (std movement)
        // A pressed  : moves on direction hit (fine movement)
        int keys= key_curr_state();
        if(key_is_down(KEY_A))
            keys &= ~key_prev_state();

        if(keys & KEY_RIGHT)
        {   win[0].ll++;        win[0].rr++;    }
        else if(keys & KEY_LEFT )
        {   win[0].ll--;        win[0].rr--;    }
        if(keys & KEY_DOWN)
        {   win[0].tt++;        win[0].bb++;    }
        else if(keys & KEY_UP )
        {   win[0].tt--;        win[0].bb--;    }

        // (1) randomize rocket position
        if(key_hit(KEY_START))
            obj_set_pos(&oam_mem[0],
                qran_range(0, 232), qran_range(0, 152));

        win_copy();
    }
}

int main()
{
    // obvious inits
    oam_init();
    init_front_map();
    init_back_map();
    init_rocket();

    // (2) windowing inits
    REG_DISPCNT= DCNT_BG0 | DCNT_BG1 | DCNT_OBJ | DCNT_OBJ_1D |
        DCNT_WIN0 |     // Enable win 0
        DCNT_WIN1;      // Enable win 1

    REG_WININ= WININ_BUILD(WIN_OBJ, (WIN_BG0);
    REG_WINOUT= WINOUT_BUILD(WIN_BG1, 0);

    win_copy();     // Initialize window rects

    test_win();

    return 0;
}
```
</div>

ウィンドウの初期化は、ポイント 2 で行っています。`REG_DISPCNT` で win0 と win1 の両方を有効にし、オブジェクトを win 0 に、背景 0 を win 1 に、背景 1 を winOut に表示します。ウィンドウの大きさは、毎フレーム `win_copy()` で設定しています。ウィンドウの長方形のレジスタ自体は書き込み専用なので、ウィンドウの位置を記録するために、長方形の変数を 2 つ使っています。結果はもう一度 {@fig:win-dgrm} を見てください。

普通は、オブジェクトは背景より手前に表示されます。しかし、ここではオブジェクトは win 0 の内側にだけ表示されるように設定されているので、それ以外の場所では実質的に隠れています。ロケットと win 0 の長方形が重なったときにだけ、ロケットやその一部が見えるのです。さらに、win 0 にはオブジェクト*だけ*が設定されているので、ウィンドウ自体は真っ黒になることにも気づくでしょう。

デモの残りは、かなり地味なものです。**A** を押している間、変数 `keys` を前回のボタンの状態でマスクすることで `key_hit()` 関数と `key_is_down()` 関数を切り替え、X 線のウィンドウを直接動かすか細かく動かすかを切り替える機能を実現している、と説明することもできますが、それほど面白い話ではありませんし、デモの本題からもかなり外れています。デモの本題からは外れますが、触れておく価値の*ある*面白い話が、ロケットの位置のランダム化です。

#### 乱数

コンピュータ上の乱数というのは、少し風変わりな概念です。コンピュータの存在意義は信頼できる計算機であることで、乱数はほとんどその正反対のものだからです。コンピュータが生成する乱数は<dfn>擬似乱数</dfn>とも呼ばれます。本質的にランダムなのではなく、そう*見える*ように決定的に生成されているだけだからです。ある手続きが十分にランダムかどうかを調べる統計的な検定もあります。しかし、ここで話しているのは原子物理学ではなく、ゲームプログラミングです。たいていの場合、たとえば敵を見分けられるパターンなしにジグザグに動かせるようなものがあれば十分です。それでモンテカルロシミュレーションがだめになるかどうかなど、まったく関係ありません。

生成器の一種に<dfn>線形合同法</dfn>の生成器があります。これは *N*<sub>i+1</sub> = (*a·N*<sub>i</sub> + *c*)%*m*（*N*<sub>i</sub>∈\[0, *m*⟩）という形に従います。パラメータ *a*、*c*、*m* を適切に選べば、この手続きはかなり実用的です。何かの標準ライブラリで `rand()` 関数に出会ったら、おそらくこの一種です。実装が簡単なだけでなく、たいていは速くもあります。

以下の手続き `qran()` は、私の数値計算法の本、[Numerical Recipes](http://www.amazon.com/gp/product/0521431085/103-4874440-3995059) の 275 ページから取ったものです。そこでは「手っ取り早くて雑な、しかし実用的な」生成器とされています。加算 1 回と乗算 1 回（*m*=2<sup>32</sup> なので、剰余は自動的に行われます）でできているので、*とても*速いです。実際に返す数は *N* の上位 15 ビットです。上位のビットのほうが下位のビットよりランダムらしいことと、15 ビットなら \[0,32767\] の範囲になり、これが私の知る限り非公式の標準のようなものだからです。生成器の<dfn>種</dfn>（シード）を設定する 2 つ目の関数 `sqran()` があることに注意してください。処理そのものはやはり決定的なので、毎回同じ数列にならないようにするには種が必要です。もちろん、実際にそうなって*ほしい*場合は別です。考えてみれば、これはそれほど変な考えではありません。たとえば、マップの生成に使えます。読み込むたびに同じに見えるようにマップ全体を保存しておく代わりに、種だけを保存しておけば済みます。[Star Control 2](https://sc2.sourceforge.net/) の惑星の地形はこうやって作られています。1000 を超える惑星すべてのビットマップを保存するのは、とても無理だったでしょう。`sqran()` が現在の *N* も返すのはこのためです。必要ならあとでリセットできるようにしているのです。

<div id="cd-qran">

```c
// from tonc_core.h/.c
// A Quick (and dirty) random number generator and its seeder

int __qran_seed= 42;     // Seed / rnd holder

// Seed routine
int sqran(int seed)
{
    int old= __qran_seed;
    __qran_seed= seed;
    return old;
}

//! Quick (and very dirty) pseudo-random number generator
/*! \return random in range [0,8000h>
*/
INLINE int qran()
{
    __qran_seed= 1664525*__qran_seed+1013904223;
    return (__qran_seed>>16) & 0x7FFF;
}
```
</div>

もう一度言いますが、これはあまり高度な乱数生成器ではありません。でも、私が必要とすることには十分です。もっと良い（けれど遅い）ものが欲しければ、[メルセンヌ・ツイスタ](https://en.wikipedia.org/wiki/Mersenne_twister)を試してみてください。PERN の [new sprite page](https://web.archive.org/web/20160323220658/http://www.drunkencoders.com/tutorials/GBA/day_3.html) に良い実装があります。

#### 範囲を指定した乱数

乱数を得ることと、特定の範囲の乱数を得ることは別の話です。もちろん、簡単そうに見えます。たとえば 0 から 240 の間の数なら、240 の剰余を使えばよいでしょう。しかし、GBA にはハードウェアの除算がないので、かなりのサイクルがかかります。幸い、簡単な抜け道があります。

`qran()` の範囲は、stdlib の `rand()` と同じく 0 から 0x8000 の間だと言いました。これを .15 の固定小数点数と解釈すれば、0 から 1 の範囲とも見なせます。240 を掛ければ、望む範囲の乱数が得られ、かかるのは乗算 1 回とシフト 1 回だけです。この方法は、最大の範囲と整数のオーバーフロー（これはどのみち気をつけるべきです）に注意しさえすれば、どんな乱数生成器にも使えます。Tonclib ではこれを `qran_range()` と呼んでいます。

```c
//! Ranged random number
/*! \return random in range [\a min, \a max>
*   \note (max-min) must be lower than 8000h
*/
INLINE int qran_range(int min, int max)
{    return (qran()*(max-min)>>15)+min;     }
```

デモでは、スプライトの位置が常に画面の内側に収まるように、`qran_range()` を 2 回使っています。少し調べれば位置そのものを前もって予測することもできるでしょうが、それほど簡単ではないと思います。そして、本当にそこまでの労力をかけたなら、その苦労に見合う何かを受け取ってもいいと思います。デモを何度か読み込み直すと、位置の並びが毎回同じであることに気づくでしょう。だからこそ、これは*擬似*乱数と呼ばれるのです。違う並びを得るには、種の値を変える必要があります。ここではそれほど重要ではないので一度も種を設定していませんが、よくある技は、時間に関係するもので種を設定することです。たとえば、ゲームの前にあるさまざまなタイトル画面などで数えた、実際にゲームを始めるまでのフレーム数やサイクル数です。種のわずかな違いでも、まったく異なる並びが得られます。

## まとめ {#sec-conc}

技術的に言えば、ゲームでモザイクやブレンドやウィンドウが本当に必要になることはおそらくないでしょうが、「被弾のショック」やスポットライトのような、さりげない効果には最適です。いろいろな種類の場面転換にもとても役立ちます。黒へのフェードは、ブレンドのレジスタを使えば簡単に実装できます。ウィンドウを使ったいろいろな HBlank 効果も楽しいものです。HBlank のたびに長方形を変えて、光線や横方向のワイプや円形のウィンドウを作れます。ただし、そうするには割り込みの扱い方を知っている必要があります。あるいは HDMA と呼ばれる DMA の特殊な使い方でもできます。ちょうど次の章で扱うのがそれです。
