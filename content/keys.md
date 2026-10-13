# 6. GBA のボタン（別名キー）

<!-- toc -->

## はじめに {#sec-intro}

きっともうご存じのとおり、GBA には十字キー（D パッド）が 1 つ、操作ボタン（セレクトとスタート）が 2 つ、普通のボタン（A と B）が 2 つ、L/R ボタンが 2 つあり、全部で 10 個の<dfn>キー</dfn>があります。ユーザーと GBA のやり取りの手段はこれだけですが、ほとんどの目的にはこれで十分です。キーの扱い方の原理はとても単純です。キーの状態が入ったレジスタが 1 つあり、そのビットが立っているか消えているかで、どのボタンが押されているかが分かります。これを説明しますが、いずれ欲しくなるであろう、もう少し高度な関数も紹介します。

## キーパッドのレジスタ {#sec-regs}

### キーパッドのレジスタ、REG_KEYINPUT {#ssec-reg-keys}

前に述べたとおり、GBA には 10 個のボタンがあり、よくキーと呼ばれます。その状態は、`0400:0130h` にある `REG_KEYINPUT` レジスタ（別名 `REG_P1`）の最初の 10 ビットにあります。正確な並びは下のとおりです。かなり見れば分かるので、ビットごとの説明は控えます。私が使う定数の名前は "`KEY_`*x*" で、*x* はボタンの名前を大文字にしたものです。

<div class="reg">
  <table class="table-reg" id="tbl-reg-keys">
    <caption class="reg">REG_KEYINPUT @ 0400:0130h</caption>
    <tr class="bits rof">
      <td>F E D C B A</td>
      <td>9</td>
      <td>8</td>
      <td>7</td>
      <td>6</td>
      <td>5</td>
      <td>4</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr class="bf">
      <td> -</td>
      <td class="rclr3">L</td>
      <td class="rclr3">R</td>
      <td class="rclr0">down</td>
      <td class="rclr0">up</td>
      <td class="rclr0">left</td>
      <td class="rclr0">right</td>
      <td class="rclr2">start</td>
      <td class="rclr2">select</td>
      <td class="rclr1">B</td>
      <td class="rclr1">A</td>
    </tr>
  </table>
</div>

キーが押されている（下がっている）かどうかを調べるのは、ある小さな点さえなければ簡単です。キーが押されているとき、ビットは*消える*のです。なので、`REG_KEYINPUT` の初期状態は `0` ではなく `0x03FF` です。そのため、`key` が押されているかどうかは、次のように調べます。

```c
#define KEY_DOWN_NOW(key)  (~(REG_KEYINPUT) & key)
```

ビット演算の知識が少しあやふやなら（早く、はっきりさせてください！）説明すると、これはまず `REG_KEYINPUT` を反転して、より直感的な（そして便利な）「押されているとビットが立つ」形にし、それを調べたいキーでマスクします。実は `key` は複数のキーの組み合わせでもよく、その場合、結果は実際に押されているキーの組み合わせになります。

<div class="note">
  <div class="nhcare">キーの状態は反転しています</div>

  キーのビットはロウアクティブです。つまり、ボタンが押されると**消え**、押されていないと**立ちます**。少し直感に反するかもしれませんが、そういうものなのです。
</div>

### キー制御のレジスタ、REG_KEYCNT {#ssec-reg-keycnt}

キーの扱いで必要になることは、ほぼすべて `REG_KEYINPUT` でできます。とはいえ、さらに制御するための、もう 1 つのキーのレジスタがあることも知っておきたいでしょう。そのレジスタは、キー制御のレジスタ `REG_KEYCNT` です。`REG_DISPSTAT` が映像の割り込みに使われたのと同じように、このレジスタはキーパッドの[割り込み](interrupts.html)に使われます。並びは `REG_KEYINPUT` と同じですが、一番上の 2 ビットが違います。下の表を見てください。`REG_KEYCNT`\{14\} で、キーパッドの割り込みを有効にできます。この割り込みを発生させる条件は、どのキーを見張るかを決める `REG_KEYCNT`\{0-9\} と、正確な条件を決める `REG_KEYCNT`\{15\} で決まります。このビットが消えていれば、さっきのキーのどれか 1 つで割り込みが発生します。立っていれば、それらのキーが全部押されたときに割り込みが発生します。ほとんどのゲームでスタート+セレクト+B+A を押すとリセットできるのは、これを使っているのではないかと思います。もちろん、このレジスタを使うには、まず[割り込み](interrupts.html)の扱い方を知っている必要があります。

