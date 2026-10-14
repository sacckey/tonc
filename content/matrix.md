# C. ベクトルと行列

<!-- toc -->

## ベクトル {#sec-vec}

ベクトルとは何かに入る前に、まずベクトルでないものについて話します。一般に、物理量は<dfn>スカラー</dfn>と<dfn>ベクトル</dfn>に分けられます。<dfn>スカラー</dfn>は量の大きさを与えます。毎日使っているのと同じ、1 つの数です。質量、エネルギー、体積はスカラーの例です。<dfn>ベクトル</dfn>は大きさ_と_向きの両方を持つもので、普通は複数の数、つまり次元ごとに 1 つの数で表されます。位置、運動量、力が代表的な例です。また、速度はベクトルですが、速さはそうではないことに注意してください。時速 50 km はベクトルではありません。60 号線を下る時速 50 km はベクトルです。ベクトルは太字で、普通は小文字で表し、**u** = (1, 4, 9) のようにかっこで囲んだ数の組か、M×1 の列として書きます。そして、そう、行ではなく列です。その理由は行列のところでわかります。

<table id="eq:vec-def">
  <tr>
    <td class="eqnrcell">({!@eq:vec-def})</td>
    <td>
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">u</mtext>
                  <mo>&#x2261;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mo>:</mo>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mi>m</mi>
                          </msub>
                        </mtd>
                      </mtr>
                    </mtable>
                    <mo data-mjx-texclass="CLOSE">]</mo>
                  </mrow>
                  <mo>&#x2261;</mo>
                  <mo stretchy="false">(</mo>
                  <msub>
                    <mi>u</mi>
                    <mn>1</mn>
                  </msub>
                  <mo>,</mo>
                  <mo>&#x22EF;</mo>
                  <mo>,</mo>
                  <msub>
                    <mi>u</mi>
                    <mi>m</mi>
                  </msub>
                  <mo stretchy="false">)</mo>
                  <mo>&#x2262;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                        <mtd>
                          <mo>&#x22EF;</mo>
                        </mtd>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mi>m</mi>
                          </msub>
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

<div class="cpt_fr" style="width:128px">
  <img src="img/math/crd_vec.png" id="fig:img-vec"
    alt="vectors and points"><br>
  <b>{*@fig:img-vec}</b>: ベクトルと点の
  違い。
</div>

座標系があれば、ベクトルは普通、その座標系の中の空間の点を表すのに使われ、ベクトルの要素が座標になります。しかし、点とベクトルの間には決定的な違いがあります。点は常に原点と関係していますが、ベクトルはどんな原点からも独立しています。右の {*@fig:img-vec} はこれを示しています。点 *P* と *Q*、ベクトル **u、v、w** があります。ベクトル **u** と **v** は等しい（長さも向きも同じ）です。しかし、**u** とそれが指す点（*P*）は同じ座標を持つのに対し、**v** と *Q* はそうではありません。実際、*Q* = **u** + **w** です。そして、さらに正確に言えば、*Q* = *O\* + **u** + **w** で、これは式の中で原点（_O_）を明示しています。

## ベクトルの演算 {#sec-vec-ops}

ベクトルの演算はスカラーの演算と似ていますが、多次元であることがいくつかの複雑さを加えます。特に掛け算の場合です。ベクトルの掛け算は_3 つ_もあるので、注意してください。右に、ベクトルの足し算とスカラーとベクトルの掛け算の例があります。**u**= (8, 3)、**v**= (-4 ,4) です。下に示す演算の定義を使えば、ほかのベクトルも求められるはずです。

### ベクトルどうしの足し算と引き算 {#ssec-vec-add}

<div class="cpt_fr" style="width:160px">
  <img src="img/math/crd_vec_ops.png" id="fig:img-vec-ops" alt="vector ops">
  <br>
  <b>{*@fig:img-vec-ops}</b>: ベクトルの足し算とスカラーとベクトルの掛け算。
</div>

足し算と引き算では、両方のオペランドが M 次元のベクトルでなければなりません。結果は同じく M 次元の別のベクトルで、その要素はオペランドの要素の和か差です。**w = u + v** なら _w_<sub>i</sub> = _u_<sub>i</sub> + _v_<sub>i</sub> です。

<table id="eq:vec-add">
  <tr>
    <td class="eqnrcell">({!@eq:vec-add})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">w</mtext>
                  <mo>=</mo>
                  <mtext mathvariant="bold">u + v</mtext>
                  <mo>&#x2261;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mn>1</mn>
                          </msub>
                          <mo>+</mo>
                          <msub>
                            <mi>v</mi>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mo>:</mo>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mi>m</mi>
                          </msub>
                          <mo>+</mo>
                          <msub>
                            <mi>v</mi>
                            <mi>m</mi>
                          </msub>
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
</table>

### スカラーとベクトルの掛け算 {#ssec-vec-scale}

これはベクトルの掛け算の 1 つ目です。スカラー _a_ とベクトル **u** があるとき、スカラーとベクトルの掛け算の結果のベクトルの要素は、元の要素にそれぞれスカラーを掛けたものです。ですから **v** = _c_&nbsp;**u** なら、_v_<sub>i</sub>= _c\*u_<sub>i</sub> です。**u** と **v** は同じ直線上にあり、長さだけが違うことに注意してください。また、引き算は **w** = **u** − **v** = **u** + (−1)\***v** とも書けることに注意してください。

<table id="eq:vec-scale">
  <tr>
    <td class="eqnrcell">({!@eq:vec-scale})</td>
    <td class="eqcell"> 
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">v</mtext>
                  <mo>=</mo>
                  <mi>c</mi>
                  <mtext mathvariant="bold">u</mtext>
                  <mo>&#x2261;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mi>c</mi>
                          <mo>&#x22C5;</mo>
                          <msub>
                            <mi>u</mi>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mo>:</mo>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mi>c</mi>
                          <mo>&#x22C5;</mo>
                          <msub>
                            <mi>u</mi>
                            <mi>m</mi>
                          </msub>
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

### 内積（別名スカラー積） {#ssec-vec-dot}

ベクトルの掛け算の 2 つ目は内積（ドット積）で、入力は 2 つのベクトルですが、出力は_スカラー_です。表記は _c_ = **u · v** で、**u** と **v** はベクトル、_c_ は結果のスカラーです。演算子が点（dot）の形をしていることに注意してください。これがこの種類の掛け算の名前の由来です。内積を求めるには、両方のベクトルの要素を対応するものどうしで掛けて、それらを全部足します。言い換えると次のとおりです。

<table id="eq:vec-dot">
  <tr>
    <td class="eqnrcell">({!@eq:vec-dot})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>c</mi>
                  <mo>=</mo>
                  <mtext mathvariant="bold">u</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">v</mtext>
                  <mo>=</mo>
                  <munderover>
                    <mo data-mjx-texclass="OP">&#x2211;</mo>
                    <mrow data-mjx-texclass="ORD"></mrow>
                    <mrow data-mjx-texclass="ORD"></mrow>
                  </munderover>
                  <msub>
                    <mi>u</mi>
                    <mi>i</mi>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <msub>
                    <mi>v</mi>
                    <mi>i</mi>
                  </msub>
                  <mo>=</mo>
                  <msub>
                    <mi>u</mi>
                    <mn>1</mn>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <msub>
                    <mi>v</mi>
                    <mn>1</mn>
                  </msub>
                  <mo>+</mo>
                  <mo>&#x22EF;</mo>
                  <mo>+</mo>
                  <msub>
                    <mi>u</mi>
                    <mi>m</mi>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <msub>
                    <mi>v</mi>
                    <mi>m</mi>
                  </msub>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

