---
title: "DevOpsやSREの面接対策・実践的な技術問題を学べる神リポジトリ devops-exercises"
type: "tech"
topics: ["interview", "devops", "sre", "docker"]
published: true
---

## 想定読者

- **「DevOps や SRE の面接対策って何をやればいいの？」** という人
- DevOps/SRE 初学者〜中級者
- DevOps/SRE関連の技術を総復習したい人

## はじめに

https://github.com/bregman-arie/devops-exercises

↑こちらのリポジトリご存知でしょうか？

- 技術面接でよく出る質問と回答
- DevOps/SRE の実践課題
- 各種クラウドやツールの学習素材

これらが詰まった神リポジトリです。
この記事では、実際に触ってみた解説記事です。

## リポジトリの全体像


```
devops-exercises/
├── certificates/        # 資格試験対策
├── exercises/           # 実践課題（Shell/Python）
├── topics/              # 技術カテゴリ別の質問集
└── prepare_for_interview.md  # 面接対策ガイド
```

特に **「topics/」** が充実しており、`Linux、Docker、Kubernetes、AWS、GCP、Terraform`など、幅広いトピックが網羅されています。

以下では、いくつかピックアップして紹介します。

## topics 触ってみた

### Linuxコマンド

`topics/linux.md` から抜粋します。

**Q. ファイルの行数を数えるコマンドは？**

```bash
wc -l filename
```


**Q. findコマンドで拡張子が .log のファイルを検索するには？**

```bash
find . -type f -name "*.log"
```

いわゆる定番問題です。

## C言語

**Q. yayは何回出力される？**

```c
#include<stdio.h>
#include <unistd.h> 

int main()
{
  fork();
  fork();
  printf("\nyay\n");
  return 0;
}
```

`fork()` が2回呼ばれ、プロセス数は 1 → 2 → 4 に増加。
各プロセスが1回ずつ `printf()` を実行するため、 **4回**

Linuxのfork()システムコールに関する典型的な面接問題ですね。

## AWSやKubernetes,Terraform もカバー

**topics** には以下のような技術領域もカバーされています。

- AWS（EC2, S3, IAM, CloudWatch など）
- Kubernetes（Pod, Deployment, Service の動き方）
- Terraform、Ansible、Prometheus
- SRE（SLI/SLO/SLA）

実務で触れる内容を幅広く学習できます。


## 面接準備ガイドも良い

リポジトリ直下の[prepare_for_interview.md](https://github.com/bregman-arie/devops-exercises/blob/master/prepare_for_interview.md)には、面接準備の心得とか書かれています。
下記にチェックリストとして整理しておきます。

### DevOps/SRE 面接対策チェックリスト

#### 1. Linuxとプログラミングは必須

- [ ] Linuxの基本操作（コマンド、シェル、ファイルシステム）
- [ ] OS、デバッグ、ネットワークの理解
- [ ] トラブルシュート・パフォーマンスチューニングの経験
- [ ] 得意な言語でのスクリプト作成（Bash/Pythonなど）
- [ ] 自動化・ツール開発・CLIツールを自作した経験
- [ ] HackerRank / LeetCode / Exercism で実践練習

#### 2. 設計力・ツール理解・シナリオ対応力

- [ ] CI/CDパイプライン設計を説明できる
- [ ] ケーラブルなログ基盤の設計経験
- [ ] マイクロサービスやインフラ設計の具体例を話せる
- [ ] 自分が使っているツールについて「なぜ使うか・どう動くか」を説明できる
- [ ] 求人票を元にシナリオ（例：サーバープロビジョニング、スクリプト作成）で練習
- [ ] 自作DevOpsプロジェクト（ポートフォリオ）を1つ以上作っている

#### 3. 自己分析 × 企業理解

- [ ] 企業の事業内容・プロダクト・強みを調査
- [ ] 面接時に聞きたい逆質問を準備している（チーム規模、WLB、成長機会など）
- [ ] 模擬質問リストを作り、回答練習
- [ ] 友人・同僚に模擬面接
- [ ] HRや採用担当に「面接フローや重点ポイント」を確認
- [ ] 面接は「自分も会社を見極める場」として臨む


## おわりに

「DevOps/SRE、どこから手を付けていいか…」という人は、**このリポジトリを上から順番にやるだけでも相当レベルアップ** できそうです。
