# 2025BeginnerTrainingGit
株式会社MIXI 新卒向けGit研修リポジトリ

Gitの内部構造の説明に使用します。
本リポジトリをcloneした後に、以下の手順で pack ファイルを展開してください。

## 手順

### 前提

- 作業はリポジトリのルート（`2025BeginnerTrainingGit`）から始める。
- `temp` はリポジトリの**中**に作る。そのため、元のリポジトリは `..`（1つ上）で指す。
- 元の手順の `../2025BeginnerTrainingGit/...` は、`temp` をリポジトリの外に作る場合のパスで、中に作ると存在しない。

### 1. 作業用フォルダを作り、Git リポジトリにする

```bash
mkdir temp && cd temp
git init
```

- `git init` を忘れると、`temp/.git/objects` が存在せず、後の手順が失敗する。
- `git int` のような打ち間違いに注意。正しくは `git init`。

### 2. pack ファイルの名前を調べる

```bash
ls ../.git/objects/pack/
```

今回の出力は次のとおり。

```
pack-aae0462828ebfb59e04b5fdcf70ef0aaba421676.idx
pack-aae0462828ebfb59e04b5fdcf70ef0aaba421676.pack
pack-aae0462828ebfb59e04b5fdcf70ef0aaba421676.rev
```

使うのは `.pack` ファイル。元の手順の `pack-xxx.pack` の `xxx` は、実際のファイル名に置き換える。

### 3. pack を展開する（temp の中で実行）

```bash
git unpack-objects < ../.git/objects/pack/pack-aae0462828ebfb59e04b5fdcf70ef0aaba421676.pack
```

`temp/.git/objects/` の下に、2文字のフォルダ（loose オブジェクト）ができる。

### 4. 展開したオブジェクトを元のリポジトリに移す

```bash
find .git/objects -mindepth 1 -type d -not -name pack -not -name info | xargs -I {} mv {} ../.git/objects/
```

- `pack` と `info` 以外のフォルダを、元の `.git/objects/` に移す。
- `-mindepth 1` は先頭側に置く（後ろに置くと警告が出る）。
- 元の手順の `ind` は `find` の打ち間違い。

### 5. 元のフォルダに戻る

```bash
cd ..
```

### 6. 展開できたか確認する

```bash
ls .git/objects
```

今回の出力は次のとおり。

```
3a  44  94  a5  ab  c7  d4  df  info  pack
```

2文字のフォルダが増えていれば成功。

## つまずいたところ

| 症状 | 原因 | 対処 |
|---|---|---|
| `git: 'int' is not a git command` | `git init` の打ち間違い | `git init` と入力する |
| `No such file or directory`（pack） | パスが違う、`xxx` が未置換 | `../.git/objects/pack/` を指し、実際のファイル名を使う |
| `find: ‘.git/objects’: No such file or directory` | `temp` で `git init` していない | 先に `git init` を実行する |
| `bash: ind: command not found` | `find` の打ち間違い | `find` と入力する |
| `.git` がエクスプローラーに見えない | VS Code の初期設定で非表示 | `ls -a` で確認できる。表示するには設定の `files.exclude` を変更する |

複数のコマンドを一度に貼り付けると、1つ失敗しても次が実行される。1行ずつ実行するとよい。

## 展開したオブジェクトを読む

オブジェクトは圧縮されているので、`cat` では読めない。`git cat-file` を使う。

```bash
git cat-file -t <ID>   # 種類を表示（blob / tree / commit）
git cat-file -p <ID>   # 中身を表示
```

ID は先頭から4文字以上で、他と重ならなければ短縮できる（例：`abf5c28`）。
`.git/objects/ab/` の `ab` はフォルダ名で、ID の先頭2文字。ファイル名は残りの38文字。

### 実行例：commit → tree をたどる

**1. commit を見る**

```bash
git cat-file -t abf5c28c73d56830e3e7cc0722be63d4d32b51a4
git cat-file -p abf5c28
```

```
commit
tree 3a2ccd96b192df80437a06ae7ca9bd2d254ef3fb
parent 44108c00a5b976158a0b5ccff04b8261bfe3d7db
author kenju.moriyama <qujen-caballus> 1744032231 +0900
committer kenju.moriyama <qujen-caballus> 1744032231 +0900

[add]add the second file second-commit.txt
```

| 行 | 意味 |
|---|---|
| `tree` | このコミット時点のルートフォルダの tree の ID |
| `parent` | 1つ前のコミットの ID |
| `author` / `committer` | 書いた人 / コミットした人と日時（Unix 時間と `+0900`） |
| 最後の1行 | コミットメッセージ |

**2. tree を見る**

```bash
git cat-file -p 3a2ccd9
```

```
100644 blob d46017d015af8702f85a62a2eeeb62ca33629e13    README.md
040000 tree 2bceb6cc10ab98b13ef6b3624bc464d5c020eb57    git-internals
```

| 列 | 意味 |
|---|---|
| 1列目 | `100644` は普通のファイル、`040000` はフォルダ |
| 2列目 | `blob`（ファイルの中身）か `tree`（フォルダ） |
| 3列目 | オブジェクトの ID |
| 4列目 | ファイル名・フォルダ名 |

ファイル名やフォルダ名は blob ではなく tree が持つ。
`git-internals` は tree なので、さらに `git cat-file -p 2bceb6c` でたどる。

### つながり

```
commit abf5c28
 ├─ parent: commit 44108c0（1つ前）
 └─ tree 3a2ccd9（ルート）
     ├─ blob d46017d  README.md
     └─ tree 2bceb6c  git-internals → （この中に blob がある）
```

## 履歴をグラフで見る

```bash
git log --oneline --graph --all
```

- `--graph` のつづりに注意（`--glaph` はエラーになる）。
- 出力から抜けるには `q` キーを押す。

## 補足

- 元の pack ファイルは残ったまま。展開したオブジェクトと内容は重複するが、害はない。
- 作業が終わったら、実験用フォルダは `rm -rf temp` で消せる。
- `.git` の中を手で編集・削除するとリポジトリが壊れるおそれがあるので、見るだけにする。