さて、これはばかげた演算に思えるかもしれませんが、実はとても便利です。1 つには、ベクトルの長さは自分自身との内積で計算します。また、内積を使えばあるベクトルの別のベクトルへの射影も求められます。これは、ベクトルをほかのベクトルで分解したり、M 次元の空間の基底ベクトルを決めたりしようとするときに欠かせません（何を何するって？！？ 心配いりません、あとで説明します）。内積の最もよくある使い道の 1 つは、2 つのベクトルの間の角度を求めることです。ベクトル **u** と **v** があり、\|**u**\| と \|**v**\| がその長さ、α が 2 つの間の角度なら、コサインは次の式で求められます。

<div class="cpt_fr" style="width:112px">
  <img src="img/math/crd_dot.png" id="fig:img-vec-dot" alt="dot-product">
  <br>
  <b>{*@fig:img-vec-dot}</b>: 内積。
</div>

<table id="eq:vec-cos">
  <tr>
    <td class="eqnrcell">({!@eq:vec-cos})</td>
    <td class="eqcell"> 
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>cos</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mrow>
                    <mo data-mjx-texclass="OPEN">(</mo>
                    <mi>&#x3B1;</mi>
                    <mo data-mjx-texclass="CLOSE">)</mo>
                  </mrow>
                  <mo>=</mo>
                  <mfrac>
                    <mrow>
                      <mtext mathvariant="bold">u</mtext>
                      <mo>&#x22C5;</mo>
                      <mtext mathvariant="bold">v</mtext>
                    </mrow>
                    <mrow>
                      <mtext mathvariant="bold">|u|</mtext>
                      <mtext>&#xA0;</mtext>
                      <mtext mathvariant="bold">|v|</mtext>
                    </mrow>
                  </mfrac>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

なぜこれでうまくいくのでしょうか？ いくつかの方法で証明できますが、これが一番エレガントなものです（思い出させてくれた Ash に感謝）。ベクトルの長さの 2 乗は自分自身との内積で与えられることを思い出してください。つまり、\|**v−u**\|<sup>2</sup> = \|**v**\|<sup>2</sup> + \|**u**\|<sup>2</sup> − 2·**u·v** です。{@fig:img-vec-dot} の三角形の余弦定理からも、\|**v−u**\|<sup>2</sup> = \|**v**\|<sup>2</sup> + \|**u**\|<sup>2</sup> − 2·\|**v**\|·\|**u**\| cos(α) が得られます。これらの関係を組み合わせると、すぐに {@eq:vec-cos} になります。それなのに、数学は難しいと言う人がいるのです。

ちなみに、これで角度を求められるだけでなく、何かが自分の後ろにあるかどうかをとても簡単に確かめる方法にもなります。**u** が見ている方向で、**v** が物体へのベクトルなら、角度が 90° を超えると **u · v** は負になります。視野の確認や、ベクトルが垂直かどうか（**u · v** = 0）を確かめるのにも便利です。物理学でも、力の分解や、位置エネルギーを求めるための力の経路積分などで、内積がどっさり出てきます。基本的に、物理学の式でコサインを見かけたら、それはおそらく内積の結果です。

### 外積（別名ベクトル積） {#ssec-vec-cross}

外積（クロス積）は、3D 空間でしか使えない特別な種類の積です。外積は 2 つのベクトル **u** と **v** を取り、結果として両方に垂直なベクトル **w** を与えます。**w** の長さは、2 つのオペランドのベクトルが張る面積です。表記は **w** = **u** × **v** で、これがクロス（×）積と呼ばれる理由です。**w** の要素は _w_<sub>i</sub>= _ε_<sub>ijk</sub>·_u_<sub>j</sub>· _v_<sub>k</sub> で、ここで _ε_<sub>ijk</sub> はレヴィ＝チヴィタ記号（_i,j,k_ の偶置換なら +1、奇置換なら −1、添字のどれかが等しければ 0）です。おそらくこれを見たことすらないでしょう（正気を保つために、そのままにしておいてください）から、{@eq:vec-dot} に完全な形で書いておきました（訳注: 参照先は外積の式のほうと思われます）。

<div class="cpt_fr" style="width:176px">
  <img src="img/math/crd_cross.png" id="fig:img-vec-cross" alt="cross product">
  <br>
  <b>{*@fig:img-vec-cross}</b>: 外積。
</div>

<table id="eq:vec-cross">
  <tr>
    <td class="eqnrcell">({!@eq:vec-cross})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">w</mtext>
                  <mo>=</mo>
                  <mtext mathvariant="bold">u</mtext>
                  <mtext>&#xA0;</mtext>
                  <mo>&#xD7;</mo>
                  <mtext>&#xA0;</mtext>
                  <mtext mathvariant="bold">v</mtext>
                  <mo>&#x2261;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mi>y</mi>
                          </msub>
                          <msub>
                            <mi>v</mi>
                            <mi>z</mi>
                          </msub>
                          <mo>&#x2212;</mo>
                          <msub>
                            <mi>u</mi>
                            <mi>z</mi>
                          </msub>
                          <msub>
                            <mi>v</mi>
                            <mi>y</mi>
                          </msub>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mi>z</mi>
                          </msub>
                          <msub>
                            <mi>v</mi>
                            <mi>x</mi>
                          </msub>
                          <mo>&#x2212;</mo>
                          <msub>
                            <mi>u</mi>
                            <mi>x</mi>
                          </msub>
                          <msub>
                            <mi>v</mi>
                            <mi>z</mi>
                          </msub>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <msub>
                            <mi>u</mi>
                            <mi>x</mi>
                          </msub>
                          <msub>
                            <mi>v</mi>
                            <mi>y</mi>
                          </msub>
                          <mo>&#x2212;</mo>
                          <msub>
                            <mi>u</mi>
                            <mi>y</mi>
                          </msub>
                          <msub>
                            <mi>v</mi>
                            <mi>x</mi>
                          </msub>
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
</table>

{@fig:img-vec-cross} に、外積が何をするかの絵があります。3D の絵なので、少し想像力を働かせる必要があります。ベクトル **u** と **v** が平行四辺形（黄色）を定めます。外積のベクトル **w** はこれらの両方に垂直で、これは **u·w** と **v·w** から導かれる事実です（訳注: どちらも 0 になることから、という意味です）。**w** の長さはこの平行四辺形の面積 _A_ で、面積の計算を覚えていれば、次のようになることがわかるでしょう。

<table id="eq:vec-sin">
  <tr>
    <td class="eqnrcell">({!@eq:vec-sin})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>A</mi>
                  <mo>=</mo>
                  <mo stretchy="false">|</mo>
                  <mtext mathvariant="bold">u</mtext>
                  <mtext>&#xA0;</mtext>
                  <mo>&#xD7;</mo>
                  <mtext>&#xA0;</mtext>
                  <mtext mathvariant="bold">v</mtext>
                  <mo stretchy="false">|</mo>
                  <mo>=</mo>
                  <mtext mathvariant="bold">&#xA0;|u|</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">|v|</mtext>
                  <mo>&#x22C5;</mo>
                  <mi>sin</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mrow>
                    <mo data-mjx-texclass="OPEN">(</mo>
                    <mi>&#x3B1;</mi>
                    <mo data-mjx-texclass="CLOSE">)</mo>
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

