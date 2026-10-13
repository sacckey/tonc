# 9. 通常のタイル背景

<!-- toc -->

## タイルマップの紹介 {#sec-intro}

タイルマップは、GBA の基本中の基本です。市販の GBA のゲームのほとんどすべてがタイルモードを使っていて、ビットマップモードが使われるのは、レイトレーシングを使う 3D のようなゲームだけです。それ以外は、すべてタイルのグラフィックを使っています。

タイルマップがこれほど広く使われている理由は、ハードウェアで実装されていて、ビットマップのグラフィックより少ない容量で済むからです。{@fig:map}a を考えてみてください。これは 512 × 256 の画像で、8bpp でも 128 KiB の VRAM を使ってしまいますが、そんな容量はありません。ゲームの普通のステージを 1 枚の大きなビットマップにすると、簡単に 1000×1000 ピクセルくらいになってしまい、とても実用的ではありません。*そのうえ*ステージをスクロールするには、毎フレーム全部のピクセルを更新する必要があります。スクロールのコードを最大限に最適化しても、かなりの時間がかかるでしょう。

さて、この画像には同じ要素が何度も出てくることに注目してください。ビットマップは 16×16 ピクセルのまとまりに分かれているようです。これが<dfn>タイル</dfn>です。重複しないタイルの一覧が<dfn>タイルセット</dfn>で、{@fig:map}b に示します。見てのとおり、この画像を作っているのは、たった 16 個の異なるタイルです。これらのタイルから画像を作るには、<dfn>タイルマップ</dfn>が必要です。画像はタイルの行列に分けられます。行列の各要素は、そこにどのタイルを描くかを表す<dfn>タイルのインデックス</dfn>を持っています。タイルマップは {@fig:map}c で見られます。

タイルセットとマップの両方が 8 ビットのエントリを使うとすると、大きさはタイルセットが 16×(16×16) = 4096 バイト、タイルマップが 32×16 = 512 バイトです。なので、場面全体で 4.5 KiB になり、前の 128 KiB と比べて 28 分の 1 の大きさになります。

<div class="cblock">
<table width=70% id="fig:map">
<tbody valign="top">
<tr>
<td colspan=2 align="center">
  <div class="cpt" style="width:512px">
  <img src="./img/bgs/brin3-full.png" alt="Brinstar map">
  <b>{*@fig:map}a</b>: 画面上の画像。
  </div>
<tr>
<td colspan=2>
  タイルマッピングの過程。
  {@fig:map}b のタイルセットと、
  {@fig:map}c のタイルマップを使うと、最終的な結果は
  {@fig:map}a になる。
<tr>
<td>
  <div class="cpt" style="width:48px">
  <img src="./img/bgs/brin3-meta-2x.png" height=264
    alt="(meta)tileset for the map"><br>
  <b>{*@fig:map}b</b>: タイルセット。
  </div>		
<td>
  <div class="cpt" style="width:528px">
  <img src="./img/bgs/brin3-map-2x.png" width=528
	alt="Superimposed tile-map"><br>
  <b>{*@fig:map}c</b>: タイルマップ（対応する
    タイルを背後に表示）。
  </div>
</tbody>
</table></div>

基本的に、タイルマップはこうして働きます。画像全体を定義するのではなく、ピクセルをタイルにまとめ、そのまとまりを使って画像を表すのです。{@fig:map} では、タイルは 16×16 ピクセルだったので、タイルマップはビットマップの 256 分の 1 の大きさです。重複しないタイルはタイルセットに入っていて、タイルセットはタイルマップより大きくなることもあります（そして、普通はそうなります）。タイルセットの大きさはまちまちです。ビットマップの変化が激しければ、たぶんたくさんの異なるタイルができます。グラフィックがすでにタイルの境界にきれいに揃っていれば（ここでのように）、タイルセットは小さくなります。タイルエンジンのゲームに独特の見た目があることが多いのは、このためです。

### GBA のタイルマップ {#ssec-intro-gba}

タイルのビデオモード（0、1、2）では、タイルマップを表示する背景を最大 4 つ持てます。マップの大きさは制御のレジスタで設定し、128×128 から 1024×1024 ピクセルまでです。各タイルの大きさは常に 8×8 ピクセルなので、{@fig:map} は GBA での働き方と*完全に*同じではありません。タイルマップへのアクセスはタイル単位で行うので、マップの大きさは 16×16 から 128×128 タイルに当たります。

タイルもタイルマップも VRAM に入っていて、VRAM は<dfn>キャラブロック</dfn>と<dfn>スクリーンブロック</dfn>に分かれています。タイルセットはキャラブロックに入り、タイルマップはスクリーンブロックに入ります。一般的な言い方では、「タイル」という言葉は、グラフィックのタイルとタイルマップのエントリの両方に使われます。これは少し紛らわしいので、私はスクリーンブロックの中の要素（つまりマップのエントリ）には<dfn>スクリーンエントリ</dfn>（略して <dfn>SE</dfn>）という言葉を使い、タイルという言葉はタイルセットに限ることにします。

64 KiB の VRAM が、タイルマップ用に割り当てられています（`0600:0000h`〜`0600:FFFFh`）。これは、スクリーンブロック*と*キャラブロックの両方に使われます。どれを使うかは制御のレジスタで自由に選べますが、重ならないように気をつけてください（{@tbl:cbb-sbb} を参照）。各スクリーンブロックの大きさは 2048（`800h`）バイトで、全部で 32 個のスクリーンブロックがあります。一番小さいものを除いて、背景はどれも、タイルマップ全体に複数のスクリーンブロックを使います。各キャラブロックの大きさは 16 KiB（`4000h` バイト）で、全部で 4 つのブロックがあります。

<div class="cblock">
<table id="tbl:cbb-sbb" rules=groups>
<caption align="bottom">
  <b>{*@tbl:cbb-sbb}</b>:
  キャラブロックとスクリーンブロックの重なり。
</caption>
<colgroup span=1 style="background-color:var(--table-background-color-custom);"></colgroup>
<colgroup span=3 style="background-color:var(--table-alternate-bg);"></colgroup>
<colgroup span=3 style="background-color:var(--table-background-color-custom);"></colgroup>
<colgroup span=3 style="background-color:var(--table-alternate-bg);"></colgroup>
<colgroup span=3 style="background-color:var(--table-background-color-custom);"></colgroup>

<tbody align="left"><tr>
  <th>メモリ
  <th colspan=3> 0600:0000 <th colspan=3> 0600:4000
  <th colspan=3> 0600:8000 <th colspan=3> 0600:C000
</tbody>
<tbody align="center"><tr>
  <th>キャラブロック
  <td colspan=3> 0 <td colspan=3> 1 <td colspan=3> 2 <td colspan=3> 3 
</tbody>
<tbody><tr>
  <th>スクリーンブロック
  <td>0 <td>&hellip; <td>7		<td>8 <td>&hellip; <td>15
  <td>16 <td>&hellip; <td> 23	<td>24 <td>&hellip; <td>31
</tbody>
</table>
</div>

:::warning タイルと「タイル」

タイルマップのエントリとタイルセットのデータは、どちらもよく「タイル」と呼ばれるので、話が紛らわしくなることがあります。私は「タイル」という言葉をグラフィックに限り、マップの中身には「スクリーン（ブロック）エントリ」か「マップエントリ」という言葉を使います。

:::

:::warning キャラブロックとスクリーンブロック

キャラブロックとスクリーンブロックは、メモリ上の同じアドレスを使います。各キャラブロックは、8 つのスクリーンブロックと重なっています。データを読み込むときは、タイル自体がマップを上書きしたり、その逆が起きたりしないように気をつけてください。

:::

大きさはタイルマップを使う利点の 1 つでしたが、速さももう 1 つの利点です。タイルマップの描画はハードウェアで行われます。PC のゲームをハードウェアモードとソフトウェアモードで遊んだことがあれば、ハードウェアがいいものだと知っているでしょう。もう 1 つのよい点は、スクロールもハードウェアで行われることです。場面全体を描き直す代わりに、正しいレジスタに座標を入れるだけで済みます。

概要で言ったとおり、タイル背景の準備には、制御、マッピング、画像データの 3 つの段階があります。画像データのほとんどと、スプライトと背景に共通する制御とマッピングの部分の一部は、[概要](objbg.html)ですでに説明しました。この章では、背景全般、特に通常の背景に特有のことだけを扱います。概要は読んでいるものとします。

