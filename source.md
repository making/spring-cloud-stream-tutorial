## Sourceの作成

まずはSourceから作成します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/bcd1fc24-2966-7765-f4f4-56132dbc61b8.png)


### プロジェクト作成

以下のコマンドを実行すると、`hello-source`フォルダに雛形プロジェクトが生成されます

```
curl https://start.spring.io/starter.tgz \
       -d artifactId=hello-source \
       -d baseDir=hello-source \
       -d packageName=com.example \
       -d dependencies=web,actuator,cloud-stream,amqp \
       -d type=maven-project \
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

Spring Cloud Streamが作成する`MessageChannel`にTweetを送信するSourceクラスを作成します。ここではHTTPでJSONを受け付けて、`Tweet`インスタンスを作成し、`StreamBridge`を利用して`Tweet`を送信しています。

`src/main/java/com/example/HelloSourceApplication.java`を次のように記述してください。

``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.function.StreamBridge;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class HelloSourceApplication {

	private final StreamBridge streamBridge;

	public HelloSourceApplication(StreamBridge streamBridge) {
		this.streamBridge = streamBridge;
	}

	@PostMapping(path = "/")
	public void tweet(@RequestBody Tweet tweet) {
		this.streamBridge.send("output", tweet);
	}

	public static void main(String[] args) {
		SpringApplication.run(HelloSourceApplication.class, args);
	}

	record Tweet(String text) {
	}
}
```

binding名`output`に対するdestination名を`application.properties`に次にように設定してください。

``` properties
spring.cloud.stream.bindings.output.destination=hello
```


次のコマンドでこのSourceアプリケーションのjarファイルを作成してください。



```
./mvnw clean package -DskipTests=true
```

`target`ディレクトリに`hello-source-0.0.1-SNAPSHOT.jar`ができていることを確認してください。

```
$ ls -lh target/
total 56840
drwxr-xr-x  4 toshiaki  staff   128B Nov  2 01:36 classes
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 01:36 generated-sources
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 01:36 generated-test-sources
-rw-r--r--  1 toshiaki  staff    28M Nov  2 01:36 hello-source-0.0.1-SNAPSHOT.jar
-rw-r--r--  1 toshiaki  staff   3.4K Nov  2 01:36 hello-source-0.0.1-SNAPSHOT.jar.original
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 01:36 maven-archiver
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 01:36 maven-status
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 01:36 test-classes
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

RabbitMQが別のサーバー上にいる場合は次のように`spring.rabbitmq.host`を指定してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.host=192.168.99.100
```

ユーザー名、パスワードが必要な場合は`spring.rabbitmq.username`、`spring.rabbitmq.password`を指定してください。

```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.host=192.168.99.100 --spring.rabbitmq.username=user --spring.rabbitmq.password=pass
```

Virtual Hostを設定している場合は`spring.rabbitmq.virtual-host`を指定してください。
```
java -jar target/hello-source-0.0.1-SNAPSHOT.jar --spring.rabbitmq.host=192.168.99.100 --spring.rabbitmq.username=user --spring.rabbitmq.password=pass --spring.rabbitmq.virtual-host=/
```


> **【ノート】 `application.properties`にRabbitMQの接続情報を設定**
> 
> RabbitMQの接続情報は起動時に指定するだけでなく、`application.properties`に設定しておくこともできます。この場合は再度jarファイルをビルドしてください。
> 
> ``` properties
> spring.rabbitmq.host=192.168.99.100
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
> {"status":"UP","components":{"diskSpace":{"status":"UP","details":{"total":1000240963584,"free":174634520576,"threshold":10485760,"exists":true}},"ping":{"status":"UP"},"rabbit":{"status":"UP","details":{"version":"3.9.5"}}}}
> ```

### Unit Testの作成

Spring Cloud Streamにはテスト支援機能も用意されています。テスト時はMessage Binderのインメモリ実装が使われるようになります。これにより、開発中はMessage Binder(RabbitMQ)を用意しなくてもテストを進めることができます。


`src/test/java/com/example/HelloSourceApplicationTests.java`に次の内容を記述してください。

``` java
package com.example;

import com.example.HelloSourceApplication.Tweet;
import org.junit.jupiter.api.Test;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.cloud.stream.binder.test.OutputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Import;
import org.springframework.messaging.Message;
import org.springframework.messaging.converter.CompositeMessageConverter;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = WebEnvironment.NONE)
@Import(TestChannelBinderConfiguration.class)
class HelloSourceApplicationTests {
	@Autowired
	HelloSourceApplication app;

	@Autowired
	OutputDestination destination;

	@Autowired
	CompositeMessageConverter messageConverter;

	@Test
	void contextLoads() {
		final Tweet tweet = new Tweet("hello!");
		this.app.tweet(tweet);
		final Message<byte[]> message = this.destination.receive(3, "hello");
		final Tweet output = (Tweet) this.messageConverter.fromMessage(message, Tweet.class);
		assertThat(output).isNotNull();
		assertThat(output.text()).isEqualTo("hello!");
	}

}
```

