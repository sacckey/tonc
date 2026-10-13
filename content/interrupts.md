# 16. 割り込み

<!-- toc -->

## はじめに {#sec-intro}

特定の条件のもとで、CPU にしていることを中断させ、代わりに別の関数を実行させて、そのあとで元の処理を続けさせることができます。この処理は<dfn>割り込み</dfn>（interrupt。r は 2 つでお願いします）と呼ばれます。割り込みを処理する関数は<dfn>割り込みサービスルーチン</dfn>（interrupt service routine）、あるいは単に割り込みと呼ばれます。割り込みを起こすことは、割り込みを<dfn>発生させる</dfn>（raise）と言います。

割り込みは、多くの場合、特定のハードウェアのイベントに結びついています。たとえば、PC のキーボードのキーを押すと割り込みが発生します。PC のもう 1 つの例は VBlank です（そう、PC にもあるのです）。GBA にも同じような割り込みがあり、ほかにも HBlank や DMA などの割り込みがあります。特に HBlank の割り込みは、気の利いた効果にたくさん使えます。割り込みの完全な一覧は、このあとすぐに示します。

割り込みは現在の処理を止め、素早く「何か」をして、制御を戻します。「素早く」という言葉を強調しておきます。割り込みは短いルーチンであるべきです。

## 割り込みのレジスタ {#sec-regs}

割り込み専用のレジスタは 3 つあります。`REG_IE`（`0400:0200h`）、`REG_IF`（`0400:0202h`）、`REG_IME`（`0400:0208h`）です。`REG_IME` は割り込みのマスター制御で、これが「1」に設定されていない限り、割り込みは完全に無視されます。特定の割り込みを有効にするには、`REG_IE` の適切なビットを設定する必要があります。割り込みが起きると、`REG_IF` の対応するビットが設定されます。割り込みを処理したことを知らせる（acknowledge）には、そのビットを再びクリアする必要がありますが、そのやり方は控えめに言っても少し直感に反します。割り込みを処理したことを知らせるには、実際にはそのビットをもう一度*設定*する必要があるのです。そうです。ビットをクリアするには、そのビット（すでに 1 になっています）に 1 を書き込む必要があります。

`REG_IE` のビットを設定するほかに、その割り込みに関係するほかのレジスタのビットも設定する必要があります。たとえば、HBlank の割り込みには `REG_DISPSTAT` のビットも必要です。私の考えでは（間違っていたら正してください）、割り込みには送り手と受け手の両方が必要で、`REG_IE` が受け手を、`REG_DISPSTAT` などのレジスタが送り手を制御しているのだと思います。それを念頭に置いて、`REG_IE` と `REG_IF` のビットの配置を見てみましょう。

<div class="reg">
<table class="table-reg" id="tbl:reg-ie">
<caption class="reg">
  REG_IE @ 0400:0200 and REG_IF @ 0400:0202
</caption>
<tr class="bits">
  <td>F E<td>D<td>C<td>B A 9 8<td>7<td>6 5 4 3<td>2<td>1<td>0
<tr class="bf">
  <td>-
  <td class="rclr7">C
  <td class="rclr6">K
  <td class="rclr5">Dma
  <td class="rclr4">Com
  <td class="rclr3">Tm
  <td class="rclr2">Vct
  <td class="rclr1">Hbl
  <td class="rclr0">Vbl
</table>
<br>

<table class="table-reg-vert">
  <col class="bits" width=40>
  <col class="bf" width="8%">
  <col class="def" width="12%">
<tr align="left"><th>ビット<th>名前<th>定義<th>説明
<tbody valign="top">
<tr class="bg0">
  <td> 0 <td class="rclr0">Vbl
  <td>IRQ_VBLANK
  <td><b>VBlank</b> 割り込み。
    <code>REG_DISPSTAT</code>{3} も必要です。
<tr class="bg1">
  <td> 1 <td class="rclr1">Hbl
  <td>IRQ_HBLANK
  <td><b>HBlank</b> 割り込み。
    <code>REG_DISPSTAT</code>{4} も必要です。HDraw の<i>後</i>に起きるので、
    ここでしたことは次のラインから反映されます。
<tr class="bg0">
  <td> 2 <td class="rclr2">Vct
  <td>IRQ_VCOUNT
  <td><b>VCount</b> 割り込み。
    <code>REG_DISPSTAT</code>{5} も必要です。
    <code>REG_DISPSTAT</code> の上位バイトで、割り込みを発生させる VCount を
    指定します。スキャンラインの<i>始まり</i>に起きます。
<tr class="bg1">
  <td>3-6<td class="rclr3">Tm
  <td>IRQ_TIMER<i>x</i>
  <td><b>タイマー</b>割り込み。タイマーごとに 1 ビット。
    <code>REG_TMxCNT</code>{6} も必要です。タイマーが
    オーバーフローしたときに割り込みが発生します。
<tr class="bg0">
  <td> 7 <td class="rclr4">Com
  <td>IRQ_COM
  <td><b>シリアル通信</b>割り込み。どうやら
    <code>REG_SCCNT</code>{E} も必要なようです。転送が
    完了したときに発生します。少なくともそう聞いていますが、
    シリアル通信のことは本当に何も知りません。
