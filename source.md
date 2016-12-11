## Sourceの作成

まずはSourceから作成します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/bcd1fc24-2966-7765-f4f4-56132dbc61b8.png)


### プロジェクト作成

以下のコマンドを実行すると、`hello-source`フォルダに雛形プロジェクトが生成されます

```
curl start.spring.io/starter.tgz \
       -d artifactId=hello-source \
       -d baseDir=hello-source \
       -d dependencies=web,actuator,cloud-stream-binder-rabbit \
       -d applicationName=HelloSourceApplication | tar -xzvf -
```

> **【ノート】 Web UIからのプロジェクト作成**
>
> `curl`によるプロジェクト作成がうまくいかない場合は[Spring Initializr](https://start.spring.io)にアクセスして、
> 次の項目を入力し、"Generate Project"ボタンをクリックしてください。`hello-source.zip`がダウンロードされるので、これを展開してください。
> 
> * Artifact: hello-source
> * Search for dependecies: Web, Actuator, Stream Rabbit
> 
> ![image](https://qiita-image-store.s3.amazonaws.com/0/1852/acd4935f-4e50-0004-55f4-2d142d8a65a2.png)


### Sourceの実装

Spring Cloud Streamが作成する`MessageChannel`に`Message`を送信するSourceクラスを作成します。ここではHTTPでJSONを受け付けて、`Message`インスタンスを作成し、`Source`インスタンスを利用して`Message`を送信しています。

`@EnableBinding(Source.class)`を指定することにより、`Source`に対応する`MessageChannel`がバインドされ、`Source`インスタンスがインジェクション可能になります。

`src/main/java/com/example/HelloSourceApplication.java`を次のように記述してください。

``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
@EnableBinding(Source.class)
public class HelloSourceApplication {

	private final Source source;

	public HelloSourceApplication(Source source) {
		this.source = source;
	}

	@PostMapping
	public void tweet(@RequestBody Tweet tweet) {
		source.output().send(MessageBuilder.withPayload(tweet).build());
	}

	public static void main(String[] args) {
		SpringApplication.run(HelloSourceApplication.class, args);
	}

	public static class Tweet {
		public String text;
	}
}
```

channel名`output`に対するdestination名とcontent-typeを`application.properties`に次にように設定してください。

``` properties
spring.cloud.stream.bindings.output.destination=hello
spring.cloud.stream.bindings.output.contentType=application/json
```

> **【ノート】 デフォルトchannel名の`output`**
> 
> ここでchanel名は`output`になっていますが、この値はSpring Cloud Streamが用意している`Source`クラスに定義されています。
> 
> ``` java
> package org.springframework.cloud.stream.messaging;
> 
> import org.springframework.cloud.stream.annotation.Output;
> import org.springframework.messaging.MessageChannel;
> 
> public interface Source {
> 
> 	String OUTPUT = "output";
> 
> 	@Output(Source.OUTPUT)
> 	MessageChannel output();
> 
> }
> ```
>
> `output`以外のchannel名を使う場合はこの`Source`クラスを同じようなクラスを作成して、`@EnableBinding`に指定すれば良いです。


> **【ノート】 contentTypeの指定**
> 
> `spring.cloud.stream.bindings.output.contentType=application/json`を指定することで、メッセージのpayloadをJSONにすることができます。
> この指定がない場合は、Java標準のシリアライゼーション機構が利用されます。すなわち受信側が同じクラスを持っていないといけません。基本的にはJSON形式を利用することをお勧めします。



次のコマンドでこのSourceアプリケーションのjarファイルを作成してください。



```
./mvnw clean package -DskipTests=true
```

`target`ディレクトリに`hello-source-0.0.1-SNAPSHOT.jar`が

```
$ ls -lh target/
total 38408
drwxr-xr-x  4 makit  720748206   136B 12 11 16:33 classes
drwxr-xr-x  3 makit  720748206   102B 12 11 16:33 generated-sources
drwxr-xr-x  3 makit  720748206   102B 12 11 16:33 generated-test-sources
-rw-r--r--  1 makit  720748206    19M 12 11 16:33 hello-source-0.0.1-SNAPSHOT.jar
-rw-r--r--  1 makit  720748206   3.8K 12 11 16:33 hello-source-0.0.1-SNAPSHOT.jar.original
drwxr-xr-x  3 makit  720748206   102B 12 11 16:33 maven-archiver
drwxr-xr-x  3 makit  720748206   102B 12 11 16:33 maven-status
drwxr-xr-x  3 makit  720748206   102B 12 11 16:33 test-classe
```

### ローカル環境で実行

ローカル環境でこのアプリケーションを実行する場合はRabbitMQのインストールが必要です。

Mac(brew)の場合は、次のコマンドでインストールしてください。

```
brew install rabbitmq
brew services start rabbitmq
```

次のコマンドでアプリケーションを起動してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar
```

RabbitMQが別のサーバー上にいる場合は次のように`spring.rabbitmq.addresse`を指定してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.addresses=192.168.99.100:5672
```

ユーザー名、パスワードが必要な場合は`spring.rabbitmq.username`、`spring.rabbitmq.password`を指定してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.addresses=192.168.99.100:5672 --spring.rabbitmq.username=user --spring.rabbitmq.password=pass
```

> **【ノート】 `application.properties`にRabbitMQの接続情報を設定**
> 
> RabbitMQの接続情報は起動時に指定するだけでなく、`application.properties`に設定しておくこともできます。この場合は再度jarファイルをビルドしてください。
> 
> ``` properties
> spring.rabbitmq.addresses=192.168.99.100:5672
> spring.rabbitmq.username=user
> spring.rabbitmq.password=pass
> ```

以下のコマンドで、メッセージを送信してください。
```
curl -v localhost:8080 -d '{"text":"Hello"}' -H 'Content-Type: application/json'
```

管理コンソール([http://localhost:15672](http://localhost:15672))にアクセスして、Exchangesタブを選択してください。`hello`Exchangeができていることが確認できます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/589d9326-1640-1375-2bf5-c09f92e0c220.png)

`hello`Exchangeをクリックして、Overviewを開いてください。この状態で再度リクエストを送ると、MessageがこのExchangeに送られていることが見て取れます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/fb35c19b-4cb8-075d-e193-26896b76095a.png)

ただし、この時点ではこのExchangeにはQueueがバインドされておらず、メッセージはそのまま流れていきます。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/13173095-8ce9-c504-d734-69b809dc2937.png)

> **【ノート】 接続しているRabbitMQのヘルスチェック**
>
> Spring Boot Actuatorを追加しているため、`/health`エンドポイントでヘルスチェック結果を見ることができます。接続RabbitMQ(及びSpring Cloud StreamのBinder)のバージョンとステータスがわかります。
> 
> ``` console
> $ curl localhost:8080/health
{"status":"UP","diskSpace":{"status":"UP","total":498937626624,"free":121821605888,"threshold":10485760},"rabbit":{"status":"UP","version":"3.6.1"},"binders":{"status":"UP","rabbit":{"status":"UP","binderHealthIndicator":{"status":"UP","version":"3.6.1"}}}} 
> ```
>
> `jq`コマンドで整形すると次のような表示になります
>
> ``` console
> $ curl -s localhost:8080/health | jq .
> {
>   "status": "UP",
>   "diskSpace": {
>     "status": "UP",
>     "total": 498937626624,
>     "free": 121820958720,
>     "threshold": 10485760
>   },
>   "rabbit": {
>     "status": "UP",
>     "version": "3.6.1"
>   },
>   "binders": {
>     "status": "UP",
>     "rabbit": {
>       "status": "UP",
>       "binderHealthIndicator": {
>         "status": "UP",
>         "version": "3.6.1"
>       }
>     }
>   }
> }
> ```

### Cloud Foundryにデプロイ

次にここで作成したアプリケーションをCloud Foundryにデプロイします。

まずは`hello-source`ディレクトリ直下に次の`manifest.yml`を作成してください。

``` yml
applications:
- name: hello-source-tmaki
  memory: 512m
  buildpack: java_buildpack
  path: target/hello-source-0.0.1-SNAPSHOT.jar
  services:
  - rabbitmq-binder
```

`tmaki`の部分は一意になるように自分のアカウント名などに置換してください。また、商用版Pivotal Cloud Foundryを使用する場合は`java_buildpack`の代わりに`java_buildpack_offline`を使用してください。

`rabbit-binder`サービスインスタンスは後ほど作成します。

次の3種類の環境でそれぞれ使い方を紹介します。違いはRabbitMQサービスインスタンスの作り方だけです。

* [Pivotal Web Services](https://run.pivotal.io) (Cloud Foundryのパブリックサービス)
* [PCF Dev](https://docs.pivotal.io/pcf-dev/) (ローカル環境で利用可能なCloud Foundry)
* RabbitMQサービスのないCloud Foundry

#### Pivotal Web Services

Pivotal Web Serivicesのアカウントがない場合は、[こちら](https://github.com/Pivotal-Japan/cf-workshop/blob/master/pivotal-web-services.md)を参照して作成してください。

Pivotal Web Serivicesにログインしていない場合は次のコマンドでログインしてください。

```
cf login -a api.run.pivotal.io
```

Pivotal Web Servicesの場合は、RabbitMQサービスとして[CloudAMQP](https://www.cloudamqp.com/)との連携が用意されています。`cloudamqp`というサービス名でfreeブランとして`lemur`を利用できます。次のコマンドで`rabbitmq-binder`サービスインスタンスを作成してください。

``` console
$ cf create-service cloudamqp lemur rabbitmq-binder
Creating service instance rabbitmq-binder in org *** / space production as ***...
OK
```

サービスインスタンスが作成されればあとは`cf push`コマンドでアプリケーションをデプロイできます。

``` console
$ cf push
Using manifest file /Users/makit/scst-ws/hello-source/manifest.yml

Creating app hello-source-tmaki in org *** / space production as ***...
OK

Creating route hello-source-tmaki.cfapps.io...
OK

Binding hello-source-tmaki.cfapps.io to hello-source-tmaki...
OK

Uploading hello-source-tmaki...
Uploading app files from: /var/folders/15/fww24j3d7pg9sz196cxv_6xm4nvlh8/T/unzipped-app180342590
Uploading 517.8K, 100 files
Done uploading               
OK
Binding service rabbitmq-binder to app hello-source-tmaki in org APJ / space production as tmaki@pivotal.io...
OK

Starting app hello-source-tmaki in org APJ / space production as tmaki@pivotal.io...
Downloading java_buildpack...
Downloaded java_buildpack
Creating container
Successfully created container
Downloading app package...
Downloaded app package (16.6M)
Staging...
-----> Java Buildpack Version: v3.10 (offline) | https://github.com/cloudfoundry/java-buildpack.git#193d6b7
-----> Downloading Open Jdk JRE 1.8.0_111 from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_111.tar.gz (found in cache)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.1s)
-----> Downloading Open JDK Like Memory Calculator 2.0.2_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/trusty/x86_64/memory-calculator-2.0.2_RELEASE.tar.gz (found in cache)
       Memory Settings: -Xms681574K -XX:MetaspaceSize=104857K -Xss349K -Xmx681574K -XX:MaxMetaspaceSize=104857K
-----> Downloading Spring Auto Reconfiguration 1.10.0_RELEASE from https://java-buildpack.cloudfoundry.org/auto-reconfiguration/auto-reconfiguration-1.10.0_RELEASE.jar (found in cache)
Exit status 0
Staging complete
Uploading droplet, build artifacts cache...
Uploading build artifacts cache...
Uploading droplet...
Uploaded build artifacts cache (109B)
Uploaded droplet (61.7M)
Uploading complete
Destroying container
Successfully destroyed container

0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
1 of 1 instances running

App started


OK

App hello-source-tmaki was started using this command `CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-2.0.2_RELEASE -memorySizes=metaspace:64m..,stack:228k.. -memoryWeights=heap:65,metaspace:10,native:15,stack:10 -memoryInitials=heap:100%,metaspace:100% -stackThreads=300 -totMemory=$MEMORY_LIMIT) && JAVA_OPTS="-Djava.io.tmpdir=$TMPDIR -XX:OnOutOfMemoryError=$PWD/.java-buildpack/open_jdk_jre/bin/killjava.sh $CALCULATED_MEMORY" && SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher`

Showing health and status for app hello-source-tmaki in org *** / space production as ***...
OK

requested state: started
instances: 1/1
usage: 512M x 1 instances
urls: hello-source-tmaki.cfapps.io
last uploaded: Sun Dec 11 08:42:21 UTC 2016
stack: cflinuxfs2
buildpack: java_buildpack

     state     since                    cpu      memory           disk           details
#0   running   2016-12-11 05:43:23 PM   217.6%   300.3M of 512M   141.7M of 1G
```

デプロイができたら次のコマンドでメッセージを送信しましょう。

```
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello"}' -H 'Content-Type: application/json'
```

> **【ノート】 RabbitMQサービスインスタンスの管理コンソールURL**
>
> `cf create-service`コマンドで作成したサービスインスタンスの管理コンソールのURLは`cf service <サービスインスタンス名>`で出力内容の`Dashboard`に記載されています。
>
> ``` console
> $ cf service rabbitmq-binder
> 
> Service instance: rabbitmq-binder
> Service: cloudamqp
> Bound apps: 
> Tags: 
> Plan: lemur
> Description: Managed HA RabbitMQ servers in the cloud
> Documentation url: http://docs.run.pivotal.io/marketplace/services/cloudamqp.html
> Dashboard: https://cloudfoundry.appdirect.com/api/custom/cloudfoundry/v2/sso/start?serviceUuid=f6f7b4ae-55c0-4df0-a450-53fb6d1cebb3
> 
> Last Operation
> Status: update succeeded
> Message: 
> Started: 2016-12-11T08:34:24Z
> Updated: 2016-12-11T08:34:24Z
> ```
> ![image](https://qiita-image-store.s3.amazonaws.com/0/1852/0a88d366-2ff2-bbb5-9f4f-81735fcb517c.png)


> **【ノート】 Spring Auto-Reconfiguration**
>
>  今回のデプロイではRabbitMQサービスインスタンスへの接続情報はアプリケーションに一切設定しませんでした。実はアプリケーションにバインドされているサービスインスタンスを自動で認識して起動時に`org.springframework.amqp.rabbit.connection.ConnectionFactory`インスタンスが生成され、DIコンテナに登録されます。起動時に次のログが出力されていることが確認できます。[Spring Auto-Reconfiguration](https://github.com/cloudfoundry/java-buildpack-auto-reconfiguration)がこの役割を担います。内部で[Spring Cloud Connectors](http://cloud.spring.io/spring-cloud-connectors/spring-cloud-cloud-foundry-connector.html)が使用されています。
>
> ```
> 2016-12-11T19:10:19.477+09:00 [APP/PROC/WEB/0] [OUT] 2016-12-11 10:10:19.472 INFO 13 --- [ main] bbitCloudServiceBeanFactoryPostProcessor : > Auto-reconfiguring beans of type org.springframework.amqp.rabbit.connection.ConnectionFactory
> 2016-12-11T19:10:19.558+09:00 [APP/PROC/WEB/0] [OUT] 2016-12-11 10:10:19.549 INFO 13 --- [ main] bbitCloudServiceBeanFactoryPostProcessor : > Reconfigured bean rabbitConnectionFactory into singleton service connector CachingConnectionFactory [channelCacheSize=25, host=zebra.rmq.cloudamqp.com, port=5672, active=true org.springframework.amqp.rabbit.connection.CachingConnectionFactory@73d4cc9e]
> ```
> 
> Spring Auto-Reconfigurationによる自動設定は便利な反面、コネクション数の設定ができません。コネクション数の設定を行う場合はSpring Cloud Connectorsを明示的に定義する必要があります。[こちら](https://github.com/Pivotal-Japan/cf-workshop/blob/master/backend-service-redis_java.md#spring-cloud-connectors)を参照してください。

#### PCF Dev


PCF Devをインストールしていない場合は、[こちら](https://network.pivotal.io/products/pcfdev)からダウンロードして、以下のコマンドでインストールしてください。

```
cf install-plugin <ダウンロードしたファイル>
```

PCF Devを起動していない場合は次のコマンドで起動してください。

```
$ cf dev start
Using existing image.
Allocating 4096 MB out of 16384 MB total system memory (6335 MB free).
Importing VM...
Starting VM...
Provisioning VM...
Waiting for services to start...
8 out of 56 running
8 out of 56 running
8 out of 56 running
40 out of 56 running
54 out of 56 running
56 out of 56 running
 _______  _______  _______    ______   _______  __   __
|       ||       ||       |  |      | |       ||  | |  |
|    _  ||       ||    ___|  |  _    ||    ___||  |_|  |
|   |_| ||       ||   |___   | | |   ||   |___ |       |
|    ___||      _||    ___|  | |_|   ||    ___||       |
|   |    |     |_ |   |      |       ||   |___  |     |
|___|    |_______||___|      |______| |_______|  |___|
is now running.
To begin using PCF Dev, please run:
   cf login -a https://api.local.pcfdev.io --skip-ssl-validation
Apps Manager URL: https://local.pcfdev.io
Admin user => Email: admin / Password: admin
Regular user => Email: user / Password: pass
```

ログインしていない場合は次のコマンドでログインしてください。

```
cf login -a api.local.pcfdev.io --skip-ssl-validation -u user -p pass
```

PCF Devの場合は、RabbitMQサービスとして`p-rabbitmq`というサービス名でfreeブランとして`standard`を利用できます。次のコマンドで`rabbitmq-binder`サービスインスタンスを作成してください。

``` console
$ cf create-service p-rabbitmq standard rabbitmq-binder
Creating service instance rabbitmq-binder in org pcfdev-org / space pcfdev-space as user...
OK
```

サービスインスタンスが作成されればあとは`cf push`コマンドでアプリケーションをデプロイできます。


```
$ cf push
Using manifest file /Users/makit/scst-ws/hello-source/manifest.yml

Updating app hello-source-tmaki in org pcfdev-org / space pcfdev-space as user...
OK

Uploading hello-source-tmaki...
Uploading app files from: /var/folders/15/fww24j3d7pg9sz196cxv_6xm4nvlh8/T/unzipped-app542166054
Uploading 517.8K, 100 files
Done uploading               
OK
Binding service rabbitmq-binder to app hello-source-tmaki in org pcfdev-org / space pcfdev-space as user...
OK

Starting app hello-source-tmaki in org pcfdev-org / space pcfdev-space as user...
Downloading java_buildpack...
Downloaded java_buildpack (246M)
Creating container
Successfully created container
Downloading app package...
Downloaded app package (16.6M)
Staging...
-----> Java Buildpack Version: v3.8.1 (offline) | https://github.com/cloudfoundry/java-buildpack.git#29c79f2
-----> Downloading Open Jdk JRE 1.8.0_91-unlimited-crypto from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_91-unlimited-crypto.tar.gz (found in cache)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.1s)
-----> Downloading Open JDK Like Memory Calculator 2.0.2_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/trusty/x86_64/memory-calculator-2.0.2_RELEASE.tar.gz (found in cache)
       Memory Settings: -Xmx317161K -XX:MaxMetaspaceSize=64M -Xss228K -Xms317161K -XX:MetaspaceSize=64M
-----> Downloading Spring Auto Reconfiguration 1.10.0_RELEASE from https://java-buildpack.cloudfoundry.org/auto-reconfiguration/auto-reconfiguration-1.10.0_RELEASE.jar (found in cache)
Exit status 0
Staging complete
Uploading droplet, build artifacts cache...
Uploading droplet...
Uploading build artifacts cache...
Uploaded build artifacts cache (109B)
Uploaded droplet (62.2M)
Uploading complete
Destroying container
Successfully destroyed container

0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
0 of 1 instances running, 1 starting
1 of 1 instances running

App started


OK

App hello-source-tmaki was started using this command `CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-2.0.2_RELEASE -memorySizes=metaspace:64m..,stack:228k.. -memoryWeights=heap:65,metaspace:10,native:15,stack:10 -memoryInitials=heap:100%,metaspace:100% -stackThreads=300 -totMemory=$MEMORY_LIMIT) && JAVA_OPTS="-Djava.io.tmpdir=$TMPDIR -XX:OnOutOfMemoryError=$PWD/.java-buildpack/open_jdk_jre/bin/killjava.sh $CALCULATED_MEMORY" && SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher`

Showing health and status for app hello-source-tmaki in org pcfdev-org / space pcfdev-space as user...
OK

requested state: started
instances: 1/1
usage: 512M x 1 instances
urls: hello-source-tmaki.local.pcfdev.io
last uploaded: Sun Dec 11 09:50:34 UTC 2016
stack: cflinuxfs2
buildpack: java_buildpack

     state     since                    cpu      memory           disk             details
#0   running   2016-12-11 06:51:26 PM   193.5%   285.4M of 512M   141.4M of 512M
```

デプロイができたら次のコマンドでメッセージを送信しましょう。

```
curl hello-source-tmaki.local.pcfdev.io -d '{"text":"Hello"}' -H 'Content-Type: application/json'
```

#### RabbitMQサービスのないCloud Foundry



ログインしていない場合は次のコマンドでログインしてください。

```
cf login -a api.<your system domain> --skip-ssl-validation
```

OSSのCloud Foundryをインストールした後などのRabbitMQが存在しない場合は、User Provided Serviceを利用します。

```
cf update-user-provided-service rabbitmq-binder -p '{"uri":"amqp://<username>:<password>@<hostname>/<vhost>"}'
```

`uri`にこの形式を指定することによりSpring Auto-ReconfigurationでSpring Cloud Connectorsを使ったRabbitMQへの自動接続が有効になります。有効条件についてはドキュメントを参照してください。後は`cf push`すれば良いです。


``` console
$ cf push
```

### Unit Testの作成

Spring Cloud Streamにはテスト支援機能も用意されています。テスト時はMessage Binderのインメモリ実装が使われるようになります。これにより、開発中はMessage Binder(RabbitMQ)を用意しなくてもテストを進めることができます。

`src/test/java/com/example/HelloSourceApplicationTests.java`に次の内容を記述してください。

``` java
package com.example;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.stream.messaging.Source;
import org.springframework.cloud.stream.test.binder.MessageCollector;
import org.springframework.messaging.Message;
import org.springframework.test.context.junit4.SpringRunner;

import com.example.HelloSourceApplication.Tweet;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
public class HelloSourceApplicationTests {

	@Autowired
	HelloSourceApplication app;
	@Autowired
	MessageCollector collector;
	@Autowired
	Source source;

	@Test
	@SuppressWarnings("unchecked")
	public void testTweet() {
		Tweet tweet = new Tweet();
		tweet.text = "hello!";
		app.tweet(tweet);

		Message<String> message = (Message<String>) collector.forChannel(source.output())
				.poll();

		assertThat(message.getPayload()).isInstanceOf(String.class);
		assertThat(message.getPayload()).isEqualTo("{\"text\":\"hello!\"}");
		assertThat(message.getHeaders().get("contentType").toString())
				.isEqualTo("application/json;charset=UTF-8");
	}

}
```

`spring.cloud.stream.bindings.output.contentType=application/json`の設定をしているため、`Message`のpayloadがJSON文字列になっていることに注意してください。


次のようにテストが通ればOKです。

```
$ ./mvnw test
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-source ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-source ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ hello-source ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /Users/makit/scst-ws/hello-source/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ hello-source ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-surefire-plugin:2.18.1:test (default-test) @ hello-source ---
[INFO] Surefire report directory: /Users/makit/scst-ws/hello-source/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
20:37:37.886 [main] DEBUG org.springframework.test.context.junit4.SpringJUnit4ClassRunner - SpringJUnit4ClassRunner constructor called with [class com.example.HelloSourceApplicationTests]
20:37:37.897 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
20:37:37.922 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
20:37:37.963 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSourceApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
20:37:37.980 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSourceApplicationTests], using SpringBootContextLoader
20:37:37.987 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTests-context.xml] does not exist
20:37:37.987 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTestsContext.groovy] does not exist
20:37:37.987 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSourceApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
20:37:37.989 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSourceApplicationTests]: HelloSourceApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
20:37:38.049 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSourceApplicationTests]
20:37:38.128 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
20:37:38.129 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
20:37:38.130 [main] DEBUG org.springframework.core.env.StandardEnvironment - Initialized StandardEnvironment with PropertySources [systemProperties,systemEnvironment]
20:37:38.147 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Resolved classpath location [com/example/] to resources [URL [file:/Users/makit/scst-ws/hello-source/target/test-classes/com/example/], URL [file:/Users/makit/scst-ws/hello-source/target/classes/com/example/]]
20:37:38.148 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Looking for matching resources in directory tree [/Users/makit/scst-ws/hello-source/target/test-classes/com/example]
20:37:38.148 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Searching directory [/Users/makit/scst-ws/hello-source/target/test-classes/com/example] for files matching pattern [/Users/makit/scst-ws/hello-source/target/test-classes/com/example/*.class]
20:37:38.153 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Looking for matching resources in directory tree [/Users/makit/scst-ws/hello-source/target/classes/com/example]
20:37:38.153 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Searching directory [/Users/makit/scst-ws/hello-source/target/classes/com/example] for files matching pattern [/Users/makit/scst-ws/hello-source/target/classes/com/example/*.class]
20:37:38.153 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Resolved location pattern [classpath*:com/example/*.class] to resources [file [/Users/makit/scst-ws/hello-source/target/test-classes/com/example/HelloSourceApplicationTests.class], file [/Users/makit/scst-ws/hello-source/target/classes/com/example/HelloSourceApplication$Tweet.class], file [/Users/makit/scst-ws/hello-source/target/classes/com/example/HelloSourceApplication.class]]
20:37:38.236 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider - Identified candidate component class: file [/Users/makit/scst-ws/hello-source/target/classes/com/example/HelloSourceApplication.class]
20:37:38.236 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSourceApplication for test class com.example.HelloSourceApplicationTests
20:37:38.239 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSourceApplicationTests]: using defaults.
20:37:38.244 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener]
20:37:38.278 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@6e75aa0d, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@7fc229ab, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@2cbb3d47, org.springframework.test.context.support.DirtiesContextTestExecutionListener@527e5409, org.springframework.test.context.transaction.TransactionalTestExecutionListener@1198b989, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@7ff95560, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@add0edd, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@2aa3cd93, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@7ea37dbf, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@4b44655e, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@290d210d, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@1d76aeea]
20:37:38.280 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
20:37:38.280 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
Running com.example.HelloSourceApplicationTests
20:37:38.282 [main] DEBUG org.springframework.test.context.junit4.SpringJUnit4ClassRunner - SpringJUnit4ClassRunner constructor called with [class com.example.HelloSourceApplicationTests]
20:37:38.282 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
20:37:38.282 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
20:37:38.282 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSourceApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
20:37:38.283 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSourceApplicationTests], using SpringBootContextLoader
20:37:38.283 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTests-context.xml] does not exist
20:37:38.284 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTestsContext.groovy] does not exist
20:37:38.284 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSourceApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
20:37:38.285 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSourceApplicationTests]: HelloSourceApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
20:37:38.295 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSourceApplicationTests]
20:37:38.297 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
20:37:38.298 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
20:37:38.298 [main] DEBUG org.springframework.core.env.StandardEnvironment - Initialized StandardEnvironment with PropertySources [systemProperties,systemEnvironment]
20:37:38.298 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSourceApplication for test class com.example.HelloSourceApplicationTests
20:37:38.301 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSourceApplicationTests]: using defaults.
20:37:38.304 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener]
20:37:38.309 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@6e0dec4a, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@96def03, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@5ccddd20, org.springframework.test.context.support.DirtiesContextTestExecutionListener@1ed1993a, org.springframework.test.context.transaction.TransactionalTestExecutionListener@1f3f4916, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@794cb805, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@4b5a5ed1, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@59d016c9, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@3cc2931c, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@20d28811, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@3967e60c, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@60d8c9b7]
20:37:38.310 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
20:37:38.310 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
20:37:38.311 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
20:37:38.311 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
20:37:38.315 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener - Before test class: context [DefaultTestContext@47d90b9e testClass = HelloSourceApplicationTests, testInstance = [null], testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@1184ab05 testClass = HelloSourceApplicationTests, locations = '{}', classes = '{class com.example.HelloSourceApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.context.SpringBootTestContextCustomizer@2758fe70, org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@551aa95a, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@7f13d6e], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]]], class annotated with @DirtiesContext [false] with mode [null].
20:37:38.316 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
20:37:38.316 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
20:37:38.318 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener - Performing dependency injection for test context [[DefaultTestContext@47d90b9e testClass = HelloSourceApplicationTests, testInstance = com.example.HelloSourceApplicationTests@6ef888f6, testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@1184ab05 testClass = HelloSourceApplicationTests, locations = '{}', classes = '{class com.example.HelloSourceApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.context.SpringBootTestContextCustomizer@2758fe70, org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@551aa95a, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@7f13d6e], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]]]].
20:37:38.343 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
20:37:38.343 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
20:37:38.344 [main] DEBUG org.springframework.core.env.StandardEnvironment - Initialized StandardEnvironment with PropertySources [systemProperties,systemEnvironment]
20:37:38.345 [main] DEBUG org.springframework.test.context.support.TestPropertySourceUtils - Adding inlined properties to environment: {spring.jmx.enabled=false, org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true, server.port=-1}
20:37:38.345 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [Inlined Test Properties] PropertySource with highest search precedence

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.4.2.RELEASE)

2016-12-11 20:37:38.779  INFO 64194 --- [           main] com.example.HelloSourceApplicationTests  : Starting HelloSourceApplicationTests on jpxxmakitm1.corp.emc.com with PID 64194 (started by makit in /Users/makit/scst-ws/hello-source)
2016-12-11 20:37:38.780  INFO 64194 --- [           main] com.example.HelloSourceApplicationTests  : No active profile set, falling back to default profiles: default
2016-12-11 20:37:38.818  INFO 64194 --- [           main] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@3e78b6a5: startup date [Sun Dec 11 20:37:38 JST 2016]; root of context hierarchy
2016-12-11 20:37:39.635  INFO 64194 --- [           main] o.s.b.f.config.PropertiesFactoryBean     : Loading properties file from URL [jar:file:/Users/makit/.m2/repository/org/springframework/integration/spring-integration-core/4.3.5.RELEASE/spring-integration-core-4.3.5.RELEASE.jar!/META-INF/spring.integration.default.properties]
2016-12-11 20:37:39.639  INFO 64194 --- [           main] o.s.i.config.IntegrationRegistrar        : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2016-12-11 20:37:39.682  INFO 64194 --- [           main] o.s.b.f.s.DefaultListableBeanFactory     : Overriding bean definition for bean 'binderFactory' with a different definition: replacing [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=org.springframework.cloud.stream.config.BinderFactoryConfiguration; factoryMethodName=binderFactory; initMethodName=null; destroyMethodName=(inferred); defined in org.springframework.cloud.stream.config.BinderFactoryConfiguration] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=org.springframework.cloud.stream.test.binder.TestSupportBinderAutoConfiguration; factoryMethodName=binderFactory; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [org/springframework/cloud/stream/test/binder/TestSupportBinderAutoConfiguration.class]]
2016-12-11 20:37:40.443  INFO 64194 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2016-12-11 20:37:40.449  INFO 64194 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'taskScheduler' has been explicitly defined. Therefore, a default ThreadPoolTaskScheduler will be created.
2016-12-11 20:37:40.584  INFO 64194 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.amqp.rabbit.annotation.RabbitBootstrapConfiguration' of type [class org.springframework.amqp.rabbit.annotation.RabbitBootstrapConfiguration$$EnhancerBySpringCGLIB$$2ebeb4b5] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 20:37:40.642  INFO 64194 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.stream.config.ChannelBindingServiceConfiguration$PostProcessorConfiguration' of type [class org.springframework.cloud.stream.config.ChannelBindingServiceConfiguration$PostProcessorConfiguration$$EnhancerBySpringCGLIB$$601d8f13] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 20:37:40.767  INFO 64194 --- [           main] o.s.b.f.config.PropertiesFactoryBean     : Loading properties file from URL [jar:file:/Users/makit/.m2/repository/org/springframework/integration/spring-integration-core/4.3.5.RELEASE/spring-integration-core-4.3.5.RELEASE.jar!/META-INF/spring.integration.default.properties]
2016-12-11 20:37:40.768  INFO 64194 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationGlobalProperties' of type [class org.springframework.beans.factory.config.PropertiesFactoryBean] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 20:37:40.770  INFO 64194 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationGlobalProperties' of type [class java.util.Properties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 20:37:41.410  INFO 64194 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService  'taskScheduler'
2016-12-11 20:37:42.381  INFO 64194 --- [           main] o.s.i.codec.kryo.CompositeKryoRegistrar  : configured Kryo registration [40, java.io.File] with serializer org.springframework.integration.codec.kryo.FileSerializer
2016-12-11 20:37:42.530  INFO 64194 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase -2147482648
2016-12-11 20:37:42.555  INFO 64194 --- [           main] o.s.integration.channel.DirectChannel    : Channel 'application:-1.output' has 1 subscriber(s).
2016-12-11 20:37:42.556  INFO 64194 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 0
2016-12-11 20:37:42.556  INFO 64194 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2016-12-11 20:37:42.557  INFO 64194 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application:-1.errorChannel' has 1 subscriber(s).
2016-12-11 20:37:42.557  INFO 64194 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started _org.springframework.integration.errorLogger
2016-12-11 20:37:42.557  INFO 64194 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 2147482647
2016-12-11 20:37:42.557  INFO 64194 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 2147483647
2016-12-11 20:37:42.586  INFO 64194 --- [           main] com.example.HelloSourceApplicationTests  : Started HelloSourceApplicationTests in 4.238 seconds (JVM running for 5.343)
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.439 sec - in com.example.HelloSourceApplicationTests
2016-12-11 20:37:42.734  INFO 64194 --- [       Thread-1] s.c.a.AnnotationConfigApplicationContext : Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@3e78b6a5: startup date [Sun Dec 11 20:37:38 JST 2016]; root of context hierarchy
2016-12-11 20:37:42.736  INFO 64194 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 2147483647
2016-12-11 20:37:42.737  INFO 64194 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 2147482647
2016-12-11 20:37:42.738  INFO 64194 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 0
2016-12-11 20:37:42.738  INFO 64194 --- [       Thread-1] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2016-12-11 20:37:42.738  INFO 64194 --- [       Thread-1] o.s.i.channel.PublishSubscribeChannel    : Channel 'application:-1.errorChannel' has 0 subscriber(s).
2016-12-11 20:37:42.739  INFO 64194 --- [       Thread-1] o.s.i.endpoint.EventDrivenConsumer       : stopped _org.springframework.integration.errorLogger
2016-12-11 20:37:42.739  INFO 64194 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase -2147482648
2016-12-11 20:37:42.745  INFO 64194 --- [       Thread-1] o.s.s.c.ThreadPoolTaskScheduler          : Shutting down ExecutorService 'taskScheduler'

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 7.861 s
[INFO] Finished at: 2016-12-11T20:37:42+09:00
[INFO] Final Memory: 18M/309M
[INFO] ------------------------------------------------------------------------
```
