## Sinkの作成

次にSinkを作成します。

![image](https://qiita-image-store.s3.amazonaws.com/0/1852/07c43f53-22ab-6ae1-26a4-948809767831.png)

### プロジェクト作成

以下のコマンドを実行すると、`hello-sink`フォルダに雛形プロジェクトが生成されます

```
curl start.spring.io/starter.tgz \
       -d artifactId=hello-sink \
       -d baseDir=hello-sink \
       -d dependencies=web,actuator,cloud-stream-binder-rabbit \
       -d applicationName=HelloSinkApplication | tar -xzvf -
```

> **【ノート】 Web UIからのプロジェクト作成**
>
> `curl`によるプロジェクト作成がうまくいかない場合は[Spring Initializr](https://start.spring.io)にアクセスして、
> 次の項目を入力し、"Generate Project"ボタンをクリックしてください。`hello-sink.zip`がダウンロードされるので、これを展開してください。
> 
> * Artifact: hello-sink
> * Search for dependecies: Web, Actuator, Stream Rabbit
> 
> ![image](https://qiita-image-store.s3.amazonaws.com/0/1852/1ebaf1ab-824b-1637-ebee-94c24c596aee.png)



Spring Cloud Streamが作成する`MessageChannel`から`Message`を受信するSinkクラスを作成します。

`@EnableBinding(Sink.class)`を指定することにより、`Sink`に対応する`MessageChannel`がバインドされ、`Sink`インスタンスがインジェクション可能になります。今回は`Sink`インスタンスは使用しません。その代わりにメッセージを処理するメソッドに`@StreamListener`アノテーションを付与し、channel名を指定します。ここではchennel名に`Sink.INPUT`(`input`)を使用します。

`src/main/java/com/example/HelloSinkApplication.java`を次のように記述してください。

``` java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;

@SpringBootApplication
@EnableBinding(Sink.class)
public class HelloSinkApplication {

	public static void main(String[] args) {
		SpringApplication.run(HelloSinkApplication.class, args);
	}

	@StreamListener(Sink.INPUT)
	public void print(Tweet tweet) {
		System.out.println("Received " + tweet.text);
	}

	public static class Tweet {
		public String text;
	}
}
```

channel名`input`に対するdestination名とConsumerGroup名を`application.properties`に次にように設定してください。

``` properties
spring.cloud.stream.bindings.input.destination=hello
spring.cloud.stream.bindings.input.group=printer
```



> **【ノート】 デフォルトchannel名の`input`**
> 
> ここでchanel名は`input`になっていますが、この値はSpring Cloud Streamが用意している`Sink`クラスに定義されています。
> 
> ``` java
> package org.springframework.cloud.stream.messaging;
> 
> import org.springframework.cloud.stream.annotation.Input;
> import org.springframework.messaging.SubscribableChannel;
> 
> public interface Sink {
> 
> 	String INPUT = "input";
> 
> 	@Input(Sink.INPUT)
> 	SubscribableChannel input();
> }
> ```
>
> `input`以外のchannel名を使う場合はこの`Sink`クラスを同じようなクラスを作成して、`@EnableBinding`に指定すれば良いです。


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
total 38408
drwxr-xr-x  4 makit  720748206   136B Dec 11 21:22 classes
drwxr-xr-x  3 makit  720748206   102B Dec 11 21:22 generated-sources
drwxr-xr-x  3 makit  720748206   102B Dec 11 21:22 generated-test-sources
-rw-r--r--  1 makit  720748206    19M Dec 11 21:22 hello-sink-0.0.1-SNAPSHOT.jar
-rw-r--r--  1 makit  720748206   3.7K Dec 11 21:22 hello-sink-0.0.1-SNAPSHOT.jar.original
drwxr-xr-x  3 makit  720748206   102B Dec 11 21:22 maven-archiver
drwxr-xr-x  3 makit  720748206   102B Dec 11 21:22 maven-status
drwxr-xr-x  3 makit  720748206   102B Dec 11 21:22 test-classes
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

`hello.printer` Queueをクリックして、Overviewを開いてください。この状態で再度リクエストを送ると、MessageがこのQueueに送られていることが見て取れます。またBindingsを見ると`hello`ExchangにこのQueueがバインドされていることもわかります。

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

### Cloud Foundryにデプロイ

次にここで作成したアプリケーションをCloud Foundryにデプロイします。

まずは`hello-sink`ディレクトリ直下に次の`manifest.yml`を作成してください。

``` yml
applications:
- name: hello-sink-tmaki
  memory: 512m
  buildpack: java_buildpack
  path: target/hello-sink-0.0.1-SNAPSHOT.jar
  services:
  - rabbitmq-binder
```

`tmaki`の部分は一意になるように自分のアカウント名などに置換してください。また、商用版Pivotal Cloud Foundryを使用する場合は`java_buildpack`の代わりに`java_buildpack_offline`を使用してください。`rabbitmq-binder`サービスインスタンスの作成方法は[Sourceの作成#Cloud Foundryにデプロイ](source.md#cloud-foundryにデプロイ)を参照してください。

```
$ cf push
Using manifest file /Users/makit/scst-ws/hello-sink/manifest.yml

Creating app hello-sink-tmaki in org *** / space production as ***...
OK

Creating route hello-sink-tmaki.cfapps.io...
OK

Binding hello-sink-tmaki.cfapps.io to hello-sink-tmaki...
OK

Uploading hello-sink-tmaki...
Uploading app files from: /var/folders/15/fww24j3d7pg9sz196cxv_6xm4nvlh8/T/unzipped-app424654486
Uploading 517.7K, 100 files
Done uploading               
OK
Binding service rabbitmq-binder to app hello-sink-tmaki in org *** / space production as ***...
OK

Starting app hello-sink-tmaki in org *** / space production as ***...
Downloading java_buildpack...
Downloaded java_buildpack
Creating container
Successfully created container
Downloading app package...
Downloaded app package (16.6M)
Staging...
-----> Java Buildpack Version: v3.10 (offline) | https://github.com/cloudfoundry/java-buildpack.git#193d6b7
-----> Downloading Open Jdk JRE 1.8.0_111 from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_111.tar.gz (found in cache)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.2s)
-----> Downloading Open JDK Like Memory Calculator 2.0.2_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/trusty/x86_64/memory-calculator-2.0.2_RELEASE.tar.gz (found in cache)
       Memory Settings: -Xmx681574K -XX:MaxMetaspaceSize=104857K -Xss349K -Xms681574K -XX:MetaspaceSize=104857K
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

App hello-sink-tmaki was started using this command `CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-2.0.2_RELEASE -memorySizes=metaspace:64m..,stack:228k.. -memoryWeights=heap:65,metaspace:10,native:15,stack:10 -memoryInitials=heap:100%,metaspace:100% -stackThreads=300 -totMemory=$MEMORY_LIMIT) && JAVA_OPTS="-Djava.io.tmpdir=$TMPDIR -XX:OnOutOfMemoryError=$PWD/.java-buildpack/open_jdk_jre/bin/killjava.sh $CALCULATED_MEMORY" && SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher`

Showing health and status for app hello-sink-tmaki in org *** / space production as ***...
OK

requested state: started
instances: 1/1
usage: 512M x 1 instances
urls: hello-sink-tmaki.cfapps.io
last uploaded: Sun Dec 11 14:17:45 UTC 2016
stack: cflinuxfs2
buildpack: java_buildpack

     state     since                    cpu      memory           disk           details
#0   running   2016-12-11 11:18:46 PM   170.0%   299.8M of 512M   141.7M of 1G
```


次のようにSource側にメッセージを送信してください。

```
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello"}' -H 'Content-Type: application/json'
```

`cf logs`コマンドでSink側のログを確認して、次のメッセージが出力されていることを確認してください。

``` console
$ cf logs hello-sink-tmaki --recent

...

2016-12-11T23:23:08.94+0900 [APP/PROC/WEB/0]OUT Received Hello
```

### Unit Testの作成

Spring Cloud Streamにはテスト支援機能も用意されています。テスト時はMessage Binderのインメモリ実装が使われるようになります。これにより、開発中はMessage Binder(RabbitMQ)を用意しなくてもテストを進めることができます。

`pom.xml`に次の依存関係を追加して下さい。

``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-support</artifactId>
    <scope>test</scope>
</dependency>
```


`src/test/java/com/example/HelloSinkApplicationTests.java`に次の内容を記述してください。

``` java
package com.example;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.rule.OutputCapture;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.integration.support.MessageBuilder;
import org.springframework.test.context.junit4.SpringRunner;

import com.example.HelloSinkApplication.Tweet;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
public class HelloSinkApplicationTests {
	@Autowired
	Sink sink;
	@Rule
	public OutputCapture capture = new OutputCapture();

	@Test
	public void testPrint() {
		Tweet tweet = new Tweet();
		tweet.text = "Hello";
		sink.input().send(MessageBuilder.withPayload(tweet).build());

		assertThat(capture.toString())
				.isEqualTo("Received Hello" + System.lineSeparator());
	}

}
```

次のようにテストが通ればOKです。


``` console
$ ./mvnw test
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building demo 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ hello-sink ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] Copying 1 resource
[INFO] Copying 0 resource
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ hello-sink ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ hello-sink ---
[INFO] Using 'UTF-8' encoding to copy filtered resources.
[INFO] skip non existing resourceDirectory /Users/makit/scst-ws/hello-sink/src/test/resources
[INFO] 
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ hello-sink ---
[INFO] Nothing to compile - all classes are up to date
[INFO] 
[INFO] --- maven-surefire-plugin:2.18.1:test (default-test) @ hello-sink ---
[INFO] Surefire report directory: /Users/makit/scst-ws/hello-sink/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
23:35:16.304 [main] DEBUG org.springframework.test.context.junit4.SpringJUnit4ClassRunner - SpringJUnit4ClassRunner constructor called with [class com.example.HelloSinkApplicationTests]
23:35:16.310 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
23:35:16.316 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
23:35:16.330 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSinkApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
23:35:16.340 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSinkApplicationTests], using SpringBootContextLoader
23:35:16.343 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSinkApplicationTests]: class path resource [com/example/HelloSinkApplicationTests-context.xml] does not exist
23:35:16.343 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSinkApplicationTests]: class path resource [com/example/HelloSinkApplicationTestsContext.groovy] does not exist
23:35:16.343 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSinkApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
23:35:16.344 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSinkApplicationTests]: HelloSinkApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
23:35:16.381 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSinkApplicationTests]
23:35:16.438 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
23:35:16.439 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
23:35:16.439 [main] DEBUG org.springframework.core.env.StandardEnvironment - Initialized StandardEnvironment with PropertySources [systemProperties,systemEnvironment]
23:35:16.455 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Resolved classpath location [com/example/] to resources [URL [file:/Users/makit/scst-ws/hello-sink/target/test-classes/com/example/], URL [file:/Users/makit/scst-ws/hello-sink/target/classes/com/example/]]
23:35:16.456 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Looking for matching resources in directory tree [/Users/makit/scst-ws/hello-sink/target/test-classes/com/example]
23:35:16.456 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Searching directory [/Users/makit/scst-ws/hello-sink/target/test-classes/com/example] for files matching pattern [/Users/makit/scst-ws/hello-sink/target/test-classes/com/example/*.class]
23:35:16.459 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Looking for matching resources in directory tree [/Users/makit/scst-ws/hello-sink/target/classes/com/example]
23:35:16.459 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Searching directory [/Users/makit/scst-ws/hello-sink/target/classes/com/example] for files matching pattern [/Users/makit/scst-ws/hello-sink/target/classes/com/example/*.class]
23:35:16.460 [main] DEBUG org.springframework.core.io.support.PathMatchingResourcePatternResolver - Resolved location pattern [classpath*:com/example/*.class] to resources [file [/Users/makit/scst-ws/hello-sink/target/test-classes/com/example/HelloSinkApplicationTests.class], file [/Users/makit/scst-ws/hello-sink/target/classes/com/example/HelloSinkApplication$Tweet.class], file [/Users/makit/scst-ws/hello-sink/target/classes/com/example/HelloSinkApplication.class]]
23:35:16.524 [main] DEBUG org.springframework.context.annotation.ClassPathScanningCandidateComponentProvider - Identified candidate component class: file [/Users/makit/scst-ws/hello-sink/target/classes/com/example/HelloSinkApplication.class]
23:35:16.525 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSinkApplication for test class com.example.HelloSinkApplicationTests
23:35:16.527 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSinkApplicationTests]: using defaults.
23:35:16.531 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener]
23:35:16.550 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@fdefd3f, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@d83da2e, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@a4102b8, org.springframework.test.context.support.DirtiesContextTestExecutionListener@11dc3715, org.springframework.test.context.transaction.TransactionalTestExecutionListener@69930714, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@7a52f2a2, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@78047b92, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@8909f18, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@79ca92b9, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@1460a8c0, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@4f638935, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@4387b79e]
23:35:16.554 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSinkApplicationTests]
23:35:16.554 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSinkApplicationTests]
Running com.example.HelloSinkApplicationTests
23:35:16.557 [main] DEBUG org.springframework.test.context.junit4.SpringJUnit4ClassRunner - SpringJUnit4ClassRunner constructor called with [class com.example.HelloSinkApplicationTests]
23:35:16.558 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating CacheAwareContextLoaderDelegate from class [org.springframework.test.context.cache.DefaultCacheAwareContextLoaderDelegate]
23:35:16.558 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating BootstrapContext using constructor [public org.springframework.test.context.support.DefaultBootstrapContext(java.lang.Class,org.springframework.test.context.CacheAwareContextLoaderDelegate)]
23:35:16.559 [main] DEBUG org.springframework.test.context.BootstrapUtils - Instantiating TestContextBootstrapper for test class [com.example.HelloSinkApplicationTests] from class [org.springframework.boot.test.context.SpringBootTestContextBootstrapper]
23:35:16.560 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Neither @ContextConfiguration nor @ContextHierarchy found for test class [com.example.HelloSinkApplicationTests], using SpringBootContextLoader
23:35:16.561 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSinkApplicationTests]: class path resource [com/example/HelloSinkApplicationTests-context.xml] does not exist
23:35:16.561 [main] DEBUG org.springframework.test.context.support.AbstractContextLoader - Did not detect default resource location for test class [com.example.HelloSinkApplicationTests]: class path resource [com/example/HelloSinkApplicationTestsContext.groovy] does not exist
23:35:16.561 [main] INFO org.springframework.test.context.support.AbstractContextLoader - Could not detect default resource locations for test class [com.example.HelloSinkApplicationTests]: no resource found for suffixes {-context.xml, Context.groovy}.
23:35:16.562 [main] INFO org.springframework.test.context.support.AnnotationConfigContextLoaderUtils - Could not detect default configuration classes for test class [com.example.HelloSinkApplicationTests]: HelloSinkApplicationTests does not declare any static, non-private, non-final, nested classes annotated with @Configuration.
23:35:16.570 [main] DEBUG org.springframework.test.context.support.ActiveProfilesUtils - Could not find an 'annotation declaring class' for annotation type [org.springframework.test.context.ActiveProfiles] and class [com.example.HelloSinkApplicationTests]
23:35:16.571 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
23:35:16.571 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
23:35:16.572 [main] DEBUG org.springframework.core.env.StandardEnvironment - Initialized StandardEnvironment with PropertySources [systemProperties,systemEnvironment]
23:35:16.572 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Found @SpringBootConfiguration com.example.HelloSinkApplication for test class com.example.HelloSinkApplicationTests
23:35:16.574 [main] DEBUG org.springframework.boot.test.context.SpringBootTestContextBootstrapper - @TestExecutionListeners is not present for class [com.example.HelloSinkApplicationTests]: using defaults.
23:35:16.577 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Loaded default TestExecutionListener class names from location [META-INF/spring.factories]: [org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener, org.springframework.test.context.web.ServletTestExecutionListener, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener, org.springframework.test.context.support.DependencyInjectionTestExecutionListener, org.springframework.test.context.support.DirtiesContextTestExecutionListener, org.springframework.test.context.transaction.TransactionalTestExecutionListener, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener]
23:35:16.581 [main] INFO org.springframework.boot.test.context.SpringBootTestContextBootstrapper - Using TestExecutionListeners: [org.springframework.test.context.web.ServletTestExecutionListener@5e316c74, org.springframework.test.context.support.DirtiesContextBeforeModesTestExecutionListener@6d2a209c, org.springframework.boot.test.autoconfigure.SpringBootDependencyInjectionTestExecutionListener@75329a49, org.springframework.test.context.support.DirtiesContextTestExecutionListener@161479c6, org.springframework.test.context.transaction.TransactionalTestExecutionListener@4313f5bc, org.springframework.test.context.jdbc.SqlScriptsTestExecutionListener@7f010382, org.springframework.boot.test.mock.mockito.MockitoTestExecutionListener@1e802ef9, org.springframework.boot.test.autoconfigure.web.client.MockRestServiceServerResetTestExecutionListener@2b6faea6, org.springframework.boot.test.mock.mockito.ResetMocksTestExecutionListener@778d1062, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverTestExecutionListener@670002, org.springframework.boot.test.autoconfigure.restdocs.RestDocsTestExecutionListener@1f0f1111, org.springframework.boot.test.autoconfigure.web.servlet.MockMvcPrintOnlyOnFailureTestExecutionListener@49c386c8]
23:35:16.582 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSinkApplicationTests]
23:35:16.582 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSinkApplicationTests]
23:35:16.583 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSinkApplicationTests]
23:35:16.583 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSinkApplicationTests]
23:35:16.590 [main] DEBUG org.springframework.test.context.support.AbstractDirtiesContextTestExecutionListener - Before test class: context [DefaultTestContext@5c909414 testClass = HelloSinkApplicationTests, testInstance = [null], testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@4b14c583 testClass = HelloSinkApplicationTests, locations = '{}', classes = '{class com.example.HelloSinkApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.context.SpringBootTestContextCustomizer@551aa95a, org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@6ebc05a6, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@23faf8f2], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]]], class annotated with @DirtiesContext [false] with mode [null].
23:35:16.591 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved @ProfileValueSourceConfiguration [null] for test class [com.example.HelloSinkApplicationTests]
23:35:16.591 [main] DEBUG org.springframework.test.annotation.ProfileValueUtils - Retrieved ProfileValueSource type [class org.springframework.test.annotation.SystemProfileValueSource] for class [com.example.HelloSinkApplicationTests]
23:35:16.594 [main] DEBUG org.springframework.test.context.support.DependencyInjectionTestExecutionListener - Performing dependency injection for test context [[DefaultTestContext@5c909414 testClass = HelloSinkApplicationTests, testInstance = com.example.HelloSinkApplicationTests@1184ab05, testMethod = [null], testException = [null], mergedContextConfiguration = [MergedContextConfiguration@4b14c583 testClass = HelloSinkApplicationTests, locations = '{}', classes = '{class com.example.HelloSinkApplication}', contextInitializerClasses = '[]', activeProfiles = '{}', propertySourceLocations = '{}', propertySourceProperties = '{org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true}', contextCustomizers = set[org.springframework.boot.test.context.SpringBootTestContextCustomizer@551aa95a, org.springframework.boot.test.context.filter.ExcludeFilterContextCustomizer@6ebc05a6, org.springframework.boot.test.mock.mockito.MockitoContextCustomizer@0, org.springframework.boot.test.autoconfigure.properties.PropertyMappingContextCustomizer@0, org.springframework.boot.test.autoconfigure.web.servlet.WebDriverContextCustomizerFactory$Customizer@23faf8f2], contextLoader = 'org.springframework.boot.test.context.SpringBootContextLoader', parent = [null]]]].
23:35:16.626 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemProperties] PropertySource with lowest search precedence
23:35:16.626 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [systemEnvironment] PropertySource with lowest search precedence
23:35:16.626 [main] DEBUG org.springframework.core.env.StandardEnvironment - Initialized StandardEnvironment with PropertySources [systemProperties,systemEnvironment]
23:35:16.627 [main] DEBUG org.springframework.test.context.support.TestPropertySourceUtils - Adding inlined properties to environment: {spring.jmx.enabled=false, org.springframework.boot.test.context.SpringBootTestContextBootstrapper=true, server.port=-1}
23:35:16.627 [main] DEBUG org.springframework.core.env.StandardEnvironment - Adding [Inlined Test Properties] PropertySource with highest search precedence

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v1.4.2.RELEASE)

2016-12-11 23:35:17.411  INFO 87906 --- [           main] com.example.HelloSinkApplicationTests    : Starting HelloSinkApplicationTests on jpxxmakitm1.corp.emc.com with PID 87906 (started by makit in /Users/makit/scst-ws/hello-sink)
2016-12-11 23:35:17.411  INFO 87906 --- [           main] com.example.HelloSinkApplicationTests    : No active profile set, falling back to default profiles: default
2016-12-11 23:35:17.434  INFO 87906 --- [           main] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@36804139: startup date [Sun Dec 11 23:35:17 JST 2016]; root of context hierarchy
2016-12-11 23:35:18.132  INFO 87906 --- [           main] o.s.b.f.config.PropertiesFactoryBean     : Loading properties file from URL [jar:file:/Users/makit/.m2/repository/org/springframework/integration/spring-integration-core/4.3.5.RELEASE/spring-integration-core-4.3.5.RELEASE.jar!/META-INF/spring.integration.default.properties]
2016-12-11 23:35:18.137  INFO 87906 --- [           main] o.s.i.config.IntegrationRegistrar        : No bean named 'integrationHeaderChannelRegistry' has been explicitly defined. Therefore, a default DefaultHeaderChannelRegistry will be created.
2016-12-11 23:35:18.166  INFO 87906 --- [           main] o.s.b.f.s.DefaultListableBeanFactory     : Overriding bean definition for bean 'binderFactory' with a different definition: replacing [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=org.springframework.cloud.stream.config.BinderFactoryConfiguration; factoryMethodName=binderFactory; initMethodName=null; destroyMethodName=(inferred); defined in org.springframework.cloud.stream.config.BinderFactoryConfiguration] with [Root bean: class [null]; scope=; abstract=false; lazyInit=false; autowireMode=3; dependencyCheck=0; autowireCandidate=true; primary=false; factoryBeanName=org.springframework.cloud.stream.test.binder.TestSupportBinderAutoConfiguration; factoryMethodName=binderFactory; initMethodName=null; destroyMethodName=(inferred); defined in class path resource [org/springframework/cloud/stream/test/binder/TestSupportBinderAutoConfiguration.class]]
2016-12-11 23:35:18.567  INFO 87906 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'errorChannel' has been explicitly defined. Therefore, a default PublishSubscribeChannel will be created.
2016-12-11 23:35:18.570  INFO 87906 --- [           main] faultConfiguringBeanFactoryPostProcessor : No bean named 'taskScheduler' has been explicitly defined. Therefore, a default ThreadPoolTaskScheduler will be created.
2016-12-11 23:35:18.619  INFO 87906 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.amqp.rabbit.annotation.RabbitBootstrapConfiguration' of type [class org.springframework.amqp.rabbit.annotation.RabbitBootstrapConfiguration$$EnhancerBySpringCGLIB$$e5a09acd] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 23:35:18.671  INFO 87906 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'org.springframework.cloud.stream.config.ChannelBindingServiceConfiguration$PostProcessorConfiguration' of type [class org.springframework.cloud.stream.config.ChannelBindingServiceConfiguration$PostProcessorConfiguration$$EnhancerBySpringCGLIB$$16ff752b] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 23:35:18.812  INFO 87906 --- [           main] o.s.b.f.config.PropertiesFactoryBean     : Loading properties file from URL [jar:file:/Users/makit/.m2/repository/org/springframework/integration/spring-integration-core/4.3.5.RELEASE/spring-integration-core-4.3.5.RELEASE.jar!/META-INF/spring.integration.default.properties]
2016-12-11 23:35:18.813  INFO 87906 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationGlobalProperties' of type [class org.springframework.beans.factory.config.PropertiesFactoryBean] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 23:35:18.814  INFO 87906 --- [           main] trationDelegate$BeanPostProcessorChecker : Bean 'integrationGlobalProperties' of type [class java.util.Properties] is not eligible for getting processed by all BeanPostProcessors (for example: not eligible for auto-proxying)
2016-12-11 23:35:19.376  INFO 87906 --- [           main] o.s.integration.channel.DirectChannel    : Channel 'application:-1.input' has 1 subscriber(s).
2016-12-11 23:35:19.491  INFO 87906 --- [           main] o.s.s.c.ThreadPoolTaskScheduler          : Initializing ExecutorService  'taskScheduler'
2016-12-11 23:35:20.279  INFO 87906 --- [           main] o.s.i.codec.kryo.CompositeKryoRegistrar  : configured Kryo registration [40, java.io.File] with serializer org.springframework.integration.codec.kryo.FileSerializer
2016-12-11 23:35:20.406  INFO 87906 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase -2147482648
2016-12-11 23:35:20.408  INFO 87906 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 0
2016-12-11 23:35:20.409  INFO 87906 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : Adding {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2016-12-11 23:35:20.409  INFO 87906 --- [           main] o.s.i.channel.PublishSubscribeChannel    : Channel 'application:-1.errorChannel' has 1 subscriber(s).
2016-12-11 23:35:20.409  INFO 87906 --- [           main] o.s.i.endpoint.EventDrivenConsumer       : started _org.springframework.integration.errorLogger
2016-12-11 23:35:20.409  INFO 87906 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 2147482647
2016-12-11 23:35:20.425  INFO 87906 --- [           main] o.s.c.support.DefaultLifecycleProcessor  : Starting beans in phase 2147483647
2016-12-11 23:35:20.445  INFO 87906 --- [           main] com.example.HelloSinkApplicationTests    : Started HelloSinkApplicationTests in 3.815 seconds (JVM running for 4.445)
Received Hello
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.95 sec - in com.example.HelloSinkApplicationTests
2016-12-11 23:35:20.525  INFO 87906 --- [       Thread-1] s.c.a.AnnotationConfigApplicationContext : Closing org.springframework.context.annotation.AnnotationConfigApplicationContext@36804139: startup date [Sun Dec 11 23:35:17 JST 2016]; root of context hierarchy
2016-12-11 23:35:20.529  INFO 87906 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 2147483647
2016-12-11 23:35:20.529  INFO 87906 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 2147482647
2016-12-11 23:35:20.530  INFO 87906 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase 0
2016-12-11 23:35:20.531  INFO 87906 --- [       Thread-1] o.s.i.endpoint.EventDrivenConsumer       : Removing {logging-channel-adapter:_org.springframework.integration.errorLogger} as a subscriber to the 'errorChannel' channel
2016-12-11 23:35:20.532  INFO 87906 --- [       Thread-1] o.s.i.channel.PublishSubscribeChannel    : Channel 'application:-1.errorChannel' has 0 subscriber(s).
2016-12-11 23:35:20.532  INFO 87906 --- [       Thread-1] o.s.i.endpoint.EventDrivenConsumer       : stopped _org.springframework.integration.errorLogger
2016-12-11 23:35:20.532  INFO 87906 --- [       Thread-1] o.s.c.support.DefaultLifecycleProcessor  : Stopping beans in phase -2147482648
2016-12-11 23:35:20.535  INFO 87906 --- [       Thread-1] o.s.s.c.ThreadPoolTaskScheduler          : Shutting down ExecutorService 'taskScheduler'

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.554 s
[INFO] Finished at: 2016-12-11T23:35:20+09:00
[INFO] Final Memory: 18M/309M
[INFO] ------------------------------------------------------------------------
```