つまり、外積を使えば 2 つのベクトルの間の角度のサインを求められます。外積は_反_交換的であることに注意してください！ つまり **u × v** = −**v × u** です。マイナスの符号に気づきましたか？ これは実は良い点を提起します。**u** と **v** で定まる平面について、この平面の法線ベクトルは上を向いていますが、「上」がどちらかはどうやって決めるのでしょうか？ 私がたいていするのは、普通の 3D の座標系（{@fig:img-vec-cross} の右下の部分にあるようなもの）を取り、_x_ 軸を **u** に合わせ、_y_ 軸が **v** に沿う（か、一番近くなる）まで回転させることです。すると **w** は _z_ 軸に沿います。{\*@eq:vec-cross} では、これがすべてすでに整理されています。ただし、これには右手系が必要です。左手系だと頭がひどく混乱するのです。

さて、ベクトルが平行なら、**u** × **v** = 0、つまり **w** はゼロベクトル **0** です。これはまた、**u** が見ている方向なら、ベクトル **v** の物体が照準のど真ん中にあることも意味します。しかし、**u** がロケットの速度で、**v** が_あなた_への相対ベクトルなら、リスポーンの準備をしてください。基本的に、内積は物体が前にあるか後ろにあるか（接線方向）を教えてくれるのに対し、外積は中心からのずれ（法線方向）を与えてくれます。赤こうらのようなものを実装したいときにとても便利です（ここで言っているのは_初代_スーパーマリオカートの赤こうらで、後のマリオカートの弱虫の即追尾こうらではありません。ブーブー！！）。外積は物理学でも、角運動量（**L = r × p**）や磁気誘導などで豊富に出てきます。

以上が 3D の場合でしたが、外積は 2D でも便利です。**w** の _z_ 成分しか必要ないことを除けば、すべてまったく同じように働きます。

### ノルム（長さ） {#ssec-vec-norm}

これまで何度か使ってきましたが、ベクトルの長さが何なのかを実際には定義していませんでした。ベクトル **u** のノルムは、自分自身との内積の平方根として定義されます。{@eq:vec-norm} を見てください。昔ながらのピタゴラスの定理は、その 2D の特殊な場合にすぎません。

ベクトルの長さ、つまりノルムは、手元にあると便利なものです。実際、長さから始めて、サインとコサインを使ってベクトルを _x_ と _y_ の成分に分解することがよくあります。その良い例が速さです。長さが役割を果たすもう 1 つのことは、長さ 1 の<dfn>単位ベクトル</dfn>を作ることです。多くの計算は何らかの形で長さを必要としますが、それが 1 なら、もう気にする必要はありません。正規化したベクトルを作るには、単にその長さで割ります。**û** = **u** / \|**u**| です。

<table id="eq:vec-norm">
  <tr>
    <td class="eqnrcell">({!@eq:vec-norm})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mo stretchy="false">|</mo>
                  <mtext mathvariant="bold">u</mtext>
                  <mo stretchy="false">|</mo>
                  <mo>=</mo>
                  <msqrt>
                    <mo stretchy="false">(</mo>
                    <mtext mathvariant="bold">u</mtext>
                    <mo>&#x22C5;</mo>
                    <mtext mathvariant="bold">u</mtext>
                    <mo stretchy="false">)</mo>
                  </msqrt>
                  <mo>=</mo>
                  <mo stretchy="false">(</mo>
                  <munderover>
                    <mo data-mjx-texclass="OP">&#x2211;</mo>
                    <mrow data-mjx-texclass="ORD"></mrow>
                    <mrow data-mjx-texclass="ORD"></mrow>
                  </munderover>
                  <msubsup>
                    <mi>u</mi>
                    <mi>i</mi>
                    <mn>2</mn>
                  </msubsup>
                  <msup>
                    <mo stretchy="false">)</mo>
                    <mrow data-mjx-texclass="ORD">
                      <mfrac>
                        <mn>1</mn>
                        <mn>2</mn>
                      </mfrac>
                    </mrow>
                  </msup>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

### ベクトルの代数的な性質 {#ssec-vec-props}

以下は、ベクトルの代数的な性質の一覧です。ほとんどは明らかに思えるでしょうが、少なくとも一度は見ておく必要があります。私の線形代数の教科書から直接取ってきました。**u、v、w** を M 次元のベクトル、_c_ と _d_ をスカラーとすると、次のようになります（訳注: 以下の表には原文の誤記と思われる箇所がいくつかあります。たとえば (c + d)·u = c·u + d·u、u · (v + w) = u·v + u·w、u × v = −(v × u) が正しい形です）。

<div class="lblock">
  <table>
    <tr>
      <td><b>u + v = v + u</b></td>
      <td>交換法則</td>
    </tr>
    <tr>
      <td><b>(u + v) + w = u + (v + w)</b></td>
      <td>結合法則</td>
    </tr>
    <tr>
      <td><b>u + 0 = 0 + u = u</b></td>
    </tr>
    <tr>
      <td><b>u + (&minus;u) = &minus;u + u = 0</b></td>
      <td>ここで &minus;<b>u</b> は (&minus;1)<b>u</b> を表す</td>
    </tr>
    <tr>
      <td>c·(<b>u + v</b>) = c·<b>u</b> + c·<b>v</b></td>
      <td>分配法則</td>
    </tr>
    <tr>
      <td>(c + d)·<b>u</b> = c·<b>u</b> d·<b>u</b></td>
      <td>分配法則</td>
    </tr>
    <tr>
      <td>c·(d·<b>u</b>) = (c·d)·<b>u</b></td>
      <td>結合法則</td>
    </tr>
    <tr>
      <td>1·<b>u</b> = <b>u</b>
    </tr>
  </table>
</div>

そして積については次のとおりです。

<div class="lblock">
  <table>
    <tr>
      <td width="256"><b>u · (v + w) = (u + v) · w</b></td>
      <td></td>
    </tr>
    <tr>
      <td><b>u ·</b> (c·<b>v</b>) = (c·<b>u</b>)<b> · v</b> = c·(<b>u · v</b>)
      <td></td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td></td>
    </tr>
    <tr>
      <td><b>u &times; v</b> = &minus;(<b>u &times; v</b>)</td>
      <td>反交換法則</td>
    </tr>
    <tr>
      <td><b>u</b> &times; (<b>v + w</b>) = <b>u &times; v + u &times; w</b></td>
      <td></td>
    </tr>
    <tr>
      <td>(<b>u + v</b>) &times; <b>w</b> = <b>u &times; w + v &times; w</b></td>
      <td></td>
    </tr>
    <tr>
      <td><b>u</b> &times; (c·<b>v</b>) = c·<b>u &times; v</b> = (c·<b>u</b>) &times; <b>v</b>
      <td></td>
    </tr>
    <tr>
      <td>&nbsp;</td>
      <td></td>
    </tr>
    <tr>
      <td><b>u ·</b> (<b>u &times; v</b>) = 0</td>
      <td></td>
    </tr>
    <tr>
      <td><b>u ·</b> (<b>v &times; w</b>) = (<b>u &times; v</b>) <b>· w</b></td>
      <td>スカラー三重積。<b>u、v、w</b> で定まる平行六面体の体積を与えます。</td>
    </tr>
    <tr>
      <td><b>u</b> &times; (<b>v &times; w</b>) = <b>u</b>(<b>v · w</b>) &minus; <b>w</b>(<b>u · v</b>)</td>
      <td>ベクトル三重積</td>
    </tr>
  </table>
