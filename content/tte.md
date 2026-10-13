# 22. Tonc のテキストエンジン

<!-- toc -->

## はじめに {#sec-intro}

[テキストについてのもう 1 つのページ](text.html)では、背景やオブジェクトに文字を出す方法を説明しました。それで動きはしましたが、制限がいくつもありました。たとえば、8×8 のフォントに限られ、すべてのビデオモードには対応しておらず、書式付きのテキストの機能もありませんでした。

Tonc のテキストエンジン（TTE）は、こうした欠点の多くを解消します。この章では、このシステムの目標と基本的な設計、そして実装の細部のいくつかを説明します。特に、いろいろな種類の描画先に使う書き込み処理の作り方を説明します。場合によっては、徹底的に最適化します。注意していないと、グリフの描画処理が 1 文字に何スキャンラインもかかることがあるからです。そして、ええ、それはアセンブリで行います。

カーソルの位置、色、さらにはフォントまで動的に変えるための、基本的なスクリプトの機能を加える方法も示します。数年前、Wintermute が devkitARM の標準 C ライブラリを変更して、GBA と NDS から stdio のルーチンを使えるようにしました。これを活用する方法も示します。

そしてもちろん、デモがあります。いやもう、デモだらけです。実は 10 個ほどあるので、これまでとは少しやり方を変えます。すべての例をまとめたメニュー付きのプロジェクトを 1 つ用意します。多すぎるので、すべての例をここで示すわけではありません。

最後に、ここまで来ればもう GBA のプログラミングについてそれなりの知識があるものとして、GBA 固有の説明は最小限にとどめます。まだ扱っていない関数が使われているのを見かけたら、詳しくは GBATEK、プロジェクトのコード、libtonc のコードを見てください。

## 基本の設計 {#sec-design}

### TTE の目標 {#ssec-design-goals}

TTE に一番欲しかったものを挙げます。

