# 4. GBA のグラフィック入門

<!-- toc -->

## 全体の紹介 {#sec-intro}

GBA の液晶画面は、幅 240 ピクセル、高さ 160 ピクセルで、32768（15 ビット）色を表示できます。リフレッシュレートは、1 秒あたり 60 フレームにわずかに足りない値（59.73 Hz）です。GBA には、グラフィックを入れられる独立したレイヤーが 5 つあります。4 つの<dfn>背景</dfn>と、1 つの<dfn>スプライト</dfn>のレイヤーです。また、2 つのレイヤーの半透明合成やモザイク、そしてもちろん回転と拡大縮小といった、いくつかの特殊効果も使えます。

音やジョイパッドの機能は、ほんのわずかなレジスタでやりくりするしかありませんが、映像のシステムは、（比較的）多くのメモリを自由に使えます。I/O メモリにある多数のレジスタのほかに、96kb のビデオメモリ（`0600:0000h` から）、パレットメモリ（`0500:0000h`）、OAM メモリ（`0700:0000h`）があります。

## 描画期間とブランク期間 {#sec-blanks}

前に述べたとおり、GBA の画面全体は 60 分の 1 秒ごとに書き換えられますが、話はそれだけではありません。1 本のスキャンラインを描き終えると（HDraw の期間、240 ピクセル）、次のスキャンラインを描き始めるまでに休み（HBlank、68 ピクセル）があります。同じように、160 本のスキャンライン（VDraw）のあとには、最初からやり直す前に、68 スキャンライン分のブランク（VBlank）があります。画面のちらつき（ティアリング）を避けるために、位置のデータはたいてい VBlank のあいだに更新します。ほとんどのゲームが 60 fps か 30 fps で動くのは、このためです。（ちなみに、VBlank に同期していることは、PAL の国々でゲームが遅くなりがちだった理由でもあります。PAL のテレビは 50Hz で動いている（いた）ので、60 fps ではなく 50 fps しか出ず、誰もそれを考慮しなければ、ゲームが 17% 遅くなってしまうのです。考慮した会社はほとんどありませんでした <kbd>:(</kbd> 。）

[CowBite Spec](http://www.cs.rit.edu/~tjh8300/CowBite/CowBiteSpec.htm#Graphics%20Hardware%20Overview) と [GBATEK](https://problemkaputt.de/gbatek.htm#lcddimensionsandtimings) のどちらにも、表示のタイミングについての興味深い詳細が載っています。画面全体を 1 回書き換えるのにちょうど 280896 サイクルかかり、これをクロックの速さで割ると、フレームレートは 59.73 になります。上で示した描画期間とブランク期間から、1 ピクセルあたり 4 サイクル、1 スキャンラインあたり 1232 サイクルだと分かります。タイミングの詳細のまとめは、表 4.1 にあります。

<br>  

<table width=80%>
<tr align="center">
  <td>
  <div class="cpt" style="width:192px;">
  <img src="./img/gba_draw.png" id="fig:gba-draw" alt=""><br>
  <b>{*@fig:gba-draw}</b>: VDraw、VBlank、HBlank の期間。
  </div>

  <td>
	<table id="tbl:disp-timing" class="table-data">
	<caption align="bottom">
	  <b>{*@tbl:disp-timing}</b>: 表示のタイミングの詳細
	</caption>
	<col>
	<col span=2 align="right">
	<tr align="center">
	  <th>対象	<th>長さ		<th>サイクル
	<tr>
	  <td>ピクセル		<td>     1		<td>     4
	<tr>
	  <td>HDraw		<td>   240px	<td>   960
	<tr>
	  <td>HBlank	<td>    68px	<td>   272
	<tr>
	  <td>スキャンライン	<td>Hdraw+Hbl	<td>  1232
	<tr>
	  <td>VDraw 	<td>160*scanline<td>197120
	<tr>
	  <td>VBlank	<td>68*scanline	<td> 83776
	<tr>
	  <td>リフレッシュ	<td>VDraw+Vbl	<td>280896
	</table>
</table>

## 色とパレット {#sec-colors}

GBA は、5.5.5 形式の 16 ビットの色を表示できます。つまり、赤に 5 ビット、緑に 5 ビット、青に 5 ビットで、残りの 1 ビットは使われません。基本的には、ビットの並びは “<code>x<font color=blue>bbbbb</font><font color= green>ggggg</font><font color= red>rrrrr</font></code>” のようになります。`color.h` には、色を扱いやすくする define とマクロがいくつかあります。
<br>  
さて、パレットについてですが……

`<rant>`  
_みなさん、ここでの言葉は **“palette”** です！ ‘l’ が 1 つ、‘t’ が 2 つ、最後に ‘e’ です。**“pallet”** ではありません。これは「倉庫などで、保管や運搬のために材料を積み重ねる、低くて持ち運べる台（多くは両面使える）」のことです。**“pallette”** でもありません。これは「鎧で、脇の下を守る板」のことです。一番よく見かける誤記の **“pallete”** に至っては、辞書にすら載っていないので、検討する価値すらありません。“palette” ですよ、みなさん、“palette” です。_  
`</rant>`

それはともかく、GBA には 2 つのパレットがあります。1 つはスプライト（オブジェクト）用、もう 1 つは背景用です。どちらのパレットにも、16 ビットの色が 256 個入っています（それぞれ 512 バイト）。背景のパレットは `0500:0000h` から始まり、その直後の `0500:0200h` からスプライトのパレットが続きます。スプライトと背景は、これらのパレットを 2 通りに使えます。256 色の 1 つのパレットとして（1 ピクセルあたり 8 ビット）か、16 色の 16 個のサブパレット、つまり<dfn>パレットバンク</dfn>として（1 ピクセルあたり 4 ビット）です。

パレットについて最後に 1 つ。インデックス 0 は<dfn>透明色のインデックス</dfn>です。パレットを使うモードでは、値が 0 のピクセルは透明になります。

## ビットマップ、背景、スプライト {#sec-vid-types}

まとめると、GBA が知っているグラフィックの表し方は 3 種類です。<dfn>ビットマップ</dfn>、<dfn>タイル背景</dfn>、<dfn>スプライト</dfn>です。ビットマップとタイル背景（単に背景とも呼ばれます）は、画面全体の組み立て方を決めるものなので、両方を同時に有効にすることはできません。  
ビットマップモードでは、ビデオメモリはちょうど *w*×*h* のビットマップのように働きます。位置 (*x,y*) にピクセルを描くには、位置 *y\*w+x* に行って色を入れます。なお、GBA では、画面いっぱいの個々のピクセルを毎フレーム組み立てることはできません。単純に、ピクセルの数が多すぎるのです。

タイル背景は、まったく違うやり方で動きます。まず、ビデオメモリのある部分に、8x8 ピクセルの<dfn>タイル</dfn>を入れておきます。次に、別の部分にタイルマップを組み立てます。タイルマップにはインデックスが入っていて、画面に見える画像にどのタイルを使うかを GBA に伝えます。画面を組み立てるには、30x20 の数値のマップがあれば十分で、その数値が指すタイルの描画はハードウェアがやってくれます。このやり方なら、画面全体を毎フレーム更新*できます*。このグラフィックの方式に頼っていないゲームは、ほとんどありません。

最後に、スプライトがあります。スプライトは、小さな（8x8 から 64x64 ピクセルの）グラフィックのオブジェクトで、それぞれ別々に変形できます。ビットマップとタイルマップのどちらの背景とも一緒に使えます。

:::tip ビットマップモードより、タイルモードを使いましょう

ほとんどの種類のゲームでは、タイルモードのほうが向いています。ほかのチュートリアルの多くはビットマップモードを中心に扱っていますが、それは初心者に分かりやすいからにすぎず、ゲームでの実用性が理由ではありません。市販のゲームの大多数はタイルモードを使っています。それが何かを物語っているはずです。

:::

これが、グラフィックの 3 つの基本的な種類です。ただし、ほかの分け方も思いつきます。たとえば、ビットマップとタイル背景は、同時には使えず、画面全体を使うので、<dfn>背景</dfn>の種類にまとめられます。また、たまたま、タイル背景のタイルとスプライトは、メモリの並び方が同じです（つまり、8x8 ピクセルのタイルの集まりです）。なので、タイル背景とスプライトは、タイルの種類にまとめられます。

## 表示のレジスタ: REG_DISPCNT、REG_DISPSTAT、REG_VCOUNT {#sec-vid-regs}

グラフィックに関わることをするときに出会う I/O レジスタが 3 つあります。ディスプレイ制御の `REG_DISPCNT (0400:0000h)`、ディスプレイのステータスの `REG_DISPSTAT (0400:0004h)`、そしてスキャンラインのカウンタの `REG_VCOUNT (0400:0006h)` です。これらの名前はメモリの位置を表す define にすぎず、原則としては好きに決められます。ただ、ここでは一番よく使われている、[Pern Project](http://www.drunkencoders.com) で使われている名前を使います。

REG_DISPCNT レジスタは、画面の一番中心となる制御です。このレジスタのビットの並びと、その意味は、次の表のとおりです。これが、レジスタやレジスタのような領域に私が使う、一般的な形式です。形式の詳細は、[はじめに](intro.html#ssec-note-reg)ですでに説明しました。

<div class="reg">
<table class="table-reg" id="tbl:reg-dispcnt">
<caption class="reg">
  REG_DISPCNT @ 0400:0000h
</caption>
<tr class="bits">
	<td>F<td>E<td>D<td>C<td>B<td>A<td>9<td>8
	<td>7<td>6<td>5<td>4<td class="rof">3<td>2 1 0
<tr class="bf">
	<td class="rclr4">OW
	<td class="rclr4">W1
	<td class="rclr4">W0
	<td class="rclr1">Obj
	<td class="rclr1">BG3
	<td class="rclr1">BG2
	<td class="rclr1">BG1
	<td class="rclr1">BG0
	<td class="rclr6">FB
	<td class="rclr2">OM
	<td class="rclr5">HB
	<td class="rclr3">PS
	<td class="rclr7">GB
	<td class="rclr0">Mode
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">
  <td>0-2 <td class="rclr0">Mode
  <td>DCNT_MODEx. <i>DCNT_MODE#</i>
  <td>ビデオモードを設定します。0、1、2 はタイルモード、3、4、5 はビットマップモードです。
<tr class="bg1">
  <td class="rof">3   <td class="rclr7">GB
  <td>DCNT_GB
  <td>カートリッジがゲームボーイカラーのゲームのときにセットされます。読み取り専用。
<tr class="bg0">
  <td>4   <td class="rclr3">PS
  <td>DCNT_PAGE
  <td>ページの選択。モード 4 と 5 では、ページフリップを使って、より滑らかな
    アニメーションにできます。このビットは、表示するページを選びます（そうすることで、
    もう一方のページには、表示を乱さずに描画できます）。
<tr class="bg1">
  <td>5   <td class="rclr5">HB
  <td>DCNT_OAM_HBL
  <td>HBlank のあいだに OAM へアクセスできるようにします。普通、OAM は VDraw のあいだはロックされています。
    1 ラインあたりに描かれるスプライトのピクセルの数が減ります。
<tr class="bg0">
  <td>6   <td class="rclr2">OM
  <td>DCNT_OBJ_1D
  <td>オブジェクトのマッピングモード。タイルのメモリは、タイルの 32x32 の
    行列とみなせます。縦に複数のタイルからなるスプライトの場合、
    このビットは、次の行のタイルが、行列の構造に合わせて前の行の
    真下にある（2D マッピング、<code>OM</code>=0）か、
    すぐ隣にあって、メモリがスプライトの配列として並んでいる（1D
    マッピング、<code>OM</code>=1）かを決めます。詳しくは
    <a href="regobj.html">スプライト</a>の章で説明します。
<tr class="bg1">
  <td>7   <td class="rclr6">FB
  <td>DCNT_BLANK
  <td>画面を強制的にブランクにします。
<tr class="bg0">
  <td>8-C  <td class="rclr1">BG0-BG3, Obj
  <td>DCNT_BGx, DCNT_OBJ. <i>DCNT_LAYER#</i>
  <td>対応する背景とスプライトの描画を有効にします。
<tr class="bg1">
  <td>D-F <td class="rclr4">W0-OW
  <td>DCNT_WINx, DCNT_WINOBJ
  <td>それぞれ、ウィンドウ 0、1、オブジェクトウィンドウを使えるようにします。
    ウィンドウを使うと、特定の領域を隠せます（ゼルダの伝説 神々のトライフォースで、
    ランプがそうしていたように）。
</tbody>
</table>
</div>

ディスプレイ制御の設定は、たぶん最初にやることになるでしょう。簡単なデモなら、一度設定したらそのままで構いません。ただ、ビデオモードを切り替えると、面白い結果になることもあります。
<br>  
さて、前に挙げたほかの 2 つのレジスタ、`REG_DISPSTAT` と `REG_VCOUNT` です。後者は、今処理しているスキャンラインを教えてくれます。このカウンタは VBlank のあいだも進み続けるので、227 まで数えてから 0 に戻る点に注意してください。前者は描画とブランクの状態を教えてくれるもので、表示の[割り込み](interrupts.html)を設定するのにも使います。ここで有効にできる割り込みを使うと、本当にすごいこともできます。たとえば、HBlank の割り込みは [Mode 7](mode7.html) のグラフィックを作るのに使われます。それがどういう仕組みか、知りたいですよね？

<div class="reg">
<table class="table-reg" id="tbl:reg-dispstat">
<caption class="reg">
  REG_DISPSTAT @ 0400:0004h
</caption>
<tr class="bits">
  <td>F E D C B A 9 8
  <td>7 6<td>5<td>4<td>3<td class="rof">2<td class="rof">1<td class="rof">0
<tr class="bf">
  <td class="rclr3">VcT
  <td>-
  <td class="rclr2">VcI
  <td class="rclr1">HbI
  <td class="rclr0">VbI
  <td class="rclr2">VcS
  <td class="rclr1">HbS
  <td class="rclr0">VbS
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="15%">
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">
  <td>0<td class="rclr0">VbS
  <td>DSTAT_IN_VBL
  <td>VBlank のステータス。読み取り専用。VBlank の間はセットされ、VDraw の間はクリアされる。
<tr class="bg1">
  <td>1 <td class="rclr1">HbS
  <td>DSTAT_IN_HBL
  <td>HBlank のステータス。読み取り専用。HBlank の間はセットされる。
<tr class="bg0">
  <td>2 <td class="rclr2">VcS
  <td>DSTAT_IN_VCT
  <td>VCount のトリガーのステータス。現在のスキャンラインが、トリガーの
    スキャンラインと一致するとセットされる（ <code>REG_VCOUNT</code> == 
    <code>REG_DISPSTAT</code>{8-F} ）。
<tr class="bg1">
  <td>3 <td class="rclr0">VbI
  <td>DSTAT_VBL_IRQ
  <td>VBlank の割り込み要求。セットすると、VBlank で割り込みが
    発生する。
<tr class="bg0">
  <td>4 <td class="rclr1">HbI
  <td>DSTAT_HBL_IRQ
  <td>HBlank の割り込み要求。
<tr class="bg1">
  <td>5 <td class="rclr2">VcI
  <td>DSTAT_VCT_IRQ
  <td>VCount の割り込み要求。現在のスキャンラインがトリガーの値と
    一致すると、割り込みが発生する。
<tr class="bg0">
  <td>8-F <td class="rclr3">VcT
  <td><i>DSTAT_VCT#</i>
  <td>VCount のトリガー値。現在のスキャンラインがこの値になると、
    ビット 2 がセットされ、要求されていれば割り込みが発生する。
</tbody>
</table>
</div><br>

<div class="reg">
<table class="table-reg" id="tbl:reg-vcount" width="320">
<caption class="reg">
  REG_VCOUNT @ 0400:0006h（読み取り専用）
</caption>
<tr class="bits">
	<td>F E D C B A 9 8
	<td class="rof">7 6 5 4 3 2 1 0
<tr class="bf">
  <td>-
  <td class="rclr0">Vc
</table>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
<tr align="left"><th>ビット<th>名前<th>説明
<tbody valign="top">
<tr class="bg0">
  <td>0-7 <td class="rclr0">Vc
  <td>垂直方向のカウント。範囲は [0,227]
</tbody>
</table>
</div>

## 垂直同期 その 1: ビジーウェイトのループ {#sec-vsync1}

前に述べたとおり、VBlank は、タイミングをとる仕組みとして、またゲームのデータを更新するために使います。これを <dfn>vsync</dfn>（**v**ertical **sync**hronisation、垂直同期）といいます。垂直同期のやり方はいくつかあります。一番よく使われる 2 つの方法は、while ループを使って、`REG_VCOUNT` か `REG_DISPSTAT` を調べるものです。たとえば、VBlank はスキャンライン 160 から始まるので、`REG_VCOUNT` がこの値を超えるのを待てばいいわけです。

```c
#define REG_VCOUNT *(u16*)0x04000006

void vid_vsync()
{    while(REG_VCOUNT < 160);   }
```

残念ながら、このコードにはいくつか問題があります。

まず、160 になるのを待つために中身が空の `while` ループを回しているだけだと、コンパイラが賢く振る舞おうとするかもしれません。ループの中で `REG_VCOUNT` が変わらないことに気づき、すぐ参照できるように、その値をレジスタに入れてしまうのです。その値がいずれかの時点で 160 より小さい可能性は十分にあるので、見事な無限ループのできあがりです。これを防ぐには、_`volatile`_ というキーワードを使います（`tonc_memmap.h` と `tonc_types.h` を見てください）。

2 つ目に、小さなデモでは、単に VBlank を待つだけでは足りません。もう一度 `vid_sync()` を呼んだときにまだ同じ VBlank の中にいるかもしれず、その場合はすぐに通り抜けてしまいます。それでは 60 fps に同期できません。そうするには、まず*次の* VDraw まで待つ必要があります。すると、`vid_sync` は次のようになります。

```c
#define REG_VCOUNT *(vu16*)0x04000006

void vid_vsync()
{
    while(REG_VCOUNT >= 160);   // wait till VDraw
    while(REG_VCOUNT < 160);    // wait till VBlank
}
```

これなら、常に次の VBlank が始まるまで待ちます。それに、`REG_VCOUNT` は今度は _`volatile`_ になっています（“`vu16`” は、<u>v</u>olatile な <u>u</u>nsigned（<u>16</u>bit）short として `typedef` されています。この種の短い呼び名をたくさん使うので、慣れてください）。これが 1 つのやり方です。もう 1 つは、ディスプレイのステータスのレジスタの最後のビット、`REG_DISPSTAT`\{0\} を調べる方法です。
<br>  
これで終わりでしょうか？ ええと……いいえ、そうとも言えません。確かに、これで簡単に垂直同期できるようになりましたが、とてもお粗末なやり方でもあります。while ループの中にいるあいだも、CPU のサイクルを燃やし続けています。もちろん、それは電池を消費します。しかも、その while ループの中ではまったく何もしていないので、電池を使っているどころか、実際には無駄にしています。さらに、最初はたぶん小さなゲームしか作らないでしょうから、*とても*多くの電池を無駄にすることになります。おすすめの垂直同期のやり方は、やることが終わったら CPU を省電力モードにして、割り込みを使って再び目覚めさせることです。その手順は[こちら](swi.html#sec-vsync2)で読めますが、[割り込み](interrupts.html)と [BIOS コール](swi.html)の使い方を知っている必要があるので、しばらく待ったほうがいいかもしれません。
