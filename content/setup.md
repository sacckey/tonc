# 2. 開発環境の準備

<!-- toc -->

## はじめに {#sec-intro}

16進エディタで命令を 2 進数で打ち込みたいのでなければ、人が読めるコードを機械語に変換するための開発環境が必要です。この章では、必要な部品を準備して、それを使って Tonc の例をコンパイルする方法を説明します。

この章を終えるころには、次のものがそろっているはずです。

* テキストエディタ
* GBA エミュレータ _(mGBA)_
* クロスコンパイラのツールチェーン _(devkitARM)_
* GBA プログラミングで使うライブラリ _(特に libtonc)_
* このチュートリアルに付属する例


:::warning コマンドラインの基本的な知識が必要です

GBA のゲームをコンパイルするには、コマンドラインの基本的な理解が必要です。なじみがない場合は、こちらの [Unix のコマンドラインのチュートリアル](https://command-line-tutorial.readthedocs.io/)が役に立つかもしれません。

Windows を使っている場合は、devkitARM に付いてくる **MSYS2** のターミナルを使ってください。ほかの OS では、標準のターミナルで十分です。

:::


## テキストエディタを選ぶ {#sec-editor}

プログラミングには、まともなテキストエディタが欠かせません。最低でも、構文の色分けに対応していて、インデントと改行コードを自分で制御できるものが必要です。つまり、残念ながら _notepad.exe_（メモ帳）では力不足です。

選択肢はたくさんあり、すでにお気に入りがあるかもしれません。もしなければ、いくつかおすすめを挙げます。

- [Visual Studio Code](https://code.visualstudio.com/) - Linux、Windows、Mac で動く、人気で多機能なエディタ

- [Kate](https://kate-editor.org/) - もう 1 つの強力なエディタ。少し軽く、完全にオープンソース

- [Geany](https://www.geany.org/) - 性能の低いマシンでもよく動き、プラグインで大きく拡張できる

- [Notepad++](https://notepad-plus-plus.org/) - Windows で広く愛されている、軽量な選択肢

エディタを選んで使い慣れたら、次の節に進んでください。

<div class="cpt cblock" style="width:420px;">
<img src="img/setup/text_editor.png" id="fig:text-editor"><br>
<b>{*@fig:text-editor}</b>: VS Code でファイルを編集しているところ。
</div>

多くのエディタでは、コードをコンパイルして実行するショートカットキー（たいていは <kbd>F5</kbd> か <kbd>Ctrl+Enter</kbd>）を設定できます。これは効率のよい作業の流れになりえますが、このチュートリアルではコマンドラインを使います。内部で何が起きているかを知ることが欠かせないからです。

同じように、コード補完やエラーの強調表示も、時間をかけて設定する価値のある便利な機能ですが、この章の範囲外です。


## GBA エミュレータを入れる {#sec-emu}

言うまでもなく、作った GBA のプログラムを実際に動かす手段が必要です。ときどき実機でテストすることは強くおすすめしますし、それも楽しみの 1 つです。でも、毎日の開発にはもっと手軽なものが欲しいでしょう。そこで、エミュレータの出番です。

これを書いている時点で、GBA の開発に最も向いているエミュレータは [<dfn>mGBA</dfn>](https://mgba.io/) です。とても正確で、メモリビューア、デバッグ用のログ出力、ステップ実行でデバッグするための GDB サーバーなど、開発者向けの機能があります。どれも、うまくいかないとき（そして、そういうときは必ず来ます）に、ずっと楽にしてくれるはずです！

<div class="cpt cblock" style="width:320px;" markdown>
<img src="img/setup/mgba_game.png" id="fig:mgba-game"><br>
<b>{*@fig:mgba-game}</b>: mGBA の中で動いている GBA の ROM
</div>

ほかに使ってみるとよいエミュレータには、[NanoBoyAdvance](https://github.com/nba-emu/NanoBoyAdvance) と [SkyEmu](https://github.com/skylersaleh/SkyEmu) があります。どちらも _サイクル単位で正確_ で、実機を使わずに実機で遊ぶ感覚に最も近づける選択肢です。

最後に、[no$gba](https://problemkaputt.github.io/gba.htm)（デバッグ版）は、やや古く正確さも劣る Windows 専用の GBA エミュレータですが、ほかでは見られない独自のデバッグ機能を持っています。具体的には、画面で操作するデバッガ、性能のプロファイラ、CPU の使用率メーター、そしてバッファオーバーフローなどを見つけられるメモリアクセスのチェックです。うまく動かせれば、とても貴重な道具になります！


## devkitARM を入れる {#sec-dkp}

<dfn>devkitARM</dfn> は、長年にわたって GBA の自作ソフト（homebrew）の標準的なツールチェーンです。<dfn>devkitPro</dfn>（dkP）というチームが提供していますが、非公式には、これらのツールもよく devkitPro と呼ばれます（保守している人たちはそれを嘆いていますが）。

devkitARM を入れるには、[devkitPro の Getting Started](https://devkitpro.org/wiki/Getting_Started) のページを開いて、自分の OS 向けの手順に従ってください。

:::danger パスにスペースを使わないでください

devkitARM はプロジェクトのビルドに [`make`](https://en.wikipedia.org/wiki/Make_(software)) を使いますが、これは（`My Documents` のような）スペースを含むパスをうまく扱えません。これは、`make` がスペースをコマンドラインのオプションの区切りとして使うのに、シェルスクリプトなどとは違って、引用符やエスケープの十分な手段を持っていないためです。ファイル名の一覧を扱うときは、なおさらです。

:::

### Windows の場合のヒント {#ssec-dkp-win}

Windows を使っている場合は、部品を自動でダウンロードしてインストールしてくれる、GUI のインストーラがあります。@fig:devkitpro のように、インストールのときに必ず「GBA Development」を選んでください。

<div class="cpt cblock" style="width:420px;">
<img src="img/setup/devkitpro.png" id="fig:devkitpro"><br>
<b>{*@fig:devkitpro}</b>: Windows で、GBA 用のパッケージと一緒に devkitARM を入れているところ。
</div>


### Linux と Mac の場合のヒント {#ssec-dkp-unix}

Linux か Mac を使っている場合は、dkP の Getting Started のページの手順に従ったあと、ターミナルで `dkp-pacman` を使って `gba-dev` のパッケージグループを入れてください _(Arch Linux を使っているなら、ただの `pacman` で構いません)_。それには、次のコマンドを実行します。

```sh
sudo dkp-pacman -S gba-dev
```

どのパッケージを入れるかを聞かれたら _("Enter a selection (default=all):")_、そのまま <kbd>Enter</kbd> を押して、`gba-dev` グループの全部を入れてください。


## Tonc の例のコードを手に入れる {#sec-examples}

このチュートリアルには、各章で教える概念を実際に示すための、[例の一式](https://github.com/gbadev-org/libtonc-examples)が付いています。

さらに、<dfn>libtonc</dfn> は Tonc に付属する GBA プログラミング用のライブラリで、例をコンパイルするのに必要です。以前は、libtonc を別にダウンロードして、プロジェクトから見つけられる場所に置く必要がありました。でも今は、devkitARM の一部として含まれています。インストールのときに `gba-dev` のパッケージを選んでいれば、_libtonc はもう入っています_。

悪い知らせは、devkitARM には Tonc の例が含まれていないことです。なので、例は自分でダウンロードする必要があります。リポジトリのページの _"Code -> Download&nbsp;Zip"_ から入手するか、ターミナルで [git](https://git-scm.com/) を使って取得できます。

```sh
git clone https://github.com/gbadev-org/libtonc-examples
```
<br>


:::tip toolbox.h と libtonc

最初のほうの章では、学ぶために libtonc の一部を再現した `toolbox.h` という自前のライブラリを作っていきます。でも、実際に使うときは、もっと機能が多く、実績のあるライブラリ（libtonc そのものなど）を使うほうがよいでしょう。

:::


## 例をコンパイルする {#sec-compile}

インストールを確かめるために、例の 1 つをビルドしてみましょう。

ターミナルで、例の 1 つ（ここでは *hello* の例にしましょう）があるディレクトリに移動して、`make` を実行します。

```sh
cd libtonc-examples/basic/hello
make
```

`make` を実行すると、今いるディレクトリの _'Makefile'_ というファイルのルールに従って、プロジェクトがビルドされます。うまくいけば `.gba` ファイルができるので、好きなエミュレータで実行できます。

<div class="cpt cblock" style="width:320px">
<img src="img/setup/mgba_hello.png" id="fig:mgba-hello"><br>
<b>{*@fig:mgba-hello}</b>: mGBA で動いている Tonc の例の 1 つ。
</div>

ここまで来られたら、おめでとうございます！ これで、自分の GBA のプログラムを書き始める準備ができました。

次の章に進んでもいいですし、もっと詳しく知りたければ、続きを読んでください。


:::tip 環境変数を設定する

`Please set DEVKITPRO in your environment` のようなエラーが出たら、環境変数が正しく設定されていません。解決方法はマシンによって違いますが、たいていは、ホームディレクトリにある `.bashrc` というファイルを編集して、次の行を足します。

```sh
export DEVKITPRO=/opt/devkitpro
export DEVKITARM=/opt/devkitpro/devkitARM
export DEVKITPPC=/opt/devkitpro/devkitPPC

export PATH=$DEVKITARM/bin:$DEVKITPRO/tools/bin:$PATH  # optional
```

最後の行は、コンパイラと関連するツールを `PATH` 環境変数に足して、ターミナルから直接使えるようにするものです。

例の makefile はビルドのときに `PATH` を設定するので、この行は必須ではありません。でも、ツールをすぐ使えるようにしておくと便利ですし、次の節を一緒に試したいなら *必須* です。

`.bashrc` を編集したら、変更を反映するために、ターミナルを閉じて開き直す必要があります。または、`source ~/.bashrc` を実行して、今のシェルに変更を反映させることもできます。

:::


## GBA の ROM を手作業でビルドする {#sec-build-steps}

`make` で GBA のプログラムをコンパイルする方法を見てきました。makefile をコピーして、自分のプロジェクトに使うのは大歓迎です！ とはいえ、内部で何が起きているかを知っておくことにも価値があります。

C/C++/アセンブリのソースを正しい GBA の ROM に変換するには、4 つの段階があります。`make` を実行したときの出力にも、それが表れています。

```sh
$ make
hello.c               # <--- invoke the compiler
linking cartridge     # <--- invoke the linker
built ... hello.gba   # <--- elf stripped
ROM fixed!            # <--- header fixed
```

段階は次のとおりです。

1. **ソースをコンパイル、アセンブルする**。人が読める C や C++ のファイル（`.c`/`.cpp`）や、アセンブリのファイル（`.s`/`.asm`）を、[オブジェクトファイル](https://en.wikipedia.org/wiki/Object_code)（`.o`）と呼ばれるバイナリ形式に変換します。ソースファイル 1 つにつき、オブジェクトファイルが 1 つできます。
    
    このためのツールは `arm-none-eabi-gcc` といいます。実はこれは本物のコンパイラの入り口にすぎないのですが、それは細かい話です。ここでの `arm-none-eabi-` という接頭辞は、この GCC が OS のない（ベアメタルの）ARM のプラットフォーム向けの機械語を作ることを表しています。ほかのプラットフォーム向けには、別の接頭辞が付きます。なお、C++ では `gcc` の代わりに `g++` を使います。

2. **オブジェクトファイルをリンクする**。次に、別々のオブジェクトファイルをリンクして、1 つの実行可能な [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) ファイルにします。指定したコンパイル済みのライブラリ（`.a`）があれば、それもこの段階でリンクされます。
    
    実は、コンパイルとリンクは一度にできますが、分けておくのがよい習慣です。本格的なプロジェクトにはたいてい複数のソースファイルがあり、1 つだけ変えたのに全部が再コンパイルされるのを待ちたくはないからです。データ（グラフィック、音楽など）を足し始めると、これはさらに重要になります。
    
    ここでも、リンカを呼び出すのには `arm-none-eabi-gcc` を使います。ただし、実際のリンカは `arm-none-eabi-ld` という名前です。

3. **純粋なバイナリにする**。ELF ファイルにはまだデバッグ用のデータが入っていて、GBA はそのままでは読めません（多くのエミュレータは受け付けてくれますが）。`arm-none-eabi-objcopy` がデバッグ用のデータを取り除き、GBA が受け付ける形にします。まあ、ほぼ受け付ける形に、ですが。

4. **ヘッダーを直す**。どの GBA のゲームにも、正しい GBA の ROM であることを確かめるためのチェックサム付きのヘッダーがあります。リンクの段階でその場所は確保されますが、中身は空のままなので、DarkFader さんの `gbafix` のようなツールを使ってヘッダーを直す必要があります。このツールは devkitARM に付いてくるので、別にダウンロードする必要はありません。


もちろん、dkP のツールが `PATH` に入っていれば、makefile を使わずに、これらのコマンドを全部自分でターミナルから実行することもできます。

*first* という名前の例で試してみましょう。どのライブラリにも依存していないので、コンパイルするのが一番簡単な例です。


```sh
cd libtonc-examples/basic/first/source

# Compile first.c to first.o
arm-none-eabi-gcc -mthumb -c first.c

# Link first.o (and standard libs) to first.elf
arm-none-eabi-gcc -specs=gba.specs -mthumb first.o -o first.elf

# Strip to binary-only
arm-none-eabi-objcopy -O binary first.elf first.gba

# Fix header
gbafix first.gba
```

これで完成です。ゼロからコンパイルした GBA のプログラムです！ まあ……もっと深く掘ることもできますが、今はこのあたりでやめておくのがよいでしょう。 <kbd>x)</kbd>

ここでツールに渡しているさまざまなオプションには、すぐには意味が分からないものもあるかもしれません。興味があれば、[makefile の付録](makefile.html#sec-flags)で説明しています。


:::tip コンパイルにバッチファイルを使わない

これらのコマンドを全部バッチファイルやシェルスクリプトにまとめて、プロジェクトのコンパイルに使いたくなるかもしれません。簡単ですが、おすすめはしません。

理由は、プロジェクトのソースファイルが 2 つ以上になったとたんに分かります。1 つのファイルを編集したら、ソースを _全部_ 再コンパイルする必要はなく、変わったものだけで済むはずです。`make` のようなビルドシステムはそれを判断できるほど賢いのですが、単純なシェルスクリプトにはできません。

プロジェクトのソースファイルが何十個にもなると、これは大きな違いになります！

:::

## ほかのツールチェーン {#sec-alt}

devkitARM の利点は、Windows、Mac、Linux のどれでも、GBA の自作ソフトをコンパイルするための一貫した環境を用意してくれることです。でも、冒険してみたい気分なら、今はほかにもよい選択肢があります。

* [gba-toolchain](https://github.com/felixjones/gba-toolchain) - Makefile の代わりに、CMake というビルドシステムを使う
* [meson-gba](https://github.com/LunarLambda/meson-gba) - Makefile の代わりに、Meson というビルドシステムを使う
* [gba-bootstrap](https://github.com/AntonioND/gba-bootstrap) - GBA のプログラムをコンパイルするのに必要な、最低限のもの。言い換えると、難しい部分はやってもらいつつ、_自分でツールチェーンを組み立てる_ ものです。

なぜこれらを使いたくなるのでしょうか？ 入れるのがもっと簡単かもしれません（多くの Linux のディストリビューションは、`arm-none-eabi-gcc` と関連するパッケージを独自にビルドして提供しています。これは本質的に、devkitARM が提供するものと同じです）。または、devkitARM が使えないマシン（Raspberry Pi など）を使っているのかもしれません。あるいは、単に makefile よりよいビルドシステムを使いたいのかもしれません。

Tonc は devkitARM を使っている前提で書いていますが、ほとんどの情報は、どのツールチェーンを使っていても役に立ちます。


:::danger 「devkitAdvance」は避ける

_devkitAdvance_ というツールチェーンを見かけることがあるかもしれません。これは 2003年から更新されていない、大昔のツールチェーンです。使うと、_20年分_ のコンパイラの改良と最適化を逃すことになります。誰かにこれを勧められたら、逃げてください！

:::
