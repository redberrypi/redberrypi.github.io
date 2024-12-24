---
layout: post
title: 최근 마주한 OutOfMemoryError 분석기
date: 2024-12-24 23:24 +0900
categories: [Java]
tags: [Java, Jvm]
---

## 최근 마주한 OutOfMemoryError에 대해

사실 꽤 전부터 우리 회사에는 OutOfMemoryError(이하 OOME)가 발생하는 어플리케이션이 있었다.
그런데 아무도 그 원인을 해결하지 못했다(...).
이번에 어쩌다 그 원인을 파악하게 되었는데, 이에 그 경험을 적어보고자 한다.

## 일반적으로 OOME가 발생하는 이유

OOME가 발생하는 이유는 여러 가지가 있다.
단적으로 Java의 경우 Closeable 인터페이스를 구현한 클래스를 제대로 close하지 않을 경우 해당 리소스가 메모리에 남아 메모리 누출을 야기하기도 한다.
다만 Closeable의 경우에는 Close하는 대상 리소스의 종류에 따라, 또는 내부 구현에 따라 close를 하지 않아도 상관이 없는 경우가 있을 수 있다.
그 외에는 Buffer를 사용했는데 release를 제대로 하지 않았다든지, 아니면 코드 상의 문제는 없다 하더라도 프로그램의 설계에 비해 너무 많은 부하가 가해진다든지 하는 경우가 있을 수 있을 것 같다.

본인이 처음 OOME 소식을 전달받았을 때, 나는 그 원인이 Buffer의 release와 관련된 문제가 아닐까 하고 생각했었다.
해당 어플리케이션이 Netty의 ByteBuffer를 직접적으로 다룬다는 것을 알고 있었고, close와는 달리 release는 전체 코드를 파악하지 못하면 까다로운 부분이 될 수 있었기 때문이었다.
하지만 결과적으로 말하자면, 이번에 OOME의 원인이 된 부분은 아예 다른 부분이었다.

## 문제의 재현

OOME를 분석하려고 할 때 제일 먼저 문제가 된 부분은 사실 문제의 재현이었다.
애초에 OOME가 일어나고 있는 곳은 실제 어플리케이션이 적용된 사이트의 내부망이었는데, 문제는 우리가 이곳에서 힙 덤프 파일을 빼올 수가 없다는 점이었다.
그래서 바깥에서 문제를 재현해야만 했는데, 다음과 같이 설정했다.
우선, JVM 옵션을 다음과 같이 설정했다.

``` Jvm
-Xmx2g
-Xms2g
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/logs/heapdump.hprof 
-XX:OnOutOfMemoryError=/scripts/log.sh
-XX:+CrashOnOutOfMemoryError
-XX:ErrorFile=/logs/hs_err_pid%p.log
```

`Xmx`, `Xms` 부분은 JVM의 최대, 최소 메모리 설정 부분이다.
나는 2g로 설정했지만 경우에 따라 OOME를 더 빨리 일으키기 위해서 더 타이트하게 제한할 수도 있을 것 같다.
나머지 옵션들은 이전 포스트에서 언급한 OOME 발생시의 옵션들이다.
여기서 중요한 부분은 `-XX:+HeapDumpOnOutOfMemoryError`와 `-XX:HeapDumpPath`으로, 힙 덤프를 떠서 분석하기 위한 용도이다.
해당 설정으로 자바 어플리케이션의 도커 컨테이너를 띄운 뒤 Jmeter를 사용하였다.
Jmeter는 HTTP 등의 요청을 보내 API, 특히 REST API를 주로 테스트하기 위한 툴이다.
해당 툴로 HTTP 요청을 마구마구 보내서 OOME를 일으키는 것이 가능하다(만약 가능하다면).
덧붙이자면 REST API를 테스트하기 위해 일반적으로 Postman이라는 툴을 많이 이용할 것이라고 생각한다.
Jmeter와 Postman 둘 다 비슷한 용도로 사용되지만, 이와 같이 성능 테스트를 하기 위한 용도로는 Jmeter를 사용하는 것이 좋다.
Postman의 경우 HTTP 호출을 반복하는 것이 가능하지만, 단일 스레드로만 동작하기 때문에 실제 운영 환경과는 다른 동작을 보여줄 가능성이 있기 때문이다.
반면 Jmeter의 경우 호출 스레드의 수를 마음대로 조절하는 것이 가능하다.
여기서 또 하나를 덧붙이자면 Jmeter의 경우 성능 테스트 시에 HTTP 호출을 위해 하위 프로세스를 따로 생성하게 된다.
만약 linux에서 `nohup` 명령어를 통해 프로세스를 실행할 경우 `ps -ef | grep ~~` 명령어를 통해 프로세스를 종료하게 될 텐데, jmeter 키워드로 검색을 하면 하위 프로세스는 보이지가 않는다.
jmx 파일 이름으로 검색을 하면 프로세스가 2개가 보일 텐데(ps 자신을 제외하고) 둘 다 kill해주면 된다.