<br>
<div class="reg">
  <table class="table-reg" id="tbl-reg-keycnt">
    <caption class="reg">REG_KEYCNT @ 0400:0132h</caption>
    <tr class="bits">
      <td>F</td>
      <td>E</td>
      <td>D C B A</td>
      <td>9</td>
      <td>8</td>
      <td>7</td>
      <td>6</td>
      <td>5</td>
      <td>4</td>
      <td>3</td>
      <td>2</td>
      <td>1</td>
      <td>0</td>
    </tr>
    <tr class="bf">
      <td class="rclr1">Op</td>
      <td class="rclr0">I</td>
      <td>-</td>
      <td class="rclr9">L</td>
      <td class="rclr9">R</td>
      <td class="rclr9">down</td>
      <td class="rclr9">up</td>
      <td class="rclr9">left</td>
      <td class="rclr9">right</td>
      <td class="rclr9">start</td>
      <td class="rclr9">select</td>
      <td class="rclr9">B</td>
      <td class="rclr9">A</td>
    </tr>
  </table>
  <br>
  <table class="table-reg-vert">
    <col class="bits" width=40>
    <col class="bf" width="8%">
    <col class="def" width="12%">
    <tr align="left">
      <th>ビット</td>
      <th>名前</td>
      <th>define</td>
      <th>説明</td>
    </tr>
    <tbody valign="top">
      <tr class="bg0">
        <td>0-9</td>
        <td class="rclr9">keys</td>
        <td>KEY_<i>x</i></td>
        <td>キーの割り込みを発生させるために調べるキー。</td>
      </tr>
      <tr class="bg1">
        <td>E<td class="rclr0">I</td>
        <td>KCNT_IRQ</td>
        <td>キーパッドの割り込みを有効にする</td>
      </tr>
      <tr class="bg0">
        <td>F</td>
        <td class="rclr1">Op</td>
        <td>KCNT_OR, KCNT_AND</td>
        <td>キーの割り込みを発生させるかどうかを決めるのに使う論理演算子。消えていれば OR を使います（ビット 0〜9 のキーのどれかが押されていれば発生）。立っていれば AND を使います（それらのキーが全部押されていれば発生）。</td>
      </tr>
    </tbody>
  </table>
</div>

## ボタンの状態の、その先へ {#sec-adv-keys}

`KEY_DOWN_NOW()` でキーの状態を調べるのは手軽で単純ですが、キーの状態を扱うには、もっとよい、あるいはもっと好ましい方法があります。ここでは、そのうちの 2 つ（か 3 つ）を説明します。まず、<dfn>同期的なキーの状態</dfn>です。これは、入力を処理するときに毎回 REG_KEYINPUT を読む代わりに、ある時点でキーの状態を読んで、その変数を使う、ということを気取って言っただけです。その延長線上にあるのが<dfn>遷移の状態</dfn>で、今の状態だけでなく、前の状態も記録しておきます。これを使うと、キーの状態そのものだけでなく、キーの状態の*変化*を調べられます。最後に、<dfn>tribool</dfn>（3 値の真偽値）です。3 つの状態（ここでは −1、0、+1）を持つ変数で、方向の処理を単純にするのに使えます。

### 同期的な状態と遷移の状態 {#ssec-adv-sync}

`KEY_DOWN_NOW()` を使うのは、<dfn>非同期</dfn>なキーの扱い方の一種です。コードが必要とする時点で状態を調べます。これでも動きますが、いつも一番いいやり方とは限りません。まず、必要になるたびにレジスタを読み込むので（volatile でしたよね？）、コードの面で効率が悪いです。2 つ目の懸念として、ボタンの状態を読むコード同士が少し離れているので、複数のボタンの同時押しが、同時押しとして認識されないことがあります。

