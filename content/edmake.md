# E. エディタから make を実行する

<!-- toc -->

:::warning この章は古くなっているかもしれない

この部分は見直しが必要かもしれず、すすめているツールややり方の中には非推奨になったものがあるかもしれません。

:::

## はじめに {#sec-intro}

makefile がどれほど優れていても、やはりコマンドラインで動かす処理であることに変わりはなく、それにつきものの問題をすべて抱えています。Unix/Linux の環境なら、それらの問題はたいていかなり小さいのですが、純粋な Windows のシステムでは、DOS のような Windows のコマンドプロンプトで作業しなければなりません。これには、人生をみじめにしかねない、とても恐ろしい欠陥がいくつもあります。ディレクトリの構造をかき分けていくのが楽しくないだけでなく、テキストのコピー/ペーストの直感的でないやり方や、矢印キーでコマンドをたどってタイプミスを直せないことも、少しいらいらします。もう 1 つ、すぐに我慢の限界に達するのは、ちっぽけな Windows のコマンドプロンプトを、瀬戸物屋に突進する狂った象の群れのように駆け抜けていくコンパイルエラーの一覧をスクロールできないことです。しかも、一覧の最初のエラーを見つけてそれを直しさえすれば、全部うまくいくことは*わかっている*のです。さて、Windows のコマンドプロンプトのひどい不備は、代わりに MSYS のウィンドウを使えば回避できます。それでも、それを最大限に活かすには Bash のシェルの使い方を学ぶ必要があります。そして、コマンドラインのウィンドウという余分なウィンドウがあることに変わりはありません。

幸い、どんな種類のコマンドラインのウィンドウもまったく使わずに済ませる方法があります。標準の Windows のメモ帳のようなひどいものを使っているのでない限り、コードのエディタから直接 make やほかのツールを実行できる可能性が高いのです。ここでは、そのうちの 3 つを見てみたいと思います。

