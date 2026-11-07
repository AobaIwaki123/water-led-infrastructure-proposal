# 提案2: AWSクラウド完結

静的Web、Control Server、Device Channel、RDB、秘密情報、監視をAWSへ寄せる案です。自宅設備への依存を減らし、公開入口と運用監視をAWSのリソースで統一します。

## 構成図

```mermaid
flowchart LR
  users["管理画面 / 観客画面"]
  route53["Route 53 任意"]

  subgraph aws["AWS"]
    cf["CloudFront"]
    s3[("S3 静的ファイル"])
    alb["Application Load Balancer"]
    subgraph ecs["ECS on Fargate"]
      app["Control Server container"]
      channel["Device Channel container"]
    end
    db[("RDS for PostgreSQL")]
    secret["Secrets Manager"]
    params["SSM Parameter Store"]
    ecr["Amazon ECR"]
    cw["CloudWatch Logs / Alarms"]
    acm["ACM certificate"]
  end

  gateway["Venue Gateway"]
  mesh["Bluetooth SIG Mesh"]
  leds["LED Nodes 約20台"]

  route53 -.->|"DNS"| cf
  route53 -.->|"DNS"| alb
  users -->|"静的Web HTTPS"| cf
  cf -->|"OAC"| s3
  users -->|"API HTTPS"| alb
  alb -->|"HTTP"| app
  gateway <-->|"外向きTLS WebSocket"| alb
  alb -->|"WebSocket"| channel
  app -->|"同一Task内の通信"| channel
  app -->|"SQL"| db
  secret -.->|"秘密情報"| app
  secret -.->|"秘密情報"| channel
  params -.->|"通常設定"| app
  ecr -.->|"イメージ"| ecs
  ecs -.->|"ログ / メトリクス"| cw
  acm -.->|"TLS"| cf
  acm -.->|"TLS"| alb
  gateway -->|"Bluetooth SIG Mesh"| mesh
  mesh <-->|"無線"| leds
```

## インフラリソース

| リソース | 役割 | 要点 |
|---|---|---|
| S3 | Webの静的ファイル保存 | バケットを公開せず、CloudFrontのOACだけに読み取りを許可する |
| CloudFront | 静的WebのCDN | キャッシュするのは静的ファイル。APIの命令・状態レスポンスはキャッシュしない |
| Application Load Balancer | APIとDevice Channelの公開入口 | HTTPS APIとWebSocket接続をECSへ振り分ける |
| ECS on Fargate | Control ServerとDevice Channelの実行 | まずは1 Service / 1 Task。OSのパッチ運用は持たない |
| RDS for PostgreSQL | 状態、履歴、命令の保存 | 初期はRDBを単一の正とし、DynamoDBとの二重管理を避ける |
| Secrets Manager | DBパスワード、Discord client secret、ゲートウェイ資格情報 | コンテナへ安全に注入し、更新時はタスク再配置を行う |
| SSM Parameter Store | 通常の設定値 | 環境名、ログレベル、エンドポイント等を管理する |
| ECR | コンテナイメージ保存 | CIからpushし、ECSを更新する |
| CloudWatch | ログ、メトリクス、アラーム | API 5xx、Task停止、ゲートウェイ切断、RDS状態を監視する |
| ACM | CloudFront / ALBのTLS証明書 | カスタムドメインを使う場合に用いる |
| Route 53 | DNS | 任意。AWS内でDNSも統一したい場合やルートドメインのAliasを使う場合に便利 |

## メリット

- 自宅の電源・回線・ノードに依存せず、公開入口と監視をAWSへ集約できる。
- CloudFrontで静的Webの初回表示と大量アクセス時のOrigin負荷を抑えやすい。
- Fargate、RDS、Secrets Manager、CloudWatchで運用責務を分担できる。
- 将来複数会場へ展開する場合の基盤をAWS上で拡張しやすい。

## デメリット

- ECS、ALB、VPC、RDS、Secrets Manager、監視を理解・運用する必要がある。
- Task、RDS、NAT Gateway等が常時稼働するため、自宅Kubernetesより固定費が発生しやすい。
- Device Channelの再接続、ACK、重複排除はECSを選んでも自前で設計する必要がある。
- CloudFront、Route 53、ACMを追加しても、APIや会場通信が自動的に改善するわけではない。

## CloudFrontの利用体験と料金の要点

- 利用者は通常のHTTPSサイトとしてアクセスする。キャッシュヒット時は近いエッジから静的ファイルが返り、初回やキャッシュミス時だけS3へ取りに行く。
- JavaScriptやCSSをハッシュ付きファイル名にし、`index.html`だけ短いキャッシュまたは再検証にすると、更新時の古い画面表示を抑えられる。
- 従量課金では、月1TBのデータ転送、1,000万HTTP/HTTPSリクエスト、200万CloudFront Functions呼び出しが無料枠に含まれる。S3、ECS、RDS、ALB、Route 53などの料金は別である。
- CloudFrontには従量課金に加えて、CloudFront・WAF・DNS・ログなどを月額でまとめる定額プランもある。小規模な初期運用では従量課金を基本とし、トラフィックやWAF要件が固まってから定額プランと比較する。
- CloudFrontの料金は、利用量、配信地域、機能で決まる。静的な小規模WebではCloudFront単体の料金より、ECS・RDS・NAT Gatewayの常時稼働費用が支配的になりやすい。
