---
title: "terminal pager ovのフォローモード（tail -f相当）"
emoji: "🐵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ov", "go", "less", "tail"]
published: true
---

[ターミナルページャー ovの紹介](https://zenn.dev/noborus/articles/2b1087a1274cf41c4c0a) でも書いていた[ov](https://github.com/noborus/ov/) にフォローモードを追加しました。

ログ監視など追記されるファイルを見るためによく`tail -f`が使用されますが、
[Stop using tail -f (mostly)](https://www.brianstorti.com/stop-using-tail/) でも紹介されているようにPagerのlessでも`less +F`により`tail -f`の代わりに使用できることができます。

今回Terminal Pagerの[ov](https://github.com/noborus/ov)でも同様のフォローモードを追加しました。

## follow-mode

`ov --follow-mode ファイル名` (または`-f` ) でファイルを監視し追加があれば更新します。

このフォローモードは起動後でも`ctrl+f`によりON/OFFを切り替えることができます。lessと違うところは、フォローモードのままでも多くの操作ができることです。例えば、ログが流れていながら検索してハイライト表示することができます。

```sh
ov --follow-mode random.log
```

![ov-f](https://raw.githubusercontent.com/noborus/ov/master/docs/ov-tail.gif)

## follow-all

フォローモードで複数ファイルを指定することもできます。ただ、`less`と同じ様に複数ファイルを指定した場合は表示しているファイル以外は裏で更新されているので、（ovでは `]`や`[` により）表示するファイルを切り替える必要があります。

そこでovでは、フォローオールモード(`--follow-all`または`-A`)を追加しました。
フォローオールモードでは更新されて行が追加されたファイルに自動で画面が切り替わります。

```sh
ov --follow-all access.log error.log
```

## exec

そして直接は関係ないですが、follow-allモードがあるので、使いやすくなるかと思ってexecモードを追加しました。
通常はCLIのコマンドは標準出力と標準エラー出力のどちらか、または画面と同じ様に両方を一つの出力にして受け取って表示することになります。

標準出力と標準エラー出力を分けて表示したいときには一旦ファイルに出力して見る必要がありました。

```sh
make >log 2>err.log
```

そこに煩わしさを感じていたので、`--exec`モードのときには、その後の引数をコマンドとして実行し標準出力と標準エラー出力をそれぞれ受け取って表示するようにしました。
片方だけ表示していると問題や経過に気づきづらいので、follow-allモードと一緒に使用することをお勧めします。

```sh
ov --follow-all --exec -- make
```

※`--exec`の後に`--`を付けると以降の引数はovの引数としては解釈されません。

![ov-exec.gif](https://raw.githubusercontent.com/noborus/ov/master/docs/ov-exec.gif)

この例ではエラーで止まっているので、「STDERR」の画面で更新は終了していますが、ワーニング等が出ても続行している場合は、画面が行ったり来たり切り替わりつつ終わります。コマンドが終了後はそのままさかのぼって確認することが可能です。

## まとめ

ovはメモリに溜め込む仕様なので、当初はフォローモードに向いてないと考えていたのですが、本当に巨大なファイルは除いて、多くの場面で使えると気付かされました。
なので、`less +F`を止めて`ov`を使...っても良いこともあるんじゃないでしょうか。
