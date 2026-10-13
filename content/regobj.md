# 8. 通常のスプライト

<!-- toc -->

## スプライトの紹介 {#sec-intro}

<div class="cpt_fr" style="width:64px">
<img src="./img/metr/std.png" id="fig:metr" 
  alt="a metroid.">
<b>{*@fig:metr}</b>. メトロイド。がおー。
</div>

ウェブスター辞典によると、スプライトとは「妖精、エルフ、ゴブリンのような、想像上の存在や精霊」です。なるほど、はっきりしてよかった。でもゲームの場合、スプライトと言えば普通は「背景から自由に動かせる、［小さな］アニメーションするオブジェクト」（PERN）のことです。代表例はゲームのキャラクターですが、スコアや体力ゲージのような状態を表すものも、よくスプライトで作られます。右の {*@fig:metr} は、みんな大好き、吸血するクラゲのメトロイドのスプライトです。このスプライトは、この章の最後のデモで使います。

スプライトはビットマップの背景より少し扱いが難しいですが、それほどではありません。自分のしていることに、もう少し注意を払う必要があるだけです。まず、グラフィックは 8×8 のタイルにまとめる必要があります。グラフィックの変換ツールがそれをできることを確かめてください。ディスプレイ制御でスプライトを有効にする、グラフィックとパレットを読み込む、といった分かりやすい作業のほかに、OAM にスプライトの属性を正しく設定する必要もあります。どれか 1 つでも抜けると、何も表示されません。この章では、こうしたことなどを扱います。

:::tip スプライトに欠かせない手順


スプライトを表示させるために、正しくやらなければならないことが 3 つあります。

-   グラフィックとパレットを、オブジェクトの VRAM とパレットに読み込む。
-   OAM の属性を設定して、適切なタイルを使い、正しい大きさにする。
-   `REG_DISPCNT` でオブジェクトを有効にし、そこでマッピングモードも設定する。

:::

:::tip スプライトはオブジェクトではない

あるいは、そのようなものです。変に聞こえるのは分かっていますが、考えれば考えるほど、スプライトとオブジェクトは同じものとみなすべきではないと気づきます。「オブジェクト」という言葉は、OAM で制御するハードウェアの機能のことです。今の私の考えでは、「スプライト」はもっと概念的な言葉で、プレイヤーのキャラクター、モンスター、弾など、登場するもののためにとっておくべきです。それらは、実際には複数のハードウェアのオブジェクトで組み立てられていることもあれば、背景を使っていることさえあります。

こう考えることもできます。オブジェクトはゲーム機そのものに結び付いた*システム*の存在で、スプライトはゲームの世界に住む*ゲーム*の存在です。違いは微妙かもしれませんが、大事な違いです。

これは私の意見にすぎず、どこまで正しいかは分かりません。Tonc では今も 2 つの言葉を行ったり来たりしていますが、今さらどうしようもないからです。私の落ち度です。この件についてほかの人の意見もぜひ聞きたいので、言いたいことがあれば遠慮なくどうぞ。

:::

## スプライトの画像データとマッピングモード {#sec-tiles}

[スプライトと背景の概要](objbg.html)で言ったとおり、スプライトは、タイルと呼ばれる 8×8 ピクセルの小さなビットマップをいくつも組み合わせてできています。タイルには 2 種類あり、4bpp（s タイル、32 バイト）と 8bpp（d タイル、64 バイト）です。スプライトに使えるタイルは、<dfn>オブジェクト VRAM</dfn>、略して <dfn>OVRAM</dfn> に入っています。OVRAM の大きさは 32 KiB で、`tile_mem` の最後の 2 つのキャラブロックに対応しています。これらは、下のスプライトのブロック（ブロック 4、`0601:0000h` から）と上のスプライトのブロック（ブロック 5、`0601:4000h`）とも呼ばれます。数え始めるのは常に下のスプライトのブロックからで、*常に* 32 バイト単位で数えます。つまり、ビット深度に関係なく、スプライトのタイル #1 は `0601:0020h` にあります（表 8.1 を参照）。キャラブロック 1 つあたり 4000h バイトなので、ちょっと計算すれば、各キャラブロックにはタイルが 512 個あり、全部で 1024 個の範囲になることが分かります。ただし、[ビットマップ](bitmaps.html)モードは下のスプライトのブロックにまで広がっているので、モード 3〜5 では上のスプライトのブロック（タイル 512〜1023 が入っています）しか使えません。