아무튼 위와 같은 방법으로 성능테스트하듯 마구마구 리퀘스트를 날렸다.
머지 않아 힙 덤프가 생성되었다.

## 힙 덤프 분석 1

일단 힙 덤프가 생성되고 나면 그것을 분석할 수 있다.
분석을 위해서는 일반적으로 Eclipse Memory Analyzer (MAT)나 VisualVM 등의 분석툴을 이용한다.
보통 MAT가 더 좋다고 하는 것 같다.
대쉬보드를 키니 다음과 같은 화면이 떠올랐다.

![MAT_LEAK_SUSPECT1](https://github.com/redberrypi/redberrypi.github.io/blob/0598e6c1901a8c04f5d8146b53b4acf94714bbd0/assets/png/MAT-leak%20report.png?raw=true)

처음에는 저 `PoolingNHttpClientConnectionManager`이라는 클래스가 무엇인지 알지 못해서 헤매었다.
해당 제품은 netty의 http 클라이언트를 사용하는데 apache 쪽의 라이브러리가 원인으로 지목된 원인을 알 수가 없었다.
그래서 분석툴 사용법을 배우기 위해 여기저기를 뒤적였다(나름 복잡하다.).
결과적으로 말하자면 저것만으로 충분한 단서가 되었다.
해당 키워드로 검색하자 ElasticSearch와 관련이 있음을 알 수가 있었다.
알고보니 ElasticSearch에서 제공하는 Java client 라이브러리 `org.elasticsearch.client:elasticsearch-rest-client`에서 해당 클래스를 사용하고 있었던 것이었다.
조금 더 파고들어가자 진정한 원인이 드러났다.
ConnectionManager라는 이름답게 해당 클래스는 HTTP Connection을 관리하고 있었는데, 각 커넥션마다 ElasticSearch로 보냈어야 할 데이터가 byte array 형태로 붙어 있었다.
그리고 그 커넥션이 무수하게 많이 축적되어 있음을 알 수 있었다.
원인은 분명했다.
커넥션 풀과 관련된 설정이 잘못된 것이었다.

``` Java
    RestClient.builder(getElasticSearchHost())
        .setHttpClientConfigCallback(httpClientConfigCallback -> httpClientConfigCallback
            .setConnectionManager(connManager)
            .setKeepAliveStrategy((response, context) -> keepAliveTime)
            .setConnectionTimeToLive(connectionTimeToLive, TimeUnit.SECONDS)
            .setDefaultCredentialsProvider(credentialsProvider)
        )
        .setRequestConfigCallback(requestConfigCallback -> requestConfigCallback
            .setConnectTimeout(connectTimeout)
            .setConnectionRequestTimeout(connectionRequestTimeout)
            .setSocketTimeout(socketTimeout)
        )
    .build()
```

ElasticSearch의 client에 대해 대충 위와 같이 설정함으로써 커넥션 수를 제한하고 관련된 메모리 오류를 없앨 수 있었다.
덧붙이자면, 위의 설정을 조금 바꾸면 원래 들어가야 할 값이 제대로 안 들어가는 경우가 생긴다(socketTimeout 등).
이러한 경우 CPool 객체를 디버깅하면 정확한 값이 들어갔는지 안 들어갔는지 확인할 수 있다.
만약 제대로 된 값이 들어가지 않았다면 뭔가 설정이 꼬인 것이니 설정 방법을 바꿔보는 것이 좋을 수도 있다(httpClientConfigCallback 대신 requestConfigCallback을 사용해본다든지, 혹은 그 반대라든지).
또 하나를 덧붙이자면, 위의 케이스에서 문제가 된 건 정확히는 socketTimeout이 -1로 들어가서 leaseRequest에 담긴 객체들이 해제되지 않아서였다.
만약 socketTimeout을 충분히 늘려서 객체가 해제되지 않게 한다면 OOME를 볼 수 있을 것이다.
다만 이 경우 객체가 해제된다면 해당 커넥션의 데이터는 ElasticSearch로 보내지지 않고 소실된다.

### 힙 덤프 1에 대한 첨언

해당 OOME를 고치고 나서 꽤 지난 뒤에 쓰는 글이지만, 지금으로서는 대용량 request를 저장시키는 것에 있어 HttpClient를 사용해야 하는 것에 대한 의문이 들기도 한다.
왜냐하면 거의 동일한 조건으로도 Kafka는 큰 문제 없이 데이터를 실시간으로 소화해내었기 때문이다.
ElasticSearch의 경우 기본적으로 HTTP 통신을 통해 데이터를 주고받는 것을 원칙으로 삼는다.
그렇기 때문에 편리한 면도 있지만, HTTP이기 때문에 성능적으로 잃게 되는 부분이 있을지도 모른다.
위와 같이 단기간에 대용량의 데이터를 주고받을 경우 다른 수단을 사용한다면 더 좋은 결과가 나오지 않을까 하는 생각도 든다.

## 힙 덤프 분석 2

안타깝게도 위와 같이 적용했음에도 OOME는 끝이 아니었다.
해당 설정을 적용하기 전에 비해 훨씬 오래 버티기는 했지만, 결국에는 어플리케이션이 터져버리고 말았다.
그나마 다행인 건 이번에 어플리케이션이 터진 이유는 다른 것이라는 점이었다.
위의 커넥션 풀 문제에 가려져 드러나지 않았던 잠재적 메모리 오류의 원인이 또 한 가지가 있었다.
MAT를 통해 LEAK SUSPECT를 검사하자 다음과 같이 나왔다.

![MAT_LEAK_SUSPECT2](https://github.com/redberrypi/redberrypi.github.io/blob/0598e6c1901a8c04f5d8146b53b4acf94714bbd0/assets/png/MAT-leak%20report2.png?raw=true)

이번에는 OpenTelemetry와 관련된 문제였다.
해당 라이브러리를 뭔가 잘못 사용하고 있는 것이 틀림 없었다.
처음에는 Scope나 Span을 제대로 close하고 있지 않기 때문이라고 생각했다.
그러나 실제로 메모리가 가득 찬 `ComponentRegistry`라는 클래스를 확인해보니 그렇게 간단하지 않다는 사실을 알 수 있었다.
MAT로 메모리를 조사한 결과, 해당 클래스의 내부에는 `private final Map<String, V> componentByName = new ConcurrentHashMap<>();`로 정의된 맵이 있으며, 이 내부에 무수하게 많은 UUID 형식의 String이 key로 담겨 있음을 알 수 있었다.
value는 null인 것도 SDK 관련 객체인 경우도 있었으나, 중요한 것은 이 key 자체가 너무 많아서 메모리를 전부 차지하고 있다는 점이었다.
게다가 환장하는 부분이 있었는데, 해당 클래스에는 애초에 `componentByName`에 무언가를 집어 넣거나 바꾸는 로직만이 존재할 뿐 해당 Map에서 무언가를 제거하는 로직은 없다는 점이었다.
심지어 상속조차 받지 않았으니, 부모 클래스에 어떤 로직이 숨겨져 있다는 가능성도 부정되었다.
아니, Map 안에 이렇게나 많은 데이터가 쌓여 있음에도 이걸 제거할 방법이 없다니!
뭔가가 단단히 잘못된 것이 틀림 없었다.
그리고 조사 결과 이유가 밝혀졌다.
그 이유는 openTelemetry를 사용할 때 Trace를 얻기 위해 사용하는 `openTelemetry.getTracer` 메소드를 잘못 이용하고 있기 때문이었다.
해당 메소드는 인자로서 `String instrumentationScopeName`을 받는데, 이것은 해당 라이브러리나 앱 이름, 혹은 패키지명 등, 무엇이든 좋으니 고정된 String 값을 의미한다.
그런데 무슨 이유에서인지 해당 인자에 자체적으로 생성한 UUID의 String 값을 넣고 있었던 것이다.
다시 말해 리퀘스트가 들어올 때마다 UUID를 생성해서 넣고, 또 다시 리퀘스트가 들어오면 UUID를 생성해서 넣고 하고 있었던 것이다.
하지만 해당 메소드는 원래 고정된 값이 들어올 것을 상정하고 만든 것이기 때문에, 해당 메소드를 통해 들어온 `instrumentationScopeName`을 내부 `ComponentRegistry`에 따로 저장을 해서 cache처럼 사용하고 있었다.
그래서 애초에 이미 등록된 값을 지울 필요조차 없었던 것이다.
그런데 그런 메소드에 UUID 폭탄을 들이 부어버렸으니, 메모리가 가득 찰 수밖에.
뜬금없는 UUID가 가득 차 있었던 것도 이 때문이었던 것이다.

``` Java
openTelemetry.getTracer(UUID.toString())
```

이 코드를

``` Java
openTelemetry.getTracer("some-library")
```

이렇게 바꿈으로써 해당 오류를 간단히 해결할 수 있었다.

## 마무리

메모리와 관련된 오류는 일견 보이지 않기에 잡아내는 것이 상당히 까다롭다.
하지만 힙 덤프와 함께라면 아무리 어려워 보이는 오류라도 잡아내는 것이 가능하다.
덧붙이자면 위의 문제를 해결하며 은근히 ChatGPT의 도움을 많이 받았다.
AI에 대한 내 입장은 아직까지 부정적이지만, 미래에 대한 입장까지 부정적이지는 않다.
AI가 완전히 자리 잡기 전에 나 자신도 빨리 자리를 잡아야 하지 않나.
그런 생각을 하곤 한다.
