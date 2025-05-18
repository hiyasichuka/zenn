---
title: "【体験記】Google Cloud Professional Cloud Developer 資格を（再）取得しました"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp","certification"]
published: true
---

# はじめに

Google Cloud の認定資格「[Professional Cloud Developer](https://cloud.google.com/learn/certification/cloud-developer?hl=ja)」を再取得しました。  
本記事では、受験に向けて準備した内容、出題傾向、活用した学習リソース、当日の流れなどをまとめています。  
これから受験される方の参考になれば幸いです。


## 対象読者

- Google Cloud Professional Cloud Developer（PCD）を取得しようとしている方
- PCD試験範囲のキャッチアップや勉強法を知りたい方


## 試験概要（2025年5月時点）

- 資格名：Professional Cloud Developer
- 時間：2時間
- 問題数：50問
- 受験形式：オンライン監視試験（←今回はこの方式で受験）/オンサイト監視試験


## 使用した教材・学習リソース


| 種別 | リンク・資料 |
|------|--------------|
| 公式ガイド | [Professional Cloud Developer ガイド](https://cloud.google.com/learn/certification/cloud-developer?hl=ja) |
| 模擬試験 | 上記の公式ガイド参照 |
| 無料問題集（100題） | [Note記事](https://note.com/aws_shikaku/n/n2e26a3e500a7) |
| 問題演習1 | [CertsTime サンプル問題](https://www.certstime.com/questions/google/professional-cloud-developer-exam) |
| 問題演習2 | [LeetQuiz 練習問題](https://leetquiz.com/certificate/google-professional-cloud-developer/practice) |
| 海外試験共有サイト | [ExamTopics](https://www.examtopics.com/exams/google/professional-cloud-developer/) |
| キュレーションメモ | [Zenn](https://zenn.dev/takaha4k/articles/pcd-cert-memo) |


## 学習中に重視したポイント

### Kubernetes & GKE
- tcpdump は非推奨 → OpenTelemetry / Trace 利用
- ネットワーク・セキュリティ・安全系（k8s secret vs Cloud KMS, Binary Authorization, Proxy）が頻出
- k8s開発（Pod, Deployment, StatefulSet, Minikube / Skaffold / Tekton など）

### Cloud Run / App Engine
- Cloud Run: gRPC対応、最大60分
- Cloud Functions: 実行9分まで
- Traffic分割

### ネットワーク / インフラ設計
- Cloud SQL への接続方法（Cloud SQL Auth Proxy）←セキュアな通信経路周りも頻出です
- Load Balancer のヘルスチェックIP範囲：`130.211.0.0/22`, `35.191.0.0/16`
- Filestore vs Cloud Storage FUSE の違い
- Service Mesh / URLMap による L7 ルーティングの理解
- グローバル外部アプリケーション ロードバランサのトラフィック管理

### CI/CD
- Cloud Build + Cloud KMS + Binary Authorization 組み合わせ
- イメージの脆弱性対応
- デプロイ戦略の違い（Blue/Green, Canary, Rolling）


## リモート監視試験当日の流れ

自宅のデスクトップPCで受験しました。
免許書の写真とって送ったり、デスク周りをカメラで映すなどやることが結構あります。

| 時刻 | 内容 |
|------|------|
| 06:35 | 入室（免許証提示したところ別のIDを提示しろと言われる。パスポート提示でOK） |
| 06:50 | 試験開始（WEBタイマーあるが、目障りなので非表示に設定） |
| 07:40 | 50分で回答を提出して試験終了。その直後、暫定結果で「合格」通知 |


## 印象に残ったテーマ

- **GKE のセキュリティまわり**
- **Cloud SQL プロキシと権限管理（roles/cloudsql.client）**
- **BigQuery Storage Write API（標準/コミットモード）**
- **Minikube, Skaffold などk8s周辺開発ツール**
- **Cloud Run、AppEngineの違い**
- **Firestore**
- **Cloud Workflows**
- **VPC Service Controls**

## 再取得にあたって気をつけたこと

- 実務経験があっても「試験向けの知識整理」は必要
- 権限管理、ネットワーク、セキュリティの問題が頻出するので手厚く学習
- k8sやコンテナ、CICDが頻出するので手厚く学習
- 暗記よりも「構成やサービスの強みを説明できること」が重要
- 新サービスもキャッチアップしておくべし


## 最後に

本記事は、あくまで一個人の体験をまとめたものです。  
内容には最新のアップデートが反映されていない可能性もあるため、**必ず公式ガイドを参照**してください。
試験を受ける方の合格を応援しています！
