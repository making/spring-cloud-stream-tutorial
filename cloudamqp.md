## [補足資料] CloudAMQPの利用

ローカル環境にRabbitMQがない場合、あるいはRabbitMQサービスのないCloud Foundryを使用している場合、RabbitMQのSaaSである[CloudAMQP](https://www.cloudamqp.com)を利用できます。

メニューバーから[Plans](https://www.cloudamqp.com/plans.html)を選択し、FREEプランであるLittle Lemurを選択（"Try a Little Lemur now"をクリック）してください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/9c59e9af-440f-f4c8-9f0d-e35fa0aa2302.png)

アカウントを作成するか、GitHubまたはGoogleアカウントでログインしてください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/ebd07f31-c909-6a9e-1856-b08b333155d4.png)

インスタンス作成画面でNameを入力します。任意の値で良いです。Data Centerも選択できるものであればどこでも良いです。(Pivotal Web ServicesはUS-East-1 Regionにいるため、ここではUS-East-1を選択します)

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/97ba6d8e-3b2f-105d-999a-bfbec51afd19.png)

インスタンス一覧画面から作成したインスタンス名をクリックしてください。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/24045eff-53ec-80f1-aa9c-f08087fe2f7b.png)

インスタンスの接続情報が表示されます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/38d4ad62-ece1-ba2a-082e-6cf8c6baa5a5.png)

このRabbitMQインスタンスにローカル環境から接続したい場合は、`application.properties`に次のように設定してください。


``` properties
spring.rabbitmq.addresses=cat.rmq.cloudamqp.com
spring.rabbitmq.username=gapchcgl
spring.rabbitmq.password=WlHWkjO2lWl51zG_CKwNlFaRrXL6Wt3V
spring.rabbitmq.virtual-host=gapchcgl
```

また、Cloud FoundryのUser Provided Serviceを作成したい場合は、次のコマンドを実行してください。

```
cf create-user-provided-service rabbitmq-binder -p '{"uri":"amqp://gapchcgl:WlHWkjO2lWl51zG_CKwNlFaRrXL6Wt3V@cat.rmq.cloudamqp.com/gapchcgl"}'
```

インスタンス一覧画面から"RabbitMQ Manager"をクリックすると管理コンソールにアクセスできます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/67cb765b-f514-67a9-dc57-360c1697ca2b.png)
