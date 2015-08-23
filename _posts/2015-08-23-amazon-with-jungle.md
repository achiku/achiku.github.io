---
layout: post
title: 'jungle: Simple UNIX-like AWS CLI'
---

jungleというUNIX-likeにAWSを操作するCLIを作った。

- https://github.com/achiku/jungle


## 利用方法抜粋

- https://github.com/achiku/jungle/blob/master/README.md

Listing all EC2 instances

```
jungle ec2 ls
```

Filtering EC2 instances by Name tag using wildcard

```
jungle ec2 ls *web*
```

Starting instance

```
jungle ec2 up -i i-xxxxxx
```

Stopping instance

```
jungle ec2 down -i i-xxxxxx
```

SSH login to instance specified by instance id

```
jungle ec2 ssh -i i-xxxxxx --key-file /path/to/key.pem
```

SSH login to instance specified by Tag Name

```
jungle ec2 ssh -n blog-web-server-01 --key-file /path/to/key.pem
```

## TL;DR
- もっとUNIX-likeにAWSを操作するCLIが欲しかった
- AWSのTag:Nameを起点にsshログイン/情報取得できる機構が欲しかった
- [Click](https://github.com/mitsuhiko/click)というライブラリが良さそうだったから使ってみたかった

## 背景
既存の公式[awscli](https://github.com/aws/aws-cli)は最高にフレキシブルだし全AWSのサービスに対応しており、自分も大変お世話になっている。しかし、オプションがやたら長かったり、「それ要る？」っていう機能てんこ盛りだったりという特徴もある(これはawscliが「AWSサービスとの連携においてできないことは無い」ってのをたぶん目指して作られているのでしょうが無い)。その為、正直普段使いのCLIとしては、素のままに使いづらいと言わざるを得ない、と思った。

また、今自分たちはAWSのTag:Nameを起点に色々なものを管理しており、サーバは作られては消えているので、IPやdomain等ではなくTag:Nameを使ってログイン/状態参照/ログ参照できるようにできたら便利だろう、というのがある。


## 解決策
- それ[ec2ssh](https://github.com/mirakui/ec2ssh)でできない？
    * EC2にログインする部分は対応できそうだけど、bastionサーバ経由してssh proxyしたりとかできないっぽいし、これは結局IPとDomainを更新し続ける使い方だなと思った
    * 追加で、ELBにぶら下がってるインスタンスのIDやらステータスやら確認したかった
    * 追加で、ASGやLCもCLIから確認したかった
    * 可能なら同じようなUIで
- それawscliでインスタンスID指定してjqでpublic_ipを取得してsshにパイプで渡せばいいんじゃない？
    * EC2にログインする部分は対応できそうな気がする
    * ただ、その他微妙に他のAWSサービスでもやりたい事ある
- 複雑なawscliのコマンドに分かりやすいaliasをつければよいのでは？
    * その通りそれでも可能だと思うけど、共有しづらくない？
- filterする部分とかは```--filters file://filters.json```とかでJSONにまとめておけばよいのでは？
    * その通りそれでも可能だと思うけど、共有しづらくない？
- awscliは「不可能なことは無い」ように作られてるけど、もっとそぎ落として日々のオペレーションで良く使うものだけまとめてPyPIで公開してpip installしてもらった方が楽なのでは？

と、色々書いてみたけど恐らく色んな代替案があるし、どれを実行してもなんとなくうまくできそうな気はしている。ただ、自分がよく実行するオペレーションを覚えやすい形で、ある程度の柔軟性だけ持たせて保存しておきたかった。あと、Click使いたかったしPyPIに何か登録してみたかったので書いた感もある。

ちょっと誰か使ってみて感想きかせて欲しいっす！！！！初Python OSSっす！！！！