タイルのアドレスを計算するのは面倒に思えるかもしれません。手でやるなら、実際に面倒です。だからこそ私は、[概要](objbg.html#ssec-img-cbb)で説明したように、VRAM 全体を `tile_mem` という、キャラブロックとタイルの行列に対応づけたのです。OVRAM のタイル #123 が必要ですか？ それなら `tile_mem[4][123]` です。そのアドレスが必要ですか？ アドレス演算子を使って `&tile_mem[4][123]` です。速くて、簡単で、安全です。

また、スプライトには専用のパレットがあり、`0500:0200h`（背景のパレットのすぐあと）から始まることを忘れないでください。タイルを正しく読み込んだはずなのに何も表示されないなら、違うパレットを埋めているのかもしれません。

<div class="lblock">
<table id="tbl:obj-tids" class="table-data">
<caption align="bottom">
  <b>{*@tbl:obj-tids}</b>: スプライトのタイルの数え方。
  常に 32 バイトごと。（8bpp のタイルに奇数の番号も使えますが、
  それに合わせて VRAM を埋めるようにしてください。）
</caption>
<tr>
  <th>メモリ 0601:<th>0000<th>0020 <th>0040<th>0060 <th>0080<th>0100 <th>...
<tr>
  <th>4bpp のタイル <td>0      <td>1  <td>2    <td>3  <td>4    <td>5 <td>&nbsp;
<tr>
  <th>8bpp のタイル <td colspan=2>0   <td colspan=2>2 <td colspan=2>4<td>&nbsp;
</table>
</div>

:::warning ビットマップモードとオブジェクト VRAM

モード 3〜5 では、スプライトに使えるのは上のスプライトのブロックだけです。ただし、インデックスは下のブロックから数えるので、タイルの範囲は 512〜1023 です。

:::

### スプライトのマッピングモード {#ssec-map}

スプライトは 1 つのタイルに限られません。実際、ほとんどのスプライトはもっと大きいです（GBA のスプライトで使える大きさの一覧は {*@tbl:obj-size} を参照）。大きなスプライトは単純に複数のタイルを使いますが、これが問題になることがあります。背景では、タイルマップで各タイルを明示的に選びます。スプライトの場合は、1D マッピングと 2D マッピングの 2 つの選択肢があります。デフォルトは 2D マッピングで、`REG_DISPCNT{6}` をセットすると 1D マッピングに切り替わります。

これらはどう働くのでしょうか？ {@fig:obj-map}a のスプライトの例を考えてみてください。{@fig:metr} のメトロイドを、タイルに分けて示したものです。2D マッピングでは、スプライトのキャラブロックを 256×256 ピクセルの 1 つの大きなビットマップとして解釈し、スプライトをそのビットマップから切り出した四角形として扱います（もちろん、まだタイルに分かれています）。この場合、スプライトのタイルの各行は、32 タイルずつずれた位置にあります。これを {@fig:obj-map}b に示します。一方、キャラブロックを 1 つの大きなタイルの配列とみなし、各スプライトのタイルを連続させることもできます。これを {@fig:obj-map}c に示します。{@fig:obj-map}a の数字は、1D マッピングと 2D マッピングの違いを示しています。タイル 0 から始めるとすると、赤の数字が 2D マッピング、シアンの数字が 1D マッピングに従っています。

GBA のプログラミングの観点からは、スプライトを入れるときにタイルの各行のずれを気にしなくて済むので、1D マッピングのほうが使いやすいです。でも、実際にスプライトを*作る*のは、2D モードのほうが簡単です。だって、ビットマップをタイル 1 つずつ*本気で*編集したいですか？ そうでしょう。もちろん、スプライトを 2D から 1D マッピングに変換するのは、出力ツールの仕事であるべきです。[Usenti](https://www.coranac.com/projects/#usenti) でもできます。

<div class="lblock">
<table id="fig:obj-map"
  cellpadding=4 cellspacing=0>
<tr valign="top">
<td>
  <div class="cpt" style="width:200px">
  <img src="./img/metr/tile_1d2d.png" alt="a metroid divided into tiles">
  <b>{*@fig:obj-map}a</b>: {*@fig:metr} を拡大して
  タイルに分けたもの。色付きの数字は
    マッピングモードを表す。赤が 2D、シアンが 1D。
  </div>
<td>
  <div class="cpt" style="width:297px">
  <img src="./img/metr/tile_2d.png" alt="2d mapping">
  <b>{*@fig:obj-map}b</b>: 2D マッピングを使う場合に、
    {@fig:obj-map}a をメモリにどう入れるべきか。
	
  </div>
  <br>
  <div class="cpt" style="width:296px">
  <img src="./img/metr/tile_1d.png" alt="1d mapping">
  <b>{*@fig:obj-map}c</b>: 1D マッピングを使う場合に、
    {@fig:obj-map}a をメモリにどう入れるべきか。
	
  </div>
</table>
</div>

:::note コマンドラインでのオブジェクトのデータの変換

コマンドラインのツールの中には、オブジェクト（とタイルマップ）で使えるように、ビットマップをタイル化できるものがあります。場合によっては、複数のスプライトのフレームを含む画像を、1D のオブジェクトマッピングモード用のオブジェクトのタイルの組に変換することもできます。*foo.bmp* が、16×16 のオブジェクトを 4 つ含む 64×16 のビットマップだとすると、gfx2gba と grit を使って 8×8 の 4bpp のタイルに変換するには、次のようにします（1D マッピングのためのフラグはかっこの中です）。

```sh
# gfx2gba
# 4x 16x16@4 objects (C array; u8 foo_Bitmap[], u16 master_Palette[]; foo.raw.c, master.pal.c)
    gfx2gba -fsrc -c16 -t8 [-T32] foo.bmp
```

```sh
# grit
# 4x 16x16@4 objects (C array; u32 fooTiles[], u16 fooPal[]; foo.c, foo.h)
    grit foo.bmp -gB4 [-Mw 2 -Mh 2]
```

ここでの 1D マッピングのフラグについて、注意が 2 つあります。まず、gfx2gba は正方形のオブジェクトしかメタタイル化（-T）できません。16×8 のオブジェクトのようなものは、自分で 1D マッピングにする必要があります。次に、grit のメタタイル化のフラグ（`-Mw` と `-Mh`）は任意の値にでき、単位はピクセルではなくタイルです。

:::

:::note 大きさの単位: タイルとピクセル

ビットマップの大きさのデフォルトの単位は、もちろんピクセルです。でも、タイルのグラフィックでは、タイル、つまりピクセルの大きさを 8 で割ったものを基本の単位にするほうが便利なことがあります。特に背景ではそうです。ほとんどの場合、どちらの意味かは文脈で分かりますが、ときどき、ピクセルなら「p」、タイルなら「t」を付けて単位を示します。たとえば、64x64p のスプライトは 8×8t のスプライトと同じです。

:::

## スプライトの制御: Object Attribute Memory {#sec-oam}

ビットマップモードとは大違いで、スプライトは自分で描く必要がありません。GBA には、それをやってくれる専用のハードウェアがあります。これを使えば、プログラムでは決して達成できないほど速く、スプライトを画面に出せます。それでも限界はあります。1 本のスキャンラインに詰め込めるスプライトのピクセルの数には上限があるのです。フォーラムの話を信じるなら、約 960 です。

なので、スプライトを自分で描く必要はありません。ただし、どう表示したいかを GBA に伝える必要は*あります*。そのためのものが <dfn>Object Attribute Memory </dfn>（オブジェクト属性メモリ）――略して OAM――です。これはアドレス `0700:0000h` から始まり、大きさは 1024 バイトです。OAM には 2 種類の構造体があります。通常のスプライトの属性のための <dfn>OBJ_ATTR</dfn> 構造体と、変換のデータを含む <dfn>OBJ_AFFINE</dfn> 構造体です。これらの構造体の定義は、下のとおりです。名前はサイトによって違うことがあるので注意してください。

<div id="cd-oam-structs">

```c
typedef struct tagOBJ_ATTR
{
    u16 attr0;
    u16 attr1;
    u16 attr2;
    s16 fill;
} ALIGN4 OBJ_ATTR;

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
</div>

これらの構造体には、面白い点がいくつかあります。まず、`fill`（詰め物）のフィールドがたくさんあります。次に、{@tbl:obj-weave} のように、`OBJ_ATTR` 構造体 4 つを `OBJ_AFFINE` 構造体 1 つに重ねると、一方の詰め物が、もう一方のデータをぴったり覆っていることが分かります。逆も同じです。これは偶然ではありません。OAM は実は、`OBJ_ATTR` と `OBJ_AFFINE` を織り交ぜたものなのです。なぜ任天堂は、属性の領域と変換データの領域を 1 つずつ持つのではなく、織り交ぜる方式を使ったのでしょうか？ よい質問で、よい答えに値します。答えが見つかったらお教えします（データのアラインメントの関係だろうと推測しています）。また、`OBJ_AFFINE` の要素は*符号付き*の short であることに注意してください。私は `s16` ではなく `u16` を使ったせいで、`obj_aff` のコードでひどく痛い目にあいました。使えるのは 1024 バイトなので、`OBJ_ATTR` 構造体なら 128 個、`OBJ_AFFINE` なら 32 個分の場所があります。このファイルの残りでは、`OBJ_ATTR` だけを使う通常のスプライトを説明します。[アフィン変換の行列](affine.html)は、それにふさわしい数学的な扱いをきちんとしたいので、[アフィンスプライト](affobj.html)はあとにとっておきます。

<div class="lblock">
<table id="tbl:obj-weave"
  class="reg" border=1 frame=void cellpadding=4 cellspacing=0>
<caption align= bottom>
  <b>{*@tbl:obj-weave}</b>: 
  OBJ_ATTR と OBJ_AFFINE の構造体の、
  メモリ上での織り交ぜ方。
 </caption>
<tr align="left"><th>メモリ（u16）<th>0<th>3<th>4<th>7<th>8<th>b<th>c<th>f
<tr><th>OBJ_ATTR
  <td>0 1 2<td>&nbsp;<td>0 1 2<td>&nbsp;
  <td>0 1 2<td>&nbsp;<td>0 1 2<td>&nbsp;
<tr><th>OBJ_AFFINE
  <td>&nbsp;<td>pa<td>&nbsp;<td>pb
  <td>&nbsp;<td>pc<td>&nbsp;<td>pd
</table>
</div>

:::tip OBJ_ATTR にアラインメントを強制する

devkitARM r19 から、構造体のアラインメントに新しい規則ができました。そのため、構造体が常にワードの境界に揃うとは限らなくなりました。`OBJ_ATTR` 構造体（などの）場合、あとで出てくる `oam_update()` の中のような `struct` のコピーは、遅くなるだけでなく、実際に壊れることがあります。そのため、私は多くの構造体に `ALIGN4` でワードのアラインメントを強制します。これは `__attribute__((aligned(4)))` のマクロです。詳しくは、[データのアラインメント](bitmaps.html#ssec-data-align)の節を見てください。

:::

## オブジェクトの属性: OBJ_ATTR {#sec-oam-entry}

各スプライトの基本的な制御は、`OBJ_ATTR` 構造体で行います。大きさ、形、位置、基点のタイルなどを決める、16 ビットの属性 3 つでできています。3 つの属性をそれぞれ下で説明します。

### 属性 0 {#ssec-obj-attr0}

最初の属性は多くのことを制御しますが、一番大事なのは *y* 座標とスプライトの形の部分です。スプライトが変形できるか（アフィンスプライトか）どうか、タイルのビット深度を 4（16 色、16 サブパレット）とみなすか 8（256 色 / 1 パレット）とみなすかも大事です。

<div class="reg">
<table class="table-reg" id="tbl-oe-attr0"
  border=1 frame=void cellpadding=4 cellspacing=0>
<caption class="reg">
  OBJ_ATTR.attr0
</caption>
<tr class="bits">
	<td>F E<td>D<td>C<td>B A<td>9 8 <td>7 6 5 4 3 2 1 0
<tr class="bf">
	<td class="rclr1">Sh
	<td class="rclr2">CM
	<td class="rclr5">Mos
	<td class="rclr4">GM
	<td class="rclr3">OM
	<td class="rclr0">Y
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-7<td class="rclr0">Y
  <td><i>ATTR0_Y#</i>
  <td><b>Y 座標</b>。スプライトの上端を表す。
<tr class="bg1">	
  <td>8-9<td class="rclr3">OM
  <td>ATTR0_REG, ATTR0_AFF, ATTR0_HIDE, ATTR0_AFF_DBL.
    <i>ATTR0_MODE#</i>
  <td><b>（アフィン）オブジェクトモード</b>。スプライトを隠したり、
    アフィンモードを制御したりするのに使う。
    <ul>
      <li><b>00</b>。通常の描画。
      <li><b>01</b>。アフィンスプライト。<code>attr1{9-D}</code> で
        指定したアフィン行列を使う
      <li><b>10</b>。描画しない（スプライトを隠す）
      <li><b>11</b>。描画範囲を 2 倍にしたアフィンスプライト。詳しくは
        <a href="affobj.html">アフィンスプライト</a>を参照。
    </ul>   
<tr class="bg0">	
  <td>A-B<td class="rclr4">GM
  <td>ATTR0_BLEND, ATTR0_WIN. <i>ATTR0_GFX#</i>
  <td><b>グラフィックモード</b>。特殊効果のフラグ。
    <ul>
      <li><b>00</b>。通常の描画。
      <li><b>01</b>。半透明合成を有効にする。
        <a href="gfx.html#sec-blend">こちら</a>で扱う。
      <li><b>10</b>。オブジェクトはオブジェクトウィンドウの一部になる。スプライト
        自体は描画されず、背景やほかのスプライトのマスクとして
        働く。（と思います。まだ使ったことがありません）
      <li><b>11</b>。使用禁止。
    </ul>
<tr class="bg1">	
  <td>C<td class="rclr5">Mos
  <td>ATTR0_MOSAIC
  <td>モザイク効果を有効にする。<a href="gfx.html#sec-mos">こちら</a>で扱う。
<tr class="bg0">	
  <td>D<td class="rclr2">CM
  <td>ATTR0_4BPP, ATTR0_8BPP
  <td><b>カラーモード</b>。消えていれば 16 色（4bpp）、
    立っていれば 256 色（8bpp）。    
<tr class="bg1">	
  <td>E-F<td class="rclr1">Sh
  <td>ATTR0_SQUARE, ATTR0_WIDE, ATTR0_TALL. <i>ATTR0_SHAPE#</i>
  <td><b>スプライトの形</b>。これとスプライトの大きさ
    （<code>attr1{E-F}</code>）で、スプライトの実際の大きさが決まる。
    <a href="#tbl-obj-size">表 8.4</a> を参照。
</tbody>
</table>
</div>

属性 0 について、追加の注意が 2 つあります。まず、`attr0` には ***y*** 座標が入り、`attr1` には ***x*** 座標が入ります。なぜか私はこの 2 つをよく取り違えます。上に動くはずのスプライトが左に動いていたら、これが原因かもしれません。次に、アフィンモードとグラフィックモードは、いつもそう呼ばれているとは限りません。特に、`attr0{9}` は単に*例の*倍サイズのフラグと呼ばれますが、その役割を果たすのはビット 8 も立っている場合だけです。立っていなければ、スプライトを隠します。そのとき、スプライトは実際にオブジェクトの描画の段階から完全に外されて、ほかのスプライトのための時間が増えるのだと思いますが、100% の確信はありません。

<div class="lblock">
<table id="tbl:obj-size"
  class="reg" border=1 cellpadding=2 cellspacing=0>
<caption align="bottom">
  <b>{*@tbl:obj-size}</b>: GBA のスプライトの大きさ
</caption>
<tr><td>形\大きさ <th>00    <th>01    <th>10    <th>11
<tr><th>00<td>8×8  <td>16×16 <td>32×32 <td>64×64
<tr><th>01<td>16×8 <td>32×8 <td>32×16 <td>64×32
<tr><th>10<td>8×16 <td>8×32 <td>16×32 <td>32×64
</table></div>

### 属性 1 {#ssec-obj-attr1}

この属性の主な部分は、*x* 座標とスプライトの大きさです。ビット 9〜13 の役割は、アフィンスプライトかどうか（`attr0{8}` で決まります）によって変わります。アフィンスプライトなら、これらのビットは、32 個の `OBJ_AFFINE` のどれを使うかを指定します。そうでなければ、反転のフラグを持ちます。

<div class="reg">
<table class="table-reg" id="tbl-oe-attr1">
<caption class="reg">
  OBJ_ATTR.attr1
</caption>
<tr class="bits">
	<td>F E<td>D<td>C<td>B A 9<td>8 7 6 5 4 3 2 1 0
<tr class="bf">
	<td class="rclr1">Sz
	<td class="rclr2">VF
	<td class="rclr2">HF
	<td>-
	<td class="rclr0">X
<tr class="bf">
    <td>-
	<td colspan=3 class="rclr3">AID
    <td>-
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-8<td class="rclr0">X
  <td><i>ATTR1_X#</i>
  <td><b>X 座標</b>。スプライトの左端を表す。
<tr class="bg1">	
  <td>9-D<td class="rclr3">AID
  <td><i>ATTR1_AFF#</i>
  <td><b>アフィンのインデックス</b>。このスプライトが使う <code>OAM_AFF_ENTY</code> を
    指定する。アフィンのフラグ（<code>attr0</code>{8}）が
    立っているときに<i>だけ</i>有効。
<tr class="bg0">	
  <td>C-D<td class="rclr2">HF, VF
  <td>ATTR1_HFLIP, ATTR1_VFLIP. <i>ATTR1_FLIP#</i>
  <td><b>水平・垂直反転</b>のフラグ。アフィンのフラグ（<code>attr0</code>）が
    消えているときに<i>だけ</i>使われる。そうでなければ、
    アフィンのインデックスの一部になる。
<tr class="bg1">	
  <td>E-F<td class="rclr1">Sz
  <td><i>ATTR1_SIZE#</i>
  <td><b>スプライトの大きさ</b>。のようなもの。形のビット
    （<code>attr0</code>{E-F}）と合わせて、スプライトの実際の大きさが決まる。
    {@tbl:obj-size} を参照。
</tbody>
</table>
</div>

ここでも言っておきます。`attr0` には *y* が、`attr1` には *x* が入ります。ビット 12 と 13 は、反転のフラグかアフィンのインデックスかの、2 つの役割を持つことに注意してください。そして、アフィンスプライトでも反転できるのかと思っているなら、答えは「できる」です。行列で負の拡大率を使えばいいのです。

### 属性 2 {#ssec-obj-attr2}

この属性は、どのタイルを表示するかと、背景に対する優先度を GBA に伝えます。4bpp のスプライトなら、どのサブパレットを使うかもここで指定します。

<div class="reg">
<table class="table-reg" id="tbl-oe-attr2">
<caption class="reg">
  OBJ_ATTR.attr2
</caption>
<tr class="bits">
	<td>F E D C<td>B A<td>9 8 7 6 5 4 3 2 1 0
<tr class="bf">
	<td class="rclr1">PB
	<td class="rclr2">Pr
	<td class="rclr0">TID
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def">
<tr align="left"><th>ビット<th>名前<th>&nbsp;<th>説明
<tbody valign="top">
<tr class="bg0">	
  <td>0-9<td class="rclr0">TID
  <td>ATTR2_ID#
  <td>スプライトの基点の<b>タイルのインデックス</b>。ビットマップモードでは、
    512 以上でなければならないことに注意。
<tr class="bg1">	
  <td>A-B<td class="rclr2">Pr
  <td>ATTR2_PRIO#
  <td><b>優先度</b>。優先度の値が大きいものほど先に描かれる（そのため、
    あとのスプライトや背景に覆われることがある）。スプライトは、
    同じ優先度の背景を覆う。同じ優先度の
    スプライト同士では、番号の大きい <code>OBJ_ATTR</code> が先に描かれる。
<tr class="bg0">	
  <td>C-F<td class="rclr1">PB
  <td>ATTR2_PALBANK#
  <td>16 色モードのときに使う<b>パレットバンク</b>。
	カラーモードのフラグ（<code>attr0</code>{C}）が立っていれば効果はない。
</tbody>
</table>
</div>

### 属性 3 {#ssec-obj-attr3}

属性 3 は*ありません*。`OBJ_ATTR` 構造体には確かに 4 つ目のハーフワードが*あります*が、これは詰め物にすぎません。その詰め物のメモリは、実は `OBJ_AFFINE` のものです。`attr3` は、どんな形でも粗末に扱ってはいけません……有効なアフィンスプライトがあるならば。

### OAM のダブルバッファリング {#ssec-oam-buffer}

スプライトのデータを全部 `0700:0000h` の OAM に直接書き込むことも*できます*が、いつもそれが最善の手とは限りません。VDraw のあいだにやると、ティアリングが起きる可能性があります。さらに悪いことに、描画の途中でスプライトのタイルのインデックスを変えてしまい、上半分がアニメーションのあるフレームで、下半分が別のフレームになってしまうかもしれません。見られたものではありません。実際には、VDraw のあいだは OAM を更新*できない*（ロックされている）ので、これは心配しなくていいのですが。よく行われるのは、いつでも変更できる OAM のエントリの別のバッファ（<dfn>オブジェクトシャドウ</dfn>とも呼ばれます）を作っておき、VBlank のあいだにそれを本物の OAM にコピーすることです。私のやり方は次のとおりです。

```c
OBJ_ATTR obj_buffer[128];
OBJ_AFFINE *const obj_aff_buffer= (OBJ_AFFINE*)obj_buffer;
```

今は 128 を使っていますが、スプライトを全部使わないなら、もっと小さな数でもいいでしょう。ともかく、これで `OBJ_ATTR` と `OBJ_AFFINE` の両方のデータのダブルバッファができ、いつでも使えます。ちょうどいいときに、*本物の* OAM にコピーするのを忘れないでください。

## ビットフィールドのマクロ（OAM でも、それ以外でも） {#sec-macro}

個々のビットを立てたり消したりするのは簡単ですが、全部自分でやるのは、あまり便利でないこともあります。特に、位置やパレットバンクのような複数のビットからなるフィールドでは、きれいにやろうとすると、マスクとシフトを使った長い文になってしまいます。これを少しでもよくするために、実際のコードの量を減らせるかもしれないマクロをいくつか用意しています。ここでのマクロは基本的に 3 種類ありますが、その話に入る前に、上の属性の一覧にあった、ハッシュ付きの（*foo*‘#”）define について、もう少し説明しておく必要があります。

ハッシュは、それぞれについて、*foo* を元にした `#define` が 3 つあることを意味します。*foo*`_MASK`、*foo*`_SHIFT`、*foo*`(_n)` です。これらは、対応する種類のビットマスク、ビットシフト、そしてビットフィールドを設定するマクロを表します。

たとえば、タイルのインデックスに付いている `ATTR2_ID#` を見てみましょう。タイルのインデックスのフィールドは 10 ビットで、ビット 0 から始まります。なので、対応する define は次のとおりです。

```c
// The 'ATTR2_ID#' from the attr2 list means these 3 #defines exist
#define ATTR2_ID_MASK       0x03FF    
#define ATTR2_ID_SHIFT           0    
#define ATTR2_ID(n)        ((n)<<ATTR2_ID_SHIFT)
```

世の中のほとんどの GBA のライブラリには、名前は違っても、こうした `#define` があります。実際のマクロは範囲のチェックをしないので 100% 安全ではありませんが、短くて手軽です。さて、Tonc の本文では、レジスタの define の一覧にハッシュがあれば、その名前に対してこの 3 つの `#define` があるということです。

特定のフィールドを設定したり取り出したりするのに使える、2 つ目の組のマクロもあります。上で説明したマスクとシフトの名前を使います。マクロの見た目がひどいのは認めますが、ちゃんと意味があって、役に立つこともあると保証します。

<div id="cd-bitfield">

```c
// bit field set and get routines
#define BF_PREP(x, name)         ( ((x)<<name##_SHIFT)& name##_MASK  )
#define BF_GET(x, name)          ( ((x) & name##_MASK)>> name##_SHIFT )
#define BF_SET(y, x, name)       (y = ((y)&~name##_MASK) | BF_PREP(x,name) )

#define BF_PREP2(x, name)        ( (x) & name##_MASK )
#define BF_GET2(y, name)         ( (y) & name##_MASK )
#define BF_SET2(y, x, name)      (y = ((y)&~name##_MASK) | BF_PREP2(x, name) )
```
</div>

まあ、警告はしましたよ。ここでの `name` 引数は、さっきの *foo* です。プリプロセッサの連結演算子を使って、マスクとシフトの完全な名前を作っています。再びタイルのインデックスを例にすると、これらのマクロは次のように展開されます。

```c
// Create bitfield:
attr2 |= BF_PREP(id, ATTR2_ID);
// becomes:
attr2 |= (id<<ATTR2_ID_SHIFT) & ATTR2_ID_MASK;

// Retrieve bitfield:
id= BF_GET(attr2, ATTR2_ID);
// becomes:
id= (attr2 & ATTR2_ID_MASK)>>ATTR2_ID_SHIFT;

// Insert bitfield:
BF_SET(attr2, id, ATTR2_ID);
// becomes:
attr2= (attr2&~ATTR2_ID_MASK) | ((id<<ATTR2_ID_SHIFT) & ATTR2_ID_MASK);
```

`BF_PREP()` は、あとで挿入したり比較したりするためのビットフィールドを用意するのに使えます。`BF_GET()` は値からビットフィールドを取り出し、`BF_SET()` はほかのビットを乱さずに、変数のビットフィールドを設定します。これは基本的に、ビットフィールドの普通の働き方そのものです。ただし、本物のビットフィールドは OR などで組み合わせることができません。

名前に「2」が付いたマクロも同じように働きますが、シフトはしません。`ATTR0_WIDE` のような、すでにシフトされた `#define` があって、ほかのマクロを使えないときに役立ちます。

```c
// Insert pre-shifted bitfield:
// BF_SET2(attr0, ATTR0_WIDE, ATTR0_SHAPE);
attr0= (attr0&~ATTR0_SHAPE_MASK) | (id & ATTR0_SHAPE_MASK);
```

この 3 つのどれにも、GBA 特有のものは何もないことに注意してください。どんなプラットフォームでも使えます。

最後に、私が組み立てマクロと呼んでいるものです。これらは、HAM の道具のマクロのように、さまざまなビットのフラグを、整然と 1 つの数値にまとめます。まだそれほど使っていませんし、使うことを強制もしませんが、特に初期化のときなどには、手元にあると役立つことがあります。

<div id="cd-oe-build">

```c
// Attribute 0
#define ATTR0_BUILD(y, shape, bpp, mode, mos, bld, win)             \
(                                                                   \
    ((y)&255) | (((mode)&3)<<8) | (((bld)&1)<<10) | (((win)&1)<<11) \
    | (((mos)&1)<<12) | (((bpp)&8)<<10) | (((shape)&3)<<14)         \
)

// Attribute 1, regular sprites
#define ATTR1_BUILD_R(x, size, hflip, vflip)         \
( ((x)&511) | (((hflip)&1)<<12) | (((vflip)&1)<<13) | (((size)&3)<<14) )

// Attribute 1, affine sprites
#define ATTR1_BUILD_A(x, size, aff_id)               \
( ((x)&511) | (((aff_id)&31)<<9) | (((size)&3)<<14) )

// Attribute 2
#define ATTR2_BUILD(id, pbank, prio)                 \
( ((id)&0x3FF) | (((pbank)&15)<<12) | (((prio)&3)<<10) )
```
</div>

ビットのフラグを自分で OR でまとめる代わりに、これを使えば少し入力の手間を省けるかもしれません。引数の順番を覚えるのが面倒な人もいるでしょうし、安全のためのチェックが少し過剰かもしれません（え、そう思う？！？）。でも、渡す数値が定数なら、演算はコンパイル時に行われるので問題ありませんし、本当に役立つこともあります。そうでもないかも <kbd>:P</kbd>。前に言ったとおり、使うことは強制しません。ひどいコードだと思って（それは認めます）、自分のプログラムを汚したくないなら、それで構いません。

`bpp` を除いて、引数はすべてマクロの中でシフトされることに注意してください。つまり、一覧にある `#define` のフラグは使わず*に*、ビットではなく別々の変数だったら使うような、小さな値を渡してください。

## デモの時間 {#sec-obj-demo}

さて、いよいよこいつらを実際に使ってみましょう。下のコードは *obj_demo* の一部です。これまで見せた中で一番複雑ですが、1 歩ずつ進めば大丈夫です。基本的に、このデモは、枠付きのメトロイドのタイルをオブジェクト用の VRAM に置き、位置や反転のフラグなど、`OBJ_ATTR` のさまざまなビットをいじれるようにします。操作は次のとおりです。

- 十字キー  
  スプライトを動かします。画面の外に十分遠くまで動かすと、反対側から現れることに注意してください。
- A ボタンと B ボタン  
  それぞれ、スプライトを水平、垂直に反転します。
- セレクトボタン  
  光らせます。というか、実際にはパレットを入れ替えます。ダメージを受けたときの点滅に便利です。
- スタートボタン  
  1D と 2D のマッピングモードを切り替えます。何が起きるかは {*@fig:obj-map}b と {@fig:obj-map}c で分かるはずです。スプライトは 1D モード用なので、2D マッピングに切り替えても実はあまり見どころはないのですが、ボタンが少し余っていたので、せっかくだからと思いまして。
- L ボタンと R ボタン  
  それぞれ、最初のタイルを減らしたり増やしたりします。これもボタンが少し余っていたからです。

```c
// Excerpt from toolbox.h

void oam_init(OBJ_ATTR *obj, uint count);
void oam_copy(OBJ_ATTR *dst, const OBJ_ATTR *src, uint count);

INLINE OBJ_ATTR *obj_set_attr(OBJ_ATTR *obj, u16 a0, u16 a1, u16 a2);
INLINE void obj_set_pos(OBJ_ATTR *obj, int x, int y);
INLINE void obj_hide(OBJ_ATTR *oatr);
INLINE void obj_unhide(OBJ_ATTR *obj, u16 mode);

// === INLINES ========================================================

//! Set the attributes of an object.
INLINE OBJ_ATTR *obj_set_attr(OBJ_ATTR *obj, u16 a0, u16 a1, u16 a2)
{
    obj->attr0= a0; obj->attr1= a1; obj->attr2= a2;
    return obj;
}

//! Set the position of \a obj
INLINE void obj_set_pos(OBJ_ATTR *obj, int x, int y)
{
    BF_SET(obj->attr0, y, ATTR0_Y);
    BF_SET(obj->attr1, x, ATTR1_X);
}

//! Hide an object.
INLINE void obj_hide(OBJ_ATTR *obj)
{   BF_SET2(obj->attr0, ATTR0_HIDE, ATTR0_MODE);        }

//! Unhide an object.
INLINE void obj_unhide(OBJ_ATTR *obj, u16 mode)
{   BF_SET2(obj->attr0, mode, ATTR0_MODE);          }
```

```c
// toolbox.c

void oam_init(OBJ_ATTR *obj, uint count)
{
    u32 nn= count;
    u32 *dst= (u32*)obj;

    // Hide each object
    while(nn--)
    {
        *dst++= ATTR0_HIDE;
        *dst++= 0;
    }
    // init oam
    oam_copy(oam_mem, obj, count);
}

void oam_copy(OBJ_ATTR *dst, const OBJ_ATTR *src, uint count)
{

// NOTE: while struct-copying is the Right Thing to do here, 
//   there's a strange bug in DKP that sometimes makes it not work
//   If you see problems, just use the word-copy version.
#if 1
    while(count--)
        *dst++ = *src++;
#else
    u32 *dstw= (u32*)dst, *srcw= (u32*)src;
    while(count--)
    {
        *dstw++ = *srcw++;
        *dstw++ = *srcw++;
    }
#endif

}
```

これがデモの基本的な道具のコードで、関数にしておきたいもののほとんどが入っています。インライン関数が、前に見せたビットフィールドのマクロをうまく使っていることに注目してください。そうしていなかったら、コードはかなり長くなっていたでしょう。

もう 1 つ言っておくべきことがあります。全部を *toolbox.h* に入れていたら、ファイルはかなり大きく、700 行くらいになっていたでしょう。この先のデモでは、さらにずっと長くなります。そこで、中身を少し分け始めました。型は全部 *types.h* に、メモリマップに関するものは全部 *memmap.h* に、レジスタの define は全部 *memdef.h* に入れ、入力のインライン関数とマクロは *input.h* にあります。残りはまだ *toolbox.h* にありますが、これも最終的には分けられます。

*toolbox.c* の 2 つの関数も、もう少し説明が必要でしょう。`oam_init()` では、オブジェクトをワードのポインタにキャストして、それを使って値を設定しています。これも単純に、そのほうがずっと速いからです。本物の OAM 以外のものを初期化するのに使われるかもしれないので、念のため、初期化したバッファを OAM にコピーしています。

もう 1 つの点は、今のコンパイラ（devkitARM r19b）の最適化の、とても特定の不具合に関係しています。あとの版で直ると思いますし、ここにある基本の版で動く*はず*ですが、念のため、OAM が壊れるのを見たら `#if` の式を 0 にしてください。どうしても知りたいなら説明すると、問題は `for` ループの中での OBJ_ATTR の `struct` のコピーのようです。ええ、それくらい特定の条件です。ワードの境界に揃っていれば `struct` のコピーは合法で速いのに、GCC はループの中の 8 バイトのブロックで混乱して、構造体ごとに `memcpy()` を使ってしまうようです。それは OAM ではうまくいきません。まあ、仕方ありません。

<div id="cd-obj-demo">

```c
#include <string.h>
#include "toolbox.h"
#include "metr.h"


OBJ_ATTR obj_buffer[128];
OBJ_AFFINE *obj_aff_buffer= (OBJ_AFFINE*)obj_buffer;

void obj_test()
{
    int x= 96, y= 32;
    u32 tid= 0, pb= 0;      // (3) tile id, pal-bank
    OBJ_ATTR *metr= &obj_buffer[0];

    obj_set_attr(metr, 
        ATTR0_SQUARE,              // Square, regular sprite
        ATTR1_SIZE_64,              // 64x64p, 
        ATTR2_PALBANK(pb) | tid);   // palbank 0, tile 0

    // (4) position sprite (redundant here; the _real_ position
    // is set further down
    obj_set_pos(metr, x, y);

    while(1)
    {
        vid_vsync();
        key_poll();

        // (5) Do various interesting things
        // move left/right
        x += 2*key_tri_horz();
        // move up/down
        y += 2*key_tri_vert();

        // increment/decrement starting tile with R/L
        tid += bit_tribool(key_hit(-1), KI_R, KI_L);

        // flip
        if(key_hit(KEY_A))  // horizontally
            metr->attr1 ^= ATTR1_HFLIP;
        if(key_hit(KEY_B))  // vertically
            metr->attr1 ^= ATTR1_VFLIP;

        // make it glow (via palette swapping)
        pb= key_is_down(KEY_SELECT) ? 1 : 0;

        // toggle mapping mode
        if(key_hit(KEY_START))
            REG_DISPCNT ^= DCNT_OBJ_1D;

        // Hey look, it's one of them build macros!
        metr->attr2= ATTR2_BUILD(tid, pb, 0);
        obj_set_pos(metr, x, y);

        oam_copy(oam_mem, obj_buffer, 1);   // (6) Update OAM (only one now)
    }
}

int main()
{
    // (1) Places the tiles of a 4bpp boxed metroid sprite 
    //   into LOW obj memory (cbb == 4)
    memcpy(&tile_mem[4][0], metr_boxTiles, metr_boxTilesLen);
    memcpy(pal_obj_mem, metrPal, metrPalLen);

    // (2) Initialize all sprites
    oam_init(obj_buffer, 128);
    REG_DISPCNT= DCNT_OBJ | DCNT_OBJ_1D;

    obj_test();

    while(1);

    return 0;
}
```
</div>

### スプライトの準備 {#ssec-demo-init}

スプライトが表示される前に、やらなければならないことが 3 つあります。この順番でなくても構いません。スプライトのグラフィックを VRAM にコピーすること、そのグラフィックを使うように OAM を設定すること、そしてディスプレイ制御の `REG_DISPCNT` でスプライトを有効にすることです。

#### ディスプレイ制御

最後のものから始めると、スプライトを有効にするには `REG_DISPCNT` のビット 12 を立てます。普通は 1D マッピングも使いたいので、ビット 6 も立てます。これをしているのが、コードの **ポイント (2)** です。

#### すべてのスプライトを隠す

ここでするもう 1 つの手順は、`oam_init()` の呼び出しです。厳密には必要ありませんが、それでもやっておくのがいい考えです。`oam_init()` がするのは、すべてのスプライトを隠すことです。なぜこれがいい考えなのでしょうか？ OAM が全部ゼロでも、スプライトが見えなくなるわけでは*ない*からです。属性を確かめてみると、ゼロの状態では、全部が 8×8 ピクセルのスプライトで、グラフィックにタイル 0 を使い、(0,0) にあることになります。最初のタイルが空でなければ、左上の隅にそのタイルが 128 個並んだ状態で始まり、かなり奇妙に見えます。なので、まずは全部が見えないようにしておきましょう。デモには `obj_hide()` と `obj_unhide()` の関数も付いていますが、ここでは使っていません。

#### スプライトのグラフィックを読み込む

最初にやること（**ポイント (1)**）は、スプライトのグラフィックをオブジェクト VRAM に入れることです。もう何度か言ったとおり、このグラフィックは、平らなビットマップではなく、8×8 ピクセルのタイルとして入れる必要があります。たとえば、ここでのスプライトの大きさは 64×64p なので、入れるために、まず 8×8 の別々のタイルに変換する必要がありました。これをし*ないと*、スプライトは本当に奇妙な見た目になります。

これらのタイルを正確にどこに置くかは、実はそれほど重要ではありません（もちろん、マッピングモードやタイルのアラインメントのような当然のことは別です）。オブジェクト VRAM はテクスチャの置き場として働き、画面とは直接何の関係もありません。使えるようにしておきたいタイルをそこに入れておき、OAM の属性を操作することで、どのタイルをどこに使いたいかをシステムに伝えるのです。スプライト 0 がタイル 42 から始まってもいけない理由はありませんし、複数のスプライトが同じタイルを使ってもいけない理由もありません。だからこそ、オブジェクト VRAM に使われることのある `OAMData` という名前は、ひどい誤称なのです。オブジェクト VRAM は OAM とは何の関係もありません。*まったく*です！ ヘッダーで <kbd>0601:0000</kbd> や、まして <kbd>0601:4000</kbd> にこの名前を使っているなら、変えてください。お願いです。それから、ビットマップモードではタイル 0〜511 を使えないので、どこに置くかに気をつけてください。

前に言ったとおり、スプライトの読み込みは、コードの **ポイント (1)** で行います。[概要](objbg.html#ssec-img-cbb)を注意して読んでいれば、`tile_mem[][]` は、キャラブロックと 4 ビットのタイルに対応した 2 次元の配列だったことを覚えているでしょう。オブジェクト VRAM がキャラブロック 4 と 5 であることも覚えているはずなので、`&tile_mem[4][0]` はオブジェクト VRAM の最初のタイルを指します。つまり、私は枠付きのメトロイドを、オブジェクト VRAM の最初の 64 タイルに読み込んでいます。

そのパレットも、スプライトのパレットに読み込んでいます。*スプライト*のパレット（<kbd>0500:0200</kbd>）であって、背景のパレットではありません。違う場所に読み込むと、何も見えません。

:::tip タイルのアドレスを求める

タイルのコピー先のアドレスを求めるには、`tile_mem` かマクロを使ってください。手で計算するより、ずっと読みやすく、保守しやすいです。コードに VRAM のアドレスを決め打ちで書くことは、決してしないでください。

:::

:::danger OAMData

ほかのサイトのヘッダーには、`#define OAMData` を VRAM の一部として定義しているものがあります。そうではありません。名前を変えてください。

:::

#### 属性を設定する

最後に、`OBJ_ATTR` を 1 つ設定して、実際にメトロイドのタイルを使うようにします。これをしているのが **ポイント (3)** で、インライン関数の `obj_set_attr()` を使っています。ちなみに、この関数は最初の引数の属性に 3 回代入するだけで、特別なことは何もしていません。3 つの別々の文を書く代わりにこうすると、入力の手間が省けるだけです。この呼び出しで、このスプライトは 64×64 ピクセル（8×8 タイル）のスプライトで、最初のタイルは `tid`、つまり 0 だと伝えています。つまり、タイル 0 から始まる 64 個のタイルを使います。

設定しているスプライトは、実際には本物の OAM ではなく、OAM のバッファの一部であることに注意してください。つまり、そこで属性を設定しても、まだ何も起きません。スプライトを確定させるには、*本物の* OAM を更新する必要があります。それをしているのが `oam_copy()` の呼び出し（**ポイント (6)**）です。この関数は、何個のスプライトを更新するかと、どのスプライトから始めるかを表す、インデックスと個数の 2 つの引数を取ります。私は `obj_copy()` も用意していて、こちらは属性 0、1、2 だけをコピーし、3 はコピー*しません*！ これは、アフィンスプライトを使い始めたときに必要になります。そうしないと、正しくコピーされないことがあるからです。

ここまでの手順で、メトロイドのスプライトを画面に出すには十分です。もちろん、話はそこで終わりではありません。スプライトでできることを、いくつか紹介します。

### スプライトの位置を決める {#ssec-demo-pos}

普通は、まず画面上の、あるいは画面の外の、ある位置にスプライトを置くことから始めます。そのためには、属性 0 と 1 にある *y* と *x* の位置のビットを、それぞれ更新する必要があります。私がよくしてしまうらしい間違いは、本当は逆なのに、*x* を attr0 に、*y* を attr1 に入れてしまうことです。スプライトの動きがおかしいなら、これが原因かもしれません。

これらの座標は、スプライトの**左上**を表すことに注意してください。また、座標のビット数から、*x* の値は 512 通り、*y* の値は 256 通りあることになります。座標の範囲は折り返すので、これらは範囲 *x* ∈ \[-256, 255\]、*y* ∈ \[-128, 127\] の符号付きの整数だと言うこともできます。そう、それだと *y* の最大値が画面の高さより小さくなってしまいますが、折り返しのおかげで、全部うまくいきます。まあ、[ほぼ](affobj.html#ssec-wrap)ですが。ともかく、整数が 2 の補数で表されるおかげで、*x* と *y* の値をそれぞれ `0x01FF` と `0x00FF` でマスクするだけで、正しい 9 ビットと 8 ビットの符号付きの値になります。これは手でやることもできますし、**ポイント (4)** で使っている `obj_set_pos()` 関数を使うこともできます。

属性の下位のビットを消してから、*x* と *y* を直接 OR で入れるコードを見かけるかもしれません。これはいい考えではありません。負の値は、実際にはデータ型の範囲の上半分で表されるからです。たとえば −1 は、全部のビットが立った状態（0xFFFFFFFF）です。上位のビットをマスクで消さないと、負の値が属性のほかのビットを上書きしてしまい、まずいことになります。

:::warning 座標をマスクしてください

スプライトの位置を決める関数を作る、またはほかの人のものを使うなら、*x* と *y* を属性に入れる前に、ビットをマスクで切り落としているかを**必ず確かめて**ください。そうしないと、負の値が属性全体を上書きしてしまいます。

これは悪い例です

```c
obj->attr0= (obj->attr0 &~ 0x00FF) | (y);
```

これはよい例です:

```c
obj->attr0= (obj->attr0 &~ 0x00FF) | (y & 0x00FF);
```
:::

#### 位置の変数と tribool の使い方

スプライトの位置を保存するのに `OBJ_ATTR` を使う代わりに、別の変数、ここでは `x` と `y` に保存しておくほうがいいです。そうすれば、座標のフィールドを毎回マスクする必要がなくなりますが、もっと大事なのは、位置が画面の大きさを超えられることです。ほとんどのゲームの世界は 1 画面に収まらないので、これは大事な点です。そして、ちょうどいいときに、これらを `oam_set_pos()` に渡してスプライトを更新します。

また、位置の更新に、私の [tribool のキーの関数](keys.html#ssec-adv-tri)を使っていることにも注目してください。入力の処理は、よく「キー X が押されたら増やし、Y の反対のキーが押されたら減らす」という形になります。tribool の関数を使うと、そういうコードが 4 行から 1 行になり、（最初のハードルを越えれば）コードが読みやすくなります。たとえば、`key_tri_horz()` は、「右」が押されていれば +1、「左」が押されていれば −1、どちらも押されていないか両方押されていれば 0 を返します。`key_tri_vert()` は縦の動きについて同じことをし、`bit_tribool()` 関数を使った行は、`key_hit()` と R、L を使ってタイルのインデックスを増やしたり減らしたりする変種です。

### ほかの属性 {#ssec-demo-attr}

スプライトの座標は、スプライトがすでに表示されているあいだにも、OAM の特定のビットで制御できる、たくさんのスプライトの属性のうちの 2 つにすぎません。分かりやすいものとしては、反転や鏡像化があり、ここでは A と B でできます。また、4bpp のスプライトを使っているなら、パレットを入れ替えて、全部の色を変えられます。デモでセレクトを押すと、パレットバンク 0 から 1 に切り替わります。パレットバンク 1 には、たまたま灰色から白へのグラデーションが入っています。これらのパレットバンクを素早く切り替えれば、スプライトを点滅させられます。ほかにも、スプライトを描く優先度を変えたり、半透明合成を切り替えたりもできますが、ここではやっていません。

さて、これらはスプライトの全体の画像を本当に変えるものではありません。でも、それを変えることも*できる*と気づいておくべきです。前にも言ったとおり、VRAM の中身がスプライト*である*わけではありません。スプライトは、何か（何でも）を画面に表示するために、VRAM の一部を*使っている*のです。たとえば、スプライトが使う最初のタイル `tid` を変えることもでき、ここでは L と R でできます。これは合法なだけでなく、アニメーションの標準的なやり方です（そのために VRAM を上書きすることもできますが、タイルのインデックスを設定し直すほうが速いです）。これを理解することが、ユーザーの視点から開発者の視点に移るときの要点の 1 つです。ユーザーには表面しか見えませんが、プログラマはその下を見て、本当に何が起きているのかが分かるのです。

通常のスプライトについては、これで以上です。複数のスプライトを使うのも、たいして違いはありません。1 つ見れば、全部見たようなものです。基本的なアニメーションも、置き場所の VRAM が足りなくなるまでは、問題にならないはずです。半透明合成やモザイクなど、まだ手を付けていない分野もいくつか残っていますが、それは[あとで](gfx.html)扱います。