`Message`のpayloadがJSON文字列になっていることに注意してください。


次のようにテストが通ればOKです。

``` console
$ ./mvnw clean test
[INFO] Scanning for projects...
[INFO] 
[INFO] ----------------------< com.example:hello-source >----------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:3.2.0:clean (default-clean) @ hello-source ---
[INFO] Deleting /Users/toshiaki/git/hello-source/target
[INFO] 
[INFO] --- maven-resources-plugin:3.3.1:resources (default-resources) @ hello-source ---
[INFO] Copying 1 resource from src/main/resources to target/classes
[INFO] Copying 0 resource from src/main/resources to target/classes
[INFO] 
[INFO] --- maven-compiler-plugin:3.10.1:compile (default-compile) @ hello-source ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/hello-source/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.3.1:testResources (default-testResources) @ hello-source ---
[INFO] skip non existing resourceDirectory /Users/toshiaki/git/hello-source/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.10.1:testCompile (default-testCompile) @ hello-source ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/hello-source/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ hello-source ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.HelloSourceApplicationTests
20:28:41.594 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Neither @ContextConfiguration nor @ContextHierarchy found for test class [HelloSourceApplicationTests]: using SpringBootContextLoader
20:28:41.599 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader -- Could not detect default resource locations for test class [com.example.HelloSourceApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
20:28:41.601 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [com.example.HelloSourceApplicationTests]: HelloSourceApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
20:28:41.640 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Using ContextCustomizers for test class [HelloSourceApplicationTests]: [ImportsContextCustomizer, ExcludeFilterContextCustomizer, DuplicateJsonObjectContextCustomizer, MockitoContextCustomizer, TestRestTemplateContextCustomizer, WebTestClientContextCustomizer, DisableObservabilityContextCustomizer, PropertyMappingContextCustomizer, Customizer]
20:28:41.751 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider -- Identified candidate component class: file [/Users/toshiaki/git/hello-source/target/classes/com/example/HelloSourceApplication.class]
20:28:41.753 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration com.example.HelloSourceApplication for test class com.example.HelloSourceApplicationTests
20:28:41.908 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Using TestExecutionListeners for test class [HelloSourceApplicationTests]: [ServletTestExecutionListener, DirtiesContextBeforeModesTestExecutionListener, ApplicationEventsTestExecutionListener, MockitoTestExecutionListener, DependencyInjectionTestExecutionListener, DirtiesContextTestExecutionListener, TransactionalTestExecutionListener, SqlScriptsTestExecutionListener, EventPublishingTestExecutionListener, ResetMocksTestExecutionListener, RestDocsTestExecutionListener, MockRestServiceServerResetTestExecutionListener, MockMvcPrintOnlyOnFailureTestExecutionListener, WebDriverTestExecutionListener, MockWebServiceServerTestExecutionListener]
20:28:41.910 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener -- Before test class: class [HelloSourceApplicationTests], class annotated with @DirtiesContext [false] with mode [null]
20:28:41.926 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener -- Performing dependency injection for test class com.example.HelloSourceApplicationTests

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.6)

2023-05-16T20:28:42.247+09:00  INFO 6684 --- [           main] c.example.HelloSourceApplicationTests    : Starting HelloSourceApplicationTests using Java 17.0.5 with PID 6684 (started by toshiaki in /Users/toshiaki/git/hello-source)
2023-05-16T20:28:42.250+09:00  INFO 6684 --- [           main] c.example.HelloSourceApplicationTests    : No active profile set, falling back to 1 default profile: "default"
2023-05-16T20:28:43.329+09:00  INFO 6684 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2023-05-16T20:28:43.338+09:00  INFO 6684 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2023-05-16T20:28:44.593+09:00  INFO 6684 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2023-05-16T20:28:44.594+09:00  INFO 6684 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 1 subscriber(s).
2023-05-16T20:28:44.595+09:00  INFO 6684 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean '_org.springframework.integration.errorLogger'
2023-05-16T20:28:44.626+09:00  INFO 6684 --- [           main] c.example.HelloSourceApplicationTests    : Started HelloSourceApplicationTests in 2.673 seconds (process running for 4.041)
2023-05-16T20:28:45.205+09:00  INFO 6684 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'hello.destination' has 1 subscriber(s).
2023-05-16T20:28:45.206+09:00  INFO 6684 --- [           main] o.s.c.s.m.DirectWithAttributesChannel    : Channel 'unknown.channel.name' has 1 subscriber(s).
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.97 s - in com.example.HelloSourceApplicationTests
2023-05-16T20:28:45.391+09:00  INFO 6684 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2023-05-16T20:28:45.391+09:00  INFO 6684 --- [ionShutdownHook] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 0 subscriber(s).
2023-05-16T20:28:45.392+09:00  INFO 6684 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : stopped bean '_org.springframework.integration.errorLogger'
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  8.092 s
[INFO] Finished at: 2023-05-16T20:28:45+09:00
[INFO] ------------------------------------------------------------------------
```
