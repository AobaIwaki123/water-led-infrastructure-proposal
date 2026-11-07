# 水上LED演出システム インフラ提案

レビュー用に切り出した独立資料です。レビュー時はIssueだけを読み、詳細を確認したい場合だけ文脈別のnoteを参照します。

## レビューの入口

レビューの入口は[GitHub Issue #1](https://github.com/AobaIwaki123/water-led-infrastructure-proposal/issues/1)です。Issue本文には共通前提、2案のMermaid、リソース比較、メリット・デメリット、認証方式、Route 53、Nginx、レビュー項目を含めます。Issueだけで案の比較と判断が完結し、noteは補足資料です。Issue本文のローカル下書きは[こちら](issues/001-review.md)です。

## 詳細note

noteは案ごとに分けず、読みたい論点ごとに分けています。

- [背景・スコープ](notes/00-background-and-scope.md)
- [論理アーキテクチャ](notes/01-logical-architecture.md)
- [実行基盤・リソース](notes/02-platform-and-resources.md)
- [公開入口・静的Web・DNS](notes/03-public-entry-and-static-web.md)
- [認証方式](notes/04-authentication.md)
- [状態・秘密情報・監視](notes/05-state-secrets-observability.md)
- [実装前検証](notes/06-validation.md)

案ごとの詳細資料は、旧来の検討内容を残すために[docs/](docs/)にも置いています。レビューで最初に読む資料ではありません。

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
