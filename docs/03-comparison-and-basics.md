# 比較・認証・基礎インフラ

## 1. 2案の比較

| 観点 | 自宅Kubernetes | AWSクラウド完結 |
|---|---|---|
| 初期費用 | 既存設備を使えば抑えやすい | AWSの常時稼働リソース費用が発生 |
| 運用負担 | ノード、電源、回線、公開入口を自分で運用 | AWSの構成・料金・権限を運用 |
| 公開入口 | 既存Ingress、固定IPやDNS更新が論点 | ALB、ACM、VPCで構成しやすい |
| 静的Web | K8s上のWeb配信 | S3 + CloudFrontでキャッシュ可能 |
| 会場通信 | 外向きTLS接続をIngressへ集約 | 外向きTLS WebSocketをALBへ集約 |
| 障害時 | 自宅設備の復旧手順が必要 | AWS側の監視・置換を利用しやすい |
| 複数会場 | 自宅回線・クラスタの拡張が必要 | ECS、RDS、監視を拡張しやすい |
| 向いている段階 | 開発、実験、低コストの小規模運用 | 実演、本番、将来の複数会場 |

## 2. 認証方式の比較

`OIDCプロバイダー`は方式の名前ではなく、アプリケーションへ認証結果を渡す役割です。DiscordはOIDCを提供する具体的なIdPの一つです。

| 方式 | 管理者の体験 | 実装・運用 | 向いている範囲 | 注意点 |
|---|---|---|---|---|
| Discord OAuth2/OIDCをControl Serverが直接利用 | Discordでログインするだけ | アプリ側で認可コード、state、issuer、署名、セッション、権限対応表を持つ | Discordを使う少人数の管理者 | Discordに依存する。Guild/Roleをそのままアプリ権限にしない |
| Cognito + Discord OIDC | Cognitoのログイン画面からDiscordへ遷移 | CognitoのUser Pool、IdP設定、claim mapping、アプリ側の権限を持つ | 複数IdP、MFA、Passkey、組織向けSSO | 構成と設定が増え、MAU課金・機能プランを確認する必要がある |
| CognitoのメールOTP / Passkey | パスワードを覚えずにログインできる | メール配信、アカウント復旧、登録・失効を運用する | Discordを要求したくない管理者、将来の利用者認証 | 初期はアカウント管理とメール配信が増える |
| アプリ独自のメールOTP | メールのコードを入力する | OTP発行、期限、レート制限、メール送信、失効を自分で持つ | 小規模な一時管理者 | 自前実装の責任が増えるため、Cognito等へ寄せる方が保守しやすい |
| イベント単位のゲストセッション | QRやURLからすぐ操作できる | 短期トークン、イベント・会場スコープ、レート制限を持つ | 観客操作 | 本人識別や行為の監査には弱い。URL共有への対策が必要 |
| 共有パスワード | 入力が最も簡単 | 失効、個人監査、権限分離が弱い | ローカルな短期検証だけ | 管理者の本番認証には提案しない |

### 初期提案

- 管理者: Discord OAuth2/OIDCを直接利用し、DiscordのユーザーIDをアプリケーション内の管理者・演出担当へ対応付ける。
- 観客: イベント・会場に限定した短期ゲストセッションを発行する。操作範囲とレート制限をControl Serverで管理する。
- Cognito: 複数のログイン先、MFA、Passkey、組織向けSSO、複数アプリケーションでの共通認証が必要になった時点で再評価する。
- Venue Gateway: 人間の認証とは分け、ゲートウェイ単位の資格情報または証明書で認証する。

## 3. DNSの判断

今回の提案ではRoute 53を使わない。利用中のDNSサービスでサブドメインのCNAMEを管理し、CloudFront、ALB、Ingressのいずれかへ向ける。

| 判断 | 提案 |
|---|---|
| 自宅Kubernetes案 | 既存DNSサービスのCNAMEをIngressへ向ける |
| AWSクラウド完結案 | 既存DNSサービスのCNAMEをCloudFrontまたはALBへ向ける |
| 将来ルートドメインを使う | DNS事業者のALIAS/ANAMEが必要になった場合だけ再検討する |

Route 53は、ルートドメインのAlias、AWS内へのDNS集約、ヘルスチェックや高度なルーティングが必要になった場合だけ再評価する。

## 4. Nginxを使うか

| 配置 | 判断 |
|---|---|
| AWSクラウド完結のECS | 原則使わない。ALBが公開入口、ASP.NET Core/KestrelがアプリのHTTPサーバーを担当する |
| 自宅Kubernetes | 既存のIngress ControllerがNginxなら再利用する。アプリPod内へ別Nginxを追加しない |
| Nginxを追加する条件 | 複数アプリの細かなリバースプロキシ制御、既存運用標準、静的配信・変換・特別なヘッダ制御が明確に必要な場合 |

Nginxを置けば自動的に高速・安全になるわけではありません。ALB、CloudFront、Ingress Controller、Kestrelの責務が重複し、設定・ログ・ヘルスチェックが増えるため、明確な理由がない限り追加しません。

## 5. 共通の基礎インフラ提案

| 項目 | 自宅Kubernetes案 | AWSクラウド完結案 |
|---|---|---|
| TLS | Ingress側の証明書管理 | ACMをCloudFront / ALBへ関連付ける |
| RDB | クラウドのマネージドRDBを推奨 | RDS for PostgreSQLを第一候補 |
| 秘密情報 | Kubernetes Secret。Gitへ平文保存しない | Secrets Manager |
| 通常設定 | ConfigMapや環境変数 | SSM Parameter Storeや環境変数 |
| ログ | クラスタの標準出力を外部から確認可能にする | CloudWatch Logsとアラーム |
| Device Channel | ゲートウェイからの外向きTLS接続 | ゲートウェイからALBへの外向きTLS WebSocket |
| 状態ストア | RDBを正とする | RDBを正とする |
| KVS | 初期は採用しない | 初期は採用しない |

## 6. レビューで決めること

- 自宅設備を実演時の本番基盤として許容するか。
- RDBを自宅へ置かず、クラウドのマネージドRDBへ分離するか。
- 利用中のDNSサービスで固定IP・CNAME・ALIAS/ANAMEを扱えるか。
- 管理者をDiscord利用者に限定してよいか。
- 観客操作を本人識別なしのゲストセッションで許容するか。
- 通信断中も会場側で演出を継続するか。

## 参考資料

- [Amazon CloudFront pay-as-you-go pricing](https://aws.amazon.com/cloudfront/pricing/pay-as-you-go/)
- [Amazon CloudFront flat-rate pricing plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html)
- [CloudFrontの動作](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html)
- [S3 Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Route 53 pricing](https://aws.amazon.com/route53/pricing/)
- [Route 53 AliasとCNAMEの比較](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [CloudFrontのカスタムドメイン](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html)
- [Discord OAuth2](https://docs.discord.com/developers/topics/oauth2)
- [Discord OpenID Connect discovery](https://discord.com/.well-known/openid-configuration)
- [Amazon Cognito User Pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools.html)
- [Amazon CognitoのOIDC IdP連携](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-idp.html)
- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [ASP.NET CoreのLinuxホスティング](https://learn.microsoft.com/aspnet/core/host-and-deploy/linux-nginx)