:::tip タイルマップに欠かせない手順

-   グラフィックを読み込む: タイルをキャラブロックに、色を背景のパレットに。
-   マップを 1 つ以上のスクリーンブロックに読み込む。
-   `REG_DISPCNT` で正しいモードに切り替え、背景を有効にする。
-   背景の制御のレジスタを、正しい CBB、SBB、ビット深度を使うように初期化する。

:::

## 背景の制御 {#sec-ctrl}

### 背景の種類 {#ssec-ctrl-bgs}

スプライトと同じく、タイル背景にも 2 種類あります。通常の背景とアフィン背景です。それぞれ、テキスト背景、回転背景とも呼ばれます。背景の種類は、ビデオモードで決まります（{@tbl:bg-types} を参照）。根本のところでは、通常の背景もアフィン背景も同じように働きます。タイルとタイルマップと、いくつかの制御のレジスタがあります。でも、似ているのはそこまでです。アフィン背景は、通常の背景より多くの、そして違うレジスタを使い、マップの形式さえ違います。このページでは、通常の背景だけを扱います。[アフィン背景](affbg.html)は、[アフィン行列](affine.html)のページのあとにとっておきます。

<div class="lblock">
<table id="tbl:bg-types" class="table-data">
<caption align="bottom">
  <b>{*@tbl:bg-types}</b>: ビデオモードと
  背景の種類
</caption>
<tbody align="center">
<tr>
  <th>モード	<th>BG0	<th>BG1	<th>BG2	<th>BG3
<tr>
  <td>0     <td>通常	<td>通常	<td>通常	<td>通常
<tr>
  <td>1     <td>通常	<td>通常	<td>アフィン	<td> -
<tr>
  <td>2     <td>-	<td>-	<td>アフィン	<td>アフィン
</tbody>
</table>
</div>

### 制御のレジスタ {#ssec-ctrl-regs}

どの背景にも、主な制御のレジスタが 3 つあります。一番の制御のレジスタは `REG_BGxCNT` で、*x* は背景 0〜3 を表します。このレジスタで、タイルマップの大きさと、どのキャラブロックとスクリーンブロックを使うかを指定します。ほかの 2 つは、スクロールのレジスタの `REG_BGxHOFS` と `REG_BGxVOFS` です。

これらはどれも 16 ビットのレジスタです。`REG_BG0CNT` は `0400:0008` にあり、ほかの背景の制御のレジスタがそのすぐあとに続きます。オフセットのレジスタは背景ごとに対になっていて、座標の組を作っています。これらは `0400:0010` から始まります。

<div class="lblock">
<table id="tbl:ctrl-ofs" class="table-data">
<caption align= bottom>
  <b>{*@tbl:ctrl-ofs}</b>: 背景のレジスタの
    アドレス
</caption>
<col span=2 align="right">
  <tr><th>レジスタ      <th>長さ  <th>アドレス
  <tr><th>REG_BGxCNT    <td>2       <td>0400:0008h + 2·<i>x</i>
  <tr><th>REG_BGxHOFS   <td>2       <td>0400:0010h + 4·<i>x</i>
  <tr><th>REG_BGxVOFS   <td>2       <td>0400:0012h + 4·<i>x</i>
</table>
</div>

`REG_BGxCNT` の説明は下のとおりです。大きさ以外は、ほとんどごく標準的です。大きさについては、実は可能な大きさの一覧が*2 つ*あります。1 つは通常のマップ用、もう 1 つはアフィンのマップ用です。どちらも同じビットを使うので、正しい `#define` を使っているか気をつける必要があるかもしれません。

<div class="reg">
<table class="table-reg" id="tbl-reg-bgxcnt">
<caption class="reg">
  REG_BGxCNT @ 0400:0008 + 2<i>x</i>
</caption>
<tr class="bits">
	<td>F E<td>D<td> C B A 9 8
	<td>7<td>6<td>5 4<td>3 2<td>1 0
<tr class="bf">
	<td class="rclr2">Sz
	<td class="rclr6">Wr
	<td class="rclr1">SBB
	<td class="rclr3">CM
	<td class="rclr5">Mos
	<td>-
	<td class="rclr0">CBB
	<td class="rclr4">Pr
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-1<td class="rclr4">Pr
  <td><i>BG_PRIO#</i>
  <td><b>優先度</b>。背景を描く順番を決める。
<tr class="bg1">	
  <td>2-3<td class="rclr0">CBB
  <td><i>BG_CBB#</i>
  <td><b>キャラクタベースブロック</b>。キャラクタ（タイル）のインデックスの
    基点となるキャラブロックを設定する。値: 0-3。
<tr class="bg0">	
  <td> 6 <td class="rclr5">Mos
  <td>BG_MOSAIC
  <td><b>モザイク</b>のフラグ。モザイク効果を有効にする。
<tr class="bg1">	
  <td> 7 <td class="rclr3">CM
  <td>BG_4BPP, BG_8BPP
  <td><b>カラーモード</b>。消えていれば 16 色（4bpp）、
    立っていれば 256 色（8bpp）。
<tr class="bg0">	
  <td>8-C<td class="rclr1">SBB
  <td><i>BG_SBB#</i>
  <td><b>スクリーンベースブロック</b>。スクリーンエントリ（マップ）のインデックスの
    基点となるスクリーンブロックを設定する。値: 0-31。
<tr class="bg1">	
  <td> D <td class="rclr6">Wr
  <td>BG_WRAP
  <td><b>アフィンの折り返し</b>のフラグ。立っていれば、アフィン背景は端で
    折り返す。通常の背景はもともと折り返すので、
    効果はない。
<tr class="bg0">	
  <td>E-F<td class="rclr2">Sz
  <td><i>BG_SIZE#</i>, <i class="mini">下を参照</i>
  <td><b>背景の大きさ</b>。通常の背景とアフィン背景では、
      使える大きさが違う。タイル単位とピクセル単位の大きさは、
      {@tbl:bg-size} にある。
</tbody>
</table>
</div>

<div class="cblock" id="tbl:bg-size">
<div style="display: inline-block">
  <table id="tbl-reg-size" class="table-data">
  <caption align="bottom">
    <b>{*@tbl:bg-size}a</b>: 通常の背景の大きさ
  </caption>
  <col><col class="def">
  <tbody align="center">
    <tr><th>Sz のフラグ	<th>define    <th>(タイル)<th>(ピクセル)
    <tr><td> 00   <td><code>BG_REG_32x32</code> <td> 32×32 <td> 256×256 
    <tr><td> 01   <td><code>BG_REG_64x32</code> <td> 64×32 <td> 512×256 
    <tr><td> 10   <td><code>BG_REG_32x64</code> <td> 32×64 <td> 256×512 
    <tr><td> 11   <td><code>BG_REG_64x64</code> <td> 64×64 <td> 512×512 
  </tbody>
  </table>
</div>
<div style="display: inline-block">
  <table id="tbl-aff-size" class="table-data">
  <caption align="bottom">
    <b>{*@tbl:bg-size}b</b>: アフィン背景の大きさ
  </caption>
  <col><col class="def">
  <tbody align="center">
    <tr><th>Sz のフラグ	<th>define    <th>(タイル) <th>(ピクセル)
    <tr><td> 00   <td><code>BG_AFF_16x16</code>  <td> 16×16  <td> 128×128 
    <tr><td> 01   <td><code>BG_AFF_32x32</code>  <td> 32×32  <td> 256×256 
    <tr><td> 10   <td><code>BG_AFF_64x64</code>  <td> 64×64  <td> 512×512 
    <tr><td> 11   <td><code>BG_AFF_128x128</code><td>128×128 <td>1024×1024
  </tbody>
  </table>
</div>
</div>

各背景には、描画をずらすための 16 ビットのスクロールのレジスタが 2 つあります（`REG_BGxHOFS` と `REG_BGxVOFS`）。これらにはいくつか面白い点があります。まず、通常の背景は折り返すので、値は本質的に *マップの大きさ* を法とした剰余になります。今はあまり関係ありませんが、もっと高度なタイルマップを扱うようになったら、これをうまく使えます。次に、これらのレジスタは**書き込み専用**です！ これは少し厄介です。`REG_BG0HOFS++` のように書いて位置を更新することができないからです。