</div>

## 行列 {#sec-mat}

一言で言えば、行列は数の 2 次元の格子です。行列は当初、連立 1 次方程式を解くための略記として使われていました。たとえば、変数 _x, y, z_ を使った連立方程式

<table id="eq:eqset">
  <tr>
    <td class="eqnrcell">({!@eq:eqset}a)</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mi>x</mi>
                </mtd>
                <mtd>
                  <mo>&#x2212;</mo>
                </mtd>
                <mtd>
                  <mn>2</mn>
                  <mi>y</mi>
                </mtd>
                <mtd>
                  <mo>+</mo>
                </mtd>
                <mtd>
                  <mi>z</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mn>0</mn>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd></mtd>
                <mtd></mtd>
                <mtd>
                  <mn>2</mn>
                  <mi>y</mi>
                </mtd>
                <mtd>
                  <mo>&#x2212;</mo>
                </mtd>
                <mtd>
                  <mn>8</mn>
                  <mi>z</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mn>8</mn>
                </mtd>
              </mtr>
              <mtr>
                <mtd></mtd>
                <mtd>
                  <mo>&#x2212;</mo>
                  <mn>4</mn>
                  <mi>x</mi>
                </mtd>
                <mtd>
                  <mo>+</mo>
                </mtd>
                <mtd>
                  <mn>5</mn>
                  <mi>y</mi>
                </mtd>
                <mtd>
                  <mo>+</mo>
                </mtd>
                <mtd>
                  <mn>9</mn>
                  <mi>z</mi>
                </mtd>
                <mtd>
                  <mo>=</mo>
                </mtd>
                <mtd>
                  <mo>&#x2212;</mo>
                  <mn>9</mn>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

は、行列を使うともっと簡潔に次のように書けます。