<tr class="bg1">
  <td>8-B<td class="rclr5">Dma
  <td>IRQ_DMA<i>x</i>
  <td><b>DMA</b> 割り込み。チャンネルごとに 1 ビット。
    <code>REG_DMAxCNT</code>{1E} も必要です。転送全体が
    完了したときに割り込みが発生します。
<tr class="bg0">
  <td>C <td class="rclr6">K
  <td>IRQ_KEYPAD
  <td><b>キーパッド</b>割り込み。
    <code>REG_KEYCNT</code>{E} も必要です。<code>REG_KEYCNT</code> で
    指定したボタンのどれか、またはすべてが押されたときに発生します。
<tr class="bg1">
  <td> D <td class="rclr7">C
  <td>IRQ_GAMEPAK
  <td><b>カートリッジ</b>割り込み。GBA からカートリッジが
    抜かれたときに発生します。
</tbody>
</table>
</div>

## 割り込みサービスルーチン {#sec-isr}

上で説明した割り込みのレジスタを使って、どの割り込みを使いたいかを示します。次の段階は、割り込みサービスルーチンを書くことです。これは単なる型のない関数（`void func(void)`）で、ほかの多くの関数と同じ C の関数です。HBlank 割り込みの例を示します。

```c
void hbl_pal_invert()
{
    pal_bg_mem[0] ^= 0x7FFF;
    REG_IF = IRQ_HBLANK;
}
```

1 行目は、パレットメモリの最初のエントリの色を反転させます。2 行目は `REG_IF` の HBlank のビットをリセットして、割り込みを処理したことを示します。これは HBlank 割り込みなので、最終的にはスキャンラインごとに色が変わります。これはそれほど想像しにくくないでしょう。

この関数を既存のプログラムに追加しただけでは、何も変わりません。なぜでしょうか？ そう、ISR はできましたが、それがどこにあるかを GBA に教える必要があるのです。そのためには、割り込みの処理全体をもっと詳しく見る必要があります。

:::warning 割り込みを正しく処理済みにするには

割り込みを処理したことを知らせるには、`REG_IF` のその割り込みのビットを**設定**する必要があります。それも、そのビット*だけ*をです。つまり、普通は「<code>REG_IF <b>|=</b> IRQ_<i>x</i></code>」ではなく、「<code>REG_IF <b>=</b> IRQ_<i>x</i></code>」が正しいやり方です。|= を使うほうは、まだ処理していないものも含めて、発生したすべての割り込みを処理済みにしてしまいます。

普通はどちらも同じ結果になりますが、複数の割り込みが同時に来ると、まずいことになります。自分が何をしているのかに注意してください。

:::

### 割り込みの処理の流れ {#ssec-isr-proc}

