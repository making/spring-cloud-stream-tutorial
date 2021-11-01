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
		return tweet -> System.out.println("Received " + tweet.text);
	}

	public static class Tweet {
		public String text;
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
		final Tweet tweet = new Tweet();
		tweet.text = "Hello";
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
[INFO] --- maven-clean-plugin:3.1.0:clean (default-clean) @ hello-sink ---
[INFO] Deleting /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-sink/target
[INFO] 
[INFO] --- maven-resources-plugin:3.2.0:resources (default-resources) @ hello-sink ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ hello-sink ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-sink/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.2.0:testResources (default-testResources) @ hello-sink ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] skip non existing resourceDirectory /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-sink/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:testCompile (default-testCompile) @ hello-sink ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-sink/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ hello-sink ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.HelloSinkApplicationTests
02:48:23.494 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
02:48:23.501 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
02:48:23.529 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSinkApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
02:48:23.536 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSinkApplicationTests], using SpringBootContextLoader
02:48:23.538 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSinkApplicationTests]: class path resource [com/example/HelloSinkApplicationTests-context.xml] does not exist
02:48:23.539 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSinkApplicationTests]: class path resource [com/example/HelloSinkApplicationTestsContext.groovy] does not exist
02:48:23.539 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSinkApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
02:48:23.539 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSinkApplicationTests]: HelloSinkApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
02:48:23.580 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSinkApplicationTests]
02:48:23.621 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider - Identified candidate component class: file [/Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-sink/target/classes/com/example/HelloSinkApplication.class]
02:48:23.622 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSinkApplication for test class com.example.HelloSinkApplicationTests
02:48:23.686 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSinkApplicationTests]: using defaults.
02:48:23.686 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.boot.test.autoconfigure.webservices.client.MockWebServiceServerTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.event.ApplicationEventsTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener, org.springframework.test.context.event.EventPublishingTestExecutionListener]
02:48:23.699 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@3eb77ea8, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@7b8b43c7, org.springframework.test.context.event.ApplicationEventsTestExecutionListener@7aaca91a, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@44c73c26, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@41005828, org.springframework.test.context.support.DirtiesContextTestExecutionListener@60b4beb4, org.springframework.test.context.transaction.TransactionalTestExecutionListener@7fcf2fc1, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@2141a12, org.springframework.test.context.event.EventPublishingTestExecutionListener@4196c360, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@41294f8, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@225129c, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@20435c40, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@573906eb, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@479ceda0, org.springframework.boot.test.autoconfigure.webservices.client.MockWebServiceServerTestExecutionListener@6d07a63d]
02:48:23.701 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener - Before test class: context [DefaultTestContext@59cba5a testClass = HelloSinkApplicationTests, testInstance = [null], testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@1bd39d3c testClass = HelloSinkApplicationTests, locations = '{}', classes = '{class com.example.HelloSinkApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[[ImportsContextCustomizer@6f19ac19 key = [org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@2cd2a21f, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@e7edb54, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@66498326, org.springframework.boot.test.autoconfigure.actuate.metrics.MetricsExportContextCustomizerFactory$DisableMetricExportContextCustomizer@693fe6c9, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@69997e9d, org.springframework.boot.test.context.SpringBootTestArgs@1, org.springframework.boot.test.context.SpringBootTestWebEnvironment@26aa12dd], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]], class annotated with @DirtiesContext [false] with mode [null].
02:48:23.713 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener - Performing dependency injection for test context [[DefaultTestContext@59cba5a testClass = HelloSinkApplicationTests, testInstance = com.example.HelloSinkApplicationTests@38b27cdc, testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@1bd39d3c testClass = HelloSinkApplicationTests, locations = '{}', classes = '{class com.example.HelloSinkApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[[ImportsContextCustomizer@6f19ac19 key = [org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@2cd2a21f, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@e7edb54, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@66498326, org.springframework.boot.test.autoconfigure.actuate.metrics.MetricsExportContextCustomizerFactory$DisableMetricExportContextCustomizer@693fe6c9, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@69997e9d, org.springframework.boot.test.context.SpringBootTestArgs@1, org.springframework.boot.test.context.SpringBootTestWebEnvironment@26aa12dd], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map['org.springframework.test.context.event.ApplicationEventsTestExecutionListener.recordApplicationEvents' -> false]]].
02:48:23.737 [main] DEBUG org.springframework.test.context.support.TestPropertySourceUtils - Adding inlined properties to environment: {spring.jmx.enabled=false, org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.5.6)

2021-11-02 02:48:24.105  INFO 46889 --- [           main] com.example.HelloSinkApplicationTests    : Starting HelloSinkApplicationTests using Java 17.0.1 on makinoMacBook-Pro.local with PID 46889 (started by toshiaki in /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-sink)
2021-11-02 02:48:24.106  INFO 46889 --- [           main] com.example.HelloSinkApplicationTests    : No active profile set, falling back to default profiles: default
2021-11-02 02:48:24.793  INFO 46889 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2021-11-02 02:48:24.801  INFO 46889 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2021-11-02 02:48:24.874  INFO 46889 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationChannelResolver' of type [org.springframework.integration.support.channel.BeanFactoryChannelResolver] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 02:48:24.878  INFO 46889 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationDisposableAutoCreatedBeans' of type [org.springframework.integration.config.annotation.Disposables] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 02:48:24.893  INFO 46889 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration' of type [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 02:48:24.897  INFO 46889 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'bindersHealthContributor' of type [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration$BindersHealthContributor] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 02:48:24.899  INFO 46889 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'bindersHealthIndicatorListener' of type [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration$BindersHealthIndicatorListener] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 02:48:24.909  INFO 46889 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.integration.config.IntegrationManagementConfiguration' of type [org.springframework.integration.config.IntegrationManagementConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 02:48:25.602  INFO 46889 --- [           main] o.s.c.s.m.DirectWithAttributesChannel    : Channel 'application.input' has 1 subscriber(s).
2021-11-02 02:48:25.726  INFO 46889 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2021-11-02 02:48:25.727  INFO 46889 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 1 subscriber(s).
2021-11-02 02:48:25.727  INFO 46889 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean '_org.springframework.integration.errorLogger'
2021-11-02 02:48:25.784  INFO 46889 --- [           main] o.s.c.stream.binder.BinderErrorChannel   : Channel 'hello.printer.errors' has 1 subscriber(s).
2021-11-02 02:48:25.786  INFO 46889 --- [           main] o.s.c.stream.binder.BinderErrorChannel   : Channel 'hello.printer.errors' has 2 subscriber(s).
2021-11-02 02:48:25.786  INFO 46889 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'hello.destination' has 1 subscriber(s).
2021-11-02 02:48:25.787  INFO 46889 --- [           main] r$IntegrationBinderInboundChannelAdapter : started org.springframework.cloud.stream.binder.test.TestChannelBinder$IntegrationBinderInboundChannelAdapter@5db948c9
2021-11-02 02:48:25.805  INFO 46889 --- [           main] com.example.HelloSinkApplicationTests    : Started HelloSinkApplicationTests in 2.066 seconds (JVM running for 2.776)
Received Hello
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.846 s - in com.example.HelloSinkApplicationTests
2021-11-02 02:48:26.297  INFO 46889 --- [ionShutdownHook] r$IntegrationBinderInboundChannelAdapter : stopped org.springframework.cloud.stream.binder.test.TestChannelBinder$IntegrationBinderInboundChannelAdapter@5db948c9
2021-11-02 02:48:26.310  INFO 46889 --- [ionShutdownHook] o.s.c.stream.binder.BinderErrorChannel   : Channel 'application.hello.printer.errors' has 1 subscriber(s).
2021-11-02 02:48:26.314  INFO 46889 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2021-11-02 02:48:26.314  INFO 46889 --- [ionShutdownHook] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 0 subscriber(s).
2021-11-02 02:48:26.314  INFO 46889 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : stopped bean '_org.springframework.integration.errorLogger'
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.876 s
[INFO] Finished at: 2021-11-02T02:48:26+09:00
[INFO] ------------------------------------------------------------------------
```