<table>
  <tr>
    <td>
      <table>
        <tr>
          <td class="eqnrcell">({!@eq:eqset}b)</td>
          <td class="eqcell">
            <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
              <mstyle displaystyle="true" scriptlevel="0">
                <mrow data-mjx-texclass="ORD">
                  <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                    <mtr>
                      <mtd>
                        <mrow data-mjx-texclass="INNER">
                          <mo data-mjx-texclass="OPEN">[</mo>
                          <mtable columnspacing="1em" rowspacing="4pt">
                            <mtr>
                              <mtd>
                                <mn>1</mn>
                              </mtd>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>2</mn>
                              </mtd>
                              <mtd>
                                <mn>1</mn>
                              </mtd>
                            </mtr>
                            <mtr>
                              <mtd>
                                <mn>0</mn>
                              </mtd>
                              <mtd>
                                <mn>2</mn>
                              </mtd>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>8</mn>
                              </mtd>
                            </mtr>
                            <mtr>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>4</mn>
                              </mtd>
                              <mtd>
                                <mn>5</mn>
                              </mtd>
                              <mtd>
                                <mn>9</mn>
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
    </td>
    <td width=64 align="center">または</td>
    <td>
      <table>
        <tr>
          <td class="eqnrcell">({!@eq:eqset}c)</td>
          <td class="eqcell">
            <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
              <mstyle displaystyle="true" scriptlevel="0">
                <mrow data-mjx-texclass="ORD">
                  <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
                    <mtr>
                      <mtd>
                        <mrow data-mjx-texclass="INNER">
                          <mo data-mjx-texclass="OPEN">[</mo>
                          <mtable columnspacing="1em" rowspacing="4pt">
                            <mtr>
                              <mtd>
                                <mn>1</mn>
                              </mtd>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>2</mn>
                              </mtd>
                              <mtd>
                                <mn>1</mn>
                              </mtd>
                              <mtd>
                                <mn>0</mn>
                              </mtd>
                            </mtr>
                            <mtr>
                              <mtd>
                                <mn>0</mn>
                              </mtd>
                              <mtd>
                                <mn>2</mn>
                              </mtd>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>8</mn>
                              </mtd>
                              <mtd>
                                <mn>8</mn>
                              </mtd>
                            </mtr>
                            <mtr>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>4</mn>
                              </mtd>
                              <mtd>
                                <mn>5</mn>
                              </mtd>
                              <mtd>
                                <mn>9</mn>
                              </mtd>
                              <mtd>
                                <mo>&#x2212;</mo>
                                <mn>9</mn>
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
    </td>
  </tr>
</table>

{\*@eq:eqset}b は<dfn>係数行列</dfn>と呼ばれ、変数の係数だけが書かれています。<dfn>拡大係数行列</dfn>（{@eq:eqset}c）には、連立方程式の右辺も含まれます。変数そのものはどこにも見当たらないことに注意してください。それがだいたいの要点なのです。数学者は世界一の怠け者で、利用できる略記があれば使いますし、なければでっち上げます。

ともかく、行列は横に走る<dfn>行</dfn>と、縦に走る<dfn>列</dfn>に分けられます。行列は大きさで表されます。M×N の行列には M 個の行と N 個の列があります。行の数が先に来ることに注意してください。これは、普通は幅が先に来る画像の大きさとは対照的です。ええ、ひどいのはわかっていますが、私にはどうしようもありません。{@eq:eqset}b の係数行列は 3x3 の行列で、{@eq:eqset}c の拡大係数行列は 3x4 です。行列全体は普通、太字の大文字で表します。行列の列は単なるベクトルで（ベクトルは M×1 の列だったことを覚えていますか？）、そのように表し、列の番号を表す添字を 1 つ付けます。行列の<dfn>要素</dfn>は、小文字の（イタリックの）文字に添字を 2 つ付けて表します。

<table id="eq:mat-def">
  <tr>
    <td class="eqnrcell">({!@eq:mat-def})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">A</mtext>
                  <mo>=</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">a</mtext>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                        <mtd>
                          <mo>&#x22EF;</mo>
                        </mtd>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">a</mtext>
                            <mi>n</mi>
                          </msub>
                        </mtd>
                      </mtr>
                    </mtable>
                    <mo data-mjx-texclass="CLOSE">]</mo>
                  </mrow>
                  <mo>&#x2261;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">a</mtext>
                            <mrow data-mjx-texclass="ORD">
                              <mn>11</mn>
                            </mrow>
                          </msub>
                        </mtd>
                        <mtd>
                          <mo>&#x22EF;</mo>
                        </mtd>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">a</mtext>
                            <mrow data-mjx-texclass="ORD">
                              <mn>1</mn>
                              <mi>n</mi>
                            </mrow>
                          </msub>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mrow data-mjx-texclass="ORD">
                            <mo>&#x22EE;</mo>
                          </mrow>
                        </mtd>
                        <mtd></mtd>
                        <mtd>
                          <mrow data-mjx-texclass="ORD">
                            <mo>&#x22EE;</mo>
                          </mrow>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">a</mtext>
                            <mrow data-mjx-texclass="ORD">
                              <mi>m</mi>
                              <mn>1</mn>
                            </mrow>
                          </msub>
                        </mtd>
                        <mtd>
                          <mo>&#x22EF;</mo>
                        </mtd>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">a</mtext>
                            <mrow data-mjx-texclass="ORD">
                              <mi>m</mi>
                              <mi>n</mi>
                            </mrow>
                          </msub>
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

ほとんどのコンピュータの言語にも行列の概念がありますが、要素の並べ方については必ずしも一致していません。たとえば Visual Basic と C のインデックスは、{@eq:mat-def} と同じく行に基づいています。一方 Fortran はベクトルに基づいているので、インデックスを逆にする必要があります。C のポインタ型のおかげで、行列に配列としてアクセスすることもできます。

```c {.proglist}
mat(i, j)     // VB matrix

mat[i][j]    // C matrix
mat[i+N*j]   // C matrix, in array form

mat(j, i)    // Fortran matrix
```

しばらく ({@eq:eqset}) に戻りましょう。**x** = (_x, y, z_)、**b**= (0, 8, −9)、係数行列に **A** を使えば、({@eq:eqset}a) は次のように書き直せます。

<table>
  <tr>
    <td class="eqnrcell">({!@eq:eqset}d)</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <msub>
                    <mi>a</mi>
                    <mn>1</mn>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <mi>x</mi>
                  <mtext>&#xA0;</mtext>
                  <mo>+</mo>
                  <mtext>&#xA0;</mtext>
                  <msub>
                    <mi>a</mi>
                    <mn>2</mn>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <mi>y</mi>
                  <mtext>&#xA0;</mtext>
                  <mo>+</mo>
                  <mtext>&#xA0;</mtext>
                  <msub>
                    <mi>a</mi>
                    <mn>3</mn>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <mi>z</mi>
                  <mo>=</mo>
                  <mtext mathvariant="bold">b</mtext>
                  <mo>=</mo>
                  <mtext mathvariant="bold">A</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">x</mtext>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

**b** の左側では列ベクトルの表記を、右側では完全な行列の表記を使いました。この形の式は、あとでも出てくるので覚えておくとよいでしょう。そして、ええ、右辺にあるのは行列の掛け算です。まだきちんと定義していませんが、これでいくらかヒントになるはずです。

## 行列の演算 {#sec-mat-ops}

### 転置 {#ssec-mat-transpose}

行列を転置するとは、対角線で鏡映することです。ときどき手元にあると便利なものです。転置の表記は大文字の「T」の上付き文字で、たとえば **B** = **A**<sup>T</sup> です。**A** が M×N の行列なら、その転置 **B** は N×M で、要素は _b_<sub>ij</sub> = _a_<sub>ji</sub> です。先ほど言ったように、対角線で鏡映するのです。もちろん、対角線そのものは変わりません。

### 行列の足し算 {#ssec-mat-add}

行列の足し算はベクトルの足し算とほぼ同じですが、2 次元です。**A、B、C** がすべて M×N の行列で **C = A + B** なら、**C** の要素は _c_<sub>ij</sub> = _a_<sub>ij</sub> + _b_<sub>ij</sub> です。もちろん、引き算も同じです。

### 行列の掛け算 {#ssec-mat-mul}

ああ、ここから面白くなってきます。行列の掛け算にはいくつかの規則があり、かなりやっかいです。掛け算には **C = A · B** を使います。問題は、最初のオペランド（**A**）の列の数が、2 つ目（**B**）の行の数と等しく_なければならない_ことです。ですから、**A** が _p×q_ の行列なら、**B** は _q×r_ の行列であるべきです。そのとき **C** の大きさは _p×r_ になります。さて、**C** の要素は次の式で与えられます。

<table id="eq:mat-mul-elem">
  <tr>
    <td class="eqnrcell">({!@eq:mat-mul-elem})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <msub>
                    <mi>c</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>i</mi>
                      <mi>j</mi>
                    </mrow>
                  </msub>
                  <mo>&#x2261;</mo>
                  <munderover>
                    <mo data-mjx-texclass="OP">&#x2211;</mo>
                    <mrow data-mjx-texclass="ORD">
                      <mi>k</mi>
                    </mrow>
                    <mrow data-mjx-texclass="ORD"></mrow>
                  </munderover>
                  <msub>
                    <mi>a</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>i</mi>
                      <mi>k</mi>
                    </mrow>
                  </msub>
                  <mo>&#x22C5;</mo>
                  <msub>
                    <mi>b</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mi>k</mi>
                      <mi>j</mi>
                    </mrow>
                  </msub>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

言い換えると、**A** の行 _i_ と **B** の列 _j_ を取り、それらの内積を取るのです。{@eq:mat-mul-elem} の _k_ はこの内積の和の添字です。**A** の列と **B** の行の大きさが等しくなければならないのもこのためです。そうでないと、どちらかのベクトルの端が余ってしまいます。別の見方もあります。**A** 全体は、{@eq:eqset}b のような連立 1 次方程式の係数行列になります。**B** の列はどれも変数のベクトルで、連立 1 次方程式で処理すると **C** の列になります。

<table id="eq:mat-mul">
  <tr>
    <td class="eqnrcell">({!@eq:mat-mul})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">C</mtext>
                  <mo>=</mo>
                  <mtext mathvariant="bold">A</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">B</mtext>
                  <mo>&#x2261;</mo>
                  <mtext mathvariant="bold">A</mtext>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">b</mtext>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                        <mtd>
                          <mo>&#x22EF;</mo>
                        </mtd>
                        <mtd>
                          <msub>
                            <mtext mathvariant="bold">b</mtext>
                            <mi>r</mi>
                          </msub>
                        </mtd>
                      </mtr>
                    </mtable>
                    <mo data-mjx-texclass="CLOSE">]</mo>
                  </mrow>
                  <mo>&#x2261;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mtext mathvariant="bold">A</mtext>
                          <mo>&#x22C5;</mo>
                          <msub>
                            <mtext mathvariant="bold">b</mtext>
                            <mn>1</mn>
                          </msub>
                        </mtd>
                        <mtd>
                          <mo>&#x22EF;</mo>
                        </mtd>
                        <mtd>
                          <mtext mathvariant="bold">A</mtext>
                          <mo>&#x22C5;</mo>
                          <msub>
                            <mtext mathvariant="bold">b</mtext>
                            <mi>r</mi>
                          </msub>
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
</table>

この見方の価値は、座標変換を説明するときに明らかになります。また、先ほど言ったように、行列の掛け算では **A** の行と **B** の列の内積を取ります。ベクトルは基本的に M×1 の行列なので、普通の内積は実は行列の掛け算の特殊な場合です。ただ、最初のベクトルの転置を取る必要があります。

<table id="eq:vec-mul">
  <tr>
    <td class="eqnrcell">({!@eq:vec-mul})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>c</mi>
                  <mo>=</mo>
                  <mtext mathvariant="bold">u</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">v</mtext>
                  <mo>=</mo>
                  <msup>
                    <mrow data-mjx-texclass="INNER">
                      <mo data-mjx-texclass="OPEN">[</mo>
                      <mtable columnspacing="1em" rowspacing="4pt">
                        <mtr>
                          <mtd>
                            <mtext mathvariant="bold">u</mtext>
                          </mtd>
                        </mtr>
                      </mtable>
                      <mo data-mjx-texclass="CLOSE">]</mo>
                    </mrow>
                    <mrow data-mjx-texclass="ORD">
                      <mi>T</mi>
                    </mrow>
                  </msup>
                  <mo>&#x22C5;</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mtext mathvariant="bold">v</mtext>
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
</table>

