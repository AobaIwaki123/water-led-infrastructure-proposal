# 水上LED演出システム インフラ提案

レビュー用に切り出した独立資料です。`ai-workspace` 内の調査経緯を踏まえつつ、共有時に読む範囲を次の3資料へ絞っています。

1. [自宅Kubernetes案](docs/01-home-kubernetes.md)
2. [クラウド完結案](docs/02-cloud-complete.md)
3. [比較・認証・基礎インフラ](docs/03-comparison-and-basics.md)

## レビューの入口

GitHubへ公開した後は、Issueをレビューの入口にします。Issue本文の下書きは[こちら](issues/001-review.md)です。Issueでは採用案、認証方式、DNS、Nginx、実装前検証の判断だけを行い、構成の細部はこのrepoのMarkdownを参照します。

## ここまでの経緯

- 約20台の水上LED機器を、会場のBluetooth SIG Meshとクラウド側の制御基盤から動かす前提を整理した。
- 自宅Kubernetes、EC2、ECS、IoT Core、Lambda、RDB、DynamoDB KVSを比較した。
- 比較対象を増やし続けるとレビューしにくくなるため、最終提案を「自宅Kubernetes基盤」と「AWSクラウド完結」の2案へ絞った。
- AWSクラウド案では、ECS + 自前Device Channel、S3 + CloudFront、ALB、RDBを基本構成とした。
- 認証は管理者と観客を分け、管理者はDiscord OAuth2/OIDC、観客はイベント単位の短期ゲストセッションを第一候補とした。

## 共通前提

- 現地にクラウドとBluetooth SIG Meshを橋渡しするVenue Gatewayを置く。
- Web ClientはLED機器へ直接接続せず、Control ServerとDevice Channelを経由する。
- ゲートウェイとクラウドの通信方式はTLS付きのHTTPSまたはWebSocketを候補とし、会場側から外向きに接続する。
- 演出の細かな実行は、通信断に備えて会場側で継続できる構成を候補とする。
- 機器型番、SIG Meshのモデル、命令フォーマット、必要遅延、通信断時の厳密な挙動は実装前検証で決める。

## この資料で決めないこと

このrepoはインフラ配置と選択肢のレビュー資料です。Bluetooth SIG Meshのパケット仕様、演出アルゴリズム、機器の型番、最終的な可用性・遅延目標までは確定しません。