割り込みの処理の流れ全体はややこしく、その一部はまったく手の出せないところにあります。以下は、プログラマであるあなたが知っておく必要のあることの一覧です。全体の話は [GBATEK : irq control](https://problemkaputt.de/gbatek.htm#gbainterruptcontrol) を見てください。

1.  割り込みが起きる。BIOS の最も深いダンジョンの奥底で何やら黒魔術が働き、CPU は IRQ モードと ARM ステートに切り替わる。いくつかのレジスタ（`r0-r3, r12, lr`）がスタックに積まれる。
2.  BIOS は `0300:7FFC` にあるアドレスを読み込み、そのアドレスに分岐する。
3.  `0300:7FFC` が指しているコードが実行される。この時点では ARM ステートなので、これは ARM コードで*なければならない*！
4.  ISR が終わったら、`REG_IF` に書き込んで割り込みを処理したことを知らせ、`bx lr` 命令を発行して ISR から戻る。
5.  先ほど保存したレジスタがスタックから取り出され、プログラムの状態が通常に戻る。

手順 1、2、5 は BIOS が行い、3 と 4 はあなたの担当です。さて、原理的には、ISR のアドレスをアドレス `0300:7FFC` に置くだけで済みます。少し楽をするために、まず関数ポインタの型を作っておきましょう。

```c
typedef void (*fnptr)(void);
#define REG_ISR_MAIN *(fnptr*)(0x03007FFC)

// Be careful when using it like this, see notes below
void foo()
{
    REG_ISR_MAIN= hbl_pal_invert;  // tell the GBA where my isr is
    REG_DISPSTAT |= VID_HBL_IRQ;  // Tell the display to fire HBlank interrupts
    REG_IE |= IRQ_HBLANK;          // Tell the GBA to catch HBlank interrupts
    REG_IME= 1;                   // Tell the GBA to enable interrupts;
}
```

さて、これでおそらく動きますが、いつものように話にはまだ続きがあります。

-   まず、`REG_ISR_MAIN` がジャンプする先のコードは ARM コードで*なければなりません*！ `-mthumb` フラグを付けてコンパイルすると、全体がぴたりと止まってしまいます。
-   割り込みの中で割り込まれたらどうなるのでしょうか？ 実は、それはあまり起こりえません。あとで説明する凝ったことをしない限りは。というのも、割り込みを許可しているのは `REG_IME` だけではなく、<dfn>プログラムステータスレジスタ</dfn>（PSR）にも IRQ 用のビットがあるのです。割り込みが発生すると、CPU は処理がすべて終わるまで、そこで割り込みを無効にします。
-   `hbl_pal_invert()` は、自分が HBlank 割り込みによって呼ばれたのかどうかを確認していません。この場合は有効になっている割り込みがそれだけなので問題ありませんが、いろいろな種類の割り込みを使うときは、それらを振り分けることが欠かせません。そのため、次の節で[割り込みの交換台](#sec-switch)を作ります。
-   最後に、割り込みを必要とする [BIOS コール](swi.html)を使うときは、`REG_IFBIOS`（== `0300:7FF8`）でも割り込みを処理済みにする必要があります。使い方は `REG_IF` と同じです。

:::warning 領域のミラーについて

GBA のメモリ領域は、一定のバイト数ごとにミラーされています。たとえば IWRAM（`0300:0000`）は 8000h バイトごとにミラーされているので、`0300:7FFC` は `03FF:FFFC`、つまり `0400:0000`−4 でもあります。こちらのほうが速いのですが、これを利用すべきかどうかは、私にはよくわかりません。no$gba v2.2b はこれをエラーとして扱います。もっとも、これはちょっとした見落としだったようで、v2.2c で修正されました。とはいえ、警告はしておきます。

:::

## 割り込みの交換台を作る {#sec-switch}

`hbl_pal_invert()` 関数は単一の割り込みの例ですが、複数の割り込みを扱わなければならないこともあります。また、状況に応じて別の ISR を使えるようにしたいこともあるでしょう。その場合、すべてを 1 つの関数に詰め込むのは最善の方法ではないかもしれません。そこで、割り込みの交換台を作ります。

<dfn>割り込みの交換台</dfn>（interrupt switchboard）は、電話の交換台に少し似た働きをします。電話（つまり `REG_IF` の割り込み）がかかってくると、交換手はそれが有効な番号かどうかを確認し（`REG_IE` と比べ）、有効なら正しい受け手（あなたの ISR）に電話をつなぎます。

この交換台には、追加の機能もいくつか付けます。割り込みに実際の ISR がつながっていない場合でも、`REG_IF` と `REG_IFBIOS` の両方で電話を処理済みにします。また、割り込みの入れ子も許しますが、そのためには ISR 自体で少し余分な作業が必要です。

### 設計とインターフェースの検討 {#ssec-switch-design}

実際の交換台は全体の一部にすぎません。構造体、変数、関数もいくつか必要です。必要な基本のものは次のとおりです。

-   **`__isr_table[]`**。割り込みの表。いろいろな ISR への関数ポインタの表です。割り込みには優先順位を付けるべきなので、表には、それぞれのポインタがどの割り込みのものかも示すべきです。このために `IRQ_REC` 構造体を使います。
-   **`irq_init()`** / **`irq_set_master()`**。マスター ISR を設定します。`irq_init()` は割り込みの表と割り込みそのものも初期化します。
-   **`irq_enable()`** / **`irq_disable()`**。割り込みを有効、無効にする関数です。`REG_IE` と、送り手のビットがあるレジスタの両方を面倒みます。これらのビットは `__irq_senders[]` という内部の表に持たせていて、これを使えるようにするため、これらの関数の入力パラメータは割り込みのフラグそのものではなく、割り込みの*インデックス*である必要があります。`IRQ_`*`foo`* のフラグに対応する `II_`*`foo`* があるのはこのためです。
-   **`irq_set()`** / **`irq_add()`** / **`irq_delete()`**。割り込みサービスルーチンを追加、削除する関数です。1 つ目は ISR の優先順位を完全に指定できます。`irq_add()` は、指定した割り込みの現在の ISR を置き換えるか、リストの最後に追加します。`irq_delete()` は ISR を 1 つ削除し、空いた場所に合わせてリストを詰めます。

これらの関数はどれも、だいたい次のようなことをします。割り込みを無効にし（`REG_IME`=0）、自分の仕事をして、それから割り込みを再び有効にします。割り込みをいじっている最中に割り込まれるのは見苦しいことになるので、こうするのは良い考えです。サービスルーチンに関わる関数は関数ポインタ（`fnptr` 型）を受け取り、以前の ISR を示す関数ポインタも返します。ISR を連鎖させたい場合に役立つかもしれません。

以下に、構造体、表、そして `irq_enable()` と `irq_add()` の実装を示します。どちらの関数でも、`__irq_senders[]` 配列を使って、割り込み要求が確実に送られるように、どのレジスタのどのビットを設定するかを決めています。`irq_add()` 関数はさらに、置き換えるために現在の表から要求された割り込みを探すか、埋めるための空きスロットを探します。ほかのルーチンも同じようなものです。もっと見たければ、libtonc の *tonc_irq.h/.c* を見てください。

```c
//! Interrups Indices
typedef enum eIrqIndex
{
    II_VBLANK=0, II_HBLANK, II_VCOUNT, II_TIMER0,
    II_TIMER1,   II_TIMER2, II_TIMER3, II_SERIAL,
    II_DMA0,     II_DMA1,   II_DMA2,   II_DMA3,
    II_KEYPAD,   II_GAMEPAK,II_MAX
} eIrqIndex;

//! Struct for prioritized irq table
typedef struct IRQ_REC  
{
    u32 flag;   //!< Flag for interrupt in REG_IF, etc
    fnptr isr;  //!< Pointer to interrupt routine
} IRQ_REC;

// === PROTOTYPES =====================================================

IWRAM_CODE void isr_master_nest();

void irq_init(fnptr isr);
fnptr irq_set_master(fnptr isr);

fnptr irq_add(enum eIrqIndex irq_id, fnptr isr);
fnptr irq_delete(enum eIrqIndex irq_id);

fnptr irq_set(enum eIrqIndex irq_id, fnptr isr, int prio);
void irq_enable(enum eIrqIndex irq_id);
void irq_disable(enum eIrqIndex irq_id);
```

```c
// IRQ Sender information
typedef struct IRQ_SENDER
{
    u16 reg_ofs;    //!< sender reg - REG_BASE
    u16 flag;       //!< irq-bit in sender reg
} ALIGN4 IRQ_SENDER;

// === GLOBALS ========================================================

// One extra entry for guaranteed zero
IRQ_REC __isr_table[II_MAX+1];

static const IRQ_SENDER __irq_senders[] =
{
    { 0x0004, 0x0008 },     // REG_DISPSTAT,    DSTAT_VBL_IRQ
    { 0x0004, 0x0010 },     // REG_DISPSTAT,    DSTAT_VHB_IRQ
    { 0x0004, 0x0020 },     // REG_DISPSTAT,    DSTAT_VCT_IRQ
    { 0x0102, 0x0040 },     // REG_TM0CNT,      TM_IRQ
    { 0x0106, 0x0040 },     // REG_TM1CNT,      TM_IRQ
    { 0x010A, 0x0040 },     // REG_TM2CNT,      TM_IRQ
    { 0x010E, 0x0040 },     // REG_TM3CNT,      TM_IRQ
    { 0x0128, 0x4000 },     // REG_SCCNT_L      BIT(14) // not sure
    { 0x00BA, 0x4000 },     // REG_DMA0CNT_H,   DMA_IRQ>>16
    { 0x00C6, 0x4000 },     // REG_DMA1CNT_H,   DMA_IRQ>>16
    { 0x00D2, 0x4000 },     // REG_DMA2CNT_H,   DMA_IRQ>>16
    { 0x00DE, 0x4000 },     // REG_DMA3CNT_H,   DMA_IRQ>>16
    { 0x0132, 0x4000 },     // REG_KEYCNT,      KCNT_IRQ
    { 0x0000, 0x0000 },     // cart: none
};


// === FUNCTIONS ======================================================

//! Enable irq bits in REG_IE and sender bits elsewhere
void irq_enable(enum eIrqIndex irq_id)
{
    u16 ime= REG_IME;
    REG_IME= 0;

    const IRQ_SENDER *sender= &__irq_senders[irq_id];
    *(u16*)(REG_BASE+sender->reg_ofs) |= sender->flag;

    REG_IE |= BIT(irq_id);
    REG_IME= ime;
}

//! Add a specific isr
fnptr irq_add(enum eIrqIndex irq_id, fnptr isr)
{
    u16 ime= REG_IME;
    REG_IME= 0;

    int ii;
    u16 irq_flag= BIT(irq_id);
    fnptr old_isr;
    IRQ_REC *pir= __isr_table;

    // Enable irq
    const IRQ_SENDER *sender= &__irq_senders[irq_id];
    *(u16*)(REG_BASE+sender->reg_ofs) |= sender->flag;
    REG_IE |= irq_flag;

    // Search for previous occurance, or empty slot
    for(ii=0; pir[ii].flag; ii++)
        if(pir[ii].flag == irq_flag)
            break;
    
    old_isr= pir[ii].isr;
    pir[ii].isr= isr;
    pir[ii].flag= irq_flag;

    REG_IME= ime;
    return old_isr;
}
```

### マスター割り込みサービスルーチン {#ssec-switch-master}

マスター ISR の主な仕事は、発生した割り込みを `___isr_table` から探し出し、`REG_IF` と `REG_IFBIOS` の両方で処理済みにすることです。その IRQ 専用のサービスルーチンがあればそれを呼び、なければそのまま BIOS に戻ります。C で書くと、だいたい次のようになります。

```c
// This is mostly what libtonc's isr_master does, but
// you really need asm for the full functionality
IWRAM_CODE void isr_master_c()
{
    u32 ie= REG_IE;
    u32 ieif= ie & REG_IF;
    IRQ_REC *pir;

    // (1) Acknowledge IRQ for hardware and BIOS.
    REG_IF      = ieif;
    REG_IFBIOS |= ieif;

    // (2) Find raised irq 
    for(pir= __isr_table; pir->flag!=0; pir++)
        if(pir->flag & ieif)
            break;

    // (3) Just return if irq not found in list or has no isr.
    if(pir->flag == 0 || pir->isr == NULL)
        return;

    // --- If we're here have an interrupt routine ---
    // (4a) Disable IME and clear the current IRQ in IE
    u32 ime= REG_IME;
    REG_IME= 0;
    REG_IE &= ~ieif;

    // (5a) CPU back to system mode 
    //> *(--sp_irq)= lr_irq;
    //> *(--sp_irq)= spsr
    //> cpsr &= ~(CPU_MODE_MASK | CPU_IRQ_OFF);
    //> cpsr |= CPU_MODE_SYS;
    //> *(--sp_sys) = lr_sys;

    pir->isr();             // (6) Run the ISR

    REG_IME= 0;             // Clear IME again (safety)

    // (5b) Back to irq mode
    //> lr_sys = *sp_sys++;
    //> cpsr &= ~(CPU_MODE_MASK | CPU_IRQ_OFF);
    //> cpsr |= CPU_MODE_IRQ | CPU_IRQ_OFF;
    //> spsr = *sp_irq++
    //> lr_irq = *sp_irq++;

    // (4b) Restore original ie and ime 
    REG_IE= ie;
    REG_IME= ime;
}
```

これらの点のほとんどはすでに説明したので、繰り返しません。`REG_IF` と `REG_IFBIOS` の処理済みにする方法の違いに注意してください。前者は単純な代入を使い、後者は |= を使います。手順 4、5、6 は、現在の IRQ が専用のサービスルーチンを持っている場合にだけ実行されます。手順 4a と 5a は、ISR（手順 6）が CPU モードで動けるようにし、ISR 自身が求めない限り割り込まれないようにするための初期化の手順です。手順 4b と 5b は、4a と 5a を元に戻します。

この関数は、5a と 5b の項目さえなければ C でうまく動くでしょう。これらは CPU モードをシステムモードに設定したり IRQ モードに戻したりするコードですが、そのために必要な命令は C では使えません。もう 1 つの問題は、リンクレジスタ（関数の戻りアドレスを保持するのに使われます）を何らかの方法で保存しなければならないことで、これも C では*絶対に*使えません。

注意: レジスタ**たち**、と複数形で言いました！ CPU のモードごとに専用のスタックとリンクレジスタがあり、名前は同じ（`lr` と `sp`）でも、実際には別物です。普通、C の関数は自分で `lr` を保存しますが、今回はそれが 2 回必要なので、コンパイラに任せるのはとても危険です。それ以外にも、割り込みが起きたときのプログラムの状態を示す、保存されたプログラムステータスレジスタ `spsr` を保存する必要があります。これも C ではうまくできないことです。そのため、マスター ISR にはアセンブリが必要です。
<br>  
というわけで、アセンブリです。下の関数は、`irs_master_c()` のアセンブリ版です。ほぼ 1 行ずつの翻訳ですが、コンパイラが使わない、あるいは使えない命令セットの機能をいくつか利用しています。ここに書いてあることすべてを本当に理解できるとは思っていませんが、少し想像力を働かせれば、ほとんどは追えるはずです。アセンブリを教えるのはこの章の範囲を*はるかに*超えていますが、私の考えでは、学ぶ努力をする価値はあります。Tonc の[アセンブリの章](asm.html)には、このほとんどを理解するのに必要な情報があり、もっと学ぶにはどこへ行けばいいかも示してあります。

```armasm
    .file   "tonc_isr_master.s"
    .extern __isr_table;

/*! \fn IWRAM_CODE void isr_master()
    \brief  Default irq dispatcher (no automatic nesting)
*/
    .section .iwram, "ax", %progbits
    .arm
    .align
    .global isr_master

    @ --- Register list ---
    @ r0 : &REG_IE
    @ r1 : __isr_table / isr
    @ r2 : IF & IE
    @ r3 : tmp
    @ ip : (IF<<16 | IE)

isr_master:
    @ Read IF/IE 
    mov     r0, #0x04000000
    ldr     ip, [r0, #0x200]!
    and     r2, ip, ip, lsr #16     @ irq= IE & IF

    @ (1) Acknowledge irq in IF and for BIOS
    strh    r2, [r0, #2]
    ldr     r3, [r0, #-0x208]
    orr     r3, r3, r2
    str     r3, [r0, #-0x208]

    @ (2) Search for irq.
    ldr     r1, =__isr_table
.Lirq_search:
        ldr     r3, [r1], #8 
        tst     r3, r2
        bne     .Lpost_search       @ Found one, break off search
        cmp     r3, #0
        bne     .Lirq_search        @ Not here; try next irq

    @ (3) Search over : return if no isr, otherwise continue.
.Lpost_search:
    ldrne   r1, [r1, #-4]           @ isr= __isr_table[ii-1].isr
    cmpne   r1, #0
    bxeq    lr                      @ If no isr: quit

    @ --- If we're here, we have an isr ---

    @ (4a) Disable IME and clear the current IRQ in IE
    ldr     r3, [r0, #8]            @ Read IME 
    strb    r0, [r0, #8]            @ Clear IME
    bic     r2, ip, r2
    strh    r2, [r0]                @ Clear current irq in IE
    
    mrs     r2, spsr
    stmfd   sp!, {r2-r3, ip, lr}    @ sprs, IME, (IE,IF), lr_irq

    @ (5a) Set mode to sys
    mrs     r3, cpsr
    bic     r3, r3, #0xDF
    orr     r3, r3, #0x1F
    msr     cpsr, r3

    @ (6) Call isr
    stmfd   sp!, {r0,lr}            @ &REG_IE, lr_sys
    mov     lr, pc
    bx      r1
    ldmfd   sp!, {r0,lr}            @ &REG_IE, lr_sys

    @ --- Unwind ---
    strb    r0, [r0, #8]            @ Clear IME again (safety)
    @ (5b) Reset mode to irq
    mrs     r3, cpsr
    bic     r3, r3, #0xDF
    orr     r3, r3, #0x92
    msr     cpsr, r3

    @ (4b) Restore original spsr, IME, IE, lr_irq
    ldmfd   sp!, {r2-r3, ip, lr}    @ sprs, IME, (IE,IF), lr_irq
    msr     spsr, r2
    strh    ip, [r0]
    str     r3, [r0, #8]
    
    bx      lr
```

:::warning 入れ子の IRQ は厄介

自分が何をしているのかを部分的にしかわかっていないと、入れ子の割り込みルーチンを動かすのは楽しい作業ではありません。たとえば、CPU のモードによって別のスタックが使われることを理解するのにしばらくかかりましたし、入れ子の ISR が動かなかった原因がリンクレジスタも別々だからだと気づくまでに、かなりの時間がかかりました。

`isr_master_nest` は主に libgba の割り込みディスパッチャをもとにしていますが、GBATEK や、[forum:4063](https://gbadev.net/forum-archive/thread/4/4063.html) で説明されている A. Bilyk と DekuTree による全体の分析からも情報を借りています。no$gba の家庭用デバッガ版も、とても貴重でした。ブレークポイント万歳。

自分で割り込みルーチンを作りたいなら、これらの情報源がとても役に立ち、正気を失う度合いをなんとか許容できる程度に抑えてくれるでしょう。

:::

:::note 非推奨のお知らせ

以前は、割り込みの入れ子と優先順位付けを自動的に面倒みる、別のマスターサービスルーチンを使っていました。複雑すぎると判断されたので、これに置き換えました。

割り込みの入れ子は今でも可能ですが、割り込み可能であることを ISR の中で自分で示す必要があります。

:::

## 入れ子の割り込みのデモ {#sec-demo}

今回のデモでは、上で説明したことを少しずつ示します。

-   HBlank 割り込みを使って、画面に色のグラデーションを表示します。
-   2 つの異なるマスター ISR を切り替えられます。プログラムの流れを HBlank の ISR に振り分ける交換台の `isr_master` と、HBlank 割り込みを直接処理する C の ISR です。後者を動かすには、もちろん ARM でコンパイルしたコードを使う必要があります。その方法も、すぐあとで示します。
-   最後に、入れ子の ISR の交換台があっても、入れ子の割り込みが実際に動いているところを見られなければあまり意味がありません。ここでは 2 つの割り込み、VCount と HBlank を使います。HBlank の ISR は縦方向の色のグラデーションを作ります。VCount の ISR は色をリセットして、数スキャンラインの間 CPU を拘束します。割り込みが入れ子にならなければ、グラデーションはしばらく止まります。入れ子になれば、いつもどおりに続きます。
-   それから、ついでに、HBlank と VCount の IRQ をオンオフできるようにしてあります。

操作は次のとおりです。
<table>
<tbody valign="top">
<tr>
  <th>A</th>
  <td>アセンブリの交換台と、C の直接の ISR を切り替える。</td>
</tr>
<tr>
  <th>B</th>
  <td>HBlank と VCount の優先順位を切り替える。</td>
</tr>
<tr>
  <th>L,R</th>
  <td>VCount と HBlank の IRQ をオンオフする。</td>
</tr>
</tbody>
</table>

```c
#include <stdio.h>
#include <tonc.h>

IWRAM_CODE void isr_master();
IWRAM_CODE void hbl_grad_direct();

void vct_wait();
void vct_wait_nest();


CSTR strings[]= 
{
    "asm/nested",   "c/direct", 
    "HBlank",       "VCount"
};

// Function pointers to master isrs.
const fnptr master_isrs[2]= 
{
    (fnptr)isr_master,
    (fnptr)hbl_grad_direct 
};

// VCount interrupt routines.
const fnptr vct_isrs[2]= 
{
    vct_wait, 
    vct_wait_nest
};


// (1) Uses tonc_isr_master.s' isr_master() as a switchboard
void hbl_grad_routed()
{
    u32 clr= REG_VCOUNT/8;
    pal_bg_mem[0]= RGB15(clr, 0, 31-clr);
}

// (2a) VCT is triggered at line 80; this waits 40 scanlines
void vct_wait()
{
    pal_bg_mem[0]= CLR_RED;
    while(REG_VCOUNT<120);
}

// (2b) As vct_wait(), but interruptable by HBlank
void vct_wait_nest()
{
    pal_bg_mem[0]= CLR_RED;
    REG_IE= IRQ_HBLANK;     // Allow nested hblanks
    REG_IME= 1;
    while(REG_VCOUNT<120);
}


int main()
{
    u32 bDirect=0, bVctPrio= 0;

    tte_init_chr4_b4_default(0, BG_CBB(2)|BG_SBB(28));
    tte_set_drawg((fnDrawg)chr4_drawg_b4cts_fast);
    tte_init_con();
    tte_set_margins(8, 8, 128, 64);

    REG_DISPCNT= DCNT_MODE0 | DCNT_BG0;

    // (3) Initialize irqs; add HBL and VCT isrs 
    // and set VCT to trigger at 80
    irq_init(master_isrs[0]);
    irq_add(II_HBLANK, hbl_grad_routed);
    BFN_SET(REG_DISPSTAT, 80, DSTAT_VCT);
    irq_add(II_VCOUNT, vct_wait);
    irq_add(II_VBLANK, NULL);

    while(1)
    {
        //vid_vsync();
        VBlankIntrWait();
        key_poll();

        // Toggle HBlank irq
        if(key_hit(KEY_R))
            REG_IE ^= IRQ_HBLANK;

        // Toggle Vcount irq
        if(key_hit(KEY_L))
            REG_IE ^= IRQ_VCOUNT;

        // (4) Toggle between 
        // asm switchblock + hbl_gradient (red, descending)
        // or purely hbl_isr_in_c (green, ascending)
        if(key_hit(KEY_A))
        {
            bDirect ^= 1;
            irq_set_master(master_isrs[bDirect]);
        }

        // (5) Switch priorities of HBlank and VCount
        if(key_hit(KEY_B))
        {
            //irq_set(II_VCOUNT, vct_wait, bVctPrio);
            bVctPrio ^= 1;
            irq_add(II_VCOUNT, vct_isrs[bVctPrio]);
        }

        tte_printf("#{es;P}IRS#{X:32}: %s\nPrio#{X:32}: %s\nIE#{X:32}: %04X", 
            strings[bDirect], strings[2+bVctPrio], REG_IE);
    }

    return 0;
}
```

上のコードには、デモの主なコード、振り分けられる HBlank と VCount の ISR、そして便利のためのこまごまとしたものが入っています。`hbl_grad_direct()` という C のマスター ISR は別のファイルにあり、あとで説明します。

まず、割り込みサービスルーチンの中身です（ポイント 1 と 2）。どちらのルーチンもかなり単純です。HBlank のルーチン（`hbl_grad_routed()`）は、スキャンラインのカウンタの値を使ってバックドロップの色を設定します。一番上では `REG_VCOUNT` は 0 なので、色は青になります。一番下では 160/8=20 になるので、青と赤の間のどこか、紫になります。さて、最初のスキャンラインが実際には青ではなく赤になっていることに気づくかもしれません。これは、a) HBlank 割り込みはスキャンラインの*後*に起きること（これは以前 [DMA のデモ](dma.html#sec-demo)でも問題になりました）と、b) HBlank は VBlank の間にも起きるので、ライン 0 の色は `REG_VCOUNT`=227 のときに設定され、それが明るい赤色になることが理由です。

VCount のルーチンは、スキャンライン 80 で動き出します。色を赤に設定し、スキャンライン 120 まで待ちます。2 つの違いは、`vct_wait()` はただ待つだけなのに対し、`vct_wait_nest()` は HBlank 割り込みを有効にすることです。`isr_master` はサービスルーチンを呼ぶ前に割り込みを無効にすることを思い出してください。そのため、後者の VCount のルーチンは `hbl_grad_routed()` によって割り込まれるはずですが、前者は割り込まれないはずです。{@fig:irq-demo}a と {@fig:irq-demo}b を見ればわかるように、まさにそのとおりのことが起きます。

ポイント 3 は、そもそも割り込みを準備しているところです。`irq_init()` の呼び出しは ISR の表をクリアし、マスター ISR を設定します。引数は NULL にでき、その場合は tonc のデフォルトのマスター ISR が使われます。`irq_add()` の呼び出しは、HBlank と VCount の割り込みとそのサービスルーチンを初期化します。サービスルーチンを渡さなければ、交換台は単に割り込みを処理済みにして戻ります。次の章で見るように、これが役に立つこともあります。`irq_add()` は `REG_IE` と `REG_DISPSTAT` の IRQ のビットの両方をすでに面倒みていますが、割り込みを発生させる VCount の設定はまだしないので、これは別に行っています。`irq_add()` の順番はあまり重要ではありませんが、順位の低いものから先に探されるので、頻度の高い割り込みを先に置くのが理にかなっています。

ポイント 4 のように、`irq_set_master()` でマスターサービスルーチンを切り替えられます。ポイント 5 では、入れ子にする VCount のルーチンとしないルーチンを選んでいます。

<div class="cblock">
<table id="fig:irq-demo">
<tbody align="center"><tr>
<td>
<div class="cpt" style="width:192px;">
  <img src="./img/demo/irq_demo_hbl.png"
    alt="Gradient hbl&gt;vct"><br>
  <b>{*@fig:irq-demo}a</b>: グラデーション。入れ子の
  <code>vct_wait_nested</code>。
</div>
</td>
<td>
<div class="cpt" style="width:192px;">
  <img src="./img/demo/irq_demo_vct.png"
    alt="Gradient vct&gt;hbl"><br>
  <b>{*@fig:irq-demo}b</b>: グラデーション。入れ子でない
  <code>vct_wait</code>。
</div>
</td>
<td>
<div class="cpt" style="width:192px;">
  <img src="./img/demo/irq_demo_c.png"
    alt="Gradient hbl in C"><br>
  <b>{*@fig:irq-demo}c</b>: グラデーション。C の
  マスター ISR での HBlank。
</div>
</td>
</tr></tbody>
</table>
</div>

これで、デモが見せられることのほとんどを説明しました。実際に使うときは、`irq_init()` と `irq_add()` があればほぼ十分ですが、このデモはほかにも面白いことを見せてくれます。結果が VBA、no$gba、実機で実際には少し違うのも面白い点で、これは別の点にもつながります。割り込みは時間に厳しいルーチンで、タイミングをエミュレートするのはかなり難しいのです。エミュレータでは動くのに実機では動かないなら、割り込みは調べ始めるのに良い場所です。

これでデモの節はほぼ終わりですが、1 つだけ残っています。C で書いた直接の HBlank の ISR です。でもそうするには、それを ARM コードにする必要があり、効率よくするには IWRAM にも置くべきです。その方法は次のとおりです。

### ARM + IWRAM のコードを使う {#ssec-demo-iwram}

マスター割り込みルーチンは ARM コードでなければなりません。これまでずっと Thumb コードにコンパイルしてきたので、これは新しいことになります。これまでずっと Thumb コードにしてきた理由は、通常のコードの領域は 16 ビットのバスなので、そこでは ARM コードが遅くなるからです。しかし、ARM コードを IWRAM に置くことはできます。IWRAM は 32 ビットのバスを持ち（ウェイトステートもありません）、そこでは ARM コードを使うことが実際に有利になります。

ARM コードとしてコンパイルするのは、実はとても簡単です。`-mthumb` の代わりに `-marm` を使うだけです。一番問題になるのは IWRAM の部分です。関数をどの領域に置くかを指定できる GCC の拡張があります。Tonclib には、そのための次のマクロがあります。

```c
#define EWRAM_DATA __attribute__((section(".ewram")))
#define IWRAM_DATA __attribute__((section(".iwram")))
#define  EWRAM_BSS __attribute__((section(".sbss")))

#define EWRAM_CODE __attribute__((section(".ewram"), long_call))
#define IWRAM_CODE __attribute__((section(".iwram"), long_call))


// --- Examples of use: ---
// Declarations
extern EWRAM_DATA u8 data[];
IWRAM_CODE void foo();


// Definitions
EWRAM_DATA u8 data[8]= { ... };

IWRAM_CODE void foo()
{
    ....
}
```

EWRAM/IWRAM のものは説明不要でしょう。<code>DATA_IN_<i>x</i></code> のものは、グローバルなデータをそれらの領域に置けるようにします。データのデフォルトの領域はどのみち IWRAM なので、少し冗長かもしれません。`EWRAM_BSS` は、初期化されていないグローバル変数に関するものです。初期化されたグローバル変数との違いは、ROM の領域を占める必要がないことです。知る必要があるのは、配列のために RAM にどれだけの領域を確保する必要があるかだけです。

関数版には `long_call` 属性も必要です。コードの分岐には届く範囲に限りがあり、領域をまたぐ分岐はたいてい遠すぎて普通の方法ではできません。それを可能にするのがこの属性です。かつて PC のプログラミングにあった「far」と「near」になぞらえることができます。

これらの拡張は、少し気まぐれなことがあることにも触れておくべきでしょう。1 つには、宣言と定義で属性をどこに置くかが影響するようです。示した例は動くと思いますが、動かなければ、属性の位置を少し動かしてみて、それで直るか試してください。もっと大きな問題は、long_call 属性がいつも働いてくれるとは限らないことです。以前の経験から、関数の定義が別のファイルにある場合*を除いて*、`long_call` は無視されるのだと考えています。呼び出す関数と同じファイルにあると「relocation error」が出ます。これは基本的に、ジャンプが遠すぎるという意味です。結局、関数に関しては、領域ごとにコードを分けなければならないということです。どのみち ARM コードは分けておきたいでしょうから、これはうまく収まります。

というわけで、ARM/IWRAM のコードには、ルーチンを入れた別のファイルを用意し、`IWRAM_CODE` マクロで領域を示し、コンパイルで `-marm` を使う必要があります。IWRAM から ROM の関数を呼びたくなる場合に備えて、`-mlong-calls` も付けておくのが良い考えです。このオプションは、すべての呼び出しを long call にします。ツールチェーンによっては（DKP も含めて）、拡張子が *.iwram.c* のファイルが自動的に IWRAM に入るようにリンカスクリプトが設定されているので、`IWRAM_CODE` は宣言にしか必要ありません。
<br>  
この場合、それは *isr.iwram.c* というファイルになります。これには C で書いた単純なマスター ISR が入っていて、HBlank と割り込みの処理済みの通知だけを扱います。

```c
#include <tonc.h>

IWRAM_CODE void hbl_grad_direct();

// an interrupt routine purely in C
// (make SURE you compile in ARM mode!!)
void hbl_grad_direct()
{
    u32 irqs= REG_IF & REG_IE;

    REG_IFBIOS |= irqs;
    if(irqs & IRQ_HBLANK)
    {
        u32 clr= REG_VCOUNT/8;
        pal_bg_mem[0]= RGB15(0, clr, 0);
    }

    REG_IF= irqs;
}
```

:::tip ARM+IWRAM のコンパイルのフラグ

コンパイルのフラグの「-mthumb」を「-marm -mlong-calls」に置き換えてください。たとえば次のようにします。

```makefile
CBASE   := $(INCDIR) -O2 -Wall

# ROM flags
RCFLAGS := $(CBASE) -mthumb-interwork -mthumb
# IWRAM flags
ICFLAGS := $(CBASE) -mthumb-interwork -marm -mlong-calls
```

詳しくは、このプロジェクトの makefile を見てください。

:::