そして 3 つ目の点は、たぶん一番大事なことで、値が実際に何を*する*のかです。一番単純な見方は、値はマップ上での画面の座標を表す、というものです。もう一度、注意して読んでください。マップ上での画面の位置です。スプライトのような、画面上でのマップの位置では*ありません*。違いはマイナス記号だけですが、符号が変わるという小さなことでも、計算を大混乱させることがあります。

<div class="lblock">
  <div class="cpt" style="width:520px;">
    <img src="./img/bgs/brin3-ofs-2x.png" id="fig:map-ofs" width=520
      alt="map-ofs-a"><br>
    <b>{*@fig:map-ofs}</b>: 
	スクロールのオフセット <b>dx</b> は、マップ上での画面の位置を
	設定する。この場合、<b>dx</b> = (192, 64)。
  </div>
</div>

なので、スクロールの値を増やすと、画面が右に動き、これは画面上でマップが*左に*動くことに当たります。数学の言葉で言うと、マップの位置を **p**、画面の位置を **q** とすると、次の式が成り立ちます。

<table id="eq:bgr-dx">
<tr>
  <td class="eqnrcell">({!@eq:bgr-dx})
  <td class="eqcell">
  <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
    <mstyle displaystyle="true" scriptlevel="0">
      <mrow data-mjx-texclass="ORD">
        <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
          <mtr>
            <mtd>
              <mi mathvariant="bold">q</mi>
              <mo>+</mo>
              <mi mathvariant="bold">d</mi>
              <mi mathvariant="bold">x</mi>
            </mtd>
            <mtd>
              <mo>=</mo>
            </mtd>
            <mtd>
              <mi mathvariant="bold">p</mi>
            </mtd>
          </mtr>
          <mtr>
            <mtd>
              <mi mathvariant="bold">q</mi>
            </mtd>
            <mtd>
              <mo>=</mo>
            </mtd>
            <mtd>
              <mi mathvariant="bold">p</mi>
              <mo>&#x2212;</mo>
              <mi mathvariant="bold">d</mi>
              <mi mathvariant="bold">x</mi>
            </mtd>
          </mtr>
        </mtable>
      </mrow>
    </mstyle>
  </math>
</table>

:::warning オフセットのレジスタの向き

オフセットのレジスタ REG_BGxHOFS と REG_BGxVOFS は、マップのどの位置を画面の左上に対応づけるかを表します。つまり、正のオフセットは、マップを左と上にスクロールします。マイナス記号に注意してください。

:::

:::warning オフセットのレジスタは書き込み専用です

オフセットのレジスタは**書き込み専用**です！ つまり、`+=` のような直接の算術演算は動きません。

:::

### 便利な型と #define {#ssec-ctrl-types}

Tonc のコードには、少し楽にしてくれる、便利な追加の型とマクロがいくつかあります。

```c
// === Additional types (tonc_types.h) ================================

//! Screen entry conceptual typedef
typedef u16 SCR_ENTRY;

//! Affine parameter struct for backgrounds, covered later
typedef struct BG_AFFINE
{
    s16 pa, pb;
    s16 pc, pd;
    s32 dx, dy;
} ALIGN4 BG_AFFINE;

//! Regular map offsets
typedef struct BG_POINT
{
    s16 x, y;
} ALIGN4 BG_POINT;

//! Screenblock struct
typedef SCR_ENTRY   SCREENBLOCK[1024];


// === Memory map #defines (tonc_memmap.h) ============================

//! Screen-entry mapping: se_mem[y][x] is SBB y, entry x
#define se_mem          ((SCREENBLOCK*)MEM_VRAM)

//! BG control register array: REG_BGCNT[x] is REG_BGxCNT
#define REG_BGCNT      ((vu16*)(REG_BASE+0x0008))

//! BG offset array: REG_BG_OFS[n].x/.y is REG_BGnHOFS / REG_BGnVOFS
#define REG_BG_OFS      ((BG_POINT*)(REG_BASE+0x0010))

//! BG affine params array
#define REG_BG_AFFINE   ((BG_AFFINE*)(REG_BASE+0x0000))
```

厳密に言えば、`SCREEN_ENTRY` の `typedef` を作る必要はありませんが、使い道が分かりやすくなります。`se_mem` は `tile_mem` とよく似た働きをします。VRAM をスクリーンブロックとスクリーンエントリに対応づけて、特定のエントリを見つけやすくします。ほかの typedef は、背景のレジスタを配列として対応づけるのに使います。たとえば、`REG_BGCNT` は、すべての `REG_BGxCNT` レジスタを対応づける配列です。`REG_BGCNT[0]` は `REG_BG0CNT`、という具合です。`BG_POINT` 型と `BG_AFFINE` 型も同じように使います。`REG_BG_OFS` が指すのは、あくまで `REG_BGxHOFS` と `REG_BGxVOFS` と同じレジスタで、書き込み専用という性質が魔法のように消えたわけではないことに注意してください。`REG_BG_AFFINE` も同じですが、その話はまたの機会にとっておきます。

理屈の上では、一種の背景の API を作るのも便利です。マップの位置を一時的に保存する構造体と、レジスタやマップを初期化したり更新したりする関数を持つものです。でも、tonc のデモのほとんどは、そこまでする必要があるほど複雑ではありません。上の型があれば、必要なものを扱うのは、今のところもう十分簡単です。

## 通常の背景のタイルマップ {#sec-map}

スクリーンブロックは、画面上の画像全体を表すスクリーンエントリの行列を形作ります。{@fig:map} の例では、タイルマップのエントリにはタイルのインデックスしか入っていませんでした。GBA のスクリーンエントリは、少し違う振る舞いをします。

通常のタイルマップでは、各スクリーンエントリの長さは 16 ビットです。タイルのインデックスのほかに、反転のフラグと、4bpp / 16 色のタイルのためのパレットバンクのインデックスを持っています。正確な並びは、下の「スクリーンエントリの形式」にあります。アフィンのスクリーンエントリは 8 ビットしかなく、8 ビットのタイルのインデックスだけを持っています。

<div class="reg">
<table class="table-reg" id="tbl-se"
  border=1 frame=void cellpadding=4 cellspacing=0>
<caption class="reg">
  通常の背景のスクリーンエントリの形式
</caption>
<tr class="bits">
	<td>F E D C<td>B<td>A<td>9 8 7 6 5 4 3 2 1 0
<tr class="bf">
	<td class="rclr1">PB
	<td class="rclr2">VF
	<td class="rclr2">HF
	<td class="rclr0">TID
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width=12%>
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-9<td class="rclr0">TID
  <td><i>SE_ID#</i>
  <td>SE の<b>タイルのインデックス</b>。
<tr class="bg1">	
  <td>A-B<td class="rclr2">HF, VF
  <td>SE_HFLIP, SE_VFLIP. <i>SE_FLIP#</i>
  <td><b>水平・垂直反転</b>のフラグ。
<tr class="bg0">	
  <td>C-F<td class="rclr1">PB
  <td><i>SE_PALBANK#</i>
  <td>16 色モードのときに使う<b>パレットバンク</b>。256 色の背景
    （<code>REG_BGxCNT{6}</code> が立っている）では効果はない。
</tbody>
</table>
</div>

### マップの並び {#ssec-map-layout}

VRAM には、タイルマップを入れるためのスクリーンブロックが 32 個あります。各スクリーンブロックの大きさは 800h バイトなので、32×32 個のスクリーンエントリが入り、これは 256×256 ピクセルのマップ 1 枚に当たります。もっと大きなマップは、単純に複数のスクリーンブロックを使います。`REG_BGxCNT` で設定するスクリーンブロックのインデックスは<dfn>スクリーンベースブロック</dfn>で、タイルマップの始まりを表します。

さて、大きさが *tw*×*th* タイル（SE）のタイルマップがあるとします。タイルの座標 (*tx*, *ty*) にあるスクリーンエントリは、SE の番号 *n* = *tx*+*ty*·*tw* にあると思うかもしれません。行列はいつもそうなっていますよね？ でも、それは間違いです。少なくとも、*一部*間違っています。

各スクリーンブロックの中ではこの式が成り立ちますが、大きな背景は単に複数のスクリーンブロックを*使う*だけでなく、実際には 4 つの別々のマップとしてアクセスされます。これがどういう仕組みかは、{@tbl:reg-layout} で分かります。番号の付いた各ブロックは、メモリ上で連続したブロックです。つまり、SE のインデックスを求めるには、どのスクリーンブロックにいるかを割り出してから、そのスクリーンブロックの中での SE の番号を求める必要があります。

