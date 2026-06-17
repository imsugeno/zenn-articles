---
title: "ヘルスチェック緑を信じてはいけない。クロスアカウント×PrivateLink×API Gatewayでハマった罠5つ"
emoji: "🪤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "apigateway", "privatelink", "terraform", "network"]
published: true
publication_name: "canly"
---

こんにちは。カンリーエンジニアの[imsugeno](https://x.com/imsugeno)です。

とあるサービスの構築にて、API Gatewayを構築してAWSクロスアカウントで既存WEBバックエンドを構築しているECSにPrivateLink経由で繋ぐ構成を取りました。この時各ELBのヘルスチェックは通るのに疎通せず、ログでも原因がわからないという厄介な状況になったので起こったことをまとめました。

## 想定する構成

API の入口を作る **コンシューマ** アカウントと、ECS のバックエンドを持つ別チームの **プロバイダ** アカウントを、PrivateLink で繋ぎます。

![クロスアカウント構成の全体図。クライアントからコンシューマ・アカウント、PrivateLinkを越えてプロバイダ・アカウントへ至る経路](https://static.zenn.studio/user-upload/6f751f984081-20260608.png)

罠はすべて、この「別アカウント／別 VPC の境界」の前後で起きます。

## 罠1: セキュリティグループで許可したのに接続できない

![罠1の発生箇所。プロバイダ側NLBのセキュリティグループをハイライト](https://static.zenn.studio/user-upload/9b3cee222ab3-20260608.png)

**症状**：接続が即座に切られる（TCP の RST = 接続をリセットして打ち切るパケットが返る）。プロバイダ側 NLB のセキュリティグループで送信元 IP を許可したはずなのに通りません。

**原因**：NLB に `enforce_security_group_inbound_rules_on_private_link_traffic` という設定があります。「PrivateLink 経由で来たトラフィックにも SG のインバウンドルールを適用するか」のフラグで、既定は `true`（適用する）。問題はこの ON のときに SG が見る送信元 IP で、公式はこう書いています。

> If you enable inbound rules on PrivateLink traffic, the source of the traffic is the private IP address of the client, not the endpoint interface.
>
> （PrivateLink トラフィックにインバウンドルールを有効化した場合、トラフィックの送信元は Endpoint のインターフェースではなく、クライアントのプライベート IP アドレスになる）

つまり SG が評価するのは VPC Endpoint のインターフェース IP ではなく、**クライアント（コンシューマ）側の実プライベート IP** です。プロバイダ側の IP や Endpoint の IP を許可しても弾かれます。

**対策**：enforce が ON なら、コンシューマ側の VPC CIDR を許可します。公式は「PrivateLink 経由のトラフィックは IP が重複しうる」とも書いており、IP で絞りたくなければ enforce を `false`（PrivateLink トラフィックには SG を適用しない）にする手もあります。経路ごとに正解が違うので、実トラフィックで確かめます。

:::message
`SecurityGroupBlockedFlowCount_Inbound`（SG に弾かれたフロー数を表す CloudWatch メトリクス）で、弾かれている数を直接観測できます。送信元 IP の取り違えを疑うときは、まずこれを見るのが速いです。
:::

引用と詳細は AWS [Update the security groups for your Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-update-security-groups.html) を参照。

## 罠2: 統合 URI に ELB の自動生成名を書くと繋がらない

![罠2の発生箇所。API GatewayからコンシューマNLBへの統合をハイライト](https://static.zenn.studio/user-upload/2cd444793713-20260608.png)

**症状**：API Gateway の統合が `INTEGRATION_FAILURE`（統合先への接続に失敗したことを示すエラー）になり、バックエンドから応答を得られません。

**原因**：統合 URI に ELB/NLB の自動生成ドメイン（`...elb.amazonaws.com` のような名前）を直接書いていました。HTTPS 統合だと証明書やホストベースのルーティングと噛み合わず、そもそも名前が不安定で管理しづらい。

**対策**：自前ドメインの CNAME を当て、統合 URI は自分の管理下のホスト名を指すようにします。AWS の自動生成名は内部実装の都合で振られる名前なので、設定値として直接参照しないのが安全です。次の罠3とまったく同じ手筋です。

## 罠3: 証明書エラーで TLS が通らない／VPC Endpoint の自動名を直に使えない

![罠3の発生箇所。コンシューマのVPC Endpointと、境界を越えた先のALBの証明書をハイライト](https://static.zenn.studio/user-upload/5f1e22fdb410-20260608.png)

ここが一番こじれました。

### 前提: アプリはどの名前で繋いでいたか

コンシューマ側のアプリ（集約サービス）から、VPC Endpoint を経由して境界の向こうのプロバイダ ALB に HTTPS で接続します。このとき、アプリが指定する接続先ホスト名には、相反する2つの要求がかかっていました。

- **証明書に一致する名前にしたい**：プロバイダ ALB は `*.svc-a.example.com` のワイルドカード証明書で TLS 終端している。アプリが繋ぐ名前がこの証明書に一致しないと、TLS 検証で落ちる。
- **VPC Endpoint に解決される名前にしたい**：実際の接続先は VPC Endpoint で、その実体は `vpce-….vpce-svc-….vpce.amazonaws.com` という AWS 自動生成名。アプリが繋ぐ名前は、最終的にこの名前へ解決されないといけない。

「証明書に合う名前」と「VPC Endpoint に解決する名前」を1つのホスト名で兼ねようとして、次の2つの壁にぶつかりました。

### 壁1: ワイルドカード証明書は「1段下のサブドメイン」しか守れない

`x509: certificate is valid for *.svc-a.example.com, not backend.internal.svc-a.example.com`（証明書は `*.svc-a.example.com` 用で、`backend.internal.svc-a.example.com` には使えない、という TLS 検証エラー）が出ます。

`*.svc-a.example.com` が守るのは、そのすぐ下の1段（`backend.svc-a.example.com`）だけです。`backend.internal.svc-a.example.com` のようにサブドメインがもう1段深い名前は対象外になります。これは仕様です。

### 壁2: VPC Endpoint の自動名は、そのままアプリの接続先にできない

プライベート DNS を無効（`private_dns_enabled = false`）で運用していたので、アプリは Endpoint を自動生成名 `vpce-….vpce-svc-….vpce.amazonaws.com` で指すことになります。

この自動名は、外（公開 DNS）からでも解決でき、Endpoint の ENI のプライベート IP を返します（その IP は VPC の内側からしか到達できません）。なので名前解決そのものは問題ありません。問題は、この名前をアプリの接続先として直に使えないこと。理由は2つです。

- **TLS 証明書に一致しない**：この名前で HTTPS 接続すると、ALB のワイルドカード証明書（`*.svc-a.example.com`）と噛み合わず、壁1と同じ検証エラーになる。
- **名前が不安定**：`vpce-…` は AWS が実装都合で振る識別子で、Endpoint を作り直すと変わる。設定値として直接参照したくない（罠2と同じ理由）。

なので自動名は CNAME で覆い、安定した別名で参照します。

### 対策: 2段の CNAME に役割を分担させる

VPC Endpoint を指す内部用の名前は、命名規則の都合でサブドメインが2段（`backend.internal.svc-a.example.com`）になっていました。壁1のせいで、これは証明書の範囲外です。そこで証明書が効く1段の名前（`backend-internal.svc-a.example.com`）を別に立て、それを内部用の名前へ転送します。「証明書を通す役」と「VPC Endpoint を隠す役」を別々のホスト名に分けるわけです。

```
① backend-internal.svc-a.example.com   ← アプリが接続する名前。サブドメイン1段なので証明書が守る
        ▼ CNAME
② backend.internal.svc-a.example.com    ← VPC Endpoint を隠す名前。2段で証明書外だが、ここは検証されない
        ▼ CNAME
③ vpce-0abc….vpce-svc-0def….ap-northeast-1.vpce.amazonaws.com   ← AWS の自動生成名
```

TLS の検証対象は、アプリが指定する①の名前だけです。①はサブドメインが1段なのでワイルドカードに収まります。①→②→③は単なる CNAME 転送で検証は走らないので、②③が証明書の範囲外でも問題になりません。

:::message
ポイントは「1つのホスト名に複数の役割を背負わせない」こと。証明書を通す役と内部の口を隠す役を別名に分けると、衝突する要件が解けます。罠2の「自動名を CNAME で覆う」も同じ発想でした。
:::

## 罠4: 子ゾーンを作っても名前解決できない（no such host）

![罠4の発生箇所。親ゾーンへのNS委任。親ドメインのアカウントとコンシューマ・アカウントにまたがる](https://static.zenn.studio/user-upload/1e8eb50be7ed-20260608.png)

**症状**：`lookup … : no such host`（名前解決に失敗し、そんなホストは無いと返る）。新しいドメイン区画（子ゾーン）を作ったのに引けません。

**原因**：子ゾーン（`internal.example.com`）を作っても、親ゾーン（`example.com`）に「ここの問い合わせは子ゾーンへ回す」という委任（NS レコード）を入れないと、世界中の DNS は子ゾーンの存在を知れません。そしてクロスアカウント構成では、親ゾーン・証明書・IaC リポジトリのオーナーがそれぞれ別チームのことが多い。「自分のリポを直しても効かない」が起きます。

**対策**：親ゾーン側に NS 委任を追加します。設定の正しさよりも「どのリソースを誰が管理しているか」で詰まるので、変更を入れる前に管轄を棚卸ししておくと早いです。

## 罠5: 内部通信だけ WAF にブロックされる

![罠5の発生箇所。プロバイダALB前段のWAFをハイライト](https://static.zenn.studio/user-upload/16faaa021b2e-20260608.png)

**症状**：ネットワーク的には通っているのに、サーバ間の内部通信（`/internal/*` など）やヘルスチェックだけがブロックされます。

**原因**：Bot 対策などのマネージド WAF ルールは「ブラウザらしさ」を前提にしたヒューリスティクスで判定します。ツール系の User-Agent やサーバ間通信のパターンを不審と誤判定しやすい。

**対策**：内部通信のパスを最優先の許可ルールで明示的に通します。ただし WAF を素通りさせる分、その経路は別の認証（署名付きの内部トークンなど）で多層に守ります。WAF は人間のブラウザを守る前提のことが多く、サーバ間通信はそもそも想定外になりがちです。

## まとめ

同じ構成を組むときに、最初から潰しておきたい点です。

- プロバイダ NLB の SG は、enforce ON なら送信元＝コンシューマ側の実 IP。許可 CIDR を取り違えない
- API Gateway の統合先は AWS 自動生成名でなく自前ドメインを指す
- 多段ドメインは、証明書のワイルドカードが届くサブドメイン1段に寄せる。VPC Endpoint の自動名は CNAME で覆う
- クロスアカウントの DNS は親ゾーンへの NS 委任を忘れない。誰が何を管理しているか棚卸しする
- 内部通信パスは WAF で明示的に許可（Allow）し、独立した認証と合わせて多層防御

クロスアカウント × PrivateLink × API Gateway は、単一の原因を探すと迷子になります。罠が層に分散していて、`apply` 成功やヘルスチェック緑といった確認サインは当てになりません。経路を層に切り、上流から1つずつ「ここまで通った」を実証で確定させる進め方が、結局いちばん速かったです。
