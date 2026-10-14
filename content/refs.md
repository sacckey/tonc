# F. 参考文献とリンク

<!-- toc -->

## 一般的なサイト {#sec-gen}

### 必須 {#ssec-essential}

- [www.devkitpro.org](https://devkitpro.org/)。GBA の開発で選ばれているツールチェーン、**devkitARM** の本拠地です。NDS などのものもあります。定期的に更新されていて、**libgba** やサンプルコードもここで見つかります。
- [www.gbadev.org](http://www.gbadev.org)。GBA 開発の拠点です。ツール、文書、チュートリアルはすべてここで見つかります。始めたばかりなら、問題を抱えているかどうかにかかわらず、[フォーラム](http://forum.gbadev.org)を訪れることを強くおすすめします。問題を抱えているなら、あなたが最初である可能性は低く、ここですでに解決されているかもしれません。スレッドを立てる前に、[投稿のルール](http://www.albinoblacksheep.com/flash/posting.php)を思い出してください。
- [Nocash](http://nocash.emubase.de/)。Martin Korth のサイトです。（素っ気ないながら）とてつもなく完全なリファレンス文書 [GBATEK](https://problemkaputt.de/gbatek.htm) と、[No\$gba](http://nocash.emubase.de/gba.htm) エミュレータが見つかります。どちらも途方もなく素晴らしいものです。
- [vba.ngemu.com](http://vba.ngemu.com)。**VisualBoy Advance** エミュレータです。タイミングについては no\$gba ほど正確ではありませんが、それでもとてもとても優れていて、もっと親しみやすいインターフェースと、タイル、マップ、IO レジスタなどのためのいろいろなかっこいいビューアがあります。

### ほかの開発環境 {#ssec-altdev}

- [www.ngine.de](http://www.ngine.de)。**HAM** の本拠地です。HAM は GBA のための完全な C の開発環境で、IDE、パレットとマップのエディタ、そしてもちろんコンパイラまでそろっています。追加の（そして最適化された）コードの入った [HEL](http://www.console-dev.de/project/hel-library-for-gba/) という拡張ライブラリもあります。見てみるのは良い考えでしょう。
- **DragonBasic**。C/アセンブリの込み入ったところがすべて嫌なら、この BASIC のような環境を試してみるのもいいでしょう。ただ、このプロジェクトは今、えーと、少し眠っています。
- [Catapult](https://web.archive.org/web/20071106103247/http://www.nocturnal-central.com/catapult.php)。Catapult についてはあまり知りませんが、見た限りでは、Gamemaker に少し似た働き方をするようです。画像や音やスクリプトを作ると、Catapult がそれらを ROM イメージにまとめてくれます。Catapult には、グラフィック、マップ、音のエディタ、チュートリアル、サンプル、エミュレータ、そしておそらくほかにもいろいろなものがそろっています。

### 個人のサイト {#ssec-personal}

（上位の）フォーラムの常連の何人かのサイトです。この人たちはしばらく前から活動していて、そのデモを遊んだりソースコードを眺めたりすることで多くのことを学べます。

- [darkfader.net](https://www.darkfader.net/main/)。Darkfader のサイトで、GBA の開発だけでなく、ほかの多くのシステムについての情報、ツール、デモ、コードがあります。
- [deku.rydia.net](https://web.archive.org/web/20160203205842/http://deku.rydia.net/)。DekuTree64 のサイトにはサウンドミキサー以外のものもあります。（ソース付きの）デモがいくつかと、16 色のタイルのための減色/ディザリングのツール **quither** などのツールです。
- Headspin は、利用できるいろいろな圧縮のルーチンや音楽プレーヤーなど、さまざまな項目の[この概要](https://web.archive.org/web/20220512153726/https://members.iinet.net.au/~freeaxs/gbacomp/)をまとめていました。
- [www.thingker.com](https://web.archive.org/web/20050205230410/http://www.thingker.com:80/gba/)。Scott Lininger のサイトで、とても手に入りにくいらしい**マルチプレイ**のコードを含む、いくつかのデモがあります。
- [www.console-dev.de](http://www.console-dev.de)。Peter Schaut のサイトで、HAMlib の IDE である VisualHam、HAM のアドオンライブラリである HEL、データ管理ツールの katie などがあります。
- [www.pineight.com](http://www.pineight.com/)。gbadev の FAQ の管理者、tepples のサイトです。面白いものがいくつもあります。特に挙げておきたいのは、テトリスのぶっ飛んだ版（あまりに多くのものがあるので、クローンとは呼べません）である **Tetanus on Drugs** と、GBA のためのファイルシステムである **GBFS** です。

## 文書 {#sec-doc}

### チュートリアル {#ssec-tut}

- [www.belogic.com](http://www.belogic.com)。GBA の音のプログラミングについては、ほぼ_唯一の_サイトです。すべてのレジスタについての情報と、_とても_完全なデモのセットがあります。
- **C/C++ のチュートリアル**を探しているなら、[ここ](http://www.cprogramming.com/tutorial.html)に良いものがあるようです
- DekuTree の[サウンドミキシングのチュートリアル](https://stuij.github.io/deku-sound-tutorial/)。Belogic が音のプログラミングの基本を示しているのに対し、このサイトは音/音楽のミキサーを作る手順を案内してくれます。
- [www.drunkencoders.com](http://www.drunkencoders.com)。ここは、GBA 開発の元祖のチュートリアルのシリーズである **PERN プロジェクト**の新しい本拠地です。PERN は全面的な刷新が予定されていましたが、それは DS のほうを優先して後回しにされたようです。DS についてもここでたくさん見つかります。
- jake2431 は、gbadev の [forum:8353](https://gbadev.net/forum-archive/thread/18/8353.html) で **NDS / C / GBA のチュートリアルのリンク**を集めています

### リファレンス文書 {#ssec-ref}

- [**comp.lang.c FAQ**](http://c-faq.com/)。かなり長いですが、GBA のプログラミングと一緒に C も学んでいるなら、とても役に立ちます。
- 数多く出回っているものの 1 つである、[**C のコーディング規約**](https://web.archive.org/web/20110624025547/http://www.jetcafe.org/~jim/c-style.html)の文書です。出回っている tonc 以外のチュートリアルのどれかをもとにコードを書いているなら、これを読む**_必要があります_**。規約に盲目的に従う必要はありませんが、ほとんどを取り入れるのは良い考えで、ほかのサイトが教える悪い癖の多くが解決するでしょう。
- Lee 氏が[最適化](https://web.archive.org/web/20170907095140/http://leto.net/docs/C-optimization.php)についていくつか言いたいことがあるそうです。読みやすさをまったく、あるいはほとんど損なわない、単純な最適化です。
- [**gbadev フォーラムの FAQ**](https://gbadev.net/forum-archive/thread/14/418.html)。初心者かどうかにかかわらず、必読です。ブックマークするなり、ローカルにコピーするなり、印刷するなり、何でもかまいませんが、FAQ を読んでください。
- [**GBATEK**](https://problemkaputt.de/gbatek.htm)。リファレンス文書です。これは基本的に GBA のプログラマの聖書です（ただしこちらは読む価値が_あります_）。情報の密度がとても高く、始めたばかりだと少し戸惑うかもしれませんが、慣れてしまえば、必要なのはほぼこれだけです。HAMLib の文書の一部でもあります。
- [CowBite Spec](http://www.cs.rit.edu/~tjh8300/CowBite/CowBiteSpec.htm)。もう 1 つのリファレンス文書です。少なくとも部分的には GBATEK に基づいています。それほど充実してはいませんが、おそらくより理解しやすいでしょう。
- [www.gnu.org](http://www.gnu.org/manual/manual.html#Development)。いろいろな形式の **GCC の文書**です。これらのサイトには GCC のツールチェーンなどのマニュアルがあります。アセンブラ（**AS**）、コンパイラ（**GCC**）、リンカ（**LD**）、そしてできれば make ツール（**make**）のファイルも手に入れてください。プリプロセッサのマニュアル（**cpp**）も役に立つかもしれません。

### ARM の文書 {#ssec-arm}

当然、ARM のサイト自体にも役に立つ文書があります。そのほとんどは PDF であることに注意してください。

- [miscPDF 8031](https://github.com/ARM-software/abi-aa/releases/download/2023Q3/aapcs32.pdf)。**Arm Architecture Procedure Call Standard**（AAPCS）。関数の間で引数がどう渡されるかを説明しています。アセンブリをしたいなら必読です。
- [PDF DAI0034A](http://netwinder.osuosl.org/pub/netwinder/docs/arm/DAI0034vA.pdf)。**Writing efficient C for ARM**（ARM のための効率的な C の書き方）。ARM 自身のコンパイラを念頭に置いて書かれていますが、いくつかのコツはほかのツールチェーンでも使えます。
- [PDF DDI0210B](https://documentation-service.arm.com/static/5f4786a179ff4c392c0ff819?token=) 大物です。ARM7TDMI の完全な**テクニカルリファレンスマニュアル**です。
- **命令セットのリファレンスシート**。[ARM + Thumb](https://documentation-service.arm.com/static/5ed66080ca06a95ce53f932d?token=) をまとめたもの。
- **アラインメントの問題**についてのサポートの FAQ: [faqdev 1228](https://web.archive.org/web/20080331064522/http://www.arm.com/support/faqdev/1228.html)、[faqdev 1469](https://web.archive.org/web/20070202123419/http://www.arm.com/support/faqdev/1469.html)、[faqip 3661](https://web.archive.org/web/20090117030418/http://www.arm.com/support/faqip/3661.html)。

## ツール {#sec-tools}

### ソースコードのツール {#ssec-tools-text}

まだメモ帳で GBA のコードを書いているなら、やめてください。自分のためと思って、とにかく……やめてください、いいですね？ 私自身はコードを書くのに Visual C を使っていますが、一般的なテキストエディタでも IDE でも、この仕事のためのとても良いツールがほかにもあります。

- **[ConTEXT](https://archive.org/details/tucows_349269_ConTEXT)**。しばらく前に、ある人が、引用すると「メモ帳はクソだ！」という理由で、メモ帳に代わるエディタを求めるスレッドがありました。そこで ConTEXT という名前が何度か挙がり、コーディングの目的に限らず、本当にとても良いものだと思います。独自の強調表示、統合されたシェルのコマンド（たとえば makefile を実行するため）、ヘルプファイルの取り付けができます
- [**Programmer's Notepad**](http://www.pnotepad.org/)（PN）。優れた多機能のテキストエディタです。devkitPro のインストールに付属しています。
- **[Eclipse IDE](http://www.eclipse.org)**。私自身は直接使ってみる時間がありませんでしたが、gbadev のフォーラムの常連のかなりの数がこれを推しています。GBA の開発のための準備の仕方は [forum:5271](https://gbadev.net/forum-archive/thread/14/5271.html) で読めます。
- **[Dev-C++](http://www.bloodshed.net/)**。Dev-C++ もよく挙がる IDE で、見てみる価値があるかもしれません。[forum:1736](https://gbadev.net/forum-archive/thread/7/1736.html) に準備の仕方の情報がありますが、古いスレッドなので、少し余分な作業が必要かもしれません。

### グラフィックのツール {#ssec-tools-gfx}

メモ帳がコーディング（と、最も単純なテキストの編集以外のすべて）にとってひどいものであるのと同じように、MS ペイントは、GBA のゲームで必要な種類のグラフィックにとって地上の地獄です。必要なのは、ビットマップのパレットを完全に制御できるツールで、MS ペイントはその点で見事に失敗しています。ついでに言えば、Visual C の組み込みのビットマップエディタもそうです。そして PhotoShop や Paint Shop Pro のような大きくて重い写真編集ツールでさえ、ここでは苦労すると聞いています。そこで、必要な種類の制御ができるツールをいくつか挙げます。どのツールを使うつもりにせよ、**パレットをめちゃくちゃにしないことを確かめてください**！ エントリを入れ替えてしまうことで知られるエディタもあります。

- **[gfx2gba](http://www.ohnehirn.de/tools/)**。タイルの重複削除やパレットの統合などの面白い機能を持ち、すべてのビット深度と BIOS の圧縮のルーチンに対応する、コマンドラインのグラフィックの変換ツールです。gfx2gba という名前の変換ツールは 2 つあることに注意してください。欲しいのは Markus のものです。HAM の配布物にはこのツールが含まれています。
- **[The GIMP](http://www.gimp.com)**。とても完全な GNU ベースのビットマップ/写真のエディタです。
- **[Graphics Gale](https://graphicsgale.com/us/)** はとても完全なグラフィックのエディタです。ビットマップのエディタに期待されるすべての道具、きちんとしたパレットのエディタ、アニメーションのツールがあります。
- **[Usenti](http://www.coranac.com/projects/#usenti)**。これは私自身のビットマップのエディタです。Graphics Gale ほど高度ではないかもしれませんが、そのぶんインターフェースはずっと簡単です。それ以外にも、パレットの入れ替えや並べ替えのような、とても面白いパレットの調整のオプションがあり、GBA の形式でバイナリ、アセンブリ、C のコードに書き出せます。

### マップのエディタ {#ssec-tools-map}

Tonc で使ったマップはその場で作ったものですが、本格的な仕事にはマップのエディタが必要です。いくつか挙げます。

- **[MapEd](http://nessie.gbadev.org)**、Nessie 作。複数のレイヤー、当たり判定のタイル、独自の書き出しが使えます。おいしい。
- **[Mappy](http://www.tilemap.co.uk/mappy.php)**。これは汎用のマップのエディタで、いろいろな種類のマップに使えます
- **[Mirach](http://www.coranac.com/projects/#mirach)**。これは私自身のマップのエディタですが、時間がないので、入れたかった道具をまだ全部は入れられていません `:(`。

### そのほかのツール {#ssec-tools-misc}

- **[excellut](http://www.coranac.com/projects/#excellut)**。GBA のプログラミングでしたくないことの 1 つは、数学の関数を呼ぶことです。正しい値を得るには[ルックアップテーブル](luts.html)が欲しいところです。excellut は MS Excel を準備して、考えつくどんな種類のルックアップテーブルでも数秒（まあ、数分）で作れるようにします。自分のルックアップテーブルの生成器をまだ作っていないなら（そしてたとえ作っていても）、間違いなく見てみる価値があります。

## 本 {#sec-books}

- Douglas Adams 著、“_The Hitchhiker's Guide to the Galaxy_”（『銀河ヒッチハイク・ガイド』）。まあ、これはリファレンスとは言えませんが、それでもおすすめです。数 42 の意味とバベルフィッシュの由来を知るためだけでも。
- Edward Angel 著、“_Interactive Computer Graphics with Open GL_”。3D の本ですが、線形代数の多くは 2D にも応用できます。関係する章は 4 章（行列による変換）と 5 章（遠近法（モード 7 が気になる人は？））です。3<sup>rd</sup> edition（第 3 版）を手に入れてください。第 2 版には恥ずかしい誤りが多すぎます。
- George B. Arfken & Hans J. Weber 著、“_Mathematical Methods for Physicists_”（『基礎物理数学』）。物理学が RPG なら、これはモンスターマニュアルでしょう。1-3 章でベクトルと行列をとても詳しく扱っています。
- André LaMothe 著、“_Black Art of 3D Game Programming_”。DOS の時代のものなので、手に入れにくいかもしれません。（ちょうど GBA のような）厳しいハードウェアの制約のもとでの 3D プログラミングを扱っています。とても良い本です。
- André LaMothe 著、“_Tricks of the Windows Game Programming Gurus_”。LaMothe 氏のまた別の 1000 ページ超の大著（数多くのうちの 1 冊）で、ゲームプログラミング全般、特に DirectX への優れた手引きです。
- David C. Lay 著、“_Linear Algebra and its Applications_”。私の[行列](matrix.html)のページのほぼすべてはこの本から来ています。
- Eric Meyer と Jennifer Niederst による、O'Reilly のポケットリファレンス “_CSS_” と “_HTML_”。このサイトのようなものにとっては、まさに命の恩人です。
- Steve Oualline 著、“_How Not to Program in C++_”。表紙にはあなたに向かって舌を出しているコンピュータがいて、はじめにの最初の文は「痛みは素晴らしい学習の道具である」です。これは良いに違いないとわかるでしょう。この本は、明らかなものから闇の帝王本人が作ったようなものまで、解くべき 111 個の壊れたコードの問題を出してきます。これらの問題の少なくとも半分に自分の姿を見いだせないなら、あなたはまだそれほど長く C でコードを書いていないのです。
