## 新しいSinkを追加

Sinkアプリケーションを追加して、メッセージの受信先を動的に増やしましょう。

### プロジェクト作成

以下のコマンドを実行すると、`tweet-viewer`フォルダに雛形プロジェクトが生成されます


```
curl start.spring.io/starter.tgz \
       -d artifactId=tweet-viewer \
       -d baseDir=tweet-viewer \
       -d dependencies=web,actuator,cloud-stream-binder-rabbit \
       -d applicationName=TweetViewerApplication | tar -xzvf -
```

`src/main/java/com/example/TweetViewerApplication.java`を次のように記述してください。


``` java
package com.example;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.stream.annotation.EnableBinding;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.cloud.stream.messaging.Sink;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
@EnableBinding(Sink.class)
public class TweetViewerApplication {

	private final List<Tweet> tweets = new CopyOnWriteArrayList<>();

	@GetMapping
	public List<Tweet> viewTweets() {
		return tweets;
	}

	@StreamListener(Sink.INPUT)
	public void collect(Tweet tweet) {
		tweets.add(tweet);
	}

	public static void main(String[] args) {
		SpringApplication.run(TweetViewerApplication.class, args);
	}

	public static class Tweet {
		public String text;
	}
}
```

このSinkアプリケーションでは受信したメッセージが`List`に追加され、HTTPのGETで確認できます。
channel名`input`に対するdestination名とConsumerGroup名を`application.properties`に次にように設定してください。


``` properties
spring.cloud.stream.bindings.input.destination=hello
spring.cloud.stream.bindings.input.group=viewer
```

はじめに作成したSourceからメッセージを受信できるようにdestination名は同じにする必要があります。しかし、先に作成したSinkとは別にメッセージを受信したいのでConsumerGroupは別にします。ここでは`viewer`という名前にしました。

次のコマンドでこのSourceアプリケーションのjarファイルを作成してください。

```
./mvnw clean package -DskipTests=true
```

`target`ディレクトリにtweet-viewer-0.0.1-SNAPSHOT.jarができていることを確認してください。

```
$ ls -lh target/
total 39048
drwxr-xr-x  4 makit  720748206   136B  2  6 01:17 classes
drwxr-xr-x  3 makit  720748206   102B  2  6 01:17 generated-sources
drwxr-xr-x  3 makit  720748206   102B  2  6 01:17 generated-test-sources
drwxr-xr-x  3 makit  720748206   102B  2  6 01:17 maven-archiver
drwxr-xr-x  3 makit  720748206   102B  2  6 01:17 maven-status
drwxr-xr-x  3 makit  720748206   102B  2  6 01:17 test-classes
-rw-r--r--  1 makit  720748206    19M  2  6 01:17 tweet-viewer-0.0.1-SNAPSHOT.jar
-rw-r--r--  1 makit  720748206   3.8K  2  6 01:17 tweet-viewer-0.0.1-SNAPSHOT.jar.original
```


### ローカル環境で実行

次のコマンドでアプリケーションを起動してください。

```
java -jar target/tweet-viewer-0.0.1-SNAPSHOT.jar --server.port=8085
```

