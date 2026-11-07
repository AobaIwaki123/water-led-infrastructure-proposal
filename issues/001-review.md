# Issue #1 下書き: 水上LEDインフラ2案レビュー

## 目的

水上LED演出システムの初期インフラについて、次の2案から採用候補を決める。

- [提案1: 自宅Kubernetes基盤](../docs/01-home-kubernetes.md)
- [提案2: AWSクラウド完結](../docs/02-cloud-complete.md)
- [比較・認証・基礎インフラ](../docs/03-comparison-and-basics.md)

## 経緯

自宅Kubernetes、EC2、ECS、IoT Core、Lambda、RDB、DynamoDB KVSを個別に比較していたが、レビュー対象が散らばったため、自宅Kubernetes案とクラウド完結案へ整理した。細かな候補は比較資料の背景として扱い、Issueでは2案の選択に集中する。

## レビューしてほしい点

- 実演時の本番基盤として、自宅の電源・回線・ノード障害を許容できるか。
- 自宅Kubernetes案を採用する場合、既存Ingress Controller、外部公開経路、DNS更新、監視を再利用できるか。
- クラウド完結案のECS、ALB、RDS、CloudFront、Secrets Manager、CloudWatchを許容できるか。
- Route 53を使う必要があるか。現在のDNSサービスで必要なレコードを管理できるか。
- アプリケーション前段のNginxを追加する明確な理由があるか。
- 管理者をDiscord OAuth2/OIDCで認証し、観客を短期ゲストセッションにする案でよいか。
- 会場ゲートウェイが外向きTLS接続し、通信断中は現地側で演出を継続する前提でよいか。

## コメントの書き方

構成の細部を提案する場合は、先に対象案を明記する。

```text
対象: 自宅Kubernetes案 / AWSクラウド完結案 / 共通
論点: リソース名または要件
判断: 採用 / 修正 / 保留
理由: 運用条件、費用、利用体験、将来拡張など
```

## 完了条件

- 採用する案、または段階的な採用順序が決まる。
- Route 53とNginxを使うかどうかの判断が決まる。
- 管理者・観客・Venue Gatewayの認証境界が決まる。
- 未決定事項が、実装前検証と別の設計課題に分離される。
