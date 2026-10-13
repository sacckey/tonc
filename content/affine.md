# 10. アフィン変換の行列（別名 **P**）

<!-- toc -->

## このページについて

ご存じかもしれませんが、GBA はスプライトや背景に、回転や拡大縮小などの幾何学的な変換をかけられます。通常のものと区別するため、変換できるものは一般に Rot/Scale スプライト、Rot/Scale 背景と呼ばれます。変換は `pa`、`pb`、`pc`、`pd` の 4 つのパラメータで表されます。置き場所や正確な名前はスプライトと背景で違いますが、今はそれは気にしなくてかまいません。

これらの数値には 2 通りの解釈があります。1 つ目は、それぞれをスプライトや背景のデータに対する個別のオフセットと考えるものです。[GBATEK](https://problemkaputt.de/gbatek.htm) や [CowBite Spec](http://www.cs.rit.edu/~tjh8300/CowBite/CowBiteSpec.htm) のようなリファレンス文書は、この形で説明しています。もう 1 つは、これらを 2x2 行列の要素と見なすもので、この行列を **P** と呼ぶことにします。ほとんどすべてのチュートリアルは、こちらの形で説明しています。そして、そうしたチュートリアルは回転と拡大縮小の行列として次のものを挙げています。

<math id="eq:incorrect_transform_matrix" class="block">
    <mo>(</mo>
    <mi>{!@eq:incorrect_transform_matrix}</mi>
    <mo>)</mo>
    <mspace width="30px" />
    <mi>𝗣</mi>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><msub><mi>p</mi><mi>a</mi></msub></mtd>
                <mtd><msub><mi>p</mi><mi>b</mi></msub></mtd>
            </mtr>
            <mtr>
                <mtd><msub><mi>p</mi><mi>c</mi></msub></mtd>
                <mtd><msub><mi>p</mi><mi>d</mi></msub></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><msub><mi>s</mi><mi>x</mi></msub><mi>cos</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
                <mtd><msub><mi>s</mi><mi>y</mi></msub><mi>sin</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
            </mtr>
            <mtr>
                <mtd><mo>-</mo><msub><mi>s</mi><mi>x</mi></msub><mi>sin</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
                <mtd><msub><mi>s</mi><mi>y</mi></msub><mi>cos</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
</math>

たしかにこれは回転と拡大縮小の行列です。残念ながら、これは<span class="rem">間違った行列</span>でもあるのです！ 少なくとも、おそらくあなたの期待どおりには動きません。たとえば、<math><msub><mi>s</mi><mi>x</mi></msub></math> = 1.5、<math><msub><mi>s</mi><mi>y</mi></msub></math> = 1.0 の拡大縮小と、α= 45 の回転を考えてみましょう。おそらく {@fig:rotatescale}a のようなものを期待するでしょうが、実際に得られるのは {@fig:rotatescale}b です。スプライトは回転していますが向きが逆で、拡大ではなく縮小されており、さらに余計なせん断までかかっています。もちろん「こうなるのを狙っていた」と言い張ることはいつでもできますが、たぶんそれは本当ではないでしょう。

<div style="display: flex; margin: 20px">
<div class="cpt" style="width:160px;">

<img src="./img/affine/metr_rs_good.png" id="fig:rotatescale">  

**{*@fig:rotatescale}a**: 「回転して拡大縮小」と言うとき、おそらく期待するのはこれでしょう…

</div>
&nbsp;
<div class="cpt" style="width:160px">
<img src="./img/affine/metr_rs_bad.png" id="fig:rotatescale">  


**{*@fig:rotatescale}b**: しかし {@eq:incorrect_transform_matrix} の **P** を使うと、こうなります。
</div>
</div>

残念ながら、変換行列については間違った情報や誤解を招く情報がたくさん出回っています。{@eq:incorrect_transform_matrix} の行列は、その一例にすぎません。そもそも「Rot/Scale」という呼び名からして実際に起きることと合っていませんし、使われる用語がきちんと定義されることもなく、しかも多くの人は情報が正しいかどうか確かめようともせずに、他人の説明をそのままコピーしてしまいます。皮肉なことに、一番の拠り所であるリファレンス文書の GBATEK は各要素を正しく説明しているのですが、チュートリアルで行列の形に書き直されるときに、なぜかそれが失われてしまったのです。

この章では、**P** 行列の**正しい**解釈を説明します。GBA がそれをどう使うのか、そして自分でどう組み立てるのかです。ただし、そのために完全に数学モードに入ります。ベクトルや行列の計算に慣れていないと、細かいところを理解するのに少し苦労するかもしれません。この話題の手がかりとして、[線形代数](matrix.html)の付録を用意してあります。

このページは純粋に理論の話です。スプライトや背景に直接関係することは何も出てきません。それは次の 2 つの節の役目です。今回もまた、愛らしいメトロイドに手伝ってもらいます（安全に使うため、冷凍保存しておいてください）。y 軸と角度の向きに注意してください。そして、[仕上げ](#sec-finish)の段落は*必ず*読んでください。そこには、実装上の重要な点がいくつも書いてあります。それより前の本文では、話の邪魔になるだけなので、それらの点は無視しています。

:::warning アフィンパラメータを扱う文書には気をつけよう

本当です。この話題を扱った文書で私が見たものは、ほとんどすべて何かしら問題を抱えています。多くは間違った回転・拡大縮小行列（つまり {@eq:incorrect_transform_matrix} の行列）を載せているか、行列やその要素の名前や説明を誤っています。

:::

## テクスチャマッピングとアフィン変換

### 一般的な 2D テクスチャマッピング

GBA がスプライトやタイル背景を画面に出すためにしていることは、テクスチャマッピングにとてもよく似ています。そこで、GBA のことはいったん忘れて、テクスチャマッピングがどう行われるのかを見てみましょう。{@fig:metroid_texture}a は、メトロイドのテクスチャです。便宜上、標準的なデカルト座標系の 2D 座標（y 軸が上向き）を使い、テクスチャを正規化しています。つまり、テクスチャの右辺と上辺は、ちょうど単位ベクトル <math><msub><mi>e</mi><mi>x</mi></msub></math> と <math><msub><mi>e</mi><mi>y</mi></msub></math>（長さは 1）に対応します。テクスチャマッピングは、（テクスチャ空間の）**p** を（画面空間の）点 **q** に移します。実際の写像は、2×2 行列 **A** で行います。

<math class="block">
    <mi>𝗾</mi>
    <mo>=</mo>
    <mi>𝗔</mi>
    <mo>&middot;</mo>
    <mi>𝗽</mi>
</math>

では、**A** はどうやって求めるのでしょうか？ 実は、そんなに難しくありません。この行列は、変換後の基底ベクトル **u** と **v** を並べてできます（ちなみに、これは何次元でも成り立ちます）。したがって次のようになります。

<math class="block">
    <mi>𝗔</mi>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><msub><mi>u</mi><mi>x</mi></msub></mtd>
                <mtd><msub><mi>v</mi><mi>x</mi></msub></mtd>
            </mtr>
            <mtr>
                <mtd><msub><mi>u</mi><mi>y</mi></msub></mtd>
                <mtd><msub><mi>v</mi><mi>y</mi></msub></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
</math>

<div style="display: flex; align-items: center" markdown>
<div class="cpt" style="width:128px;" markdown>
<img src="./img/affine/metr_tex.png" id="fig:metroid_texture">  

**{*@fig:metroid_texture}a**: テクスチャ。
</div>

<span style="font-size: 3em; display: inline-flex; flex-direction: column; padding: 10px">
A <br>
→
</span>

<div class="cpt" style="width:128px;" markdown>
<img src="./img/affine/metr_texmapA.png" id="fig:metroid_texture">  

**{*@fig:metroid_texture}b**: マッピングされたテクスチャ
</div>
</div>

アフィン行列 **A** による順方向のテクスチャマッピング。

### アフィン変換

2D の行列でできる変換は、<dfn>[アフィン](https://en.wikipedia.org/wiki/Affine_geometry)</dfn>変換と呼ばれます。アフィン変換の技術的な定義は「平行な線を平行なまま保つ変換」です。これは基本的に、行列による変換として書けるということであり、長方形がアフィン変換によって平行四辺形になるということでもあります（{@fig:metroid_texture}b を見てください）。

アフィン変換には、回転と拡大縮小だけでなく、せん断*も*含まれます。私が「Rot/Scale」という名前に反対するのはこのためです。この用語は特殊な場合を指しているだけで、一般の変換を指してはいません。これは、色のことを「赤の色合い」と呼ぶようなものです。たしかに赤も色ですが、すべての色が赤というわけではありませんし、そう呼んでしまうと対象を歪めて捉えることになります。

先ほど述べたように、基本的な 2D の変換は 3 つあります。ただし、どれか 1 つはいつでも残りの 2 つで表せます。その変換とは、回転（**R**）、拡大縮小（**S**）、せん断（**H**）です。{*@tbl:transformation_matrices_and_their_inverses} は、それぞれの変換が普通のメトロイドのスプライトをどう変えるかを示しています。黒い軸は通常の基底ベクトル（*y* が下向きなのに注意！）、青い軸は変換後の基底ベクトル、シアンの変数は変換の引数です。各変換の行列と逆行列も載せてあります。なぜかって？ そのうちわかります。

  
<math id="eq:transformation_matrix_and_inverse" class="block">
    <mo>(</mo>
    <mi>{!@eq:transformation_matrix_and_inverse}</mi>
    <mo>)</mo>
    <mspace width="30px" />
    <mi>𝗔</mi>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><mi>a</mi></mtd>
                <mtd><mi>b</mi></mtd>
            </mtr>
            <mtr>
                <mtd><mi>c</mi></mtd>
                <mtd><mi>d</mi></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
    <mspace width="30px"/>
    <msup>
        <mi>𝗔</mi>
        <mn>-1</mn>
    </msup>
    <mo>≡</mo>
    <mfrac>
        <mn>1</mn>
        <mrow>
            <mi>a</mi><mi>d</mi><mo>-</mo><mi>b</mi><mi>c</mi>
        <mrow>
    </mfrac>
        <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><mi>d</mi></mtd>
                <mtd><mo>-</mo><mi>b</mi></mtd>
            </mtr>
            <mtr>
                <mtd><mo>-</mo><mi>c</mi></mtd>
                <mtd><mi>a</mi></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
</math>

{*@tbl:transformation_matrices_and_their_inverses}: 変換行列とその逆行列。
<table id="tbl:transformation_matrices_and_their_inverses" class="table-data">
    <thead>
        <tr>
            <th>恒等変換</th>
            <th>回転</th>
            <th>拡大縮小</th>
            <th>せん断</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td markdown><img alt="Normal metroid" src="img/affine/metr_id.png" /></td>
            <td markdown><img alt="Rotated metroid" src="img/affine/metr_rot.png" /></td>
            <td markdown><img alt="Scaled metroid" src="img/affine/metr_scale.png" /></td>
            <td markdown><img alt="Sheared metroid" src="img/affine/metr_shear.png" /></td>
        </tr>
        <tr>
            <td>
                <math>
                    <mi>𝗜</mi>
                    <mo>=</mo>
                    <mrow>
                        <mo>[</mo>
                        <mtable>
                            <mtr>
                                <mtd><mn>1</mn></mtd>
                                <mtd><mn>0</mn></mtd>
                            </mtr>
                            <mtr>
                                <mtd><mn>0</mn></mtd>
                                <mtd><mn>1</mn></mtd>
                            </mtr>
                        </mtable>
                        <mo>]</mo>
                    </mrow>
                </math>
            </td>
            <td>
                <math>
                    <mi>𝗥</mi>
                    <mo>(</mo>
                    <mn>&theta;</mn>
                    <mo>)</mo>
                    <mo>=</mo>
                    <mrow>
                        <mo>[</mo>
                        <mtable>
                            <mtr>
                                <mtd><mi>cos</mi><mo>(</mo><mn>&theta;</mn><mo>)</mo></mtd>
                                <mtd><mo>-</mo><mi>sin</mi><mo>(</mo><mn>&theta;</mn><mo>)</mo></mtd>
                            </mtr>
                            <mtr>
                                <mtd><mi>sin</mi><mo>(</mo><mn>&theta;</mn><mo>)</mo></mtd>
                                <mtd><mi>cos</mi><mo>(</mo><mn>&theta;</mn><mo>)</mo></mtd>
                            </mtr>
                        </mtable>
                        <mo>]</mo>
                    </mrow>
                </math>
            </td>
            <td>
                <math>
                    <mi>𝗦</mi>
                    <mo>(</mo>
                    <msub><mi>s</mi><mi>x</mi></msub>
                    <mo>,</mo>
                    <msub><mi>s</mi><mi>y</mi></msub>
                    <mo>)</mo>
                    <mo>=</mo>
                    <mrow>
                        <mo>[</mo>
                        <mtable>
                            <mtr>
                                <mtd><msub><mi>s</mi><mi>x</mi></msub></mtd>
                                <mtd><mn>0</mn></mtd>
                            </mtr>
                            <mtr>
                                <mtd><mn>0</mn></mtd>
                                <mtd><msub><mi>s</mi><mi>y</mi></msub></mtd>
                            </mtr>
                        </mtable>
                        <mo>]</mo>
                    </mrow>
                </math>
            </td>
            <td>
                <math>
                    <mi>𝗛</mi>
                    <mo>(</mo>
                    <msub><mi>h</mi><mi>x</mi></msub>
                    <mo>,</mo>
                    <msub><mi>h</mi><mi>y</mi></msub>
                    <mo>)</mo>
                    <mo>=</mo>
                    <mrow>
                        <mo>[</mo>
                        <mtable>
                            <mtr>
                                <mtd><mn>1</mn></mtd>
                                <mtd><msub><mi>h</mi><mi>x</mi></msub></mtd>
                            </mtr>
                            <mtr>
                                <mtd><msub><mi>h</mi><mi>y</mi></msub></mtd>
                                <mtd><mn>1</mn></mtd>
                            </mtr>
                        </mtable>
                        <mo>]</mo>
                    </mrow>
                </math>
            </td>
        </tr>
        <tr>
            <td>
                <math>
                    <msup><mi>𝗜</mi><mn>-1</mn></msup>
                    <mo>=</mo>
                    <mi>𝗜</mi>
                </math>
            </td>
            <td>
                <math>
                    <msup><mi>𝗥</mi><mn>-1</mn></msup><mo>(</mo><mn>&theta;</mn><mo>)</mo>
                    <mo>=</mo>
                    <mi>𝗥</mi><mo>(</mo><mn>-&theta;</mn><mo>)</mo>
                </math>
            </td>
            <td>
                <math>
                    <msup><mi>𝗦</mi><mn>-1</mn></msup>
                    <mo>(</mo>
                    <msub><mi>s</mi><mi>x</mi></msub>
                    <mo>,</mo>
                    <msub><mi>s</mi><mi>y</mi></msub>
                    <mo>)</mo>
                    <mo>=</mo>
                    <mi>𝗦</mi>
                    <mo>(</mo>
                    <mfrac><mn>1</mn><msub><mi>s</mi><mi>x</mi></msub></mfrac>
                    <mo>,</mo>
                    <mfrac><mn>1</mn><msub><mi>s</mi><mi>y</mi></msub></mfrac>
                    <mo>)</mo>
                </math>
            </td>
            <td>
                <math>
                    <msup><mi>𝗛</mi><mn>-1</mn></msup>
                    <mo>(</mo>
                    <msub><mi>h</mi><mi>x</mi></msub>
                    <mo>,</mo>
                    <msub><mi>h</mi><mi>y</mi></msub>
                    <mo>)</mo>
                    <mo>=</mo>
                    <mfrac>
                        <mrow>
                            <mi>𝗛</mi>
                            <mo>(</mo>
                            <mn>-</mn><msub><mi>h</mi><mi>x</mi></msub>
                            <mo>,</mo>
                            <mn>-</mn><msub><mi>h</mi><mi>y</mi></msub>
                            <mo>)</mo>
                        </mrow>
                        <mrow>
                            <mn>1</mn>
                            <mo>-</mo>
                            <msub><mi>h</mi><mi>x</mi></msub>
                            <msub><mi>h</mi><mi>y</mi></msub>
                        </mrow>
                    </mfrac>
                </math>
            </td>
        </tr>
    </tbody>
</table>

これらの定義を使えば、<math><msub><mi>s</mi><mi>x</mi></msub></math> と <math><msub><mi>s</mi><mi>y</mi></msub></math> による拡大のあとに α（=−θ）による**反時計回り**の回転をする、正しい行列を行列の積で求められます。

<math id="eq:inverse_transform" class="block">
    <mo>(</mo>
    <mi>{!@eq:inverse_transform}</mi>
    <mo>)</mo>
    <mspace width="30px" />
    <mi>𝗔</mi>
    <mo>=</mo>
    <mi>𝗥</mi><mo>(</mo><mn>-&alpha;</mn><mo>)</mo>
    <mo>&middot;</mo>
    <mi>𝗦</mi><mo>(</mo><msub><mi>s</mi><mi>x</mi></msub><mo>,</mo><msub><mi>s</mi><mi>y</mi></msub><mo>)</mo>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><msub><mi>s</mi><mi>x</mi></msub><mi>cos</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
                <mtd><msub><mi>s</mi><mi>y</mi></msub><mi>sin</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
            </mtr>
            <mtr>
                <mtd><mo>-</mo><msub><mi>s</mi><mi>x</mi></msub><mi>sin</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
                <mtd><msub><mi>s</mi><mi>y</mi></msub><mi>cos</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
</math>

…えーと、ちょっと待ってください…なんだか妙な既視感が…

:::note 時計回りと反時計回り

細かい話ですが、触れておかなければなりません。**R** の定義が時計回りの回転なのに、なぜ急に反時計回りを使うのでしょうか？ 実は、**R** は伝統的にあの形の行列で表され、その角度は x 軸から y 軸に向かって進みます。*y* が下向きなので、これは時計回りになります。しかし、BIOS のアフィン関数は反時計回りの回転を使っているので、私の関数でもそれを指針にするのがよいと考えたのです。

:::

:::note 用語: Affine と Rot/Scale

行列 **P** は回転行列でも拡大縮小行列でもなく、一般的なアフィン変換の行列です。この行列は主に回転と拡大縮小に使われるかもしれませんが、だからといって、「Rot/Scale」という用語が示すように、それしかできないわけではありません。

通常の背景やスプライトと区別するためなら、「Rotation」や「Rot/Scale」でも、完全に正確ではないにせよ十分だと思います。しかし、**P** 行列をそうした名前で呼ぶのは、単純に間違いです。

:::

## 「我々が信じている真実の多くは、自分自身の物の見方次第なのじゃ。」

<div class="cpt_fr" style="width:160px">

<img src="./img/affine/metr_texmapA.png" id="fig:human_pov">  

**{*@fig:human_pov}**: 人間から見た写像の過程。**u** と **v** は（画面空間での）**A** の列です。

</div>

お気づきのとおり、{@eq:inverse_transform} は、私が間違いだと言った {@eq:incorrect_transform_matrix} とまったく同じです。いったいどういうことでしょうか？ たしかに、この行列を `pa-pd` の要素に入れると、期待とは違うものが得られます。ただし今回は、そもそも何を期待すべきだったのか（つまり、<math><msub><mi>s</mi><mi>x</mi></msub></math> と <math><msub><mi>s</mi><mi>y</mi></msub></math> による拡大縮小のあとに、α による反時計回りの回転）を、きちんと証明しました。*本当の*問題はもちろん、なぜこれがうまくいかないのか、です。これに答えるために、2D の写像の過程について 2 つの異なる見方を紹介します。

### 人間の視点

「こんにちは、Cearn の脳です。私は幾何学を理解していて、行列の変換を頭の中でできます。まあ、正確には彼の頭の中ですが。テクスチャマッピングでは、元のマップ（テクスチャ空間にあるもの）を見て、それから変換を思い描きます。元のマップを見て、マップのピクセルが画面上のどこに行き着くかを見るのです。このための変換行列が **A** で、**q**= **A · p** によってテクセル **p** を画面のピクセル **q** に結びつけます。**A** の列は、単に変換後の単位ベクトルです。π のように簡単です。」

### コンピュータの視点

<div class="cpt_fr" style="width:160px">

<img src="./img/affine/metr_texmapB.png" id="fig:comp_pov">  

**{*@fig:comp_pov}**: コンピュータから見た写像の過程。（テクスチャ空間での）**u** と **v** は **B** の列で、画面空間の主軸に写されます。

</div>

「こんにちは、Cearn の GBA です。ポケットに収まる、無駄のない凶悪なゲームマシンで、ピクセルを押し出すことにかけては誰にも負けません。まあ、持ち主の GeForce 4 Ti4200 は別かもしれませんが、あの見せびらかし屋め。それはともかく、私の仕事の 1 つがテクスチャマッピングです。それも普通のテクスチャマッピングではなく、回転や拡大縮小のようなかっこいいこともできます。私がするのはピクセルを埋めることです。私が知る必要があるのは、ピクセルの色をどこから取ってくればいいのか、それだけです。言い換えると、画面のピクセル **q** を埋めるには、**p = B · q** で正しいテクセル **p** を教えてくれる行列 **B** が必要です。どんな行列を渡されても喜んで使います。あなたが望む変換の行列をちゃんと用意してくれると、全面的に信頼していますから。」

### 結論

2 つの視点の決定的な違いに気づいたでしょうか。**A** はテクスチャ空間*から*画面空間*へ*写しますが、**B** はちょうどその逆です（つまり <math><mi>𝗕</mi><mo>=</mo><msup><mi>𝗔</mi><mn>-1</mn></msup></math>）。GBA にどちらを渡すべきか、もうわかりますね。そうです。**P = B** であって、**A** ではありません。この 1 つの情報こそが、アフィン行列のパズルを解く決定的なピースなのです。

ですから、**P** の要素は 2 通りの方法で求められます。人間の視点のまま考えて、最後に行列を逆行列にする方法。アフィン変換の逆行列も載せておいたのはこのためです。あるいは、GBA の見方で物事を捉えて、正しい行列を直接求める方法もあります。Tonc の主なアフィン関数（<i>tonc_video.h</i>、<i>tonc_obj_affine.c</i>、<i>tonc_bg_affine.c</i>）は GBA の方法で **P** を直接設定しますが、"`_inv`" という接尾辞の付いた逆方向の関数も用意しています。ただし、こちらは少し遅くなります。拡大縮小がからむ場合は別で、その場合は*ずっと*遅くなります。

気になる人のために書いておくと、(<math><msub><mi>s</mi><mi>x</mi></msub></math>, <math><msub><mi>s</mi><mi>y</mi></msub></math>) による拡大縮小と、α による反時計回りの回転の正しい行列は次のとおりです。

<math class="block">
    <mi>𝗔</mi>
    <mo>=</mo>
    <mi>𝗥</mi><mo>(</mo><mn>-&alpha;</mn><mo>)</mo>
    <mo>&middot;</mo>
    <mi>𝗦</mi><mo>(</mo><msub><mi>s</mi><mi>x</mi></msub><mo>,</mo><msub><mi>s</mi><mi>y</mi></msub><mo>)</mo>
</math>

<math class="block">
    <mtable>
        <mtr>
            <mtd>
                <mi>𝗣</mi>
            </mtd>
            <mtd columnalign="left">
                <mo>=</mo>
                <msup><mi>𝗔</mi><mn>-1</mn></msup>
            </mtd>
        </mtr>
        <mtr>
            <mtd />
            <mtd columnalign="left">
                <mo>=</mo>
                <mo>(</mo>
                <mi>𝗥</mi><mo>(</mo><mn>-&alpha;</mn><mo>)</mo>
                <mo>&middot;</mo>
                <mi>𝗦</mi><mo>(</mo><msub><mi>s</mi><mi>x</mi></msub><mo>,</mo><msub><mi>s</mi><mi>y</mi></msub><mo>)</mo>
                <msup><mo>)</mo><mn>-1</mn></msup>
            </mtd>
        </mtr>
        <mtr>
            <mtd />
            <mtd columnalign="left">
                <mo>=</mo>
                <msup><mi>𝗦</mi><mn>-1</mn></msup><mo>(</mo><msub><mi>s</mi><mi>x</mi></msub><mo>,</mo><msub><mi>s</mi><mi>y</mi></msub><mo>)</mo>
                <mo>&middot;</mo>
                <msup><mi>𝗥</mi><mn>-1</mn></msup><mo>(</mo><mn>-&alpha;</mn><mo>)</mo>
            </mtd>
        </mtr>
    </mtable>
</math>

先ほどの逆行列を使うと、次のようになります。

<math id="eq:correct_matrix" class="block">
    <mo>(</mo>
    <mi>{!@eq:correct_matrix}</mi>
    <mo>)</mo>
    <mspace width="30px" />
    <mi>𝗣</mi>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd><msub><mi>p</mi><mi>a</mi></msub></mtd>
                <mtd><msub><mi>p</mi><mi>b</mi></msub></mtd>
            </mtr>
            <mtr>
                <mtd><msub><mi>p</mi><mi>c</mi></msub></mtd>
                <mtd><msub><mi>p</mi><mi>d</mi></msub></mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
    <mo>=</mo>
    <mrow>
        <mo>[</mo>
        <mtable>
            <mtr>
                <mtd>
                    <mfrac>
                        <mrow><mi>cos</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mrow>
                        <msub><mi>s</mi><mi>x</mi></msub>
                    </mfrac>
                </mtd>
                <mtd>
                    <mfrac>
                        <mrow><mn>-</mn><mi>sin</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mrow>
                        <msub><mi>s</mi><mi>x</mi></msub>
                    </mfrac>
                </mtd>
            </mtr>
            <mtr>
                <mtd>
                    <mfrac>
                        <mrow><mi>sin</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mrow>
                        <msub><mi>s</mi><mi>y</mi></msub>
                    </mfrac>
                </mtd>
                <mtd>
                    <mfrac>
                        <mrow><mi>cos</mi><mo>(</mo><mn>&alpha;</mn><mo>)</mo></mrow>
                        <msub><mi>s</mi><mi>y</mi></msub>
                    </mfrac>
                </mtd>
            </mtr>
        </mtable>
        <mo>]</mo>
    </mrow>
</math>

<div class="note" markdown>
念のため、はっきりさせておきます。

アフィン行列 **P** は画面空間からテクスチャ空間*へ*写すのであって、その逆ではありません！

言い換えると:  

&nbsp;<math><msub><mi>p</mi><mi>a</mi></msub></math>: ピクセルあたりのテクスチャの *x* の増分

&nbsp;<math><msub><mi>p</mi><mi>b</mi></msub></math>: スキャンラインあたりのテクスチャの *x* の増分

&nbsp;<math><msub><mi>p</mi><mi>c</mi></msub></math>: ピクセルあたりのテクスチャの *y* の増分

&nbsp;<math><msub><mi>p</mi><mi>d</mi></msub></math>: スキャンラインあたりのテクスチャの *y* の増分
</div>

## 仕上げ {#sec-finish}

**P** 行列が何に使われるのかを知ることと、それを正しく使う方法を知ることは別の話です。アフィンのオブジェクトや背景、アフィン行列を扱うときに覚えておくべき点が、さらに 3 つあります。

1.  データ型
2.  ルックアップテーブル
3.  初期化

### アフィンの要素のデータ型

アフィン変換は数学の一部であり、一般的に言って、数学の数は実数です。つまり浮動小数点数です。しかし、**P** の要素に浮動小数点数を使おうとすると、2 つの手痛い驚きが待っています。

1 つ目は、行列の要素が浮動小数点数ではなく整数だということです。その理由は、<span class="ack">GBA には浮動小数点演算ユニットがないから</span>です！ 浮動小数点の演算はすべてソフトウェアで行わなければならず、FPU なしではかなり遅くなります。いずれにせよ、整数の演算よりはずっと遅いです。さて、こう考えると、精度などの問題がいくつか出てきます。たとえば、サイン・コサインの関数の値域は −1 から 1 で、整数にとってはあまり広いとは言えない範囲です。しかし、1 単位ではなく、たとえば 1/256 単位のような分数で数えれば、範囲はずっと広くなります。すると \[−1, +1\] の範囲は \[−256, +256\] になります。

実数をスケールした整数で表すこの方法は、<dfn>固定小数点演算</dfn>と呼ばれます。詳しくは[こちらの付録](fixed.html)や [Wikipedia](https://en.wikipedia.org/wiki/Fixed-point_arithmetic) を読んでください。GBA はアフィンパラメータに固定小数点を使いますが、ほかのことにも使えます。**P** 行列の要素は 8.8 の固定小数点数です。つまり、整数部 8 ビットと小数部 8 ビットのハーフワードです。行列を単位行列（対角成分が 1、それ以外が 0）にするには、次のようには書かず、

```c
    // Floating point == Bad!!
    pa= pd= 1.0;
    pb= pc= 0.0;
```

次のように書きます。

```c
    // .8 Fixed-point == Good
    pa= pd= 1<<8;
    pb= pc= 0;
```

小数部が *Q* ビットの固定小数点の体系では、「1」は <math><msup><mn>2</mn><mi>Q</mi></msup></math>、つまり 1<<*Q* で表されます。単純に、分数とはそういうものだからです。

さて、固定小数点数も結局はただの整数ですが、整数にもいろいろな型があり、正しい型を使うことが大事です。8.8f は 16 ビットの変数なので、`short` を選ぶのが自然です。ただし、これは*符号付き*の short、つまり `u16` ではなく `s16` にするべきです。どちらでもかまわない場合もありますが、算術演算をするなら符号付きにしたほうがよいでしょう。CPU は内部では 32 ビットのワード単位で動いていて、16 ビットの変数はそれに変換されることを思い出してください。たとえば 16 ビットの "−1"（`0xFFFF`）は 32 ビットの "−1"（`0xFFFFFFFF`）になってほしいはずで、符号なしの short を使うと起きてしまう "65535"（`0x0000FFFF`）にはなってほしくないはずです。また、固定小数点の計算をするときは、符号付きの int（32 ビットのもの）を使うことをおすすめします。それ以外の型を使うと遅くなりますし、オーバーフローの問題が起きるかもしれません。

  

:::tip アフィンの一時変数には 32 ビットの符号付き int を使おう

もちろん、どのみちすべてに 32 ビットの変数を使うべきです（コードを肥大させて遅くしたいのでない限り）。16 ビットの変数（`short` や `s16`）を使うと、変数を 16 ビットに保つための余分な命令が追加されるのでコードが遅くなるだけでなく、オーバーフローの問題もずっと早く起きる可能性があります。

8.8 形式にするのは、ハードウェアに渡す最後の段階だけにしましょう。それまでは、速度と精度の両方のために、大きな型を使ってください。

:::

### ルックアップテーブル {#sec-luts}

固定小数点演算を使うのは、浮動小数点演算が効率よく使うには遅すぎるからです。自分で書く計算ならそれで十分ですが、sin() や cos() のような数学関数はどうでしょうか？ これらは内部ではやはり浮動小数点（さらに悪いことに *`double`*！）なので、とんでもなく遅くなります。

関数を直接使う代わりに、コストの高い数学関数を使わずに済ませる昔ながらの方法を使います。サインとコサインの値を収めた<dfn>ルックアップテーブル</dfn>（LUT）を作るのです。作り方はいくつかあります。簡単な方法がよければ、360 個の 8.8f の数値の配列を 2 つ宣言して、プログラムの初期化時に埋めればよいでしょう。しかしこれは良くないやり方で、その理由は付録の [LUT の節](fixed.html#sec-lut)で説明しています。

Tonclib には、サインとコサインの両方に使えるサインのテーブルが 1 つだけあります。このテーブルは `sin_lut` という名前で、512 個の 4.12f（小数部 12 ビット）のエントリを持つ `const short` の配列です。私の LUT 作成ツール [excellut](http://www.coranac.com/projects/#excellut) で作ったものです。<i>tonc_math.h</i> には、サインとコサインの値を取り出す 2 つのインライン関数があります。

```c
//! Look-up a sine and cosine values
/*! \param theta Angle in [0,FFFFh] range
*   \return .12f sine value
*/

INLINE s32 lu_sin(uint theta)
{   return sin_lut[(theta>>7)&0x1FF];           }

INLINE s32 lu_cos(uint theta)
{   return sin_lut[((theta>>7)+128)&0x1FF];     }
```

ここで、角度の範囲に注意してください。0-10000h です。円を 360 度に分ける*必要は*ないことを思い出してください。実際、コンピュータでは円を 2 の累乗で分けたほうがよいのです。ここでは、BIOS の関数との互換性のために角度は 2<sup>16</sup> 分割になっていて、ルックアップ関数の中で 512 の範囲に落とされます。

### 初期化

背景やオブジェクトをアフィンに設定したら、`pa-pd` に少なくとも何らかの値を入れ*なければなりません*。これらはデフォルトでゼロになっていることを思い出してください。オフセットがゼロだと、全体に最初のピクセルが使われてしまいます。背景やスプライトが単色になってしまったら、おそらくこれが原因です。これを避けるには、**P** を単位行列か、ゼロでないほかの行列にしておきましょう。

## Tonc のアフィン関数

Tonclib には、`OBJ_AFFINE` 構造体と `BG_AFFINE` 構造体で使われる、オブジェクトや背景のアフィンパラメータを操作するための関数がいくつかあります。アフィン行列の格納のされ方が両者で違うので同じ関数では設定できませんが、機能は同じです。{@tbl:affine_functions} に基本的な形と説明を載せてあります。オブジェクトなら *foo* を `obj_aff` に、*FOO* を `OBJ` に、背景なら `bg_aff` と `BG` にそれぞれ置き換えてください。関数そのものは、オブジェクト用は <i>tonc_obj_affine.c</i>、背景用は <i>tonc_bg_affine.c</i> にあり、両方のインライン関数は `tonc_video.h` の…どこかにあります。

<table class="cblock table-data" id="tbl:affine_functions">
    <thead>
        <tr>
            <th>関数</th>
            <th>説明</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td>void <i>foo</i>_copy(FOO_AFFINE *dst, const FOO_AFFINE *src, uint count);</td>
            <td>アフィンパラメータをコピーする</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_identity(FOO_AFFINE *oaff);</td>
            <td><math><mi>P</mi><mo>=</mo><mi>I</mi></td>
        </tr>
        <tr>
            <td>void <i>foo</i>_postmul(FOO_AFFINE *dst, const FOO_AFFINE *src);</td>
            <td>Post-multiply: <math><mi>D</mi><mo>=</mo><mi>D</mi><mo>&middot;</mo><mi>S</mi></math></td>
        </tr>
        <tr>
            <td>void <i>foo</i>_premul(FOO_AFFINE *dst, const FOO_AFFINE *src);</td>
            <td>左から掛ける: <math><mi>D</mi><mo>=</mo><mi>S</mi><mo>&middot;</mo><mi>D</mi></math></td>
        </tr>
        <tr>
            <td>void <i>foo</i>_rotate(FOO_AFFINE *aff, u16 alpha);</td>
            <td>α·π/8000h だけ反時計回りに回転する。</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_rotscale(FOO_AFFINE *aff, FIXED sx, FIXED sy, u16 alpha);</td>
            <td><math><mfrac><mn>1</mn><msub><mi>s</mi><mi>x</mi></msub></mfrac></math> と <math><mfrac><mn>1</mn><msub><mi>s</mi><mi>y</mi></msub></mfrac></math> で拡大縮小してから、α·π/8000h だけ反時計回りに回転する。</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_rotscale2(FOO_AFFINE *aff, const AFF_SRC *as);</td>
            <td><code><i>foo</i>_rotscale()</code> と同じだが、入力は <code>AFF_SRC</code> 構造体に格納する。</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_scale(FOO_AFFINE *aff, FIXED sx, FIXED sy);</td>
            <td><math><mfrac><mn>1</mn><msub><mi>s</mi><mi>x</mi></msub></mfrac></math> と <math><mfrac><mn>1</mn><msub><mi>s</mi><mi>y</mi></msub></mfrac></math> で拡大縮小する</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_set(FOO_AFFINE *aff, FIXED pa, FIXED pb, FIXED pc, FIXED pd);</td>
            <td>P の要素を設定する</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_shearx(FOO_AFFINE *aff, FIXED hx);</td>
            <td>上辺を <math><msub><mi>h</mi><mi>x</mi></msub></math> だけ右にせん断する</td>
        </tr>
        <tr>
            <td>void <i>foo</i>_sheary(FOO_AFFINE *aff, FIXED hy);</td>
            <td>左辺を <math><msub><mi>h</mi><mi>y</mi></msub></math> だけ下にせん断する</td>
        </tr>
    </tbody>
</table>
**{*@tbl:affine_functions}**: アフィン関数

### 回転・拡大縮小関数の例

（{@eq:correct_matrix} のような）拡大縮小してから回転する関数の、オブジェクト版の私のコードを以下に示します。これはコンピュータの視点で書かれているので、`sx` と `sy` は縮小する方向に効くことに注意してください。また、角度 `alpha` は 1 周を 10000h とする単位を使っていて（つまり α の単位は π/8000h = 0.096 mrad、または 180/8000h = 0.0055°）、サインのテーブルは .12f 形式なので、12 ビットのシフトが必要です。背景版は、名前と型以外はまったく同じです。もしこれが C++ なら、テンプレートがとても役に立ったでしょう。

```c
void obj_aff_rotscale(OBJ_AFFINE *oaff, int sx, int sy, u16 alpha)
{
    int ss= lu_sin(alpha), cc= lu_cos(alpha);

    oaff->pa= cc*sx>>12;    oaff->pb=-ss*sx>>12;
    oaff->pc= ss*sy>>12;    oaff->pd= cc*sy>>12;
}
```

この章の内容で、アフィン行列について知っておくべきことのほとんどがわかったはずです。まず、なぜこれを単なる回転行列や rot/scale 行列など、よそで見かけるほかの名前ではなく、*アフィン*行列と呼ぶべきなのか。そして、この行列が実際に何をするのか、望む効果のための行列をどう組み立てるのか。固定小数点数とルックアップテーブルについても少しわかったでしょうし（詳しくは[付録](fixed.html)を見てください）、なぜそれらが良いものなのかもわかったはずです。それまで明らかでなかったなら、使うデータ型の選択が実際に*重要*であり、最初に思いついたものをそのまま使うべきではないことにも気づいたはずです。

ここで説明していないのは、オブジェクトや背景を実際にアフィン変換を使うように設定する方法です。それは次の 2 つの章で扱います。アフィン変換についてさらに知りたければ、「linear algebra（線形代数）」で検索してみてください。
