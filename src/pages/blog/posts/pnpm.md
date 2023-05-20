---
layout: "../../../layouts/MarkdownBlogPost.astro"
title: "pnpmと他のパッケージマネージャの比較"
published: 2023-04-18
updated: 2023-04-18
---

# 初めに

なにやら pnpm とやらが話題になっていたので、何がすごいのかを npm や yarn との比較も交えながら調べてみた。
本記事で使用した環境は以下の通り。

```
Nodejs: v18.16.0
npm: 9.5.1
yarn: 1.22.19
pnpm: 8.2.0
```

# 概要

まず、pnpm は performed npm の略で、npm@2 までの課題を npm@3 や yarn とは異なったアプローチで解決したもののようだ。
古いバージョンの npm には幾つかの課題があり、その内の一つが依存関係にあるパッケージの管理方法が非効率的なことだった。

## npm や yarn についておさらい

例えば express に依存するプロジェクトを立ち上げたとすると、古い npm では node_modules 配下に express ディレクトリが作成される。

```
% ls -al ./node_modules/express
total 272
drwxr-xr-x   9 ****  staff     288  4 18 20:41 .
drwxr-xr-x   3 ****  staff      96  4 18 20:41 ..
-rw-r--r--   1 ****  staff  113107 10 26  1985 History.md
-rw-r--r--   1 ****  staff    1249 10 26  1985 LICENSE
-rw-r--r--   1 ****  staff    5419 10 26  1985 Readme.md
-rw-r--r--   1 ****  staff     224 10 26  1985 index.js
drwxr-xr-x  10 ****  staff     320  4 18 20:41 lib
drwxr-xr-x  33 ****  staff    1056  4 18 20:41 node_modules
-rw-r--r--   1 ****  staff    5311  4 18 20:41 package.json
```

更に express ディレクトリ内部にも node_modules が作成され、express が依存関係するパッケージがコピーされる。

```
% ls -al ./node_modules/express/node_modules
total 0
drwxr-xr-x  33 ****  staff  1056  4 18 20:41 .
drwxr-xr-x   9 ****  staff   288  4 18 20:41 ..
drwxr-xr-x   8 ****  staff   256  4 18 20:41 accepts
drwxr-xr-x   6 ****  staff   192  4 18 20:41 array-flatten
...省略
```

依存するパッケージはそのまま

```
% ls -al ./node_modules/express/node_modules/accepts
total 64
drwxr-xr-x   8 ****  staff   256  4 18 20:41 .
drwxr-xr-x  33 ****  staff  1056  4 18 20:41 ..
-rw-r--r--   1 ****  staff  5096 10 26  1985 HISTORY.md
-rw-r--r--   1 ****  staff  1167 10 26  1985 LICENSE
-rw-r--r--   1 ****  staff  4123 10 26  1985 README.md
-rw-r--r--   1 ****  staff  5252 10 26  1985 index.js
drwxr-xr-x   4 ****  staff   128  4 18 20:41 node_modules
-rw-r--r--   1 ****  staff  3514  4 18 20:41 package.json
```

この方法では依存関係が深くなればなる程ディレクトリの階層が増えてしまい、パス名の制限に引っかかる可能性が出てくる。
その上パッケージの共通化がされていないので、全く同じパッケージが node_modules 配下に複数作成される可能性もあった。
(公式では lodash を使った例が挙げられているが、プロジェクトが 10 個のパッケージに依存していて、更にそれぞれのパッケージが lodash に依存している場合、node_modules 配下には全く同じ lodash ディレクトリが少なくとも 10 個は作られることになる。)

これはインストール時にパフォーマンスに悪影響となるだけでなく、不必要にディスク容量を食い潰してしまう。
ここで npm@3 や yarn が取った方法は node_modules 配下をフラットにすることであった。
つまり express の依存パッケージを node_modules 直下に配置してしまうのだ。

```
node_modules % ls -l
total 0
drwxr-xr-x   7 ****  staff  224  4 18 00:00 accepts
drwxr-xr-x   6 ****  staff  192  4 18 00:00 array-flatten
...省略
drwxr-xr-x   8 ****  staff  256  4 18 00:00 express
...省略
drwxr-xr-x   7 ****  staff  224  4 18 00:00 media-typer
drwxr-xr-x   7 ****  staff  224  4 18 00:00 merge-descriptors
drwxr-xr-x   7 ****  staff  224  4 18 00:00 methods
drwxr-xr-x  11 ****  staff  352  4 18 00:00 mime
drwxr-xr-x   8 ****  staff  256  4 18 00:00 mime-db
drwxr-xr-x   7 ****  staff  224  4 18 00:00 mime-types
drwxr-xr-x   6 ****  staff  192  4 18 00:00 ms
drwxr-xr-x   8 ****  staff  256  4 18 00:00 negotiator
...省略
drwxr-xr-x   7 ****  staff  224  4 18 00:00 vary
```