行列の掛け算でできることはほかにも山ほどありますが、次の 2 つの注意だけにしておきます。まず、この演算は_交換可能ではありません_！ つまり **A · B** ≠ **B · A** です。行と列の条件から察していたかもしれませんが、それらが合っていても、やはり交換可能ではありません。私の[アフィンスプライトのデモ](affobj.html#sec-demo)がこれをいわば示しています。回転してから拡大縮小するのと、拡大縮小してから回転する（おそらく望んでいたのはこちら）のとでは、同じ結果になりません。**A · B** が **B · A** と等しくなるのは、ごく特別な場合だけです。

もう 1 つの注意は、行列の掛け算は高くつくということです。**C** の各要素について内積（_q_ 回の掛け算）をしなければならないので、_p\*q\*r_ 回の掛け算になります。これは O(3)（訳注: 通常は O(N³) と書きます）の演算で、最も厄介な部類です。まあ、2x2 の行列ならたいしたことはありませんが、（私が仕事でするように）27x18 の行列を扱うとなると問題になります。幸い計算の数を減らす方法はありますが、それはこのチュートリアルの範囲を超えています。

### 行列式 {#ssec-mat-det}

行列式は、<dfn>正方行列</dfn>（N×N の大きさ）の要素を特定の方法で組み合わせて得られるスカラーです。行列式の素敵ではっきりした定義をあちこち探しましたが、ほとんど見つかりませんでした。いくつかの使い道があるようですが、最もよく使われるのは、連立方程式の式（あるいはベクトルの組）が 1 次独立かどうか、したがって係数行列が逆行列を持つかどうかを確かめる簡単な確認としてです。N×N の行列 **A** の行列式の数学的な定義は漸化式で、次のようになります。

<table id="eq:mat-det">
  <tr>
    <td class="eqnrcell">({!@eq:mat-det})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>det</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <mtext mathvariant="bold">A</mtext>
                  <mo>=</mo>
                  <mo stretchy="false">|</mo>
                  <mtext mathvariant="bold">A</mtext>
                  <mo stretchy="false">|</mo>
                  <mo>=</mo>
                  <munderover>
                    <mo data-mjx-texclass="OP">&#x2211;</mo>
                    <mrow data-mjx-texclass="ORD">
                      <mi>j</mi>
                    </mrow>
                    <mrow data-mjx-texclass="ORD"></mrow>
                  </munderover>
                  <mo stretchy="false">(</mo>
                  <mo>&#x2212;</mo>
                  <mn>1</mn>
                  <msup>
                    <mo stretchy="false">)</mo>
                    <mrow data-mjx-texclass="ORD">
                      <mn>1</mn>
                      <mo>+</mo>
                      <mi>j</mi>
                    </mrow>
                  </msup>
                  <mtext>&#xA0;</mtext>
                  <msub>
                    <mi>a</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mn>1</mn>
                      <mi>j</mi>
                    </mrow>
                  </msub>
                  <mtext>&#xA0;</mtext>
                  <mi>det</mi>
                  <mo data-mjx-texclass="NONE">&#x2061;</mo>
                  <msub>
                    <mi>A</mi>
                    <mrow data-mjx-texclass="ORD">
                      <mn>1</mn>
                      <mi>j</mi>
                    </mrow>
                  </msub>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

これをもっと詳しく説明することもできますが、実はそうしてもあまり意味がありません。2x2 と 3x3 の場合の公式だけを示します。実は、すでに示しています。外積のところでです。行列 **A** = \[**a**<sub>1</sub> **a**<sub>2</sub> **a**<sub>3</sub>\] があれば、\|**A**\| = **a**<sub>1</sub> · (**a**<sub>2</sub> × **a**<sub>3</sub>) です。2×2 の行列 **B** = \[**b**<sub>1</sub> **b**<sub>2</sub>\] なら、それは _b_<sub>11</sub>·_b_<sub>22</sub> − _b_<sub>12</sub>·_b_<sub>21</sub> で、これも実は外積を使っています。これは単なる偶然ではありません。行列式の使い道の一部は、行列が逆行列を持てるかどうかを判定することです。基本的に、\|**A**\| = 0 なら逆行列はありません。さて、外積はベクトルの間の面積の計算に関わっていることを思い出してください。これが 0 になりうるのは、ベクトルが同一直線上にある場合だけです。そして 1 次独立性は、逆行列を持つための重要な条件の 1 つです。また、行列式の表記 det **A** = \|**A**\| にも注目してください。ベクトルのノルムに少し似ていますよね？ まあ、関係のある外積はベクトルの間に張られる面積に関係して_いる_ので、筋は通っていると思います。

### 逆行列 {#ssec-mat-inv}

（またしても）{@eq:eqset} に戻ると、**A · x** = **b** となるような、変数 **x**= (_x, y, z_) と行列 **A** の連立方程式があります。それは結構なのですが、たいていわからないのは **b** ではなく **x** のほうです。必要なのは **x** から **b** への道（それが **A** です）ではなく、その<dfn>逆</dfn>です。必要なのは **x** = **A**<sup>−1</sup> **· b** です。**A**<sup>−1</sup> は行列の逆行列の表記です。その基本の定義は **A · A**<sup>−1</sup> = **I** で、**I** は<dfn>単位行列</dfn>、つまり対角成分が 1 でそれ以外がすべて 0 の行列です。逆行列を計算する方法はいくつかあります。もちろん試行錯誤もあります（考えることすらしないでください！）が、連立 1 次方程式を解く普通のやり方、行基本変形によるものもあります。そのやり方はまだ説明していないので、1 つの場合、つまり 2x2 の場合の公式を示すだけにします。

<table id="eq:2d-inv">
  <tr>
    <td class="eqnrcell">({!@eq:2d-inv})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mtext mathvariant="bold">A</mtext>
                  <mo>=</mo>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mi>a</mi>
                        </mtd>
                        <mtd>
                          <mi>b</mi>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mi>c</mi>
                        </mtd>
                        <mtd>
                          <mi>d</mi>
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
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <msup>
                    <mtext mathvariant="bold">A</mtext>
                    <mrow data-mjx-texclass="ORD">
                      <mo>&#x2212;</mo>
                      <mn>1</mn>
                    </mrow>
                  </msup>
                  <mo>&#x2261;</mo>
                  <mfrac>
                    <mn>1</mn>
                    <mrow>
                      <mi>a</mi>
                      <mi>d</mi>
                      <mo>&#x2212;</mo>
                      <mi>b</mi>
                      <mi>c</mi>
                    </mrow>
                  </mfrac>
                  <mrow data-mjx-texclass="INNER">
                    <mo data-mjx-texclass="OPEN">[</mo>
                    <mtable columnspacing="1em" rowspacing="4pt">
                      <mtr>
                        <mtd>
                          <mi>d</mi>
                        </mtd>
                        <mtd>
                          <mo>&#x2212;</mo>
                          <mi>b</mi>
                        </mtd>
                      </mtr>
                      <mtr>
                        <mtd>
                          <mo>&#x2212;</mo>
                          <mi>c</mi>
                        </mtd>
                        <mtd>
                          <mi>a</mi>
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

これは逆行列の最も単純な場合です。そして、そう、分母は行列式です。それがゼロだったらどうなるかわかりますね。さて、逆行列について知っておくべきことがほかにもいくつかあります。逆行列を持つ可能性があるのは正方行列だけです。実際に可能かどうかは、行列式で確かめられます。さらに、逆行列の逆行列は元の行列に戻ります。もちろんもっとあります（ああ、本当にもっとあります）が、今はこれで十分です。

### 行列の代数的な性質 {#ssec-mat-props}

**A** と **B** は M×N の行列、**C** は N×P、**D** と **E** は N×N です。**e**<sub>i</sub> は **E** の列ベクトルです。_c_ はスカラーです。

<div class="lblock">
<table cellpadding=2 cellspacing=0>
  <tr>
    <td><b>A + B</b> = <b>B + A</b></td>
  </tr>
  <tr>
    <td>c·(<b>A + B</b>) = c<b>B</b> + c<b>A</b></td>
  </tr>
  <tr>
    <td><b>A·I</b> = <b>I·A</b> = <b>A</b></td>
  </tr>
  <tr>
    <td>
      <b>A·C</b> = <b>C·A</b> となるのは M=P の場合<i>だけ</i>で、しかもごく特別な条件のもとで<i>だけ</i>
    </td>
  </tr>
  <tr>
    <td>
      <b>E·F</b> = <b>I</b> なら、<b>E</b><sup>&minus;1</sup> = <b>F</b> かつ <b>F</b><sup>&minus;1</sup> = <b>E</b>
    </td>
  </tr>
  <tr>
    <td>(<b>A</b><sup>T</sup>)<sup>T</sup> = <b>A</b></td>
  </tr>
  <tr>
    <td>
      (<b>A·C</b>)<sup>T</sup> = <b>C</b><sup>T</sup> · <b>A</b><sup>T</sup>
    </td>
  </tr>
  <tr>
    <td>
      (<b>A·C</b>)<sup>&minus;1</sup> = <b>C</b><sup>&minus;1</sup> · <b>A</b><sup>&minus;1</sup>
    </td>
  </tr>
  <tr>
    <td>
      <b>a</b><sub>i</sub> · <b>a</b><sub>j</sub> = &delta;<sub>ij</sub> なら、<b>A</b><sup>&minus;1</sup> = <b>A</b><sup>T</sup>（言い換えると、ベクトルが単位ベクトルで互いに垂直なら、逆行列は転置行列です。）
    </td>
  </tr>
</table>
</div>

## 空間、基底、座標変換 {#sec-space}

ありうるすべてのベクトルの集まりは<dfn>ベクトル空間</dfn>と呼ばれます。次元の数は、ベクトルの数の個数で与えられます（あるいは逆だったでしょうか？）。2D の空間には要素が 2 つのベクトルがあり、3D のベクトルには 3 つある、という具合です。さて、普通はベクトルの要素が空間のどこにいるかを教えてくれますが、話はそれだけではありません。位置を完全に定めるには、次のものが必要です。

- 基底
- 原点
- 座標

慣れているベクトルは座標の部分を受け持ちますが、ほかの 2 つがなければ座標は何も意味せず、ただの数にすぎません。(2, 1) のような座標の組は、たとえば速さ 1 と同じくらいしか意味を持ちません。それらが何かを意味するには、基準の枠組みが必要です。物理量なら、それは単位です（km/h、マイル/h、m/s など。速さにとってどれほど違いが出るかわかりますね？）。空間なら、それは基底と原点です。

### 座標系 {#ssec-space-sys}

<div class="lblock">
  <table id="fig:img-sys">
    <tr>
      <td>
        <div class="cpt" style="width:192px">
          <img src="img/math/crd_cart.png" alt="Cartesian coords.">
          <br>
          <b>{*@fig:img-sys}a</b>: 標準の座標系 S。点 <i>P</i> は座標 <nobr>(3, 2)</nobr> で与えられます。
        </div>
      </td>
      <td>
        <div class="cpt" style="width:192px">
          <img src="img/math/crd_shear.png" alt="Sheared coords">
          <br>
          <b>{*@fig:img-sys}b</b>、せん断した座標系 S'。点 <i>P</i> は座標 <nobr>(1, 2)</nobr> で与えられます。
        </div>
      </td>
    </tr>
  </table>
</div>

{\*@fig:img-sys}a は、おそらくおなじみの 2D のデカルト座標系です。横の x 軸（**i** = (1, 0) ）と縦の y 軸（**j** = (0, 1) ）があります。そしてその中に点 _P_ があります。格子線をたどると x=3、y=2 なので、_P_= (3, 2) ですよね？ まあ、そうでもあり、そうでもありません。私の意見では、ほとんどそうではありません。

問題は、空間の点には本当の座標など_ない_ということです。それはただそこにあるだけです。座標は基準の枠組みによって決まり、それは基本的に任意です。これを示すために、{@fig:img-sys}b を見てください。この絵には座標系 S' があり、横の x 軸（**u** = (1, 0) ）はそのままですが、y 軸（**v** = (1, 1) ）は 45° せん断されています。そしてこの座標系では、点 _P_ は (3, 2) ではなく座標 (1, 2) で与えられます。ある座標系の座標を別の座標系でそのまま使うと、まずいことが起きます。

ここで 2 つの疑問が出てきます。なぜわざわざ別の座標の組を使うのか、そして 2 つの座標系の間をどう変換するのかです。後者はこの記事の残りで扱います。前者については、デカルト座標系はとても便利ですが、それにこだわると現実の（あるいは仮想の）世界の計算がとてつもなく複雑になる場面がたくさんあります。1 つには、惑星の軌道や磁気に関わることを記述するのは、球座標や円柱座標のほうがずっと簡単です。また、テクスチャマッピングでは、テクセルを持つテクスチャを、ほぼすべての場合にワールドの座標ときれいに揃っていない面に貼る必要があります。[アフィン変換](affine.html)はこの完璧な例です。ですから、そう、デカルト座標でない座標を使うのは本当にとても役に立つのです。

### 座標の基底を作る {#ssec-space-build}

デカルト座標系のほかにも座標系があると言うのは結構ですが、実際にはどう使えばいいのでしょうか？ 実はとても簡単です。デカルト座標系で座標を使うとき、本当は何をしているのかを考えてみてください。もう一度 {@fig:img-sys}a を見てください。(x, y)= (3, 2) のような座標の組が与えられたとします。その位置を見つけるには、x 軸に沿って 3、y 軸に沿って 2 進めば、点 _P_ にたどり着きます。さて、座標系 S'（{@fig:img-sys}b）には (x', y') = (1, 2) がありますが、y 軸がないので、S で使った手順はここではうまくいきません。しかし、ベクトル **u** と **v** は_あります_。**u** に沿って 1、**v** に沿って 2 進めば、また点 _P_ に来ます。座標系 S に戻ると、x 軸と y 軸は実はそれぞれベクトル _i_ と _j_ なので、結局どちらの座標系でも同じ手順を使っていたのです。基本的に、していることは次のとおりです。

<table id="eq:coords">
  <tr>
    <td class="eqnrcell">({!@eq:coords}a)</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>P</mi>
                  <mo>=</mo>
                  <mtext mathvariant="bold">i</mtext>
                  <mo>&#x22C5;</mo>
                  <mi>x</mi>
                  <mtext>&#xA0;</mtext>
                  <mo>+</mo>
                  <mtext>&#xA0;</mtext>
                  <mtext mathvariant="bold">j</mtext>
                  <mo>&#x22C5;</mo>
                  <mi>y</mi>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  <tr>
    <td class="eqnrcell">({!@eq:coords}b)</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>P</mi>
                  <mo>=</mo>
                  <mtext mathvariant="bold">u</mtext>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mi>x</mi>
                    <mo data-mjx-alternate="1">&#x2032;</mo>
                  </msup>
                  <mtext>&#xA0;</mtext>
                  <mo>+</mo>
                  <mtext>&#xA0;</mtext>
                  <mtext mathvariant="bold">v</mtext>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mi>y</mi>
                    <mo data-mjx-alternate="1">&#x2032;</mo>
                  </msup>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

さて、注意を払っていたなら、これらの式の構造に見覚えがあるはずです。そう、前に {@eq:eqset}d で見ました。ベクトルと座標を行列とベクトルに書き直すと、次のようになります。

<math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
  <mstyle displaystyle="true" scriptlevel="0">
    <mrow data-mjx-texclass="ORD">
      <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
        <mtr>
          <mtd>
            <mtext mathvariant="bold">M</mtext>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <mtext mathvariant="bold">i</mtext>
                  </mtd>
                  <mtd>
                    <mtext mathvariant="bold">j</mtext>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <mn>1</mn>
                  </mtd>
                  <mtd>
                    <mn>0</mn>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mn>0</mn>
                  </mtd>
                  <mtd>
                    <mn>1</mn>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
            <mo>,</mo>
            <mtext>&#xA0;</mtext>
            <mtext>&#xA0;</mtext>
            <mtext mathvariant="bold">x</mtext>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <mi>x</mi>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mi>y</mi>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
            <mo>;</mo>
            <mtext>&#xA0;</mtext>
            <mtext>&#xA0;</mtext>
            <msup>
              <mtext mathvariant="bold">M</mtext>
              <mo data-mjx-alternate="1">&#x2032;</mo>
            </msup>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <mtext mathvariant="bold">u</mtext>
                  </mtd>
                  <mtd>
                    <mtext mathvariant="bold">v</mtext>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <mn>1</mn>
                  </mtd>
                  <mtd>
                    <mn>1</mn>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <mn>0</mn>
                  </mtd>
                  <mtd>
                    <mn>1</mn>
                  </mtd>
                </mtr>
              </mtable>
              <mo data-mjx-texclass="CLOSE">]</mo>
            </mrow>
            <mo>,</mo>
            <mtext>&#xA0;</mtext>
            <mtext>&#xA0;</mtext>
            <msup>
              <mtext mathvariant="bold">x</mtext>
              <mo data-mjx-alternate="1">&#x2032;</mo>
            </msup>
            <mo>=</mo>
            <mrow data-mjx-texclass="INNER">
              <mo data-mjx-texclass="OPEN">[</mo>
              <mtable columnspacing="1em" rowspacing="4pt">
                <mtr>
                  <mtd>
                    <msup>
                      <mi>x</mi>
                      <mo data-mjx-alternate="1">&#x2032;</mo>
                    </msup>
                  </mtd>
                </mtr>
                <mtr>
                  <mtd>
                    <msup>
                      <mi>y</mi>
                      <mo data-mjx-alternate="1">&#x2032;</mo>
                    </msup>
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
<br>
<table>
  <tr>
    <td class="eqnrcell">({!@eq:coords}c)</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>P</mi>
                  <mo>=</mo>
                  <mtext mathvariant="bold">M</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">x</mtext>
                  <mo>=</mo>
                  <msup>
                    <mtext mathvariant="bold">M</mtext>
                    <mo data-mjx-alternate="1">&#x2032;</mo>
                  </msup>
                  <mo>&#x22C5;</mo>
                  <msup>
                    <mtext mathvariant="bold">x</mtext>
                    <mo data-mjx-alternate="1">&#x2032;</mo>
                  </msup>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
</table>

ベクトル **x** と **x**' には、これまでどおり座標が入っています。新しいのは、座標系を行列 **M** と **M'** の形で定義したことです。行列を作っているベクトルは、その座標系の<dfn>基底ベクトル</dfn>です。もちろん、座標系 S の基底ベクトルは標準の単位ベクトルなので、それらが作る行列は単位行列 **M == I** で、安全に省けます（そして普通は省きます）が、それが舞台裏にあることを忘れないでください。実は、式に普通は暗黙のうちに加えられているものがもう 1 つあります。原点 _O_ です。標準の原点はゼロベクトルですが、そうである必要はありません。

{\*@eq:pt-def} は、点の定義の完全な式です。_O_ は座標系の原点、**M** は基底ベクトルを定め、**x** はその基底での、原点から始まる座標の組です。これらはそれぞれ完全に任意であることに注意してください。先ほどの説明の **M** と **x** は、それらの例にすぎません。

<table id="eq:pt-def">
  <tr>
    <td class="eqnrcell">({!@eq:pt-def})</td>
    <td class="eqcell">
      <math xmlns="http://www.w3.org/1998/Math/MathML" display="block">
        <mstyle displaystyle="true" scriptlevel="0">
          <mrow data-mjx-texclass="ORD">
            <mtable rowspacing=".5em" columnspacing="1em" displaystyle="true">
              <mtr>
                <mtd>
                  <mi>P</mi>
                  <mo>=</mo>
                  <mi>O</mi>
                  <mo>+</mo>
                  <mtext mathvariant="bold">M</mtext>
                  <mo>&#x22C5;</mo>
                  <mtext mathvariant="bold">x</mtext>
                </mtd>
              </mtr>
            </mtable>
          </mrow>
        </mstyle>
      </math>
    </td>
  </tr>
</table>

### 最後の注意 {#ssec-space-notes}

点は、単なる座標の組としてではなく、{@eq:pt-def} の形（つまり原点、基底の行列、座標のベクトル）で考えるのが本当に一番です。この技法はものすごくたくさんの問題に応用でき、それらに一般的な記述があると、問題を解くのが簡単になることがわかるでしょう。たとえば、スプライトや背景の回転と拡大縮小は、座標系の変更にほかなりません。_pa-pd_ には何の魔法もありません。それらは画面→テクスチャ空間の変換を定める行列にすぎないのです。

座標系の変更を扱うときは、何が何をするのかを理解しているか、とても注意してください。2 つの座標系の間で変換するとき、意図していたこととちょうど逆のことを書いてしまうのはとても簡単です。たとえば、前の段落の座標系 S と S' について、**x = M · x'**、つまり **M** は S' から S へ変換することがわかります。しかし **M** の基底ベクトルは座標系 S の_中に_あるので、S から S' へ変換すると思いたくなるかもしれません。でもそうではないのです。GBA が使う **P** 行列でも同じようなことが起きます。この行列の基底ベクトルはテクスチャ空間の中にあります（[アフィン](affine.html)のページの図 5 を見てください）。つまり、それが行う変換は画面からテクスチャ空間へのもので、その逆では_ありません_。

基底の行列は正方である必要はありません。どんな M×N の行列でも使えます。これは N 次元から M 次元への変換に相当します。たとえば M=3、N=2（つまり 3D のベクトルが 2 つ）なら、3D の世界の中の平らな平面になります。N\>M なら射影になります。
