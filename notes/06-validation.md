# Note 06: 実装前検証

提案資料では決めず、採用案が決まった後に確認する項目を分離する。

| 検証 | 確認すること |
|---|---|
| Gateway接続 | 外向きTLS、再接続、ハートビート、認証情報の更新 |
| Device Channel | 命令の順序、重複、ACK、タイムアウト、接続切替 |
| Bluetooth SIG Mesh | 採用モデル、メッセージサイズ、再送、Provisioning、鍵管理 |
| ローカル実行 | 通信断中にどこまで演出を継続できるか |
| 自宅Kubernetes | 固定IPまたは外部公開経路、Ingress、証明書、電源断復旧 |
| AWSクラウド | ECSデプロイ、ALB WebSocket、RDS復元、Secrets更新、監視アラーム |
| CloudFront | キャッシュ更新、古いアセット、S3 OAC、API非キャッシュ |
| 認証 | Discord callback、権限対応表、ゲスト期限、失効、レート制限 |

実機で測るべき遅延・同期精度・同時操作数は、インフラ案の採否とは別の検証結果として記録する。
