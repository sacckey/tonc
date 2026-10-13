# 24. ラボ

<!-- toc -->

## ラボの紹介 {#sec-intro}

ラボは「laboratory（実験室）」の略で、私が新しいものをいじって遊ぶ場所です。役に立ちそうな新しいものがあっても、まだほかのどこかに置ける状態になっていなければ、しばらくここに置いておきます。つまり、ときどき散らかることもあるかもしれませんが、どのみち実験室とはそういうものなので問題ありません。副産物として、「lab」は「labyrinth（迷宮）」の意味も兼ねるかもしれませんが、それはちょっとしたおまけです <kbd>:)</kbd>。

## 優先順位と描画の順序 {#sec-prio}

この節では、背景とオブジェクトの制御のビットのうち、まだ説明していない最後のもの、<dfn>優先順位</dfn>を扱います。優先順位には 0-3 の 4 段階があり、背景の優先順位は [REG_BGxCNT](regbg.html#tbl-reg-bgxcnt) のビット 0 と 1 で、オブジェクトの優先順位は[属性 2](regobj.html#tbl-oe-attr2) のビット 10 と 11 で設定できます。優先順位の考え方は単純です。優先順位の値が大きいものが先に描画されるので、値の小さいものの後ろに表示されます。これで、たとえばオブジェクトを背景の後ろに置けます。

これはとても単純に聞こえますし、実際そうなのですが、描画の処理の順序にはもう少しだけ話があります。一方には優先順位の設定があり、もう一方にはオブジェクトと背景の<dfn>番号</dfn>があります。オブジェクトには 0 から 127、背景には 0 から 3 の番号が付いています。ここでも、大きな番号は小さな番号の後ろに表示されます。オブジェクトが積み重なっていれば obj 0 が一番上で、背景 0 はほかの背景を覆い、オブジェクトは背景の手前に描かれます。これは優先順位の設定によるものでは*ありません*。実際、優先順位の存在意義はまさに、このデフォルトの順序を変えられるようにすることです。

上のことは、*同じ*優先順位のオブジェクトと背景について成り立ちます。最終的なオブジェクト/背景の順序は、優先順位とオブジェクト/背景の番号の両方で構成されていて、優先順位が最上位の部分だと言えます。たとえば、優先順位 0 に obj 1 と背景 2 が、優先順位 1 に obj 0 と背景 1 があれば、順序は obj1 (prio0)、bg2 (prio0)、obj0 (prio1)、bg1 (prio1) になります。

まあ、*だいたい*は……

### オブジェクトの優先順位のバグとオブジェクトのソート {#ssec-prio-objsort}

ほとんどの部分では、順序は上で述べたとおりですが、obj 0 と obj 1 が重なる部分は例外です。設計上の欠陥と呼んでいいと思うものがあるせいで、「順序 = 優先順位.番号」という考え方は完全には正しくないのです。優先順位と番号の関係が食い違った 2 つのオブジェクトがあり、その間に背景があると、背景の下にあるはずのオブジェクトが、2 つのオブジェクトが重なる領域で背景を透けて見えてしまいます。とても複雑に聞こえますが、それは起きることを言葉ではうまく捉えきれないからにすぎません。基本的には、obj1+prio0、bg0+prio0、obj0+prio1 のようなものは、これらのオブジェクトの長方形が重なると、かなりひどいグラフィックのアーティファクトを起こします。しかし、obj0+prio0、bg0+prio0、obj1+prio1 なら問題ありません。オブジェクトの番号が優先順位と揃っているからです。

ここで<dfn>オブジェクトのソート</dfn>の話になります。先にあるべきオブジェクトが実際に先に*なる*ように、つまり小さいオブジェクトの番号を持つようにする処理です。これは実は優先順位とは別の問題ですが、両方を一度にやってしまうと便利です。原理的には、オブジェクトのソートはほかのソートと同じです。物の配列やリスト（この場合は OBJ_ATTR）があり、それらを何らかの<dfn>キー</dfn>、つまりソートされた順序がどうなるべきかを示す値によって並べなければなりません。

さて、キーは基本的に何でもかまいません。見下ろし型やクォータービューのゲームの多くは Y ソートを使います。Y の値が大きいほど、オブジェクトは手前にあるからです。3D やモード 7 のゲームでは Z ソートを使えます。厳密には、Y ソートはその特殊な場合です。

ソートのアルゴリズムも必要です。[ソートのアルゴリズムの Wiki](https://en.wikipedia.org/wiki/Sorting_algorithm) には選べるものがたくさんあります。アルゴリズムを選ぶときは、ここでソートする項目の数はせいぜい 100 個ほどであること、そして項目の順序はそれほど頻繁には変わらない可能性が高いことを思い出してください。最後に、全体をまとめる方法が必要です。手元のスプライトと OBJ_ATTR の構造体で動くようにするのです。

今のところは、速さではなく、主に*単純さ*を重視して選びました。ソート処理 `id_sort_shell()` は、Numerical Recipes（8 章、321 ページ）にある[シェルソート](https://en.wikipedia.org/wiki/Shellsort)のアルゴリズムを少し変えた版を使っています。パラメータはキーの値の配列と要素の数です。ただし、これらを直接ソートするのではなく（ここではオブジェクトに結びついていないので、それはまったく無意味でしょう）、ソートの結果をインデックスの表 `ids[]` に記録します。

<dfn>インデックスの表</dfn>とは、まあ当然ですが、インデックスの表です。ルーチンが終わったあとで、キーのソートされた順序を示してくれます。この方法なら、オブジェクトのダブルバッファをそのまま保てます。スプライトの管理が単純になるので、私はこれが気に入っています。また、構造体を丸ごと入れ替える必要がありません（もっとも、それはたいていポインタでやるものですが）。そして、このルーチンをオブジェクト専用ではなく、汎用のソート処理として使えるようになります。インデックスの表の型にバイトを選んだことでこれは制限されますが、それは時々しなければならない、領域とのトレードオフの 1 つにすぎません。もちろん、完全な整数を使うように変えるのは難しくありません。

<div id="cd-oe-sort1">

```c
// sort routine (in IWRAM!) 

//! Sort indices via shell sort
/*! \param keys Array of sort keys
    \param ids Array of indices. After completion keys[ids[ii]]
      will be sorted in ascending order.
    \param number of entries.
*/
IWRAM_CODE void id_sort_shell(int keys[], u8 ids[], int count)
{
    u32 ii, inc;
    // find initial 'inc' in sequence x[i+1]= 3*x[i]+1 ; x[1]=1
    for(inc=1; inc<=count; inc++)
        inc *= 3;
    // actual sort
    do
    {
        // division is done by reciprocal multiplication. So no worries.
        inc /= 3;                   // for ARM compile
        // inc = (inc*0x5556)>>16); // for Thumb compile
        for(ii=inc; ii<count; ii++)
        {
            u32 jj, id0= ids[ii]; 
            int key0= keys[id0]
            for(jj=ii; jj>=inc && keys[ids[jj-inc]]>key0; jj -= inc)
                ids[jj]= ids[jj-inc];
            ids[jj]= id0;
        }
    } while(inc > 1);
}
```

</div>

<div id="cd-oe-sort2">

```c
// example of use
IWRAM_CODE void id_sort_shell(int keys[], u8 ids[], int count);

int sort_keys[SPR_COUNT];      // sort keys
u8 sort_ids[SPR_COUNT];        // sorted OAM indices

void foo()
{
    int ii;
    for(ii=0; ii<SPR_COUNT; ii++)
    {
        // setup sort keys  ... somehow
        sort_keys[ii]= ... ;
    }

    // sort the indices
    id_sort_shell(sort_keys, sort_ids, SPR_COUNT);

    // custom object update
    for(ii=0; ii<SPR_COUNT; ii++)
        oam_mem[ii]= obj_buffer[sort_ids[ii]];
}
```

</div>

このルーチンは IWRAM に（そして ARM のコードとしてコンパイルして）置くつもりであることに注意してください。**とんでもなくクソ遅い**からです！ あるいは、遅いと言うべきではなく、高くつくと言うべきかもしれません。

基本的なソートの仕組みを考えてみてください。ソートする要素が *N* 個あります。原理的には、それぞれをほかのすべての要素と比べなければならないので、ルーチンの速さは *N*<sup>2</sup> に比例し、普通は *O*(*N*<sup>2</sup>) と表されます。ここで *O* はオーダー（桁）を表します。ソートにとって、*O*(*N*<sup>2</sup>) は悪いものです。たとえば *N*=128 なら、16k 回の確認が必要になります。それに、実際の確認と更新にかかるサイクル数が掛かります。うれしくありません。

幸い、もっと速い方法があります。ソートのアルゴリズムには少なくとも *O*(*N*·log<sub>2</sub>(*N*)) が欲しいところで、先ほどの Wiki を見ればわかるように、そういうものはたくさんあり、シェルソートもその 1 つです。残念ながら、これでもかなり高くつくことがあります。ここでも *N*=128 なら約 900 になり、係数は 80 以上と大きくなりうるのは確実です。ARM+IWRAM なら、それを 20-30 にまで下げられ、アセンブリでちょっと練習してみると、*N*·log<sub>2</sub>(*N*) の 13 から 22 倍という許容できる値になります。

:::note オーダー記法（ビッグ O 記法）

「ビッグ O」、つまりオーダー記法は、アルゴリズムを比べるのに便利な表現です。表記は *O*( f(*N*) ) で、*N* は扱う要素の数、f(*N*) は関数で、たいていは累乗と対数の組み合わせです。これは、*N* が増えるにつれてアルゴリズムの実行時間がどう増えるかを示します。低いオーダーの関数はいずれ高いオーダーの関数に追い越されるので、一般に前者のほうが望ましいです。

ただし、ここでの鍵となる言葉は「いずれ」です。この記法はアルゴリズムの規模（係数）については何も言っておらず、それは場合によって違います。*N* が十分に小さく、規模が十分に違えば、高いオーダーのルーチンのほうが低いオーダーのものより実際に速いこともあります。

:::

さて、今の設計がどのみち最適とは言えないことは、真っ先に認めます。インデックスの表の代わりに連結リストを使えば速くなるかもしれませんし、ほかにもいろいろあります（除算は[ごまかせる](fixed.html#sec-rmdiv)ので問題ではありません）。しかし、そうするともうそれほど単純ではなくなり、それはここで私が目指していたものではありません。

`id_sort_shell()` が終わると、`obj_buffer[sort_ids[ii]]` がソートされた OAM のエントリを与えるように並んだインデックスの表が得られ、それを使って本当の OAM を更新します。

### 檻の中の DNA {#ssec-prio-demo}

この節のデモは、おそらくこれまでで一番かっこよく、一番複雑なものです。オブジェクトでできた二重らせんが、トーラス状の檻の中心の周りを回転します（{!@fig:prio-demo} を見てください）。ここでは 4 つの背景すべてを使います。1 つはテキスト用（ほんの少ししかありませんが）で、3 つは檻の部品です。オブジェクトを覆い隠す手前の部分、ほかのすべての後ろにある奥の部分、そしてトーラスの真ん中で、その周りをオブジェクトが回ります。つまり、オブジェクトは時によってその手前を通ったり後ろを通ったりします。そして、らせんを作るオブジェクトがあります。2 本の鎖は、それぞれ 48 個の 16x16 の球形のオブジェクトでできています。鎖は色で区別でき、1 本は赤、もう 1 本はシアンです。これらは中央の面の後ろを通るときに暗い赤と暗いシアンに変わります。優先順位の設定を使ってオブジェクトが近くの背景の後ろを通れるようにし、優先順位*と*ソートによって、オブジェクトの順序をなめらかにし、先ほど述べた obj-bg-obj のバグを避けます。まとめると次のとおりです。

-   さまざまな優先順位の設定を持つ 4 つの背景
-   中央の柱の周りを（3D で）回る 96 個のオブジェクト。
-   正しい順序にするための、オブジェクトの優先順位と番号のソート。
-   近くのオブジェクトと遠くのオブジェクトを区別するためのパレットの切り替え。

全体の模式図は {@fig:prio-demo}b で見られます。色の意味は @tbl:prio-demo-x で説明しています。

<div class="cblock">
<table id="fig:prio-demo"
  border=0 cellpadding=2 cellspacing=0>
<tbody valign="top">
<tr>
<td>
  <div class="cpt_fr" style="width:240px;">
  <img src="img/demo/prio_demo.png"
    alt="Priority demo"><br>
  <b>{*@fig:prio-demo}a</b>（左）: 
    優先順位とスプライトの順序のデモ。
  </div>
<td>
  <div class="cpt_fr" style="width:240px;">
  <img src="img/demo/prio_demo_x.png"
    alt="Priority demo schematic"><br>
  <b>{*@fig:prio-demo}b</b>: 
    {!@fig:prio-demo}a の模式図。
  </div>
</tbody>
</table>

<table id="tbl:prio-demo-x" class="table-data">
<caption align="bottom">
  <b>*@tbl:prio-demo-x</b>: 
  {!@fig:prio-demo}b の凡例。
</caption>
<tr>
  <th>色	<th>説明		<th>obj/bg	<th>prio
<tr>
  <th>黄 <td>檻の手前	<td>bg1	<td>prio0
<tr>
  <th>緑	<td>檻の中央	<td>bg2	<td>prio1
<tr>
  <th>青	<td>檻の奥	<td>bg3	<td>prio2
<tr>
  <th>赤	<td>鎖 1	<td>obj_buffer[00..47]	<td>可変
<tr>
  <th>シアン	<td>鎖 2	<td>obj_buffer[48..95]	<td>可変
<tr>
  <th>明るい赤/シアン	<td>手前の球	<td>OAM[0..47]	<td>prio 1
<tr>
  <th>暗い赤/シアン	<td>奥の球	<td>OAM[48..95]	<td>prio 2
</table>
</div>

#### スプライトとらせんの模様

想像がつくでしょうが、このデモで一番やっかいなのはスプライトの部分です。らせんは本質的に 3 次元の経路なので、各球の位置に 3D のベクトルが必要で、座標はもちろん固定小数点数です。スプライト（球）を正しい OBJ_ATTR（画面上のオブジェクト）に結びつけるために、OAM の影へのインデックスも必要です。

```c
typedef struct tagSPR_BASE
{
    VECTOR pos; // position (x, y, z)
    int id;     // oe-id in OAM buffer
} SPR_BASE;

#define SPR_COUNT 96

SPR_BASE sprites[SPR_COUNT];    // Sprite list
```

<div class="cpt_fr" style="width:144px;">
<img src="img/math/helix.png" id="fig:helix" 
  alt="3 periods of a helix" width=144><br>
<b>*@fig:helix</b>: らせんの 3 周期。
</div>

らせんは、単に円の媒介変数表示を、その法線の軸の方向に押し出したものです（@fig:helix を見てください）。3 つの主軸の向きに注意してください。これは右手系で、*x* と *y* は画面の軸の向きに従い、*z* は画面の奥を指しています。*y* 軸の周りを回転するらせんは、次の関係で表せます。

<!--
\textbf{x}(y,t)=\begin{bmatrix}
A \cdot \cos(k\cdot y+\omega t) \\
y \\
A \cdot \sin(k\cdot y+\omega t)
\end{bmatrix}
-->
<table id="eq:helix">
<tr>
  <td class="eqnrcell">(!@eq:helix)
  <td class="eqcell">
<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtext mathvariant="bold">x</mtext>
            <mo stretchy="false">(</mo>
            <mi>y</mi>
            <mo>,</mo>
            <mi>t</mi>
            <mo stretchy="false">)</mo>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <mi>A</mi>
                    <mo>&#x22C5;</mo>
                    <mi>cos</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mrow>
                      <mo data-mjx-texclass="OPEN">(</mo>
                      <mi>k</mi>
                      <mo>&#x22C5;</mo>
                      <mi>y</mi>
                      <mo>+</mo>
                      <mi>&#x3C9;</mi>
                      <mi>t</mi>
                      <mo data-mjx-texclass="CLOSE">)</mo>
                    </mrow>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mi>y</mi>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mi>A</mi>
                    <mo>&#x22C5;</mo>
                    <mi>sin</mi>
                    <mo data-mjx-texclass="NONE">&#x2061;</mo>
                    <mrow>
                      <mo data-mjx-texclass="OPEN">(</mo>
                      <mi>k</mi>
                      <mo>&#x22C5;</mo>
                      <mi>y</mi>
                      <mo>+</mo>
                      <mi>&#x3C9;</mi>
                      <mi>t</mi>
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
  </td>
</tr>
</table>

*A* はらせんの半径、*k* は波数（*k*=2π/λ）、ω は角速度（ω=2π/*T*）です。波数はらせんの層の間隔（つまりピッチ）を決め、角速度は回転の速さを与えます。@fig:helix のらせんを作るには実際には負の波数が必要ですが、今それはあまり重要ではないことに注意してください。

実際のコードでは、らせん全体を乱さずに ω を可変にできるように、上の式を少し変えます。単なる ω*t* の代わりに、その積分を初期位相の角度に使います。φ<sub>0</sub>=∫ω(*t*)d*t* です。φ<sub>0</sub> はらせんを作る関数のパラメータになり、ほかのところで管理します。模様のもう 1 つのパラメータは、らせんの基準点 **p**<sub>0</sub> です。これは必ず 1 つ必要です。

```c
// some constants
const VECTOR P_ORG= { 112<<8, 0<<8, 0<<8 };
#define AMP     0x3800  // amplitude (.8)
#define WAVEN  -0x002C  // wave number (.12)
#define OMEGA0  0x0200  // angular velocity (.8)

//   phi0(t) = INT(w(t'), t', 0, t)
//   (x,y,z) = ( x0+A*cos(k*y+ft), y0+y, z0+A*sin(k*y+phi0) )
void spr_helix(const VECTOR *p0, int phi0)
{
    int ii, phi;
    VECTOR dp= {0, 0, 0};
    SPR_BASE *sprL= sprites, *sprR= &sprites[SPR_COUNT/2];

    for(ii=0; ii<SPR_COUNT/2; ii++)
    {
        // phi: 0.9f ; dp: 0.8f ; WAVEN:0.12f ; phi0: 0.8f
        phi= (WAVEN*dp.y>>11) + (phi0>>7);

        // red helix
        dp.x=  AMP*lut_cos(phi)>>8;
        dp.z=  AMP*lut_sin(phi)>>8;
        vec_add(&sprL[ii].pos, p0, &dp);

        // cyan helix
        dp.x= -dp.x;
        dp.z= -dp.z;
        vec_add(&sprR[ii].pos, p0, &dp);

        dp.y += 144*256/(SPR_COUNT/2);
    }
}
```

このルーチンはかなり素直です。*y* の進行中のカウンタを `dp.y` の形で保持し、それを使って完全な位相を計算し、そこからサインとコサインを得ます。赤とシアンのらせんは逆位相なので、一方の *x* と *z* のオフセットは、もう一方の符号を入れ替えるだけで得られます。本当にやっかいなのは、位相のいろいろな固定小数点のスケールの管理だけです。固定小数点の計算を扱うときは、常に小数部のビット数を示しておいてください。とても簡単に迷子になりますから。



二重らせんの模様ができたので、ソートなども含めて、それをオブジェクトに結びつける方法が必要です。

```c
void spr_update()
{
    int ii, prio, zz, *key;
    u32 attr2;
    int *key= sort_keys;
    SPR_BASE *spr= sprites;
    OBJ_ATTR *oe;

    for(ii=0; ii<SPR_COUNT; ii++)
    {
        oe= &obj_buffer[spr->id];
        // set x/y pos
        obj_set_pos(oe, spr->pos.x>>8, spr->pos.y>>8);

        // set priority based on depth.
        // HAX 1: palette swapping
        attr2= oe->attr2 & ~(ATTR2_PRIO_MASK | (1<<ATTR2_PALBANK_SHIFT));
        zz= spr->pos.z;
        if(zz>0)
        {
            prio= 2;
            attr2 |= 1<<ATTR2_PALBANK_SHIFT;
        }
        else
            prio= 1;
        oe->attr2= attr2 | (prio<<ATTR2_PRIO_SHIFT);

        // HAX 2: sort-key contruction
        *key++= (prio<<30) + (zz>>2) - 6<<28;
        spr++;
    }

    if(g_state & S_SORT) // sort and update
    {
        id_sort_shell(sort_keys, sort_ids, SPR_COUNT);
        for(ii=0; ii<SPR_COUNT; ii++)
            oam_mem[ii]= obj_buffer[sort_ids[ii]];
    }
    else                 // regular update
        oam_update(0, SPR_COUNT);
}
```

ここでの大きなループは、*本当の OAM ではなく*、OAM の影を更新します！ スプライトの *x* と *y*（もちろん固定小数点を補正したもの）を使ってオブジェクトの位置を更新し、*z* を使って優先順位を設定します。手前側（中央の柱の前）にあれば 1、奥側（柱の後ろ）にあれば 2 です。パレットについても*ちょっと*変わったことをしていて、それがこの関数の 1 つ目の小技で、すぐあとに 2 つ目が続きます。

<div class="cpt_fr"  style="width:120px;">
<table id="tbl:prio-hack1" class="table-data">
<caption align="bottom">
  <b>*@tbl:prio-hack1</b>: オブジェクトのパレットバンク。
</caption>
<tr><th>バンク	<th>色
<tr><td>4		<td>明るい赤
<tr><td>5		<td> 暗い赤
<tr><td>6		<td>明るいシアン
<tr><td>7		<td> 暗いシアン
</table>
</div>

**小技 1**。オブジェクトのパレットを、赤がパレットバンク 4 と 5 に、シアンがバンク 6 と 7 に来るように並べておきました（@tbl:prio-hack1）。つまり、パレットバンクの最初のビット、attr2 のビット 12 を切り替えることで、明るい版と暗い版を切り替えられます。

そのすぐあとに 2 つ目の小技、ソートのキーの作成があります。

**小技 2**。ソートのキーは、優先順位（2 ビット）と奥行き *z*（残り）の組み合わせです。`zz` の下位 30 ビットは優先順位の段階に対する**符号付き**のオフセットとして働くので、各優先順位は、必要なら \[-2\<\<30,2\<\<30⟩ という自分の奥行きの範囲を持ちます。問題は、キーも符号付きなので、優先順位 2 と 3 が負とみなされ、優先順位 0 と 1 より手前にソートされてしまうことで、それはまずいことです。これを解決するために 0x60000000 を引きます。これで優先順位 0 の範囲が、あるべき一番負の範囲にずれます。

関数の最後の部分は、ソートありかなしで、OAM の影を OAM に反映します。

:::note 無効なオブジェクトのソート

ちなみに、ソートのキーの作成を、無効な/隠れたオブジェクトも考慮するように簡単に変えられます。必要なのは、ソートのキーに（符号付きの）最大値、この場合は 0x7FFFFFFF を割り当てることだけです。

```c
if( (oe->attr0&ATTR0_MODE_MASK) != ATTR0_HIDE )
    *key++= (prio<<30) + (zz>>2) - 6<<28; 
else
    *key++= 0x7FFFFFFF;
```

:::

#### コードの残り

コードの残りは `main()` と初期化のコードだけです。初期化のコードのほとんどは、グラフィックの読み込みやレジスタの初期化など、かなり標準的なものです。唯一面白い部分はオブジェクトの初期化で、赤とシアンの球のパレットバンクを 0x4000 と 0x6000 に設定しています。そして、ソートはオブジェクトのバッファそのものを変えるのではなくインデックスの表を使うので、主な色を正しく保つために必要なのはこれだけです。

```c
#define S_AUTO  0x0001
#define S_SORT  0x0002

const VECTOR P_ORG= { 112<<8, 0<<8, 0<<8 };

int g_phi= 0;                   // phase, integration of omega over time
int g_omega= OMEGA0;            // rotation velocity (.8)
u32 g_state= S_AUTO | S_SORT;   // state switches


void main_init()
{
    int ii;
    // --- init gfx ---
    // bgs
    memcpy32(pal_bg_mem, cagePal, cagePalLen/4);
    pal_bg_mem[0]= CLR_BLACK;
    memcpy32(tile_mem[1], cageTiles, cageTilesLen/4);
    // Hacx 3: there are 3 maps in cageMap, which have to be extracted manually
    // front part, priority 0
    memcpy32(se_mem[5], &cageMap[ 1*32], 20*32/2);
    REG_BG1CNT= BG_CBB(1) | BG_SBB(5) | BG_8BPP | BG_PRIO(0);
    // center, priority 1
    memcpy32(se_mem[6], &cageMap[22*32], 20*32/2);
    REG_BG2CNT= BG_CBB(1) | BG_SBB(6) | BG_8BPP | BG_PRIO(1);
    // back, priority 2
    memcpy32(se_mem[7], &cageMap[43*32], 20*32/2);
    REG_BG3CNT= BG_CBB(1) | BG_SBB(7) | BG_8BPP | BG_PRIO(2);

    // object
    memcpy32(&tile_mem[4][1], ballTiles, ballTilesLen/4);
    memcpy32(pal_obj_mem, ballPal, ballPalLen/4);

    // -- init vars ---
    // init sort list
    for(ii=0; ii<SPR_COUNT; ii++)
        sprites[ii].id= sort_ids[ii]= ii;

    // --- init sprites and objects ---
    oam_init();
    for(ii=0; ii<SPR_COUNT/2; ii++)
    {
        obj_set_attr(&obj_buffer[ii], 0, ATTR1_SIZE_16, 0x4001);
        obj_set_attr(&obj_buffer[ii+SPR_COUNT/2], 0, 
            ATTR1_SIZE_16, 0x6001);
    }

    spr_helix(&P_ORG, 0);
    spr_update();

    REG_DISPCNT= DCNT_BG_MASK | DCNT_OBJ | DCNT_OBJ_1D;
    int_init();
    int_enable_ex(II_VBLANK, NULL);
    txt_init_std();
    txt_init_se(0, BG_CBB(3)|BG_SBB(31), 0, 0xEC00021F, 0xEE);
}

int main()
{
    char str[32];

    main_init();

    while(1)
    {
        VBlankIntrWait();
        // kery handling
        key_poll();
        if(key_hit(KEY_START))
            g_state ^= S_AUTO;
        if(key_hit(KEY_SELECT))
            g_state ^= S_SORT;

        // movement
        if(g_state & S_AUTO)
        {
            g_omega += key_tri_shoulder()<<4;
            g_phi += g_omega;
        }
        else
            g_phi += g_omega*key_tri_shoulder();

        // sprite/obj update
        spr_helix(&P_ORG, g_phi);
        spr_update();

        // print omega
        siprintf(str, "%6d", g_omega);
        se_puts(8, 136, str, 0);
    }

    return 0;
}
```

<div class="cpt_fr" style="width:240px;">
<img src="img/demo/prio_demo2.png" id="fig:prio-demo2" 
  alt=""><br>
<b>*@fig:prio-demo2</b>: ソートを切った状態。
</div>

メインループは状態の変化を確認し、スプライトとオブジェクトを進めて更新し、現在の角速度を表示します。

`g_state` には 2 つの状態のスイッチがあります。1 つはソートの処理を切り替えるもの（`S_SORT`、**Select** ボタン）、もう 1 つは回転を自動にするかどうかを設定するもの（`S_AUTO`、**Start** ボタン）です。ソートの切り替えは、優先順位を設定するだけだと何が起きるかがわかるので面白いです。これには 2 つの影響があります（@fig:prio-demo2 を見てください）。1 つ目は、各鎖の球の順序が固定され、すべてのオブジェクトが左隣のものを一部覆い隠すことで、鎖が遠ざかっていく部分ではこれは正しくありません。これは一番右側で最もよく見え、鎖が切れているように見えます。2 つ目の影響は、オブジェクトの優先順位/番号の順序のバグで、奥にあるオブジェクトが、それを覆い隠しているはずの背景を透けて見えてしまいます。

Start ボタンは自動と手動の回転を切り替えます。自動モードの間は、**L** ボタンと **R** ボタンで ω を変えられます。手動モードでは、L と R が現在の角速度で位相を更新します。速さをとても遅くすれば、何が起きているかをもっと詳しく調べられます。たとえば、縦の中心線にあるオブジェクトが左右両方の隣のものより手前にあることがはっきりわかります。まさに期待どおりです。ソートが切られていなければ、ですが。

これで優先順位とオブジェクトのソートの話は終わりです。描画の順序を決めるのはオブジェクトと背景の優先順位だけではなく、各優先順位の段階ではオブジェクトや背景の番号も重要だということを覚えておいてください。オブジェクトと背景の優先順位を混ぜ始めたら、オブジェクトの番号が優先順位と同じ順序になるようにしてください。それにはたいていオブジェクトのソートが必要です。

単純で柔軟なソートの方法を説明しましたが、時間がかかることは警告しておきます。十分に良ければ、ぜひ使ってください。そうでなければ、もっと速い方法はきっと作れます。連結リスト、範囲の確認、手作りのアセンブリ（たとえば *prio_demo* ディレクトリの *id_sort_shell2.s* を見てください）は、どれも速くするのに役立ちます。でも、最終的な実装はあなたにお任せします。
