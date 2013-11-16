---
layout: post
title: 'AWS: EIP + VPC + NAT + ELB + EC2 + RDS #1'
---

Amazonの提供するクラウド系サービスを利用してひと通りの環境を構築してみる。作りたいのはパブリックなサブネットとプライベートなサブネットに分割し、Public側にNAT Instance + ELB + 踏み台サーバを配置、Private側にWeb/APサーバ + RDSを置く、という構成。DMZとDC内部、的な構成でやってみようと思います。本当はDC内部はVPNつないでみたいけど、今回は踏み台サーバを用意してプライベート側のEC2にログインして作業する構成とする。

キャプチャは取らない方式でまとめます。

アカウント作成
--------------
まずはAWS用のアカウントを作成する。既にAmazonのアカウント持っている人はそのアカウントにAWSアカウントをひもづける事が可能。アカウント作成時に必要なものは住所、電話（アカウント作成時に電話で認証をとっている）、クレジットカード情報。

[Amazon Web Service](http://aws.amazon.com/jp/)

VPC
---
Amazon Virtual Private Cloudの略。要はいつもオンプレミスでやっているようなNWレイヤでのセキュリティコントロールや、ルーティングやサブネットの分割がAWS上でもできるということ。設定したVPC上にEC2等のAWSオブジェクトを作成していく。また、今回はやらないけど、既存のデータセンターとの間にHardware VPN(IPsec VPN)を使って接続することもできる。うーむ。便利。以下VPCのサイトから引用。

>Amazon VPC のネットワーク設定は容易にカスタマイズすることができます。例えば、インターネットとのアクセスが可能なウェブサーバーのパブリック サブネットを作成し、データベースやアプリケーションサーバーなどのバックエンドシステムをインターネットとのアクセスを許可していないプライベート サブネットに配置できます。セキュリティグループやネットワークアクセスコントロールリストなどの複数のセキュリティレイヤーを活用し、各サブネットの Amazon EC2 インスタンスへのアクセスをコントロールすることができます。[Amazon VPC公式サイトから引用](http://aws.amazon.com/jp/vpc/)

VPCのコントロールパネルを開く。Start VPC Wizardをクリックしてウィザード開始。
VPC with Public and Private Subnetsを選択。デフォルトの選択のまま、Create VPCを実行。

そうすると作成されるものが以下。
- VPC x 1
- Subnet x 2
- Route Table x 2
- Internet Gate Way x 1
- Elastic IP x 1
- EC2 NAT Instance x 1

NAT Instanceは普通にEC2のインスタンス作成時にNATインスタンス用にAMIを選択したやつっぽい。最初NATのサービスを探してた。AMI IDは、amzn-ami-vpc-nat-pv-2013.03.1.x86_64-ebs。

NATは外部と通信するためGlobal IPが必要なので、VPCウィザード先輩が自動的にElastic IPを振り出しNATインスタンスに割り当てている模様。先輩はいい感じにやってくれる。

更に先輩は、2つ作ったサブネットの内、一つのデフォゲをInternet Gateway、もう一つのサブネットのデフォゲを自動的に作成したNATに設定してくれる。これでInternet Gatewayがデフォゲに指定されている方がパブリックサブネット、いわゆるDMZ的なものになる。一方で、NATがデフォゲに指定されている方がプライベートサブネット、というくくりになる。


Bastion Server(EC2)
-------------------
EC2のマネジメントコンソールから新規にEC2インスタンスを作成する。
この際、踏み台サーバはPublicサブネットに配置する。というか踏み台サーバの事をBastion Host/Serverという事を今日始めて知りました。ばすちょん。

[What is a bastion host?](http://www.sans.org/security-resources/idfaq/bastion.php)

作成が完了すると、pem形式のキーの名前を聞かれるので適当に命名。ダウンロードしたpem keyをローカルに配置し、以下のコマンドを実行してBastionサーバに接続できることを確認する。pem keyを何単位で分けるのが適切かは、セキュリティと運用の容易さのトレードオフで決めるのかな。今回は一つのpem keyで全てのサーバにアクセスできるようにしておく。本当は本番/ステージング別とか、プライベート/パブリックサブネット別とかに分けるのだろう。


{% highlight bash %}
    $ ssh -i tokyoPemKey.pem ubuntu@your-ec2-domain.compute.amazonaws.com
{% endhighlight %}


AP Server(EC2)
---------------
EC2のマネジメントコンソールから新規にEC2インスタンスを作成する。
この際、APサーバはPrivateサブネットに配置する。VPCをウィザードで作成すると、自動的に10.0.1.0/24がプライベートなサブネット（Default GawtewayがNATインスタンスになってるやつ）になっているっぽい。逆に、10.0.0.0/24がパブリックになっており、Default GatewayにはInternet Gatewayが指定されている。よって、Configuration Instance Detailsから以下の設定を実施する。それ以外はデフォルトで問題なし。プライベートなサブネットに配置するので、Public IPとかは設定しないでよし。ちなみに今回はUbuntu Server 13.10 - ami-b945ddb8を利用。

- Network: 10.0.0.0/16
- Subnet: subnet-xxxxx(10.0.1.0/24)


Load Balancer(EC2, ELB)
-----------------------
EC2ダッシュボードに行くと、左側のNETWORK&SECURITYタグの中にLoad Balancerという項目があるので、ここから新規作成する。

適当な名前をつける。
Create LB Inside:の部分は新規に作成したVPCを選択。今回の場合は、10.0.0.0/16。
LBが通すプロトコルとポートを指定する。今回はデフォルトのHTTPとポート80番のみ。
ヘルスチェックはデフォルトのまま。

ロードバランス「される」インスタンスが存在するサブネットを選択する。今回はAPサーバをPrivateサブネット（Internet Gatewayがついてない方のサブネット）に配置しているので、そちらを選択。デフォルトでは、10.0.1.0/24。
>You will need to select a Subnet for each Availability Zone where you wish to have load balanced instances. A Virtual Network Interface will be placed inside the Subnet and allow traffic to be routed into that Availability Zone. Only one subnet per Availability Zone may be selected.


セキュリティグループは一旦デフォルト。


作成完了したら、EC2のマネジメントコンソール左ペインからLoad Balancerを選択して新規作成されたレコードを確認。Instancesに先ほど作成したAP Serverをひもづける。コレで以下の通信経路の基礎が完成。次回はSecurity Groupと、APサーバに実際のWebアプリをデプロイする！

::
    
    [client] -> [ELB] -> [Web/AP Server]