<div class="lblock">
<table class="table-reg" id="tbl:reg-layout"
  border=1 frame=void cellpadding=4 cellspacing=0>
<caption align="bottom">
  <b>{*@tbl:reg-layout}</b>: 通常の背景の
  スクリーンブロックの並び。
 </caption>
<col span=4 align="center">
<tr><th>32×32<th>64×32<th>32×64<th>64×64
<tr>
<td>
  <table border=frame cellpadding= 8 cellspacing=0>
    <tr><td bgcolor=red><b>0</b>
  </table>
<td>
  <table border=frame cellpadding= 8 cellspacing=0>
    <tr><td bgcolor=red><b>0</b><td bgcolor= green><b>1</b>
  </table>
<td>
  <table border=frame cellpadding= 8 cellspacing=0>
    <tr><td bgcolor=red><b>0</b>
    <tr><td bgcolor=blue><b>1</b>
  </table>
<td>
  <table border=frame cellpadding= 8 cellspacing=0>
    <tr><td bgcolor=red><b>0</b><td bgcolor= green><b>1</b>
    <tr><td bgcolor=blue><b>2</b><td bgcolor= gray><b>3</b>
  </table>
</table>
</div>

こうした入れ子の問題は、見た目ほど難しくありません。1 つのスクリーンブロックに何個のタイルが入るかは分かっているので、SBB の座標を求めるには、タイルの座標を SBB の幅と高さで割るだけです。*sbx*=*tx*/32、*sby*=*ty*/32 です。すると、SBB の番号は、行列から配列への標準的な式で求められます。SBB の中での SE の番号を求めるには、*tx*%32 と *ty*%32 を使って SBB の中での座標を求め、それをまた 2 次元の座標から 1 つの要素の番号に変換します。最終的な番号を求めるには、それを SBB の番号と SBB の大きさの積だけずらします。最終的な形は次のようになります。

<div id="cd-se-index">

```c
//! Get the screen entry index for a tile-coord pair
//  And yes, the div and mods will be converted by the compiler
uint se_index(uint tx, uint ty, uint pitch)
{
    uint sbb= (ty/32)*(pitch/32) + (tx/32);
    return sbb*1024 + (ty%32)*32 + tx%32;
}
```
</div>

一般的な式は、読者への練習問題とします。私の意見では、十分取り組む価値のある問題です。この種の処理は、タイルに分けたビットマップの座標のオフセットを求めるときや、1D のオブジェクトマッピングでのタイルの座標など、いろいろな場所で出てきます。

こうした演算を見て気分が悪くなるなら、2×2 の並び専用の、もっと速い版もあります。まず、32×32t のマップであるかのように番号を計算します。幅 64t のマップではこれは正しくないので、0x0400−0x20（つまり、ブロックあたりのタイル数 − 1 行あたりのタイル数）を足して補正します。大きさが 64×64t なら、さらにブロック 1 つ分の補正が必要です。

<div id="cd-se-index-fast">

```c
//! Get the screen entry index for a tile-coord pair.
/*! This is the fast (and possibly unsafe) way.
*   \param bgcnt    Control flags for this background (to find its size)
*/

uint se_index_fast(uint tx, uint ty, u16 bgcnt)
{
    uint n= tx + ty*32;
    if(tx >= 32)
        n += 0x03E0;
    if(ty >= 32 && (bgcnt&BG_REG_64x64)==BG_REG_64x64)
        n += 0x0400;
    return n;
}
```
</div>