でも、これらは小さな懸念にすぎません。一番の問題は、これでは本当にたいしたことができない、ということです。今の状態は分かりますが、それだけです。これがゲームには不十分な理由の簡単な例として、ゲームの一時停止とその解除を考えてみてください。これは普通、スタートを押して一時停止し、もう一度スタートを押して解除します。それで問題なさそうですが、ゲームはあなたが反応するより速く動いている（これは人生の基本的な事実です。あなたがゲームに勝てるのは、ゲームが勝たせてくれているからにすぎません。受け入れてください）ことを考えると、`Start` ボタンは何フレームにもわたって押されたままになります。`KEY_DOWN_NOW()` だと、そのあいだにゲームは一時停止*と*解除を繰り返し、最終的にボタンを離したときのゲームの状態は、本質的にでたらめになります。言うまでもなく、これはよくないこと™です。

そこで同期的な状態の出番です。状態を、たとえばフレームの最初に 1 回だけ読み、それをそのフレーム全体での「唯一の」状態として使うのです。これで、REG_KEYINPUT を余分に読むことも、同時押しを見逃すおそれもなくなります。状態の変化を追うために、前のフレームの状態も保存しておきます。なので、少なくとも 2 つの変数と、それを更新する関数が必要で、ついでに状態を調べる関数もいくつか用意します。どれもかなり小さいので、インライン関数にするのが理にかなっています。

```c
// === (tonc_core.c) ==================================================
// Globals to hold the key state
u16 __key_curr=0, __key_prev=0;
```

```c
// === (tonc_input.h) =================================================
extern u16 __key_curr, __key_prev;

#define KEY_A        0x0001
#define KEY_B        0x0002
#define KEY_SELECT   0x0004
#define KEY_START    0x0008
#define KEY_RIGHT    0x0010
#define KEY_LEFT     0x0020
#define KEY_UP       0x0040
#define KEY_DOWN     0x0080
#define KEY_R        0x0100
#define KEY_L        0x0200

#define KEY_MASK     0x03FF

// Polling function
INLINE void key_poll()
{
    __key_prev= __key_curr;
    __key_curr= ~REG_KEYINPUT & KEY_MASK;
}

// Basic state checks
INLINE u32 key_curr_state()         {   return __key_curr;          }
INLINE u32 key_prev_state()         {   return __key_prev;          }
INLINE u32 key_is_down(u32 key)     {   return  __key_curr & key;   }
INLINE u32 key_is_up(u32 key)       {   return ~__key_curr & key;   }
INLINE u32 key_was_down(u32 key)    {   return  __key_prev & key;   }
INLINE u32 key_was_up(u32 key)      {   return ~__key_prev & key;   }
```

キーの状態は `__key_curr` と `__key_prev` に保存されます。それを更新する関数が `key_poll()` です。この関数はすでに REG_KEYINPUT を反転しているので、変数はハイアクティブになり、あとの操作がより直感的になることに注目してください。たとえば、A が今押されているかを調べるには、`__key_curr` を A のビットである `KEY_A` でマスクするだけです。これが `key_is_down()` のしていることです。`KEY_DOWN_NOW()` も（ほぼ）同じ答えを返しますが、それでも `key_is_down()` を使うことをおすすめします。

:::tip REG_KEYINPUT を読んだら、できるだけ早く反転する

キーの状態を調べてやりたいことは、ハイアクティブの形のほうが単純に楽です。なので、キーの状態の変数もその形で扱えるようにするのがいい考えです。

:::

### 遷移の状態 {#ssec-adv-trans}

