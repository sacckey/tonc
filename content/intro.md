# はじめに

<!-- toc -->

## 構成 {#sec-org}

<!-- <div class="cpt_fr" style="width:120px;">
  <img alt="Tonc directory structure" src="./img/toncdirs.png" id="fig:toncdirs">

  **{*@fig:toncdirs}**: directories.
</div> -->

TONC は3つの要素でできています。<dfn>本文</dfn>（チュートリアルそのもの）、いろいろなデモのソースコードと makefile がすべて入った <dfn>examples</dfn> リポジトリ、そしてチュートリアル全体で紹介する、便利で再利用できるコードをまとめた <dfn>libtonc</dfn> というライブラリです。

以前はどれも zip ファイルで配布されていましたが、時代は変わり、今はそれぞれ別の git リポジトリになっています。ここから、それぞれを詳しく説明します。


### Tonc の本文 {#ssec-org-text}

今読んでいる本文では、GBA プログラミングの原則を詳しく扱います。重点は、何かを実現する方法そのものより、物事が実際にどう*動いているのか*、そしてなぜそのやり方をするのかにあります。それが分かれば、*どうやるか*は自然に分かることが多いものです。どの章にも、扱った理論を示すデモが1つ以上あり、そのデモについての短い解説が付いています。

どうか、デモの解説だけを読むという間違いはしないでください。仕組みをきちんと理解するには、本文を全部読む必要があります。読み飛ばしてもいい部分や、実際の GBA のコーディングとはほとんど関係なさそうな退屈な文章がページまるごと続くところもありますが、それにはちゃんと理由があります。たいていは、概念についての補足や、ハマりどころが書かれています。

最初、本文にはコードをほとんど載せていませんでした。デモのコードは手元にあるはずなので、両方を行き来するのも苦にならないだろうと考えたからです。でも、その考えは間違いだったと気づいたので、今はもっと多くのコードをこのページに載せるようにしています。コピー&ペーストしてそのままきれいにコンパイルできるほどではないかもしれませんが、デモの説明を理解するには十分な量です。

主に使う言語は C で、アセンブリも少しだけ使います。ほかの言語もありますが、GBA プログラミングで主に使われるのはこの2つです。プログラミングの基本は言語に関係ないので、自分の選んだ言語にも簡単に応用できるはずです。

