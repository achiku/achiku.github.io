---
layout: post
title: 'AWS EMRでPrestoを動かしてshibからクエリ流してみる'
---

[Spark, SQL on Hadoop etc. Advent Calendar 2014](http://qiita.com/advent-calendar/2014/distributedcomputing)の最終日です！！


## 書くこと
AWS EMR/S3 + Hive + Presto + Hue + Shibの環境を構築し、簡単にトライアルしてみる。


## Prestoについて
Facebookがオープンソースで開発しているMPP(Massively Parallel Processing)クエリエンジン。

- 本家: [Presto - Distributed SQL Query Engine for Big Data](http://prestodb.io/)
- TDさんのわかりやすい解説: [『Prestoとは何か，Prestoで何ができるか』](http://treasure-data.hateblo.jp/entry/2014/07/10/150250)

同じ系統のクエリエンジン括りだとImpalaやApache DrillがOSSとして開発されている。MPPクエリエンジン/データベースの大まかな流れや種類、それぞれの使いドコロについてははコチラの記事が最高にまとまっていて参考になる。
[MPP on Hadoop, Redshift, BigQuery](http://repeatedly.github.io/ja/2014/07/mpp-on-hadoop-redshift-bigquery/)

Facebook以外にもNetfilixやTreasureData等、デカイデータ扱う企業がこぞって開発に参加している印象があり、開発のスピードが速く矢継ぎ早にリリースを繰り返し、リリース毎に便利になっていく。「商用DWHを置き換えっぞ」っていう気合の入った目標に見合うパフォーマンスに加え、SQL分析関数を早々に実装しANSI SQLを完コピしつつJSONも華麗に扱る上、既存Hadoopエコシステムとのつなぎ込み部分(HiveのDDL/メタストアを利用)や、RDBMSとのつなぎ込み部分(MySQL Connector等)含め、なんというか、本当にいつも大変お世話になっています。

## shibについて
PrestoとHiveにクエリを実行できるWebUI。最初はnodeで作られていて不思議だなーって思ったのですが、このエントリを読んでnodeを選んだ理由がわかり、かっこいいなーと思っております。

- 本家: [shib -WebUI for query engines: Hive and Presto](https://github.com/tagomoris/shib)
- なぜnodeか: [Node.jsなWebアプリでJobQueueなしにラクラク巨大処理を実行](http://d.hatena.ne.jp/tagomoris/20110805/1312536706)


## AWS EMRについて

AWSが提供するAPIで操作できるマネージドHadoop環境。APIを利用してHadoopクラスタを作成し、監視等もよしなにやってくれる。要らなくなったら消しておける。

- 本家: [AWS Elastic MapReduce](http://aws.amazon.com/jp/elasticmapreduce/)
- 知る限り最強のEMR資料: [Best Practices for Amazon EMR](https://media.amazonwebservices.com/AWS_Amazon_EMR_Best_Practices.pdf)


前置きが長くなるとアレなので早速awscliを利用し環境を構築。


## 環境概要

- HadoopはEMRを使ってクラスタ起動
- AMI 3.3.1 (Hadoop Amazon 2.4.0, Hive 0.13.1)
- Hive/HueはEMRデフォルトの機能を使ってインストール
- Hiveメタストアは一旦ローカルで
- PrestoはEMRのBootstrap Actionを使ってインストール(v0.85)
- ShibはEMRのBootstrap Actionを使ってインストール(v0.3.8)
- S3上にあるファイルにPrestoからクエリを投げる
- S3からHDFS上にフラットファイルで持ってきてPrestoからクエリを投げる
- S3からHDFS上にORCファイルに変換して持ってきてPrestoからクエリを投げる


## 構築方法

### AWS CLIの準備

CUIからEMRのAPIを叩くためにawscliを利用します。Python若干古いけどとりあえず実行した環境は以下

- Python 2.7.5
- virtualenv==1.10.1
- virtualenvwrapper==4.1.1
- awscli==1.6.10


```
$ mkvirtualenv emr
(emr)$ pip install awscli
(emr)$ pip freeze | grep awscli
awscli==1.6.10
(emr)$ mkdir ~/.awscli
(emr)$ cat <<-EOF >>  ~/.awscli/config
[profile development]
aws_access_key_id=<development_access_key>
aws_secret_access_key=<development_secret_key>
region=ap-northeast-1
EOF
(emr)$ cat <<-EOF >>  $VIRTUAL_ENV/bin/activate
export AWS_CONFIG_FILE=~/.awscli/config
export AWS_DEFAULT_PROFILE=development
source aws_zsh_completer.sh
EOF
```



### S3に必要なプログラムを配置

2014/12/25時点でPrestoの最新版は0.89ですが、0.86以降はJava8が必要で、EMRで立ち上げるAmazon Linuxに入っているJavaがJava7という都合上今回は0.85を利用して環境構築します。たぶんJava8入れて頑張ればできるはずですがサクッといきましょう。

今回使ったソースは以下

- [presto-server-0.85.tar.gz](https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.85/presto-server-0.85.tar.gz)
- [presto-cli-0.85-executable.jar](https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.85/presto-cli-0.85-executable.jar)
- [shib v0.3.8](https://github.com/tagomoris/shib/tree/v0.3.8)

上記をダウンロードし、適当なS3キーに配置しておきます。以下例。

```
$ wget https://repo1.maven.org/maven2/com/facebook/presto/presto-server/0.85/presto-server-0.85.tar.gz
$ aws s3 cp presto-server-0.85.tar.gz s3://yourbucket/libs/
$ wget https://repo1.maven.org/maven2/com/facebook/presto/presto-cli/0.85/presto-cli-0.85-executable.jar
$ aws s3 cp presto-cli-0.85-executable.jar s3://yourbucket/libs/
$ wget https://github.com/tagomoris/shib/archive/v0.3.8.zip
$ unzip v0.3.8.zip && mv shib-0.3.8 shib
$ tar cvfz shib.tar.gz shib
$ aws s3 cp v0.3.8.zip s3://yourbucket/libs/
```


### S3に必要な設定ファイルを配置

shibがPrestoにクエリするための設定ファイル。

```
var servers = exports.servers = {
  listen: 3000,
  fetch_lines: 1000,   // lines per fetch in shib
  query_timeout: null, // shib waits queries forever
  setup_queries: [],
  storage: {
    datadir: './var'
  },
  engines: [
    { label: 'PrestoCluster',
      executer: {
        name: 'presto',
        host: 'localhost',
        port: 8080,
        catalog: 'hive',  // required configuration argument
        support_database: true,
        default_database: 'default'
      },
      monitor: null
    },
  ],
};
```

```
$ aws s3 cp shib-config.js s3://yourbucket/libs/
```

### S3に必要なBootstrap Actionを配置

Bootstrap ActionとはEMRが管理するHadoopクラスタ構築時に差し挟める任意の処理です。HiveやPig等はデフォルトでEMRのAPIからインストールを指定できるのですが、それ以外のソフトウェアに関してはBootstrap Actionで指定してインストール、設定を行う事になります。

#### PrestoのBootstrap Action
こちらのAWS Labリポジトリにあるので流用します。

- [awslabs/emr-bootstrap-actions](https://github.com/awslabs/emr-bootstrap-actions)
- [Presto BootstrapAction](https://github.com/awslabs/emr-bootstrap-actions/tree/master/presto)

PrestoのBootstrap Actionはそんなオフィシャルにサポートしてはいないですが、READMEもちゃんと書いてあるし、そこまで複雑ではないので要件によっては自作も有りかなと思います。

```
$ wget https://raw.githubusercontent.com/awslabs/emr-bootstrap-actions/master/presto/install-presto.rb
$ aws s3 cp install-presto.rb s3://yourbucket/libs/
```

#### shibのBootstrap Action

shibは公開されているBootstrap Actionが無いのでシェルを雑に書いた。プロセスの立ち上げ部分もnohupとバックグラウンド実行でペイっと起動してるだけなので本番運用に耐えれるしろものではないですが一旦これで。

- install-shib.sh

```
#!/bin/bash

shib_tarball_path=$1
config_file_path=$2

sudo yum -y update
sudo yum install -y git
sudo yum install -y nodejs npm --enablerepo=epel

aws s3 cp ${shib_tarball_path} /home/hadoop/shib.tar.gz
{
  cd /home/hadoop && tar xvfz /home/hadoop/shib.tar.gz
  cd /home/hadoop/shib && npm install
}
aws s3 cp ${config_file_path} /home/hadoop/shib/config.js
{
  cd /home/hadoop/shib && nohup npm start &
}
```

これも同様にS3にあげる。

```
$ aws s3 cp install-shib.sh s3://yourbucket/libs/
```

### AWS CLIでクラスタ立ち上げ

- create-cluster.sh

```
#!/bin/bash

aws emr create-cluster --ami-version 3.3.1 \
    --name "$USER (AMI 3.3.1 Hive + Hue + Presto + shib) $( date '+%Y%m%d%H%M' )" \
    --tags Name=presto-on-emr environment=development \
    --ec2-attributes KeyName=yourkey \
    --applications Name=hive Name=hue \
    --instance-groups file://./large-instance-setup.json \
    --bootstrap-actions file://./bootstrap-presto.json \
    --log-uri 's3://yourbucket/jobflow_logs/' \
    --no-auto-terminate \
    --use-default-roles \
    --visible-to-all-users
```
※AWSアカウントによっては--use-default-rolesをつけているとPrestoからS3にクエリできない事があります。ご注意。

instance-groupとbootstrap-actionsは長くなったのでJSONファイルに切り出しています。内容は以下。

- large-instance-setup.json

```
[
  {
     "Name": "emr-master-production",
     "InstanceGroupType": "MASTER",
     "InstanceCount": 1,
     "InstanceType": "m3.xlarge"
  },
  {
     "Name": "emr-core-production",
     "InstanceGroupType": "CORE",
     "InstanceCount": 2 ,
     "InstanceType": "m3.xlarge"
  }
]
```

- bootstrap-presto.json

```
[
  {
    "Name": "Install/Setup Shib",
    "Path": "s3://yourbucket/libs/install-shib.sh",
    "Args": [
        "s3://yourbucket/libs/shib.tar.gz",
        "s3://yourbucket/libs/shib-config.js"
    ]
  },
  {
    "Name": "Install/Setup Presto",
    "Path": "s3://yourbucket/libs/install-presto.rb",
    "Args": [
        "--s3-path-to-presto-server-bin","s3://yourbucket/libs/presto-server-0.85.tar.gz",
        "--s3-path-to-presto-cli", "s3://yourbucket/libs/presto-cli-0.85-executable.jar"
    ]
  }
]
```



## Shib/Hueアクセス方法

### FoxyProxyとSSHトネリングを利用
FoxyProxyインストールしてXMLを設定。sshコマンドを使ってトネリング。

- [FoxyProxy - Firefox/Chrome](http://getfoxyproxy.org/)


- sample-emr-proxy.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<foxyproxy>
    <proxies>
        <proxy name="emr-socks-proxy" id="2322596116" notes="" fromSubscription="false" enabled="true" mode="manual" selectedTabIndex="2" lastresort="false" animatedIcons="true" includeInCycle="true" color="#0055E5" proxyDNS="true" noInternalIPs="false" autoconfMode="pac" clearCacheBeforeUse="false" disableCache="false" clearCookiesBeforeUse="false" rejectCookies="false">
            <matches>
                <match enabled="true" name="*ec2*.amazonaws.com*" pattern="*ec2*.amazonaws.com*" isRegEx="false" isBlackList="false" isMultiLine="false" caseSensitive="false" fromSubscription="false" />
                <match enabled="true" name="*ec2*.compute*" pattern="*ec2*.compute*" isRegEx="false" isBlackList="false" isMultiLine="false" caseSensitive="false" fromSubscription="false" />
                <match enabled="true" name="10.*" pattern="http://10.*" isRegEx="false" isBlackList="false" isMultiLine="false" caseSensitive="false" fromSubscription="false" />
            </matches>
            <manualconf host="localhost" port="8157" socksversion="5" isSocks="true" username="" password="" domain="" />
        </proxy>
    </proxies>
</foxyproxy>
```


```
$ ssh -i ~/yourkey.pem -ND 8157 hadoop@youremrhost.compute.amazonaws.com
```

これでFoxyProxyがインストールされているブラウザを介すると、以下のURLにアクセスするとShibにアクセスできます。

- http://youremrhost.compute.amazonaws.com:3000

また、以下のURLにアクセスするとHueにアクセスできます。

- http://youremrhost.compute.amazonaws.com:8888


### AWS SecurityGroupを変更
EMRのMasterノードに付与されるSecurityGroupにオフィスのIPからの3000/8888を許可する。これはIPアドレスでのアクセス制限はできますが、通信自体は暗号化されずにHTTPでやりとりされるのであまりお勧めしません。


## パフォーマンス
パフォーマンスもやりたかった。。S3とHDFS FlatFileとHDFS ORCでかなり差がでるのでたぶん後ほど追記します。


## 終わりに
いかがでしたでしょうか。
PrestoとEMRはまだそんなにビッグじゃないけど、大きくなりそうなデータに対して、インタラクティブクエリとスケールしやすさを与えてくれるいいコンビなんじゃないかなと思っております。
もうあと25日は2時間しか残ってませんがハッピーメリークリスマス！