-   [ConTEXT](https://archive.org/details/tucows_349269_ConTEXT)
-   [Programmer's Notepad](http://www.pnotepad.org/)。そう、devkitARM に付属しているものです。
-   Microsoft Visual C++

ConTEXT と PN は基本的にテキストエディタの高度な形で、その手のものはかなりたくさんあります。ほとんどは、タブでのファイルの切り替え、検索と置換、カスタマイズできる構文の強調表示、マクロ、シェルのコマンドを使えます。まだ、まあ、本当に*何にでも* Windows のメモ帳を使っているなら、もっと高度なテキストエディタを 1 つダウンロードして、その代わりに使うべきです。メモ帳として知られる大げさなエディットボックスは、どうしようもない場合を除いて、どんな種類のプレーンテキストのファイルにも近づけるべきではありません。ググって見つかるプログラマ向けのエディタは、どれもあらゆる点でメモ帳より優れている可能性が高く、中には実際の notepad.exe を置き換えられるものさえあります。Windows XP Service Pack 2 からは、システムファイルの保護がそれを何度も復活させるので難しくなりましたが、私の意見では、それを永久に撃ち落とす手間は十分にかける価値があります。

ゴホン、失礼しました。標準の Windows のツールがときどきどれほどひどいかを思い出すと、少し熱くなってしまうことがあるのです。ともかく、本題に入りましょう。この章の残りでは、ConTEXT と PN で、現在開いている makefile の make を実行させる方法を示します。この章の最後の節では、その仕事のために MSVC を準備する方法を扱います。どれにも興味がなければ、いつでも[次の章](first.html)に飛ばしてかまいません。

## ConTEXT から make を実行する {#sec-context}

[ConTEXT](https://archive.org/details/tucows_349269_ConTEXT) は軽量で無料のテキストエディタで、私はプレーンテキストの編集のほとんどにこれを使っています。プログラマ向けのエディタがすべきことはすべてでき、メモ帳の置き換え機能や、コードを html 形式に書き出せるツールもあって、tonc を書くのに本当にとても役立ってきました。小さな欠点が 1 つか 2 つありますが、特に気になるものはありません。

シェルのコマンドの管理は、Options-\>Environment Options...-\>Execute Keys（@fig:ctxt-make）にあり、拡張子ごとに働きます。私の場合、それは .mak です。ConTEXT では拡張子ごとに 4 つのコマンドを使えます。私は F9 を「build」ターゲットの make に、F10 をクリーンの操作に使っています。

F9 : make build

-   **Execute:** `make.exe`（必要ならフルパスを加える）
-   **Parameters:** `-f %f build`
-   **Capture output:** yes

F10 : make clean

-   **Execute:** `make.exe`
-   **Parameters:** `-f %f clean`
-   **Capture output:** yes

devkitARM と msys の bin ディレクトリがシステムのパスに入っていることを確かめてください。そうでないと、ConTEXT は make.exe やコンパイラのツールを見つけられません。

<div class="cpt" style="width:464px">

<img src="img/setup/ctxt_make.png" 
  alt="context commands" id="fig:ctxt-make"><br>
<b>{*@fig:ctxt-make}</b>: ConTEXT のシェルのコマンド。
</div>

## Programmer's Notepad 2 から make を実行する {#sec-pn2}

devkitARM に付属するようになるまで PN のことはよく知りませんでしたが、本当に良さそうです。私自身はそれほど使っていませんが、それは単に ConTEXT にまだ満足しているからです。とはいえ、PN のほうがおそらく良いエディタで、ツールチェーンに付属していることもあるので、すでに持っている可能性が高いでしょう。

いろいろ利点はありますが、これは言っておくべきでしょう。デフォルトでは、デスクトップの配色を無視するようです。大したことではないように聞こえるかもしれませんが、背景色がデフォルトできつい白だったので、文字どおり 1 分以上見ていることすらできませんでした。最初にオプションでこれを直そうとしたときは、全体ではなく種類ごとにしか変えられないように見えました。ずっと間違った場所を見ていたことに気づくまでしばらくかかりました <kbd>:P</kbd>。Tools-\>Options-\>Schemes ではなく、Tools-\>Options-\>Styles を見てください。

makefile のためのコマンドを加えるには、Tools-\>Options-\>Tools（@fig:pn-make）に行き、「Make」を選びます。それから「make build」と「make clean」の 2 つのコマンドを加えます。

F9 : make build

  -   **Name:** `mk build`
  -   **Command:** `E:\dev\devkitPro\msys\bin\make.exe`
  -   **Folder:** `%d`（makefile のディレクトリ）
  -   **Parameters:** `-f %f build`
  -   **Shortcut:** F9

F10 : make clean

  -   **Name:** `mk clean`
  -   **Command:** `E:\dev\devkitPro\msys\bin\make.exe`
  -   **Folder:** `%d`（makefile のディレクトリ）
  -   **Parameters:** `-f %f clean`
  -   **Shortcut:** F10

もちろん名前とショートカットは違ってもかまいません。残りは上のとおりにしてください。.mak の拡張子が「Make」の方式に結びついていることを確かめる必要があるかもしれません。

<div class="cpt" style="width:392px">

<img src="img/setup/pn_make.png" 
  alt="PN commands" id="fig:pn-make"><br>
<b>*@fig:pn-make</b>: Programmer's Notepad のシェルの
コマンド。
</div>

エディタに make のコマンドを加えれば、tonc のすべてのデモの makefile を実行できるはずです。問題にぶつかったら、おそらくどこかでパスを設定し忘れています。

## MS Visual C++ 6 から make を実行する {#sec-msvc6}

皆さんの多くは、公式に、学校を通して、あるいは……ほかの方法で、何らかの形で Visual Studio の何らかの版を手に入れていることでしょう。MSVC は実は独自の種類の makefile と NMAKE という make ツールで動きますが、ここではそれは無視して、代わりに GNU の make を使います。この節の手順はバージョン 5 と 6 で使えますが、それより後のバージョンについてはよくわかりません。聞くところによると、それらではたくさんのことが変わったので、それらのどれかを持っているなら、自分で少し調べる必要があるかもしれません。ウィザードで GBA のプロジェクトを作れるアドオンもあることは知っていますが、これも自分で探してください。

### VC と makefile のプロジェクト {#ssec-msvc-make}

#### 段階 1: パスを設定する

まだしていなければ、最初にする必要があるのは、Visual C がツールを見つけられるようにパスを設定することです。\[Tools/Options\] のダイアログを開いて \[Directories\] タブに行き、\[Show Directories for\] のボックスから \[Executable files\] の一覧を選びます（下の @fig:msvc-dirs を見てください）。そこで、MSYS と dkARM の bin ディレクトリを加える必要があります。これらのディレクトリを autoexec.bat に設定することもできます。devkitARM のディレクトリは makefile そのものの中で設定することもできますが、私は Tonc を書くのに 4 台の違うコンピュータを使っているので、そうしないほうが好きです。

<div class="cpt" style="width:424px">

<img src="img/setup/msvc_dirs.png" id="fig:msvc-dirs"
  alt="Adding the the DKA paths to the executable list"><br>
<b>*@fig:msvc-dirs</b>: 実行ファイルの一覧に
dkARM のパスを加える。
</div>

#### 段階 2: makefile のプロジェクトを作る

2 つ目の段階は、独自の makefile を使うプロジェクト/ワークスペースを作ることです。これは、ほかに何と呼ぶでしょう、<dfn>makefile プロジェクト</dfn>と呼ばれます。\[File/New\] のダイアログの \[Projects\] タブ（下の @fig:msvc-new に示します）に行き、Makefile を選んで名前を付け、OK を押します。これは makefile を作るのでは*なく*、プロジェクトを作るだけであることに注意してください！ また、ここで私が使っているプロジェクトの名前は「tonc」です。自分のプロジェクトの名前に変えてください。

<div class="cpt" style="width:568px">

<img src="img/setup/msvc_new_proj.png" id="fig:msvc-new"
  alt="creating a makefile project"><br>
<b>*@fig:msvc-new</b>: makefile のプロジェクトを作る。
</div>

#### 段階 3: 利益！\^H\^H\^H\^Hプロジェクトの設定！

（訳注: 見出しは、"Profit!" を ^H（バックスペース）で消して "Project settings" に書き直すというジョークです。）

OK をクリックすると、プロジェクトの設定に行くように求められます。そうすると、図 6 のダイアログが表示されます。最初にあるのは \[Build command line\] のエディットボックスです。今は、次のようなものになっています。

```sh
NMAKE /f tonc.mak
```

これを次のように変えます。

```sh
make -f tonc.mak build
```

なぜでしょうか？ 標準の VC の make（NMAKE）ではなく、GNU の make（make）を使うからです。なぜでしょうか？ 無料で、プラットフォームに依存せず、たいてい開発キットに付属しているのでプロジェクトの移植性が高くなり、より強力で文書もより充実しているからです。なぜでしょうか？ それは……とにかくそうなんです、いいですか？ これは Rebuild（F7）を押したときに実行されるコマンドです。-f フラグは、どの makefile を使うかを指定します。makefile の中には複数のサブプロジェクトを持てます。この場合は、build と呼ばれるものが有効なものです。

ほかの設定は私たちの目的には重要ではないので、そのままにしておいてください。そう、出力のファイル名もです。makefile がそれを面倒みます。ちなみに、@fig:msvc-make-cfg のワークスペースには 3 つのプロジェクトがあることに注意してください。実際の tonc のための tonc と libtonc、そして保管庫のプロジェクトです。コンパイルはしたくないけれど参照のために手元に置いておきたいソースファイル（ひな形や例など）を保管できる、保管庫のプロジェクトを 1 つ持つのが私の標準のやり方です。私のワークスペースにはどれも 1 つあり、強くおすすめします。

<div class="cpt" style="width:584px">

<img src="img/setup/msvc_make_cfg.png" id="fig:msvc-make-cfg"
  alt="Project Settings."><br>
<b>*@fig:msvc-make-cfg</b>: プロジェクトの設定。

</div>

:::tip GCC の報告を MSVC の報告に変換する

普通の MSVC のプロジェクトをビルドすると、エラーや警告が報告され、それをダブルクリックすると、それを生んだ行に飛べます。GCC の報告の形式は少し違うので、devkitARM ではこれは動きません。

```
# GCC error: {filename}:{line}: error: ...
foo.c:42: error: 'bar' undeclared (first use in this function)
# MSVC error: {dir}\{filename}(line): error ...
dir\foo.c(42) : error C2065: 'bar' : undeclared identifier
```

行番号の形式の違いのせいで、MSVC は混乱して、行どころかファイルすら見つけられません。幸い、make の出力を、msys に付属する bash のシェルの文字列エディタ sed にパイプで通せば、これを変えられます。そうするには、ビルドの呼び出しを次のように変えます。

```sh
make -f tonc.mak build 2>&1 | sed -e 's|\(\w\+\):\([0-9]\+\):|\1(\2):|'
```

`2>&1 | ` は make の標準出力を sed の標準入力に送ります。残りは、最初の 2 つのコロンの前の部分を見つけて、MSVC が期待するかっこ付きの形式に変換する sed のコマンドです。tonc のビルドの行はディレクトリの構造のせいで少し複雑ですが、本当に重要なのは上の行であることに注意してください。

:::

#### 段階 3b: ビルドの構成

これは厳密には必要ありませんが、役に立つかもしれません。Visual C++ では、それぞれ独自のプロジェクトの設定を持つ、複数の<dfn>ビルドの構成</dfn>を持てます。Debug と Release のビルドはおそらくおなじみでしょうが、\[Build/Configurations\] のダイアログ（@fig:msvc-bld-cfg に示します）で自分のものを加えることもできます。tonc のプロジェクトには 5 つの構成があり、どれも tonc.mak の別々のターゲットを動かします。`Build` は現在のデモをビルドし、`Clean` はすべての中間ファイルと出力ファイル（.O、.ELF、.GBA）を取り除きます。特定のデモをビルド/クリーンするには、プロジェクトの設定を変えるか、できれば tonc.mak の中の `DEMO` 変数をそのデモの名前に設定する必要があります。`Build All` と `Clean All` は、すべてのデモについてそれぞれ `Build` と `Clean` を実行します。「Utils」の構成は、あとの例のいくつかで必要になる tonc のライブラリを作ります。

<div class="cpt" style="width:344px">

<img src="img/setup/msvc_bld_cfg.png" id="fig:msvc-bld-cfg"
  alt="Build Configurations.">
<b>*@fig:msvc-bld-cfg</b>: ビルドの構成。

</div>

Visual C++ についてはだいたいこれで全部です。参照している makefile（この場合は tonc.mak）を実際に作る必要はまだあります。テキストファイルの作り方は知っていますよね？ makefile のプロジェクトについて覚えておくべきもう 1 つのことは、ビルドのコマンドはすべて makefile の中にあるということです。ファイルビューアに出てくるファイルは見せるためだけのもので、「普通の」VC のプロジェクトのようにそれ自体がコンパイルされるわけではありません。

:::note tonc.mak で開発キットを簡単に切り替える

Tonc の makefile は、それぞれ単独で成り立ちますが、`DEMO` 変数を使って中央の makefile である tonc.mak から呼ぶこともできるようになっています。そこには、個々の makefile の `CROSS` を上書きする `CROSS`（接頭辞を入れる）変数も置きました。tonc.mak でそれを変えれば、実質的にすべての場所で変わります。

:::

:::tip MSVC 6.0 の役に立たないディレクトリをなくす

Visual Studio 6（とそれ以降も？）には、ワークスペースに加えたプロジェクトごと、そしてプロジェクトの構成ごとに、あらゆる種類の余分なディレクトリを作るという、とても迷惑な癖があるようです。おそらく使うつもりもなく、*間違いなく*頼んでもいない、プロジェクトを散らかすディレクトリです。ディスクから消しても問題は解決しません。プロジェクトや構成を選ぶだけで、また現れるからです。

\*ぶつぶつ\*

良い知らせは、普通のプロジェクトなら、プロジェクトの設定からそれらを取り除き、それからディスクから消せば、すべてまたきれいになることです。悪い知らせは、私たちが使っているのは普通のプロジェクトではなく makefile のプロジェクトで、問題の設定のタブがないことです。そこで、テキストエディタで .DSP を開いて、次のような行をすべて取り除く必要があります。

```
# PROP BASE Output_Dir [DIR]
# PROP BASE Intermediate_Dir [DIR]
# PROP Output_Dir [DIR]
# PROP Intermediate_Dir [DIR]
```

いいえ、自分が何をしているのか正確にわかっているわけではありませんが、ええ、これでディレクトリを消すと、消えた*ままに*なります。実は、DSP からはかなりたくさんの行を取り除けると確信していますが、プロジェクトファイルのコマンドのマニュアルはないので、危険は冒さないことにしています。

さて、DSP ファイルのリファレンスガイドを持っている人や、この不愉快なふるまいが後の MSVC の版でもまだあるかどうか教えてくれる人がいれば、ぜひ聞かせてください。

:::
