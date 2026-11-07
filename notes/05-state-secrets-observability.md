# Note 05: 状態・秘密情報・監視

## 状態ストア

初期はRDBを状態と履歴の正にする。会場、ゲートウェイ、演出、命令、利用者対応表をSQLで扱い、現在状態と履歴を別のサービスへ二重保存しない。

DynamoDB KVSは、現在状態、短期セッション、命令の重複排除を高頻度に扱う必要が明確になった場合の再評価候補とする。KVSを採用しても、Device Channelの再接続やLEDの発光結果は自動的には解決しない。

## 秘密情報

| 構成 | 保管先 | 注意点 |
|---|---|---|
| 自宅Kubernetes | Kubernetes Secret | Gitへ平文保存しない。バックアップと更新手順を決める |
| AWSクラウド | AWS Secrets Manager | DB、Discord、ゲートウェイの秘密情報をコンテナへ注入する |
| 通常設定 | ConfigMap / Parameter Store | 秘密情報と通常設定を分ける |

## 監視

最低限、次の単位でログを追跡できるようにする。

- `venueId`
- `gatewayId`
- `commandId`
- `connectionId`
- APIのリクエストID

監視対象はAPI 5xx、TaskまたはPod停止、ゲートウェイ切断、RDB接続エラー、命令タイムアウトとする。

## 運用の境界

- 自宅Kubernetes案: ノード、Ingress、証明書、ログ、バックアップ、復旧を自分で運用する。
- AWSクラウド完結案: ECS、ALB、RDS、Secrets Manager、CloudWatchの設定・料金・権限を運用する。
- どちらの案でも、Device Channelのプロトコルと会場側のSIG Mesh変換は自分たちの責務である。