ここでの *n* はアドレスではなく、SE の番号だということを思い出してください。通常の SE の大きさは 2 バイトなので、アドレスにするには *n* に 2 を掛ける必要があります。（もちろん、`u16` のポインタや配列を使っているなら、*n* のままでうまくいきます。）また、これは通常の背景にしか使えません。アフィン背景は線形のマップ構造を使うので、そこではこの余分な作業は要りません。ちなみに、アフィン背景では、スクリーンエントリとマップの並びの両方が違います。その形式については、アフィン背景のページの[マップの形式](affbg.html#sec-map)の節を見てください。

### 背景のタイルの微妙な点 {#ssec-map-subtle}

タイルマップにタイルを使うときには、ほかに知っておくべきことが 2 つあります。1 つ目は、タイルの番号の付け方です。スプライトでは、番号は 4 ビットのタイル（s タイル）単位で付けられ、8 ビットのタイル（d タイル）には 2 の倍数を使う必要がありました（u16 のアドレスがメモリ上で常に 2 の倍数になるのと少し似ています）。でもタイルマップでは、d タイルは d タイル単位で番号が付けられます。言い換えると、スプライトでは、インデックス *id* は 4 ビットのタイルでも 8 ビットのタイルでも同じタイル、つまり *id*·20h から始まるタイルを指します。でもタイルマップでは、4 ビットのタイルなら *id*·20h から始まりますが、8 ビットのタイルなら *id*·40h から始まります。

<div class="lblock">
<table id="tbl:bg-tids" class="table-data">
<caption align="bottom">
  {*@tbl:bg-tids}: 背景のタイルの数え方。
  ビット深度に従う。
</caption>
<tbody align="center">
<tr>
  <th>メモリのオフセット<th>000h<th>020h <th>040h<th>060h <th>080h<th>100h <th>...
<tr>
  <th>4bpp のタイル <td>0      <td>1  <td>2    <td>3  <td>4    <td>5 <td>...
<tr>
  <th>8bpp のタイル <td colspan=2>0   <td colspan=2>1 <td colspan=2>2<td>...
</tbody>
</table>
</div>

2 つ目は、まあ、これもタイルの番号の話ですが、どれだけの数のタイルを使えるかについてです。通常の背景のマップのエントリは、タイルのインデックスに 10 ビットを持っているので、最大 1024 個のタイルを使えます。でも、ちょっと計算すると、キャラブロックに入るのは 4000h/20h= 512 個の s タイル、または 4000h/40h= 256 個の d タイルです。では、どういうことなのでしょうか？ 実は、`REG_BGxCNT` で設定するキャラブロックのインデックスは、タイルを数え始めるブロック、つまり<dfn>キャラクタベースブロック</dfn>にすぎません。そのあとのブロックも使えるのです。すごいでしょう？ でも待ってください。続くキャラブロックにもアクセスできるなら、基点のキャラブロックを 3 にすれば、スプライトのブロック（基本的にはブロック 4 と 5）も使えるということでしょうか？

答えは、イエス。そして<span class="ack">ノー</span>です！

2000年代初めのエミュレータでは、これができます。でも、本物の GBA ではできません。ただし、*何か*は表示されます。スクリーンエントリそのものがタイルのデータとして使われるのですが、その使われ方はまったく説明のつかないものです。ここは私を信じてください。今の tonc のデモの中で、VBA が間違えるのはここです。

:::note 使えるタイル

4bpp と 8bpp のどちらの通常の背景でも、1024 個のタイルにアクセスできます。唯一の注意点は、インデックスがそれを指していても、オブジェクトのキャラブロックにあるタイルにはアクセスできないことです。

:::

もう 1 つ疑問に思うかもしれないのは、今使っているキャラブロックの中にある特定のスクリーンブロックを使えるかどうかです。たとえば、ある背景にキャラブロック 0 とスクリーンブロック 1 を使ってもいいのでしょうか。これも、できます。キャラブロック全体を埋めることはまずないので、そのあとのほうのスクリーンブロックをマップのデータに使うのはいい考えです。（真のハッカーの証は、同じデータをタイルと SE の両方に使って、しかも意味のある画像を出すことでしょう（この最後の部分が大事です）。もしやり遂げたら、ぜひ教えてください。）

:::note コマンドラインでのタイルマップのデータの変換

（オブジェクト用に）画像をタイル化できる変換ツールは、タイルマップ用のタイルセットも作れます。ただし、重複したタイルがたくさんできる可能性が高いです。変換ツールの中には、タイルセットを重複しないタイルだけに減らし、それに対応するタイルマップを出してくれるものもあります。{@fig:map} のブリンスタのビットマップは 512×256 の画像で、パレットの情報と鏡像化も含めてタイルの重複をなくした 4bpp のタイルセットを使えば、64×32 のマップにタイル化できます。

```sh
# gfx2gba
# (C array; u8 foo_Tiles[], u16 foo_Map[], 
# u16 master_Palette[]; foo.raw.c, foo.map.c, master.pal.c)
    gfx2gba -fsrc -c16 -t8 -m foo.bmp
```

```sh
# grit
# (C array; u32 fooTiles[], u16 fooMap[], u16 fooPal[]; foo.c, foo.h)
    grit foo.bmp -gB4 -mRtpf
```

gfx2gba について、注意が 2 つあります。まず、パレットを 1 つの 16 色の配列にまとめ、その過程で並べ替えます。次に、readme にはメタマッピングのオプションが載っていますが、実際にはメタマップとメタタイルセットを作るわけではなく、マップを別々のブロックに整形するだけです。

:::

## タイルマップのデモ {#sec-demo}

この章にはデモが 4 つあります。1 つ目は *brin_demo* で、とてもとても短く、タイルの読み込みとスクロールの基本的な手順を示します。次の *sbb_reg* と *cbb_demo* は技術デモで、複数のスクリーンブロックの並び方と、4bpp と 8bpp の背景でタイルのインデックスがどう付けられるかを示します。どちらの場合も、ここでは手で作るほうが便利なので、マップのデータは手で作っていますが、マップエディタで作ったマップのデータを使っても、本当にたいして違いはありません。

### タイルマップに欠かせない手順: brin_demo {#ssec-demo-brin}

この章を通してブリンスタの 512×256 の部分を使ってきたので、デモにも使ってみることにしました。

使えるマップエディタはいくつかあります。よいものを 2 つ挙げると、Nessie さんの [MapEd](https://nessie.gbadev.org) と [Mappy](https://www.tilemap.co.uk/mappy.php) で、どちらも面白い機能がいくつもあります。私も自分のマップエディタ [mirach](https://www.coranac.com/projects/#mirach) を持っていますが、ごく基本的なものにすぎません。チュートリアルによっては、GBAMapEditor を勧めているかもしれません。このエディタは使わ*ないでください*。かなりバグが多く、ときどきタイルマップの半分が抜け落ちます。タイルマップは、マップのデータが壊れているかどうかを心配しなくても、初心者にとっては十分厄介なものです。

ただし、今回はエディタをまったく使っていません。グラフィックの変換ツールの中には、タイルセットとタイルマップに変換できるものがあります。標準的なやり方ではありませんが、小さなマップならそのほうが簡単かもしれません。今回は Usenti を使いましたが、grit や gfx2gba でも同じようにできます。ここでのマップは 64×32 タイルなので、スクリーンブロックに分ける必要があることに注意してください。Usenti ではこれを「sbb」の並びと呼び、grit では「-mLs」、gfx2gba では「-mm 32」を使います……たぶん。いずれにせよ、変換すると、パレット、タイルセット、タイルマップができます。

<div class="cblock">
<table id="fig:brin" width=100%>
<tr>
  <td valign="top" width=160>
    <div class="cpt">
    <img src="./img/demo/brin_demo_pal.png" 
      alt=""><br>
    <b>{*@fig:brin}a</b>: <i>brin_demo</i> のパレット。
	</div>
  <td rowspan=2>

```c
const unsigned short brinMap[2048]=
{
    // Map row 0
    0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x3001,0x3002,
    0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,
    0x3001,0x3002,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,
    0x3001,0x3002,0x0000,0x0000,0x3001,0x3002,0x0000,0x0000,

    // Map row 1
    0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x3003,0x3004,
    0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,
    0x3003,0x3004,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,
    0x3003,0x3004,0x0000,0x0000,0x3003,0x3004,0x0000,0x0000,

    // Map row 2
    0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,0x0000,
    0x3001,0x3002,0x3005,0x3006,0x3007,0x3008,
    // ... etc
```
<tr>
  <td valign="bottom">
    <div class="cpt">
    <img src="./img/demo/brin_demo_tiles.png" alt=""><br>
    <b>{*@fig:brin}b</b>: <i>brin_demo</i> のタイルセット。
    </div>
</table>
</div>

{@fig:brin} には、パレット全体、タイルセット、そしてマップの一部が見えます。{@fig:brin}b のタイルセットは、{@fig:map}b のものとは違うことに注意してください。前者は 8×8 のタイルを使い、後者は 16×16 のタイルを使っていたからです。また、ここに見えるスクリーンエントリは、0（つまり空のタイル）か、`0x3xxx` の形のどちらかであることにも注意してください。上位のニブルはパレットバンクを表していて、ここでは 3 です。パレット（{@fig:brin}a）を見れば、これが青っぽい色になることが分かります。

では、これらのデータを使ってみましょう。ここでの欠かせない手順を思い出してください。

-   グラフィックを読み込む: タイルをキャラブロックに、色を背景のパレットに。
-   マップを 1 つ以上のスクリーンブロックに読み込む。
-   `REG_DISPCNT` で正しいモードに切り替え、背景を有効にする。
-   背景の制御のレジスタを、正しい CBB、SBB、ビット深度を使うように初期化する。

正しくやれば、画面に何かが表示されるはずです。表示されなければ、エミュレータのタイル、マップ、メモリのビューアを見てみてください。たいていは、どこに問題があるかの見当がつきます。よくあるのが、`REG_BGxCNT` の CBB と SBB が、データを置いた場所と食い違っていることで、その場合はたぶん、マップかタイルセットが空のままになります。

*brin_demo* のコード全体を下に示します。3 回の `memcpy()` の呼び出しで、パレット、タイルセット、タイルマップを読み込みます。たぶんファミコンや 8 ビットのゲームボーイがビデオメモリのどこにスクリーンブロックを置いていたかに関係しているのでしょうが、GBA でもマップを最後のほうのスクリーンブロックに置くのが慣習になっています。今回は、64×32t のマップにはブロックが 2 つ必要なので、31 ではなく 30 です。スクロールの部分では、スクロールのレジスタが書き込み専用なので、位置を保存して更新するために 2 つの変数を使っています。(192, 64) から始めているのは、さっきの {@fig:map-ofs} のスクロールの図でそれを使ったからです。

<div id="cd-brin-demo">

```c
#include <string.h>

#include "toolbox.h"
#include "input.h"
#include "brin.h"

int main()
{
    // Load palette
    memcpy(pal_bg_mem, brinPal, brinPalLen);
    // Load tiles into CBB 0
    memcpy(&tile_mem[0][0], brinTiles, brinTilesLen);
    // Load map into SBB 30
    memcpy(&se_mem[30][0], brinMap, brinMapLen);

    // set up BG0 for a 4bpp 64x32t map, using
    //   using charblock 0 and screenblock 31
    REG_BG0CNT= BG_CBB(0) | BG_SBB(30) | BG_4BPP | BG_REG_64x32;
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    // Scroll around some
    int x= 192, y= 64;
    while(1)
    {
        vid_vsync();
        key_poll();

        x += key_tri_horz();
        y += key_tri_vert();

        REG_BG0HOFS= x;
        REG_BG0VOFS= y;
    }

    return 0;
}
```
</div>

<div class="cblock">
<table id="fig:brin-demo">
<tr>
  <td>
	<div class="cpt" style="width:240px;">
	  <img src="./img/demo/brin_demo.png"
		alt=""><br>
	  <b>{*@fig:brin-demo}a</b>: <b>dx</b>=(192, 64) での
	    <i>brin_demo</i>。
	</div>
  <td>
	<div class="cpt" style="width:240px;">
	  <img src="./img/demo/brin_demo0.png"
		alt=""><br>
	  <b>{*@fig:brin-demo}b</b>: <b>dx</b>=(0, 0) での
	    <i>brin_demo</i>。
	</div>
</table>
</div>

#### 幕間: SBB 用に準備されていないマップの高速なコピー

これは必須の知識ではありませんが、読み物として面白いはずです。このデモでは、もともと複数の SBB 用に準備された、複数の SBB にまたがるマップを使っています。変換ツールが、マップの左のブロックが右のブロックより先に来るようにしてくれたのです。そうでなければ、マップ全体を一度に読み込むことはできません。左のブロックの 2 行目に、右のブロックの 1 行目が入ってしまう、という具合になるからです（{@fig:brin-bad} を参照）。

<div class="lblock">
	<div class="cpt" style="width:512px;">
	  <img src="./img/demo/brin_demo_bad.png" id="fig:brin-bad"
		alt=""><br>
	  <b>{*@fig:brin-bad}</b> 先に SBB ごとのブロックに分けなかった場合の
	  <i>brin_demo</i>。
	</div>
</div>

SBB 用に準備されていないマップを複数のスクリーンブロックにコピーするには、単純で遅い方法がいくつかと、単純で速い方法が 1 つあります。遅い方法は、二重ループで各スクリーンブロックを 1 行ずつ処理することです。速い方法は、構造体のコピーとポインタの計算を使うもので、次のようになります。

<div id="lin2sbb-fast">

```c
typedef struct { u32 data[8]; } BLOCK;

int iy;
BLOCK *src= (BLOCK*)brinMap;
BLOCK *dst0= (BLOCK*)se_mem[30];
BLOCK *dst1= (BLOCK*)se_mem[31];

for(iy=0; iy<32; iy++)
{
    // Copy row iy of the left half
   *dst0++= *src++;     *dst0++= *src++;

     // Copy row iy of the right half
   *dst1++= *src++;     *dst1++= *src++;
}
```
</div>

`BLOCK` の構造体のコピー 1 回で半行分を処理するので、2 回でスクリーンブロックの 1 行全体を処理できます（そう、`BLOCK` を 16 ワードの構造体として定義することもできますが、そうするとうまくいかなくなります。信じてください）。その時点で、`src` ポインタはマップの右半分に来ているので、次の行を右側のコピー先 `dst1` にコピーします。それが終わると、`src` は左側の 2 行目を指しています。これを 32 行全部に対して行います。構造体のコピーとポインタに万歳！

### スクリーンブロックのデモ {#ssec-demo-sbb}

2 つ目のデモ *sbb_reg* は、64×64t の背景を使って、大きなマップに複数のスクリーンブロックがどう使われるかを、もっと詳しく示します。*brin_demo* も複数の SBB にまたがるマップを使っていましたが、マップが不規則だったので、どこが何なのか見分けにくかったはずです。このデモではとても単純なタイルセットを使うので、スクリーンブロックの境界がはっきり見えます。また、スクロールに `REG_BGxHOFS` と `REG_BGxVOFS` ではなく、`REG_BG_OFS` レジスタを使う方法も示します。


<div id="cd-demo-sbb">

```c
#include "toolbox.h"
#include "input.h"

#define CBB_0  0
#define SBB_0 28

#define CROSS_TX 15
#define CROSS_TY 10

BG_POINT bg0_pt= { 0, 0 };
SCR_ENTRY *bg0_map= se_mem[SBB_0];


uint se_index(uint tx, uint ty, uint pitch)
{
    uint sbb= ((tx>>5)+(ty>>5)*(pitch>>5));
    return sbb*1024 + ((tx&31)+(ty&31)*32);
}

void init_map()
{
    int ii, jj;

    // initialize a background
    REG_BG0CNT= BG_CBB(CBB_0) | BG_SBB(SBB_0) | BG_REG_64x64;
    REG_BG0HOFS= 0;
    REG_BG0VOFS= 0;

    // (1) create the tiles: basic tile and a cross
    const TILE tiles[2]=
    {
        {{0x11111111, 0x01111111, 0x01111111, 0x01111111,
          0x01111111, 0x01111111, 0x01111111, 0x00000001}},
        {{0x00000000, 0x00100100, 0x01100110, 0x00011000,
          0x00011000, 0x01100110, 0x00100100, 0x00000000}},
    };
    tile_mem[CBB_0][0]= tiles[0];
    tile_mem[CBB_0][1]= tiles[1];

    // (2) create a palette
    pal_bg_bank[0][1]= RGB15(31,  0,  0);
    pal_bg_bank[1][1]= RGB15( 0, 31,  0);
    pal_bg_bank[2][1]= RGB15( 0,  0, 31);
    pal_bg_bank[3][1]= RGB15(16, 16, 16);

    // (3) Create a map: four contingent blocks of 
    //   0x0000, 0x1000, 0x2000, 0x3000.
    SCR_ENTRY *pse= bg0_map;
    for(ii=0; ii<4; ii++)
        for(jj=0; jj<32*32; jj++)
            *pse++= SE_PALBANK(ii) | 0;
}

int main()
{
    init_map();
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0 | DCNT_OBJ;

    u32 tx, ty, se_curr, se_prev= CROSS_TY*32+CROSS_TX;

    bg0_map[se_prev]++; // initial position of cross
    while(1)
    {
        vid_vsync();

        key_poll();

        // (4) Moving around
        bg0_pt.x += key_tri_horz();
        bg0_pt.y += key_tri_vert();

        // (5) Testing se_index 
        // If all goes well the cross should be around the center of
        // the screen at all times.
        tx= ((bg0_pt.x>>3)+CROSS_TX) & 0x3F;
        ty= ((bg0_pt.y>>3)+CROSS_TY) & 0x3F;

        se_curr= se_index(tx, ty, 64);
        if(se_curr != se_prev)
        {
            bg0_map[se_prev]--;
            bg0_map[se_curr]++;
            se_prev= se_curr;
        }

        REG_BG_OFS[0]= bg0_pt;  // write new position
    }
    return 0;
}
```
</div>

<div class="cpt_fr" style="width:240px">
<img src="./img/demo/sbb_reg.png" id="fig:sbb-reg"
  alt="sbb_reg"><br>
<b>{*@fig:sbb-reg}</b>: <i>sbb_reg</i>。
  64×64t の背景の {@tbl:reg-layout} と比べてみてください。
  左上の隅にある小さなバツ印に注目。
</div>

`init_map()` には、初期化の手順が全部入っています。レジスタ、タイル、パレット、マップの準備です。前のデモと違って、タイル、パレット、マップはすべて手で作っています。この場合は、そのほうが単純に簡単だからです。ポイント (1) で、2 つのタイルを定義しています。1 つ目は窓ガラスのような見た目で、2 つ目は簡単なバツ印です。どちらも画面写真（{@fig:brin-demo}）ではっきり見えます。窓ガラスのようなタイルはタイル 0 に読み込むので、マップの「デフォルト」のタイルになります。

パレットはポイント (2) で設定します。色は {@tbl:reg-layout} と同じで、赤、緑、青、灰色です。どのパレットのエントリを使っているかに注目してください。マップを埋めるときにパレットの入れ替えを使えるように、色は別々のパレットバンクに入れてあります。そういえば……

マップ自体の読み込み（ポイント (3)）は、二重ループで行います。外側のループで、スクリーンエントリのパレットバンクを設定します。内側のループで、パレットを入れ替えたタイル 0 で 1024 個の SE を埋めます。さて、もし大きなマップが平らな並び方をしていたら、結果は 4 色の帯の大きなマップになるはずです。でも実際には、帯ではなく*ブロック*が見えます。これは、{@tbl:reg-layout} のとおり、通常のマップが本当にスクリーンブロックに分かれていることの証拠です。そう、面倒ですが、そういうものなのです。

マップを作る部分はここまでで、次は `main()` の中のメインループです。キー（ポイント (4)）でマップをスクロールできます。右ボタンは *x* の正の変化に結び付いていますが、マップ自体は実際には*左に*スクロールします！ こう言うと直感に反するように思えるかもしれませんが、デモを見れば、実際には理にかなっていることが分かります。仮のプレイヤーのスプライトの視点で考えてみてください。スプライトが世界の中を動くとき、スプライトが画面から出ないように背景を更新する必要があります。そのためには、背景の動きはスプライトの動きと逆向きでなければなりません。たとえば、スプライトが*右に*動いたら、それを打ち消すために背景を*左に*動かす必要があります。

最後に、話しておくべきことがもう 1 つあります。マップの中央に表示されるバツ印です。スクロールしながらこれを表示するために、いくつかの変数と `se_index()` 関数を使って、画面の中央にあるスクリーンエントリを追いかけています。変数 `tx` と `ty` は画面の中央のタイルの座標で、背景のピクセルの座標をシフトしてマスクすることで求めています。これらを `se_index()` に渡すと、スクリーンベースブロックからのスクリーンエントリのオフセットが得られます。これが前のオフセットと違えば、前のオフセットを窓ガラスに描き直し、新しいオフセットをバツ印にします。こうすると、バツ印はまるでスプライトのように、マップの上を動いているように見えます。これは実は `se_index()` のテストとして作ったものです。関数に欠陥があれば、バツ印はどこかで消えてしまうはずです。でも消えません。やったね <kbd>^_^</kbd>

### キャラブロックのデモ {#ssec-demo-cbb}

3 つ目のデモ *cbb_demo* は、キャラブロックの細かい点と、4bpp と 8bpp のタイルの違いを扱います。対象の背景は BG 0 と BG 1 です。どちらも 32×32t の背景ですが、BG 0 は 4bpp のタイルと CBB 0 を使い、BG 2 は 8bpp のタイルと CBB 2 を使います。スクリーンブロックの正確な位置と中身は重要ではありません。大事なのは、6 つのキャラブロックすべての先頭にタイルを読み込んで、どうなるかを見ることです。

<div id="cd-cbb-demo">

```c
#include <toolbox.h>
#include "cbb_ids.h"

#define CBB_4 0
#define SBB_4 2

#define CBB_8 2
#define SBB_8 4

void load_tiles()
{
    int ii;
    TILE *tl= (TILE*)ids4Tiles;
    TILE8 *tl8= (TILE8*)ids8Tiles;

    // Loading tiles. don't get freaked out on how it looks
    // 4-bit tiles to blocks 0 and 1
    tile_mem[0][1]= tl[1];      tile_mem[0][2]= tl[2];
    tile_mem[1][0]= tl[3];      tile_mem[1][1]= tl[4];
    // and the 8-bit tiles to blocks 2 though 5
    tile8_mem[2][1]= tl8[1];    tile8_mem[2][2]= tl8[2];
    tile8_mem[3][0]= tl8[3];    tile8_mem[3][1]= tl8[4];
    tile8_mem[4][0]= tl8[5];    tile8_mem[4][1]= tl8[6];
    tile8_mem[5][0]= tl8[7];    tile8_mem[5][1]= tl8[8];

    // And let's not forget the palette (yes, obj pal too)
    u16 *src= (u16*)ids4Pal;
    for(ii=0; ii<16; ii++)
        pal_bg_mem[ii]= pal_obj_mem[ii]= *src++;
}

void init_maps()
{
    // se4 and se8 map coords: (0,2) and (0,8)
    SB_ENTRY *se4= &se_mem[SBB_4][2*32], *se8= &se_mem[SBB_8][8*32];
    // show first tiles of char-blocks available to bg0
    // tiles 1, 2 of char-block CBB_4
    se4[0x01]= 0x0001;      se4[0x02]= 0x0002;

    // tiles 0, 1 of char-block CBB_4+1
    se4[0x20]= 0x0200;      se4[0x21]= 0x0201;

    // show first tiles of char-blocks available to bg1
    // tiles 1, 2 of char-block CBB_8 (== 2)
    se8[0x01]= 0x0001;      se8[0x02]= 0x0002;

    // tiles 1, 2 of char-block CBB_8+1
    se8[0x20]= 0x0100;      se8[0x21]= 0x0101;

    // tiles 1, 2 of char-block CBB_8+2 (== CBB_OBJ_LO)
    se8[0x40]= 0x0200;      se8[0x41]= 0x0201;

    // tiles 1, 2 of char-block CBB_8+3 (== CBB_OBJ_HI)
    se8[0x60]= 0x0300;      se8[0x61]= 0x0301;
}

int main()
{
    load_tiles();
    init_maps();

    // init backgrounds
    REG_BG0CNT= BG_CBB(CBB_4) | BG_SBB(SBB_4) | BG_4BPP;
    REG_BG1CNT= BG_CBB(CBB_8) | BG_SBB(SBB_8) |  BG_8BPP;
    // enable backgrounds
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0 | DCNT_BG1 | DCNT_OBJ;

    while(1);

    return 0;
}
```
</div>

タイルセットは *cbb_ids.c* にあります。各タイルには 2 つの数字が書かれています。1 つはそのタイルを置くキャラブロックの番号、もう 1 つはそのブロックの中でのタイルのインデックスです。たとえば、キャラブロック 0 のタイル 1 に置きたいタイルには「01」、CBB 1 のタイル 0 には「10」、CBB 1 のタイル 1 には「11」と書かれている、という具合です。全部で 12 個のタイルがあり、BG 0 用の s タイルが 4 個、BG 1 用の d タイルが 8 個です。

さて、タイルの組が 6 つあり、それを 6 つのキャラブロックそれぞれの先頭のタイルに置くつもりです（ただし CBB 0 と 2 は除きます。そこではタイル 0 が背景のデフォルトのタイルとして使われるので、空のままにしておきたいのです）。そう、6 つです。スプライトのキャラブロックにも読み込んでいます。これを手でやることもできます。アドレスを全部手で計算して（CBB 0 のタイル 1 なら `0600:0020`、など）、間違えないことを祈り、あとでデモを見直したときに自分が何をしていたか思い出せることを祈るのです。あるいは、私の `tile_mem` と `tile8_mem` のメモリマップの行列を使えば、アドレスを素早く、何の手間もなく求められます。さらにいいことに、C では構造体の代入ができるので、個々のタイルを単純な代入で読み込めます！ `load_tiles()` でしているのは、まさにそれです。元のタイルは、4bpp と 8bpp のタイル用に、それぞれ `TILE` と `TILE8` の配列にキャストしています。そうすれば、タイルの読み込みは本当にとても簡単です。

マップ自体は `init_maps()` で作っています。このデモで関心があるのは、どのキャラブロックがどう使われるかを示すことだけなので、マップの細部はあまり重要ではありません。マップには、`load_tiles()` で読み込んだタイルを表示できることだけを求めています。ここで作っている 2 つのポインタ、`se4` と `se8` は、それぞれ BG 0 と BG 1 が使うスクリーンブロックの中のスクリーンエントリを指しています。s タイルを使う BG 0 のマップは、1 と 512 のオフセットを使います。8bpp のタイルを使う BG 1 のエントリは、1 と 256 のオフセットを持っています。前に言った、ビット深度ごとのタイルのインデックスについての話が正しければ、読み込んだタイル全部の中身が見えるはずです。そして、デモの結果（{@fig:cbb-demo}）を見ると、私の計算は正しかったようです。スプライトが常に 32 バイト単位で数えるのとは対照的に、背景のタイルのインデックスは、背景に割り当てたビット深度に従います。

ただし、気になる点が 1 つあります。実機では、実際にオブジェクト VRAM（ブロック 4 と 5）にあるタイルは見えません。アドレスからすると背景にスプライトのブロックも使えそうに思えますが、GBA の実際の配線では、それが禁じられているようです。だからこそ、実機で試すことが大事なのです。エミュレータはいつも完璧とは限りません。でも、実機で試せないなら、複数のエミュレータで試してください。振る舞いが違ったら、それを生み出したコードを疑ってかかりましょう。

<div class="lblock">
<table id="fig:cbb-demo">
<tr valign="top">
<td>
  <div class="cpt" style="width:240px">
  <img src="./img/demo/cbb_demo_vba.png" alt="cbb_demo on VBA"><br>
  <b>{*@fig:cbb-demo}a</b>: 古いエミュレータ（VBA や Boycott Adv など）での
  <i>cbb_demo</i>。
  </div>
<td>
  <div class="cpt" style="width:240px">
  <img src="./img/demo/cbb_demo_hw.png" alt="cbb_demo on hardware"><br>
  <b>{*@fig:cbb-demo}b</b>: 実機での
    <i>cbb_demo</i>。違いを探してみてください！
  </div>
</table>
</div>

### おまけのデモ: テキスト背景の「テキスト」と libtonc の紹介 {#ssec-demo-hello}

わーい、おまけのデモです！ この例にはいくつかの目的があります。1 つ目は、GBA での暮らしを少し楽にしてくれるコードのライブラリ、libtonc を紹介することです。これまでのデモでは、便利なマクロや関数を *toolbox.h/c* に入れてきました。とても小さなプロジェクトならそれで構いませんが、コードが増えるにつれて、全部を保守するのがとても難しくなります。共通の機能は、プロジェクト間で共有できる[ライブラリ](https://en.wikipedia.org/wiki/Library_(computing))に入れておくほうがいいです。

2 つ目の理由は、文字を出力する方法を示すことです。これは明らかに、持っておくべき大事な能力です。tonclib には、文字の描画のためのたくさんのオプションがあり、ここで説明しきれないほどですが、インターフェースはかなり簡単です。詳しくは、[Tonc Text Engine の章](tte.html)を見てください。

とにかく、例は次のとおりです。

<div id="cd-hello">

```c
#include <stdio.h>
#include <tonc.h>

int main()
{
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    // Init BG 0 for text on screen entries.
    tte_init_se_default(0, BG_CBB(0)|BG_SBB(31));

    tte_write("#{P:72,64}");        // Goto (72, 64).
    tte_write("Hello World!");      // Print "Hello world!"

    while(1);

    return 0;
}
```
</div>

<div class="lblock">
<table id="fig:hello">
<tbody valign="top">
<tr>
  <td>
    <div class="cpt" style="width:112px;">
      <img src="./img/demo/hello.png" alt=""><br>
      <b>{*@fig:hello}a</b>: <i>hello</i> のデモ。
    </div>
  <td>
    <div class="cpt" style="width:256px;">
      <img src="./img/demo/hello_tiles.png" alt=""><br>
      <b>{*@fig:hello}b</b>: <i>hello</i> のデモのタイルセット。
    </div>
</tbody>
</table>
</div>

そう、これはまさに「hello world」のデモで、C/C++ の入門チュートリアルのほぼすべてが出発点にしているものです。でも、そうしたチュートリアルはたいてい PC のプラットフォーム向けで、`printf()` や `cout` のような、コンソールの機能を標準で持っています。GBA にはそれがありません。（いや、「ありませんでした」と言うべきでしょう。今はそれを使う方法があります。詳しくは [tte:conio](tte.html#ssec-misc-conio) を見てください。）

Tonc の文字のサポートは、`tte_` の関数を通して行います。ここでは、`tte_init_se_default()` が、タイルマップで文字を表示するように背景 0 を準備します。また、デフォルトの 8×8 のフォントをキャラブロック 0 に読み込みます（{@fig:hello}b を参照）。そのあとは、`tte_write` で文字を書き込めます。`#{P:x,y}` という並びは、TTE がカーソルの位置を決めるのに使う書式のコマンドです。こうしたコマンドはいくつかあり、そのいくつかはあとの章でも出てきます。

ここから先は、値などを表示するために、例の中で libtonc の文字の機能を遠慮なく使います。たいていは説明なしで使います。デモの本題ではないからです。繰り返しますが、中の仕組みを知りたければ、[TTE の章](tte.html)を見てください。

#### コードのライブラリを作って使う

関数そのものを使うのはとても簡単ですが、関数は複数のファイルに散らばっていて、さらに多くのファイルを参照しています。そのため、プロジェクトをコンパイルするためにソースの一覧にどのファイルを足せばいいのかを探すのが面倒です。もちろん全部を足すこともできますが、それも気の進むことではありません。一番いい解決策は、便利なコードを前もってライブラリにコンパイルしておくことです。

ライブラリは、本質的にはオブジェクトファイルの寄せ集めです。オブジェクトを直接実行ファイルにリンクする代わりに、arm-none-eabi-ar で<dfn>アーカイブ</dfn>します。コマンドは、リンクの段階のものとも似ています。オブジェクト foo.o、bar.o、baz.o からライブラリ libfoo.a を作るには、次のようにします。

```makefile
# archive rule
libfoo : foo.o bar.o baz.o
    arm-none-eabi-ar -crs libfoo.a foo.o bar.o baz.o
# shorthand rule: $(AR) rcs $@ $^
```

3 つのフラグは、それぞれアーカイブを作る（**c**reate）、メンバーを置き換える（**r**eplace）、シンボルテーブルを作る（**s**ymbol table）を表します。これらとほかのアーカイブのフラグについて詳しくは、[binutils](https://sourceware.org/binutils/) のツール群の一部であるマニュアルを見てください。フラグのあとにライブラリの名前が続き、そのあとに全部のオブジェクト（アーカイブしたい「メンバー」）が続きます。

ライブラリを使うには、それを実行ファイルにリンクする必要があります。ここで関係するリンカのフラグは 2 つ、`-L` と `-l` です。大文字と小文字の「L」です。前者の `-L` は、ライブラリのパスを足します。小文字の `-l` は実際のライブラリを足しますが、ここにひとひねりあります。ライブラリの名前の根っこの部分だけが必要なのです。たとえば、ライブラリ *libfoo.a* をリンクするには、`-lfoo` を使います。接頭辞の *lib* と拡張子の *.a* は、リンカが補ってくれます。

```makefile
# using libfoo (assume it's in ../lib)
$(PROJ).elf : $(OBJS)
    $(LD) $^ $(LDFLAGS) -L../lib -lfoo -o $@
```

もちろん、たくさんのものを詰め込めば、こうしたアーカイブはかなり大きくなります。ライブラリをプロジェクトに足すと、その全部がリンクされるのかと思うかもしれません。答えはノーで、全部はリンクされません。リンカは賢いので、実際に参照している関数を含むファイルだけを使います。たとえばこのデモでは、さまざまな文字の関数を使っていますが、[アフィン](affine.html)の関数や表は使っていないので、それらは除かれます。除かれるのは、*関数*単位ではなく*ファイル*単位であることに注意してください。ライブラリにファイルが 1 つしかなければ（または全部を `#include` していれば、同じことですが）、すべてがリンクされます。

あとのいくつかのデモでは、libtonc を使うつもりです。特に、メモリマップ、文字、コピーのルーチンがよく出てくるでしょう。それらがデモの中で何をしているかは気にせず、本題の中身そのものに集中してください。libtonc のドキュメントは、*libtonc* のフォルダ（`tonc/code/libtonc`）と、[tonclib のウェブサイト](https://www.coranac.com/man/libtonc/)にあります。

:::tip もっとよいコピーと塗りつぶしのルーチン: memcpy16/32 と memset16/32

文字のルーチンのために libtonc をライブラリとして使うようになったので、コピーと塗りつぶしのルーチンにも使ってしまいましょう。コピーの名前は `memcpy16()` と `memcpy32()`、塗りつぶしは `memset16()` と `memset32()` です。16 と 32 は、それぞれ得意なデータ型、つまりハーフワードとワードを表しています。引数は従来の `memcpy()` や `memset()` と似ていますが、大きさがバイト数ではなく、コピーする要素の数である点が違います。

```c
void memset16(void *dest, u16 hw, uint hwcount);
void memcpy16(void *dest, const void *src, uint hwcount);

void memset32(void *dest, u32 wd, uint wcount) IWRAM_CODE;
void memcpy32(void *dest, const void *src, uint wcount) IWRAM_CODE;
```

これらのルーチンは最適化されたアセンブリなので、[速い](text.html#ssec-demo-se2)です。また、[DMA のルーチン](dma.html#sec-func)や [BIOS のルーチン](swi.html)の `CpuFastSet()` より安全です。基本的に、強くおすすめしますし、私も使えるところでは使います。

:::

:::warning リンカのオプション: オブジェクトファイルはライブラリより前に

ほとんどの場合、オプションとファイルの順番は自由に変えられますが、リンカの場合は、プロジェクトのオブジェクトファイルを、リンクするライブラリより*前に*書くことが大事です。そうしないと、リンクに失敗します。これが標準の振る舞いなのか、リンカの仕組みの見落としなのかは分かりませんが、ここで問題が起きうることを意識しておいてください。

:::

## まとめ {#sec-conc}

タイルマップは、ほとんどの種類の GBA のゲームに欠かせません。[正確にこなす必要のある手順](#ssec-demo-brin)が多いので、ビットマップモードやスプライトより習得が少し難しいです。それにもちろん、マップを作ってくれたエディタが、本当に期待どおりのデータを出しているかを確かめる必要もあります。デモで少し遊んでみてください。実行して、コードを変えて、どうなるか見てみましょう。たとえば、brin_demo にスクロールのコードを足して、マップ全体を見られるようにしてみてもいいでしょう。スクリーンブロックを変え、キャラブロックを変え、ビット深度を変え、*わざと*めちゃくちゃにして、何がうまくいかなくなりうるのかを見てみてください。そうすれば、自分のマップを作るときに備えができます。十分に自信がついたら、そこで初めて自分のマップを作り始めてください。退屈なやり方なのは分かっていますが、長い目で見れば得をします。
