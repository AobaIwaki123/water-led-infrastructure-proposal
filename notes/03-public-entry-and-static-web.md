# Note 03: 公開入口・静的Web・DNS

## CloudFrontの役割

CloudFrontはクラウド完結案の静的Web配信に限定する。S3バケットは非公開にし、CloudFrontのOrigin Access Controlからだけ読めるようにする。APIの命令・状態レスポンスはキャッシュしない。

```mermaid
flowchart LR
  browser["Browser"]
  cf["CloudFront"]
  s3[("Private S3")]
  alb["ALB API"]

  browser -->|"静的ファイル"| cf
  cf -->|"OAC"| s3
  browser -->|"API / 命令"| alb
```

利用者からは通常のHTTPSサイトに見える。キャッシュヒット時はエッジから返り、キャッシュミス時にS3へ取りに行く。JavaScriptやCSSはハッシュ付きファイル名、`index.html`は短いキャッシュまたは再検証にする。

## Route 53

Route 53はDNSサービスであり、CloudFrontやALBの必須部品ではない。

| 状況 | 判断 |
|---|---|
| サブドメインを使う | 既存DNSサービスのCNAMEで足りる |
| ルートドメインをCloudFrontへ向ける | Route 53 Alias、またはDNS事業者のALIAS/ANAMEが便利 |
| AWSへDNSを集約する | Route 53を使う |
| 自宅Kubernetes案 | 現在のDNSサービスでIngressへ向け、Route 53は追加しない |

## Nginx

- AWS案ではALB + Kestrelを基本とし、Nginxを置かない。
- 自宅Kubernetes案では、既存Ingress ControllerがNginxならそれを使う。
- アプリPod内にNginxを追加するのは、特別なリバースプロキシ制御や既存運用標準が必要な場合だけにする。

## TLS

- AWS案: ACM証明書をCloudFrontとALBへ関連付ける。
- 自宅Kubernetes案: 既存Ingressの証明書管理方法を利用する。

## 参考

- [CloudFrontの動作](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/HowCloudFrontWorks.html)
- [S3 Origin Access Control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Route 53 AliasとCNAMEの比較](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [CloudFrontのカスタムドメイン](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html)
