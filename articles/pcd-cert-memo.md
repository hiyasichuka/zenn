---
title: "Google Cloud Professional Cloud Developer を（再）取得したい"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Google Cloud", "GKE", "Kubernetes"]
published: true
---
# はじめに

3年前ぐらいに取得した「Professional Cloud Developer」認定資格の有効期限が切れてしまったため、再取得を目指します。本記事は、Google が提示した試験範囲に沿って、理解を深めるために調査・整理した知識を体系的にまとめました。

# Professional Cloud Developerの試験範囲

公式ドキュメントに記載されている試験範囲は以下の通りです（2025年5月時点）。

1. スケーラビリティ、可用性、信頼性に優れたクラウドネイティブ アプリケーションの設計
2. アプリケーションのビルドとテスト
3. アプリケーションのデプロイ
4. アプリケーションと Google Cloud サービスの統合

---

## 第1章: スケーラビリティ、可用性、信頼性に優れたクラウドネイティブ アプリケーションの設計

### 主な観点

#### Kubernetesの基本構成要素

- **Deployment**：ステートレスなアプリケーションを複数のPodで管理するためのリソース。
- **StatefulSet**：ステートフルなアプリケーション（例：DBなど）を状態付きで管理。Pod名が固定され、順序や永続ディスクの扱いも考慮される。
- **Service**：Pod へのアクセスを抽象化。内部通信や外部公開を制御するためのエンドポイント。
- **ConfigMap**：環境変数や設定ファイルなどを外部化してPodに注入。

#### セキュリティ設計

- **Workload Identity**：GKE 上の Pod に Google サービスアカウントを割り当てることで、鍵を使わずに Google Cloud API にアクセスできる仕組み。
- **NetworkPolicy**：Kubernetes 上で Pod 間通信を制限するファイアウォール的な設定。
- **mTLS（相互TLS）**：Istio などのサービスメッシュを使って、Pod 間の通信を暗号化し、かつ相互に認証させる。

#### スケーラビリティ設計

- **水平スケーリング**：Podの数を増やすことでスケーラビリティを担保（HPA: Horizontal Pod Autoscaler）。
- **垂直スケーリング**：1つのPodのCPUやメモリ割り当てを変更する（VPA: Vertical Pod Autoscaler）。

#### チーム分離と権限制御

- **Namespace**：チームごとに論理的にリソースを分離できる単位。CI/CD、監査、ログ収集も分けられる。
- **RBAC（Role-Based Access Control）**：特定のユーザーやサービスアカウントに対し、操作できるリソースを定義する。

#### IAMと認証設計

- **最小権限の原則（Least Privilege）**：不必要な権限を与えず、必要な操作のみ許可する。
- **OAuth / JWT / サービスアカウント**：アクセス先や用途に応じて認証方式を使い分ける。

#### 秘匿情報の管理

- **Secret Manager**：APIキーや認証情報などの秘匿情報を安全に保存・取得。
- **Cloud KMS**：暗号鍵の生成・管理を一元化し、セキュアに暗号化・復号化を実現。

#### Cloud Storageの運用とセキュリティ

- **保持ポリシー（Retention Policy）**：バケット内のオブジェクトが一定期間削除されないよう制御。コンプライアンス対策に有効。
- **ライフサイクルポリシー**：アクセス頻度や経過日数に応じて、オブジェクトの削除やアーカイブを自動化。
- **Cloud Storage FUSE**：Cloud Storage バケットをローカルファイルのようにマウントしてアクセス。Pod や VM から使えるが、並列性能やレイテンシに注意。
- **限定公開 Google アクセス（Private Google Access）**：外部IPを持たないGKEノードやVMからGoogle Cloud APIへセキュアに接続する手段。NATなしでもアクセス可。

### 理解チェック

#### Deployment と StatefulSet の違い  
Deployment はステートレスなサービスに向いており、Pod名は自動で割り当てられる。StatefulSet は状態を持つアプリ（DBなど）に適し、Pod名・順序・永続ディスクが維持される。

