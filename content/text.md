# 19. テキストシステム

<!-- toc -->



:::note 非推奨のお知らせ

この章の内容は [TTE](tte.html) に置き換えられました。この章の情報はまだ役に立つかもしれませんが、本格的に使うなら TTE を選ぶべきです。

:::

## はじめに {#sec-intro}

<div id="{#cd-hello}">

```c
#include <stdio.h>

int main()
{
    printf("Hello World");
    return 0;
}
```

</div>

ああ、そうです、“Hello world” です。あらゆる C の講座やシステムで最初に出てくる定番の例です。ただし、ゲーム機は別です。PC で文字を表示するのは世界一簡単なことですが、ゲーム機では実は少しやっかいです。`printf()` 関数がないというより、書き込む先がどこにもなく、書くためのフォントすらないのです（しかも、考えるべきことはそれだけではありません）。そう、文字を表示できるようにしたければ、全部自分でゼロから作らなければなりません。そして、画面に文字を書けるようにはしたいはずです。

では、テキストシステムには何が必要でしょうか？ 実は、これは簡単な質問ではありません。当然、フォントが必要です。いろいろな文字が描かれたビットマップがあれば十分で、GBA でベクターフォントに悩む必要はありません。次に、特定の文字を取り出して画面に表示する方法が必要です。

でも、ちょっと待ってください。どのビデオモードを使うのでしょうか？ タイルマップ、ビットマップモード、スプライトから選べますが、どれもまったく違うやり方で扱う必要があります。そのうちの 1 つで済ませるのか、それともすべてで使えるものを作るのか。また、どんなフォントを使い、文字の大きさはどうするのか。固定幅か、可変幅か。可変幅や大きさは、ビットマップモードではあまり問題になりませんが、タイルでつなぎ合わせるのはひどく面倒です。それから、タイルの場合だけですが、フォント全体を VRAM に置いておくのか。そうするなら、タイルがたくさん必要になります。特に、すべてを同時に使うことはまずないことを考えるとなおさらです。そのとき使っているグリフだけをコピーするほうが VRAM の効率は良いでしょう。ただし、それには管理の手間がかかります。

これらの項目だけでも、20 通り以上のテキストシステムの実装を作れるだけの選択肢があり、しかもそれらはとても微妙な点で互換性がありません。少なくとも、それぞれに `putc()` と `puts()` が必要です。それから `printf()` のような関数も必要かもしれません。グリフの配置は内側で行われるので、テキストの種類ごとにです。画面のクリアもあるかもしれません。スクロールの機能はどうでしょう。まあ、言いたいことはわかりますね。

誰もが必要としうるあらゆる要求に合わせた、大きくて複雑なシステムを作ることもできるとは思います。でも、私はそうしません。まず、それは少し時間の無駄だからです。たとえばビットマップモードとタイルマップモードを同時に使えるようにする必要がある可能性は、実質的に（あるいは実際に）ゼロです。たいていは、1 つのビデオモードを使い、それで通します。ほとんど使われることのない想像しうるすべての組み合わせに対応するために時間（と容量）を費やすのは、おそらく割に合いません。それに、処理の核心にあるちょっとした細部以外はほとんど同じコードを大量に書くのは、単純にうんざりします。

この章の目的は、単純で軽量なテキストの書き込み処理をひとそろい作って使う方法を示すことです。究極のテキストシステムは期待しないでください。私の主な関心は、肝心なこと、つまり文字列の文字を画面に出すことを実現することです。これは、次の機能を持つ中核的なテキストシステムです。

- ビットマップ（モード 3、4、5）、通常のタイルマップ（モード 0、1）、スプライトに対応。
- 文字列を表示する `xxx_puts()` と、それを消す `xxx_clrs()` を用意する。引数は文字列、描く位置、色の情報。スクロールや書式指定が欲しければ、それはお任せします。
- フォントは固定幅でモノクロ、1 文字につき 8x8 のタイル 1 つ。グリフは 8x8 より小さくてもよく、可変幅のためのフックも残しておきますが、複数タイルのフォントを許すと、ひどいことになってしまいます。
- 可変の文字マップ。文字の小さな集合だけを使うつもりの場合や、ASCII ではないグリフの並びの場合に、とても便利な機能です。

この構成は最も基本的な場合に対応し、準備の仕方にいくらかの変化をつけられますが、それ以外はほとんどありません。とはいえ、そうした追加の機能はおそらくゲームに強く依存するもので、汎用のテキストシステムには向かないかもしれません。追加の機能が欲しければ、自分で書くのはそれほど難しくないはずです。

:::note printf() がない？ マジで？

GBA には `printf()` がないと言いましたが、これは正確ではありません。少なくとも、今ではもう違います。標準の入出力ルーチンに自分の入出力の仕組みをつなぐことができ、`libgba` ではそれが行われています。

:::

:::note 半ば時代遅れ

