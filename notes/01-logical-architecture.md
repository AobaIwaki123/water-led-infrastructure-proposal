# Note 01: 論理アーキテクチャ

## 共通構造

2案とも、利用者、制御アプリ、会場ゲートウェイ、水上デバイスの責務を分離する。

```mermaid
flowchart LR
  subgraph user["利用者層"]
    admin["管理画面"]
    audience["観客画面"]
  end
  subgraph control["制御層"]
    api["Control API"]
    coordinator["Command Coordinator"]
    channel["Device Channel"]
    state[("State Store")]
  end
  subgraph venue["会場エッジ層"]
    gateway["Gateway"]
    scheduler["Local Scheduler"]
    mesh["Bluetooth SIG Mesh"]
  end
  nodes["LED Nodes 約20台"]

  admin -->|"HTTPS"| api
  audience -->|"HTTPS"| api
  api -->|"要求検証"| coordinator
  coordinator -->|"状態更新"| state
  coordinator -->|"命令"| channel
  channel <-->|"外向きTLS"| gateway
  gateway -->|"ローカル実行"| scheduler
  scheduler -->|"デバイス命令"| mesh
  mesh <-->|"無線"| nodes
```

## 責務の境界

- Control Serverは利用者、会場、操作内容を検証する。
- Device Channelはゲートウェイとの接続、接続状態、命令配送を扱う。
- Gatewayはクラウド側の命令を現地のSIG Meshへ変換する。
- Local Schedulerは、必要な演出を現地で継続する。
- LED NodeはクラウドAPIを直接公開しない。

## 通信断の考え方

ゲートウェイがすでに受信した演出を通信断中も続けるかは未確定だが、インターネット経由の都度制御だけに依存しない構成を候補とする。再接続後は、ゲートウェイが現在状態と最後に処理した世代を報告できる設計を検証する。
