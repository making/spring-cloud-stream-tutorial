## Sourceの作成

まずはSourceから作成します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/bcd1fc24-2966-7765-f4f4-56132dbc61b8.png)


### プロジェクト作成

以下のコマンドを実行すると、`hello-source`フォルダに雛形プロジェクトが生成されます

```
curl start.spring.io/starter.tgz \
       -d artifactId=hello-source \
       -d baseDir=hello-source \
       -d packageName=com.example \
       -d dependencies=web,actuator,cloud-stream,amqp \
       -d applicationName=HelloSourceApplication | tar -xzvf -
```

> **【ノート】 Web UIからのプロジェクト作成**
>
> `curl`によるプロジェクト作成がうまくいかない場合は[Spring Initializr](https://start.spring.io)にアクセスして、
> 次の項目を入力し、"Generate Project"ボタンをクリックしてください。`hello-source.zip`がダウンロードされるので、これを展開してください。
> 
> * Artifact: hello-source
> * Search for dependecies: Web, Actuator, Cloud Stream, RabbitMQ
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



次のコマンドでこのSourceアプリケーションのjarファイルを作成してください。



```
./mvnw clean package -DskipTests=true
```

`target`ディレクトリに`hello-source-0.0.1-SNAPSHOT.jar`ができていることを確認してください。

```
$ ls -lh target/
total 49912
drwxr-xr-x  10 maki  staff   320B  1 28 02:42 .
drwxr-xr-x  11 maki  staff   352B  1 28 02:42 ..
drwxr-xr-x   4 maki  staff   128B  1 28 02:42 classes
drwxr-xr-x   3 maki  staff    96B  1 28 02:42 generated-sources
drwxr-xr-x   3 maki  staff    96B  1 28 02:42 generated-test-sources
-rw-r--r--   1 maki  staff    24M  1 28 02:42 hello-source-0.0.1-SNAPSHOT.jar
-rw-r--r--   1 maki  staff   3.7K  1 28 02:42 hello-source-0.0.1-SNAPSHOT.jar.original
drwxr-xr-x   3 maki  staff    96B  1 28 02:42 maven-archiver
drwxr-xr-x   3 maki  staff    96B  1 28 02:42 maven-status
drwxr-xr-x   3 maki  staff    96B  1 28 02:42 test-classes
```

### ローカル環境で実行

ローカル環境でこのアプリケーションを実行する場合はRabbitMQのインストールが必要です。

Mac(brew)の場合は、次のコマンドでインストールしてください。

```
brew install rabbitmq
brew services start rabbitmq
```
Dockerを使う場合は、次のコマンドを実行してください。

```
docker run -p 15672:15672 -p 5672:5672 rabbitmq:3-management
```

次のコマンドでアプリケーションを起動してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar
```

RabbitMQが別のサーバー上にいる場合は次のように`spring.rabbitmq.addresses`を指定してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.addresses=192.168.99.100:5672
```

ユーザー名、パスワードが必要な場合は`spring.rabbitmq.username`、`spring.rabbitmq.password`を指定してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.addresses=192.168.99.100:5672 --spring.rabbitmq.username=user --spring.rabbitmq.password=pass
```

Virtual Hostを設定している場合は`spring.rabbitmq.virtual-host`を指定してください。
```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.addresses=192.168.99.100:5672 --spring.rabbitmq.username=user --spring.rabbitmq.password=pass --spring.rabbitmq.virtual-host=/
```


> **【ノート】 `application.properties`にRabbitMQの接続情報を設定**
> 
> RabbitMQの接続情報は起動時に指定するだけでなく、`application.properties`に設定しておくこともできます。この場合は再度jarファイルをビルドしてください。
> 
> ``` properties
> spring.rabbitmq.addresses=192.168.99.100:5672
> spring.rabbitmq.username=user
> spring.rabbitmq.password=pass
> spring.rabbitmq.virtual-host=/ # if needed
> ```
>
> CloudAMQPを利用する場合は「[[補足資料] CloudAMQPの利用](cloudamqp.md)」を参照してください。


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
> Spring Boot Actuatorを追加しているため、`/actuator/health`エンドポイントでヘルスチェック結果を見ることができます。
> 
> ``` console
> $ curl localhost:8080/actuator/health
> {"status":"UP"}
> ```
> 
> `management.endpoint.health.show-details`プロパティを`always`にすればヘルスチェックの詳細も表示されます。
>
> ```
> java -jar target/hello-source-0.0.1-SNAPSHOT.jar --management.endpoint.health.show-details=always
> ```
> 
> ``` console
> $ curl localhost:8080/actuator/health
> {"status":"UP","details":{"rabbit":{"status":"UP","details":{"version":"3.7.2"}},"diskSpace":{"status":"UP","details":{"total":499963170816,"free":49994924032,"threshold":10485760}},"binders":{"status":"UP","details":{"rabbit":{"status":"UP","details":{"binderHealthIndicator":{"status":"UP","details":{"version":"3.7.2"}}}}}}}}
> ```

### Cloud Foundryにデプロイ

次にここで作成したアプリケーションをCloud Foundryにデプロイします。

まずは`hello-source`ディレクトリ直下に次の`manifest.yml`を作成してください。

``` yml
applications:
- name: hello-source-tmaki
  memory: 768m
  path: target/hello-source-0.0.1-SNAPSHOT.jar
  services:
  - rabbitmq-binder
```

`rabbit-binder`サービスインスタンスは後ほど作成します。

次の3種類の環境でそれぞれ使い方を紹介します。違いはRabbitMQサービスインスタンスの作り方だけです。

* [Pivotal Web Services](https://run.pivotal.io) (Cloud Foundryのパブリックサービス)
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
Binding service rabbitmq-binder to app hello-source-tmaki in org *** / space production as ***...
OK

Starting app hello-source-tmaki in org *** / space production as ***...
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

#### RabbitMQサービスのないCloud Foundry



ログインしていない場合は次のコマンドでログインしてください。

```
cf login -a api.<your system domain> --skip-ssl-validation
```

OSSのCloud Foundryをインストールした後などのRabbitMQが存在しない場合は、User Provided Serviceを利用します。

```
cf create-user-provided-service rabbitmq-binder -p '{"uri":"amqp://<username>:<password>@<hostname>/<vhost>"}'
```

CloudAMQPを利用する場合は「[[補足資料] CloudAMQPの利用](cloudamqp.md)」を参照してください。

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
				.isEqualTo("application/json");
	}

}
```

`Message`のpayloadがJSON文字列になっていることに注意してください。


次のようにテストが通ればOKです。

``` console
 $ ./mvnw test
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------------< com.example:hello-source >----------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-resources-plugin:3.1.0:resources (default-resources) @ hello-source ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.0:compile (default-compile) @ hello-source ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/maki/git/scst/hello-source/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.1.0:testResources (default-testResources) @ hello-source ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /Users/maki/git/scst/hello-source/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.0:testCompile (default-testCompile) @ hello-source ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/maki/git/scst/hello-source/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.1:test (default-test) @ hello-source ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.HelloSourceApplicationTests
02:51:55.484 [main] DEBUG org.springframework.test.context.junit4.SpringJUnit4ClassRunner - SpringJUnit4ClassRunner constructor called with [class com.example.HelloSourceApplicationTests]
02:51:55.489 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
02:51:55.495 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
02:51:55.511 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSourceApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
02:51:55.521 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSourceApplicationTests], using SpringBootContextLoader
02:51:55.526 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTests-context.xml] does not exist
02:51:55.534 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTestsContext.groovy] does not exist
02:51:55.535 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSourceApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
02:51:55.537 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSourceApplicationTests]: HelloSourceApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
02:51:55.585 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSourceApplicationTests]
02:51:55.679 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider - Identified candidate component class: file [/Users/maki/git/scst/hello-source/target/classes/com/example/HelloSourceApplication.class]
02:51:55.680 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSourceApplication for test class com.example.HelloSourceApplicationTests
02:51:55.750 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSourceApplicationTests]: using defaults.
02:51:55.751 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener]
02:51:55.770 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@7d3d101b, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@30c8681, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@5cdec700, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@6d026701, org.springframework.test.context.support.DirtiesContextTestExecutionListener@78aa1f72, org.springframework.test.context.transaction.TransactionalTestExecutionListener@1f75a668, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@35399441, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@4b7dc788, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@6304101a, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@5170bcf4, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@2812b107, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@df6620a]
02:51:55.772 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
02:51:55.774 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
02:51:55.778 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
02:51:55.779 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
02:51:55.779 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
02:51:55.780 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
02:51:55.786 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener - Before test class: context [DefaultTestContext@6ee4d9ab testClass = HelloSourceApplicationTests, testInstance = [null], testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@5a5338df testClass = HelloSourceApplicationTests, locations = '{}', classes = '{class com.example.HelloSourceApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@55740540, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@1018bde2, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@3c947bc5, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@e1de817], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]], class annotated with @DirtiesContext [false] with mode [null].
02:51:55.787 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSourceApplicationTests]
02:51:55.788 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSourceApplicationTests]
02:51:55.798 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener - Performing dependency injection for test context [[DefaultTestContext@6ee4d9ab testClass = HelloSourceApplicationTests, testInstance = com.example.HelloSourceApplicationTests@7d286fb6, testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@5a5338df testClass = HelloSourceApplicationTests, locations = '{}', classes = '{class com.example.HelloSourceApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@55740540, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@1018bde2, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@3c947bc5, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@e1de817], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]]].
02:51:55.823 [main] DEBUG org.springframework.test.context.support.TestPropertySourceUtils - Adding inlined properties to environment: {spring.jmx.enabled=false, org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true, server.port=-1}

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.1.2.RELEASE)

2019-01-28 02:51:56.088  INFO 28652 --- [           main] com.example.HelloSourceApplicationTests  : Starting HelloSourceApplicationTests on makinoMacBook-puro.local with PID 28652 (started by maki in /Users/maki/git/scst/hello-source)
2019-01-28 02:51:56.090  INFO 28652 --- [           main] com.example.HelloSourceApplicationTests  : No active profile set, falling back to default profiles: default
2019-01-28 02:51:57.225  INFO 28652 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2019-01-28 02:51:57.230  INFO 28652 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'taskScheduler' has been explicitly defined. Therefore, a default ThreadPoolTaskScheduler will be created.
2019-01-28 02:51:57.236  INFO 28652 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2019-01-28 02:51:57.339  INFO 28652 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.amqp.rabbit.annotation.RabbitBootstrapConfiguration' of type [org.springframework.amqp.rabbit.annotation.RabbitBootstrapConfiguration$$EnhancerBySpringCGLIB$$6f4f0989] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2019-01-28 02:51:57.447  INFO 28652 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationDisposableAutoCreatedBeans' of type [org.springframework.integration.config.annotation.Disposables] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2019-01-28 02:51:57.489  INFO 28652 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.integration.config.IntegrationManagementConfiguration' of type [org.springframework.integration.config.IntegrationManagementConfiguration$$EnhancerBySpringCGLIB$$13d5068a] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2019-01-28 02:51:59.254  INFO 28652 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService 'taskScheduler'
2019-01-28 02:51:59.745  INFO 28652 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2019-01-28 02:51:59.745  INFO 28652 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 1 subscriber(s).
2019-01-28 02:51:59.745  INFO 28652 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started _org.springframework.integration.errorLogger
2019-01-28 02:51:59.764  INFO 28652 --- [           main] o.s.c.s.m.DirectWithAttributesChannel    : Channel 'application.output' has 1 subscriber(s).
2019-01-28 02:51:59.795  INFO 28652 --- [           main] com.example.HelloSourceApplicationTests  : Started HelloSourceApplicationTests in 3.969 seconds (JVM running for 4.775)
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 4.862 s - in com.example.HelloSourceApplicationTests
2019-01-28 02:52:00.250  INFO 28652 --- [       Thread-3] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2019-01-28 02:52:00.250  INFO 28652 --- [       Thread-3] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 0 subscriber(s).
2019-01-28 02:52:00.251  INFO 28652 --- [       Thread-3] o.s.i.endpoint.EventDrivenConsumer       : stopped _org.springframework.integration.errorLogger
2019-01-28 02:52:00.253  INFO 28652 --- [       Thread-3] o.s.s.c.ThreadPoolTaskScheduler          : Shutting down ExecutorService 'taskScheduler'
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 8.471 s
[INFO] Finished at: 2019-01-28T02:52:00+09:00
[INFO] ------------------------------------------------------------------------
```
