## Sinkのスケールアウト

次にSinkをスケールアウトしましょう。

### ローカル環境でスケールアウト

現在8082番ポートで起動しているSinkの他に2インスタンスSinkを起動します。

```
java -jar target/hello-sink-0.0.1-SNAPSHOT.jar --server.port=8083
```


```
java -jar target/hello-sink-0.0.1-SNAPSHOT.jar --server.port=8084
```

この状態で、Sourceにリクエストを5回送ります。


```
curl -v localhost:8080 -d '{"text":"Hello1"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello2"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello3"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello4"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello5"}' -H 'Content-Type: application/json'
```

おそらく、3つのSinkそれぞれが順番にメッセージを受信し、に次のようなログが出力されていると思います。

* Sink1

```
Received Hello1
Received Hello4
```

* Sink2

```
Received Hello2
Received Hello5
```

* Sink3

```
Received Hello3
```


> 【ノート】 Consumer Groupの設定がない場合
>
> Sourceからのメッセージの受信が分散されていることを確認できました。これはSpring Cloud Streamでは同じConsumerGroupを持つConsumerのうち最低1つメッセージが届くことが保証されるという特徴があるためです。
> ではConsumer Groupを設定せずに実行するとどうなるでしょうか。
> 
> この場合、Exchangeに対して、Sinkごとに匿名のQueueが出来上がります。
> ![image](https://qiita-image-store.s3.amazonaws.com/0/1852/be80d8f4-03f0-2c79-8cd9-acc17d016c9e.png)
>
> その結果、全てのSinkに次のログが出力されます。
>
> ``` 
> Received Hello1
> Received Hello2
> Received Hello3
> Received Hello4
> Received Hello5
> ``` 


#### Cloud Foundry上でスケールアウト


Cloud Foundry上のスケールアウトはとても簡単です。`cf scale -i <インスタンス数> <アプリケーション名>`でスケールアウト可能です。

Sinkインスタンスを3インスタンスにスケールアウトしましょう。別のターミナルで`cf logs hello-sink-tmaki`を実行し、ログを確認してください。

```
cf scale -i 3 hello-sink-tmaki
```

ログに

```
2017-02-06T00:54:48.89+0900 [CELL/1]     OUT Container became healthy
```

及び

```
2017-02-06T00:54:51.52+0900 [CELL/2]     OUT Container became healthy
```

が出力されたらスケールアウト成功です。ログ中の`[.../1] ...`は2インスタンス目のログ、`[.../2] ...`は3インスタンス目のログを意味します。

この状態で、Sourceにリクエストを5回送ります。

```
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello1"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello2"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello3"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello4"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello5"}' -H 'Content-Type: application/json'
```

次のようなログが出力されてでしょう。

```
2017-02-06T00:57:38.78+0900 [APP/PROC/WEB/0]OUT Received Hello1
2017-02-06T00:57:39.45+0900 [APP/PROC/WEB/1]OUT Received Hello2
2017-02-06T00:57:39.60+0900 [APP/PROC/WEB/2]OUT Received Hello3
2017-02-06T00:57:39.92+0900 [APP/PROC/WEB/0]OUT Received Hello4
2017-02-06T00:57:40.28+0900 [APP/PROC/WEB/1]OUT Received Hello5
```

ローカル環境と同様に3つのSinkにメッセージが分散していることがわかります。


次にSinkを2インスタンスに減らして、

```
cf scale -i 2 hello-sink-tmaki
```


再度、Sourceにリクエストを5回送ります。

```
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello1"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello2"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello3"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello4"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello5"}' -H 'Content-Type: application/json'
```

今回は、2インスタンスで分散されていることがわかります。

```
2017-02-06T00:59:34.86+0900 [APP/PROC/WEB/0]OUT Received Hello1
2017-02-06T00:59:35.18+0900 [APP/PROC/WEB/1]OUT Received Hello2
2017-02-06T00:59:35.57+0900 [APP/PROC/WEB/0]OUT Received Hello3
2017-02-06T00:59:35.96+0900 [APP/PROC/WEB/1]OUT Received Hello4
2017-02-06T00:59:36.51+0900 [APP/PROC/WEB/0]OUT Received Hello5
```

Cloud Foundry上で簡単にSinkを伸縮できることを確認しました。
