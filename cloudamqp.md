## [補足資料] CloudAMQPの利用

ローカル環境にRabbitMQがない場合、FREEプランのあるRabbitMQのSaaSである[CloudAMQP](https://www.cloudamqp.com)を使えます。

アカウントを作成するか、GitHubまたはGoogleアカウントでログインしてください。

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/6d040eb3-5ebc-4a56-adaa-779f62c5ce8f">

インスタンス作成画面でNameを入力します。Planは"Little Lemur (Free)"を選択してください。

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/e22dc14a-7e1e-470c-b4de-b2af24318318">

RegionとData Centerは"Amazon Web Services"の"AP_NorthEast-1 (Tokyo)"を選択し、インスタンスを作成してください。

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/c692b2f3-be4a-4e1e-8933-0c0178811f60">


インスタンス一覧画面から作成したインスタンス名をクリックしてください。

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/0abfca3e-ae60-443a-b5af-d7f625327bc2">

インスタンスの接続情報が表示されます。

<img width="1024" alt="image" src="https://github.com/making/blog.ik.am/assets/106908/972a40ba-171b-451c-8d28-0313388b195f">


このRabbitMQインスタンスにローカル環境から接続したい場合は、"URL"の文字列をコピーして、`application.properties`に次のように設定してください。


``` properties
spring.rabbitmq.addresses=amqps://fmzkajiy:2g6hsLdxgPBLtdaSm19C1byLc-chHIZM@cougar.rmq.cloudamqp.com/fmzkajiy
```

> `spring.rabbitmq.host`, `spring.rabbitmq.port`, `spring.rabbitmq.username`, `spring.rabbitmq.password`, `spring.rabbitmq.addresses`を設定する方法もありますが、<br>
> `spring.rabbitmq.addresses` であれば1プロパティで済みます。また、TLSの設定(`spring.rabbitmq.ssl.enabled=true`)も`amqps://...`から自動で判断して設定されます。<br>
> 詳しくは https://docs.spring.io/spring-boot/docs/current/reference/html/messaging.html#messaging.amqp を参照

インスタンス一覧画面から"RabbitMQ Manager"をクリックすると管理コンソールにアクセスできます。

