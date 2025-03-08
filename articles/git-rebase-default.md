---
title: "git pull で Your branch and ‘origin/main’ have diverged"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git"]
published: true
---

# 背景
`git pull` した際に、ローカルとリモートの履歴が分岐（diverge）していると、以下のようなエラーが発生する。

```sh
Your branch and ‘origin/main’ have diverged,
and have 1 and 6 different commits each, respectively.
(use “git pull” to merge the remote branch into yours)
```

# ゴール
本ドキュメントでは、この原因の理解し、適切な対処を行えることを目的とする。
また、今後同じ問題が発生しない設定方法も紹介する。

# TL;DR

`git pull` を実行するたびに `rebase` を指定するのは手間なので、デフォルトの挙動を `rebase` に設定する。

```sh
git config --global pull.rebase true
git config --global --get pull.rebase
```
出力が true になっていれば、設定が正しく適用されている。

# 主なエラーの原因

1. **リモートで新しいコミットが追加された**  
   他のメンバーが `main` に新しい変更をプッシュしたが、ローカルの `main` を更新しないまま作業を進めたため、履歴が分岐してしまった。

2. **ローカルで新しいコミットを追加した**  
   自分のローカルで `main` に対してコミットを追加したが、リモートの最新の変更を取り込まずにいたため、ローカルとリモートの履歴が一致しなくなった。

3. **リモートの履歴が書き換えられた**  
   `git rebase` や `git push --force` などによってリモートの履歴が変更された結果、ローカルの履歴と不整合が生じた。

# 解決策

ローカルとリモートの変更を統合するには、３つの方法がある。
それぞれの特徴を理解し、状況に応じて適切な方法を選択する。

| 方法 | メリット | デメリット |
|------|---------|-----------|
| **マージ（merge）**<br>`git pull --no-rebase` | コンフリクトが発生しづらい<br>履歴がそのまま残るため、過去の変更をたどりやすい | マージコミットが増え、履歴が複雑になる |
| **リベース（rebase）【推奨】**<br>`git pull --rebase` | 履歴が整理され、余分なマージコミットが発生しない<br>他のメンバーとの作業履歴が直線的になり、履歴をたどりやすい | コンフリクトが発生した場合、手動で解決する必要がある |
| **ファストフォワードのみ（fast-forward only）**<br>`git pull --ff-only` | 履歴を変更せず、最も安全な方法 | ローカルに変更がある場合、エラーになり `pull` できない |


## マージがコンフリクトを発生しにくい理由

マージは **「共通の祖先」+「ローカルの変更」+「リモートの変更」** の3つを比較し、自動で統合する。  
異なるファイルや異なる行の変更はそのまま適用され、コンフリクトが発生しない。
一方、**同じファイルの同じ行をローカルとリモートで変更がある**場合はコンフリクトする。


# 今後同じ問題が発生しない設定方法
`git pull` を実行するたびに `rebase` を指定するのは手間なので、デフォルトの挙動を `rebase` に設定する。


```sh
git config --global pull.rebase true
```
これにより、今後 git pull を実行する際に、常に rebase が適用される。

設定が適用されたか確認するには、以下のコマンドを実行。
```sh
git config --global --get pull.rebase
```

出力が true になっていれば、設定が正しく適用されている。

# まとめ
- `git pull` 時に `Your branch and 'origin/main' have diverged` エラーが発生するのは、ローカルとリモートの履歴が分岐しているため。
- merge はコンフリクトが発生しにくいが、履歴が複雑になる。
- rebase を使うと履歴が整理され、きれいな状態を維持できるため、推奨。
- 今後この問題を防ぐために `git config --global pull.rebase true` を設定し、デフォルトでrebaseを適用するようにする。