では express ディレクトリ配下はどうなっているだろうか？

```
node_modules % ls -l express
total 264
-rw-r--r--   1 ****  staff  113107  4 18 00:00 History.md
-rw-r--r--   1 ****  staff    1249  4 18 00:00 LICENSE
-rw-r--r--   1 ****  staff    5419  4 18 00:00 Readme.md
-rw-r--r--   1 ****  staff     224  4 18 00:00 index.js
drwxr-xr-x  10 ****  staff     320  4 18 00:00 lib
-rw-r--r--   1 ****  staff    2624  4 18 00:00 package.json
```

express ディレクトリ下にはファイルが列挙されているが、express が依存しているパッケージの姿が見えない。
その理由は express が何にも依存せずにスタンドアロンで動作するからではなく、依存関係が別の場所で解決されているからだ。
現に package.json を見てみれば dependencies が記載されていることが分かる。

```
node_modules % cat express/package.json
{
  "name": "express",
  "description": "Fast, unopinionated, minimalist web framework",
  "version": "4.18.2",
  "author": "TJ Holowaychuk <tj@vision-media.ca>",
  ...省略
  "dependencies": {
    "accepts": "~1.3.8",
    "array-flatten": "1.1.1",
    ...省略
    "merge-descriptors": "1.0.1",
    "methods": "~1.1.2",
    "on-finished": "2.4.1",
    ...省略
    "vary": "~1.1.2"
  },
  "devDependencies": {
    "after": "0.8.2",
    ...省略
    "vhost": "~3.0.2"
  }
}
```

では引き続き express の dependencies に列挙されているパッケージについても見ていこう。ここでは一例として accepts パッケージを取り上げてみたい。
node_modules 直下の accepts フォルダには以下のファイルが含まれており、やはり accepts が依存するパッケージの姿は見えない。

```
accepts % ls
HISTORY.md	LICENSE		README.md	index.js	package.json
```

ここで、package.json の中身を見てみると以下のようになっている。

```
accepts % cat package.json
{
  "name": "accepts",
  "description": "Higher-level content negotiation",
  "version": "1.3.8",
  ...省略
  "dependencies": {
    "mime-types": "~2.1.34",
    "negotiator": "0.6.3"
  }
}
```

依存パッケージは mime-types と negotiator だ。この二つのライブラリはどちらも express からは直接参照されていない。
にも関わらず、node_modules 直下にはしっかりと両者のフォルダが存在している。
このように、依存関係にある dependencies のパッケージを全て node_modules に配置するのが npm@3 以降(そして yarn も)の管理方法である。
この方法のデメリットとして挙げられるのは、依存関係の解決が難しいことと、直接の依存関係がないパッケージも参照できてしまうことである。
更に、npm@2 よりも軽減されたとはいえプロジェクト毎にパッケージをダウンロードしてくることには変わらないので、プロジェクトの数が増えるほどディスク容量を圧迫することになる。
これらの課題をリンクによって解決しているのが pnpm になる。

## pnpm について

前置きが長くなったが、早速 pnpm について見ていこう。pnpm が採用した手法は npm や yarn とは若干異なる。まず、node_modules 配下には直接依存する express と、そして.pnpm ディレクトリだけが存在している。
ここで注意してほしいのは、express が依存するパッケージの姿が見えないこと、そして express 自体が.pnpm ディレクトリ内部の express へのシンボリックリンクとなっていることである。

```
node_modules % ls -al
total 8
drwxr-xr-x   5 ****  staff   160  4 17 08:32 .
drwxr-xr-x   5 ****  staff   160  4 17 08:32 ..
-rw-r--r--   1 ****  staff  2863  4 17 08:32 .modules.yaml
drwxr-xr-x  61 ****  staff  1952  4 17 08:32 .pnpm
lrwxr-xr-x   1 ****  staff    41  4 17 08:32 express -> .pnpm/express@4.18.2/node_modules/express

```

.pnpm ディレクトリ内部には express と、その依存するパッケージが列挙されている。
依存パッケージは一見フラットに配置されているように見えるが、これには実はカラクリがある。これについては後程詳しく説明したい。