GBA プログラミングはハードウェアに近いところで行うので、ポインタ、[16進数](numbers.html#sec-num)、[ブール代数とビット演算](numbers.html#sec-bitops)を理解していることを願っています。数学もかなり出てきます。ほとんどは[ベクトルと行列](matrix.html)の話なので、線形代数の知識もしっかりしていることを願っています。最後に、あなたの知的能力はそこらの実験用のサルよりは上だと想定しているので、私が当たり前だと思うことは、あまり詳しく説明しません。

はじめにと付録を除くと、本文は3部に分かれています。最初は「基礎」で、何かをするための、絶対に欠かせない基本を説明します。開発環境の準備や、グラフィックとボタンの基本的な使い方がここに入ります。低レベルのプログラミングをするとはどういうことか、効率よくプログラミングするとはどういうことか、という内容もあります。私の考えでは、どちらも早く学んでおくに越したことはない話です。第2部では、特殊なグラフィック効果、タイマー、割り込みなど、GBA のほかの要素のほとんどを扱います。最後の部では、すべての章の要素を使う、より高度な話題を扱います。文字の表示（そう、GBA ではこれは高度な話題なのです）、Mode 7 のグラフィック、ARM アセンブリの章などです。

本文の Markdown のソースは、すべて [GitHub で公開されています](https://github.com/gbadev-org/tonc)。誤字や改善できる点を見つけたら、気軽に貢献してください。

<!--
TODO: maybe provide links to PDF and 'offline' html downloads?

Individual html: [tonc-text.zip](http://www.coranac.com/files/tonc-text.zip) (663 kb)  
Compiled html (CHM, v1.4 version): [tonc.chm](http://www.coranac.com/files/tonc.chm) (1.2 MB).  
PDF: [tonc.pdf](http://www.coranac.com/files/tonc.pdf) (3.1 MB)
-->

### Tonc のコード（libtonc と examples） {#ssec-org-code}

本文に出てくるデモのソースコードは、すべて [libtonc-examples](https://github.com/gbadev-org/libtonc-examples) リポジトリにあります。本文と同じく、例も *basic*（基礎）、*extended*（応用）、*advanced*（上級）の3部に分かれています。ほかに、面白いプロジェクトがいくつか入った `lab` ディレクトリもありますが、こちらはまだ完成していないかもしれません。それでも、見てみる価値はあります。

使う言語は **C** で、アセンブリを少しだけ使います（C++ では*ありません*）。この言語に慣れていることを前提にしています。慣れていないなら、先に学んできてください。ここは C の講座ではないので、私は教えません。[参考文献](refs.html#ssec-tut)に、C のチュートリアルへのリンクをいくつか載せています。

古い GBA のチュートリアルの中には、例のプロジェクトをバッチスクリプトでビルドするものもありますが、tonc では **makefile** を使います。そのほうが、とにかく優れている™からです。使い方は次の章で説明します。コンパイル済みの例の ROM を試したいだけなら、今もここから入手できます: [tonc-bin.zip](http://www.coranac.com/files/tonc-bin.zip)。

例は [libtonc](https://github.com/gbadev-org/libtonc) に依存しています。libtonc は、チュートリアル全体で紹介する重要な #define と関数をすべてまとめたライブラリです。すべてのビデオモード用の文字表示、BIOS ルーチン、かなり高度な割り込みディスパッチャ、安全で高速なメモリのコピーと塗りつぶしのルーチンなど、多くのものが入っています。昔は `libtonc` も例と一緒に配布されていましたが、今は devkitARM に含まれているので、自分でダウンロードする必要はありません。


### 執筆の目的 {#ssec-org-sop}

私が Tonc を書いた理由は2つあります。1つ目は、自分の考えを整理するためです。何かを書き出してみると、物事が違って見えることがよくあり、その経験から学べます。2つ目は、世の中のほかのチュートリアルには、*とてもひどい*情報がたくさんあるからです（私が知っている例外は、[新しい PERN](http://www.drunkencoders.com/web.archive.org/web/20030413142151fw_/http_/www.thepernproject.com/English/tutorial.html) と [Deku のサウンドのチュートリアル](https://stuij.github.io/deku-sound-tutorial/) だけです）。ええ、これがどう聞こえるかは分かっていますが、残念ながら事実なのです。いくつか例を挙げます。

-   ごく基本的な情報しか載っておらず、ときには[間違った情報](affine.html)さえある。
-   ビットマップモードに偏っている。本格的な GBA プログラミングでは、ほとんど使われないモードです。
-   [悪いプログラミングの習慣](first.html#ssec-notes-bad)。[ファイルを #include する](bitmaps.html#ssec-data-hdr)ことでコードやデータをプロジェクトに加える、大昔の[ツールチェーン](setup.html#sec-alt)を使う、最適でないコンパイラの設定やデータ型を使う、非効率な（ときには*とても*非効率な）コードを書く、など。

ほかのチュートリアルに従ってきた初心者には、すべてうまく動いているように見えるでしょう。では、何が問題なのでしょうか？ 実は、それこそが問題の一部なのです。すべてうまく動いているように*見える*のは、大きなプロジェクトを始めるまでです。そのときになって、隠れていたエラーが見つかり、遅いコードが足を引っ張ることに気づきます。そして、身につけた悪い習慣をすべて捨て、最初から全部やり直すことになります。GBA は、効率のよいコードを書くことに今でも意味がある、数少ないプラットフォームの1つです。データ型やコンパイラのスイッチを変えるだけで済むこともあります。こうしたことは、最初から正しくやっておくに越したことはありません。

私は、まず網羅性を、次に分かりやすさを目指しました。ある髪の乱れた科学者がかつて言ったように、「物事はできる限り単純にすべきだが、それ以上単純にしてはならない」のです。そのため、ときどき少し専門的に感じるかもしれません。でもそれは、物事が実際にときどきかなり専門的だからで、そうではないふりをしても意味がありません。

要するに、Tonc は「サルでもわかる GBA プログラミング」では*ありません*。これまでもそうではなかったし、これからもそうなることはありません。そういうものは、もう十分すぎるほどあります。自分をおばかさんだと思うなら（初心者という意味ではなく、本当におばかさんという意味です）、Tonc は向いていないかもしれません。でも、本気で GBA プログラミングを学びたいなら、Tonc に代わるものはありません。

## 用語と表記 {#sec-nota}

私は物理学を学んできたので、数学とその表記の慣習はよく知っています。Tonc では、それらと、いくつかの html タグの慣習をよく使います。認識をそろえるために、一覧にしておきます。

<div class="lblock">
  <table class="table-data">
    <tr><th>種類	<th>表記	<th>例
    <tr><td><code>foo</code> のビット n		<td><code>foo</code>{n}	
    <td><code>REG_DISPCNT{4}</code>（アクティブページのビット）
    <tr><td>コード	<td>&lt;code&gt; タグ	<td> <code>sx</code>
    <tr><td>コマンド、ファイル <td>&lt;tt&gt; タグ <td> <tt>vid.h</tt>
    <tr><td>行列	<td>太字、大文字		<td> <b>P</b>
    <tr><td>メモリ  <td>16進数 + code			<td> <code>0400:002eh</code>
    <tr><td>新しい用語 <td>太字、斜体		<td> <dfn>charblock</dfn>
    <tr><td>変数 <td>斜体			<td> <i>x</i>
    <tr><td>ベクトル	<td>太字、小文字		<td> <b>v</b>
  </table>
</div>

ASCII 以外の記号もいくつか使います。ブラウザが古いと、正しく表示されないかもしれません。次のものです。

<div class="lblock">
<table class="table-data">
<tr><th>記号<th>説明
<tr><td>&alpha;, &beta;, &gamma;	<td>ギリシャ文字
<tr><td>&asymp;		<td>ほぼ等しい
<tr><td>&frac12;	<td>2分の1
<tr><td>&frac14;	<td>4分の1
<tr><td>&frac34;	<td>4分の3
<tr><td>&ge;		<td>以上
<tr><td>&harr;		<td>両向きの矢印
<tr><td>&isin;		<td>（区間に）含まれる
<tr><td>&lang; &rang;		<td>&lsquo;ブラ&rsquo;と&lsquo;ケット&rsquo;
<tr><td>&rarr;		<td>右向きの矢印
<tr><td>&sup2;		<td>上付きの2
<tr><td>&times;		<td>掛ける
</table>
</div>

`char` や `int` などの C の基本型には、短い別名もよく使います。これらは、使う変数の大きさがより分かりやすい typedef です。ゲーム機のプログラミングではこれがとても重要なので、よく使われます。一覧は次のとおりです。

<div class="lblock">
<table class="table-data">
<tr><th>基本型	<th>別名 <th>符号なし	<th>符号あり	<th>volatile
<tr><th>char		<td>バイト	 <td>u8			<td>s8		<td>vu8 / vs8
<tr><th>short		<td>ハーフワード <td>u16		<td>s16		<td>vu16 / vs16
<tr><th>int			<td>ワード	 <td>u32		<td>s32		<td>vu32 / vs32
</table>
</div>

最後に、16進数にはいくつかの表記があり、場面によって使い分けます。普通の数には C の表記（‘0x’ の接頭辞、0x0400）をよく使いますが、ときにはアセンブリの接尾辞（‘h’、0400:0000h）も使います。ここでのコロンは、読みやすくするためだけのものです。コロンがないと、ゼロがいくつあるのか分かりにくいからです。

### レジスタ名とその説明 {#ssec-note-reg}

GBA に何かをさせるには、たいてい、いわゆる <dfn>IO レジスタ</dfn>を使います。メモリの特定のアドレスにある特定のビットが、GBA でできるさまざまな効果のスイッチとして使えるのです。各レジスタには普通の変数として別名が付けられていて、ビット演算でビットを立てたり消したりします。それぞれのレジスタがどこにあり、どのビットが何をするかは後で説明します。ここでは、それらを本文でどう*示し*、どう参照するかを紹介しておきます。

各レジスタ（またはレジスタのように使うアドレス）は、参照外しされたポインタに対応づけられていて、たいてい16ビットの長さです。たとえば、ディスプレイのステータスレジスタは次のとおりです。

```c
#define  REG_DISPSTAT *(u16*)0x04000004     
```

レジスタを紹介するときは、毎回、次のようにビットの概要を示します。

<div class="reg">
<table class="table-reg"
  border=1 frame=void cellPadding=4 cellSpacing=0>
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
</div>

この表には、レジスタの名前（`REG_DISPSTAT`）、アドレス（0400:0004h）、そして個々のビットやビットフィールドが書かれています。ビットやレジスタ全体が、読み取り専用や書き込み専用のこともあります。**読み取り専用**は、（ここで使っているように）赤い上線で示します。**書き込み専用**は、青い下線で示します。そのあとに、各ビットを説明する一覧が続きます。そこには、そのビットに私が使う #define も書いてあります。

<div class="reg">
<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="15%">
<tr align="left"><th>ビット<th>名前<th>define<th>説明
<tbody valign="top">
<tr class="bg0">
  <td>0 <td class="rclr0">VbS
  <td>DSTAT_IN_VBL
  <td>VBlank のステータス。読み取り専用。VBlank の間はセットされ、VDraw の間はクリアされる。
<tr class="bg1">
  <td colspan=4 align="center"><i>ほかのフィールド</i>
<tr class="bg0">
  <td>8-F <td class="rclr3">VcT
  <td><i>DSTAT_VCT#</i>
  <td>VCount のトリガー値。現在のスキャンラインがこの値になると、
    ビット2がセットされ、要求されていれば割り込みが発生する。
</tbody>
</table>
</div>

REG_DISPSTAT の全項目は[こちら](video.html#tbl-reg-dispstat)にあります。ちなみに、#define はたいてい tonc 独自のものです。この点は、サイトや API ごとに用語が違います。大事なのは名前ではなく、それが表す数値なので、こういうことができるのです。もちろん、レジスタ自体の名前も同じです。#define についてもう1点。一覧の中には、ハッシュ（‘#’）の接尾辞が付いたものがあります。これは、そのフィールドに *foo*`_SHIFT` と *foo*`_MASK` の #define と、*foo*`()` マクロがあることを示す略記です。たとえば、ディスプレイのレジスタには8ビットの VCount トリガーのフィールドがあり、define の列には ‘*DSTAT_VCT#*’ と書かれています。これは、tonc のヘッダーに次の3つがあることを意味します。

```c
#define DSTAT_VCT_MASK      0xFF00
#define DSTAT_VCT_SHIFT          8
#define DSTAT_VCT(_n)       ((_n)<<DSTAT_VCT_SHIFT)
```

最後に、レジスタの特定のビットを表す略記として、波かっこを使います。中の数字は16進数です。たとえば、REG_DISPCNT{0} は VBlank のステータスビット（上の VbS）で、REG_DISPCNT{8-F} は VCount のトリガーのバイト全体を表します。（訳注: 上の表に合わせるなら、ここは REG_DISPCNT ではなく REG_DISPSTAT が正しいと思われます。）


## 誤りや提案について {#sec-feedback}

私たち（cearn と gbadev.net のコミュニティ）は、つづりや文法の誤り、リンク切れなどをできる限り取り除こうとしてきましたが、見落としがあるかもしれません。見つけたら、[GitHub のリポジトリ](https://github.com/gbadev-org/tonc)で Issue を作ってください（Pull Request を送ってもらえると、なおうれしいです）。もちろん、分かりにくいところや、_\*まさかの\*_ 間違いがあったとき、提案があるときも、ぜひ知らせてください！ [Discord / IRC](https://gbadev.net/resources.html#community) や[フォーラム](https://forum.gbadev.net/)でも連絡できます。


そして、もちろん:

> この配布物は現状のまま提供され、いかなる保証もありません。この配布物の使用、または使用できないことによって生じたいかなる損害についても、私は責任を負いません。コードはエミュレータと実機でできる限りテストしましたが、100% 正しいことは保証できません。
> 本文とコードは、いつでも変更されることがあります。ときどき確認して、何か変わっていないか見てください。<!--Time stamps are at the bottom of every page, and at the top of all source-files.--> 付録には[更新履歴](log.html)もあります。

以上です。楽しんでください。

<div style="margin-left:1.2cm;">

-- _Jasper Vijn_ (jakvijn at gmail dot com) と _gbadev.net_ コミュニティ<br>

<!--(Mar 24, 2013)-->
<!-- TODO: figure out how to put the build date here? -->
</div>
