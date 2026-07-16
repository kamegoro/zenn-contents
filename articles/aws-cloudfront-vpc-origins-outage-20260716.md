---
title: "CloudFront VPC Originsとは何だったのか？ 障害から学ぶ「便利機能」の裏側"
emoji: "🌩️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "cloudfront", "vpc", "障害", "sre"]
published: true
---

## はじめに

2026年7月16日、AWSの CDN サービス「Amazon CloudFront」で障害が発生し、PayPay、note、ニコニコ生放送など日本国内の様々なサービス、Hugging Faceなど世界中の多数のサービスに影響が出ました。

@[card](https://www.itmedia.co.jp/news/articles/2607/16/news096.html)
@[card](https://www.theregister.com/off-prem/2026/07/16/aws-cloudfront-outage-serves-errors-instead-of-websites/5272421)

「CloudFrontが落ちた」と聞くと大災害に思えますが、実際に影響を受けたのは **CloudFront全体ではなく「VPC Origins」という特定機能を使っていた利用者だけ** でした。

この記事では、

- VPC Originsとは何のための機能か
- どんなユースケースで使われていたか
- なぜそのユースケースが軒並み死んでしまったのか

を、専門用語をなるべく噛み砕きながら整理します。

## そもそもCloudFrontって何？

CloudFrontはAWSのCDN（コンテンツ配信ネットワーク）です。ユーザーからのリクエストを、世界中に配置された「エッジロケーション」という拠点でまず受け止め、そこからオリジン（実際のアプリケーションサーバー）にリクエストを転送します。

キャッシュによる高速化だけでなく、「オリジンを直接インターネットに晒さず、CloudFrontだけを窓口にする」というセキュリティ目的でも広く使われています。

## その前に：VPCとサブネット

この先「VPC」「サブネット」「パブリックIP」という言葉が何度も出てきます。AWSを普段触らない方向けに、先に整理しておきます（知っている方は読み飛ばしてください）。

[VPC（Amazon Virtual Private Cloud）](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)は、AWSクラウドの中に作る「自分専用のプライベートなネットワーク」です。感覚としては社内LANに近いもの——外部から直接pingやアクセスができない、閉じたネットワーク空間だと思ってください。

このVPCの中を、IPアドレスの範囲でいくつかの区画に分けたものが「サブネット」です。区画には2種類あります。

- **パブリックサブネット**: インターネットから直接通信できる区画。ここに置いたサーバーには「パブリックIP」（インターネット上のどこからでも指定できる住所）を持たせられる
- **プライベートサブネット**: インターネットから直接は通信できない区画。そもそもパブリックIPを持たせる意味がない

これまでの多くのWebサービスは、本当は外部に直接晒したくないALBやEC2にも、あえてパブリックIPを持たせてパブリックサブネットに置くことで、インターネット経由でアクセスできるようにしていました。パブリックIPを持つ以上、悪意ある第三者からのスキャンや攻撃のリスクにもさらされます。

この前提をふまえて、VPC Originsの話に入ります。

## VPC Originsとは何のための機能か

これまでCloudFrontからオリジンに接続する方法は、基本的に「パブリックにアクセス可能なエンドポイント」を指す必要がありました。つまり、

- ALB（ロードバランサー）にパブリックIPを持たせて、パブリックサブネットに置く
- セキュリティグループやWAFで「CloudFrontのIPレンジからしか来ない」ように頑張って制限する

といった、地味に面倒な設定が必要でした。

**[VPC Origins](https://aws.amazon.com/blogs/aws/introducing-amazon-cloudfront-vpc-origins-enhanced-security-and-streamlined-operations-for-your-applications/)**（2024年11月発表の比較的新しい機能）は、これを解決します。

ALBやNLB、EC2インスタンスをプライベートサブネットに置いたまま、CloudFrontから直接つなげる機能です。CloudFrontは、プライベートサブネットに置かれた「パブリックIPを持たないALB/EC2」には、通常のインターネット経由ではアクセスできません。パブリックIPがない以上、外からは存在すら見えないからです。[VPC Originsの公式ドキュメント](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)によると、CloudFrontはこの制約を、AWSが内部に用意した専用の経路で解決します。CloudFrontのエッジロケーションから、インターネットを経由せず、AWSのネットワーク内部だけを通ってプライベートサブネット内のALB/EC2まで直接到達できるようにする機能です。この経路を使うことで、

- オリジン側にパブリックIPが不要になる
- インターネットゲートウェイへのルートも不要になる
- 「CloudFront以外からは絶対に来ない」を、面倒な許可リスト管理なしで実現できる

というのが謳い文句でした。追加コストもかからないため、「セキュリティを上げつつ運用の手間を減らせる」機能として、比較的新しいプロジェクトを中心に採用が進んでいたと考えられます。

図にすると、こういう構成です。

![CloudFront VPC Originsの構成図](/images/aws-cloudfront-vpc-origins-outage-20260716/vpc-origins-purpose.png)
*CloudFrontとオリジンの間に「AWS管理のプライベート接続」という新しいレイヤーが挟まる*

### 典型的なユースケース

- 社内向け・パートナー向けの管理画面をCloudFront経由でのみ公開したい
- Private API GatewayをCloudFront経由で配信したい
- 「ALBを完全プライベートにしてコストと攻撃面を削減したい」という構成（パブリックALB + WAFの代替）

要するに、**「オリジンを丸裸でインターネットに晒したくない」という、まっとうなセキュリティ意識の高い構成ほどVPC Originsを使っていた** ということです。

## 何が起きたか

- 発生: 日本時間 2026年7月16日 17:40頃（太平洋夏時間 01:45頃、AWS/海外メディア報道）
- 復旧: 日本時間 21:20頃に終息を発表（ITmedia報道）
- 症状: VPC Origins経由のCloudFront配信で5xxエラー（504 Gateway Timeoutなど）が多発
- 影響範囲: **VPC Originsを使っている顧客のみ**。他のオリジンタイプ（パブリックALBなど）を使っている顧客は無事
- 原因: CloudFrontのエッジロケーションからVPC内リソースへ接続をつなぐ「パケット処理サブシステム」に起因（AWS発表。この記事の執筆時点でAWSの公式Post-Event Summaryは未公開）
- 暫定対応: AWSは「VPC Originsが必須でなければ、一時的に他のオリジンタイプに切り替える」ことを回避策として案内

つまり、**CloudFront自体ではなく、CloudFrontからVPC内部への「専用の接続経路」部分だけが壊れた** という障害でした。パブリックサブネットに置かれた従来型のALB（インターネット経由でアクセスする構成）は無事だったのに、VPC Origins特有の「CloudFrontからプライベートサブネットへの内部経路」だけが不通になった、ということです。

## なぜこのユースケースだけ死んだのか

ポイントは、VPC Originsが「便利な代わりに、CloudFrontとオリジンの間に新しい依存コンポーネント（専用の接続経路＝パケット処理サブシステム）を挟んでいる」という点です。

- パブリックALB構成: CloudFront → インターネット経由でALBに直接アクセス（既存の枯れた経路）
- VPC Origins構成: CloudFront → **AWS管理の専用の接続経路（プライベート接続層）** → プライベートサブネット内のALB（新しい経路）

セキュリティ上のメリット（パブリックIPを持たなくてよくなる）と引き換えに、この新しい接続経路そのものが単一障害点（SPOF）になっていた、というのが今回の構造です。「セキュリティを上げるために採用した新機能が、可用性の面では新しい依存関係＝リスクにもなっていた」という、よくある教訓でもあります。

同じCloudFrontを使っていても、オリジンへのつなぎ方が違うだけで明暗が分かれた様子を図にすると、以下のようになります。

![パブリックALB構成とVPC Origins構成の障害時比較図](/images/aws-cloudfront-vpc-origins-outage-20260716/outage-comparison.png)
*同じCloudFrontでも「オリジンへの経路」が違うだけで生死が分かれた*

## まとめ

| 項目 | 内容 |
|---|---|
| 壊れた機能 | CloudFront VPC Origins（エッジ→VPC内オリジンの接続部分） |
| 何のための機能か | オリジンをパブリックIPなしで守りつつCloudFront経由で配信する |
| 影響を受けた構成 | Private ALB/NLB/EC2をVPC Origins経由で公開していた構成 |
| 影響を受けなかった構成 | パブリックALB＋WAF等の従来型構成 |
| 教訓 | 可用性向上のための「新しい抽象化」は、それ自体が新しい障害点になり得る |

VPC Originsは決して悪い機能ではなく、むしろ地味な設定ミスによる情報漏洩リスクを減らす、まっとうなベストプラクティスです。ただ今回のように「新しい依存が増える」という副作用も併せて理解した上で、マルチオリジンのフォールバックなど、可用性側の設計もセットで考えるのが大事だな、と改めて感じた障害でした。

---

参考:
- [Introducing Amazon CloudFront VPC Origins - AWS Blog](https://aws.amazon.com/blogs/aws/introducing-amazon-cloudfront-vpc-origins-enhanced-security-and-streamlined-operations-for-your-applications/)
- [Restrict access with VPC origins - Amazon CloudFront 公式ドキュメント](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)
- [What is Amazon VPC? - Amazon VPC 公式ドキュメント](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html)
- [AWS CloudFront outage serves errors instead of websites - The Register](https://www.theregister.com/off-prem/2026/07/16/aws-cloudfront-outage-serves-errors-instead-of-websites/5272421)
- [AWSで障害 「CloudFront」世界規模で不調 PayPayに影響、noteやニコニコ生放送もつながりにくく - ITmedia](https://www.itmedia.co.jp/news/articles/2607/16/news096.html)
- [AWS（主にCloudFront）障害のまとめ（2026/7/16）｜zephel01](https://note.com/zephel01/n/n274043a396db?hl=en)