一時停止と解除の問題に戻りましょう。`KEY_DOWN_NOW()` が引き起こすいやな振る舞いは、<dfn>キーバウンス</dfn>として知られています。これは、マクロが今の状態しか調べないからです。一時停止と解除をちゃんとするには、キーが押されているかどうかではなく、キーが押され*ようとしている*かどうかを調べるものが必要です。つまり、遷移を調べる必要があります。そこで前の状態が役に立ちます。キーが押された瞬間、つまり押され始めた瞬間には、そのキーは今の状態では押されていて、前の状態では押されていません。言い換えると、「押された（hit）」キーとは、今押されていて、前には押されていなかったキーです。`__key_curr&~__key_prev` です。そのあとは、いつものように単純なマスクで特定のキーを調べられます。これをするのが `key_hit()` です。

本当にこれだけです。同じように、離されたこと（前は押されていて、AND 今は押されていない）や、押し続けていること（前も押されていて、AND 今も押されている）などを調べる関数も作れます。繰り返しますが、これがすべてこんなに単純に見えるのは、状態をすでに反転してあるからです。最初にこれらの関数を作ったときは、ロウアクティブの論理に惑わされて、正しいビット演算を見つけるのにひどく苦労しました。まあ、*本当は*そこまでではないのですが、最初から反転してあれば、もっと楽だったでしょう。

```c
// Transitional state checks.

// Key is changing state.
INLINE u32 key_transit(u32 key)
{   return ( __key_curr ^  __key_prev) & key;   }

// Key is held (down now and before).
INLINE u32 key_held(u32 key)
{   return ( __key_curr &  __key_prev) & key;  }

// Key is being hit (down now, but not before).
INLINE u32 key_hit(u32 key)
{   return ( __key_curr &~ __key_prev) & key;  }

Key is being released (up now but down before)
INLINE u32 key_released(u32 key)
{   return (~__key_curr &  __key_prev) & key;  }
```

### キーの tribool 状態 {#ssec-adv-tri}

