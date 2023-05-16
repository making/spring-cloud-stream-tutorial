## Sinkの作成

次にSinkを作成します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/07c43f53-22ab-6ae1-26a4-948809767831.png)

### プロジェクト作成

以下のコマンドを実行すると、`hello-sink`フォルダに雛形プロジェクトが生成されます

```
curl https://start.spring.io/starter.tgz \
       -d artifactId=hello-sink \
       -d baseDir=hello-sink \
       -d packageName=com.example \
       -d dependencies=web,actuator,cloud-stream,amqp \
       -d type=maven-project \
       -d applicationName=HelloSinkApplication | tar -xzvf -
```

> **【ノート】 Web UIからのプロジェクト作成**
>
> `curl`によるプロジェクト作成がうまくいかない場合は[Spring Initializr](https://start.spring.io)にアクセスして、
> 次の項目を入力し、"Generate Project"ボタンをクリックしてください。`hello-sink.zip`がダウンロードされるので、これを展開してください。
> 
> * Artifact: hello-sink
> * Search for dependecies: Web, Actuator, Cloud Stream, RabbitMQ
> 
> ![image](https://qiita-image-store.s3.amazonaws.com/0/1852/1ebaf1ab-824b-1637-ebee-94c24c596aee.png)



Spring Cloud Streamが作成する`MessageChannel`から`Message`を受信するSinkクラスを作成します。

`@EnableBinding(Sink.class)`を指定することにより、`Sink`に対応する`MessageChannel`がバインドされ、`Sink`インスタンスがインジェクション可能になります。今回は`Sink`インスタンスは使用しません。その代わりにメッセージを処理するメソッドに`@StreamListener`アノテーションを付与し、channel名を指定します。ここではchennel名に`Sink.INPUT`(`input`)を使用します。

`src/main/java/com/example/HelloSinkApplication.java`を次のように記述してください。

``` java
package com.example;

import java.util.function.Consumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;

@SpringBootApplication
public class HelloSinkApplication {

	public static void main(String[] args) {
		SpringApplication.run(HelloSinkApplication.class, args);
	}

	@Bean
	public Consumer<Tweet> tweetPrinter() {
		return tweet -> System.out.println("Received " + tweet.text());
	}

	record Tweet(String text) {
	}
}
```

channel名`input`に対するdestination名とConsumerGroup名を`application.properties`に次にように設定してください。

``` properties
spring.cloud.stream.function.bindings.tweetPrinter-in-0=input
spring.cloud.stream.bindings.input.destination=hello
spring.cloud.stream.bindings.input.group=printer
```

> **【ノート】 ConsumerGroupの設定**
> 
> `spring.cloud.stream.bindings.input.group`にConsumerGroup名を指定しました。Spring Cloud Streamでは同じConsumerGroupを持つConsumerのうち最低1つメッセージが届くことが保証されます。ConsumerGroupを指定しないと同じConsumerアプリケーションでも全てのインスタンスにメッセージが届きます。特別な理由がなければConsumerアプリケーションごとに同じConsumerGroup名を指定するするのが良いです。

次のコマンドでこのSourceアプリケーションのjarファイルを作成してください。



```
./mvnw clean package -DskipTests=true
```

`target`ディレクトリに`hello-sink-0.0.1-SNAPSHOT.jar`が


```
$ ls -lh target/
total 56840
drwxr-xr-x  4 toshiaki  staff   128B Nov  2 02:06 classes
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 02:06 generated-sources
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 02:06 generated-test-sources
-rw-r--r--  1 toshiaki  staff    28M Nov  2 02:06 hello-sink-0.0.1-SNAPSHOT.jar
-rw-r--r--  1 toshiaki  staff   3.7K Nov  2 02:06 hello-sink-0.0.1-SNAPSHOT.jar.original
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 02:06 maven-archiver
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 02:06 maven-status
drwxr-xr-x  3 toshiaki  staff    96B Nov  2 02:06 test-classes
```


### ローカル環境で実行


次のコマンドでアプリケーションを起動してください。

```
java -jar target/hello-sink-0.0.1-SNAPSHOT.jar --server.port=8082
```

RabbitMQに関する設定は[Sourceの作成#ローカル環境で実行](source.md#ローカル環境で実行)を参照して下さい。

以下のコマンドで、Source側にメッセージを送信してください。

```
curl -v localhost:8080 -d '{"text":"Hello"}' -H 'Content-Type: application/json'
```

Sink側のログを見て

```
Received Hello
```

が出力されていることを確認してください。

管理コンソール([http://localhost:15672](http://localhost:15672))にアクセスして、Queuesタブを選択してください。`hello.printer` Queueができていることが確認できます。Queue名は`Destination名.ConsumerGroup名`になります。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/dd5fb790-b348-ee1e-44aa-040cd4ff78e6.png)

`hello.printer` Queueをクリックして、Overviewを開いてください。この状態で再度リクエストを送ると、MessageがこのQueueに送られていることが見て取れます。またBindingsを見ると`hello`ExchangeにこのQueueがバインドされていることもわかります。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/17375bd3-fcad-3c2e-4f6f-1e662e288d19.png)

`hello` Exchangeを再度確認し、Bindingsを見るとこのExchangeに`hello.printer` Queueがバインドされていることがわかります。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/6c223200-ad11-ca8f-73f2-95df8eb9826d.png)

RabbitMQのExchangeとQueueが結びつくことで、SourceとSinkがつながりました。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/c3d7b31b-6a81-bd9d-db06-3abdae4ffd94.png)

Sink側のアプリケーションをCtrl+Cで止めてください。その後、Sourceに次のメッセージを送信してください。

```
curl -v localhost:8080 -d '{"text":"Hello2"}' -H 'Content-Type: application/json'
```

次のコマンドで再度Sinkアプリケーションを起動してください。

```
java -jar target/hello-sink-0.0.1-SNAPSHOT.jar --server.port=8082
```

起動するや否や、次のログが出力されることを確認できるでしょう。

```
Received Hello2
```

Sinkが一度でも接続されていれば、Sourceにメッセージを書き込んだ段階でSinkがダウンしていてもSinkが復帰したタイミングでメッセージを受信することができます。
これにより、書き込みに対するシステムの耐障害性を高くすることができます。

### Unit Testの作成

Spring Cloud Streamにはテスト支援機能も用意されています。テスト時はMessage Binderのインメモリ実装が使われるようになります。これにより、開発中はMessage Binder(RabbitMQ)を用意しなくてもテストを進めることができます。

`src/test/java/com/example/HelloSinkApplicationTests.java`に次の内容を記述してください。

``` java
package com.example;

import com.example.HelloSinkApplication.Tweet;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.context.SpringBootTest.WebEnvironment;
import org.springframework.boot.test.system.CapturedOutput;
import org.springframework.boot.test.system.OutputCaptureExtension;
import org.springframework.cloud.stream.binder.test.InputDestination;
import org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration;
import org.springframework.context.annotation.Import;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = WebEnvironment.NONE)
@Import(TestChannelBinderConfiguration.class)
@ExtendWith(OutputCaptureExtension.class)
class HelloSinkApplicationTests {
	@Autowired
	InputDestination destination;

	@Test
	void contextLoads(CapturedOutput capture) {
		final Tweet tweet = new Tweet("Hello");
		final Message<Tweet> message = MessageBuilder.withPayload(tweet).build();
		this.destination.send(message, "hello");
		assertThat(capture.toString()).contains("Received Hello" + System.lineSeparator());
	}

}
```

次のようにテストが通ればOKです。


``` console
$ ./mvnw clean test
[INFO] Scanning for projects...
[INFO] 
[INFO] -----------------------< com.example:hello-sink >-----------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- maven-clean-plugin:3.2.0:clean (default-clean) @ hello-sink ---
[INFO] Deleting /Users/toshiaki/git/hello-sink/target
[INFO] 
[INFO] --- maven-resources-plugin:3.3.1:resources (default-resources) @ hello-sink ---
[INFO] Copying 1 resource from src/main/resources to target/classes
[INFO] Copying 0 resource from src/main/resources to target/classes
[INFO] 
[INFO] --- maven-compiler-plugin:3.10.1:compile (default-compile) @ hello-sink ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/hello-sink/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.3.1:testResources (default-testResources) @ hello-sink ---
[INFO] skip non existing resourceDirectory /Users/toshiaki/git/hello-sink/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.10.1:testCompile (default-testCompile) @ hello-sink ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/hello-sink/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ hello-sink ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.HelloSinkApplicationTests
21:34:10.794 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Neither @ContextConfiguration nor @ContextHierarchy found for test class [HelloSinkApplicationTests]: using SpringBootContextLoader
21:34:10.797 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader -- Could not detect default resource locations for test class [com.example.HelloSinkApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
21:34:10.798 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils -- Could not detect default configuration classes for test class [com.example.HelloSinkApplicationTests]: HelloSinkApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
21:34:10.823 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Using ContextCustomizers for test class [HelloSinkApplicationTests]: [ImportsContextCustomizer, ExcludeFilterContextCustomizer, DuplicateJsonObjectContextCustomizer, MockitoContextCustomizer, TestRestTemplateContextCustomizer, WebTestClientContextCustomizer, DisableObservabilityContextCustomizer, PropertyMappingContextCustomizer, Customizer]
21:34:10.895 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider -- Identified candidate component class: file [/Users/toshiaki/git/hello-sink/target/classes/com/example/HelloSinkApplication.class]
21:34:10.896 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Found @SpringBootConfiguration com.example.HelloSinkApplication for test class com.example.HelloSinkApplicationTests
21:34:11.023 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper -- Using TestExecutionListeners for test class [HelloSinkApplicationTests]: [ServletTestExecutionListener, DirtiesContextBeforeModesTestExecutionListener, ApplicationEventsTestExecutionListener, MockitoTestExecutionListener, DependencyInjectionTestExecutionListener, DirtiesContextTestExecutionListener, TransactionalTestExecutionListener, SqlScriptsTestExecutionListener, EventPublishingTestExecutionListener, ResetMocksTestExecutionListener, RestDocsTestExecutionListener, MockRestServiceServerResetTestExecutionListener, MockMvcPrintOnlyOnFailureTestExecutionListener, WebDriverTestExecutionListener, MockWebServiceServerTestExecutionListener]
21:34:11.024 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener -- Before test class: class [HelloSinkApplicationTests], class annotated with @DirtiesContext [false] with mode [null]
21:34:11.040 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener -- Performing dependency injection for test class com.example.HelloSinkApplicationTests

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.0.6)

2023-05-16T21:34:11.362+09:00  INFO 6899 --- [           main] com.example.HelloSinkApplicationTests    : Starting HelloSinkApplicationTests using Java 17.0.5 with PID 6899 (started by toshiaki in /Users/toshiaki/git/hello-sink)
2023-05-16T21:34:11.365+09:00  INFO 6899 --- [           main] com.example.HelloSinkApplicationTests    : No active profile set, falling back to 1 default profile: "default"
2023-05-16T21:34:12.212+09:00  INFO 6899 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2023-05-16T21:34:12.220+09:00  INFO 6899 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2023-05-16T21:34:13.218+09:00  INFO 6899 --- [           main] o.s.c.s.m.DirectWithAttributesChannel    : Channel 'application.input' has 1 subscriber(s).
2023-05-16T21:34:13.497+09:00  INFO 6899 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2023-05-16T21:34:13.497+09:00  INFO 6899 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 1 subscriber(s).
2023-05-16T21:34:13.497+09:00  INFO 6899 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean '_org.springframework.integration.errorLogger'
2023-05-16T21:34:13.507+09:00  INFO 6899 --- [           main] o.s.c.stream.binder.BinderErrorChannel   : Channel '22249027.input.errors' has 1 subscriber(s).
2023-05-16T21:34:13.508+09:00  INFO 6899 --- [           main] o.s.c.stream.binder.BinderErrorChannel   : Channel '22249027.input.errors' has 2 subscriber(s).
2023-05-16T21:34:13.509+09:00  INFO 6899 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'hello.destination' has 1 subscriber(s).
2023-05-16T21:34:13.510+09:00  INFO 6899 --- [           main] r$IntegrationBinderInboundChannelAdapter : started org.springframework.cloud.stream.binder.test.TestChannelBinder$IntegrationBinderInboundChannelAdapter@77896335
2023-05-16T21:34:13.526+09:00  INFO 6899 --- [           main] com.example.HelloSinkApplicationTests    : Started HelloSinkApplicationTests in 2.457 seconds (process running for 3.333)
Received Hello
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.306 s - in com.example.HelloSinkApplicationTests
2023-05-16T21:34:13.997+09:00  INFO 6899 --- [ionShutdownHook] r$IntegrationBinderInboundChannelAdapter : stopped org.springframework.cloud.stream.binder.test.TestChannelBinder$IntegrationBinderInboundChannelAdapter@77896335
2023-05-16T21:34:14.002+09:00  INFO 6899 --- [ionShutdownHook] o.s.c.stream.binder.BinderErrorChannel   : Channel 'application.22249027.input.errors' has 1 subscriber(s).
2023-05-16T21:34:14.006+09:00  INFO 6899 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2023-05-16T21:34:14.007+09:00  INFO 6899 --- [ionShutdownHook] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 0 subscriber(s).
2023-05-16T21:34:14.008+09:00  INFO 6899 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : stopped bean '_org.springframework.integration.errorLogger'
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  6.723 s
[INFO] Finished at: 2023-05-16T21:34:14+09:00
[INFO] ------------------------------------------------------------------------
```