管理コンソール([http://localhost:15672](http://localhost:15672))にアクセスして、Queuesタブを選択してください。`hello.viewe` Queueが追加されていることが確認できます。


![image](https://qiita-image-store.s3.amazonaws.com/0/1852/3934e335-14c8-84ae-05b0-a36e804adf39.png)


Sourceにリクエストを送りましょう。

```
curl -v localhost:8080 -d '{"text":"Hello1"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello2"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello3"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello4"}' -H 'Content-Type: application/json'
curl -v localhost:8080 -d '{"text":"Hello5"}' -H 'Content-Type: application/json'
```

一つ目のSinkにはいつも通りのログが取得されます（出力結果はSinkの数によって異なります）。

```
Received Hello1
Received Hello2
Received Hello3
Received Hello4
Received Hello5
```

その一方で、今回作成したSinkにもメッセージが届いていることを次のように確認してください。

```
$ curl http://localhost:8085/
[{"text":"Hello1"},{"text":"Hello2"},{"text":"Hello3"},{"text":"Hello4"},{"text":"Hello5"}]
```


### Cloud Foundryにデプロイ

`tweet-viewer`ディレクトリ直下に次の`manifest.yml`を作成してください。



``` yml
applications:
- name: tweet-viewer-tmaki
  memory: 512m
  buildpack: java_buildpack
  path: target/tweet-viewer-0.0.1-SNAPSHOT.jar
  services:
  - rabbitmq-binder
```

`tmaki`の部分は一意になるように自分のアカウント名などに置換してください。また、商用版Pivotal Cloud Foundryを使用する場合は`java_buildpack`の代わりに`java_buildpack_offline`を使用してください。


デプロイします。

```
$ Lcf push
Using manifest file /Users/makit/git/tweet-viewer/manifest.yml

Creating app tweet-viewer-tmaki in org APJ / space Development as tmaki@pivotal.io...
OK

Creating route tweet-viewer-tmaki.cfapps.io...
OK

Binding tweet-viewer-tmaki.cfapps.io to tweet-viewer-tmaki...
OK

Uploading tweet-viewer-tmaki...
Uploading app files from: /var/folders/15/fww24j3d7pg9sz196cxv_6xm4nvlh8/T/unzipped-app311973830
Uploading 520.9K, 100 files
Done uploading               
OK
Binding service rabbitmq-binder to app tweet-viewer-tmaki in org APJ / space Development as tmaki@pivotal.io...
OK

Starting app tweet-viewer-tmaki in org APJ / space Development as tmaki@pivotal.io...
Downloading java_buildpack...
Downloaded java_buildpack
Creating container
Successfully created container
Downloading app package...
Downloaded app package (16.9M)
Staging...
-----> Java Buildpack Version: v3.12 (offline) | https://github.com/cloudfoundry/java-buildpack.git#6f25b7e
-----> Downloading Open Jdk JRE 1.8.0_121 from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_121.tar.gz (found in cache)
       Expanding Open Jdk JRE to .java-buildpack/open_jdk_jre (1.2s)
-----> Downloading Open JDK Like Memory Calculator 2.0.2_RELEASE from https://java-buildpack.cloudfoundry.org/memory-calculator/trusty/x86_64/memory-calculator-2.0.2_RELEASE.tar.gz (found in cache)
       Memory Settings: -Xss349K -Xmx681574K -XX:MaxMetaspaceSize=104857K -Xms681574K -XX:MetaspaceSize=104857K
-----> Downloading Container Certificate Trust Store 1.0.0_RELEASE from https://java-buildpack.cloudfoundry.org/container-certificate-trust-store/container-certificate-trust-store-1.0.0_RELEASE.jar (found in cache)
       Adding certificates to .java-buildpack/container_certificate_trust_store/truststore.jks (0.6s)
-----> Downloading Spring Auto Reconfiguration 1.10.0_RELEASE from https://java-buildpack.cloudfoundry.org/auto-reconfiguration/auto-reconfiguration-1.10.0_RELEASE.jar (found in cache)
Exit status 0
Staging complete
Uploading droplet, build artifacts cache...
Uploading droplet...
Uploading build artifacts cache...
Uploaded build artifacts cache (108B)
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

App tweet-viewer-tmaki was started using this command `CALCULATED_MEMORY=$($PWD/.java-buildpack/open_jdk_jre/bin/java-buildpack-memory-calculator-2.0.2_RELEASE -memorySizes=metaspace:64m..,stack:228k.. -memoryWeights=heap:65,metaspace:10,native:15,stack:10 -memoryInitials=heap:100%,metaspace:100% -stackThreads=300 -totMemory=$MEMORY_LIMIT) && JAVA_OPTS="-Djava.io.tmpdir=$TMPDIR -XX:OnOutOfMemoryError=$PWD/.java-buildpack/open_jdk_jre/bin/killjava.sh $CALCULATED_MEMORY -Djavax.net.ssl.trustStore=$PWD/.java-buildpack/container_certificate_trust_store/truststore.jks -Djavax.net.ssl.trustStorePassword=java-buildpack-trust-store-password" && SERVER_PORT=$PORT eval exec $PWD/.java-buildpack/open_jdk_jre/bin/java $JAVA_OPTS -cp $PWD/. org.springframework.boot.loader.JarLauncher`

Showing health and status for app tweet-viewer-tmaki in org APJ / space Development as tmaki@pivotal.io...
OK

requested state: started
instances: 1/1
usage: 512M x 1 instances
urls: tweet-viewer-tmaki.cfapps.io
last uploaded: Sun Feb 5 16:56:59 UTC 2017
stack: cflinuxfs2
buildpack: java_buildpack

     state     since                    cpu      memory           disk           details
#0   running   2017-02-06 01:58:03 AM   199.1%   309.3M of 512M   142.4M of 1G
```


`cf apps`を実行して3つのアプリが起動していることを確認してください。

```
$ cf apps
...

hello-source-tmaki              started           1/1         512M     1G     hello-source-tmaki.cfapps.io
hello-sink-tmaki                started           2/2         512M     1G     hello-sink-tmaki.cfapps.io
tweet-viewer-tmaki              started           1/1         512M     1G     tweet-viewer-tmaki.cfapps.io
```

Sourceにリクエストを5回送ります。

```
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello1"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello2"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello3"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello4"}' -H 'Content-Type: application/json'
curl hello-source-tmaki.cfapps.io -d '{"text":"Hello5"}' -H 'Content-Type: application/json'
```

一つ目のSinkには次のようなログが出力されます。


```
$ cf logs --recent hello-sink-tmaki 
...
2017-02-06T02:10:33.91+0900 [APP/PROC/WEB/1]OUT Received Hello1
2017-02-06T02:10:34.27+0900 [APP/PROC/WEB/0]OUT Received Hello2
2017-02-06T02:10:34.63+0900 [APP/PROC/WEB/1]OUT Received Hello3
2017-02-06T02:10:35.00+0900 [APP/PROC/WEB/0]OUT Received Hello4
2017-02-06T02:10:35.37+0900 [APP/PROC/WEB/1]OUT Received Hello5
```

また、2つ目のSinkにアクセスすると受信した全メッセージを取得できます。

```
$ curl tweet-viewer-tmaki.cfapps.io
[{"text":"Hello1"},{"text":"Hello2"},{"text":"Hello3"},{"text":"Hello4"},{"text":"Hello5"}]
```

----

Consumer Groupの仕組みを利用して、メッセージを受信するSinkを後から追加できることを確認しました。

連携するサービスが後から増えても、サービス呼び出し側（メッセージ送信側)のコードを変更することなくサービスを追加することができます。

このような手法を[Choreography Style](https://www.thoughtworks.com/insights/blog/scaling-microservices-event-stream)と呼び、マイクロサービスアークテクチャを実現する上で重要なパターンとなります。

Spring Cloud Streamを利用することでChoreography Styleを簡単に実現できます。
