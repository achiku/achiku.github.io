---
layout: post
title: 'MQTT as a Service sango + paho-mqtt Python'
---

本日のMQTT(もきゅっと)の会直前に時雨堂さんがMQTT as a Service, sangoをリリースしていたので触ってみた。

[sango](https://sango.shiguredo.jp/)


GitHubアカウントを利用して上記サイトから無料登録する。ログインするとユーザ名、パスワードが出てくるのでメモる。
次にMQTTのPythonクライアントライブラリである[phao-mqtt](https://pypi.python.org/pypi/paho-mqtt)を準備。

{% highlight bash %}
$ pip install phao-mqtt
{% endhighlight %}


これでphao-mqttのPythonライブラリが入るので、サンプルを元に以下のような雑プログラムを作ってみた。

### sub.py
{% highlight python %}
# -*- coding: utf-8 -*-
import paho.mqtt.client as mqtt


def on_connect(client, userdata, flags, rc):
    print('Connected with result code '+str(rc))
    client.subscribe("achiku@github/#")


def on_message(client, userdata, msg):
    print(msg.topic + ' ' + str(msg.payload))


if __name__ == '__main__':

    username = 'yourname@github'
    password = 'yourpass'
    host = 'free.mqtt.shiguredo.jp'
    port = 1883

    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_message = on_message

    client.username_pw_set(username, password=password)
    client.connect(host, port=port, keepalive=60)
    client.loop_forever()
{% endhighlight %}



### pub.py
{% highlight python %}
# -*- coding: utf-8 -*-
import paho.mqtt.client as mqtt
from time import sleep


if __name__ == '__main__':

    username = 'yourname@github'
    password = 'yourpass'
    host = 'free.mqtt.shiguredo.jp'
    topic = 'achiku@github/test_topic'
    port = 1883

    client = mqtt.Client()
    client.username_pw_set(username, password=password)
    client.connect(host, port=port, keepalive=60)

    for i in range(10):
        print '[{}] Sending message to sango.'.format(i)
        client.publish(topic, '[{}] message from pub coming through sango!'.format(i))
        sleep(0.5)
{% endhighlight %}


0.5秒おきにsango MQTT brokerにメッセージをPublishするpub.pyと、sangoが受けているメッセージをSubscribeし続けるsub.pyという形。
まずはsub.pyを起動しておく。

{% highlight bash %}
$ python sub.py
Connected with result code 0
{% endhighlight %}

上のような表示が出れば適切にサービスに繋がり、Subscribeできてる。
次にpub.pyでメッセージをsangoに送る。

{% highlight bash %}
$ python pub.py
[0] Sending message to sango.
[1] Sending message to sango.
[2] Sending message to sango.
[3] Sending message to sango.
[4] Sending message to sango.
[5] Sending message to sango.
[6] Sending message to sango.
[7] Sending message to sango.
[8] Sending message to sango.
[9] Sending message to sango.
{% endhighlight %}

sub.py側で以下のような表示が確認できるはず。


{% highlight bash %}
achiku@github/test_topic [0] message from pub coming through sango!
achiku@github/test_topic [1] message from pub coming through sango!
achiku@github/test_topic [2] message from pub coming through sango!
achiku@github/test_topic [3] message from pub coming through sango!
achiku@github/test_topic [4] message from pub coming through sango!
achiku@github/test_topic [5] message from pub coming through sango!
achiku@github/test_topic [6] message from pub coming through sango!
achiku@github/test_topic [7] message from pub coming through sango!
achiku@github/test_topic [8] message from pub coming through sango!
achiku@github/test_topic [9] message from pub coming through sango!
{% endhighlight %}



それでは新宿であいましょう！！(時間がやばい)

