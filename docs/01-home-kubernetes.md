# 提案1: 自宅Kubernetes基盤

自宅のKubernetesでWeb、Control Server、Device Channelを運用し、RDBはクラウドのマネージドサービスへ分離する案です。既存のKubernetes運用知識と初期費用の低さを優先します。

## 構成図

```mermaid
flowchart LR
  users["管理画面 / 観客画面"]
  dns["既存のDNSサービス"]

  subgraph home["自宅 Kubernetes"]
    ingress["既存のIngress Controller"]
    web["Web配信"]
    app["Control Server"]
    channel["Device Channel"]
    secrets["Kubernetes Secret"]
    logs["クラスタのログ / 監視"]
  end

  subgraph cloud["外部クラウド"]
    db[("マネージドRDB"])
  end

  gateway["Venue Gateway"]
  mesh["Bluetooth SIG Mesh"]
  leds["LED Nodes 約20台"]

  dns -.->|"名前解決"| ingress
  users -->|"HTTPS"| ingress
  ingress -->|"静的ファイル"| web
  ingress -->|"API"| app
  app -->|"命令"| channel
  channel <-->|"外向きTLS接続"| gateway
  app -->|"SQL"| db
  secrets -.->|"設定 / 秘密情報"| app
  secrets -.->|"設定 / 秘密情報"| channel
  app -.->|"ログ / メトリクス"| logs
  gateway -->|"Bluetooth SIG Mesh"| mesh
  mesh <-->|"無線"| leds
```

## インフラリソース

| リソース | 役割 | 要点 |
|---|---|---|
| Kubernetes cluster | Web、Control Server、Device Channelの実行 | 既存クラスタを利用する。ノード、電源、回線、ストレージを自分で運用する |
| Ingress Controller | HTTPSの公開入口とルーティング | 既存のIngress Controllerを使う。既存がNginxならNginx Ingressを再利用し、アプリPodへ別Nginxを重ねない |
| Web配信 | HTML、CSS、JavaScriptの配信 | 小規模ならWebコンテナまたは既存の静的配信でよい。CloudFrontは必須ではない |
| Control Server | API、権限、演出状態 | Kubernetes Deploymentとして動かす |
| Device Channel | ゲートウェイとの接続 | ゲートウェイが外向きにTLS付きHTTPS/WebSocketを確立する |
| RDB | 演出、命令、履歴、利用者対応表 | 自宅ではなくクラウドのマネージドRDBを推奨。自宅DBにするとバックアップと復旧が増える |
| Kubernetes Secret | DB接続情報、外部APIキー等 | Gitへ平文で保存しない。暗号化・バックアップ・更新手順を別途決める |
| DNS | ドメイン名からIngressへ接続 | Route 53は不要。現在のDNSサービスでA/AAAAまたはCNAMEを管理する |
| ログ / 監視 | 接続状態、APIエラー、ノード状態 | 既存のクラスタ監視を利用する。実演時に外部からログを確認できるようにする |

## メリット

- 既存のKubernetes知識と資産を活用できる。
- 常時稼働するWebとDevice Channelを自由に配置できる。
- 小規模な検証ではAWSのマネージドサービスを増やさずに始められる。

## デメリット

- 自宅の電源、回線、ルーター、ノードが本番の障害点になる。
- 公開入口、TLS、バックアップ、更新、障害復旧を自分で運用する。
- 会場実演中に自宅側で障害が起きた場合の復旧担当と手順が必要になる。
- 観客が多い場合、Ingressと自宅回線の容量がボトルネックになる。

## DNSとNginxの扱い

- Route 53は使わない。既存のDNSサービスでサブドメインのCNAMEをIngressへ向ける。
- Nginxはアプリの前段へ別途置く必要はない。Kubernetesに既存のIngress Controllerがある場合だけ、それを公開入口として使う。
- 固定IPがない場合は、DNS更新または別の外部公開方式が必要になる。この運用負担はクラウド完結案との比較点になる。

## この案を選ぶ条件

- 自宅設備を実演期間中も運用できる担当者がいる。
- 既存クラスタと外部公開経路を再利用できる。
- 一時的な停止や手動復旧を許容できる。
- AWSの固定的なマネージドサービス費用より、既存設備の活用を優先する。
