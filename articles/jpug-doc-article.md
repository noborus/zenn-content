---
title: "PostgreSQL日本語マニュアルのこれまでとこれから"
emoji: "📘"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [jpugdoc, postgresql, 翻訳]
published: false
---
## はじめに

私はPostgreSQLマニュアルの日本語翻訳に参加しています。

PostgreSQLのマニュアルは、ファイル数が400ぐらい、行数が50万行ぐらい、A4のPDFで3,000ページぐらいあります。

これが1年に1回のメジャーバージョンアップにより、毎回量は違いますが、およそ1.5万行（そのうちリリースノートが4分の1ぐらい）改訂されます。次のメジャーリリースが出るまで、マイナーバージョンにも対応します。

PostgreSQL日本語マニュアルは、改訂された分の翻訳と既存訳の修正をしています。

## PostgreSQL日本語マニュアルのこれまで

PostgreSQLの日本語マニュアルの管理に参加したのは[GitHub](https://github.com/pgsql-jp/jpug-doc)に移行してからです。

それからはPostgreSQL日本語マニュアルについてシステム面で多く担当しているので、私がやったことを書いておきます。

GitHub以降後の初期におこなったのは以下になります。だいたいPostgreSQL 9.5から10.0ぐらいの間におこなったものです。

* ソースの日本語マニュアルをEUC-JPからUTF-8へ変更
* HTMLのCSSを見やすく修正＆ヘッダーフッターの修正
* 公開が中断されていたPDF版（以前はTex経由で作成されていた）を[fop](https://xmlgraphics.apache.org/fop/)経由で作成するように構築 ※1
* SGMLのDocBookからXMLのDocBookを先行採用してHTMLを作成するように変更（本家もこの後に変更）※2
* CIの整備（最初はTravis-CIでGitHub Actionsが使用できるようになってからはそちらに移行）
* 翻訳開発版（日時毎）を公開する[サイト](https://pgsql-jp.github.io/)の構築
* 翻訳開発版は英語と日本語表記になるように構築
* man版、epub版の公開を復活
* [対訳集](https://github.com/pgsql-jp/taiyaku)の再構築

※1 日本語PDFはHTMLと違って、word wrapされない言語対応や日本語フォントの設定からイタリック対応していないフォントの代替指定等結構多めの修正が入っています。

※2 SGMLからXMLの以降は元々本家のソースコードには両方でビルドできるようになっていて、
SGMLのDocBookツールでビルドが使用されていましたが、
日本語版ではUTF-8の相性のよさから先行してXMLのDocBookツールでビルドしたものを公開するように変更しました。

この辺ぐらいまでは、公開して長いので知っている人も多いでしょう。
成果物は、現在も大きく変わっていません。

## 開発の問題点

この後、変更したのは主に開発（翻訳）側のやり方です。

PostgreSQLの日本語マニュアルは、新しいバージョンが出たら、そのバージョンの翻訳版ブランチを用意して同じファイルに（原文をコメントアウトして）翻訳を追加するという方法です。

そこで問題となっていたのが、日本語訳と新しいバージョンのマージ作業です。分量が多いため、コンフリクトの解消がどうしても時間がかかる作業でした。さらにマージミスや、変更があるにもかかわらず日本語訳の追随が見逃されることもありました。
マージ作業を主に担当していたため、この作業を効率化する方法を試行錯誤していたのですが、なかなかうまくいかなかったため専用のツールを作成することにしました。

## 翻訳支援ツールの開発

翻訳支援ツールとして[jpug-doc-tool](https://github.com/noborus/jpug-doc-tool)というツールを作成してます。
これはPostgreSQL 15から本格的に使用を開始しました。少しづつ改良を加えています。

### マージ作業の変更

最初から仕様が決まったわけではなく、最初はXMLをパースして日本語訳を入れた版を作ることを考えていました。
PostgreSQLのマニュアルはXMLのDocBook形式ですが、自動フォーマットが入っていないため、XMLツールを使用して変更するとインデント等は戻すことができません。そのため、元の原文を変換してから翻訳版を入れる二段階方式等も考えましたが、ビルド手順の複雑化や、差分の取り方の複雑化が懸念されたたため、見送って元の形式のまま（原文に日本語翻訳を追加してdiffが取りやすい形）にすることにしました。

最終的には、すでにできているバージョンで英語と日本語訳のペアを抽出し、新しいバージョンで、英語がマッチしたら英語と日本語訳のペアに置き換えるという方法にしました。

```xml
<para>
This is a SQL.
</para>
```

という原文があるとすると、日本語訳が入ったバージョンは以下のようになります。

```xml
<para>
<!-- 
This is a SQL.
-->
これはSQLです。 
</para>
```

このコメント化されている英語と後の日本語訳を抽出します。これを当初は行単位で判別して抽出していましたが、例外的な処理が少なからずあったため、漏れが多くありました。その後は`git diff`の結果を抽出する方式に変更しています。本家のリリースタグ(REL_17_0)と日本語版のブランチdiffを取ると判別が簡単になります。

```diff xml
 <para>
+<!--
 This is a SQL.
+-->
+これはSQLです。 
 </para>
```

このように、`+<!--`と`+-->`で囲まれた部分を英語として抽出して、その後の`+`で続く部分が日本語訳として解釈できます。これを抽出すると
英語の`This is a SQL.`と日本語の`これはSQLです。`のペアができます。

これを翻訳が入っていないマニュアルに対して、英語の部分がマッチしたら書き換えるという方法にしました。

まず完了したバージョンで抽出をおこないます。16が完了したバージョンで、17が新しいバージョンとするとします。

```shell
git checkout doc_ja_16 # 前バージョン
cd doc/src/sgml # マニュアルのディレクトリ
jpug-doc-tool extract # 抽出 doc/src/sgml/.jpug-doc-tool/ に抽出される
```

次に新しいバージョンで適用をおこないます。

```shell
git checkout REL_17_0 # 新バージョン
git switch -c doc_ja_17 # 新バージョンの日本語ブランチ
git merge doc_ja_16 # 前バージョンのマージ
 #（コンフリクトを起こしているため翻訳対象のファイルは元に戻す）
cd doc/src/sgml # マニュアルのディレクトリ
jpug-doc-tool replace # 適用 doc/src/sgml/.jpug-doc-tool/ に抽出されたものを適用
```

これで既存訳の適用ができて、コンフリクトが発生しないマージ版ができます。

これが、完全一致の置き換えです。

### 類似文への対応

ただ、これにより適用されたバージョンは英語版に少しでも変更があれば適用されません。

ちょうどその頃、PostgreSQLのマニュアル全体で、`a SQL`を`an SQL`に変更するという小さな変更が、しかし多くのファイルにまたがる変更がおこなわれました。そのため、日本語訳が適用されずに英語のみのままの箇所が多く残りました。

このような問題に対応するために、類似文を探して適用するという機能を追加しました。

類似文を探すためには、日本語訳の対象となりそうな英文を抽出する必要があります。まず、上記の既存訳の適用をした後に、英語のみになっている翻訳対象を探します。

これは、単純な`<para>`で囲まれているような場合は良いですが、間で分割されたり、翻訳対象外をスキップしたりと翻訳をする人によって微妙な違いがありました。
そのために条件をひたすら追加していって、ある程度は近くなるようにしました。
並行して、元の翻訳対象にするブロックの選択も共通になるように修正しました。それでも100%には達していませんが、かなりの部分が適用できるようになりました。
（これは正規表現をいくつも使用して対応しているため、ソースはひどいものになっています。）

完全一致の置き換えでは以下は適用されません（a SQLをan SQLになっているため）。

```xml
 <para>
 This is an SQL.
 </para>
```

そのため、翻訳対象として`This is an SQL.`を抽出後、先に抽出してあった`This is a SQL.`との比較をおこないます。これは対象ファイルの抽出済み文全部とレーベルシュタイン距離で比較して、一番近いものを適用します。
その際に一応文の長さを考慮した（完全一致を100とする）スコアを出して、《マッチ度 99.99》のようなマークを先頭に追加しています。

```xml
 <para>
 <!--
 This is an SQL.
 -->
《マッチ度 99.99》これはSQLです。
 </para>
```

この場合は`《マッチ度 99.99》`を削除すれば翻訳完了になります。他にも変更が合ったりした場合は、日本語訳が修正になりますが、このスコアが参考になります。
現状ではこのスコアが50以上であれば類似文として適用されるようにしてます。

### 機械翻訳の追加

類似文への対応をしたことで、英文の抽出がある程度可能になりました。

しかし、新規の文や、変更が大きな文は類似文も適用されないので、英文のみのままになります。そこで翻訳する人の参考になるように機械翻訳を入れるようにしました。
機械翻訳は、ライセンス的に問題がなく、カスタマイズが可能な[みんなの自動翻訳](https://mt-auto-minhon-mlt.ucri.jgn-x.jp/)のAPIを使用しています。

[みんなの自動翻訳](https://mt-auto-minhon-mlt.ucri.jgn-x.jp/)のAPIを使用するために、まず[go-textra](https://github.com/noborus/go-textra)というライブラリを作成しました。
これを[jpug-doc-tool](https://github.com/noborus/jpug-doc-tool)に組み込んで、機械翻訳を使用できるようにしました。

機械翻訳は`マッチ度`のスコアが90未満の場合であれば、機械翻訳を追加します。そのため、マッチ度と機械翻訳が両方入る場合もあります。マッチ度が50未満であれば機械翻訳のみです。

```xml
 <para>
 <!--
 This is an SQL and procedure.
 -->
《マッチ度 60.00》これはSQLです。
《機械翻訳》これはSQLと手続きです。
 </para>
```

[みんなの自動翻訳](https://mt-auto-minhon-mlt.ucri.jgn-x.jp/)のAPIは実行制限があるため、バージョンアップ対応時に大量の翻訳をおこなうと制限にかかってしまいます。また、時間もかかるため、機械翻訳は別でするようになっています。

replaceではマークのみ付けます。

```shell
jpug-doc-tool replace # 完全一致、類似文の置き換え、機械翻訳対象にマークをつける
```

```xml
 <para>
 <!--
 This is an SQL and procedure.
 -->
《マッチ度 60.00》これはSQLです。
《機械翻訳》«This is an SQL and procedure.»
 </para>
```

mtreplaceでは`«»`で囲まれた部分を機械翻訳に置き換えます。
（つまり、翻訳対象の英語を`«»`で囲むと他の文章でも使用できます。）

```shell
jpug-doc-tool mtreplace # 機械翻訳を適用
```

```xml
 <para>
 <!--
 This is an SQL and procedure.
 -->
《マッチ度 60.00》これはSQLです。
《機械翻訳》これはSQLです。
 ```

### 翻訳のチェックの追加

:::message
jpug-doc-toolでのチェックは、まだGitHub ActionsではCIをおこなっていません。ローカルで使用して、その中からの問題を手動で修正してます。
:::

翻訳支援ツールにより英語と日本語訳のペアが抽出できるようになると、翻訳のチェックができることに気づきました。

英語と日本語訳のペアを表示してみると以下のようになります。これは、`jpug-doc-tool list`で出力できます。

> Here are some suggestions about the easiest ways to perform common tasks when updating catalog data files.
> **カタログデータファイルを更新する共通の作業を実施するためのもっとも簡単な方法の提案を示します。**

> `<filename>reformat_dat_file.pl</filename>` preserves blank lines and comment lines as-is.
> **`<filename>reformat_dat_file.pl</filename>`は空白行とコメント行をそのまま維持します。**

このリストを見ていると機械的にチェックする方法があることに気づきました。

* 日本語訳に`<filename>`のようなインラインタグがある場合は、英語の原文にも同じタグがあるはずです。
* `reformat_dat_file.pl`のような英単語（英単語とは言えないですが、英語で構成された文字列）がある場合は、日本語訳にも同じ単語があるはずです。

また、

> Defines the desired false positive rate used by `<acronym>BRIN</acronym>` bloom indexes for sizing of the Bloom filter. The values must be between 0.0001 and 0.25. The default value is 0.01, which is 1% false positive rate.
> **ブルームフィルタのサイズ設定のために`<acronym>BRIN</acronym>`ブルームインデックスによって使用される、必要な偽陽性率を定義します。 値は0.0001から0.25の間でなければなりません。デフォルト値は0.01で、これは1%の偽陽性率です。**

`0.0001`や`0.25`のような数字がある場合は、日本語訳にも同じ数字があるはずです。

このような規則をチェックするように組み込みました。ただ、日本語訳で補完や変更している場合もあって、チェックに引っかかった場合に間違っていない場合もあるので、候補として表示するだけにしています。たとえば `400 million`を`4億`に変更している場合は、チェックに引っかかりますが、確認してスルーしています。

```shell
jpug-doc-tool check -t -w -n
```

`-t`でタグのチェック、`-w`で単語のチェック、`-n`で数字のチェックのように使用します。

このチェックにより多くのミスを修正しました。

### 文の数のチェック

上記で説明したように英語と日本語ペアはパラグラフ単位に近いブロック単位で抽出しています。このため、英語は（.）ピリオド、日本語は（。）で終わる複数の文が含まれています。
長い英文の場合は、日本語訳で文を分割して、複数の文にする場合がありますが、英語の2つの文を1つの日本語の文にすることは（めったに）ありません。実際にはいくつかありましたが、支障がなければこれを直そうと（現在）しています。
そのため、英語の文より日本語の文の方が少ない場合はチェックして表示する機能を追加しました。

```shell
jpug-doc-tool check -e
```

ところが、これにより多くの箇所がチェックにひっかかることがわかりました。

PostgreSQLのマニュアルでは、括弧とピリオドの順を括弧を閉じた後にピリオドと、ピリオドの後に括弧を閉じている両方の場合があります。
たとえば単語の補足のような場合は`The default is zero (<literal>0</literal>).`のように括弧を閉じてからピリオドがあります。

それとは別に文自体が括弧で囲まれている場合はピリオドの後に閉じ括弧があります。この場合は前の文がピリオドで終わっています。

> Once you have everything set up, change to the directory `<filename>doc/src/sgml</filename>` and run one of the commands described in the following subsections to build the documentation. (Remember to use GNU make.)

日本語訳ではこのような場合に、最後の（。）だけしか付いていない場合がかなりあり、厳密になっていませんでした。以下のピリオドで終わって括弧がはじまり、ピリオドの後に閉じ括弧がある場合は、日本語訳でも同様に（。）を付けて（。）の後に閉じ括弧をつけるように修正してます。

> **全ての設定が完了したら、`<filename>doc/src/sgml</filename>`ディレクトリに移動し、 次の副節で説明されているコマンドのいずれかを実行して文書を構築します（GNU makeを使うのを忘れずに）。**

↓

> **全ての設定が完了したら、`<filename>doc/src/sgml</filename>`ディレクトリに移動し、 次の副節で説明されているコマンドのいずれかを実行して文書を構築します。**
> **（GNU makeを使うのを忘れずに。）**

これにより文のチェックができるようになりました。

このようなルールにちゃんとしたがっていると、括弧で囲まれた文を読むときにも理解の手助けになります。
括弧の前に「。」がない場合は、その文の単語等の補足として解釈できます。括弧の前に「。」で終わって、括弧がはじまっている場合は、その前の文の補足として解釈できます。

### 対訳語のチェック

英語の単語と日本語訳の単語は必ず一意になるとは限りませんが、PostgreSQLのマニュアルでは、この単語になるという場合があります。これもチェックしたいですが、単純に対訳集を使用というわけにはいきません。そのため、英単語と日本語単語を渡してチェックするというサブコマンドを別途追加しました。

```shell
jpug-doc-tool word -e file -j ファイル
```

こうすると`file`という英単語が入った文の訳に`ファイル`が入って**いない**日本語訳があった場合に表示します。

```text
high-availability.sgml
    WAL file control commands will not work during recovery,
    e.g., <function>pg_backup_start</function>, <function>pg_switch_wal</function> etc.
リカバリの間WALの制御コマンドは稼働しません。
例えば、<function>pg_backup_start</function>や<function>pg_switch_wal</function>などです。
```

WALの場合は`WALファイル`と書かなくても十分と判断されて訳では省略されてしまっていますが、翻訳のブレにつながるので`WALファイル`と直すといったことができます。

[対訳集](https://github.com/pgsql-jp/taiyaku)を使用して登録されている語を全部チェックできないかを考えましたが、対訳集は一意にならない単語も多いため、別途1対1のリストを整備する必要を感じています。

### NG語の登録

翻訳時に単語がブレてしまうと、読み手にとって誤解を招くことがあります。用語は統一されていたほうが望ましいですが、とくにPostgreSQLのマニュアルではカタカナの長音を省略するかどうかで両方の単語が存在する問題が長年続いていました。
見つけたところを修正しても、翻訳を続けていると再度同じ問題が発生していることがあります。そういった単語を指摘しているツールもあり、CIでチェックしたかったのですが、混在した状態で下手に入れてしまうと、指摘事項が見きれないほどでて邪魔になってしまいます。

そこで、ツールの使用はいったん諦めて、自前で`git grep`でチェックを入れるようにしました。

これはNG語のリストファイルを同じレポジトリ内にいれておいて、`Makefile`に追加して、`make check-ng`で実行できるようにしました。
こうすると`.ng.list`ファイルを編集するだけで単語の追加、削除ができ管理がしやすいです。単語は正規表現で記述できます。

```makefile
check-ng:
	@ln -sf $(top_builddir)/.ng.list .ng.list
	@if git --no-pager grep -E --color=always -f .ng.list *sgml ref/*sgml; \
	then \
	  echo "使用NGな語が見つかりました。";\
	  false;\
	fi
```

GitHub actionsでも実行されるため、pull requestするとチェックされます。

```text
少数点           # 間違いやすいtypoも登録してます。
内臓
ププランナ
ユーザー         # PostgreSQLユーザ会なのでユーザーはNGです。
オーバ[^ー]      # 逆にオーバーはOKで、オーバはNGです。
```

これで手戻りがなくなったため、カタカナの長音を統一（現在多くの単語は省くようになっています）するように置き換えをしています。
長音ナシから長音アリに変更する場合も一括変換後にNG語として登録すれば手戻りがなくなります。

### バックポート、フォワードポート

英語、日本語訳の抽出、適用ができるようになったため、日本語訳がすでにある文に対しても抽出した文で上書きすれば、訳の更新ができることに気づきました。

そこで日本語訳を置き換える更新オプションを追加しました。

```shell
jpug-doc-tool replace -u
```

英語文が一致していて、日本語訳が一致していない場合にのみ置き換えます。これはバージョンの新、旧を気にしていないため、バックポート、フォワードポートが可能になります。
以前は、別のブランチで修正した箇所を別のブランチにマージするのが難しかったため、作業対象は1つのブランチだけでした。

このポーティングが可能になったことで、翻訳を追加していく対象のブランチと翻訳が完了しているブランチに修正を入れることが可能になりました。

2025年2月現在では、PostgreSQL 17の翻訳対象ブランチである `doc_ja_17`ブランチでは翻訳を追加して、翻訳が完了している`doc_ja_16`ブランチには修正を入れるという形で作業を進めています。`doc_ja_17`の翻訳が完了してリリースする前に`doc_ja_16`の修正をフォワードポートして`doc_ja_17`を更新する予定です。

すでに`doc_ja_16`には完了して公開した後に50回以上、1500行ぐらいの修正を入れています。これはミスだけではなく翻訳文の統一や同じ表現するといった修正も含まれています。

## 今後の対応

対訳の単語は、めぼしい単語は統一できたので、他にも見つけ次第修正していっています。

それとは別に同じ英文の場合に日本語訳が微妙に違う場合があります。これを統一しつつ、対訳文として抽出しておけば、新規ファイルでも確定の文（翻訳済みの文）として扱うことができて、翻訳者の負担とレビューアーの負担を減らすことができます。
すでに共通のタイトルは統一していますが、これを文にも適用します。

英文が共通で日本語訳が統一されていない文の抽出は、英語と日本語訳ペアのリストを扱いやすい形式（tsv）で出力して、英語の重複を抽出すればできます。

拙作の[trdsql](https://github.com/noborus/trdsql)を使用すると出力形式が選べて便利です。

```shell
jpug-doc-tool list -s -t >jpug.tsv
trdsql -db pdb -ovf -id "\t" "SELECT count(c1), c1 as en,STRING_AGG(DISTINCT c2, '｜') as ja FROM jpug.tsv WHERE c1 != '' GROUP BY c1 HAVING count(c1) > 1 and count(DISTINCT c2) > 1  ORDER BY length(c1)"
```

こういう感じで現在200ぐらいのペアがあります。これを統一していく予定です。

```text
---[ 47]----------------------------------------------------------------
  count | 2
     en | Here's a simple example of usage:
     ja | 簡単な使用例を示します。｜簡単な例を示します。
---[ 48]----------------------------------------------------------------
  count | 2
     en | Name of a new or existing column.
     ja | 新しい列または既存の列の名前です。｜新規または既存の列の名前です。
```

これらが統一できると1000文ぐらいが対訳文として使用できるようになります。

### その他

現在検討中なのが、独立したブロックではないが含まれている文も規定の文として扱えないかを考えています。

AIレビューや修正の自動化も取り入れたいですが、まだ実現には至っていません。使えるものは使って、翻訳者の負担を減らし、質を上げていこうと考えています。

## PostgreSQLマニュアルの翻訳手順の現在

全体の流れは以下のwikiを確認してください。

[JPUG DOCの翻訳活動への参加方法(初めての方向け)](https://github.com/pgsql-jp/jpug-doc/wiki/JPUG-DOC%E3%81%AE%E7%BF%BB%E8%A8%B3%E6%B4%BB%E5%8B%95%E3%81%B8%E3%81%AE%E5%8F%82%E5%8A%A0%E6%96%B9%E6%B3%95(%E5%88%9D%E3%82%81%E3%81%A6%E3%81%AE%E6%96%B9%E5%90%91%E3%81%91))

### 翻訳作業のホントのところ

ltree.sgmlを例として説明します。doc_ja_17ブランチで翻訳をおこなっているとします。
`pg164tail2`というタグが前バージョンの翻訳が完了した時点のタグです。ここからdoc_ja_17にマージされました。
diffを見てみます。

```diff xml
diff --git a/doc/src/sgml/ltree.sgml b/doc/src/sgml/ltree.sgml
index 2e4c972e388..e1478d5909e 100644
--- a/doc/src/sgml/ltree.sgml
+++ b/doc/src/sgml/ltree.sgml
@@ -873,6 +873,17 @@ Europe &amp; Russia*@ &amp; !Transportation
 <type>ltree</type>に対するB-treeインデックス：<literal>&lt;</literal>、<literal>&lt;=</literal>、<literal>=</literal>、<literal>&gt;=</literal>、<literal>&gt;</literal>
     </para>
    </listitem>
+   <listitem>
+    <para>
+<!--
+     Hash index over <type>ltree</type>:
+     <literal>=</literal>
+-->
+《マッチ度[50.000000]》<type>ltree</type>演算子
+《機械翻訳》ハッシュインデックスオーバー<type>ltree</type>:<literal>=</literal>.
+    </para>
+   </listitem>
+
    <listitem>
     <para>
 <!--
@@ -1001,6 +1012,7 @@ INSERT INTO test VALUES ('Top.Collections.Pictures.Astronomy.Galaxies');
 INSERT INTO test VALUES ('Top.Collections.Pictures.Astronomy.Astronauts');
 CREATE INDEX path_gist_idx ON test USING GIST (path);
 CREATE INDEX path_idx ON test USING BTREE (path);
+CREATE INDEX path_hash_idx ON test USING HASH (path);
 </programlisting>
 
   <para>
```

後方の`+CREATE INDEX path_hash_idx ON test USING HASH (path);`は`programlisting`に追加された例で翻訳対象ではありません。

そうすると翻訳対象は`Hash index over <type>ltree</type>:<literal>=</literal>`のみです。
類似文と機械翻訳が入っています。これを修正します。訳すにあたって、ltree.sgmlの他の箇所も確認してみます。すると直前に以下の文があって、

```xml
   <listitem>
    <para>
<!--
     B-tree index over <type>ltree</type>:
     <literal>&lt;</literal>, <literal>&lt;=</literal>, <literal>=</literal>,
     <literal>&gt;=</literal>, <literal>&gt;</literal>
-->
<type>ltree</type>に対するB-treeインデックス：<literal>&lt;</literal>、<literal>&lt;=</literal>、<literal>=</literal>、<literal>&gt;=</literal>、<literal>&gt;</literal>
    </para>
   </listitem>
```

`B-tree index over`が`Hash index over`になっていますが、よく似た文です。
類似文としてこっちをもってきてくれていたら、単語を置き換えるだけで済んだのに。。。と、まあこういうこともあります。
注意点としては、`B-tree`は`B-tree`のままですが、`Hash`は他のところでも`ハッシュ`になっているので、`ハッシュ`に統一します。

```diff xml
diff --git a/doc/src/sgml/ltree.sgml b/doc/src/sgml/ltree.sgml
index e1478d5909e..525b2d2ee46 100644
--- a/doc/src/sgml/ltree.sgml
+++ b/doc/src/sgml/ltree.sgml
@@ -879,8 +879,7 @@ Europe &amp; Russia*@ &amp; !Transportation
      Hash index over <type>ltree</type>:
      <literal>=</literal>
 -->
-《マッチ度[50.000000]》<type>ltree</type>演算子
-《機械翻訳》ハッシュインデックスオーバー<type>ltree</type>:<literal>=</literal>.
+<type>ltree</type>に対するハッシュインデックス：<literal>=</literal>
     </para>
    </listitem>
```

ということで、参考として入っていた日本語訳を削除して、新しい日本語訳に修正しました。

これをpull requestで提出して、レビューをもらってマージされるという流れです。

マッチ度、機械翻訳が入っていないところでも翻訳対象となる場合があるため、確認は必要です。
できそうな気がしてきましたよね？

## PostgreSQLマニュアル翻訳は作業中です

現時点（2025年2月7日）で、PostgreSQLマニュアルの翻訳は[現在17.0を対象にまた作業中](https://github.com/pgsql-jp/jpug-doc/wiki/%E3%83%89%E3%82%AD%E3%83%A5%E3%83%A1%E3%83%B3%E3%83%88%E7%BF%BB%E8%A8%B3%E6%8B%85%E5%BD%93%E3%83%AA%E3%82%B9%E3%83%8817.0)です。

半分を超えているので、翻訳が大変なファイルだけが残っているだけでしょと思われるかもしれませんが、実は簡単に数箇所、場合によっては1つだけ修正すればよいようなファイルもまだ残っています。
これは翻訳に誘い込むためのワナなので、

「ふ、このワシを誘い込もうというのか、こしゃくなマネを...」
「いや、ここはひとつのってみてやるか...」

と言いながら、ぜひpull requestを送ってください。

そして、新しく翻訳をしながら、既存訳の修正もおこなえるような体制になったので、既存の訳への指摘もしてやってください。