これは、[PA_Lib の wiki](https://web.archive.org/web/20110318222049/http://www.palib.info/wiki/doku.php?id=day3) から拝借したちょっとした技法です。キーそのものの話というより、関数の使い方を短く書く方法の話です。この小節で説明することが自分に合うかどうかは、自分で判断してください。

何かを動かせるゲームやデモなどを想像してください。たとえば、キャラクターを左右に動かすには、次のようなものを使うかもしれません。

```c
// variable x, speed dx
if(key_is_down(KEY_RIGHT))
    x += dx;
else if(key_is_down(KEY_LEFT))
    x -= dx;
```

右に動くと *x* が増え、左に動くと *x* が減る。簡単です。ちゃんと動きもします。でも、これは私の if 嫌いが顔を出しているだけかもしれませんが、あまりきれいなコードではありません。なので、もっと滑らかなものが見つからないか探してみましょう。

このコードが実際に何をしているかを見てみてください。2 つの選択によって、変数は増える（+）か、減る（−）か、変わらない（0）かのどれかです。これはまさに <dfn>tribool</dfn>、つまり 3 つの状態（ここでは +1、0、−1）を取りうる変数の定義そのものです。私が欲しいのは、この状態を使って次のように書けるものです。

```c
x += DX*key_tri_horz();
```

この関数の中に `if` を包んでしまうこともできますが、私はビット演算でやるほうが好きです。そのためには、特定のキーのビットを下にシフトして、1 でマスクし、その結果を引き算するだけです。

```c
// === (tonc_core.h) ==================================================
// tribool: 1 if {plus} on, -1 if {minus} on, 0 if {plus}=={minus}
INLINE int bit_tribool(u32 x, int plus, int minus)
{   return ((x>>plus)&1) - ((x>>minus)&1);  }
```

```c
// === (tonc_input.h) =================================================
enum eKeyIndex
{
    KI_A=0, KI_B, KI_SELECT, KI_START,
    KI_RIGHT, KI_LEFT, KI_UP, KI_DOWN,
    KI_R, KI_L, KI_MAX
};

// --- TRISTATES ---
INLINE int key_tri_horz()       // right/left : +/-
{   return bit_tribool(__key_curr, KI_RIGHT, KI_LEFT);  }

INLINE int key_tri_vert()       // down/up : +/-
{   return bit_tribool(__key_curr, KI_DOWN, KI_UP);     }

INLINE int key_tri_shoulder()   // R/L : +/-
{   return bit_tribool(__key_curr, KI_R, KI_L);         }

INLINE int key_tri_fire()       // B/A : -/+
{   return bit_tribool(__key_curr, KI_A, KI_B);         }
```

インライン関数の `bit_tribool()` は、（レジスタでもそれ以外でも）ある数の任意の 2 つのビットから、tribool の値を作ります。ここに挙げたほかの関数は、今のキーの状態とキーのビットを使って、左右、上下、L/R、A/B のボタンの tribool を作ります。ほかのものも、比較的簡単に作れます。これらの関数を使うとコードがすっきりし、しかも速くなります。これらは、これからよく目にすることになります。

上の関数は `__key_curr` しか使っていませんが、ほかの種類のキーの状態を使うコードも簡単に書けます。たとえば、`key_hit` を使った左右の版は、次のようになるでしょう。

```c
// increase/decrease x on a right/left hit
x += DX*bit_tribool(key_hit(-1), KI_RIGHT, KI_LEFT);
```

`__key_curr` の代わりに `key_hit()` を使って、`bit_tribool()` を呼んでいるだけです。そこで “−1” が何をしているのか気になるかもしれませんが、押された状態全体を取り出すために必要なだけです。−1 は 16 進数で `0xFFFFFFFF`、つまり全部のビットが立ったマスクなので、最終的なコードでは最適化で消えることを思い出してください。こうした tribool の使い方も、何度か目にすることになります。

## 簡単なキーのデモ {#sec-demo}

<div class="cpt_fr" style="width:120px;">
<img alt="key_demo screenshot" src="./img/demo/gba_sm.png" id="fig:key-demo">

**{*@fig:key-demo}**: key_demo の画面。L と B を押し続けているところ。
</div>

*key_demo* のデモは、これらのキーの関数の使い方を示しています。モード 4 で GBA の絵（240x160 の 8 ビットのビットマップ）を表示し、ボタンの押し方に応じて色が変わります。普段は灰色で、キーを押すと赤くなり、離すと黄色になり、押し続けているあいだは緑になります。{*@fig:key-demo} は、L と B のボタンでそれを示しています。実際の仕事をしているコードは次のとおりです。

<div id="cd-key-demo">

```c
#include <string.h>

#include "toolbox.h"
#include "input.h"

#include "gba_pic.h"

#define BTN_PAL_ID  5
#define CLR_UP   RGB15(27,27,29)

int main()
{
    int ii;
    u32 btn;
    COLOR clr;
    int frame=0;

    memcpy(vid_mem, gba_picBitmap, gba_picBitmapLen);
    memcpy(pal_bg_mem, gba_picPal, gba_picPalLen);

    REG_DISPCNT= DCNT_MODE4 | DCNT_BG2;

    while(1)
    {
        vid_vsync();
        // slowing down polling to make the changes visible
        if((frame & 7) == 0)
            key_poll();
        // check state of each button
        for(ii=0; ii<KI_MAX; ii++)
        {
            clr=0;
            btn= 1<<ii;
            if(key_hit(btn))
                clr= CLR_RED;
            else if(key_released(btn))
                clr= CLR_YELLOW;
            else if(key_held(btn))
                clr= CLR_LIME;
            else
                clr= CLR_UP;
            pal_bg_mem[BTN_PAL_ID+ii]= clr;
        }
        frame++;
    }

    return 0;
}
```
</div>

`BTN_PAL_ID` は、ボタンに使うパレットの部分の最初のインデックスで、`CLR_UP` は灰色の一種です。ほかの色は見れば分かるはずです。ボタンの色の変化が実際に目に見えるように、キーの読み取りは 8 フレームに 1 回だけにしています。そうしないと、赤や黄色のボタンはほとんど見られないでしょう。（ちなみに、私は実際にはボタンの色を変えているのではなく、そのボタンのピクセルが使うパレットの色を変えているだけです。パレットアニメーションはよいこと™です。）