このページで説明しているものよりずっと強力な（つまり、本当にすべてのビデオモードで動き、printf もある）別のテキストシステムが手元にあります。ただ、それはかなり大きく、完全には仕上がっておらず、説明のページを書いてデモに合わせて本文を直すには、少し時間がかかりそうです。関係する変更を入れた libtonc の版は [http://www.coranac.com/files/misc/tonclib-1.3b.rar](http://www.coranac.com/files/misc/tonclib-1.3b.rar) にあります。

:::

## テキストシステムの内部 {#sec-in}

### 変数 {#ssec-in-tb}

テキストシステムの状態を管理するために、いくつかの変数が必要です。明らかに必要なのは、フォントと文字マップです。柔軟にしておきたいので、これらには 2 つのポインタを使い、望むなら自分のフォントや文字マップを使えるようにします。どこに書き込みたいかも知る必要があり、これは書き込み先の基準のポインタで指定します。おまけとして、グリフの間隔を変えられるように文字の大きさの変数も持たせ、可変幅のフォントに備えて、文字幅の配列へのポインタまで用意します。

これらを格納するのに構造体を使います。私にとって管理しやすいからでもありますが、CPU とコンパイラが構造体をより効率よく扱えるからでもあります。将来の拡張に備えて、数バイトを空けておきます。最後に、この構造体のインスタンスと、それへのポインタを用意します。必要になれば（まずないでしょうが）、別のシステムに切り替えられるようにするためです。たしかに数バイトを無駄にしていますが、これのせいで IWRAM を使い切るようなら、もっと大きな心配事を抱えていると言わざるをえません。

```c{#cd-txt-base}
// In text.h
typedef struct tagTXT_BASE
{
    u16 *dst0;      // writing buffer starting point
    u32 *font;      // pointer to font used
     u8 *chars;     // character map (chars as in letters, not tiles)
     u8 *cws;       // char widths (for VWF)
     u8  dx,dy;     // letter distances
    u16  flags;     // for later
     u8  extra[12]; // ditto
} TXT_BASE;

extern TXT_BASE __txt_base, *gptxt;

// In text.c
TXT_BASE __txt_base;                Main TXT_BASE instance
TXT_BASE *gptxt= &__txt_base;        and a pointer to it
```

### フォント {#ssec-in-font}

<div class="lblock">
  <div class="cpt" style="width:400px;">
    <img src="img/tonc_font.png" alt="Default tonc font" id="fig:img-tonc-font">
    <br>

**{\*@fig:img-tonc-font}**: tonc のデフォルトのフォント。mini-ascii、モノクロ、グリフあたり 8x8 ピクセル。

  </div>
</div>

@fig:img-tonc-font は、私が使うフォントです。このフォントはモノクロで、各グリフは 8x8 の枠に収まります。96 個のグリフは ASCII 全体の一部で、私はこれを <dfn>mini-ascii</dfn> と呼んでいます。これは標準の ASCII 表の大部分を含む ASCII の下半分ですが、ASCII 0-31 は省いています。それらはエスケープコードで、どのみち表示できる文字ではないからです。

別のグリフの並びを持つ別のフォントを使うこともできますが、以下で示す関数は、1 グリフにつき 1 タイルで、_しかも_タイルの配置になっていることを前提にしています。この配置が必要なのは、すべてのモードで使うつもりだからで、1 タイルでない形式はタイルモードでは地獄になります。

もう 1 つの制約は、フォントを 1bpp にビットパックしなければならないことです。これにはいくつか理由があります。まず、大きさの問題です。96 グリフの 16 ビットのフォント（モード 3/5 用）は 12kB を占めます。それを 1bpp にパックすれば 1kB 未満です！ たしかにモノクロに限られますが、フォントにとってそれはあまり問題ではありません。フォントはたいていどのみちモノクロですし、1 ビットしか必要ないところに 16 ビットを使うのは少し無駄に思えます。次に、16bpp のフォントを 4bpp や 8bpp のタイルでどうやって使えばいいのでしょうか？ 低い bpp から高い bpp に変換するほうがずっと簡単です。もちろん、この構成が気に入らなければ、自由に自分の関数を書いてください。

フォントのデータそのものについては、これで全部です。

<pre id="cd-toncfont"><code class="language-c hljs">const unsigned int toncfontTiles[192]=
{
    0x00000000, 0x00000000, 0x18181818, 0x00180018, 0x00003636, 0x00000000, 0x367F3636, 0x0036367F,
    0x3C067C18, 0x00183E60, 0x1B356600, 0x0033566C, 0x6E16361C, 0x00DE733B, 0x000C1818, 0x00000000,
    0x0C0C1830, 0x0030180C, 0x3030180C, 0x000C1830, 0xFF3C6600, 0x0000663C, 0x7E181800, 0x00001818,
    0x00000000, 0x0C181800, 0x7E000000, 0x00000000, 0x00000000, 0x00181800, 0x183060C0, 0x0003060C,
    0x7E76663C, 0x003C666E, 0x181E1C18, 0x00181818, 0x3060663C, 0x007E0C18, 0x3860663C, 0x003C6660,
    0x33363C38, 0x0030307F, 0x603E067E, 0x003C6660, 0x3E060C38, 0x003C6666, 0x3060607E, 0x00181818,
    0x3C66663C, 0x003C6666, 0x7C66663C, 0x001C3060, 0x00181800, 0x00181800, 0x00181800, 0x0C181800,
    0x06186000, 0x00006018, 0x007E0000, 0x0000007E, 0x60180600, 0x00000618, 0x3060663C, 0x00180018,

    0x5A5A663C, 0x003C067A, 0x7E66663C, 0x00666666, 0x3E66663E, 0x003E6666, 0x06060C78, 0x00780C06,
    0x6666361E, 0x001E3666, <span class="rem">0x1E06067E</span>, <span class="rem">0x007E0606</span>, 0x1E06067E, 0x00060606, 0x7606663C, 0x007C6666,
    0x7E666666, 0x00666666, 0x1818183C, 0x003C1818, 0x60606060, 0x003C6660, 0x0F1B3363, 0x0063331B,
    0x06060606, 0x007E0606, 0x6B7F7763, 0x00636363, 0x7B6F6763, 0x00636373, 0x6666663C, 0x003C6666,
    0x3E66663E, 0x00060606, 0x3333331E, 0x007E3B33, 0x3E66663E, 0x00666636, 0x3C0E663C, 0x003C6670,
    0x1818187E, 0x00181818, 0x66666666, 0x003C6666, 0x66666666, 0x00183C3C, 0x6B636363, 0x0063777F,
    0x183C66C3, 0x00C3663C, 0x183C66C3, 0x00181818, 0x0C18307F, 0x007F0306, 0x0C0C0C3C, 0x003C0C0C,
    0x180C0603, 0x00C06030, 0x3030303C, 0x003C3030, 0x00663C18, 0x00000000, 0x00000000, 0x003F0000,

    0x00301818, 0x00000000, 0x603C0000, 0x007C667C, 0x663E0606, 0x003E6666, 0x063C0000, 0x003C0606,
    0x667C6060, 0x007C6666, 0x663C0000, 0x003C067E, 0x0C3E0C38, 0x000C0C0C, 0x667C0000, 0x3C607C66,
    0x663E0606, 0x00666666, 0x18180018, 0x00301818, 0x30300030, 0x1E303030, 0x36660606, 0x0066361E,
    0x18181818, 0x00301818, 0x7F370000, 0x0063636B, 0x663E0000, 0x00666666, 0x663C0000, 0x003C6666,
    0x663E0000, 0x06063E66, 0x667C0000, 0x60607C66, 0x663E0000, 0x00060606, 0x063C0000, 0x003E603C,
    0x0C3E0C0C, 0x00380C0C, 0x66660000, 0x007C6666, 0x66660000, 0x00183C66, 0x63630000, 0x00367F6B,
    0x36630000, 0x0063361C, 0x66660000, 0x0C183C66, 0x307E0000, 0x007E0C18, 0x0C181830, 0x00301818,
    0x18181818, 0x00181818, 0x3018180C, 0x000C1818, 0x003B6E00, 0x00000000, 0x00000000, 0x00000000,
};
</code></pre>

そう、これがフォントの_全体_で、1 ページにきれいに収まっています。これがビットパックのなせる業です。ただ、ほかの圧縮方法と同じく、これが本当に先ほどのフォントなのかを見て取るのは少し難しいかもしれないので、目の前にあるものについて少し説明します。

#### ビットパック

<div class="cpt_fr">
<table id="tbl:endian" class="table-data">
<caption align="bottom">
  <b>{@tbl:endian}</b>: バイト列 01h、02h、03h、04h の
  ビッグエンディアンとリトルエンディアンでの解釈
</caption>
<tbody align="center">
<tr>
  <th>big u32     <td colspan=4> 0x01020304
<tr>
  <th>big u16     <td colspan=2> 0x0102 <td colspan=2> 0x0304
<tr>
  <th>u8          <th> 0x01 <th> 0x02   <th> 0x03 <th> 0x04
<tr>
  <th>little u16  <td colspan=2> 0x0201 <td colspan=2> 0x0403
<tr>
  <th>little u32  <td colspan=4> 0x04030201
</tbody>
</table>
</div>

ビットパックを理解するのは難しくありません。データは、大きなビットの畑のようなものにすぎません。ビットパックでは、一定の間隔でビットを取り除き、残りをつなぎ合わせるだけです。私たちのフォントはモノクロなので、情報は 1 ビットしかありません。C で最も小さいデータ型のバイトでも、1 ピクセルに 1 バイトを使うと、7 ビットが使われずに残ります。しかし、1 バイトに 8 ピクセルを詰め込むこともでき、そうすれば容量を 8 分の 1 に節約できます。ちなみに、これは 88% の圧縮率で、かなり良いと言えるでしょう。もちろん、ほかのページをすべて読んでいれば、すでにビットパックの例に気づいているはずです。4bpp のタイルは 1 バイトに 2 ピクセルでビットパックされています。ですから、これはまったく新しい話ではないはずです。

ビットパックで容量を大きく節約できますし、原理的には、マスクとシフトだけの話なので簡単です。ただし、大きな落とし穴が 1 つあります。<dfn>エンディアン</dfn>です。ほかのデータの配列で、すでにその一つの形を見ています。ワード `0x01234567` は、ARM（と Intel）のシステムでは、実際にはバイト列 `0x67`、`0x45`、`0x23`、`0x01` として格納されます。これは<dfn>リトルエンディアン</dfn>と呼ばれます。ワードの小さい端（複数バイトの型の下位のバイト）が下位のアドレスに格納されるからです。最上位のバイトから先に格納する<dfn>ビッグエンディアン</dfn>もあります。違いは @tbl:endian で見られます。16 進エディタやメモリビューア（たとえば VBA のもの）の中には、データをバイト、ハーフワード、ワードのどれで表示するかを切り替えられるものがあるので、そこで違いを実際に確かめられます。これによってデータそのものが変わるわけでは_なく_、違う見方で_見ている_だけだということを忘れないでください。

ビットパックでは、ビットの単位でもエンディアンを扱う必要があります。フォントのデータは、3 つの理由から、ビットもバイトも一貫してリトルエンディアンの形式でパックしています。まず、GBA のビットパックされたデータはどのみちこの形で動くので、BIOS の BitUnpack ルーチンが使えます。次に、数え方の点でこちらのほうが自然な形です。下位のビットが先に来ます。3 つ目に、常に下にシフトし、処理済みのビットをそうやって捨てていけるので、マスクがより簡単で速くなります。さて、私たちは数をビッグエンディアンで書くので、見た目にはビッグエンディアンのほうが自然でしょう。そのため、ビットマップもビットの単位ではビッグエンディアンであることがよくあります（訳注: 原文は "bit-little" ですが、文脈と直後の BMP の例から "bit-big" の意味と解釈しました）。たとえば Windows の BMP ファイルは、一番左のピクセルが最上位のビットにあり、ビットの単位ではビッグエンディアンになっています。ところが Windows は Intel のアーキテクチャで動いていて、それは実は_バイト_の単位ではリトルエンディアンなので、混乱は最大になります。やれやれ。まあいいでしょう。

まだ少しぼんやりしているなら、@fig:img-fontpack を見てください。「F」が 8x8 ピクセルから 2 つのワードにどうパックされるかを示しています。64 個のピクセルすべてに 0 から 63 の番号を付けています。これらはビット番号に対応します。連続する 8 ビットずつが 1 バイトになります。0-7 がバイト 0、8-15 がバイト 1、という具合です。私たちは普通、数をビッグエンディアンで書くので、ビットが左右に鏡映しになっているように見えることに注意してください。ですから、それは忘れて、メモリ上のビットを 0 から 63 まで順にたどっていくと考えてください。ビットをワードとして見ることもできます。ビット 0-31 がワード 0、32-63 がワード 1 です。

<div class="cblock">
<div class="cpt">
<div style="width:700px;">
<table id="fig:img-fontpack" style="width:100%;">
  <tbody align="center">
  <tr>
    <th style="width:150px;">ピクセル
    <th>
    <th style="width: 150px;">ビット
    <th>
    <th style="width: 75px;">バイト
    <th>
    <th style="width: 75px;">ワード
  <tr>
    <td>
    <img src="img/font_pack.png" alt="pixels" style="width: 160px; margin-top: 24px;">
    <td><span class="rarr">&rarr;</span>
    <td> <!-- bits -->
    <table cellpadding=1 cellspacing=0 class="reg">
      <tr>
        <td class="bdrLL">&nbsp;
        <th> 7 <th> 6 <th> 5 <th> 4 <th> 3 <th> 2 <th> 1 <th> 0
        <td class="bdrRR">&nbsp;
      <tr>
        <td class="bdrLL" rowspan=8>&nbsp;
        <td> 0 <th> 1 <th> 1 <th> 1 <th> 1 <th> 1 <th> 1 <td> 0
        <td class="bdrRR" rowspan=8>&nbsp;
      <tr>
        <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <th> 1 <th> 1 <td> 0
      <tr>
        <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <th> 1 <th> 1 <td> 0
      <tr>
        <td> 0 <td> 0 <td> 0 <th> 1 <th> 1 <th> 1 <th> 1 <td> 0
      <tr>
        <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <th> 1 <th> 1 <td> 0
      <tr>
        <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <th> 1 <th> 1 <td> 0
      <tr>
        <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <th> 1 <th> 1 <td> 0
      <tr>
        <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <td> 0 <td> 0
    </table>
    <td><span class="rarr">&rarr;</span>
    <td> <!-- bytes -->
    <table cellpadding=1 cellspacing=0>
      <tr>
        <th colspan=3>&nbsp;
      <tr>
        <td class="bdrLL" rowspan=4>&nbsp;
          <td> 0x7E
        <td class="bdrRR" rowspan=4>&nbsp;
      <tr><td> 0x06
      <tr><td> 0x06
      <tr><td> 0x1E
      <tr>
        <td class="bdrLL" rowspan=4>&nbsp;
          <td> 0x06
        <td class="bdrRR" rowspan=4>&nbsp;
      <tr><td> 0x06
      <tr><td> 0x06
      <tr><td> 0x00
    </table>
    <td><span class="rarr">&rarr;</span>
    <td> <!-- words -->
    <table cellpadding=1 cellspacing=0 style="display: block;">
      <tr>
        <th colspan=3>&nbsp;
      <tr>
        <td class="bdrLL">&nbsp;
          <td> 0x1E06067E
        <td class="bdrRR">&nbsp;
      <tr>
        <td class="bdrLL">&nbsp;
          <td> 0x00060606
        <td class="bdrRR">&nbsp;
    </table>
   </tbody>
</table>
  <b>@fig:img-fontpack</b>: 
  「F」を、8x8 のタイルから、1bpp でビットもバイトもリトルエンディアンの
  ワードにしたもの。
</div>
</div>
</div>

### 文字マップ {#ssec-in-charmap}

mini-ascii のフォントがあるのは結構なのですが、文字列は ASCII 全体を使うので、これが問題になるかもしれません。いや、実はそうでもないのですが、変換の方法はいくつかあります。

1 つ目は、たとえば「A」（ASCII 65）をグリフのインデックス 33 に変換するような、巨大な switch ブロックを作ることです。それを 96 個のグリフすべてについて書きます。これがひどいやり方なのは明らかでしょう。明らか_なはず_なのですが、実際にはそうでもないようで、そういうコードが出回っています。ここでこれに触れたのは、それが何なのかを見分けて、うんと遠ざかっていられるようにするためです。簡単に言うと、各ケースの違いが別のオフセット（しかも_固定の_オフセット）を返すことだけの switch ブロックを書いているなら、何かをとてもとても間違えています。

2 つ目の方法は、あらゆる点で大幅に改善されたもので、単に 32 を引くことです。そもそも mini-ascii はそうやって定義されたのですから。速くて、短くて、的を射ています。

しかし、私は 3 つ目の選択肢、ルックアップテーブルがちょっと気に入っています。ルックアップテーブルが数学でどれほど役立つかはすでに見ましたが、それ以外にもずっと多くのことに使えます。この場合のルックアップテーブルは<dfn>文字マップ</dfn>で、各 ASCII 文字のグリフのインデックスが入っています。これには単純な引き算の利点がほぼすべてあります（表引きは数サイクル遅いかもしれません）が、ずっと柔軟です。たとえば、ASCII ではない文字マップにしたり、大文字と小文字を同じグリフに割り当てたりといったことができます。もう 1 つ「面白い」のは、フォントがいわゆる文字である必要は実はないということです。マップできる画像データなら何でもかまいません。ルックアップテーブルを使えば、枠線の「フォント」さえあれば、テキストシステムを使って枠線を簡単に描けます。私が使っているルックアップテーブルは 256 バイトの長さです。Unicode には足りないかもしれません（東洋の皆さん、ごめんなさい）が、私の目的には十分です。

#### 全体の設計

コードの面で最初にすることは、テキストの基準の構造体のメンバを初期化することです。つまり、フォントを結びつけ、グリフの大きさを設定し、ルックアップテーブルを初期化します。これは `txt_init_std()` でできます。

```c

u8 txt_lut[256];

// Basc initializer for text state
void txt_init_std()
{
    gptxt->dx= gptxt->dy= 8;

    gptxt->dst0= vid_mem;
    gptxt->font= (u32*)toncfontTiles;
    gptxt->chars= txt_lut;
    gptxt->cws= NULL;

    int ii;
    for(ii=0; ii<96; ii++)
        gptxt->chars[ii+32]= ii;
}
```

テキストの種類によっては、もっと専用の初期化関数が必要になることもあります。それはそのときに扱います。文字列を書き込む処理の基本的な構造は、下のとおりです。実はとても単純で、とても汎用的なのですが、残念ながら `xxx_putc()` が内側のループにあるので、テキストの方式ごとに、それぞれの文字を描く処理の周りにほぼ同じラッパーを用意しなければなりません。画面から文字列を消す `xxx_clrs()` という関数もあります（画面全体を消すわけではありません）。形は `puts()` の兄弟とほぼ同じで、かなり単純でもあるので、ここでは詳しく説明しません。

```c
// Pseudo code for xxx_puts
void xxx_puts(int x, int y, const char *str, [[more]])
{
    [[find real writing start]]
    while(c=*str++)     // iterate through string
    {
        switch(c)
        {
        case [[special chars ('\n' etc)]]:
            [[handle special]]
        case [[normal chars]]:
            [[xxx_putc(destination pointer, lut[c])]]
            [[advance destination]]
        }
    }
}
```

## ビットマップのテキスト {#sec-bm}

ビットマップのテキストは、モード 3、4、5 に関するものです。モード 3 ができれば、モード 5 もほぼできたようなものです。2 つの違いは、ピッチと、場合によっては開始点だけだからです。モード 4 は違います。8bpp だからというだけでなく、そのせいで 2 ピクセルを同時に扱わなければならないからです。

### 内部のルーチン {#ssec-bm-intl}

私はビットマップ関係の関数を 2 つの部分に分けて作ることにしています。アドレスとピッチを引数に取る、16 ビット用と 8 ビット用の内部の関数があり、それから座標を受け取ってそれらを呼ぶインラインのインターフェース関数があります。16 ビット用の内部の書き込み処理を下に示し、その下で主な部分を説明します。

```c{#cd-bm16-puts}
void bm16_puts(u16 *dst, const char *str, COLOR clr, int pitch)
{
    int c, x=0;

    while((c=*str++) != 0)      // (1) for each char in string
    {
        // (2) real char/control char switch
        if(c == '\n')       // line break
        {
            dst += pitch*gptxt->dy;
            x=0;
        }
        else                // normal character
        {
            int ix, iy;
            u32 row;
            // (3) point to glyph; each row is one byte
            u8 *pch= (u8*)&gptxt->font[2*gptxt->chars[c]];
            for(iy=0; iy<8; iy++)
            {
                row= pch[iy];
                // (4) plot pixels until row-byte is empty
                for(ix=x; row>0; row >>= 1, ix++)
                    if(row&1)
                        dst[iy*pitch+ix]= clr;
            }
            x += gptxt->dx;
        }
    }
}
```

1.  文字列のすべての文字をたどる伝統的な方法です。`c` が扱う文字になりますが、それが終端文字（`'\0'`）なら止まります。
2.  通常の文字と制御文字の切り替えです。`'\n'` や `'\t'` のような制御文字は別に扱うべきです。今は改行だけを確認していますが、ほかのものも簡単に追加できます。
3.  ここから面白くなります。この行では、まずルックアップテーブルを使ってフォントの中のグリフのインデックスを調べ、フォントの中の実際のグリフを調べ（グリフあたり 2 ワードあるので 2 を掛けます）、それからバイトのポインタ `pch` をそのグリフを指すように設定しています。  
    ここではいくつかのことがうまく噛み合っています。まず、すべてのグリフがちょうど 8 バイトずつ離れているので、グリフのデータを見つけるのがとても簡単です。自分のフォントで自分のテキストシステムを作るなら、「I」のような小さな文字でピクセルを無駄にするとしても、一定のオフセットを使うことをおすすめします。次に、1bpp のタイル形式なので、各行はちょうど 1 バイトの長さで、グリフのビットはすべて連続したバイトにあります。そのため、新しい行ごとにあちこち飛ぶ必要がありません。これは良いことです。
4.  `ix` のループはさらに面白いです。まず、実際のピクセルの行を（ワードの）変数 `row` に読み込みます。ピクセルを書き込む必要があるかどうかを調べるには、単に特定のビットを確認します。しかし、パックが_リトル_エンディアンなので、2 つの近道ができます。  
    1 つ目は、ビットを下位から上位へたどるので、反復のたびに右にシフトしてビット 0 を調べればよいということです。その結果として、処理済みのビットは捨てられていきます。そして_このこと_から、`row` が 0 になれば、もうピクセルはなく、その行は終わりだということになります。この打ち切りは_三重_のループの一番内側で起きるので、速度の向上はかなりのものになりえます。

この関数は、文字列を画面に出すのに最低限必要なことしかしません。ゼロでないピクセルだけを描き（文字は透明）、端での折り返しもスクロールもありません。唯一の些細でない機能は、改行ができることです。改行すると、カーソルは画面上の元の x の位置に戻ります。

8 ビット用の関数もこれとほとんど同じです。「ほとんど」なのは、VRAM にはバイト単位で書き込めないという規則があるからです。明らかな違いは、ピッチと文字の間隔を半分にしなければならないことです。また、各文字の開始位置が偶数ピクセルの境界にあることを**必須**にしています。そうすれば、前とほぼ同じ内側のループが使えます。1 ピクセルの代わりに 2 ピクセルを処理するだけです。そう、これは場当たり的な手です。いいえ、気にしません。

```c{#cd-bm8-puts}
void bm8_puts(u16 *dst, const char *str, u8 clrid)
{
    int c, x=0, dx= gptxt->dx >> 1;

    while((c=*str++) != 0)
    {
        // <snip char-switch and iy loop>
                for(ix=x; row>0; row >>= 2, ix++)
                {
                    pxs= dst[iy*120+ix];
                    if(row&1)
                        pxs= (pxs&0xFF00) | clrid;
                    if(row&2)
                        pxs= (pxs&0x00FF) | (clrid<<8);

                    dst[iy*120+ix]= pxs;
                }
        // <snip>
    }
}
```

### インターフェース関数 {#ssec-bm-iface}

インターフェース関数は素直なものです。することは、内部のルーチンのために書き込み先の開始位置を用意することと、16 ビット版ではピッチを渡すことだけです。モード 3 は `vid_mem` を基準に使い、モード 4 と 5 はページフリップでも動くように `vid_page` を使います。`m4_puts()` は文字が偶数ピクセルから始まるようにもしています。それから、このルーチンは本当の色ではなく、色のインデックスを使うことを忘れないでください。

```c{#cd-mx-puts}
// Bitmap text interface. Goes in text.h
INLINE void m3_puts(int x, int y, const char *str, COLOR clr)
{    bm16_puts(&vid_mem[y*240+x], str, clr, 240);     }

INLINE void m4_puts(int x, int y, const char *str, u8 clrid)
{    bm8_puts(&vid_page[(y*240+x)>>1], str, clrid);   }

INLINE void m5_puts(int x, int y, const char *str, COLOR clr)
{    bm16_puts(&vid_page[y*160+x], str, clr, 160);    }
```

### テキストを消す {#ssec-bm-clrs}

テキストを消すのは、文字列を書き出すのとほぼ同じです。機能的な違いは、元の文字の代わりに、常に空白（というより、塗りつぶした長方形）を置くことだけです。それでも、行がどこまで続くのか、何行あるのかを知るために、文字列全体は必要です。

それを念頭に置けば、下の `bm16_clrs()` 関数はそれほど理解しにくくないはずです。要点は、文字列を読んで、文字列の各行のピクセル単位の長さ（`nx*gptxt->dx`）を求め、その長さと文字の高さ（`gptxt->dy`）で決まる長方形を塗りつぶすことです。すべてが計画どおりに進むようにするための帳尻合わせが少しありますが、結局はそれだけのことです。ほかのテキストの種類の消去ルーチンも同じなので、それらは示しません。

```c{#cd-bm16-clrs}
void bm16_clrs(u16 *dst, const char *str, COLOR clr, int pitch)
{
    int c, nx=0, ny;

    while(1)
    {
        c= *str++;
        if(c=='\n' || c=='\0')
        {
            if(nx>0)
            {
                nx *= gptxt->dx;
                ny= gptxt->dy;
                while(ny--)
                {
                    memset16(dst, clr, nx);
                    dst += pitch;
                }
                nx=0;
            }
            else
                dst += gptxt->dy*pitch;
            if(c=='\0')
                return;
        }
        else
            nx++;
    }
}
```

## タイルマップのテキスト {#sec-tile}

ある意味で、タイルモードのテキストは実はビットマップより簡単です。フォントをキャラブロックに詰め込んでしまえば、それ以降はフォントそのものを参照する必要がなくなるからです。可変幅のフォントにしたい場合は別で、その場合はビットシフト地獄に陥ります。でも私は固定幅で 1 タイルのフォントにこだわるので、話はとても単純なままです。

### タイルの初期化 {#ssec-tile-init}

最初にすべきことは、フォントを 4 ビットか 8 ビットに展開できるようにすることです。一番簡単なのは、`BitUnpack()` の呼び出しを準備して済ませることです。しかし、VBA でのその実装は、私が予定していた使い方に対しては完全には正しくありません（でした。今では直っているかもしれません）。そこで、自分で書くことにします。引数 `dstv` と `srcv` はそれぞれ書き込み先と読み込み元のアドレス（訳注: 原文は "source and destination" の順ですが、引数名に合わせました）、`len` は読み込み元のバイト数、`bpp` は書き込み先のビット深度です。`base` には 2 つの目的があります。主な目的は、ビット 31 が設定されていればすべてのピクセルに、クリアされていればゼロ以外のすべての値に加える数です。これで、読み込み元のビット深度が 1 の場合に得られる 0 と 1 だけでなく、より広い範囲の結果が得られます。それと、あとで説明するもう 1 つのちょっとした技にも使います。

```c{#cd-txt-bup}
// Note, the BIOS BitUnpack does exactly the same thing!
void txt_bup_1toX(void *dstv, const void *srcv, u32 len, int bpp, u32 base)
{
    u32 *src= (u32*)srcv;
    u32 *dst= (u32*)dstv;

    len= (len*bpp+3)>>2;    // # dst words
    u32 bBase0= base&(1<<31);    // add to 0 too?
    base &= ~(1<<31);

    u32 swd, ssh=32;    // src data and shift
    u32 dwd, dsh;       // dst data and shift
    while(len--)
    {
        if(ssh >= 32)
        {
            swd= *src++;
            ssh= 0;
        }
        dwd=0;
        for(dsh=0; dsh<32; dsh += bpp)
        {
            u32 wd= swd&1;
            if(wd || bBase0)
                wd += base;
            dwd |= wd<<dsh;
            swd >>= 1;
            ssh++;
        }
        *dst++= dwd;
    }
}
```

実際のマップのテキストの初期化は `txt_init_se()` で行います。最初の 2 つの引数は、予想どおりのものです。テキストに使う背景と、そこに設定する制御フラグ（キャラブロック、スクリーンブロック、ビット深度など、その手のもの）です。3 つ目の引数 `se0` は、展開の基準と同じように、パレットとタイルのインデックスの「基準」を示します。形式は普通のスクリーンエントリとまったく同じで、`se0`{0-9} がタイルのオフセット、`se0`{C-F} が 16 色のパレットバンクです。`clrs` にはテキストの色が入っていて、それはサブパレットで示されるパレットに入ります。そして 5 つ目の引数 `base` は、ビットの展開の基準です。

今のところは、`clrs` の_2 つ目_の色と、4 bpp のための余分なパレットへの書き込みは無視してください。おそらく知りたくはないでしょう。でも、どのみち[あとで](#ssec-demo-se1)説明します。

```c{#cd-txt-init-se}
void txt_init_se(int bgnr, u16 bgcnt, SB_ENTRY se0, u32 clrs, u32 base)
{
    bg_cnt_mem[bgnr]= bgcnt;
    gptxt->dst0= se_mem[BF_GET(bgcnt, BG_SBB)];

    // prep palette
    int bpp= (bgcnt&BG_8BPP) ? 8 : 4;
    if(bpp == 4)
    {
        COLOR *palbank= &pal_bg_mem[BF_GET(se0, SE_PALBANK)<<4];
        palbank[(base+1)&15]= clrs&0xFFFF;
        palbank[(base>>4)&15]= clrs>>16;
    }
    else
        pal_bg_mem[(base+1)&255]= clrs&0xFFFF;

    // account for tile-size difference
    se0 &= SE_ID_MASK;
    if(bpp == 8)
        se0 *= 2;

    // Bitunpack the tiles
    txt_bup_1toX(&tile_mem[BF_GET(bgcnt, BG_CBB)][se0],
        toncfontTiles, toncfontTilesLen, bpp, base);
}
```

いろいろなオフセットを扱いたくなければ、3 つ目と 5 つ目の引数はゼロのままでかまいません。ほかの引数をゼロのままにするのはおそらく良い考えではありませんが、この 2 つなら問題ありません。

### スクリーンエントリの書き込み処理 {#ssec-tile-puts}

これは、おそらくテキストの書き込み処理の中で最も単純なものです。スクリーンエントリ 1 つにつきグリフが 1 つなので、スクリーンブロックの正しい位置にハーフワードを 1 つ書けば、1 文字になります。これを文字列全体について繰り返します。

ただし、この実装についてはいくつか注意すべき点があります。まず、前と同じく、折り返しもスクロールもありません。それが欲しければ、全部自分でする必要があります。また、_x_ と _y_ の座標は、タイル単位ではなく、やはり_ピクセル_単位です。これは主にほかの書き込み処理との一貫性のためで、それ以上の理由はありません。そうそう、まだ気づいていなかったなら言っておくと、`gptxt->dst0` は `txt_init_se()` の中で、背景のスクリーンブロックの先頭を指すように初期化されています。最後に、実際のスクリーンエントリを作るのに `se0` を加えています。初期化でゼロでない `se0` を使ったなら、おそらくここでもそれを使いたいはずです。

```c
void se_puts(int x, int y, const char *str, SB_ENTRY se0)
{
    int c;
    SB_ENTRY *dst= &gptxt->dst0[(y>>3)*32+(x>>3)];

    x=0;
    while((c=*str++) != 0)
    {
        if(c == '\n')    // line break
        {    dst += (x&~31) + 32;    x=0;    }
        else
            dst[x++] = (gptxt->chars[c]) + se0;
    }
}
```

## スプライトのテキスト {#sec-obj}

スプライトのテキストはタイルマップのテキストと似ていますが、今度はスクリーンエントリの代わりに OBJ_ATTR を使います。位置は自分で設定する必要があり（属性 0 と 1）、属性 2 は通常のタイルマップのスクリーンエントリとほぼ同じです。初期化関数 `txt_init_obj()` は `txt_init_se()` と似ていますが、タイルマップの詳細が OAM のそれに置き換えられています。スクリーンブロックの代わりに基準の OBJ_ATTR `oe0` を指し、`attr2` は `se0` とほぼ同じように働きます。使うオブジェクトには常に 4bpp のタイルを使え、ほかのものを乱さずに済むので、コードは実はもっと単純です。

```c
// OAM text initializer
void txt_init_obj(OBJ_ATTR *oe0, u16 attr2, u32 clrs, u32 base)
{
    gptxt->dst0= (u16*)oe0;

    COLOR *pbank= &pal_obj_mem[BF_GET(attr2, ATTR2_PALBANK)<<4];
    pbank[(base+1)&15]= clrs&0xFFFF;
    pbank[(base>>4)&15]= clrs>>16;

    txt_bup_1toX(&tile_mem[4][attr2&ATTR2_ID_MASK], toncfontTiles,
        toncfontTilesLen, 4, base);
}
```

```c

// OAM text writer
void obj_puts(int x, int y, const char *str, u16 attr2)
{
    int c, x0= x;
    OBJ_ATTR *oe= (OBJ_ATTR*)gptxt->dst0;

    while((c=*str++) != 0)
    {
        if(c == '\n')    // line break
        {    y += gptxt->dy; x= x0; }
        else
        {
            if(c != ' ') // Only act on a non-space
            {
                oe->attr0= y & ATTR0_Y_MASK;
                oe->attr1= x & ATTR1_X_MASK;
                oe->attr2= gptxt->chars[c] + attr2;
                oe++;
            }
            x += gptxt->dx;
        }
    }
}
```

書き込み処理そのものの構造は、もうおなじみのはずです。`attr2` は、ここでもパレットの切り替えやタイルの開始位置のずらしができるように、基準のオフセットとして働きます。属性 0 と 1 には位置しか入れておらず、それ以外は何もしていないことに注意してください。そうできるのは、残りがすでに望みどおり、つまり 4bpp のタイルを使う飾り気のない 8x8p のスプライトに設定されているからです。たしかに、人によってはこれで困ることがあるかもしれませんが、すべてをきちんとマスクしたら、今度はほかのことで困ることになります。これは判断の問題なので、自由に反対して変えてください。

この書き込み処理は常に決まった OBJ_ATTR から始まり、それまでのものを上書きします。それは望ましくないかもしれないので、2 つ目のスプライトの書き込み処理 `obj_puts2` も用意しました。これは新しい基準となる OBJ_ATTR を引数に取ります。

```c
INLINE void obj_puts2(int x, int y, const char *str, u16 attr2, OBJ_ATTR *oe0)
{
    gptxt->dst0= (u16*)oe0;
    obj_puts(x, y, str, attr2);
}
```

メモリの使い方について、触れておくべき注意点がいくつかあります。OBJ_ATTR は 128 個しかないことを思い出してください。1 グリフにつき 1 エントリだと、たくさん使えば、とても手が出ないほど高くつくかもしれません。同じように、1024 タイルは多く思えるかもしれませんが、完全なアニメーションもいくつか入れていれば、すぐに足りなくなります。また、ビットマップモードでは 512 タイルしかないことも思い出してください。ビットマップモードで ASCII の文字一式を入れると、スプライトのタイルの*半分*を使ってしまいます！

ほんの数文字に使うだけなら問題になることはまずありませんが、画面いっぱいのテキストが欲しいなら、何か別のものを使ったほうがよいかもしれません。もちろん、これらを回避する方法はあります。かなり単純な方法さえあります。でも、それらは本当にゲームに依存するので、一般的な解決策を示すのは難しいのです。

## いくつかのデモ {#sec-demo}

### ビットマップのテキストのデモ {#ssec-demo-bm}

“Hello world” から始めてもいいのですが、それはかなり退屈なので、もっと面白いものから始めようと思いました。`txt_bm` デモは `bm_modes` と似たことをします。画面に何かを表示し、モード 3、4、5 を切り替えて違いを見られるようにするのです。ただし今回は、ビットマップ用の `puts()` を使って、現在のモードを示す文字列を実際に書きます。それでもまだかなり退屈なので、動かせるカーソルを画面に置いて、その座標も書き出すことにします。完全なコードは次のとおりです。

```c{#cd-txt-bm}
#include <stdio.h>
#include <tonc.h>

#define CLR_BD    0x080F

const TILE cursorTile=
{{  0x0, 0x21, 0x211, 0x2111, 0x21111, 0x2100, 0x1100, 0x21000  }};

void base_init()
{
    vid_page= vid_mem;

    // init interrupts
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);

    // init backdrop
    pal_bg_mem[0]= CLR_MAG;
    pal_bg_mem[CLR_BD>>8]= CLR_BD;
    pal_bg_mem[CLR_BD&255]= CLR_BD;
    m3_fill(CLR_BD);

    // init mode 4 pal
    pal_bg_mem[1]= CLR_LIME;
    pal_bg_mem[255]= CLR_WHITE;

    // init cursor
    tile_mem[5][0]= cursorTile;
    pal_obj_mem[1]= CLR_WHITE;
    pal_obj_mem[2]= CLR_GRAY;
}

int main()
{
    base_init();

    txt_init_std();

    // (1) print some string so we know what mode we're at
    m3_puts( 8,  8, "mode 3", CLR_CYAN);
    m4_puts(12, 32, "mode 4", 1);
    m5_puts(16, 40, "mode 5", CLR_YELLOW);

    // init variables
    u32 mode=3, bClear=0;
    OBJ_ATTR cursor= { 80, 120, 512, 0 };

    // init video mode
    REG_DISPCNT= DCNT_BG2 | DCNT_OBJ | 3;

    // init cursor string
    char str[32];
    siprintf(str, "o %3d,%3d", cursor.attr1, cursor.attr0);

    while(1)
    {
        VBlankIntrWait();
        oam_mem[0]= cursor;
        key_poll();

        if(key_hit(KEY_START))
            bClear ^= 1;

        // move cursor
        cursor.attr1 += key_tri_horz();
        cursor.attr0 += key_tri_vert();

        // adjust cursor(-string) only if necessary
        if(key_is_down(KEY_ANY))
        {
            // (2) clear previous coords
            if(bClear)
                bm_clrs(80, 112, str, CLR_BD);

            cursor.attr0 &= ATTR0_Y_MASK;
            cursor.attr1 &= ATTR1_X_MASK;
            // (3) update cursor string
            siprintf(str, "%c %3d,%3d", (bClear ? 'c' : 'o'),
                cursor.attr1, cursor.attr0);
        }

        // switch modes
        if(key_hit(KEY_L) && mode>3)
            mode--;
        else if(key_hit(KEY_R) && mode<5)
            mode++;
        REG_DISPCNT &= ~DCNT_MODE_MASK;
        REG_DISPCNT |= mode;

        // (4) write coords
        bm_puts(80, 112, str, CLR_WHITE);
    }

    return 0;
}
```

<div class="cpt_fr" style="width:240px;">
<img src="img/demo/txt_bm.png" alt="" id="fig:img-txt-bm"><br>
<b>@fig:img-txt-bm</b>: <tt>txt_bm</tt> デモ。
</div>

操作:

<table>
<tbody valign="top">
<tr><th>D-pad<td>カーソルを動かす。
<tr><th>Start<td>文字列の消去を切り替える。
<tr><th>L, R<td>モードを減らす、増やす。
</tbody>
</table>

ここにある多くのことは、説明不要か、あまり重要ではないかのどちらかのはずです。面白いところには番号を付けてあるので、順に見ていきましょう。

**1. モードの表示**。ここでは、モードを示す 3 つの文字列を VRAM に書き込みます。インターフェースがほとんど同じであることに注意してください。本当の違いは、`m4_puts()` の 4 つ目の引数が本当の色ではなく、パレットのインデックスであることだけです。

**2. 前のカーソルの文字列を消す**。カーソルの文字列は、画面上を動くカーソルを追いかけます。最初に気づくのは、ビットマップの書き込み処理はフォントの_ゼロでない_ピクセルしか書かないので、文字列がひどくぐちゃぐちゃになることです。言い換えると、そのグリフに割り当てられた場所の残りを消さ_ない_のです。要するに、`mx_puts()` は透明な文字列の書き込み処理なのです。

もちろん、グリフの領域全体を消すスイッチを書き込み処理に付けることもできました。実際、とても簡単で、`else` 節を 1 つ追加するだけです。しかし、今のやり方のほうが実際には実用的です。1 つには、本当に透明に_したい_場合はどうするのでしょうか？ そのためだけに別のルーチンを書かなければならなくなります。私が選んだのは、別に消去のルーチン（どのみち必要になるでしょう）を用意する方法です。グリフ全体を上書きするには、先に `mx_clrs()` を呼ぶだけです。ここではそうしています。まあ、`bClear` 変数が設定されている（Start で切り替えます）間だけですが。

2 つ目の理由は、この方法のほうがずっと速いことです。領域全体を消さなければならないとすると、`ix` のループを早めに打ち切ることができなくなりますし、余分な分岐があるだけで（三重のループの中で）サイクルが増えます。それだけでなく、個々の文字を描くのは、ブロック全体をまとめて処理するより常に遅くなります。`mx_clrs()` は `memset16()` を使っています。これは基本的に安全策を加えた `CpuFastSet()` で、ほんの半ダースほどのピクセルで、もう速くなります。

そうそう、コードには `bm_clrs()` と書いてあるのに、なぜ `mx_clrs()` の話をしているのかと思ったなら、前者の関数は、現在のビットマップのモードで switch ブロックを使って、モードごとの正しい文字列の消去処理を呼ぶだけの関数です。

**3. カーソルの文字列を更新する**。書き込み処理には書式指定のフィールドがないのに、どうやって数値を書けばいいのでしょうか？ 簡単です。先に `sprintf()` で文字列を用意して、代わりにそれを使えばいいのです。というより、`siprintf()` を使います。これは `sprintf()` の整数専用の版で、どのみち浮動小数点数は使わないことになっている GBA のプログラミングにはこちらのほうが向いています。`siprintf()` と `mx_puts()` を包む関数を作るのは比較的簡単なはずですが、その手間に見合うかどうかはわかりません。

`siprintf` など、数値を文字列に変換できるルーチンは、そのために 10 による除算を使うことは指摘しておくべきかもしれません。それが何を意味するかはわかりますね。それに、数値の変換を頼まなくても、標準ライブラリのルーチンを十数個ほど呼び出すので、バイナリが 25kb ほど増えます。ROM にとっては大したことではありませんが、マルチブートのもの（最大 256kb）では問題になるかもしれません。それを踏まえて、[Dan Posluns](https://www.danposluns.com/gbadev/) の **posprintf** を見てみてください。これは 10 進数への変換に特別なアルゴリズムを使った、手書きのアセンブリです。`siprintf()` ほどオプションは豊富ではないかもしれませんが、速さでも小ささでも大差で勝っているので、ぜひ調べてみる価値があります。

**4. カーソルの文字列を書く**。現在のカーソルの文字列を位置 (80, 120) に書きます。文字列を消す場合と同じように、現在のモードの書き込み処理を切り替える `bm_puts()` 関数を使っています。

### スプライトのテキスト。Hello world！ {#ssec-demo-obj}

そうです！ Hello world です！ さて、原理的には `txt_init()`、`txt_init_obj()` を呼んで、それから正しいパラメータで `obj_puts()` を呼ぶだけでいいのですが、それもやはり退屈なので、面白いことも加えます。`txt_obj` デモは、スプライトで一番うまくできることの 1 つ、つまり文字ごとのアニメーションを示します。“hello world!” という語句の文字が画面の上から落ちてきて、床（画面の真ん中あたりの緑の線）で跳ねて止まります。

```c{#cd-txt_obj}
#include <tonc.h>

// === CONSTANTS & STRUCTS ============================================

#define POS0 (80<<8)
#define GRAV 0x40
#define DAMP 0xD0
#define HWLEN 12

const char hwstr[]= "Hello world!";

typedef struct
{
    u32 state;
    int tt;
    FIXED fy;
    FIXED fvy;
} PATTERN;

// === FUNCTIONS ======================================================

void pat_bounce(PATTERN *pat)
{
    if(pat->tt <= 0)    // timer's run out: play pattern
    {
        pat->fvy += GRAV;
        pat->fy += pat->fvy;

        // touched floor: bounce
        if(pat->fy > POS0)
        {
            // damp if we still have enough speed
            // otherwise kill movement
            if(pat->fvy > DAMP)
            {
                pat->fy= 2*POS0-pat->fy;
                pat->fvy= DAMP-pat->fvy;
            }
            else
            {
                pat->fy= POS0;
                pat->fvy= 0;
            }
        }
    }
    else    // still in waiting period
        pat->tt--;
}

int main()
{
    REG_DISPCNT= DCNT_MODE3 | DCNT_BG2 | DCNT_OBJ;

    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    memset16(&vid_mem[88*240], CLR_GREEN, 240);

    // (1) init sprite text
    txt_init_std();
    txt_init_obj(&oam_mem[0], 0xF200, CLR_YELLOW, 0xEE);
    // (2) 12 px between letters
    gptxt->dx= 12;

    // (3) init sprite letters
    OBJ_ATTR *oe= oam_mem;
    obj_puts2(120-12*HWLEN/2, 8, hwstr, 0xF200, oe);

    int ii;
    PATTERN pats[HWLEN];

    for(ii=0; ii<HWLEN; ii++)
    {
        // init patterns
        pats[ii].state=0;
        pats[ii].tt= 3*ii+1;
        pats[ii].fy= -12<<8;
        pats[ii].fvy= 0;

        // init sprite position
        oe[ii].attr0 &= ~ATTR0_Y_MASK;
        oe[ii].attr0 |= 160;
    }

    while(1)
    {
        VBlankIntrWait();

        for(ii=0; ii<HWLEN; ii++)
        {
            pat_bounce(&pats[ii]);

            oe[ii].attr0 &= ~ATTR0_Y_MASK;
            oe[ii].attr0 |= (pats[ii].fy>>8)& ATTR0_Y_MASK;
        }
    }

    return 0;
}
```

<div class="cpt_fr" style="width:240px;">
  <img id="fig:img-txt-obj" src="img/demo/txt_oe.png" />

**{@fig:img-txt-obj}**: `txt_obj` デモ。

</div>

このコードのうち、文字列そのものに関係しているのはごく一部、つまり項目 1、2、3 だけです。基本の初期化のための `txt_init_std()` の呼び出しと、スプライトのテキストの初期化関数 `txt_init_obj()` の呼び出しがあります。2 つ目の引数は属性 2 の基準です（属性 2 が何だったか覚えていなければ、もう一度[スプライト](regobj.html#ssec-obj-attr2)の章を見てください）。`0xF200` は、サブパレット 15 を使い、文字のタイルをタイルのインデックス 512 から始める（ビットマップモードなので）という意味です。フォントの色は黄色で、インデックス 255 にあります。パレットバンクから 240、展開から `0x0E`=14、実際の 1bpp のピクセルから 1 で、240+14+1=255 です。この呼び出しのあとで、文字を少し広げるために、横方向のピクセルのオフセットも 12 に設定しています。そのあとは、`obj_puts2()` を呼んで OAM の最初の数個のスプライトを準備し、“hello world!” が画面の上部の中央に表示されるようにするだけです。

ここで終わりにすることもできましたが、デモは実はまだ始まったばかりです。スプライトをグリフとして使うことの良いところは、それらがまだ普通のスプライトとして_ふるまえる_ことです。`obj_puts()` は、スプライトらしいグラフィックの代わりに文字を使うように準備するだけなのです。

#### ぴょんぴょんぴょん

ここでの目標は、文字を画面の上から落とし、床に当たったら、摩擦などのせいで前より少し遅い速さで、また上に跳ね返らせることです。物理的には、落ちる部分は一定の加速度 _g_ を使って行います。加速度は速度の変化なので、速度は線形になります。速度は位置の変化なので、高さは放物線になります。跳ね返るところでは<dfn>非弾性衝突</dfn>、つまりエネルギーが失われる衝突をさせます。原理的には、これは衝突の前後の速度の 2 乗の差が一定になる（ \|**v**<sub>out</sub>\|<sup>2</sup> - \|**v**<sub>in</sub>\|<sup>2</sup> = Q ）ことを意味します。しかし、そうすると新しい速度を求めるのに平方根が必要になり、今はそれを気にしたくないので、ここでは 2 乗を捨ててしまいます。これが実際にかなり妥当な状況もあるはずです <kbd>:P</kbd>。さらに単純化して、位置には 1 次の積分を使います。これで、動きの基本的なコードはとても単純になります。

```c
// 1D inelastic reflections
// y, vy, ay: position, velocity, acceleration.
// Q: inelastic collision coefficient.
vy += ay;
 y += vy;
if(y>ymay)  // collision
{
    if((ABS(vy)>Q)
    {
        vy= -(vy-SGN(vy)*Q);  // lower speed, switch direction
        y= 2*ymay-y;          // Mirror y at r: y= r-(y-r)= 2r-y
    }
    else  // too slow: stop at ymay
    {   vy= 0; y= ymay; }
}
```

これは、2 次の積分と「正しい」反発を使った、次のより正確なコードに置き換えることもできますが、積分の改良はほとんど目に見えません。実は、平方根を使うものより、単純な線形の反発のほうが見た目は好きです。

```c
// accelerate
 k= vx+GRAV;
// Trapezium integration rule:
//   x[i+1]= x[i] + (v[i]+v[i+1])/2;
 x += (vx+k)/2;
vx= k;
if(x>xmax)  // collision
{
    if(vx*vx > Q2)
    {   vx= -Sqrt(vx*vx-Q2); x= 2*xmax-x; }
    else
    {   vx= 0;               x= xmax;     }
}
```

### マップのテキスト: 色と枠線 {#ssec-demo-se1}

次は、2 つあるマップのテキストのデモの 1 つ目です。私が通常の背景と呼んでいるものの公式の名前は「テキスト背景」で、そう呼ばれるのには理由があります。テキストがある場合、ほとんどは通常の背景を使って表示されるからです。もちろん、ほとんどの場合はほかのすべても_また_通常の背景で表示されるので、厳密に言えば「テキスト」と結びつけるのは名前として不適切ですが、今日のところは大目に見ましょう。1 つ目のデモは、テキストの関数をいろいろな効果に使う方法についてです。単にテキストを表示する（退屈）だけでなく、パレットの切り替えやテキストを枠で囲むこと、そして異なるフォントと枠線を同時に簡単に使う方法を見ていきます。私の関数の設計のおかげで、これにはパラメータを 1 つ変えるだけで済みます。かっこいいでしょう？

このデモでは、モノクロのフォントに影を付けることと、不透明な背景を付けることも見せます。さて、私がこれからやるやり方は、おそらくコンピュータサイエンス地獄に席を確保することになるでしょうが、まあ、技のかっこよさのおかげで、そこで焼かれずに済むかもしれません。

```c{#cd-txt_se1}
#include <tonc.h>
#include "border.h"

// === CONSTANTS & STRUCTS ============================================

#define TID_FRAME0        96
#define TID_FRAME1       105
#define TID_FONT           0
#define TID_FONT2        128
#define TID_FONT3        256
#define TXT_PID_SHADE   0xEE
#define TXT_PID_BG      0x88

// === FUNCTIONS ======================================================

void init()
{
    int ii;
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    irq_init(NULL);
    irq_add(II_VBLANK, NULL);

    txt_init_std();

    // (1a) Basic se text initialization
    txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0x1000, CLR_RED, 0x0E);

    // (1b) again, with a twist
    txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0xF000|TID_FONT2,
        CLR_YELLOW | (CLR_MAG<<16), TXT_PID_SHADE);

    // (1c) and once more, with feeling!
    txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0xE000|TID_FONT3,
        0, TXT_PID_SHADE);
    u32 *pwd= (u32*)&tile_mem[0][TID_FONT3];
    for(ii=0; ii<96*8; ii++)
        *pwd++ |= quad8(TXT_PID_BG);

    // extra border initialisation
    memcpy32(pal_bg_mem, borderPal, borderPalLen/4);
    memcpy32(&tile_mem[0][TID_FRAME0], borderTiles, borderTilesLen/4);

    // (2) overwrite /\ [] `% ^_ to use border tiles
    //  / ^ \
    //  [ # ]
    //  ` _ '
    const u8 bdr_lut[9]= "/^\\[#]`_\'";
    for(ii=0; ii<9; ii++)
        gptxt->chars[bdr_lut[ii]]= TID_FRAME0+ii;

    // (3) set some extra colors
    pal_bg_mem[0x1F]= CLR_RED;
    pal_bg_mem[0x2F]= CLR_GREEN;
    pal_bg_mem[0x3F]= CLR_BLUE;

    pal_bg_mem[0xE8]= pal_bg_mem[0x08]; // bg
    pal_bg_mem[0xEE]= CLR_ORANGE;   // shadow
    pal_bg_mem[0xEF]= pal_bg_mem[0x0F]; // text
}

void txt_se_frame(int l, int t, int r, int b, u16 se0)
{
    int ix, iy;
    u8 *lut= gptxt->chars;
    u16 *pse= (u16*)gptxt->dst0;
    pse += t*32 + l;
    r -= (l+1);
    b -= (t+1);

    // corners
    pse[32*0  + 0] = se0+lut['/'];
    pse[32*0  + r] = se0+lut['\\'];
    pse[32*b  + 0] = se0+lut['`'];
    pse[32*b  + r] = se0+lut['\''];

    // horizontal
    for(ix=1; ix<r; ix++)
    {
        pse[32*0+ix]= se0+lut['^'];
        pse[32*b+ix]= se0+lut['_'];
    }
    // vertical + inside
    pse += 32;
    for(iy=1; iy<b; iy++)
    {
        pse[0]= se0+lut['['];
        pse[r]= se0+lut[']'];
        for(ix=1; ix<r; ix++)
            pse[ix]= se0+lut['#'];
        pse += 32;
    }
}

int main()
{
    init();

    // (4a) red, green, blue text
    se_puts(8, 16, "bank 1:\n  red",   0x1000);
    se_puts(8, 40, "bank 2:\n  green", 0x2000);
    se_puts(8, 72, "bank 3:\n  blue",  0x3000);
    // (4b) yellow text with magenta shadow
    se_puts(8, 96, "bank 15:\n yellow, \nwith mag \nshadow", 0xF000|TID_FONT2);

    // (5a) framed text, v1
    txt_se_frame(10, 2, 29, 9, 0);
    se_puts( 88, 24, "frame 0:", 0);
    se_puts(104, 32, "/^\\[#]`_'", 0);
    se_puts( 88, 40, "bank  0:\n  basic text,\n  transparent bg", 0);

    // (5b) framed text, v2
    txt_se_frame(10, 11, 29, 18, TID_FRAME1-TID_FRAME0);
    se_puts( 88,  96, "frame 1:", 0xE000|TID_FONT3);
    se_puts(104, 104, "/^\\[#]`_'", 9);
    se_puts( 88, 112, "bank 14:\n  shaded text\n  opaque bg", 0xE000|TID_FONT3);

    while(1)
        VBlankIntrWait();
    return 0;
}
```

<div class="lblock">
  <table id="fig:img-txt-se1">
    <tr>
      <td>
        <div class="cpt" style="width:240px">
          <img src="img/demo/txt_se1.png" alt="txt_se1 demo"><br>
          <b>{@fig:img-txt-se1}a</b>: 1 つ目のマップのテキストのデモ。
        </div>
      <td>
        <div class="cpt" style="width:256px">
          <img src="img/demo/txt_se1_tiles.png" alt="txt_se1 tileset"><br>
          <b>{@fig:img-txt-se1}b</b>: それに対応するタイルセット。
        </div>
  </table>
</div>

#### コードの概要

@fig:img-txt-se1 は、このコードが生み出すものを示しています。実際のテキストの描画はすべて main 関数で行っているので、1 つずつ見ていきます。最初の 3 つは赤、緑、青のテキストで（ポイント 4a）、パレットの切り替えで実現しています。赤、緑、青をパレットのインデックス `0x1F`、`0x2F`、`0x3F` に読み込んでおき（ポイント 3）、`se_puts()` の最後のパラメータで切り替えられるようにしています。覚えているでしょうが、このパラメータは各スクリーンエントリに加えられます。値 `0x1000`、`0x2000`、`0x3000` は、それぞれパレットバンク 1、2、3 を使うという意味です。

よく見ると、4 つ目のテキスト（ポイント 4b）は黄色で、各文字の右端にマゼンタ（いいえ、ピンクではありません。_マゼンタ_です）の影が付いているのがわかるでしょう。少なくともその一部は `se0` パラメータでやっていて、今回は `0xF080` です。影が付いているのは最後の部分のおかげです。実は、タイル 128 から始まる、少し違うフォントを使っているのです。繰り返しますが、これをすべて同じ関数でできるのは、`se_puts()` のオフセットのパラメータのおかげです。

ポイント (5a) と (5b) は、枠と、その中のテキストのためのものです。関数 `txt_se_frame()` が私の枠線を描きます。長方形を入力として受け取り、そこに枠を描きます。枠は左上を含み、右下を含まないことに注意してください。ここでも、オフセットとして余分な `se0` パラメータが 1 つあります。2 つ目の枠線は実際にこうやって描いています。枠線のタイルの差だけずらしているだけです。

枠線そのものは、実はほとんどテキストと同じように描いています。`init()` で、文字のルックアップテーブルの 9 文字を、主な枠線のタイルセットのタイルのインデックスを使うように割り当て直しています（ポイント 2）。そうしている特別な理由はなく、単にできるからです。テキストの書き込み処理と、ルックアップテーブルのちょっとした巧みな操作で何ができるかを示しているだけです。

枠の中のテキストも面白い話です。1 つ目の枠の中のテキストからわかるように、標準のテキストはうまくいきません。問題は、私が使っている主なタイルセットは透明なのに、枠の背景は透明ではないことです。2 つを混ぜると、ぶつかり合います。では、どう解決すればいいのでしょうか？ そう、背景の色が 0 ではない_別の_フォントを作るのです。その方法はいくつかあり、その 1 つはビットの展開のフラグに 1\<\<31 を加えることです。でも私は別の方法を選びます。それはあとで説明します。私が何をしているにせよ、ちゃんとうまくいっていることに注目してください。2 つ目の枠の中のテキストは、たしかに不透明です。このテキストはパレットバンク 14 を使って書いていて、今度はフォントに_3 つ目_のタイルセットを使っていることに注意してください。

さて、ここまではかなり簡単でした。`se_puts()` と `txt_se_frame()` の使い方のことです。ここまでのことをすべて理解できたことを願います。残りはかなり面白くなるからです。「ああ神様、ああ神様、みんな死んじゃう」というほどの面白さではありませんが、人によっては少しばかり身の毛がよだつかもしれません。

#### ビットいじりの楽しみ

3 つの異なるフォントを使っていると言いました。でもコードをよく調べても、フォントの定義やコピーの跡はどこにも見つからないでしょう。何もないからです。すべて、先ほど示したのと同じビットパックされたフォントがもとになっています。また、数学の素養のある人は、1bpp のフォントをビットパックすると 2 色になることに気づいたでしょう。そもそもそれが 1bpp の_意味_なのですから。でも、私のものには背景の色、前景の色、そして影があります。3 色です。さらに、影を付けるコードはどこにもないようです。これらはすべて、1 つの単純な疑問につながります。つまり、いったい私は何をしているのか？

ええと……これです。

```c
#define TID_FONT           0
#define TID_FONT2        128
#define TID_FONT3        256
#define TXT_PID_SHADE   0xEE
#define TXT_PID_BG      0x88

// (1a) Basic se text initialization
txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0x1000, CLR_RED, 0x0E);

// (1b) again, with a twist
txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0xF000|TID_FONT2,
    CLR_YELLOW | (CLR_MAG<<16), TXT_PID_SHADE);

// (1c) and once more, with feeling!
txt_init_se(0, BG_CBB(0) | BG_SBB(31), 0xE000|TID_FONT3,
    0, TXT_PID_SHADE);
u32 *pwd= (u32*)&tile_mem[0][TID_FONT3];
for(ii=0; ii<96*8; ii++)
    *pwd++ |= quad8(TXT_PID_BG);
```

この 6 つの文で、影と不透明さも含めて、3 つのフォントを準備しています。1 つ目は標準のフォントを準備しています。キャラブロック 0、スクリーンブロック 31、パレットバンク 1 で、ビットの展開のオフセットに `0x0E` を使うので、テキストの色は `0x1F` になります。オブジェクトのテキストでも同じものを見ました。

<div class="cpt_fr">
<table id="tbl:bupshade">
<caption align="bottom">
  <b>*@tbl:bupshade</b>: 
  基準 <code>0xEE</code> でのビットの展開。
</caption>
<tbody align="center">
  <tr>
    <td class="bdrLL" rowspan=9>
    <th>ビット 
    <th class="bdrL">値
    <td class="bdrRR" rowspan=9>
    <td class="bdrLL">
    <th> 7 <th> 6 <th> 5 <th> 4 <th> 3 <th> 2 <th> 1 <th> 0
    <td class="bdrRR">
  <tr>
    <th>0	<td class="bdrL"> 0
    <td class="bdrLL" rowspan=8>
    <td> . <td> . <td> . <td> . <td> . <td> . <td> . <td> 0
    <td class="bdrRR" rowspan=8>
  <tr>
    <th> 1	<td class="bdrL"> 1
    <td> . <td> . <td> . <td> . <td> . <td> E <td> F <td> .
  <tr>
    <th> 2	<td class="bdrL"> 1
    <td> . <td> . <td> . <td> . <td> E <td> F <td> . <td> .
  <tr>
    <th> 3	<td class="bdrL"> 1
    <td> . <td> . <td> . <td> E <td> F <td> . <td> . <td> .
  <tr>
    <th> 4	<td class="bdrL"> 0
    <td> . <td> . <td> . <td> 0 <td> . <td> . <td> . <td> .
  <tr>
    <th> 5	<td class="bdrL"> 0
    <td> . <td> . <td> 0 <td> . <td> . <td> . <td> . <td> .
  <tr>
    <th> 6	<td class="bdrL"> 1
    <td> E <td> F <td> . <td> . <td> . <td> . <td> . <td> .
  <tr>
    <th> 7	<td class="bdrL"> 0
    <td> 0 <td> . <td> . <td> . <td> . <td> . <td> . <td> .
<tr>
  <td colspan=4> OR: 
    <td class="bdrLL">
    <th> E <th> F <th> 0 <th> E <th> F <th> F <th> F <th> 0
    <td class="bdrRR">
</tbody>
</table>
</div>

`txt_se_init()` の 2 回目の呼び出しは、2 つ目のフォント、つまり影付きのものを準備しています。`se0` はパレットバンク 15 を使って 128 から始めることを示していますが、大事なことは `clrs` と `base` のパラメータで起きています。今回は `clrs` に黄色とマゼンタの 2 色が入っています。下位のハーフワードがテキストの色、上位のハーフワードが影の色になります。

実際に影が付くのは、`base` の値が `0xEE` であることと、ビットの展開のルーチン全体の働き方のおかげです。オフセットはパックされたフォントの「オン」の各ビットに加えられて `0xEF` になり、それが適切なシフトで現在のワードに OR されます。4bpp のフォントを扱っているので、結果は実際には次の 4 ビットにあふれ出します。さて、次のビットもオンなら、`0xEF` とあふれた値 `0x0E` が OR されます。`0xF` \| `0xE` は単に `0xF` なので、あふれは起きなかったのと同じです。しかし、次のビットが_オフ_なら、そのピクセルの値は `0xE` になります。最後に、読み込み元のビットがゼロであふれもなければ、結果は 0 です。これで 3 つの値が得られます。0（背景）、14（影）、15（テキスト）です。@tbl:bupshade は、この手順をもっと図で示したものです。左が読み込み元のバイトのビットで、右のグリッドに、各ビットを展開した結果を正しい位置に置いています。それらを OR すると最終的な結果になります。`0x46` なら、ワード `0xEF0EFFF0` になります。1 ワードは 4bpp のタイルの 8 ピクセルの 1 行で、下位の 4 ビットが一番左のピクセルなので、影は上位のビットを使っているにもかかわらず、文字の右側に付きます。

基準 `0xEE` は、この技がうまくいく多くの値の 1 つです。大事なのは、上位の 4 ビットが、下位の 4 ビット+1 で完全に上書きされなければならないことです。上位と下位の 4 ビットが等しく偶数の数なら、どれでもうまくいきます。

さて、これが場当たり的な技だということは、真っ先に認めます。うまくいくには、たくさんのことが噛み合わなければなりません。ワードの大きさがタイルの 1 行全体を収められること、パックされたデータも展開したデータもビットとバイトの両方の順序でリトルエンディアンであること、展開のルーチンが実際にあふれを許すこと、そして今は思いつかないほかのこともいくつかあるでしょう。これらの条件は GBA ではすべて満たされていますが、ほかのシステムでこの技が使えるかは大いに疑問です。もちろん、影を付けるほかの方法、それももっと良い方法はあります。ただ、これがあまりにおいしく意地悪なので、使わずにはいられないのです。

最後の `txt_se_init()` も、2 つ目とほぼ同じように、あふれによって影を付けます。違うのは、タイルを不透明にすることです。BitUnpack でそうすることもできますが、1 回の呼び出しでそれ_と_影の両方を得ることはできません。単純にうまくいかないのです。でも、ほかの方法があります。タイルを不透明にするのに本当に必要なのは、背景のピクセルにゼロ以外の何らかの値を入れることだけです。それなら簡単です。すべてをある数だけずらせば（加えるか OR すれば）いいのです。この場合はテキストの値がすでに最大なので値を加えることはできず、ここでは OR を使います。OR する値は `0x88888888` で、これはテキストや影を変えずに、背景のピクセルを `8` にするので、望みどおりになります。

と、まあ、こうやってやるのです。少なくとも、_私_はこうやっています。上の話がちんぷんかんぷんに思えても、同じやり方をしろと強制する人はいません。手持ちのものから組み立てる代わりに、複数のフォントをプログラムに入れるという簡単な道を選ぶことはいつでもできます。私はただ、少し工夫したコードで何ができるかを見せているだけです。

### マップのテキスト: プロファイリング {#ssec-demo-se2}

最後にお見せするのは簡単なものですが、いくつかのものを最適化するときに役立つかもしれません。まだ気づいていなかったなら言っておくと、GBA のプログラムのデバッグは PC のプログラムのデバッグほど簡単ではありません。Insight と GDB（GCC のデバッガ）でデバッグすることもできますが、それでも心もとないと聞いています。でも、自分でテキストを表示できるようになったので、少なくともそれに似たことはできます。診断のメッセージなどを書き出すのです。

でも、今からお見せするのはそれではありません。最後のデモでは、普通はデバッグの_後_に来るもの、つまりプロファイリングのやり方を示します。プロファイリングは、何にどれだけの時間が費やされているかを教えてくれるので、どこを最適化してみるのが一番良いかがわかります。お見せするのは、関数の中で費やされた時間を得る単純な方法です。こういうことを知っておくのは良いことです。特に、速さや効率といった、その手のくだらないことをまだ気にしなければならないこのようなプラットフォームではなおさらです。

次のデモでは、データをコピーする 5 つの方法の時間を計ります。この場合は、モード 4 のビットマップを EWRAM（私のコードはデフォルトでマルチブート用に設定されていて、すべてが ROM ではなく EWRAM に置かれます）から VRAM にコピーします。方法は次のとおりです。

- **u16 の配列**。16 ビット（ハーフワード）単位でコピーします。おそらくほかのチュートリアルで一番よく見かけるものですが、ここでは違います。理由はすぐにわかります。
- **u32 の配列**。32 ビット（ワード）単位でコピーします。
- **`memcpy()`**。標準 C のコピーのルーチンで、前のデモで使っているものです。まあ、今はそうしています。
- **`memcpy32()`**。自家製のアセンブリで、[ここ](asm.html#sec-cpy)で詳しく説明しています。基本的に `CpuFastSet()` と同じことをしますが、ワード数が 8 の倍数でなければならないという制約がありません。
- **`dma_memcpy()`**。32 ビットの DMA でコピーします。

```c{#cd-txt-se2}
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

#include <tonc.h>

#include "gba_pic.h"

// === CONSTANTS & STRUCTS ============================================

int gtimes[5];

const char *strs[5]=
{   "u16 array", "u32 array", "memcpy", "memcpy32", "DMA32" };

// === FUNCTIONS ======================================================

// copy via u16 array
void test_0(u16 *dst, const u16 *src, u32 len)
{
    u32 ii;
    profile_start();
    for(ii=0; ii<len/2; ii++)
        dst[ii]= src[ii];
    gtimes[0]= profile_stop();
}

// copy via u32 array
void test_1(u32 *dst, const u32 *src, u32 len)
{
    u32 ii;
    profile_start();
    for(ii=0; ii<len/4; ii++)
        dst[ii]= src[ii];
    gtimes[1]= profile_stop();
}

// copy via memcpy
void test_2(void *dst, const void *src, u32 len)
{
    profile_start();
    memcpy(dst, src, len);
    gtimes[2]= profile_stop();
}

// copy via my own memcpy32
void test_3(void *dst, const void *src, u32 len)
{
    profile_start();
    memcpy32(dst, src, len/4);
    gtimes[3]= profile_stop();
}

// copy using DMA
void test_4(void *dst, const void *src, u32 len)
{
    profile_start();
    dma3_cpy(dst, src, len);
    gtimes[4]= profile_stop();
}

int main()
{
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    irq_init(NULL);
    irq_add(II_VBLANK, NULL);

    test_0((u16*)vid_mem, (const u16*)gba_picBitmap, gba_picBitmapLen);
    test_1((u32*)vid_mem, (const u32*)gba_picBitmap, gba_picBitmapLen);
    test_2(vid_mem, gba_picBitmap, gba_picBitmapLen);
    test_3(vid_mem, gba_picBitmap, gba_picBitmapLen);
    test_4(vid_mem, gba_picBitmap, gba_picBitmapLen);

    // clear the screenblock I'm about to use
    memset32(&se_mem[7], 0, SBB_SIZE/4);

    // init map text
    txt_init_std();
    txt_init_se(0, BG_SBB(7), 0, CLR_YELLOW, 0);

    // print results
    int ii;
    char str[32];
    for(ii=0; ii<5; ii++)
    {
        siprintf(str, "%12s %6d", strs[ii], gtimes[ii]);
        se_puts(8, 8+8*ii, str, 0);
    }

    while(1)
        VBlankIntrWait();

    return 0;
}
```

コードは説明不要でしょう。プロファイリングしたいものごとに 5 つの関数を用意しました。関数を分けたのは、そうすれば最適化が邪魔をしない（最適化はコードを移動させることがあります）とわかっているからです。これらの関数を実行したあとで、テキストの関数を準備して結果を表示します。

プロファイリングそのものには、`profile_start()` と `profile_stop()` という 2 つのマクロを使います。これらは libtonc の `core.h` にあります。マクロは、タイマー 2 と 3 を開始、停止して、呼び出しの間の時間を返します。つまり、プロファイリングするコードではこれらのタイマーを使えないということです。

```c
INLINE void profile_start()
{
    REG_TM2D= 0;    REG_TM3D= 0;
    REG_TM2CNT= 0;  REG_TM3CNT= 0;
    REG_TM3CNT= TM_ENABLE | TM_CASCADE;
    REG_TM2CNT= TM_ENABLE;
}

INLINE u32 profile_stop()
{
   REG_TM2CNT= 0;
   return (REG_TM3D<<16)|REG_TM2D;
}
```

<div class="lblock">
<table id="fig:img-txt-se2">
<tr>
<td>
  <div class="cpt" style="width:240px">
  <img src="img/demo/txt_se2_vba.png" alt="txt_se2 on vba"><br>
  <b>{@fig:img-txt-se2}a</b>: VBA での <tt>txt_se2</tt>。
  </div>
<td>
  <div class="cpt" style="width:240px">
  <img src="img/demo/txt_se2_nocash.png" alt="txt_se2 on no$gba"><br>
  <b>{@fig:img-txt-se2}b</b>: no$gba での <tt>txt_se2</tt>。
  </div>
</table>
</div>

<div class="lblock">
  <table id="tbl:txt-se2"
    border=1 cellpadding=2 cellspacing=0>
<caption align="bottom">
  <b>@tbl:txt-se2</b>: 実機、vba、no$gba での
  時間の計測結果。
</caption>
<tbody align="right">
<tr>
  <th>&nbsp;
  <th> 実機	<th> vba	<th> no$gba		<th> vba の誤差	<th> no$ の誤差
<tr>
  <th>u16 array
  <td> 614571	<td> 499440	<td> 614571	<td> -18.73	<td> 0.00
<tr>
  <th>u32 array
  <td> 289825	<td> 230383	<td> 288098	<td> -20.51	<td> -0.60
<tr>
  <th>memcpy
  <td> 195156	<td> 161119	<td> 194519	<td> -17.44	<td> -0.33
<tr>
  <th>memcpy32
  <td> 86816	<td> 79336	<td> 85329	<td> -8.62	<td> -1.71
<tr>
  <th>DMA32
  <td> 76889	<td> 250	<td> 76888	<td> -99.67	<td> 0.00

<!--<tr>
  <th>u16 array
  <td> 674978	<td> 557081	<td> 672162		<td> 17.5%		<td> 0.4%
<tr>
  <th>u32 array
  <td> 260299	<td> 192183	<td> 259309		<td> 26.2%		<td> 0.4%
<tr>
  <th>memcpy
  <td> 195171	<td> 160367	<td> 194608		<td> 17.8%		<td> 0.3%
<tr>
  <th>memcpy32
  <td> 86846	<td> 80049	<td> 85283		<td> 7.8%		<td> 1.8%
<tr>
  <th>DMA32
  <td> 76902	<td> 222	<td> 76901		<td> 99.7%		<td> 0.0%
-->
</tbody>
</table>
</div>

*@fig:img-txt-se2 は、VisualBoy Advance と no\$gba で実行した時間の計測結果です。両者がまったく同じではないことに注意してください。そこで、2 つの意見が食い違ったときにいつもすべきことをします。3 つ目の意見を聞くのです。この場合は、本当に重要な唯一のもの、つまり実機を使います。3 つの比較は @tbl:txt-se2 で見られます。これを見ると、no\$gba の時間はとても正確ですが、VBA はそれほどでもないことがわかります。見積もりや相対的な時間を得るにはまだ使えると思いますが、本当の正確さはそこにはありません。それには実機か no\$gba が必要です。

数値そのものについてです。開きは約 9 倍で、これはかなりのものです。ここで示した方法はどれも特に理解しにくいものではありませんし、データのコピーにはたくさんの時間を費やしかねないので、最初から速い方法を活用したほうがよいでしょう。

チュートリアルのコードのほとんどや、出回っているデモのコードの多くは、u16 の配列でコピーする方法を使っています。おそらく、特定の領域ではバイト単位のコピーが使えないからでしょう。でもご覧のとおり、**u16 のコピーは u32 のコピーの 2 倍以上遅い**のです！ たしかに、データをコピーする最も遅い方法ではありませんが、大差はありません（ループの変数にも u16 を使うと（これもよくあることです）、さらに 20% ほど遅くなります。試してみればわかります）。GBA は 32 ビットのマシンです。32 ビットのデータを_好み_、その命令セットは 32 ビットの単位を扱うほうが得意です。よそで身につけたかもしれない u16 へのこだわりは捨ててください。できるならワードの大きさのデータを使い、ほかのものはやむをえない場合にだけ使いましょう。とはいえ、[データのアラインメント](bitmaps.html#ssec-data-align)には気をつけてください！ u8 や u16 の配列は常にワード境界に揃っているとは限らず、キャストで問題を起こします。

:::warning GCC とウェイトステート、そして時間の計測結果
 
正確な時間の計測結果を示すのは、いくつかの要因のせいで難しいのです。まず、ハードウェアの面では、異なるウェイトステートを持つ異なるメモリ領域があり、腰を据えてアセンブリを読み、命令のサイクル数を足し合わせない限り、話が複雑になります。これはひどい作業です。信じてください。2 つ目の問題は、GCC がこのコードについて理論上の最適にまだ達していないので、結果が新しいリリースごとに変わりがちなことです。上に示したものは良い目安ですが、結果は環境によって変わるかもしれません。

:::

大きなデータの塊をコピーする速い方法はいくつかあります。単純なループを自分で書くよりは速い方法、ということです。よく使われるのは、どのプラットフォームでも使える標準の `memcpy()` と、GBA 専用の 2 つの方法、`CpuFastSet()` という BIOS コール（または私自身の版である `memcpy32()`）と DMA です。最初の 2 つはワード境界への整列を_必要とします_。DMA は単に整列しているほうがうまく動きます。`memcpy()` の性能は実はそれほど悪くなく、どこでも使えるので、最初に使うには良いものです。ほかのものはもっと速いですが、代償があります。`memcpy32()` は手書きのアセンブリですし、`CpuFastSet()` はワード数が 8 で割り切れる必要があり、DMA は CPU を止めてしまうので割り込みを邪魔することがあります。もう少し速さが必要になったときは、これらのことを覚えておくとよいでしょう。

## そのほかの検討事項 {#sec-misc}

ここで示したいくつかの関数は、テキストシステムについてはほんの表面をなでたにすぎません。もっと大きなフォント、色付きのフォント、きちんとした影、可変の文字幅などが考えられます。それぞれがどのモードにも当てはまり、さらに文字の揃えや配置のための書式、VRAM の使用を減らすためにマップや OAM の変更と合わせてタイルのメモリを更新すること、などなどがあります。すべての組み合わせを詳しく見ていくには、それだけでサイトが 1 つ必要になるので、ここまでにしておきます。テキストシステムに入っている基本のいくつかを身につけてもらえたなら幸いです。その知識で何をするかは、あなたにお任せします。