#### Pod 間のセキュアな通信を担保する方法  
NetworkPolicy で許可した通信のみを許可。Istio などを併用すると mTLS による暗号化・認証も実現できる。

#### Workload Identity を使う理由と利点  
秘密鍵不要で、GCP サービスアカウントと Kubernetes のサービスアカウントをマッピングできる。運用の安全性と柔軟性が高まる。

#### Cloud Storage でコンプライアンスを担保するには？  
バケットに保持ポリシーを設定することで、一定期間データの削除を防ぎ、監査対応や社内規定の順守に寄与できる。

#### Cloud Storage FUSE を使う利点と注意点は？  
Cloud Storage バケットをファイルのように扱えるが、読み書き性能はローカルディスクと異なるため、大量I/O用途では工夫が必要。

#### 限定公開 Google アクセスとは？  
外部IP不要で Google API へアクセスできるネットワーク機構。セキュリティ制限の厳しいネットワークでも API 呼び出しを可能にする。

※ 参考文献 
- [Professional Cloud Developer 認定ガイド（Google Cloud）](https://cloud.google.com/certification/guides/cloud-developer)  
- [Workload Identity 公式ガイド](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)  
- [Cloud Storage FUSE ドキュメント](https://cloud.google.com/storage/docs/gcs-fuse)  
- [Private Google Access 公式ページ](https://cloud.google.com/vpc/docs/private-google-access)
---

## 第2章: アプリケーションのビルドとテスト

### 主な観点

#### 開発ツールの活用

- **Cloud Shell**  
  ブラウザ上で動作する Google 提供の開発環境。gcloud CLI、kubectl、Terraform、Docker などがプリインストールされており、即座に開発・検証可能。

- **Cloud Code**  
  VS Code / IntelliJ 用のプラグイン。GKE や Cloud Run のマニフェスト補完、デプロイ支援、デバッグ、ログ確認などが IDE 上で完結。

#### CI/CD

- **Cloud Source Repositories（CSR）**  
  GCPが提供するGit互換のリポジトリ。Cloud Buildと簡単に連携できる。

- **Cloud Build**  
  ビルドステップを `cloudbuild.yaml` で定義し、Docker イメージのビルドやユニットテスト、デプロイ等を自動化できる。GitHub Actions など他のCIツールとの連携も可能。

- **ビルド構成ファイル（cloudbuild.yaml）**  
  各ステップをコンテナベースで定義でき、並列実行や条件分岐もサポート。

#### セキュアなビルドパイプラインの構築

- **Artifact Registry**  
  ビルドされたコンテナイメージやアーティファクトの保管場所。Vulnerability Scanning により既知の脆弱性をチェック可能。

- **Binary Authorization**  
  許可された署名済みイメージだけが Kubernetes 等にデプロイ可能となるよう検証するセキュリティレイヤ。SLSA や SBOM の整合性担保にも活用される。

#### テスト戦略の実装

- **ローカルテスト環境**  
  Cloud Functions Emulator や Cloud Pub/Sub Emulator を使って、実環境に近い形での単体テストをローカル実行。

- **負荷テスト（Load Testing）**  
  Kubernetes 上で [Locust](https://locust.io/) を実行し、HTTPベースのワークロードに対してスケーラブルな負荷テストを実施可能。

- **CI環境での自動テスト**  
  Cloud Build や GitHub Actions に統合することで、テスト失敗時の即時フィードバックを実現。

### 理解チェック

#### Cloud Build を使った CI/CD パイプラインのメリット
一貫したビルド環境でセキュアかつ拡張性のあるジョブを定義できる。ステップを明示的に管理でき、複数のビルド条件や通知とも連携可能。

#### Cloud Build の構成ファイル（cloudbuild.yaml）の基本構造  
各ステップは Docker イメージで定義され、順序付きの `steps` ブロックと、出力先 Artifact Registry を指定する `images` ブロックが基本。

#### Artifact Registry での脆弱性検出の仕組み  
Container Analysis API を通じて、既知のCVE情報に基づいた脆弱性がスキャンされる。セキュリティセンターと連携も可能。

#### Binary Authorization の導入目的と仕組み  
CI パイプラインで署名済みコンテナのみを本番にデプロイさせることで、信頼できるソフトウェア供給のみを許可。ポリシーで検証を制御可能。

#### Cloud Functions Emulator の用途  
実際の Cloud Functions にデプロイせずにローカルで関数の実行検証ができる。Cloud Pub/Sub や Firestore と組み合わせたテストにも有用。

#### Locust による負荷テスト
Python ベースでシナリオを定義し、分散実行も可能な負荷テストツール。Kubernetes と組み合わせることで大規模な並列テストも可能。

※ さんこうぶn 
- [Cloud Build ドキュメント](https://cloud.google.com/build/docs)  
- [Binary Authorization 公式ページ](https://cloud.google.com/binary-authorization)  
- [Artifact Registry の脆弱性スキャン](https://cloud.google.com/artifact-registry/docs/vulnerability-scanning)  
- [Cloud Functions Emulator](https://github.com/googleapis/google-cloud-node/tree/main/packages/functions-emulator)  
- [Locust - Scalable Load Testing Tool](https://locust.io/)

## 第3章: アプリケーションのデプロイ

### 主な観点

#### デプロイ方式の理解と選択

- **インプレースデプロイ**  
  既存のアプリケーションを停止し、新しいバージョンを上書きするシンプルな方式。ただしダウンタイムが発生するリスクが高い。

- **ローリングアップデート**  
  徐々に旧バージョンから新バージョンに入れ替えていく方式。Kubernetes では標準でサポートされており、可用性を保ちながら安全にアップデートできる。

- **カナリアデプロイ**  
  一部のトラフィックだけを新バージョンへルーティングし、問題がないか確認してから段階的に切り替える手法。予期せぬ障害の早期検知が可能。

- **Blue-Green デプロイ**  
  本番環境とは別に新バージョンをデプロイし、切り替えタイミングを完全に制御する方式。切り戻しも迅速に行えるのが利点。

#### Google Cloud におけるトラフィック分割・段階的リリース

- **Cloud Run**  
  コンテナを簡単にデプロイでき、バージョンごとにトラフィックを任意の比率で分割可能。ローリング、カナリア、Blue-Green すべてに応用可能。

- **App Engine**  
  バージョン単位でトラフィック配分が可能。ステージング環境として使いつつ、本番切替もワンクリック。

- **GKE + Istio / ASM (Anthos Service Mesh)**  
  Istio の VirtualService や DestinationRule を用いることで、HTTP レベルでのトラフィック制御が可能。Header などを使ったユーザーセグメントへのルーティングも対応。

#### テスト戦略と検証手法

- **A/Bテスト**  
  異なるバージョンのアプリをユーザーごとに分けて提示し、効果を比較するテスト方式。Istio のルーティング制御と連携しやすい。

- **シャドーテスト**  
  新バージョンに実トラフィックをコピーして流すが、ユーザーには応答を返さない方式。本番相当の負荷で安全に検証が可能。

- **カナリアテスト**  
  実トラフィックのごく一部を新バージョンに割り当てる。モニタリングと併用し、エラー率やレイテンシを監視して判断。

#### API公開と互換性管理

- **Cloud Endpoints**  
  OpenAPI仕様でAPIを定義し、デプロイ時にバージョン管理やトラフィック制御が可能。認証（APIキーやJWT）やレート制限も対応。

- **APIのバージョニング**  
  `/v1/` や `/v2/` といった形式でバージョンを管理しつつ、古いクライアントを破壊しないように運用することが重要。

- **APIゲートウェイの導入**  
  外部公開時の統一エントリポイントとして、セキュリティと運用の一元化に寄与。

---

### 理解チェック

#### ローリングアップデートとカナリアデプロイの違いは？  
ローリングアップデートは自動的に全体を段階的に入れ替える方式、カナリアは一部トラフィックのみを新バージョンに送り段階的に判断しながら切り替える。

#### Cloud Run や App Engine のトラフィック分割の用途  
段階的なリリースや A/B テスト、カナリアリリースに活用可能。GUIやCLIで比率変更も容易。

#### Blue-Greenデプロイの利点と注意点  
切り戻しが即座にできる点が強み。ただしコストやインフラ負荷が高まる可能性があるため、運用設計が必要。

#### シャドーテストの活用シーン  
実トラフィックを使って動作検証したいが、ユーザー影響を避けたいとき。ログ解析やモニタリングとセットで使うと有効。

#### Cloud Endpoints の役割  
APIの管理と保護を行う仕組み。API キーや JWT による認証、リクエストのレート制限、モニタリング、ログ収集などを一元的に行うことができ、API の不正利用を防げる。バージョン管理やアクセス制御も可能で、チームや組織全体での API 利用を安全かつ効率的に統制できる。

---

※ 参考文献  
- [Cloud Run トラフィック分割](https://cloud.google.com/run/docs/deploying#splitting-traffic)  
- [Blue-Green / カナリアデプロイの比較](https://cloud.google.com/architecture/deploying-applications-blue-green-canary-rollouts)  
- [Cloud Endpoints 公式ドキュメント](https://cloud.google.com/endpoints/docs)  
- [Istio Traffic Management](https://istio.io/latest/docs/tasks/traffic-management/)

## 第4章: アプリケーションと Google Cloud サービスの統合

### 主な観点

#### イベントドリブンな統合

- **Eventarc**  
  Cloud Storage や Firestore、Pub/Sub、Audit Logs などのイベントをトリガーとして、Cloud Run などのワークロードを自動的に起動できる。Cloud Functions の後継的存在としてより多様なイベントソースをサポートし、トレーサビリティやセキュリティの強化も可能。

- **Cloud Pub/Sub**  
  メッセージング基盤として、非同期処理や分散システム間のイベント連携に活用される。多対多の配信や push/pull モデルに対応し、信頼性・スケーラビリティに優れる。

- **Cloud Tasks**  
  非同期タスクの管理に特化したサービス。タスクの順序制御、リトライポリシー、最大同時実行数の制御が可能で、スケジューラとしても活用できる。

- **Cloud Workflows**  
  API呼び出しやCloud Function実行などをワークフローとして定義・連携可能。エラー制御や待機処理も含めたプロセス全体を視覚的に制御したい場合に便利。

#### サーバーレス / コンテナサービスとの統合

- **Cloud Functions / Cloud Run**  
  イベントドリブンまたはHTTPベースのマイクロサービス実行に最適。スケーラブルで疎結合な構成を取れる。Eventarc, Pub/Sub, Cloud Storage などと連携可能。

- **App Engine**  
  HTTP アプリケーションを完全マネージドで運用可能。アプリケーションのスケーリング、ログ管理、バージョン管理が自動化されている。

#### データサービスとの連携

- **Firestore / Cloud SQL / Spanner**  
  各種データベースとアプリケーションの統合。トランザクション設計や接続管理（プール、認証）に注意。Cloud SQL は JDBC/MySQL 互換、Spanner はスケーラブルなRDB。

- **BigQuery**  
  分析系に特化した統合先。クエリの実行やストリーミング挿入、MLモデルの推論呼び出しまで可能。アプリからの直接呼び出しは IAM やコスト管理を意識すべき。

- **Cloud Storage**  
  ファイルの読み書き、署名付きURLによる一時アクセス発行、イベント連携（Eventarc、Cloud Functions）など多用途に活用される。

#### API管理とゲートウェイ

- **API Gateway**  
  REST API への共通認証、認可、モニタリングを一元管理。Cloud Run や App Engine、Cloud Functions をバックエンドとしたマイクロサービス連携に向く。

- **Cloud Endpoints**  
  OpenAPI や gRPC ベースで API を定義・管理可能。API キー認証、OAuth、レート制限、ログ追跡などを組み込み、API の品質・信頼性を担保できる。

#### セキュリティと監査

- **IAM の活用**  
  各統合に際して、サービスアカウントやWorkload Identityにより必要最小限のアクセス許可を設定。信頼できるアプリケーション単位で制御を行う。

- **Cloud Logging / Monitoring / Audit Logs**  
  API 呼び出しやイベント実行のログを Cloud Logging で取得。システム健全性やエラー検出は Cloud Monitoring で行う。監査目的には Audit Logs が不可欠。

---

### 理解チェック

#### Eventarc の強みとは？  
多様な GCP イベントソースから Cloud Run などに自動でイベントを送信でき、サーバレスかつトレーサブルな処理が可能。複雑なイベントルーティングにも対応。

#### Pub/Sub と Cloud Tasks の違い  
Pub/Sub は「イベントの通知と購読」に特化し、多数のコンシューマへブロードキャスト可能。Cloud Tasks は「順序性・制御付きの非同期処理」を実現するジョブキュー。

#### Cloud Endpoints の役割とユースケース  
OpenAPI ベースで定義された API に対し、認証・レート制限・トラフィック管理を提供。API の仕様変更やアクセス制御を統一的に扱える。

#### Cloud Functions vs Cloud Run  
Functions は関数単位で、Run はコンテナベースで柔軟な開発・デプロイが可能。HTTPやPub/Subイベントへの対応において、Cloud Runはより複雑なワークロードにも対応可能。

#### アプリケーションから BigQuery に安全に書き込むには？  
BigQuery Storage Write API を使い、サービスアカウントに最小限の権限を付与。ストリーミング挿入時はスキーマ整合性とコストにも注意。

---

※ 以下の情報を参考にしています  
- [Eventarc 公式ガイド](https://cloud.google.com/eventarc/docs)  
- [Cloud Pub/Sub ドキュメント](https://cloud.google.com/pubsub/docs)  
- [Cloud Tasks ドキュメント](https://cloud.google.com/tasks/docs)  
- [Cloud Endpoints ドキュメント](https://cloud.google.com/endpoints/docs)  
- [API Gateway 公式ページ](https://cloud.google.com/api-gateway)  
- [BigQuery Storage Write API](https://cloud.google.com/bigquery/docs/write-api)

#### Cloud Pub/Sub と Cloud Tasks の違い  
Cloud Pub/Sub は「Pub/Subモデル」でイベントを多数のコンシューマに送るのに向いており、Cloud Tasks は「キューイングモデル」で順序性と制御性を持って非同期タスクを処理できる。

#### Cloud Functions で Cloud Storage をトリガーにするメリットは？  
ファイルアップロードなどの操作をトリガーに自動処理を実行でき、監視や後処理が容易に行える。スケーラブルでイベント駆動の仕組みに最適。

#### BigQuery へのリアルタイム連携方法  
BigQuery Storage Write API を使えば、ストリーミングで高速かつスキーマ整合性を保ったデータ書き込みが可能。

#### API Gateway と Cloud Endpoints の使い分け  
API Gateway は Cloud Run/Functions と統合する最新の方式で、Cloud Endpoints は OpenAPI による仕様管理や認証に強みがある。用途と管理レベルに応じて選択。

#### Cloud Monitoring と Cloud Logging の違い  
Logging はログデータの収集・分析、Monitoring は指標の可視化やアラート通知を担当。両者を統合的に使うことで運用の可視性が向上する。

※ 参考文献
- [Cloud Pub/Sub ドキュメント](https://cloud.google.com/pubsub/docs)  
- [Cloud Tasks 公式ガイド](https://cloud.google.com/tasks/docs)  
- [Cloud Functions イベントトリガー](https://cloud.google.com/functions/docs/calling)  
- [BigQuery Storage Write API](https://cloud.google.com/bigquery/docs/write-api)  
- [API Gateway 公式ページ](https://cloud.google.com/api-gateway)  
- [Cloud Monitoring と Cloud Logging の概要](https://cloud.google.com/products/operations)

## 最後に

本記事が、試験対策で役立つものであれば幸いです。
ここまで読んでいただきありがとうございました！