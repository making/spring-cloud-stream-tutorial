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

	@PostMapping
	public void tweet(@RequestBody Tweet tweet) {
		this.streamBridge.send("output", tweet);
	}

	public static void main(String[] args) {
		SpringApplication.run(HelloSourceApplication.class, args);
	}

	public static class Tweet {
		public String text;
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
		final Tweet tweet = new Tweet();
		tweet.text = "hello!";
		this.app.tweet(tweet);
		final Message<byte[]> message = this.destination.receive(3, "hello");
		final Tweet output = (Tweet) this.messageConverter.fromMessage(message, Tweet.class);
		assertThat(output).isNotNull();
		assertThat(output.text).isEqualTo("hello!");
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
[INFO] --- maven-clean-plugin:3.1.0:clean (default-clean) @ hello-source ---
[INFO] Deleting /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-source/target
[INFO] 
[INFO] --- maven-resources-plugin:3.2.0:resources (default-resources) @ hello-source ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:compile (default-compile) @ hello-source ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-source/target/classes
[INFO] 
[INFO] --- maven-resources-plugin:3.2.0:testResources (default-testResources) @ hello-source ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Using 'UTF-8' encoding to copy filtered properties files.
[INFO] skip non existing resourceDirectory /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-source/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.8.1:testCompile (default-testCompile) @ hello-source ---
[INFO] Changes detected - recompiling the module!
[INFO] Compiling 1 source file to /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-source/target/test-classes
[INFO] 
[INFO] --- maven-surefire-plugin:2.22.2:test (default-test) @ hello-source ---
[INFO] 
[INFO] -------------------------------------------------------
[INFO]  T E S T S
[INFO] -------------------------------------------------------
[INFO] Running com.example.HelloSourceApplicationTests
01:55:41.380 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
01:55:41.389 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
01:55:41.423 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSourceApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
01:55:41.432 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSourceApplicationTests], using SpringBootContextLoader
01:55:41.435 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTests-context.xml] does not exist
01:55:41.436 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSourceApplicationTests]: class path resource [com/example/HelloSourceApplicationTestsContext.groovy] does not exist
01:55:41.436 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSourceApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
01:55:41.436 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSourceApplicationTests]: HelloSourceApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
01:55:41.478 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSourceApplicationTests]
01:55:41.530 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider - Identified candidate component class: file [/Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-source/target/classes/com/example/HelloSourceApplication.class]
01:55:41.531 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSourceApplication for test class com.example.HelloSourceApplicationTests
01:55:41.601 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSourceApplicationTests]: using defaults.
01:55:41.601 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.boot.test.autoconfigure.webservices.client.MockWebServiceServerTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.event.ApplicationEventsTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener, org.springframework.test.context.event.EventPublishingTestExecutionListener]
01:55:41.614 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@2d6c53fc, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@25f4878b, org.springframework.test.context.event.ApplicationEventsTestExecutionListener@4e423aa2, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@7fbdb894, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@3081f72c, org.springframework.test.context.support.DirtiesContextTestExecutionListener@3148f668, org.springframework.test.context.transaction.TransactionalTestExecutionListener@6e005dc9, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@7ceb3185, org.springframework.test.context.event.EventPublishingTestExecutionListener@436c81a3, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@3561c410, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@59e32960, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@7c214cc0, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@5b67bb7e, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@609db546, org.springframework.boot.test.autoconfigure.webservices.client.MockWebServiceServerTestExecutionListener@20f5281c]
01:55:41.618 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener - Before test class: context [DefaultTestContext@8e50104 testClass = HelloSourceApplicationTests, testInstance = [null], testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@37e4d7bb testClass = HelloSourceApplicationTests, locations = '{}', classes = '{class com.example.HelloSourceApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[[ImportsContextCustomizer@6f7923a5 key = [org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@346d61be, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@3d1cfad4, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@272ed83b, org.springframework.boot.test.autoconfigure.actuate.metrics.MetricsExportContextCustomizerFactory$DisableMetricExportContextCustomizer@5ace1ed4, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@2fd1433e, org.springframework.boot.test.context.SpringBootTestArgs@1, org.springframework.boot.test.context.SpringBootTestWebEnvironment@4bbfb90a], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map[[empty]]], class annotated with @DirtiesContext [false] with mode [null].
01:55:41.636 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener - Performing dependency injection for test context [[DefaultTestContext@8e50104 testClass = HelloSourceApplicationTests, testInstance = com.example.HelloSourceApplicationTests@39ac0c0a, testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@37e4d7bb testClass = HelloSourceApplicationTests, locations = '{}', classes = '{class com.example.HelloSourceApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[[ImportsContextCustomizer@6f7923a5 key = [org.springframework.cloud.stream.binder.test.TestChannelBinderConfiguration]], org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@346d61be, org.springframework.boot.test.json.DuplicateJsonObjectContextCustomizerFactory$DuplicateJsonObjectContextCustomizer@3d1cfad4, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.web.client.TestRestTemplateContextCustomizer@272ed83b, org.springframework.boot.test.autoconfigure.actuate.metrics.MetricsExportContextCustomizerFactory$DisableMetricExportContextCustomizer@5ace1ed4, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@2fd1433e, org.springframework.boot.test.context.SpringBootTestArgs@1, org.springframework.boot.test.context.SpringBootTestWebEnvironment@4bbfb90a], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]], attributes = map['org.springframework.test.context.event.ApplicationEventsTestExecutionListener.recordApplicationEvents' -> false]]].
01:55:41.654 [main] DEBUG org.springframework.test.context.support.TestPropertySourceUtils - Adding inlined properties to environment: {spring.jmx.enabled=false, org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.5.6)

2021-11-02 01:55:41.864  INFO 36569 --- [           main] com.example.HelloSourceApplicationTests  : Starting HelloSourceApplicationTests using Java 17.0.1 on makinoMacBook-Pro.local with PID 36569 (started by toshiaki in /Users/toshiaki/git/spring-cloud-stream-tutorial/tmp/hello-source)
2021-11-02 01:55:41.865  INFO 36569 --- [           main] com.example.HelloSourceApplicationTests  : No active profile set, falling back to default profiles: default
2021-11-02 01:55:42.553  INFO 36569 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2021-11-02 01:55:42.561  INFO 36569 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2021-11-02 01:55:42.639  INFO 36569 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationChannelResolver' of type [org.springframework.integration.support.channel.BeanFactoryChannelResolver] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 01:55:42.644  INFO 36569 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationDisposableAutoCreatedBeans' of type [org.springframework.integration.config.annotation.Disposables] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 01:55:42.664  INFO 36569 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration' of type [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 01:55:42.670  INFO 36569 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'bindersHealthContributor' of type [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration$BindersHealthContributor] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 01:55:42.672  INFO 36569 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'bindersHealthIndicatorListener' of type [org.springframework.cloud.stream.config.BindersHealthIndicatorAutoConfiguration$BindersHealthIndicatorListener] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 01:55:42.684  INFO 36569 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.integration.config.IntegrationManagementConfiguration' of type [org.springframework.integration.config.IntegrationManagementConfiguration] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2021-11-02 01:55:43.390  INFO 36569 --- [           main] c.f.c.c.BeanFactoryAwareFunctionRegistry : Can't determine default function definition. Please use 'spring.cloud.function.definition' property to explicitly define it.
2021-11-02 01:55:43.549  INFO 36569 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2021-11-02 01:55:43.550  INFO 36569 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 1 subscriber(s).
2021-11-02 01:55:43.550  INFO 36569 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started bean '_org.springframework.integration.errorLogger'
2021-11-02 01:55:43.563  INFO 36569 --- [           main] com.example.HelloSourceApplicationTests  : Started HelloSourceApplicationTests in 1.906 seconds (JVM running for 2.675)
2021-11-02 01:55:43.872  INFO 36569 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'hello.destination' has 1 subscriber(s).
2021-11-02 01:55:43.874  INFO 36569 --- [           main] o.s.c.s.m.DirectWithAttributesChannel    : Channel 'unknown.channel.name' has 1 subscriber(s).
{id=d9bbf7bb-55b9-ea74-d77d-9d6c60009d04, contentType=application/json, target-protocol=streamBridge, timestamp=1635785743906}
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.641 s - in com.example.HelloSourceApplicationTests
2021-11-02 01:55:43.968  INFO 36569 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2021-11-02 01:55:43.968  INFO 36569 --- [ionShutdownHook] o.s.i.channel.PublishSubscribeChannel    : Channel 'application.errorChannel' has 0 subscriber(s).
2021-11-02 01:55:43.969  INFO 36569 --- [ionShutdownHook] o.s.i.endpoint.EventDrivenConsumer       : stopped bean '_org.springframework.integration.errorLogger'
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  5.944 s
[INFO] Finished at: 2021-11-02T01:55:44+09:00
[INFO] ------------------------------------------------------------------------
```
