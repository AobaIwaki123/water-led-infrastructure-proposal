# Note 02: 実行基盤・リソース配置

## 自宅Kubernetes案

自宅Kubernetesでは、Web、Control Server、Device Channelを同じクラスタで運用する。公開入口は既存のIngress Controllerを使い、アプリPod内へ別のNginxを追加しない。

| リソース | 責務 | 運用上の論点 |
|---|---|---|
| Kubernetes cluster | アプリの実行 | ノード、電源、回線、ストレージ、更新 |
| Ingress Controller | 公開HTTPSとルーティング | 固定IP、証明書、外部公開経路 |
| Control Server | API、認証、演出状態 | Deployment、ログ、再起動 |
| Device Channel | ゲートウェイ接続 | 接続状態、再接続、単一接続の扱い |
| マネージドRDB | データ保存 | 自宅DBを避け、バックアップを外部へ分離 |

## AWSクラウド完結案

AWS案では、ECS on Fargateの同一Task内にControl ServerとDevice Channelを置くところから始める。Nginxは使わず、ALBが公開入口、KestrelがアプリのHTTPサーバーを担当する。

| リソース | 責務 | 運用上の論点 |
|---|---|---|
| ECS on Fargate | アプリの実行 | 1 Service / 1 Taskから開始。Task数を増やす条件を決める |
| ALB | APIとWebSocketの入口 | HTTPS、ターゲット、ヘルスチェック |
| RDS for PostgreSQL | 状態・履歴 | バックアップ、復元、接続数 |
| Secrets Manager | 秘密情報 | ローテーション、タスク再配置 |
| ECR | イメージ | CI/CD、タグ、ロールバック |
| CloudWatch | ログ・監視 | API 5xx、Task停止、Gateway切断 |

## 選択の境界

ECSを選んでもDevice Channelの再接続・ACK・重複排除は自動化されない。Kubernetesを選んでも公開入口・証明書・監視・バックアップは必要になる。比較の中心はサービス名ではなく、誰がその運用責務を持つかである。

## 旧案の位置づけ

EC2、Lambda、IoT Core、DynamoDB KVSは、要求が変わった場合の再評価候補として残す。初回レビューでは2案へ含めない。