```
.pnpm % ls -al
total 32
drwxr-xr-x  61 ****  staff   1952  4 17 08:32 .
drwxr-xr-x   5 ****  staff    160  4 17 08:32 ..
drwxr-xr-x   3 ****  staff     96  4 17 08:32 accepts@1.3.8
drwxr-xr-x   3 ****  staff     96  4 17 08:32 array-flatten@1.1.1
...省略
drwxr-xr-x   3 ****  staff     96  4 17 08:32 express@4.18.2
...省略
drwxr-xr-x   3 ****  staff     96  4 17 08:32 media-typer@0.3.0
drwxr-xr-x   3 ****  staff     96  4 17 08:32 merge-descriptors@1.0.1
drwxr-xr-x   3 ****  staff     96  4 17 08:32 methods@1.1.2
drwxr-xr-x   3 ****  staff     96  4 17 08:32 mime-db@1.52.0
drwxr-xr-x   3 ****  staff     96  4 17 08:32 mime-types@2.1.35
drwxr-xr-x   3 ****  staff     96  4 17 08:32 mime@1.6.0
drwxr-xr-x   3 ****  staff     96  4 17 08:32 ms@2.0.0
drwxr-xr-x   3 ****  staff     96  4 17 08:32 ms@2.1.3
drwxr-xr-x   3 ****  staff     96  4 17 08:32 negotiator@0.6.3
...省略
```

一旦 express 内部を参照してみよう。

```
express@4.18.2 % ls -al
total 0
drwxr-xr-x   3 ****  staff    96  4 17 08:32 .
drwxr-xr-x  61 ****  staff  1952  4 17 08:32 ..
drwxr-xr-x  34 ****  staff  1088  4 17 08:32 node_modules
```

驚くべきことに node_modules しか存在していない。更に node_modules を掘って行ってみよう。

```
node_modules % ls -al
total 0
drwxr-xr-x  34 ****  staff  1088  4 17 08:32 .
drwxr-xr-x   3 ****  staff    96  4 17 08:32 ..
lrwxr-xr-x   1 ****  staff    40  4 17 08:32 accepts -> ../../accepts@1.3.8/node_modules/accepts
lrwxr-xr-x   1 ****  staff    52  4 17 08:32 array-flatten -> ../../array-flatten@1.1.1/node_modules/array-flatten
...省略
drwxr-xr-x   8 ****  staff   256  4 17 08:32 express
...省略
lrwxr-xr-x   1 ****  staff    34  4 17 08:32 vary -> ../../vary@1.1.2/node_modules/vary
```

これで分かるように、.pnpm/express@4.18.2/node_modules には、express が依存するパッケージへのシンボリックリンクが貼られている。
唯一シンボリックリンクが貼られていないのは express フォルダだけだ。
pnpm では自身を除く依存関係の解決にシンボリックリンクが利用されている。では express フォルダの中身はどうなっているだろう。

```
express % ls -ail
total 264
21769618 drwxr-xr-x   8 ****  staff     256  4 17 08:32 .
21768897 drwxr-xr-x  34 ****  staff    1088  4 17 08:32 ..
21768491 -rw-r--r--   2 ****  staff  113107  4 17 08:32 History.md
21768395 -rw-r--r--   2 ****  staff    1249  4 17 08:32 LICENSE
21768490 -rw-r--r--   2 ****  staff    5419  4 17 08:32 Readme.md
21768396 -rw-r--r--   2 ****  staff     224  4 17 08:32 index.js
21769626 drwxr-xr-x  10 ****  staff     320  4 17 08:32 lib
21768419 -rw-r--r--   2 ****  staff    2624  4 17 08:32 package.json
```

一見して普通のディレクトリに見えるが、実際にはこれらのファイルは pnpm がグローバルに保存している express へのハードリンクになっている。

```
express % find ~ -inum 21768491
~/Library/pnpm/store/v3/files/c9/c78a034e659e6055722da3d5bcda4d587645b7e72440318ab516ff7cb5e444d536c02169997848a5b8a1bc0ae44e9d80568a2b60abce9e6b78dbad0aa73c49
~/省略/node_modules/.pnpm/express@4.18.2/node_modules/express/History.md
```

つまるところ、node_modules 配下にはパッケージの実態が一切存在しないのだ。その全てはシステムに一つだけ存在する実パッケージへのハードリンクであり、パッケージ間の依存関係はシンボリックリンクによって解決されている。
node はファイルの種類がシンボルリンクやハードリンクかどうかについては頓着しないので、このような実装が可能になっているようだ。

上記の方法により、pnpm はプロジェクト毎に依存パッケージをダウンロードしてくる必要がなくなり、ディスク容量を削減できる。
加えて、インストールに際してもリンクを貼る時間は、ファイルをダウンロードしてくる時間よりもはるかに短く済むので、パフォーマンス上でも有利となる。

## 索引

この記事を書くにあたって、以下のサイトを参考にさせて頂いた。

[Why should we use pnpm?](https://www.kochan.io/nodejs/why-should-we-use-pnpm.html)
