# spring-cloud-stream-tutorial

本チュートリアルでは、簡単なSourceとSinkを作成して、Spring Cloud Streamの基本的な開発方法を学びます。

次の図のように、SourceがHTTPで受け付けたTweetメッセージを送信し、Sinkがメッセージを受信してログに出力します。
Message BinderにはRabbitMQを使用します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/a785e432-9a25-c467-b394-a1875b644376.png)

Spring Cloud Streamに関する詳しい情報は
[Event Driven Microservices with Spring Cloud Stream](http://www.slideshare.net/makingx/event-driven-microservices-with-spring-cloud-stream-jjugccc-ccca3)
を参照してください。

* [Sourceの作成](source.md)
* [Sinkの作成](sink.md)
* Sinkのスケールアウト
* 新しいSinkを追加


### 利用規約

無断で本ドキュメントの一部または全部を改変したり、本ドキュメントを用いた二次的著作物を作成することを禁止します。ただし、ドキュメント修正のためのPull Requestは大歓迎です。