- **あらゆる場面で使える、網羅的で拡張可能なグリフの書き込み処理のセット**。まあ、_ほぼ_あらゆる場面です。古いシステムは通常の背景、ビットマップモード、オブジェクトで動きました。今回はそれをアフィン背景とタイルへの描画にまで広げます。必要なものが標準のセットになければ、自分の書き込み処理を簡単に作って、代わりにそれを使えます。書き込み処理は [UTF-8](https://en.wikipedia.org/wiki/UTF-8) の文字列を受け付けるので、256 文字に限られることはありません。
- **フォント: 任意の幅と高さ、可変幅の文字**。8x8@1 のグリフに限られる代わりに、TTE の標準の書き込み処理は、どんな幅と高さのフォントも（常識の範囲内で。画面いっぱいのグリフはやめてください）、可変幅のフォントも（これも常識の範囲内で。タイルマップで可変幅フォントはあまり意味がありません）使えます。原理的には任意のビット深度を使う可能性もありますが、標準の描画処理は 1bpp に限られます。
- **描画先の詳細に依存しない、単純な書き込みのインターフェース**。古いシステムでは、`m3_puts()`、`se_puts()`、`obj_puts()` などがありました。それで動きはしましたが、モードごとに別のものを使わなければなりませんでした。TTE では、システムを準備するための初期化関数はモードごとに違いますが、文字列の書き込み処理は、とにかく動く `tte_write()` 1 つだけです。
- **テキストのパラメータのためのスクリプト**。つまり、位置や出力の色のようなパラメータを、文字列そのもので制御できるということです。この機能はかなり基本的なものですが、十分うまく動きます。注意: これは完全な会話システムでは_ありません_！ とはいえ、これを中心にそういうものを作ることはできるはずです。
- **`printf()` への対応**。理由はかなり明らかでしょう。

### 構造体と主な部品 {#ssec-design-items}

TTE に関係する情報はすべて、3 つの構造体にまとめて保持されます。<dfn>テキストコンテキスト</dfn> `TTC`、<dfn>フォントの記述</dfn> `TFont`、そして<dfn>グラフィックの描画先の記述</dfn> `TSurface` です。

`TTC` 構造体には、エンジンの主なパラメータが入っています。描画先についての情報、カーソルの位置、フォントの情報、色の属性、そのほかいくつかのものです。グリフを描いたり消したりするための 2 つのコールバックもあります。

`TFont` 構造体には、グリフのデータへのポインタ、グリフやセルの寸法、そのほかいくつかのものがあります。可変幅や可変高さのフォントのための、幅と高さの表へのポインタもあります。しばらく前に [usenti](http://www.coranac.com/projects/#usenti) に `TFont` を作る機能をむりやり付け加えたので、標準のフォントからこれらを簡単に作れますが、ゼロから自分で作ることもできます。

`TSurface` 構造体は、実はテキストとは何の関係もありません。描画する先の面（サーフェス）の種類を表す構造体です。ビットマップ、タイル、タイルマップなど何でもかまいません。Tonclib にはこれらの描画先を扱うための基本的なピクセル、線、長方形のルーチンがあるので、それを使わない手はありません。

```c{#cd-tte-types}
//# From tonc_tte.h : main TTE types.

typedef struct TFont
{
    const void  *data;      //!< Character data.
    const u8    *widths;    //!< Width table for variable width font.
    const u8    *heights;   //!< Height table for variable height font (mostly unused).
    u16 charOffset;         //!< Character offset
    u16 charCount;          //!< Number of characters in font.
    u8  charW;              //!< Character width (fwf).
    u8  charH;              //!< Character height.(fhf).
    u8  cellW;              //!< Glyph cell width.
    u8  cellH;              //!< Glyph cell height.
    u16 cellSize;           //!< Cell-size (bytes).
    u8  bpp;                //!< Font bitdepth;
    u8  extra;              //!< Padding. Free to use.
} TFont;

//! TTE context struct.
typedef struct TTC
{
    // Members for renderers
    TSurface dst;           //!< Destination surface.
    s16 cursorX;            //!< Cursor X-coord.
    s16 cursorY;            //!< Cursor Y-coord.
    TFont *font;            //!< Current font.
    u8  *charLut;           //!< Character mapping lut, if any.
    u16 cattr[4];           //!< ink, shadow, paper and special color attributes.
    // Higher-up members
    u16 reserved;
    u16 ctrl;               //!< BG control flags.
    u16 marginLeft;
    u16 marginTop;
    u16 marginRight;
    u16 marginBottom;
    s16 savedX;
    s16 savedY;
    // Callbacks and table pointers
    fnDrawg  drawgProc;         //!< Glyph render procedure.
    fnErase  eraseProc;         //!< Text eraser procedure.
    const TFont **fontTable;    //!< Pointer to font table for
    const char  **stringTable;  //!< Pointer to string table for
} TTC;
```

```c{#cd-tte-types2}
//# Supporting types

//! Glyph render function format.
typedef void (*fnDrawg)(int);

//! Erase rectangle function format.
typedef void (*fnErase)(int left, int top, int right, int bottom);

typedef struct TSurface
{
    u8  *data;          //!< Surface data pointer.
    u32 pitch;          //!< Scanline pitch in bytes.
    u16 width;          //!< Image width in pixels.
    u16 height;         //!< Image width in pixels.
    u8  bpp;            //!< Bits per pixel.
    u8  type;           //!< Surface type.
    u16 palSize;        //!< Number of colors.
    u16 *palData;       //!< Pointer to palette.
} TSurface;
```

#### TFont の詳細

<div class="cpt_fr" style="width:128px;">
  <img src="img/tte/verdana9.png" id="fig:img-verdana9" width="128" alt="">
  <br>
  <b>{*@fig:img-verdana9}</b>: Verdana 9 の文字シート
</div>

{\*@fig:img-verdana9} は、`TFont` で使える文字シートです。シートは<dfn>セル</dfn>の行列で、各セルに 1 文字が入っています。`cellW/H` メンバはこれらのセルの寸法で、`cellSize` はセルあたりのバイト数です。

各セルには 1 つのグリフがありますが、実際のグリフはセルより小さくてもかまいません（白とマゼンタの部分）。これはメモリを少し無駄にしますが、いくつかの利点もあります。利点の 1 つは、`cellSize` を使って任意のグリフのアドレスをすばやく求められることです。次に、私のフォントはビットマップ_と_タイルの両方で使えるようにしたいので、グリフの枠はどのみち 8 の倍数にする必要があります。さらに、このフォントは 1bpp なので、無駄な部分があっても、メモリの使用量はとても小さく済みます（3.5kB）。

固定幅や固定高さのフォントでは、メンバ `charW` と `charH` が実際の文字の幅と高さを表します。可変幅のフォントでは、`widths` メンバがグリフの幅を入れたバイトの配列を指し、`heights` についても同じようなことが言えます。`charOffset` は、データが始まる（ASCII の）文字です。フォントのシートはたいてい空白（' '）から始まるので、これはたいてい 32 です。`charCount` は文字の数で、シート全体を VRAM にコピーする必要がある場合（タイルマップの場合など）に使えます。

`TFont` のデータがどう使われるかは、ほぼ完全にグリフの描画処理しだいだということに注意してください。libtonc に付属するほとんどの描画処理は、次の形式を前提にしています。

- 大きさのために、**1 bpp** にビットパックされている。実は描画の速さのためでもあります。メモリからの読み込みは高くつくからです。
- グリフごとにタイル化されている。各グリフのデータは連続していて、グリフどうしの間隔は `cellSize` バイトです。これは 1D のオブジェクトの仕組みと似ていますが、重要な違いが 1 つあります。
- 各グリフの中のタイルは**列優先**です（タイル 1 はタイル 0 の下にあります）。これは、[行優先](https://en.wikipedia.org/wiki/Row-major_order)（タイル 1 はタイル 0 の右にあります）になりがちなオブジェクトとは対照的です。この形式を**タイルストリップ**と呼ぶことにします。これを選んだ理由はあとで説明します。

例外もありますが、ここで示すほとんどの描画処理はこの形式を使います。自分の描画処理を作りたいなら、データには適切だと思う形式を自由に使ってかまいません。

#### TTC の詳細

テキストコンテキスト `TTC` には、システムの最も重要なデータが入っています。上から順に見ていきましょう。まず描画先 `dst` です。これは描画する先の面を定義します。そこで最も重要な項目は、そのメモリアドレスと、**ピッチ**、つまりスキャンラインあたりのバイト数です。ピッチは描画にとって_とても_重要なパラメータで、実は幅や高さより重要です。描画先にはパレットのメンバもあり、その色にアクセスするのに使えます。`TFont` のメンバと同じく、このデータがどう使われるかは描画処理しだいです。

メンバ `cursorX/Y` は現在のカーソルの位置です。`margin` の長方形は、画面のどの部分をテキストに使うかを示します。カーソルが右のマージンを超えると、左のマージンに移り、1 行下がります。マージンは画面のクリアやページの先頭に戻るときにも使われます。

`cattr` の表は特別なものです。そのエントリは<dfn>色の属性</dfn>です。インク（前景色）、影、紙（背景色）のパラメータがここに入り、さらにとても状況に依存する「特殊」なフィールドがあります。これらの色の属性は、必ずしも色を表すわけではないことに注意してください。モード 3 と 5 では色ですが、モード 4 やタイルの書き込み処理では色のインデックスです。「色の属性」よりもっと良い名前がおそらくあるのでしょうが、sodomy non sapiens（訳注: 「知るもんか」という意味の英語の俗語をもじった、偽ラテン語のジョークです）。

グリフの描画や画面（の一部）の消去は、コールバック `drawgProc` と `eraseProc` を通して行います。考え方としては、テキストの形式に合ったルーチンでシステムを初期化し、TTE がそれらを使って実際の書き込みを行うというものです。1 つのグリフを描くのにコールバックを使うと、特にタイルマップのような単純な種類のテキストでは、かなりのオーバーヘッドになりうることは指摘しておくべきでしょう。

### TTE の主な変数と関数 {#ssec-design-writer}

TTE のシステムの状態は、`tte_get_context()` でアクセスできる `TTC` の変数に保持されます。システムへの変更はすべてそこを通ります。_場合によっては_、状態を 2 組用意して、適宜切り替えると便利なことがあります（画面が 2 つある場合などです。やあ、NDS）。そのためには、`tte_set_context)` を使ってポインタを差し替えます。

```c {#cd-ttc-funcs}
TTC __tte_main_context;
TTC *gp_tte_context= &__tte_main_context;

//! Get the master text-system.
INLINE TTC *tte_get_context()
{   return gp_tte_context;                          }


//! Set the master context pointer.
void tte_set_context(TTC *tc)
{
    gp_tte_context= tc ? tc : &__tte_main_context;
}
```

文字を表示するには、`tte_putc()` と `tte_write()` を使います。

```c {#cd-tte-putc}
//! Get the glyph index of character \a ch.
INLINE uint tte_get_glyph_id(int ch)
{
    TTC *tc= tte_get_context();
    ch -= tc->font->charOffset;
    return tc->charLut ? tc->charLut[ch] : ch;
}

//! Get the width of glyph \a id.
INLINE uint tte_get_glyph_width(uint gid)
{
    TFont *font= tte_get_context()->font;
    return font->widths ? font->widths[gid] : font->charW;
}

//! Render a character.
int tte_putc(int ch)
{
    TTC *tc= tte_get_context();
    TFont *font= tc->font;

    // (4) translate from character to glyph index
    uint gid= tte_get_glyph_id(ch);

    // (5) get width for cursor update
    int charW= tte_get_glyph_width(gid);

    if(tc->cursorX+charW > tc->marginRight)
        [[ simulate newline ]]

    // (6) Draw and update position
    tc->drawgProc(gid);
    tc->cursorX += charW;

    return charW;
}
```

```c {#cd-tte-write}
//! Render a string.
/*! \param text String to parse and write.
    \return     Number of parsed characters.
*/
int tte_write(const char *text)
{
    int ch;
    uint gid, charW;
    const char *str= text;
    TTC *tc= tte_get_context();

    while( (ch= *str++) != '\0' )
    {
        // (1) Act according to character type
        switch(ch)
        {
        case '\n':  [[ update cursorX/Y for newline ]];     break;
        case '\t':  [[ update cursorX for tab ]];           break;
        default:
            // (2) more special thingies
            if(ch=='#' && str[0]=='{')          // (2a) Command sequence
            {
                str= tte_cmd_default(str+1);
                break;
            }
            else if(ch=='\\' && str[0]=='#')    // (2b) Escaped command
                ch= *str++;
            else if(ch>=0x80)                   // (2c) UTF8 character
                ch= utf8_decode_char(str-1, &str);

            // (3) draw character
            tte_putc(ch);
        }
    }

    return str - text;
}
```

ここではいくつかの部分のコードを省きましたが、考え方はわかるはずです。まず文字を読みます。次に、それが特殊な文字（改行、タブ、書式のコマンド）かどうかを確認し、そうならそれに応じて処理します。`tte_write()` は UTF-8 に対応しているので、それも確認し、文字列をデコードして UTF-8 の文字全体を得ます。それがすべて済んだら、その文字を `tte_putc()` に渡します。この関数はそれをグリフのインデックスに変換し、グリフを描いて、カーソルを進めます。

注意: ここで説明したやり方は、物事を行う_ひとつの_方法であって、_唯一の_方法ではありません。そんなものは実際には存在しないからです。ここで行っているいくつかの手順は、考えているテキストの種類によってはやりすぎかもしれません。たとえば、文字からグリフのインデックスを得るのに、フォントの文字のオフセットと、場合によっては文字のルックアップテーブルを使っていますが、どちらも厳密には必要ありません。同様に、端での折り返しは、文字列そのものの中で改行文字によってすでに行われているかもしれません。一方で、もっと複雑な折り返し、文字の揃え、スクロール、その他何でも欲しいかもしれません。そうしたものが欲しければ、自分のルーチンを作るのはそれほど難しくないはずです。

### 用語について {#ssec-design-names}

TTE で使ういくつかの用語には、とても特定の意味があります。用語の違いは微妙なことがあるので、明示的に定義しておくことが大事です。また、TTE では説明が必要な頭字語や略語もいくつか使います。

- **文字（char(acter)）とグリフのインデックス**。「文字」は ASCII の文字を指し、「グリフのインデックス」はフォントの中の対応するインデックスです。たとえば「A」は文字 65 ですが、フォントが空白（' '、ASCII 32）から始まるなら、「A」はグリフのインデックス 65−32=33 です。原則として、`ch` という名前の変数は文字で、`gid` はグリフのインデックスを意味します。描画処理の入力は文字ではなく、グリフのインデックスです。

- **描画先（サーフェス）**。サーフェスは、文字を表示するために操作するものを何でも表すのに使っている用語です。普通は VRAM ですが、オブジェクトのテキストでの OBJ_ATTR のように、ほかのものでもかまいません。

- **ピッチ**。ピッチは実はグラフィックではよく使われる用語ですが、グラフィックの用語はそれほど一般的ではないかもしれないので、繰り返しておく価値があります。厳密には、<dfn>ピッチ</dfn>は行と行の間のスキャンラインの数です（訳注: 通常はバイト数で、本章の TTC の詳細でもスキャンラインあたりのバイト数と説明しています）。ここではこれを少し広げて、行列の_特徴的な主方向の間隔_という意味で使います。行列は 2D のもので、一方の方向には隣り合った要素があり、もう一方の方向にはもっと大きな間隔があります。普通、これらはそれぞれ _x_ と _y_ ですが、いつもそうとは限りません。_副_方向の間隔は<dfn>ストライド</dfn>と呼ぶことにします。

- **色と色の属性**。「色」は本当の 5.5.5 の BGR の色で、「色の属性」は描画処理が描画先で使うものなら何でもです。それは色のこともありますが、パレットのインデックスや、まったく別のもののこともあります。解釈は描画処理しだいです。

- **描画/テキストのファミリー**。これは、特定の種類のテキストを表す概念上のグループ名です。{\*@tbl:tte-family} に、使えるファミリーの概要を示します。これはおおむね `TSurface` の種類に対応します。

- **描画処理の種類**。各ファミリーの中には、いろいろな種類のフォントや効果のための、別々の描画処理がありえます。たとえば、8bpp のビットマップ（bmp8 ファミリー）に描画するときは、フォントのビット深度（たとえば 1bpp か 8bpp）やグリフの配置（ビットマップかタイルか）ごとに別の描画処理を持てます。一部のピクセルを透明に描いたり、アンチエイリアスをかけたりもできます。あるいは、それらのどんな組み合わせでも。要するに、ここには_たくさん_の選択肢があるということです。

  1 行まるごとを占めるような名前は本当に嫌いなので、描画処理の名前には、それが何をするかを示す略語を使います。それらの意味は {@tbl:tte-drawg} を見てください。描画処理のほとんどは、任意の幅と高さを持つフォントで、グリフは 1bpp のタイルストリップ、それを透明に描き、ピクセルの色を付け直すものです。これを `*_b1cts` で表します。

<div class="lblock">
  <table id="tbl:tte-family" class="table-data">
    <caption align="bottom">
      <b>{*@tbl:tte-family}</b>: TTE の描画ファミリーの識別子と初期化関数。4bpp のタイルは行優先にも列優先にもできます（<code>crh4r</code> か <code>chr4c</code>）。
    </caption>
    <tr>
      <th width=25%>ファミリー</th>	
      <th>接頭辞</th> 
      <th>初期化関数</th>		
    </tr>
    <tr>
	    <td>通常のタイルマップ（モード 0/1）</td>	
      <td>se</td>
	    <td>
        void tte_init_se(int bgnr, u16 bgcnt, SCR_ENTRY se0, u32 colors, u32 bupofs, const TFont *font, fnDrawg proc);
      </td>
    </tr>
    <tr>
	    <td>アフィンのタイルマップ（モード 1/2）</td>
      <td>ase</td>
	    <td>
        void tte_init_ase(int bgnr, u16 bgcnt, u8 ase0, u32 colors, u32 bupofs, const TFont *font, fnDrawg proc);
      </td>
    </tr>
    <tr>
	    <td>4bpp のタイル（モード 0/1/obj）</td>	
      <td>chr4<i>(c/r)</i></td>
	    <td>
        void tte_init_chr4<i>(c/r)</i>(int bgnr, u16 bgcnt, u32 cattrs, u32 colors, const TFont *font, fnDrawg proc);
      </td>
    </tr>
    <tr>
	    <td>8bpp のビットマップ（モード 4）</td>
      <td>bmp8</td>
	    <td>
        void tte_init_bmp(int vmode, const TFont *font, fnDrawg proc);
      </td>
    </tr>
    <tr>
	    <td>16bpp のビットマップ（モード 3/5）</td>
      <td>bmp16</td>
	    <td>void tte_init_bmp(int vmode, const TFont *font, fnDrawg proc);</td>
    </tr>
    <tr>
	    <td>オブジェクト</td>
      <td>obj</td>
	    <td>
        void tte_init_obj(OBJ_ATTR *dst, u32 attr0, u32 attr1, u32 attr2, u32 colors, u32 bupofs, const TFont *font, fnDrawg proc);
      </td>
    </tr>
  </table>
</div>
<br>
<div class="lblock">
  <table id="tbl:tte-drawg" class="table-data">
    <caption align="bottom">
      <b>{*@tbl:tte-drawg}</b>: 描画処理の種類のまとめ。
    </caption>
    <tr>
      <th>コード</th>
      <th>説明</th>
    </tr>
    <tr>
      <td>b<i>x</i></td>
      <td>元のフォントの<b>ビット</b>深度。（<code>b1</code> = 1 bpp）</td>
    </tr>
    <tr>
      <td>w<i>x</i></td>
      <td>特定の<b>幅</b>（width。<code>w8</code> = 幅 8）</td>
    </tr>
    <tr>
      <td>h<i>x</i></td>
      <td>特定の<b>高さ</b>（height。<code>h8</code> = 高さ 8）</td>
    </tr>
    <tr>
      <td>c</td>
      <td>
        色の付け直し（re-<b>c</b>oloring）。色の属性が何らかの形でピクセルに適用されます。
      </td>
    </tr>
    <tr>
      <td>t/o</td>
      <td>紙のピクセルが<b>透明</b>（transparent）か<b>不透明</b>（opaque）か。</td>
    </tr>
    <tr>
      <td>s</td>
      <td>グリフがタイル<b>ストリップ</b>（tile-strip）形式。</td>
    </tr>
  </table>
</div>

最後に、描画のコードで使っている略語について少し説明します。いくつかの用語は何度も何度も出てくるので、それらに略記を使うことにしています。基本の形は `fooX` で、`foo` は関係するビットマップや描画先、`X` は幅、高さ、データなどを表す 1 文字のコードです。そう、1 文字の名前を使うのは嫌われますし、一般にはおすすめしませんが、この特定の場合には、慎重に使えば、自分のコードを読む助けになることがわかりました。

<div class="lblock">
  <table id="tbl:tte-brevs" class="table-data">
    <caption align="bottom">
       <b>{*@tbl:tte-brevs}</b>: 描画のコードで使う略語。
    </caption>
    <tr> 
      <th>用語</th>
      <th>意味</th>
    </tr>
    <tr>
      <td><code>fooW</code></td>
      <td><code>foo</code> の幅</td>
    </tr>
    <tr>
      <td><code>fooH</code></td>
      <td><code>foo</code> の高さ</td>
    </tr>
    <tr>
      <td><code>fooB</code></td>
      <td><code>foo</code> のビット深度</td>
    </tr>
    <tr>
      <td><code>fooP</code></td>
      <td><code>foo</code> のピッチ</td>
    </tr>
    <tr>
      <td><code>fooD</code></td>
      <td><code>foo</code> の主なデータのポインタ</td>
    </tr>
    <tr>
      <td><code>fooL</code></td>
      <td><code>foo</code> の副のデータのポインタ</td>
    </tr>
    <tr>
      <td><code>fooS</code></td>
      <td><code>foo</code> の大きさ</td>
    </tr>
    <tr>
      <td><code>fooN</code></td>
      <td><code>foo</code> の数</td>
    </tr>
  </table>
</div>

## タイルマップのテキスト {#sec-map}

### 通常のタイルマップのテキスト {#ssec-map-reg}

タイルマップのテキストは、実装するのが一番簡単です。実際には何も描画する必要がないからです。フォントのタイルをすべてキャラブロックに読み込み、実際のテキストのためにスクリーンエントリを置くだけです。

通常のタイルマップの初期化関数は `tte_init_se()` です。最後の 2 つの追加のパラメータ `font` と `proc` を除けば、[`txt_init_se()`](text.html#cd-txt-init-se) と同じです。これらは使うフォントと、描画先の操作を行う描画処理を表します。TTE のすべての初期化関数にこの 2 つのパラメータがあります。何を使えばいいかわからなければ、NULL を渡しても安全です。その場合は、そのファミリーのデフォルトの選択肢が使われます。

`font` が NULL なら、デフォルトのフォントになります。これは、固定幅の場面では [`system8Font`](text.html#img-tonc-font)、可変幅が適している場面では `verdana9Font`（{@fig:img-verdana9}）です。これらはそれぞれ `fwf_default` と `vwf_default` でも参照できます。

各ファミリーにはデフォルトの描画処理もあり、_`foo`_`_drawg_default` として #define されています。ここで _foo_ はファミリーの接頭辞です。デフォルトの描画処理は汎用のルーチンで、すべての文字の幅と高さ（固定でも可変のフォントでも）に対応しています。もちろん、それは特定のグリフの大きさ向けに書かれたルーチンより遅いということでもあります。これは特にタイルマップのテキストに当てはまり、そのため、そこでは特定の `_w8h8` 版と `_w8h16` 版も用意しています。

初期化関数は長くて退屈になりがちなので、ここではあまり紙幅を割きません。基本的には、テキストコンテキストをクリアし、マージンと描画先の変数に妥当な値を入れ、フォント、描画処理、消去処理を準備します。パレットと色の属性もいくらか埋めます。

この章で示すコードは、ほとんどが描画処理そのものについてです。下に、デフォルトのスクリーンエントリの書き込み処理 `se_drawg_s()` と、8×8 のフォント専用のもの `se_drawg_w8h8` のコードを示します。

```c {#cd-se-drawg .proglist}
//! Character-plot for reg BGs, any sized, vertically tiled font.
void se_drawg_s(uint gid)
{
    int ix, iy;

    // (1) Get main variables.
    TTC *tc= tte_get_context();
    TFont *font= tc->font;
    uint charW= (font->cellW+7)/8, charH= (font->cellH+7)/8;

    uint x0= tc->cursorX, y0= tc->cursorY;
    uint dstP= tc->dst.pitch/2;
    u16 *dstD= (u16*)(tc->data + (y0*dstP+x0)*2);

    // (2) Get the base tile index.
    u32 se= tc->cattr[TTE_SPECIAL] + gid*charW*charH;

    // (3) Loop over all tiles to draw glyph.
    for(ix=0; ix<charW; ix++)
    {
        for(iy=0; iy<charH; iy++)
            dstD[iy*dstP]= se++;
        dstD++;
    }
}

//! Character-plot for reg BGs using an 8x8 font.
void se_drawg_w8h8(uint gid)
{
    TTC *tc= tte_get_context();

    uint x0= tc->cursorX, y0= tc->cursorY;
    uint dstP= tc->dst.pitch/2;
    u16 *dstD= (u16*)(tc->data + (y0*dstP+x0)*2);

    dstD[0]= tc->cattr[TTE_SPECIAL] + gid;
}
```

単純なほうから始めましょう。`se_drawg_w8h8()` です。GBA のタイルマップに 8×8 のグリフを描くということは、単に正しい場所にスクリーンエントリを 1 つ書くことです。ここでの正しい場所は、カーソルの位置と描画先のデータ（`tc->dst`）から求めます。「特殊」な色の属性は、パレットの切り替えなどのために、グリフのインデックスを修正するのに使われます。

このルーチンはグリフを描くことだけを扱っていることに注意してください。ASCII からグリフのインデックスへの変換やカーソルの位置の変更は、すべてほかのところで行います。

もっと汎用のルーチン `se_drawg_s()` は少し複雑です。それでもまず、グリフの描画先へのポインタ `dstD` と、ピッチ（次の行までの距離）`dstP` を得るところから始まります。**すべての**描画処理は、このようなもので始まります。また、大きさが前もって決まっていない限り、すべての描画処理は文字の幅と高さも取得します。描画で使う名前は常に同じなので、それらを初期化する式が少しややこしくても、何が何を意味するかはわかるはずです。

ともかく、ポインタとピッチを得たら、グリフの左上のタイルのインデックスを計算して `se` に入れます。そのあとで、グリフのタイルを両方向にループでたどります。デフォルトのフォントがそう並んでいるので、ループの順序は行優先ではなく列優先であることに注意してください。

実際、グリフはたいてい幅より高さのほうが大きいので、列優先の描画のほうがテキストでは効率が良くなりがちです。また、タイルマップのテキストでは `charW` と `charH` は小さくなりがちで、多くの場合 1 か 2 です。つまり、ループを使うのはきわめて非効率です。どれほど非効率かは、[「描画処理のプロファイリング」の小節](#ssec-misc-profile)で見ます。`se_drawg_w8h8()` や `se_drawg_w8h16()` のようにループを展開すると、ずっと良い性能が得られます。

### 通常のタイルマップの例 {#ssec-test-se4}

```c {#cd-test-se4 .proglist}
void test_tte_se4()
{
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    // --- (1) Base TTE init for tilemaps ---
    tte_init_se(
        0,                      // Background number (BG 0)
        BG_CBB(0)|BG_SBB(31),   // BG control (for REG_BGxCNT)
        0,                      // Tile offset (special cattr)
        CLR_YELLOW,             // Ink color
        14,                     // BitUnpack offset (on-pixel = 15)
        NULL,                   // Default font (sys8)
        NULL);                  // Default renderer (se_drawg_s)

    // --- (2) Init some colors ---
    pal_bg_bank[1][15]= CLR_RED;
    pal_bg_bank[2][15]= CLR_GREEN;
    pal_bg_bank[3][15]= CLR_BLUE;
    pal_bg_bank[4][15]= CLR_WHITE;
    pal_bg_bank[5][15]= CLR_MAG;

    pal_bg_bank[4][14]= CLR_GRAY;

    // --- (3) Print some text ---

    // "Hello world in different colors"
    tte_write("\n Hello world! in yellow\n");
    tte_write(" #{cx:0x1000}Hello world! in red\n");
    tte_write(" #{cx:0x2000}Hello world! in green\n");

    // Color use explained
    tte_set_pos(8, 64);
    tte_write("#{cx:0x0000}C#{cx:0x1000}o#{cx:0x2000}l");
    tte_write("#{cx:0x3000}o#{cx:0x4000}r#{cx:0x5000}s");
    tte_write("#{cx:0} provided by \\#{cx:#}.");


    // --- (4) Init for 8x16 font and print something ---
    GRIT_CPY(&tile_mem[0][256], cyber16Glyphs); // Load tiles
    tte_set_font(&cyber16Font);                 // Attach font
    tte_set_special(0x4100);                    // Set special to tile 256, pal 4
    tte_set_drawg(se_drawg_w8h16);              // Attach renderer

    tte_write("#{P:8,80}Also available in 8x16");

    key_wait_till_hit(KEY_ANY);
}
```

上のコードは、タイルマップで TTE を使ってできることをいくつか示しています。`tte_init_se()` の呼び出しは、キャラブロック 0 とスクリーンブロック 31 を使って背景 0 にテキストを表示し、デフォルトのフォントと描画処理を使うようにシステムを初期化します。5 つ目のパラメータはビットの展開のオフセットです。これを 14 にすることで、フォントの値が 1 のピクセルはすべて 14+1=15、つまりパレットバンクの最後のインデックスに移ります。パレットが {@fig:img-test-se4}b のようになるように、ほかの色もいくつか設定しています。

手順 3 では、`tte_write()` でテキストをいくつか表示しています。色の違いは、文字列の中で `#{cx:`_`num`_`}` を使うことで実現しています。これは特殊な色の属性を _num_ に設定します。この種のコマンドについては、[「スクリプト、コンソール入出力、そのほかの便利なもの」の節](#sec-misc)で詳しく説明します。`se` の描画処理は最終的な出力のためにこの値をグリフのインデックスに加えるので、パレットの切り替えに使えます。

手順 4 では、2 つ目のフォントを読み込んで使う方法を示しています。`cyber16Font` は、懐かしのスーパーファミコンのゲーム、重装機兵ヴァルケン（Cybernator）で使われていた 8×16 のフォントを再現したものです（{@fig:img-cyber16} を見てください）。このフォントは 4bpp のデータとして書き出したので、そのまま VRAM にコピーできますが、古いフォントも残しておきたいので、オフセットを使う必要があります。これでキャラブロックには 2 組のグリフがあります（{@fig:img-test-se4}c を見てください）。

<div class="cblock">
  <table width=60%>
    <tr valign="bottom">
      <td>
	      <div class="cpt_fl" style="width:128px;">
	        <img src="img/tte/cyber16.png" id="fig:img-cyber16" alt="Cybernator font: 8&times;16.">
          <br>
	        <b>{*@fig:img-cyber16}</b>: Cybernator のフォント: 8&times;16。
	      </div>
      </td>
      <td>
	      <div class="cpt" style="width:240px;">
	        <img src="img/tte/test_tte_se4.png" id="fig:img-test-se4" alt="test_tte_se4">
          <br>
	        <b>{*@fig:img-test-se4}a</b>: <code>test_tte_se4</code> の出力。
	      </div>
      </td>
    </tr>
    <tr valign="top">
      <td>
	      <div class="cpt" style="width:128px;">
	        <img src="img/tte/tte_se4_pal.png" alt="Palette for <code>test_tte_se4</code>.">
          <br>
	        <b>{*@fig:img-test-se4}b</b>: パレット。
	      </div>
      </td>
      <td>
	      <div class="cpt" style="width:256px;">
	        <img src="img/tte/tte_se4_tiles.png" alt="VRAM Tiles for <code>test_tte_se4</code>.">
          <br>
	        <b>{*@fig:img-test-se4}c</b>: タイルセット。
	      </div>
      </td>
    </tr>
  </table>
</div>

原理的には、別のフォントを使うには `tte_set_font()` でそれを選ぶだけでいいのですが、タイルがずれた位置にあるので、特殊な色の属性も調整する必要があります。ここでは、オフセット（0x0100）とパレットバンク（0x4000）の両方を考慮して、値 0x4100 を使っています。この機会に別の描画処理も選んでいますが、デフォルトの描画処理でも 8×16 のフォントを同じように扱えるので、これはほとんど見せるためです。そのあとは、新しいフォントで新しい文字列を表示するために、また `tte_write()` を呼ぶだけです。

### アフィンのタイルマップのテキスト {#ssec-map-affine}

アフィンのタイルマップのテキストは、通常のタイルマップとほとんど同じように動きます。マップの大きさや使えるビット深度など、2 種類の背景の違いを覚えておくだけです。関数のプロトタイプは、`se` が `ase` に置き換わっていること以外は同じです。

内部での唯一の本当の違いは、描画処理が出力するもの、つまりハーフワードではなくバイトであることです。そしてここで、VRAM のあの風変わりな事実にまたぶつかります。VRAM には 1 バイトだけを書き込めないのです。つまり、描画処理が少し複雑になります。でも少しだけです。スクリーンエントリを置くのに、バイトを描くルーチンを呼ぶだけです。アフィンのマップは本質的に 8bpp のビットマップの描画先なので、8bpp のビットマップの描画先のための標準の描画関数 `_sbmp8_plot()` が使えます。この 1 つの違いを除けば、`ase_` の描画処理は `se_` のものと同じです。

```c {#cd-ase-drawg .proglist}

//! Character-plot for affine BGs using an 8x8 font.
void ase_drawg_w8h8(uint gid)
{
    TTC *tc= tte_get_context();
    u8 se= tc->cattr[TTE_SPECIAL] + gid;

    _sbmp8_plot(&tc->dst, tc->cursorX/8, tc->cursorY/8, se);
}

//! Character-plot for affine BGs, any sized, vertically oriented font.
void ase_drawg_s(int gid)
{
    TTC *tc= tte_get_context();
    TFont *font= tc->font;
    uint charW= (font->cellW+7)/8, charH= (font->cellH+7)/8;
    uint x0= tc->cursorX/8, y0= tc->cursorY/8;

    u8 se= tc->cattr[TTE_SPECIAL] + gid*charW*charH;

    int ix, iy;
    for(ix=0; ix<charW; ix++)
        for(iy=0; iy<charH; iy++, se++)
            _sbmp8_plot(&tc->dst, ix+x0, iy+y0, se);
}
```

アフィンのマップのテキストのデモは `text_tte_ase()` です。考え方は単純で、256×256 ピクセルのマップにテキストを準備し、そこに文字を書いて、本当にアフィン背景であることを示すために背景を回転させます。回転の中心は、画面の中央の「o」です。そこに置くのに、`#{P:x,y}` コードを使っています。これはカーソルを (x, y) で与えられる絶対位置に設定します。もう 1 つの文字列も、この方法でマップ上に配置しています。

```c {#cd-test-ase .proglist}
void test_tte_ase()
{
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    REG_DISPCNT= DCNT_MODE1 | DCNT_BG2;

    // Init affine text for 32x32t bg
    tte_init_ase(
        2,                      // BG number
        BG_CBB(0) | BG_SBB(28) | BG_AFF_32x32,  // BG control
        0,                      // Tile offset (special cattr)
        CLR_YELLOW,             // Ink color
        0xFE,                   // BUP offset (on-pixel = 255)
        NULL,                   // Default font (sys8)
        NULL);                  // Default renderer (ase_drawg_s)

    // Write something
    tte_write("#{P:120,80}o");
    tte_write("#{P:72,104}Round, round, #{P:80,112}round we go");

    // Rotate it
    AFF_SRC_EX asx= { 124<<8, 84<<8, 120, 80, 0x100, 0x100, 0 };
    bg_rotscale_ex(&REG_BG_AFFINE[2], &asx);

    while(1)
    {
        VBlankIntrWait();
        key_poll();

        asx.alpha += 0x111;
        bg_rotscale_ex(&REG_BG_AFFINE[2], &asx);

        if(key_hit(KEY_START))
            break;
    }
}
```

<div class="cpt" style="width:240px;">
  <img src="img/tte/test_tte_ase.png" id="fig:img-test-ase"
    alt="<code>test_tte_ase</code>."><br>
  <b>{*@fig:img-test-ase}</b>: <code>test_tte_ase</code>。
</div>

## ビットマップのテキスト {#sec-bmp}

ビットマップのテキストの描画はマップのテキストとは少し違い、望むものによって、難しさは簡単なものから正気の沙汰でないものまであります。でも、その核心は常に同じ処理です。すべてのピクセルをたどって、描画先に描くのです。たとえば、ピクセルを透明に描く汎用のグリフの描画処理は、次のようなものになるでしょう。

```c {.proglist}
// Pseudo code for a general glyph printer.
void foo_drawg(uint gid)
{
    TTC *tc= tte_get_context();
    TFont *font= tc->font;

    // Drawing with color keying.
    // Loop over all pixels. If the glyph's pixel is not zero, draw it.
    // Other wise, nevermind.
    for(iy=0; iy<tte_get_glyph_height(gid); iy++)
    {
        for(ix=0; ix<tte_get_glyph_width(gid); ix++)
        {
            u16 color= font_get_pixel(font, gid, ix, iy);
            if(color != 0)
                foo_plot(&tc->dst, tc->cursorX+ix, tc->cursorY+iy, color);
        }
    }
}
```

ここで _`foo`_ はどんな描画ファミリーでもかまいません。`foo_plot()` は汎用のピクセルの描画関数で、`font_get_pixel()` はピクセルの取得関数です。これらの関数の実装はフォントと描画先の詳細によりますが、グリフの描画処理はそれを知る必要がありません。

### bmp16 から bmp16 への基本のグリフ表示処理 {#ssec-bmp16-base}

次の関数は、16bpp のフォントを 16bpp のビットマップに表示する処理の例です。

```c {#cd-bmp16-drawg .proglist}
//! Glyph renderer from bmp16 glyph to bmp16 destination.
void bmp16_drawg(uint gid)
{
    // (1a) Basic variables
    TTC *tc= tte_get_context();
    TFont *font= tc->font;

    u16 *srcD= (u16*)(font->data+gid*font->cellSize), *srcL= srcD;
    uint charW= tte_get_glyph_width(gid)
    uint charH= tte_get_glyph_height(gid);

    uint x0= tc->cursorX, y0= tc->cursorY;
    uint srcP= font->cellW, dstP= tc->dst.pitch/2;
    u16 *dstD= (u16*)(tc->dst.data + (y0*dstP + x0)*2);

    // (2) The actual rendering
    uint ix, iy;
    for(iy=0; iy<charH; iy++)
    {
        for(ix=0; ix<charW; ix++)
            if(srcD[ix] != 0)
                dstD[ix]= srcD[ix];

        srcD += srcP;
        dstD += dstP;
    }
}
```

ブロック 1a と 1b は、ループで使う主な変数を準備します。最も重要なのは、読み込み元と書き込み先のポインタ `srcD` と `dstD`、そしてそれらのピッチ `srcP` と `dstP` です。フォントはセルのグリッドで構成されているので、読み込み元のピッチ `srcP` は文字の幅ではなく、セルの幅であることに注意してください。ポイント 2 のコードは、フォントから描画先へピクセルを選択的にコピーします。

#### 幕間: 性能についての検討

なぜ `bmp16_drawg()` が先ほどの `foo_drawg()` の形にもっと忠実に従っていないのか、不思議に思うかもしれません。答えはもちろん性能です。そして、誰かが私に Knuth を引用する前に言っておくと、コードを速くしようとする努力のすべてが早すぎる最適化というわけではありません。それほど手間をかけずに、読みやすさを損なうことなくコードの速さを上げられるなら、そうしない理由はあまりありません。

この場合、ここで適用した最適化は 2 つの種類に分けられます。ローカル変数とポインタ演算です。これらの技法は、すべての C プログラマが知っている_べき_ものですが、これで速さを 5 倍に上げられました。

ポインタから始めましょう。ここでは 2 か所でポインタを活用しています。まず、フォントのデータのポインタと書き込み先のポインタを直接使う代わりに、ポインタ `srcD` と `dstD` を作り、グリフの左上と、それを描画する先に向けています。このようにアクセスを近道させると、ループの中で行きたいところに行くために余分なオフセットを加える必要がなくなります。ループに本質的でない式が入らないので、速くなるうえに、実は読みやすくもなります。

```c {.proglist}
//# Example of a more standard bitmap copier.
for(iy=0; iy < charH; iy++)
    for(ix=0; ix < charW; ix++)
        dstD[iy*dstP + ix]= srcD[iy*srcP+ix];
```

2 つ目の点は、`y*pitch+x` の形の代わりに、増分のオフセットを使うことです（上を見てください）。これはほとんど好みの問題だと思いますが、まったく不要な乗算を避けることは効いてきます。

2 つ目の最適化はローカル変数です。つまり、メモリにある変数（グローバル変数や構造体のメンバ）や、よく使う関数の結果を、ローカルの一時変数に読み込んでおくということです。指摘するのもばかばかしく思えるかもしれませんが、これで節約できる時間は実はかなり大きいのです。

ここでの `tte_get_glyph_width()` の使い方を考えてみてください。グリフの幅がループの間に変わらないことは_わかっている_ので、ループの条件そのものの中で幅を得るために関数を呼ぶのはただの愚かなことです。別の例は、文字列のすべての文字をループでたどるときに `strlen()` を呼ぶことです。そうしている人へ: だめ！ 悪いプログラマ、だめですよ！ 値をローカル変数に保存して、代わりにそれを使ってください。

もう 1 つの点は、グローバル変数や構造体/クラスのメンバのようなものを、2 回以上使うなら前もって読み込んでおくことです。次のコードを考えてみてください。先ほどのものと同じですが、今度は文字の高さとピッチをローカルの一時変数に読み込んでいません。

```c {.proglist}
//# Another bitmap-copy example. DO NOT USE THIS !!!
for(iy=0; iy < font->charH; iy++)
    for(ix=0; ix < charW; ix++)
        dstD[iy*tc->dst.pitch/2 + ix]= srcD[iy*font->cellW + ix];
```

結果: 関数の速さは**半分**になりました！ 遅くなるとは予想していましたが、この無害に見える変更で実際に 2 倍も遅くなるとは、かなりの驚きでした。

ですから、そう、ループの中で変わらないメモリ上の量には、ローカル変数をどんどん使ってください。そうすれば、毎回メモリから読み込まれずに済みます。おまけに、ループそのものの記述が短くなり、より一般化されるので、再利用しやすくなります。

ポインタの工夫も変数の事前の読み込みも、本当はコンパイラの最適化の仕事なのですが、現在の版の GCC はこれらをあまりうまくやらないか、まったくやりません。また、この最適化が_できない_こともあります。メモリの参照の間に関数が呼ばれると、それらの関数が中身を変えたかもしれないので、コンパイラはデータを読み込み直さなければなりません。当然、ローカル変数ではそういうことは起きません。

:::tip 構造体のメンバやグローバル変数にはローカル変数を使おう

構造体のメンバやグローバル変数は、CPU のレジスタではなくメモリにあります。CPU がそのデータを使えるようになる前に、まずメモリからレジスタに読み込む必要があり、これは必要以上に何度も起きることがよくあります。メモリへのアクセス（特に ROM へのアクセス）はレジスタへのアクセスより遅いので、それを 2 回以上使うと、アルゴリズムの足を本当に引っ張りかねません。ローカル変数を作れば、この無駄な作業を避けられます。

速さが上がるだけでなく、ローカル変数には短い名前を使えるので、実際に仕事をする部分のコードが短く読みやすくなります。どっちに転んでも得しかありませんよ、ベイビー。

:::

### グリフと描画先の形式 {#ssec-srf-format}

上で説明した描画処理は、グリフが 16bpp のビットマップの形式であることを前提にしています。しかし、TTE のデフォルトのフォントは 1bpp のタイルストリップ形式なので、別のものを使う必要があります。その関数の詳細に入る前に、いろいろなグリフの形式と、なぜ普通のビットマップではなくタイルストリップを使っているのかを説明しておきます。

グリフの形式と言うとき、本当に意味しているのは、ピクセルにアクセスする順序です。主な変種は 3 つあります。

- **線形**または**ビットマップ**の配置。これは単純な行優先の行列です。ループは 2 つで、_y_ 用と _x_ 用です。
- **タイル**。具体的には 8×8 のタイルです。これは GBA の標準のタイル形式で、8×8 ピクセルの各グループが行優先の行列を作り、さらにタイルそのものがまたより大きな行優先の行列の一部になっています。これをたどるには**4 つ**のループが必要です。それぞれの行列に 2 つずつです。
- **タイルストリップ**。これも 8×8 のタイルを使いますが、今度はタイルが列優先の順序で並んでいます。言い換えると、タイル 1 はタイル 0 の右ではなく下に来ます。これには、続くタイルの行が連続しているというなかなか素敵な性質があります。_y_ 方向の切れ目がなくなるので、ループは 3 つで済み、しかもコードが単純になります。

{\*@fig:img-src-loops} は、これら 3 つの配置を、ループの構造と、1bpp のフォントでピクセルをたどる順序も含めて示しています。ビットパックのせいで、事情は少し違います。1 bpp は 1 バイトに 8 ピクセルということです。その結果、ビットマップの _x_ のループは 8 個ずつのグループに分けなければならず、ビットマップの形式では 2 つではなく 3 つの入れ子のループを使うことになります。タイルの形式では、どのみち 8 ピクセルずつまとまっているので違いはありません。それだけでなく、よく使われるグリフの大きさについてループのオーバーヘッドの合計を計算してみると、この配置は実はタイルストリップで特にうまくいくことがわかります。これは読者への演習として残しておきます（ヒント: 比較の回数を数えてください）。

<div class="cblock">
  <div class="cpt" style="width:600px;">
    <img src="img/tte/src_loops.png" id="fig:img-src-loops" width=600 alt="Pixel traversal in glyphs for bitmap, tile and tile-strip formats. ">
    <br>
    <b>{*@fig:img-src-loops}</b>: 1bpp のビットマップ、タイル、タイルストリップの形式での、グリフのピクセルのたどり方。数字はループとその入れ子を示します。
  </div>
</div>

### bmp16_drawg_b1cts: 透明と色付けありの 1bpp から 16bpp {#ssec-bmp16-std}

次のルーチンは、1bpp のタイルストリップのグリフを取り、16bpp のビットマップ背景に合った出力に変えます。出力は色にインクの属性を使い、元のデータでビットが 1 の場合にだけピクセルを描くので、透明になります。先頭の 3 つのマクロは基本の変数を宣言、定義するもので、`bmp16_drawg()` の手順 1a に相当します。

<pre><code class="language-c hljs">
void bmp16_drawg_b1cts(uint gid)
{
    // (1) Basic variables
    TTE_BASE_VARS(tc, font);
    TTE_CHAR_VARS(font, gid, <span class="bold">u8</span>, srcD, srcL, charW, charH);
    TTE_DST_VARS(tc, <span class="bold">u16</span>, dstD, dstL, dstP, x0, y0);
    uint srcP= font-&gt;cellH;

    dstD += x0;

    u32 ink= tc-&gt;cattr[TTE_INK], raw;

    // (2) Rendering loops.
    uint ix, iy, iw;
    for(iw=0; iw&lt;charW; iw += 8)            // loop over tile-strips
    {
        dstL= &amp;dstD[iw];
        for(iy=0; iy&lt;charH; iy++)           // loop over tile-lines
        {
            raw= srcL[iy];
            for(ix=0; raw&gt;0; raw&gt;&gt;=1, ix++) // loop over tile-scanline (8 pixels)
                if(raw&amp;1)
                    dstL[ix]= ink;

            dstL += dstP/2;
        }
        srcL += srcP;
    }
}
</code></pre>

このルーチンは、3 つのマクロ `TTE_CHAR_VARS()`、`TTE_CHAR_VARS`、`TTE_DST_VARS()` の呼び出しで始まります。これらは関係するローカル変数のほとんどを宣言、定義するもので、`bmp16_drawg()` の手順 1a と似ています。ここでの引数のうち 2 つは、それぞれ読み込み元と書き込み先のポインタのデータ型の識別子であることに注意してください。`srcD` と `srcL` は、最初は読み込み元のデータの先頭を指します。ほかのポインタ `dstD` と `dstL` は、書き込み先の領域の_スキャンラインの_先頭を指します。まだ _x_ の位置は補正されていません。それはそのすぐあとで行います。

ここで 2 組のポインタ（主な_データ_のポインタ `fooD` と_ライン_のポインタ `fooL`）を使っているのは、ポインタ演算のためです。データのポインタは固定したままで、ラインのポインタが内側のループの中で動き回ります。

{@fig:img-src-loops} のタイルストリップの部分は、このルーチンがすべてのピクセルをどう動いていくかを示しています。1bpp でビットパックされたフォントで、タイルの 1 ラインあたり 8 ピクセルなので、1 回のバイトの読み込みで 1 ラインぶんのピクセルをまるごと得られます。透明に描くことで、良い最適化の機会も得られます。タイルのラインが空（つまり `raw`==0）なら、そのラインにはもう見えるピクセルはないので、次に進めます。{@fig:img-verdana9} の verdana 9 のフォントをちらっと見れば、このおかげでピクセルの 50% を飛ばせるかもしれないことがわかるでしょう。

### bmp8_drawg_b1cts: 透明と色付けありの 1bpp から 8bpp {#ssec-bmp8-std}

前の関数の 8bpp 版は `bmp8_drawg_b1cts()` という名前で、下に示します。コードは 16bpp の関数ととてもよく似ていますが、今度はピクセルがバイトなので、細かいところでいくつか違いがあります。

<pre><code class="language-c hljs">void bmp8_drawg_b1cts(uint gid)
{
    // <span class="bold">(1)</span> Basic variables
    TTE_BASE_VARS(tc, font);
    TTE_CHAR_VARS(font, gid, u8, srcD, srcL, charW, charH);
    TTE_DST_VARS(tc, u16, dstD, dstL, dstP, x0, y0);
    uint srcP= font-&gt;cellH;

    dstD += x0/2;

    u32 ink= tc-&gt;cattr[TTE_INK], raw, px;
    uint odd= x0&amp;1;                         // <span class="bold">(2)</span> Source offset.

    uint ix, iy, iw;
    for(iw=0; iw&lt;charW; iw += 8)            // Loop over strips.
    {
        dstL= &amp;dstD[iw/2];
        for(iy=0; iy&lt;charH; iy++)           // Loop over lines.
        {
            raw= srcL[iy]&lt;&lt;odd;             // <span class="bold">(3)</span> Apply source offset.
            for(ix=0; raw&gt;0; raw&gt;&gt;=2, ix++) // Loop over pixels.
            {
                // <span class="bold">(4)</span> 2-bit -> 2-byte unpack, then used as masks.
                px= ( (raw&amp;3)&lt;&lt;7 | (raw&amp;3) ) &amp;~ 0xFE;
                dstL[ix]= (dstL[ix]&amp;~(px*255)) + ink*px;
            }
            dstL += dstP/2;
        }
        srcL += srcP;
    }
}

</code></pre>

`bmp16_drawg_b1cts` との唯一の本当の違いは、一番内側のループにあります。VRAM にはバイト単位で書き込めないという問題があるので、1 回で 2 ピクセルを書く必要があります。そのために、2 つのビットを取り出して 2 つのバイトに展開し、それらを使って新しいピクセルとピクセルのマスクを作ります。内側のループの最初の行が展開を行います。ビットのパターン _`ab`_ を `0000 000`_`a`_` 0000 000`_`b`_ に変換します。これで、このハーフワードの両方のバイトは、_a_ と _b_ がオンかオフかに応じて 0 か 1 になります。`ink` と 255 を掛ければ、色の付いたピクセルと、挿入のための適切なマスクが得られます。

<pre><code class="language-sh hljs"># 2-bit to 2-byte unpacking.
0000 0000  hgfe dcb<b>a</b>    p  = raw (start)
0000 0000  0000 00<span class="rem">b<b>a</b></span>    p &amp;= 3
0000 000<span class="rem"><b>b</b>  a</span>000 00b<b>a</b>    p |= p&lt;&lt;7;
0000 000<b>b</b>  0000 000<b>a</b>    p &amp;= ~0xFE;
</code></pre>

正しいハーフワードを用意するのは仕事の一部にすぎません。`cursorX`（つまり `x0`）が奇数なら、グリフも奇数の開始位置に描かなければなりません。しかし、書き込み先のポインタ `dstL` はハーフワードのポインタで、常にハーフワード境界に揃っていなければなりません。これに対処するには、パターン「`abcd efgh`」を奇数の境界に展開するのは、「`a bcde fgh`**`0`**」を偶数の境界に展開するのと同じであることに注目します。`odd` による余分なシフトは、まさにそのためのものです。

### 例: サブピクセルレンダリング {#ssec-bmp-demo}

この節のデモでは、[<dfn>サブピクセルレンダリング</dfn>](https://en.wikipedia.org/wiki/Subpixel_rendering)と呼ばれる技法を使ってみたいと思います。これは、ほかのピクセルから色を「借りる」ことで、描画の横方向の解像度を実質的に 3 倍にする方法です。

{@fig:img-subpx}a に示す文字「A」を考えてください。ご存じのとおり、各ピクセルは赤、緑、青の 3 つの色でできています。これらがサブピクセルです。サブピクセルのグリッド上では、文字は {@fig:img-subpx}b のようになります。色がまだピクセル単位でまとまっていて、サブピクセルのグリッド上ではとてもギザギザの縁になっていることに注目してください。サブピクセルレンダリングのコツは、サブピクセルのまとまりを左右にずらして、縁をなめらかにすることです（{@fig:img-subpx}c）。それからピクセルをまた RGB の色に組み合わせると、{@fig:img-subpx}d になります。{@fig:img-subpx} のように拡大すると、サブピクセルレンダリングは大したことがないように見えるかもしれませんが、適切な大きさで使うと、その効果はかなり見事なものになりえます。

<div class="cblock">
  <div class="cpt" style="width:576px;">
    <img src="img/tte/subpx.png" id="fig:img-subpx" alt="Subpixel rendering. Left: ">
    <br>
    <b>{*@fig:img-subpx}</b>: サブピクセルレンダリング。
    <b>a</b>: 8&times;8 のグリッド上の「A」。
    <b>b</b>: 左と同じものを R、G、B のサブグリッドで。
    <b>c</b>: サブピクセルを均等に分配するために行をずらしたもの。
    <b>d</b>: ピクセルでの新しい色の分布（黒と白は反転）。
  </div>
</div>

サブピクセルレンダリングは何にでも役立つわけではありません。ピクセルと色の概念を少し混ぜてしまうので、グレースケールの画像にしか役立ちません。もちろん、だからこそテキストにはうってつけなのです。次に、サブピクセルの並び順も重要です。**{@fig:img-subpx}** に示した処理は RGB の順の画面ではうまくいきますが、ピクセルが BGR の順なら見事に失敗します。細かいことを全部説明するのはここでは多すぎるので、[http://www.grc.com/ctwhat.htm](https://www.grc.com/ctwhat.htm) を紹介しておきます。そこでは考え方をもっと詳しく説明していて、例もいくつか載っています。

<div class="cpt_fr" style="width:128px;">
  <img src="img/tte/yesh1.png" id="fig:img-yesh" width=128 alt="4&times;8 subpixel font">
  <br>
  <b>{*@fig:img-yesh}</b>: 4&times;8 のサブピクセルフォント
</div>

JanoS（[http://www.haluz.org/yesh/](http://www.haluz.org/yesh/)）が、GBA と NDS で使える素敵な 4×8 のサブピクセルフォントを作りました（{@fig:img-yesh} を見てください）。幅 4 は本当に小さく、普通の描画ではその大きさのグリフで読めるテキストにすることは不可能です。しかし、サブピクセルレンダリングならまだ見栄えが良く、普段よりずっと多くの文字を画面に出せます。

デモの出力は {@fig:img-test-bmp16} で見られます。サブピクセルレンダリングは見ているハードウェアに強く結びついているので、ほとんどの画面や紙ではおそらくひどく見えるでしょう。効果を十分に味わうには、本当に GBA の画面で見る必要があります。

この場合は、フォントを `bmp16_drawg()` で使えるように変換しました。ビットマップの配置の 16bpp のフォントです。8 ビット版を作るのもそれほど難しくないでしょう。このフォントには 2 色より多くの色があるので、1bpp にビットパックしたフォントはもちろん不可能です。サブピクセルフォントを正しく見せるには、実はたくさんの色が必要です。R、G、B の組み合わせごとに 1 色、しかもそれぞれの異なる濃淡が必要です。とはいえ、JanoS は質をあまり落とさずに色の数を 20 にまで減らしてくれました。欲しい人がいれば、4bpp のフォントで使うための 15 色版もあります。

```c {#cd-tte-test-bmp16 .proglist}
//! Testing a bitmap renderer with JanoS' sub-pixel font.
void test_tte_bmp16()
{
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    REG_DISPCNT= DCNT_MODE3 | DCNT_BG2;

    tte_init_bmp(3, &yesh1Font, bmp16_drawg);
    tte_init_con();

    const char *str=
    "https://en.wikipedia.org/wiki/Subpixel_rendering :\n"
    "Subpixel rendering is a way to increase the "
    "apparent \nresolution of a computer's liquid crystal "
    "display (LCD).\nIt takes advantage of the fact that "
    "each pixel on a color\nLCD is actually composed of "
    "individual red, green, and\nblue subpixel stripes to "
    "anti-alias text with greater\ndetail.\n\n"
    "  4x8 sub-pixel font by JanoS.\n"
    "  http://www.haluz.org/yesh/\n";

    tte_write(str);
    key_wait_till_hit(KEY_ANY);
}
```

<div class="lblock">
  <div class="cpt" style="width:240px;">
    <img src="img/tte/test_tte_bmp16.png" id="fig:img-test-bmp16" alt="Sub-pixel rendering demo">
    <br>
    <b>{*@fig:img-test-bmp16}</b>: サブピクセルレンダリングのデモ
  </div>
</div>

## オブジェクトのテキスト {#sec-obj}

オブジェクトのテキストは、文字を少し動かしたいときや、単に背景に空きがないときに役立ちます。オブジェクトのテキストにはいくつかの可能性があります。一番明らかなのは、すべての文字をオブジェクトの VRAM に読み込み、オブジェクトのタイルのインデックスを正しいタイルを使うように設定することです。TTE のオブジェクトのシステムはこれを使います。

多くの点で、この種のオブジェクトのテキストはタイルマップのテキストと似ています。タイルは前もって読み込んでおき、関係する対応づけのエントリ（この場合はオブジェクトの `attr2`）を正しい番号に変えます。もちろん、目立った違いもいくつかあります。

1 つには、文字の位置をオブジェクトに書き込まなければなりません。それだけでなく、オブジェクトは自分がどれくらいの大きさになるべきか、回転やパレットのようなほかの興味深い性質を持つかどうかも知る必要があります。そのため、色の属性 0、1、2 を使ってオブジェクトの属性 0、1、2 を格納することにしました。

もう 1 つの問題は、どのオブジェクトを、いくつ使うかです。後者は実は大きな問題になりえます。普通のスプライトにもオブジェクトを使いたいかもしれず、それが突然テキストシステムに上書きされたら、本当にまずいことになるからです。

後者の問題には、コンテキストの `dst` メンバを使います（あるいは悪用します）。各グリフは 1 つのオブジェクトで表されるので、オブジェクトの配列が必要ですが、ちょっとひねったやり方をします。配列の_最後_から始めて、下のほうのオブジェクトは普通どおりスプライトに使えるようにするのです。要するに、OAM を空の下降スタックとして使っています。この構成では、`dst.data` はスタックの頂上（つまり配列の最後の要素）を指し、`dst.pitch` は現在のオブジェクトのインデックス、`dst.width` はスタックの長さです。

オブジェクトのデフォルトの描画関数は `obj_drawg` です。ここでは `dst.pitch` はインデックスとして使われ、`dst.data` はスタックの頂上なので、現在のオブジェクトを得るには_負の_インデックスを使うことを思い出してください。そのあとで、座標と正しいグリフのインデックスを色の属性と組み合わせて、最終的なオブジェクトを作ります。

```c {#cd-obj-drawg .proglist}
//! Glyph-plotter using objects.
void obj_drawg(uint gid)
{
    TTC *tc= tte_get_context();
    TFont *font= tc->font;
    uint x0= tc->cursorX, y0= tc->cursorY;

    // (1) find the right object, and increment index.
    uint id= tc->dst.pitch;
    OBJ_ATTR *obj= &((OBJ_ATTR*)tc->dst.data)[-id];
    tc->dst.pitch= (id+1 < tc->dst.width ? id+1 : 0);

    // (2) Set object attributes.
    obj->attr0= tc->cattr[0] + (y0 & ATTR0_Y_MASK);
    obj->attr1= tc->cattr[1] + (x0 & ATTR1_X_MASK);
    obj->attr2= tc->cattr[2] + gid*font->cellW*font->cellH/64;
}
```

そして、ええ、`dst` メンバのこの使い方が少し……型破りなのはわかっています。でも、どのみちここでは使われていなかったので、別にいいでしょう。もっとまともなものを使うことも考えていますが、まだです。また、このシステムはフォントがすでに VRAM に読み込まれていることを前提にしていて、それが使えるタイルの_かなり_の部分を占めうることも覚えておいてください。verdana 9 のフォントを使うと、2\*240 = 480 タイルになります。それはオブジェクトの VRAM のほぼ半分です。もっと安全な代わりの方法は、必要なタイルを動的に読み込むことですが、それにはもっとリソースの管理が必要になります。

:::warning TTE のオブジェクトのテキストは見苦しい

TTE でのオブジェクトのテキストの扱い方は動きはしますが、実装はきれいとは言えません。ここでの `TTC.dst` の使い方は、まあ、良くありません。あとで少しきれいにするか、少なくとも実装をもっとうまく隠す可能性は高いです。

:::

### 例: 文字たち。経路の上で {#ssec-obj-demo}

オブジェクトの決定的な特徴は、背景から独立していて、画面上を自由に動き回れることです。オブジェクトのテキストは、動的なテキストや、何らかの経路に沿って移動しなければならないテキストに使われることが多いでしょう。この場合は、[リサージュ曲線](https://en.wikipedia.org/wiki/Lissajous_curve)と呼ばれる媒介変数表示の経路の上を飛ばします（{@fig:img-test-obj} を見てください）。

<div class="cpt_fr" style="width:240px;">
  <img src="img/tte/test_tte_obj.png" id="fig:img-test-obj" alt="Object text on a path.">
  <br>
  <b>{*@fig:img-test-obj}</b>: 経路の上のオブジェクトのテキスト。
</div>

コードを下に示します。いつもの面々を初期化したあとで、`tte_init_obj()` を呼びます。オブジェクトのスタックは OAM の後ろから始まります。最初のパラメータに `NULL` を渡した場合も、システムはそれをデフォルトにします。次の 3 つはオブジェクトの属性です。デフォルトの可変幅のフォント verdana 9 を使いたいので、属性は 8×16 のオブジェクトに設定するべきです。使うタイルの数を制限内に収めるため、タイルのビット深度は常に 4 です。初期化の残りは簡単に理解できるはずです。

文字列そのものを作るのは手順 2 です。“omg” を赤くするために、文字列で紙の色の属性（obj.attr2 に対応します）も 0x1000 に設定していることに注意してください。この数行で、テキストの処理自体は完了です。

手順 3 では、経路上の座標を計算します。`t` パラメータは、経路上をどれだけ進んだかを示します。これを使って各文字の座標を計算します。最初の文字は `t` そのものを使い、残りは基本的に時間を遅らせたものです。マジックナンバーに気を取られないでください。それらの値の理由は、効果を見栄え良くすることだけです。少しいじってみて、正確に何をするのか確かめてみてください。

```c {#cd-test-obj .proglist}
// Object text demo
void test_tte_obj()
{
    // Base inits
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    REG_DISPCNT= DCNT_MODE0 | DCNT_OBJ | DCNT_OBJ_1D;
    oam_init(oam_mem, 128);
    VBlankIntrWait();

    // (1) Init object text, using verdana 9 (8x16 objects)
    OBJ_ATTR *objs= &oam_mem[127];
    tte_init_obj(
        objs,               // Start at back of OAM
        ATTR0_TALL,         // attr0: 8x16 objects
        ATTR1_SIZE_8,       // attr1: 8x16 objects
        0,                  // attr2: nothing special
        CLR_YELLOW,         // Yellow ink
        0x0E,               // ink pixel 14+1 = 15
        &vwf_default,          // Verdana 9 font
        NULL);              // Default renderer (obj_drawg)

    pal_obj_bank[1][15]= CLR_RED;

    // (2) Write something (and prep for path)
    const char *str= "Parametrized object text, omg!!!";
    const int len= strlen(str);
    tte_write("Parametrized object text, #{cp:0x1000}omg#{cp:0}!!!");

    // Play with the objects
    int ii, t= 0x9000;
    while(1)
    {
        VBlankIntrWait();
        key_poll();

        // (3) Make lissajous figure
        for(ii=0; ii<len; ii++)
        {
            int ti= t-0x380*ii;             // Get the path param for letter ii
            obj_set_pos(&objs[-ii],
                (96*lu_cos(  ti)>>12)+120,  // y= Ay*cos(  t) + y0
                (64*lu_sin(2*ti)>>12)+80);  // x= Ax*sin(2*t) + x0
        }
        t += 0x00A0;

        if(key_hit(KEY_START))
            break;
    }
}
```

## タイルへの描画 {#sec-chr}

テキストにタイルマップを使うのは便利ですが、グリフの寸法が 8 の倍数の場合にしか使えません。読みやすさの点でもいくつか欠点があります。「i」のような細い文字は、幅が広すぎるように見えるか、たくさんの空のピクセルに囲まれてしまいます。また、タイルの数には限りがあるので、1 行にたくさんの文字を置けません。

可変幅のフォント（<dfn>vwf</dfn>。プロポーショナルフォントとも呼ばれます）はこの問題を解決します。[「ビットマップのテキスト」の節](#sec-bmp)で示したように、ビットマップで可変幅のフォントを使うのはとても簡単です。しかし、タイルマップのモードで使うのは少しやっかいです。タイルの大きさが 8×8 のタイルマップに、どうやって描けばいいのでしょうか？

まあ、描かないのです。正確には違います。鍵は、マップに描くのではなく、マップが表示しているタイルに描くことです。

### 基本のタイルへの描画 {#ssec-chr-base}

タイルマップの普通の使い方は、タイルセットを読み込み、タイルマップを埋めることで、画面に表示したいタイルを選ぶというものです。その場合、タイルセットは静的なことが多く、スクロールなどのためにマップのほうを更新します。タイルへの描画は、その手順を逆にします。

まず、各エントリが固有のタイルを指すマップを準備する必要があります。これで基本的に、タイルからグラフィックの描画先ができ、ほかのものと同じようにそこに描けるようになります。一番明らかなやり方は、スクリーンブロックを単に連続した番号で埋めることです（{@fig:img-chr4-map}a を見てください）。しかし、もっと良いやり方は、タイルを列優先の順序で対応づけることです（{@fig:img-chr4-map}b を見てください）。グリフの形式でそれを選んだのと同じ理由で、タイルの列の中のワードが連続するからです。

<div class="cblock" id="fig:img-chr4-map">
  <table border=0 cellpadding=4 cellspacing=0 width=70%>
    <tr>
      <td>
        <img src="img/tte/chr4r_map.png" alt="">
        <b>{*@fig:img-chr4-map}a</b>。行優先のタイルのインデックス。
      </td>
      <td>
        <img src="img/tte/chr4c_map.png" alt="">
        <b>{*@fig:img-chr4-map}b</b>。列優先のタイルのインデックス。
      </td>
    </tr>
  </table>
</div>

マップの準備は簡単な部分です。問題は、ピクセルを描くのに、どのタイルのどの部分を編集すればいいかを知ることです。まず、座標をタイルの座標と、タイルの中のピクセルの座標に分ける必要があります。これは、それぞれ 8 による除算と剰余になります。列優先のモードでは、これを _x_ 座標についてだけ行えばよいことに注意してください。この情報で、正しいワードを見つけられます。タイルの中の横方向の座標から、ワードのどの 4 ビットを更新すればいいかがわかり、そこからはいつものビットフィールドの挿入です。

Tonclib には、4bpp で列優先のタイル（<dfn>chr4c</dfn> モードと呼びます）に描くためのルーチンがあります。描画関数とマップの準備の関数を、その使い方を説明するためのデモのルーチンとともに下に示します。

```c {#cd-chr4c-test .proglist}
//# From tonc_schr4c.c

//! Plot a pixel on a 4bpp tiled, column-major surface.
void schr4c_plot(const TSurface *dst, int x, int y, u32 clr)
{
    uint xx= x;     // fluff to make x unsigned.
    u32 *dstD= (u32*)(dst->data + xx/8*dst->pitch);
    uint shift= xx%8*4;

    dstD[y] = (dstD[y] &~ (15<<shift)) | (clr&15)<<shift;
}

//! Prepare a screen-entry map for use with chr4c mode.
void schr4c_prep_map(const TSurface *srf, u16 *map, u16 se0)
{
    uint ix, iy;
    uint mapW= srf->width/8, mapH= srf->height/8, mapP= srf->pitch/32;

    for(iy=0; iy<mapH; iy++)
        for(ix=0; ix<mapW; ix++)
            map[iy*32+ix]= ix*mapP + iy + se0;
}

//# --- Simple test ---------------------------------------------------

void test_chr4()
{
    // (1) The usual
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;
    REG_BG0CNT= BG_CBB(0) | BG_SBB(31);

    pal_bg_mem[1]= CLR_RED;
    pal_bg_mem[2]= CLR_GREEN;
    pal_bg_mem[3]= CLR_BLUE;
    pal_bg_mem[4]= CLR_WHITE;

    // (2) Define a surface
    TSurface srf;
    srf_init(&srf,
        SRF_CHR4C,          // Surface type.
        tile_mem[0],        // Destination tiles.
        SCREEN_WIDTH,       // Surface width.
        SCREEN_HEIGHT,      // Surface height.
        4,                  // Bitdepth (ignored due to SRF_CHR4C).
        pal_bg_mem);        // Palette.

    // (3) Prepare the map
    schr4c_prep_map(&srf, se_mem[31], 0);

    // (4) Plot some things
    int ii, ix, iy;
    for(iy=0; iy<20; iy++)
        for(ix=0; ix<20; ix++)
            schr4c_plot(&srf, ix+3, iy+11, 4);

    for(ii=0; ii<20; ii++)
    {
        schr4c_plot(&srf, ii+4,    12, 1);  // Red line
        schr4c_plot(&srf, ii+4, ii+12, 2);  // Green line
        schr4c_plot(&srf,    4, ii+12, 3);  // Blue line
    }
}
```

ピクセルの描画関数は、目的のピクセルがあるタイルの列を求めるところから始まります。列のインデックスは単に _x_/8 です。これにピッチを掛けて、その列の一番上へのポインタを得ます。ピッチの使い方がいつもと少し違うことに注意してください。普通は次のスキャンラインまでのバイト数を表しますが、この場合は次のタイルの列までのバイトのオフセットとして使っています。列優先のモードでは、これは *高さ*×*bpp*\*8/8 になりますが、それはすべて `srf_init()` の中で行われます。正しいタイルが得られたら、目的のピクセルは _x_%8<sup>th</sup> の 4 ビットにあるので、挿入に必要なシフトは _x_%8\*4 です。そのあとは色を挿入するだけです。<span class="mini">（気になる人のために: _x_ を最初に unsigned int にキャストしているのは、そうすると除算と剰余がきちんとシフトとマスクに最適化されるからです。）</span>

`schr4c_prep_map()` 関数は、{@fig:img-chr4-map}b に示した順序でマップを初期化するだけです。まあ、ほぼそうです。パレットやタイルのオフセットのためにいつもしているように、各スクリーンエントリに値を加えてもいます。

`test_chr4()` の出力は {@fig:img-chr4-test}a で見られます。予想どおり、赤、緑、青の線がある白い長方形です。{\*@fig:img-chr4-test}b は VBA のタイルビューアから取った画像で、描画先の中身がどうなっているかを示しています。画面に映っているものとはあまり似ていませんね？ それでも、よく見れば仕組みがわかります。20 タイルずつの組が、画面上のタイルの 1 列になっています（黄色のブロックで示しています）。これらのタイルを縦に積み重ねると、{@fig:img-chr4-test}a の絵が浮かび上がってきます。

<div class="cblock">
  <table width=60% id="fig:img-chr4-test">
    <tr valign="top">
      <td>
	      <div class="cpt" style="width:80px;">
	        <img src="img/tte/chr4_test.png" alt="Output of chr4_test">
          <br>
	        <b>{*@fig:img-chr4-test}a</b>: chr4_test() の出力
	      </div>
      </td>
      <td>
	      <div class="cpt" style="width:289px;">
	        <img src="img/tte/chr4_test_tiles.png" alt="Output of chr4_test">
          <br>
	        <b>{*@fig:img-chr4-test}b</b>: chr4_test() のタイル。黄色のブロックは 1 列ぶんのタイルを示します。
	      </div>
      </td>
    </tr>
  </table>
</div>

### タイルへのテキストの描画 {#ssec-chr-drawg}

#### 版 1: 1 ピクセルずつ

グリフをタイルに描く一番簡単な方法は、[「ビットマップのテキスト」の節](#sec-bmp)のひな形に従うことです。それを次の関数で行っています。

```c {#cd-chr4-drawg-b1cts-a .proglist}
//! Simple version of chr4 renderer.
void chr4_drawg_b1cts_base(uint gid)
{
    TTE_BASE_VARS(tc, font);
    TTE_CHAR_VARS(font, gid, u8, srcD, srcL, charW, charH);
    uint x0= tc->cursorX, y0= tc->cursorY;
    uint srcP= font->cellH;

    u32 ink= tc->cattr[TTE_INK], raw;

    uint ix, iy, iw;
    for(iw=0; iw<charW; iw += 8)
    {
        for(iy=0; iy<charH; iy++)
        {
            raw= srcD[iy];
            for(ix=0; raw>0; raw>>=1, ix++)
                if(raw&1)
                    schr4c_plot(&tc->dst, x0+ix, y0+iy, ink);
        }
        srcD += srcP;
        x0 += 8;
    }
}
```

さて、`schr4c_plot()` での計算のやり直しのせいで、これはかなり遅く動くと思うかもしれません。そのとおりですが、実のところ、最初に思っていたほど悪くはありません。単にインライン化するだけでも速くできますが、本当の向上は、ピクセルを並列に描くことから来ます。

#### 版 2: 8 ピクセルを同時に

ピクセルを 1 つずつ描く代わりに、複数のピクセルを同時に描くこともできます。前に見た `bmp8_drawg_b1cts()` の描画処理はこれをしていました。2 ピクセルを展開して一緒に描いていたのです。4bpp のタイルの場合は、元のバイトを 1 つの（32 ビットの）ワードに展開して、_8_ ピクセルを同時に描けます。唯一の欠点は、おそらく 2 つのタイルに分けなければならないことです。

次の関数は、TTE のタイル用の主なグリフの描画処理で、これがなかなかの代物です。内側のループには描画の 2 つの段階があります。元のバイト `raw` のビットの展開と、用意したピクセル `px` を隣り合った 2 つのタイルに分けることです。これらはそれぞれ手順 3 と 4 に対応します。

普通、ビットの展開はループで行いますが、別の方法のほうが速いこともあります。詳しくは、私の[ビットの技](https://www.coranac.com/documents/bittrick/#sec-bup)についての文書を見てください。手順 3 の最初の 5 行が展開を行います。たとえば、2 進数の `0001 1011` を 16 進数の `0x00011011` に変換します。それから、これに 15 と `ink` を掛けて、それぞれピクセルのマスク `pxmask` と色の付いたピクセル `px` を得ます。

手順 4 では、必要ならピクセルの入ったワードを 2 つのタイルに分配します。手順 1 で、この処理のためのビットのオフセットとなる左右のシフトを用意しておきました。さて、大きなグリフでは、書き込み先のワードのいくつかが 2 回使われることになりますが、これは仕方がありません（実は仕方がなくはないのですが、その手順は見苦しく、おそらくその価値はありません）。代わりの方法として、書き込み先を 1 回だけ使い、元のデータを 2 回読む（そして展開/色付けする）こともできます。しかし、VRAM は ROM よりかなり速いので、それが得になるとは思えません。

<pre><code class="language-c hljs">//! Render 1bpp fonts to 4bpp tiles; col-major order.
void chr4c_drawg_b1cts(uint gid)
{
    // Base variables.
    TTE_BASE_VARS(tc, font);
    TTE_CHAR_VARS(font, gid, u8, srcD, srcL, charW, charH);
    uint x= tc-&gt;cursorX, y= tc-&gt;cursorY, dstP= tc-&gt;dst.pitch/4;
    uint srcP= font-&gt;cellH;

    // <span class="bold">(1)</span> Prepare dst pointers and shifts.
    u32 *dstD= (u32*)(tc-&gt;dst.data + (y + x/8*dstP)*4), *dstL;
    x %= 8;
    uint lsl= 4*x, lsr= 32-4*x, right= x+charW;

    // Inner loop vars.
    u32 px, pxmask, raw;
    u32 ink= tc-&gt;cattr[TTE_INK];
    const u32 mask= 0x01010101;

    uint iy, iw;
    for(iw=0; iw&lt;charW; iw += 8)    // Loop over strips
    {
        // <span class="bold">(2)</span> Update and increment main data pointers.
        srcL= srcD;     srcD += srcP;
        dstL= dstD;     dstD += dstP;

        for(iy=0; iy&lt;charH; iy++)   // Loop over scanlines
        {
            raw= *srcL++;
            if(raw)
            {
                // <span class="bold">(3)</span> Unpack 8 bits into 8 nybbles and create the mask
                raw |= raw&lt;&lt;12;
                raw |= raw&lt;&lt;6;
                px   = raw &amp; mask&lt;&lt;1;
                raw &amp;= mask;
                px   = raw | px&lt;&lt;3;

                pxmask= px*15;
                px   *= ink;

                // <span class="bold">(4a)</span> Write left tile:
                dstL[0] = (dstL[0] &amp;~ (pxmask&lt;&lt;lsl) ) | (px&lt;&lt;lsl);

                // <span class="bold">(4b)</span> Write right tile (if any)
                if(right &gt; 8)
                    dstL[dstP]= (dstL[dstP] &amp;~ (pxmask&gt;&gt;lsr) ) | (px&gt;&gt;lsr);
            }
            dstL++;
        }
    }
}
</code></pre>

`chr4c_drawg_b1cts()` はかなり速いです。前の版より約 33% 速いのは確かです。実は bmp8 の描画処理より速いのですが、それはほんのわずかな差です。

もちろん、いつでももう一段上を目指せます。いろいろなシフトや条件があるので、これは Thumb より ARM のコードにうってつけです。そして、確実に計画どおりに動くように、アセンブリで書きます。

#### 版 3: ARM アセンブリ

次の関数は `chr4_drawg_b1cts_fast()` で、版 2 の ARM アセンブリ版です。C とアセンブリのループはほぼ 1 対 1 で対応しているので、説明は C の版を見てください。

速さの点では、アセンブリ版は C の版より_ずっと_優れています。ARM のコードにとって_とても_不利な ROM の中でさえ、Thumb の版より速いのです。これをさらに速くできる細かい点が 1 つか 2 つありますが、おおむね、任意の寸法のフォントについてはこれで決まりのはずです。もちろん、フォントの大きさが固定で、色の付け直しや透明が必要ないなら、事情は少し違ってきます。

```armasm {#cd-chr4c-drawg-b1cts-fast .proglist}
// Include TTC/TFont member offsets plz.
#include "tte_types.s"

/*
IWRAM_CODE void chr4c_drawg_b1cts_fast(int gid);
*/
    .section .iwram, "ax", %progbits
    .arm
    .align
    .global chr4c_drawg_b1cts_fast
chr4c_drawg_b1cts_fast:
    stmfd   sp!, {r4-r11, lr}

    ldr     r5,=gp_tte_context
    ldr     r5, [r5]

    @ Preload dstBase (r4), dstPitch (ip), yx (r6), font (r7)
    ldmia   r5, {r4, ip}
    add     r3, r5, #TTC_cursorX
    ldmia   r3, {r6, r7}

    @ Get srcD (r1), width (r11), charH (r2)
    ldmia   r7, {r1, r3}            @ Load data, widths
    cmp     r3, #0
    ldrneb  r11, [r3, r0]           @ Var charW
    ldreqb  r11, [r7, #TF_charW]    @ Fixed charW
    ldrh    r3, [r7, #TF_cellS]
    mla     r1, r3, r0, r1          @ srcL
    ldrb    r2, [r7, #TF_charH]     @ charH
    ldrb    r10, [r7, #TF_cellH]    @ cellH

    @ Positional issues: dstD(r0), lsl(r8), lsr(r9), right(lr), cursorX
    mov     r3, r6, lsr #16         @ y
    bic     r6, r6, r3, lsl #16     @ x

    add     r0, r4, r3, lsl #2      @ dstD= dstBase + y*4
    mov     r3, r6, lsr #3
    mla     r0, ip, r3, r0

    and     r6, r6, #7              @ x%7
    add     lr, r11, r6             @ right= width + x%8
    mov     r8, r6, lsl #2          @ lsl = x%8*4
    rsb     r9, r8, #32             @ lsr = 32-x%8*4

    ldr     r6,=0x01010101
    ldrh    r7, [r5, #TTC_ink]

    @ --- Reg-list for strip/render loop ---
    @ r0    dstL
    @ r1    srcL
    @ r2    scanline looper
    @ r3    raw
    @ r4    px / tmp
    @ r5    pxmask
    @ r6    bitmask
    @ r7    ink
    @ r8    left shift
    @ r9    right shift
    @ r10   dstD
    @ r11   charW
    @ ip    dstP
    @ lr    split indicator (right edge)
    @ sp00  charH
    @ sp04  deltaS = cellH-charH     (delta srcL)

    cmp     r11, #8
    @ Prep for single-strip render
    suble   sp, sp, #8
    ble     .Lyloop
    @ Prep for multi-strip render
    sub     r3, r10, r2
    mov     r10, r0
    stmfd   sp!, {r2, r3}           @ Store charH, deltaS
    b       .Lyloop

    @ --- Strip loop ---
.Lsloop:
        @ (2) Update and increment main data pointers.
        ldmia   sp, {r2, r3}        @ Reload charH and deltaS
        add     r10, r10, ip        @ (Re)set dstD/dstL
        mov     r0, r10
        add     r1, r1, r3
        sub     lr, lr, #8

        @ --- Render loop ---
.Lyloop:
            @ (3) Prep px and pxmask
            ldrb    r3, [r1], #1
            orrs    r3, r3, r3, lsl #12
            beq     .Lnopx              @ Skip if no pixels
            orr     r3, r3, r3, lsl #6
            and     r4, r3, r6, lsl #1
            and     r3, r3, r6
            orr     r3, r3, r4, lsl #3

            rsb     r5, r3, r3, lsl #4
            mul     r4, r3, r7

            @ (4a) Render to left tile
            ldr     r3, [r0]
            bic     r3, r3, r5, lsl r8
            orr     r3, r3, r4, lsl r8
            str     r3, [r0]

            @ (4b) Render to right tile
            cmp     lr, #8
            ldrgt   r3, [r0, ip]
            bicgt   r3, r3, r5, lsr r9
            orrgt   r3, r3, r4, lsr r9
            strgt   r3, [r0, ip]
.Lnopx:
            add     r0, r0, #4
            subs    r2, r2, #1
            bne     .Lyloop

        @ Test for strip loop
        subs    r11, r11, #8
        bgt     .Lsloop

    add     sp, sp, #8
    ldmfd   sp!, {r4-r11, lr}
    bx      lr

@ EOF
```

### 多色のフォントと影付きのフォント {#ssec-chr4-drawg-b4}

ビットパックしたフォントでは、モノクロのグリフになります。影やアンチエイリアスのためにもっと色が欲しければ、もっと多くのビットを使う必要があります。そのコードは 1bpp でビットパックした版とほとんど同じです。最も重要な違いは、元のデータの型が違うことと、正しいマスクを求める方法が違うことです。ああ、それからもちろん、もうビットを展開する必要はありません。

次のコードは、4 ビットのピクセルのワードから透明のマスクを作る方法を示しています。基本的には、4 ビットのすべてのビットをまとめて、その 4 ビットのほかのビットをマスクで消します。これで、4 ビット全体が空なら 0、そうでなければ 1 になります。これにまた 15 を掛ければ、正しいマスクになります。

```c {.proglist}
// Create pixel mask from 8x 4 bits
u32 *srcL= ...;         // Source is now 32bit.

raw     = *srcL++;      // Source word: 8x 4 bits
pxmask  = raw;
pxmask |= pxmask>>2;    // bit0 = bit0 | bit2
pxmask |= pxmask>>1;    // bit0 = bit0 | bit1 | bit2 | bit3;
pxmask &= 0x11111111;   // bit0 is 0 only if bits 0-3 were all 0
pxmask *= 15;
```

<div class="cpt_fr" style="width:128px;">
  <img src="img/tte/verdana9_b4.png" id="fig:img-verdana9-b4" width="128" alt="">
  <br>
  <b>{*@fig:img-verdana9-b4}</b>: 影付きの Verdana 9。
</div>

#### 影付きの文字

いいえ、怪し*げ*な人物（shady characters）ではありません。影*付き*の文字（shaded characters）です。ゲームでよく見かけるのは、テキストに縁取りがあるか、片側に少し影があるものです。1bpp のフォントで影を作ることもできますが、単にフォントそのものに組み込むほうが簡単です（{@fig:img-verdana9-b4} を見てください）。そうすると 1bpp で扱える以上の色が必要になるので、ここでは 2bpp のフォントを使いたくなるかもしれません。しかし、本当にメモリが_ひどく_足りないのでなければ、ここでも 4bpp を使うほうが便利です。

そこからは、先ほど説明した手順に従えます。しかし、影を作っているビットを巧みに使えば、影の色も可変にできます。たとえば、ビット 0 を「インク」のビット、ビット 1 を「影」のビット、必要ならビット 2 を「紙」のビットと決めることができます。そうすると、`raw&0x11111111` で「インク」のマスクが、`(raw>>1)&0x11111111` で「影」のマスクが得られ、これらを使って色を付けたり、完全なマスクを作ったりできます。以下は、これをどうやってできるかの実演です。ここの各行はちょうど 1 つの ARM の命令に対応するので、効率の良い方法のはずです。まあ、いずれにせよ ARM のコードではです。

```c {.proglist}
// Use bits 0 and 1 from each nybble to create masks and apply colors.
u32 *srcL= ...;

raw     = *srcL++;              // Source word: 8x 4 bits
px      = raw    & 0x11111111;  // Bit 0 for ink pixels
raw     = raw>>1 & 0x11111111;  // Bit 1 for shadow pixels
pxmask  = px | raw;             // Mask of ink and shadow bits
pxmask *= 15;

px      = px * ink;             // Color with ink
px     += raw* shadow;          // Add shadow pixels
```

`chr4c_drawg_b4cts()` の描画処理は、この方法を使ってインクと影の両方のピクセルに色を付けます。太字の部分とビットの展開がなくなっていることを除けば、基本的に `chr4c_drawg_b4cts()` です（訳注: 比較の対象は 1bpp 版の描画処理と思われます）。ここでの「ピクセルなし」の条件にも注意してください。`pxmask` がゼロなら、することは何もないので、何もしません。

<pre><code class="language-c hljs">//! 4bpp font, tilestrips with ink/shadow coloring.
void chr4c_drawg_b4cts(uint gid)
{
    TTE_BASE_VARS(tc, font);
    TTE_CHAR_VARS(font, gid, u32, srcD, srcL, charW, charH);
    uint x= tc-&gt;cursorX, y= tc-&gt;cursorY;
    uint srcP= font-&gt;cellH, dstP= tc-&gt;dst.pitch/4;

    // <span class="bold">(1)</span> Prepare dst pointers and shifts.
    <span class="bold">u32 *dstD= (u32*)(tc-&gt;dst.data + (y+x/8*dstP)*4), *dstL</span>;
    x %= 8;
    uint lsl= 4*x, lsr= 32-4*x, right= x+charW;

    // Inner loop vars
    u32 amask= 0x11111111;
    u32 px, pxmask, raw;
    u32 ink=   tc-&gt;cattr[TTE_INK];
    <span class="bold">u32 shade= tc-&gt;cattr[TTE_SHADOW]</span>;

    uint iy, iw;
    for(iw=0; iw&lt;charW; iw += 8)    // Loop over strips
    {
        srcL= srcD;     srcD += srcP;
        dstL= dstD;     dstD += dstP;

        for(iy=0; iy&lt;charH; iy++)   // Loop over scanlines
        {
            raw= *srcL++;

            // <span class="bold">(3a)</span> Prepare pixel mask
            px    = (raw    &amp; amask);
            raw   = (raw&gt;&gt;1 &amp; amask);
            pxmask= px | raw;
            if(pxmask)
            {
                px *= ink;          // <span class="bold">(3b)</span> Color ink pixels
                px += raw*shade;    // <span class="bold">(3c)</span> Color shadow pixels
                pxmask *= 15;       // <span class="bold">(3d)</span> Create mask

                // <span class="bold">(4a)</span> Write left tile:
                dstL[0] = (dstL[0] &amp;~ (pxmask&lt;&lt;lsl) ) | (px&lt;&lt;lsl);

                // <span class="bold">(4b)</span> Write right tile (if any)
                if(right &gt; 8)
                    dstL[dstP]= (dstL[dstP] &amp;~ (pxmask&gt;&gt;lsr) ) | (px&gt;&gt;lsr);
            }
            dstL++;
        }
    }
}
</code></pre>

### タイルへの描画を速くするコツ {#ssec-chr4-tips}

これらのタイルの描画処理についてはかなりプロファイリングをしたので、どの技法が効率的で、どれがそうでないかについてはそれなりにわかっているつもりです。以下は私の観察の一部です。

- **プロファイリングする**。凝ったルーチンをひねり出す前に、元の単純な版が最適化に値すること、_そして_賢いルーチンが本当に速いことを確かめてください。
- **透明に描く**。さて、これは遅くなるだろうと思うかもしれませんが、そうとは限りません。透明なテキストの特徴は、前景のピクセルが背景のピクセルよりずっと少ないことで、そのため描画するピクセルの数も少なくなります。
- **バッファを使わない**。最初の試作では、展開/色付けと VRAM への挿入の段階が分かれていました。用意したピクセルを IWRAM のバッファに入れ、それから VRAM にコピーしていたのです。記憶が正しければ、ループを 1 つにしてバッファを捨てたことで 30% 節約できました。
- **並列化する**。正しいデータにたどり着くまでの道のりは長いです。その道をあまり何度も通らずに済めば助かります。とはいえ、空のピクセルが多いなら、8 ピクセルを同時に描くのは努力の無駄かもしれません。これはフォントによります。
- **ARM のコードは最強**。これらのルーチンには、シフト、マスク、量がたくさんあります。そのため、Thumb より ARM のコードに特に向いています。実際、16 ビットのバスの ROM の中でさえ、ARM 版は Thumb でコンパイルしたものに勝ちます。とはいえ……
- **ARM では GCC に定数のマスクを使わせない**。（0x11111111 のような）リテラルとの AND に関して、ARM の最適化には残念なバグがあります。単純な `ldr`+`and` の組を出す代わりに、賢く立ち回って、マスクを複数のバイトの大きさのマスクに分けることで読み込みを避けようとするのです。その結果、内側のループで 1 命令で済むはずが 4 命令になります。余分にどれだけのレジスタを使うかによっては、さらに増えるかもしれません。これは定数の場合_だけ_、しかも ARM でコンパイルしたコードの場合だけに起きることに注意してください。回避策は、マスクをグローバル変数に入れておき、ループの前に読み込むことです。いくつかのルーチンを手でアセンブルしたのは、一つにはこのためです。
- **できるなら特殊な場合のためにコードを書く**。フォントが 1 つしかなく、色付けなどが必要ないなら、その場合だけのためにコードを書けば、大幅に時間を節約できる可能性があります。読み込み元と書き込み先の寸法に、メモリにあるものではなく定数を使うのも少し役立ちます。
- **列優先のアクセスを使う**。上で示したルーチンは、タイルのある行から別の行に移るための余分なコードを必要とします。タイルを列優先の配置で使えば、それは必要なくなります。

このリストには、お決まりの但し書きを付けておいてください。これらの技法は私の場合にはうまくいきましたが、すべての場合に当てはまるわけではありません。たとえば、ほかのシステム（\*ゴホン\* NDS）は CPU のアーキテクチャやメモリの特性が違い、それが速さに影響するでしょう。

### 会話ウィンドウの上の色付きのテキスト {#ssec-chr4-demo}

<div class="cpt_fr" style="width:240px;">
  <img src="img/tte/test_tte_chr4.png" id="fig:img-test-chr4" alt="Text on tiles.">
  <br>
  <b>{*@fig:img-test-chr4}</b>: タイルの上のテキスト。
</div>

{@fig:img-test-chr4} に描かれている状況はおなじみのはずです。要点は、背景のマップと、テキストの入った会話ボックスがあることです。このテキストは静的ですが、左上の隅の位置は、マップに沿ってスクロールするにつれて絶えず更新されます。

このデモの中心となる関数は `test_tte_chr4()` です。最初に、`tte_init_chr4c()` を呼んで、テキストシステムを chr4c モード用に初期化します。3 つ目の引数はマップのエントリのオフセットで、`0xF000` はサブパレット 15 を使うという意味です。4 つ目は色の属性のためのワードで、インクに 13、影に 15、ほかには 0 です。このデモでは、verdana9 の 4bpp 版（{@fig:img-verdana9-b4} を見てください）と、グリフの描画に速いアセンブリ版を使っています。

手順 2 では、背景のマップと会話ボックスを読み込みます。会話ボックスは、テキストを描画するタイルにコピーしていることに注意してください。テキストを表示すると、グリフが本当に透明に描画されていることがわかります。これはつまり、標準の消去処理はボックスまで消してしまうので、ほぼ使えないということでもあります。

会話のテキストは手順 3 で描きます。`ci` タグと `cs` タグは、それぞれインクと影の色の属性を設定します。これで、文字列 “arrows” は色 1 と 2（正確には 0xF1 と 0xF2）を使う、という具合になります。

<pre><code class="language-c hljs">//! Set up a rectangle for text, with the non-text layers darkened for contrast.
void win_textbox(uint bgnr, int left, int top, int right, int bottom, uint bldy)
{
    REG_WIN0H= left&lt;&lt;8 | right;
    REG_WIN0V=  top&lt;&lt;8 | bottom;
    REG_WIN0CNT= WIN_ALL | WIN_BLD;
    REG_WINOUTCNT= WIN_ALL;
    
    REG_BLDCNT= (BLD_ALL&amp;~BIT(bgnr)) | BLD_BLACK;
    REG_BLDY= bldy;

    REG_DISPCNT |= DCNT_WIN0;

    tte_set_margins(left, top, right, bottom);
}

//! Test chr4 shaded text renderer
void test_tte_chr4()
{
    irq_init(NULL);
    irq_add(II_VBLANK, NULL);
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0 | DCNT_BG2;

    // <span class="bold">(1)</span> Init for text
    tte_init_chr4c(
        0,                              // BG number.
        BG_CBB(0)|BG_SBB(10),           // BG control.
        <span class="bold">0xF000,                         // Screen-entry base
        bytes2word(13,15,0,0),          // Color attributes.</span>
        CLR_BLACK,                      // Ink color
        &amp;verdana9_b4Font,               // Verdana 9, with shade.
        (fnDrawg)chr4c_drawg_b4cts_fast);    // b4cts renderer, asm version
    tte_init_con();                     // Initialize console I/O

    // <span class="bold">(2)</span> Load graphics
    LZ77UnCompVram(dungeon01Map, se_mem[12]);
    LZ77UnCompVram(dungeon01Tiles, tile_mem[2]);
    LZ77UnCompVram(dungeon01Pal, pal_bg_mem);

    GRIT_CPY(&amp;tile_mem[0][16*30], dlgboxTiles);
    GRIT_CPY(pal_bg_bank[15], dlgboxPal);

    // <span class="bold">(3)</span> Create and print to a text box.
    win_textbox(0, 8, 160-32+4, 232, 160-4, 8);
    CSTR text=
        "#{P}Scroll with #{ci:1;cs:2}arrows#{ci:13;cs:15}, "
        "quit with #{ci:1;cs:2}start#{ci:13;cs:15}\n"
        "Box opacity with #{ci:3;cs:4}L/R#{ci:7;cs:9}";
    tte_write(text);

    // Reset margins for coord-printing
    tte_set_margins(8, 8, 232, 20);

    int x=128, y= 32, ey=8&lt;&lt;3;

    REG_BG2HOFS= x;
    REG_BG2VOFS= y;

    // Invisible map buildup!
    REG_BG2CNT= BG_CBB(2) | BG_SBB(12) | BG_REG_64x64;
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0 | DCNT_BG2 | DCNT_WIN0;

    while(1)
    {
        VBlankIntrWait();
        key_poll();
        
        // <span class="bold">(4)</span> Scroll and blend
        x = clamp(x + key_tri_horz(), 0, 512+1-SCREEN_WIDTH);
        y = clamp(y + key_tri_vert(), 0, 512+1-SCREEN_HEIGHT);
        ey= clamp(ey+ key_tri_shoulder(), 0, 0x81);

        REG_BG2HOFS= x;
        REG_BG2VOFS= y;
        REG_BLDY= ey&gt;&gt;3;

        // (5) Erase and print new position.
        tte_printf("#{es;P}%d, %d", x, y);
        
        if(key_hit(KEY_START))
            break;      
    }   
}
</code></pre>

この節の締めくくりに、テキストボックスについて少し説明します。よく見ると、半透明になっているのがわかるでしょう。正確に言うと、テキストとボックス自体は普通の明るさですが、それが覆っている背景が普段より暗くなっています。この素敵でちょっとした効果には、2 つのことが必要です。

- 内側と外側のウィンドウを定義しなければなりません。どちらのウィンドウもすべてのレイヤーを含むようにしますが、内側のウィンドウはブレンドを使う（WIN_BLD）ように設定しなければなりません。これで内側でだけブレンドが有効になります。
- ブレンドのモードは、テキストボックスのある背景を除くすべてのレイヤーについて、黒へのフェード（BLD_BLACK）に設定するべきです。

`win_textbox()` はそのためのものです。この関数は、テキストがボックスの中できれいに折り返されるようにマージンも設定します。

## スクリプト、コンソール入出力、そのほかの便利なもの {#sec-misc}

### TTE の書式のコマンド {#ssec-misc-tags}

TTE のコンテキストには、位置、色、フォント、そのほかいくつかのものを制御するメンバがあります。これらのパラメータを変える方法は 2 つあります。1 つ目は、メンバへの直接のアクセスや `tte_set_ink()` のような関数で、状態の変更をハードコードすることです。これは素直で速いのですが、あまり柔軟ではありません。2 つ目は、文字列そのものの中で<dfn>書式のタグ</dfn>を使うことです。システムは文字列からこれらのタグを解析し、それに応じて解釈します。これは基本的に一種のスクリプトです。

TTE が使うタグは次のようなものです。

```
#{`_`tag0`_`:`_`args`_`; `_`tag1`_`:`_`args`_`}
```

コード自体は \``#{`' で始まり \``}`' で終わります。各コマンドはタグで構成され、必要ならそのあとにコロンとカンマ区切りの引数が続きます。複数のコマンドはセミコロンで区切れます。たとえば、\``#{es; P:10,16}`' は画面をクリアして、カーソルを (10, 16) に設定します。

さて、これの解析の仕方をお見せすることもできますが、今使っているパーサは、まあ、長くてとても見苦しいとだけ言っておきましょう。基本的には、次のようなものが入った巨大な switch ブロック（ときには_二重の_ switch ブロック）です。

<pre><code class="language-c hljs">char *tte_cmd_default(const char *str)
{
    int ch, val;
    char *curr= (char*)str, *next;

    TTC *tc= tte_get_context();

    while(1)
    {
        ch= *curr;
        next= curr+1;

        // <span class="bold">(1)</span> Check first character
        switch(ch)
        {
        // <span class="bold">(2)</span> --- Absolute Positions ---
        case 'X':
            tc-&gt;cursorX= curr[1]==':'           // If there's an argument ...
                ? strtol(curr+2, &amp;next, 0)     // set cursor X to arg
                : tc-&gt;marginLeft;               // else move to start of line.
            break;

        // ... more cases ...

        // <span class="bold">(3)</span> Find EOS/EOC/token and act on it
        curr= tte_cmd_next(next);

        if(curr[0] == '\0')
            return curr;
        else if(curr[0] == '}')
            return curr+1;
    }
}
</code></pre>

言ったとおり見苦しいですが、今のところはこれで我慢してもらうしかありません。入ってくるポインタは '`#{`' の次の最初の文字を指しています。コマンドのタグはすべて 1 文字か 2 文字で、switch は認識できる文字を探して、それに応じて処理します。

タグの 1 つは '`X`' で、これはカーソルの絶対的な X 座標を設定します。`tc->cursorX` は、引数があればそれに、なければ左のマージンに設定されます。ここで [`strtol()`](https://en.cppreference.com/w/c/string/byte/strtol) を使っていることに注目してください。これはとても面白い関数です。10 進数と 16 進数の両方の文字列に使えるだけでなく、2 つ目の引数を通して、文字列の中の数値の直後へのポインタを得られます。代わりに `sscanf()` や `atoi()` も使えますが、`strtol()` のほうが便利です。

タグを処理したら、さらにタグを探すか、終わりの区切り文字か文字列の終わりが見つかれば抜けます。

{\*@tbl:tte-cmd} に、使えるタグを示します。大文字と小文字を区別すること、そしてパラメータの数によっては 2 つ以上のことをする項目もあることに注意してください。

<div class="cblock">
  <table id="tbl:tte-cmd" class="table-data" width= 70%>
    <caption align="bottom">
      <b>{*@tbl:tte-cmd}</b>: 使える TTE の書式のタグ。
    </caption>
    <tr>
      <th>コード</th>
      <th>説明</th>
    </tr>
    <tr>
      <td>P </td>
      <td>位置を左上のマージンに戻す。 </td>
    </tr>
    <tr>
      <td>Pr </td>
      <td>カーソルの位置を復元する（<code>Ps</code> も参照）。 </td>
    </tr>
    <tr>
      <td>Ps </td>
      <td>カーソルの位置を保存する。 </td>
    </tr>
    <tr>
      <td>P: <i>x</i>,<i>y</i> </td>
      <td>カーソルを座標 (<i>x</i>,&nbsp;<i>y</i>) に設定する。 </td>
    </tr>
    <tr>
      <td>X </td>
      <td><code>cursorX</code> を左のマージンに戻す。</td>
    </tr>
    <tr>
      <td>X: <i>x</i> </td>
      <td><code>cursorX</code> を <i>x</i> に設定する。 </td>
    </tr>
    <tr>
      <td>Y </td>
      <td><code>cursorY</code> を上のマージンに戻す。 </td>
    </tr>
    <tr>
      <td>Y: <i>y</i> </td>
      <td><code>cursorY</code> を <i>y</i> に設定する。 </td>
    </tr>
    <tr>
      <td>c[ispx]:&nbsp;<i>cattr</i> </td>
      <td>
        インク（<code>ci</code>）、影（<code>cs</code>）、紙（<code>cp</code>）、特殊（<code>cx</code>）の色の属性を <i>cattr</i> に設定する。
      </td>
    </tr>
    <tr>
      <td>e[slbf] </td>
      <td>
        マージンの間の画面（<code>es</code>）、現在の行（<code>el</code>）、現在の行のカーソルまで（<code>eb</code>。後ろ向き）、現在の行のカーソルから（<code>ef</code>。前向き）を消す。
      </td>
    </tr>
    <tr>
      <td>er: <i>l</i>,<i>t</i>,<i>r</i>,<i>b</i> </td>
      <td>(<i>l</i>,<i>t</i>) から (<i>r</i>,<i>b</i>) までの長方形を消す。 </td>
    </tr>
    <tr>
      <td>f: <i>idx</i> </td>
      <td><code>font</code> を <code>TTC.fontTable[<i>idx</i>]</code> に設定する。 </td>
    </tr>
    <tr>
      <td>m[ltrb]:&nbsp;<i>value</i> </td>
      <td>
          左（<code>ml</code>）、上（<code>mt</code>）、右（<code>mr</code>）、下（<code>mb</code>）のマージンを <i>value</i> に設定する。 
      </td>
    </tr>
    <tr>
      <td>m: <i>l</i>,<i>t</i>,<i>r</i>,<i>b</i> </td>
      <td>マージンを長方形 (<i>l</i>,<i>t</i>) - (<i>r</i>,<i>b</i>) に設定する </td>
    </tr>
    <tr>
      <td>p: <i>dx</i>, <i>dy</i> </td>
      <td>カーソルを (<i>dx</i>, <i>dy</i>) だけ動かす。 </td>
    </tr>
    <tr>
      <td>s: <i>idx</i> </td>
      <td><code>TTC.stringTable</code> の <i>idx</i> 番目の文字列を表示する。 </td>
    </tr>
    <tr>
      <td>w: <i>count</i> </td>
      <td><i>count</i> フレーム待つ。 </td>
    </tr>
    <tr>
      <td>x: <i>dx</i> </td>
      <td>カーソルを <i>dx</i> だけ右に動かす。 </td>
    </tr>
    <tr>
      <td>y: <i>dy</i> </td>
      <td>カーソルを <i>dy</i> だけ下に動かす。 </td>
    </tr>
  </table>
</div>

今のところ、コマンドはまだ壊れやすいので、気をつけて使ってください。たとえば、位置のコマンドは単にカーソルを動かすだけで、マージンで切り詰めはしません。フォントと文字列のコマンド（それぞれ `f` と `s`）にも注意してください。`tte_cmd_default()` はインデックスが配列の範囲外かどうかを調べないので、……おかしなことになるかもしれません。いつかこれらを直したいと思っていますが、今は優先度が高くありません。私が使えるもっと頑丈なものを持っている人がいれば、ぜひ声をかけてください。

:::warning TTE の書式のコマンド: 買い手は用心せよ

今の TTE のコマンドは、まだ誰が使っても安全というわけではありません。まっとうな使い方をしていれば、かなりうまく動くはずです。でも、注意しないと、自分の足を撃ってしまうのはまだ簡単です。

:::

### コンソール入出力を使う {#ssec-misc-conio}

`tte_write()` のようなものは純粋な文字列には便利ですが、本当に助かるのは `printf()` のようなものがあることです。昔（2006 年より前）は、`printf()`、`putc` などのコンソール出力の関数は使えませんでしたが、Wintermute が devkitArm の標準 C ライブラリに、ゲーム機でもそれを使えるようにする仕組みを加えました。

その鍵は、`sys/iosupport.h` で定義されている `devoptab_t` 構造体です。これには、デバイスの操作への関数ポインタの表が入っています。ここで興味があるポインタは `write_r` です。これは `printf()` などが最終的な出力のために呼ぶ関数です。

```c {.proglist}
// Partial devoptab_t definition
typedef struct {
    const char *name;
    int structSize;
    int (*open_r)(struct _reent *r, void *fileStruct, const char *path,
        int flags,int mode);
    int (*close_r)(struct _reent *r,int fd);
    int (*write_r)(struct _reent *r,int fd,const char *ptr,int len);
    ...
} devoptab_t;
```

GBA で標準のコンソールのルーチンを動かす鍵は、コンソール出力のためのデフォルトの `write_r` を、自分で作ったものにすり替えることです。これがどう動くかを説明する前に、これは黒魔術にかなり近いものだということを理解しておいてほしいと思います。ライブラリの根っこまで降りていくことになり、これがどう動くかについての文書はほとんどありません。完全な説明に一番近いものとして見つけられたのはこの記事 [Embedding GNU: Newlib, Part 2](https://web.archive.org/web/20100209210107/https://www.embedded.com/story/OEG20020103S0073) ですが、これも説明が多いとは言えません。

別の言い方をすると、あなたは洞窟の中にいます。真っ暗で、あたりには [グルー](<https://en.wikipedia.org/wiki/Grue_(monster)>) がいます。

それが済んだところで、続けましょう。最初の段階は、置き換えの書き込み処理を作ることです。TTE の場合、これは `tte_con_write()` です。`tte_write()` とほとんど同じですが、`devoptab_t.write_r` で決められた形式に合わせなければなりません。結局、次のようになります。

<pre><code class="language-c hljs">//! internal output routine used by printf.
/*! \param r    Reentrancy parameter.
    \param fd   File handle (?).
    \param text Text buffer containing the string prepared by printf.
    \param len  Length of string.
    \return     Number of output bytes (?)
    \note       \a text is NOT zero-terminated!!!!one!
*/
int tte_con_write(struct _reent *r, int fd, const char *text, int len)
{
    // <span class="bold">(1)</span> Safety checks
    if(!sConInitialized || !text || len&lt;=0)
        return -1;

    int ch, gid, charW;
    const char *str= text, *end= text+len;

    // <span class="bold">(2)</span> check for end of text
    while( (ch= *str) != 0 &amp;&amp; str &lt; end)
    {
        str++;
        switch(ch)
        {

        // <span class="bold">(3)</span> --- VT100 sequence ( ESC[foo; ) ---
        case 0x1B:
            if(str[0] == '[')
                str += tte_cmd_vt100(str);
            break;

        //# <span class="bold">(4)</span> Other character cases. See tte_write()
        }
    }

    return str - text;
}

</code></pre>

ここでは引数の説明を付けましたが、ほとんど推測に基づいています。`r` パラメータには[再入可能性](https://en.wikipedia.org/wiki/Reentrancy_(computing))の情報が入っていて、複数のスレッドがある場合に役立ちます。GBA はシングルスレッドのシステムなので、これは気にしなくてよいはずです。`fd` は何らかのファイルハンドルだと思いますが、ファイルに書き込むわけではないので、これも気にしなくてかまいません。

本当に興味がある引数は `text` と `len` です。`text` 引数は、描画する文字列の入ったバッファを指します。`printf()` の場合、それは書式化の_あと_の文字列で、`%d` などのコードはすべて処理済みです。そしてここが最も重要な部分です。`text` はヌル終端され**ていません**。だから長さの変数もあるのです。

私にわかる限り、`printf` はスタック上の大きなバッファ（約 1300 バイト）を使い、そこに書式化した数値を書き込みます。このバッファは、再び呼び出しても消去されず、書き込み処理に送られるときに「\\0」で終端されることもありません。これには次のような影響があります。

- 1300 バイトは IWRAM のかなりの部分です。そのための十分な領域があることを確かめてください。このルーチンは遅く、呼び出しが入れ子になってすべてを破壊し始めかねないので、割り込みから `printf()` を呼ば_ないで_ください。
- `len` パラメータを忘れないでください。バッファはゼロにされないので、古いデータの残りがまだあるかもしれず、ゴミが出てしまいます。
- ここには、書式のコマンドの解析に関するさらなる危険もありえます。文字列がバッファの長さを超えると、より小さな塊に分けられるのだと思います。分割がコマンドの途中で起きたらどうなるかはわかりませんが、良いことにはならないでしょう。もちろん、どのみち画面には収まらないので、そんなに長い文字列を使うべきではありません。

それを除けば、`tte_con_write()` は素直なものです。先ほど言ったように、ループの中身は `tte_write()` のものとほとんど同じです。唯一の本当の違いはポイント 3 です。これは VT100 の書式の文字列の確認で、次の小節で扱います。

新しい書き込み処理を使うには、何らかの方法でそれをデバイスの一覧につなぎ込む必要があります。まず、正しい場所に書き込み処理を入れた `devoptab_t` のインスタンスを作ります。`devoptab_list` というデバイスの操作の一覧があります。興味のあるデバイスはストリーム `stdout` と `stderr` で、これらは一覧のエントリ `STD_OUT` と `STD_ERR` です。これらのエントリを自分の構造体に向けるだけです。

2 つ目は、これらのストリームのバッファを設定することです。これが本当に必要かどうかはわかりませんが、libgba ではそうしていて、その作者はこの仕組みを一番よく知っているので、ここで異を唱えるつもりはありません。そのための関数は `setvbuf()` です。必要な初期化の手順を下に示します。

```c {#cd-tte-init-con .proglist}

static int sConInitialized= 0;

const devoptab_t tte_dotab_stdout=
{
    "ttecon",
    0,
    NULL,
    NULL,
    tte_con_write,
    NULL,
    NULL,
    NULL
};

//! Init stdio capabilities for TTE.
void tte_init_con()
{
    // attach our operations to stdout and stderr.
    devoptab_list[STD_OUT] = &tte_dotab_stdout;
    devoptab_list[STD_ERR] = &tte_dotab_stdout;

    // Set buffers.
    setvbuf(stderr, NULL , _IONBF, 0);
    setvbuf(stdout, NULL , _IONBF, 0);

    sConInitialized = 1;
}
```

`tte_init_con()` を呼ぶと stdio の機能が有効になり、`printf()` などが使えるようになります。素の `printf()` はかなり重く、浮動小数点のオプションもありますが、それは GBA の環境で使われることはまずありません。そのため、普通は整数専用の親戚 `iprintf()` を使うことになります。また、TTE の実装は libgba のものとは**違う**ので、2 つを混同しないでください。そのため、`iprintf()` という名前は `tte_printf` マクロの後ろに隠しています。

以下は、その使い方の短い例です。ここでは `tte_printf()` を使っていますが、`printf()` や `iprintf()` でも同じように動いたでしょう。

```c {#cd-hello .proglist}
#include <stdio.h>
#include <tonc.h>

int main()
{
    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    // Init BG 0 for text on screen entries.
    tte_init_se_default(0, BG_CBB(0)|BG_SBB(31));

    // Enable TTE's console functionality
    tte_init_con();

    tte_printf("#{P:72,64}");        // Goto (72, 64).
    tte_printf("Hello World!");      // Print "Hello world!"

    while(1);

    return 0;
}
```

:::warning printf のお荷物

`printf()` は素晴らしいものですが、欠点もあります。まず、とても重い関数で、かなりの数の関数を呼び出し、それらをすべてリンクしなければなりません。次に、ものすごく遅いです。とても多くのことができるので、そのいろいろな場合をすべて確認しなければならないのです。また、文字列から 10 進数への変換に除算を使いますが、それは GBA にとって本当にまずいことです。

`printf()` がどれだけ高くつくかを意識してください。それがボトルネックになるとわかったら、自分で余計なものを削った版を作ってみてください。まともな `sprintf()` の代わりとしては、[Dan Posluns](https://www.danposluns.com/gbadev/) が作った `posprintf()` があります。

:::

### VT100 のエスケープシーケンス {#ssec-misc-vt100}

C の本はどれも、コンソールの画面に文字を出せると教えてくれます。たいてい教えてくれないのは、環境によっては書式も制御できるということです。そうした環境の 1 つが [VT100](https://en.wikipedia.org/wiki/VT100) で、書式を示すのに<dfn>エスケープシーケンス</dfn>を使っていました。devkitPro がいろいろなシステム向けに配布しているライブラリはこれらのシーケンスを使っているので、これらにも対応しておくのは良い考えです。

コードの一般的な形式は次のとおりです。

```
CSI n1;n2 ... letter
```

ここで _CSI_ は<dfn>コマンドシーケンス指示子</dfn>（command sequence indicator）の ASCII コードで、この場合はエスケープ文字（27、0x1B、または 033）のあとに '\[' が続いたものです。最後の文字が書式のコードの種類を表し、_n1_、_n2_ …… が書式のパラメータです。Wikipedia の[ここ](https://en.wikipedia.org/wiki/ANSI_escape_code)に標準のセットの良い概要があり、こちらにももう 1 つあります: [VT100 commands and control sequences](https://web.archive.org/web/20080813073220/http://local.wasp.uwa.edu.au/~pbourke/dataformats/vt100/)。devkitPro のライブラリですべてのコードに対応しているわけではないことに注意してください。一番よく出会うのは次のものです。

<div class="lblock">
  <table id="tbl:vt100" class="table-data">
    <caption align="bottom">
      <b>{*@tbl:vt100}</b>: よく使われる VT100 のシーケンス
    </caption>
    <tr>
      <td>ESC[<i>dy</i>A</td>
      <td>カーソルを <i>dy</i> 行上に動かす。</td>
    </tr>
    <tr>
      <td>ESC[<i>dy</i>B</td>
      <td>カーソルを <i>dy</i> 行下に動かす。</td>
    </tr>
    <tr>
      <td>ESC[<i>dx</i>C</td>
      <td>カーソルを <i>dx</i> 列右に動かす。</td>
    </tr>
    <tr>
      <td>ESC[<i>dx</i>D</td>
      <td>カーソルを <i>dx</i> 列左に動かす。</td>
    </tr>
    <tr>
      <td>ESC[<i>y</i>;<i>x</i>H</td>
      <td>カーソルを列 <i>x</i>、行 <i>y</i> に設定する。</td>
    </tr>
    <tr>
      <td>ESC[2J</td>
      <td>画面を消す。</td>
    </tr>
    <tr>
      <td>ESC[<i>n</i>K</td>
      <td>
        <ol start=0>
	        <li>行末まで消す。</li>
          <li>行頭まで消す。</li>
	        <li>行全体を消す。</li>
	      </ol>
      </td>
    </tr>
    <tr>
      <td>ESC[<i>y</i>;<i>x</i>f</td>
      <td>ESC[<i>y</i>;<i>x</i>H と同じ</td>
    </tr>
    <tr>
      <td>ESC[s</td>
      <td>カーソルの位置を保存する。</td>
    </tr>
    <tr>
      <td>ESC[u</td>
      <td>カーソルの位置を復元する。</td>
    </tr>
  </table>
</div>

このリストを {@tbl:tte-cmd} と比べると、これらのコードのほとんどに対応する TTE のコマンドがあることがわかるでしょう。どちらを使ってもかまいませんが、クロスプラットフォームにするつもりのものを作るなら、VT100 のコードを使ってください。

:::warning 標準からの逸脱

私の実装はできるだけ標準に近づけようとしています。主に、TTE は通常の背景の 8x8 の文字以外のものも使うからです（訳注: 文脈から、標準どおりにいかない部分がある理由の説明と思われます）。特に、ここではスクロールがなく、色のコードもありません。今のところは。

:::

### UTF-8 {#ssec-misc-utf}

[ASCII](https://en.wikipedia.org/wiki/ASCII) というちょっとしたものについて聞いたことがあるかもしれません。これは文字列の標準のエンコーディングです（でした。よくわかりません）。各文字は 1 バイトの長さで、文字や数字などに 256 個の番号があります。{\*@fig:img-verdana9} と {@fig:img-verdana9-b4} には、Windows で普通に表示されるとおりの、文字 32 から 255 が入っています。ASCII は西洋の言語にはうまく使えますが、何千もの文字がある日本語のような言語にはまったく不十分です。これを解決するために Unicode が考え出され、1 文字あたり 16 ビットになりました。

その中間にあるのが [UTF-8](https://en.wikipedia.org/wiki/UTF-8) です。下位の 128 個の ASCII コードには相変わらず 8 ビットの文字を使いますが、0x80 以上のバイトは複数バイトのコードの始まりを表し、それとそれに続くいくつかの文字が、最大 21 ビットの 1 つの大きな文字になります。

UTF-8 は、いいとこ取りの素敵な方法です。ラテン文字には相変わらず普通の文字を使えるので ASCII のプログラムでもそのまま動き、しかも大きな数を表す方法もあります。

<div class="cblock">
  <table id="tbl:utf8" class="table-data">
    <caption align="bottom">
      {*@tbl:utf8}。UTF-8 から u32 への変換表。
    </caption>
    <tr> 
      <th>文字列（2 進数）</th>
      <th>数値（2 進数）</th>
      <th>範囲（16 進数）</th>
    </tr>
    <tbody style="font:85%, Courier New;">
      <tr>
        <td>0zzzzzzz		</td>
        <td>0zzzzzzz		</td>
        <td>0x000000 - 0x00007F (7 bit)	</td>
      </tr>
      <tr>
        <td>110yyyyy 10zzzzzz	</td>
        <td>00000yyy yyzzzzzz	</td>
        <td>0x000080 - 0x0007FF (11 bit)	</td>
      </tr>
      <tr>
        <td>1110xxxx 10yyyyyy 10zzzzzz	</td>
        <td>xxxxyyyy yyzzzzzz	</td>
        <td>0x000800 - 0x00FFFF (16 bit)	</td>
      </tr>
      <tr>
        <td>11110www 10xxxxxx 10yyyyyy 10zzzzzz	</td>
        <td>000wwwxx xxxxyyyy yyzzzzzz	</td>
        <td>0x010000 - 0x10FFFF (21 bit)	</td>
      </tr>
    </tbody>
  </table>
</div>

{\*@tbl:utf8} は変換の仕組みを示しています。バイトが 128 より小さければ、単純な ASCII の文字です。それ以上なら、3 種類の複数バイトの数のどれかに当てはまります。そのバイトの範囲で全体のバイト数が決まります。それがわかれば、表が示すように、それらのバイトから適切なビットのパターンを取り出して 1 つの数につなげる必要があります。詳しくは Wikipedia のページを見てください。

下に、文字列から UTF-8 の文字を 1 つ読み込んでデコードするルーチンを示します。そう、条件のごちゃごちゃした塊ですが、すべての文字が本当に形式に従っているかを確認するにはそれが必要なのです。形式に従っていなければ、その範囲の最初のバイトを拡張 ASCII の文字として解釈します。望むなら、\``if((*src>>6)!=2) break;`' の文をすべて省いてもかまいません。

```c {#cd-utf8-decode .proglist}
//! Retrieve a single multibyte utf8 character.
uint utf8_decode_char(const char *ptr, char **endptr)
{
    uchar *src= (uchar*)ptr;
    uint ch8, ch32;

    // Poor man's try-catch.
    do
    {
        ch8= *src;
        if(ch8 < 0x80)                      // 7bit
        {
            ch32= ch8;
        }
        else if(0xC0<=ch8 && ch8<0xE0)      // 11bit
        {
            ch32  = (*src++&0x1F)<< 6;  if((*src>>6)!=2)    break;
            ch32 |= (*src++&0x3F)<< 0;
        }
        else if(0xE0<=ch8 && ch8<0xF0)      // 16bit
        {
            ch32  = (*src++&0x0F)<<12;  if((*src>>6)!=2)    break;
            ch32 |= (*src++&0x3F)<< 6;  if((*src>>6)!=2)    break;
            ch32 |= (*src++&0x3F)<< 0;
        }
        else if(0xF0<=ch8 && ch8<0xF8)      // 21bit
        {
            ch32  = (*src++&0x0F)<<18;  if((*src>>6)!=2)    break;
            ch32 |= (*src++&0x3F)<<12;  if((*src>>6)!=2)    break;
            ch32 |= (*src++&0x3F)<< 6;  if((*src>>6)!=2)    break;
            ch32 |= (*src++&0x3F)<< 0;
        }
        else
            break;

        // Proper UTF8 char: set endptr and return
        if(endptr)
            *endptr= (char*)src;

        return ch32;
    } while(0);

    // Not really UTF: interpret as single byte.
    src= (uchar*)ptr;
    ch32= *src++;
    if(endptr)
        *endptr= (char*)src;

    return ch32;
}
```

`tte_write()` も `tte_write_con()` も、文字列がそれを必要とするときは `utf_decode_char()` を使います。大きな文字を使えば、もっと大きなフォントのシートにアクセスできます。大きなシートを使って言語の対応を良くしたり、あるいは標準の文字のセットを矢印やほかの種類の記号で拡張したりできるでしょう。

ただし、stdio で UTF-8 を使うには落とし穴が 1 つあります。内部では、stdio は何を受け付けるかについて本当にうるさいのです。たとえば、著作権の記号 © は拡張文字の番号 0xA9 です。UTF-8 でなければ、文字列の中で 0xA9 だけを使えば正しい記号になりました。しかし、0xA9 だけでは {\*@tbl:utf8} のどの形式にも当てはまらないので、UTF-8 では無効なコードです。この場合 `utf8_decode_char()` は大目に見てくれますが、stdio はそうではなく、終端文字として解釈します。言い換えると、拡張 ASCII の文字には気をつけてください。stdio の関数を使いたいなら、正しい UTF-8 の形式を使わ_なければなりません_。

:::warning printf、UTF-8、拡張 ASCII

devkitArm r22 から、`printf()` などの stdio の関数は UTF-8 のロケールを使います。つまり実質的に、古い版のように「©」や「è」のような文字を直接使うことはできません。完全な複数バイトの UTF-8 の表記を使う必要があります。

:::

### 描画処理のプロファイリング {#ssec-misc-profile}

自分が作ったものがどれくらい速いのかを見ておくのは、いつでも良い考えです。ほとんどのビットマップとタイルの描画処理のように、関数が複雑な場合は特にそうです。

{\*@tbl:tte-profile} に、使える描画処理の大部分について、グリフあたりのサイクル数を示します。これらは、文字列（とライブラリのコード）を ROM に置き、デフォルトのウェイトステート、-O2 の最適化で計測しました。使ったフォントは verdana 9 で、セルの大きさは 8x16 なので、固定幅と可変幅の両方に簡単に使えます。テストの文字列は、[Portal](<https://en.wikipedia.org/wiki/Portal_(video_game)>) の 194 文字の台詞です。

> 「失敗に対する罰則を追加したことにご注意ください。部屋の床に触れると、公式のテスト記録に『不十分』の評価が付き、その後死亡します。幸運を祈ります！」

<div class="lblock">
  <table id="tbl:tte-profile" class="table-data">
    <caption align="bottom">
      <b>{*@tbl:tte-profile}</b>: 描画処理の時間。条件: 194 文字、verdana 9、ROM のコード、デフォルトのウェイト、-O2。
    </caption>
    <tr>
      <th>描画処理</th>
      <th>サイクル/文字</th>
    </tr>
    <tr>
      <td>null</td>
      <td align="right"> 221</td>
    </tr>
    <tr>
      <td>se_drawg</td>
      <td align="right"> 595</td>
    </tr>
    <tr>
      <td>se_drawg_w8h16</td>
      <td align="right"> 370	</td>
    </tr>
    <tr>
      <td>ase_drawg_w8h16</td>
      <td align="right"> 458	</td>
    </tr>
    <tr>
      <td>chr4_drawg_b1cts_base</td>
      <td align="right">3049	</td>
    </tr>
    <tr>
      <td>chr4_drawg_b1cts</td>
      <td align="right">2044	</td>
    </tr>
    <tr>
      <td>chr4_drawg_b1cts_fast</td>
      <td align="right"> 631	</td>
    </tr>
    <tr>
      <td>bmp8_drawg_b1cts_base</td>
      <td align="right">2875	</td>
    </tr>
    <tr>
      <td>bmp8_drawg_b1cts</td>
      <td align="right">2078	</td>
    </tr>
    <tr>
      <td>bmp8_drawg_b1cts_fast</td>
      <td align="right"> 619	</td>
    </tr>
    <tr>
      <td>bmp16_drawg_b1cts_base</td>
      <td align="right">2456	</td>
    </tr>
    <tr>
      <td>bmp16_drawg_b1cts</td>
      <td align="right">1503</td>
    </tr>
    <tr>
      <td>obj_drawg</td>
      <td align="right"> 423</td>
    </tr>
  </table>
</div>

まず、値の大きな違いに注目してください。タイルマップやオブジェクトの数百から、ビットマップやタイルの描画処理の場合の_数千_までです。しかもこれは 1 文字あたりなので、大量のテキストを書くとかなりの処理落ちにつながりかねません。

`null()` 描画処理はダミーの描画処理で、TTE のシステムのオーバーヘッドを調べるのに使います。いろいろ考えれば、200 は実はそれほど悪くありません（ROM のコードだということを思い出してください）。とはいえ、この数値を通常のタイルマップの時間と比べてみてください。ここではオーバーヘッドが時間のかなりの部分を占めています。また、`se_drawg` の標準版と 8×16 版の違いにも注目してください。これは純粋にループのせいです。

TTE のオーバーヘッドの半分は、実は折り返しのコードから来ています。カーソルの設定と確認は比較的遅いことがあるのです。しかも、ここではクリッピングさえ考えていません。

ビットマップとタイルの描画処理については、3 つの版の時間を計りました。[「タイルへのテキストの描画」の節](#ssec-chr-drawg)の `chr4_drawg_b1cts_base()` のひな形を使った「base」版、デフォルトの描画処理である C で最適化した版、そして速いアセンブリ版です。

`bmp16` の変種は、描画先にマスクして挿入する必要がないので、ほかのものより速くなっています。ただ面白いのは、`bmp8` と `chr4` の違いが実質的にゼロであることです。これはおそらく、フォントそのものの配置と何か関係があるのでしょう。

また、base、通常、速い版の比較にも注目してください。`chr4_drawg_b1cts()` は base 版より 33% 速く、`chr4_drawg_b1cts_fast` はさらに 3 倍速いです。そして、631 のうち 200 は TTE のオーバーヘッドだということを思い出してください。つまり実際には 4.5 倍速いのです。これは IWRAM の恩恵だけによるものではありません。ARM か Thumb か、手作りのアセンブリかコンパイルしたコードか、ということも関係しています。

## まとめ {#sec-conc}

私に言わせれば、この章は基本的に[前のテキストの章](text.html)を正しくやり直したものです。あらゆる種類のグラフィックを扱いました。通常/アフィンのタイルマップ、8bpp/16bpp のビットマップ、4bpp のタイル、オブジェクトです。まあ、8bpp のタイルは省きましたが、どのみちタイルへの描画にはひどいモードです。ここで示したグリフの描画の関数は、任意の大きさ、固定幅と可変幅のフォントに使え、しかも効率よくそうしているはずです。

さらに、Tonc のテキストエンジンを紹介しました。これは、これらのいろいろなテキストのファミリーをすべて比較的楽に扱うための仕組みです。最初の準備のあとは、描画先に固有の側面は基本的に片づくので、その機能はずっと再利用しやすくなります。表示のために文字列を処理する最も基本的な側面も扱いました。UTF-8 でエンコードされた文字をフォントのシートのグリフのインデックスに変換する方法と、位置、色、フォントを動的に変えるための書式のタグを実装する方法です。最後に、stdio のルーチンが出力のために呼べるコールバックを作る方法を説明し、`printf()` とその仲間を一般に使えるようにしました。

この章全体が、TTE とそれでできることのショーケースでした。完全に仕上がった状態ではありませんが、テキストを扱うための貴重な財産になりうると思います。少なくとも、ここで示した考え方は、自分のグリフの描画処理やテキストシステムを設計するのに役立つはずです。
