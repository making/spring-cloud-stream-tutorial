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