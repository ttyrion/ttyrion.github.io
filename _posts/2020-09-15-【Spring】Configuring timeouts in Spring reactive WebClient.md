---
layout:         page
title:          【Spring】Configuring timeouts in Spring reactive WebClient
date:           2020-09-15
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

> 原文地址：[Configuring timeouts in Spring reactive WebClient](https://medium.com/@kalpads/configuring-timeouts-in-spring-reactive-webclient-4bc5faf56411)

关于详细的 reactive-stack web applications，可以参考Spring文档：[Web on Reactive Stack](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html)

## WebClient Introduction
[WebClient](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/web/reactive/function/client/WebClient.html)是在通过响应式编程来开发web服务过程中处理**reactive downstream endpoints**调用的一个事实上的接口标准。它是一个处理HTTP请求的非阻塞（non-blocking）的，反应式（reactive）的客户端，以**Reactor Netty**作为底层HTTP客户端函数库。同时，它也支持开发者在需要的时候，自定义一个实现。

在一个弹性系统（resilient system）中调用downstream endpoints的一个关键，就是优雅地处理超时以及阻止系统产生大量失败事务的可能性。如果没有经过仔细的设计，大量未处理的超时可能耗尽系统资源。

## The illusion of signal timeout
一个典型的使用WebClient请求downstream reactive endpoint的场景可能是这样的：
```java
webClient.get()
        .uri(uri)
        .retrieve()
        .onStatus(HttpStatus::isError, clientResponse -> {
            LOGGER.error("Error while calling endpoint {} with status code {}",
            uri.toString(), clientResponse.statusCode());
            throw new RuntimeException("Error while calling  accounts endpoint");
		})
		.bodyToMono(JsonNode.class)
        .timeout(Duration.ofMillis(timeout));

```
请注意，最后一行调用了timeout()方法。或许有人会觉得这个timeout是由Mono/ Flux publisher处理HTTP连接、读取超时的一个通用值。

但是，这个timeout其实与TCP层的超时没有任何关系。这个timeout值来自[Mono publisher类](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html#timeout-java.time.Duration-)，对于Flux也是这样。

### Mono Timeout
思考下这个问题：**If this is not the TCP timeout values what does this timeout value mean?**

这里的这个timeout指的是若给定的publisher未能在给定时间内发出下一个信号，则Mono会抛出一个TimeoutException。

看看如下示例。下面这个测试类利用MockServer来模拟带延时的downstream responses。它会启动一个局部的 mock server并且注册一个响应/accounts的延时5秒的 mock response。
```java

package com.example.application;

// import...

public class WebClientTimeoutTest {

  private static final Logger LOGGER = LoggerFactory.getLogger(ApplicationTests.class);

  private static String BASE_URL = "http://localhost:1080";

  private ClientAndServer mockServer;
  private WebClient webClient;

  @BeforeEach
  public void startMockServer() {
    mockServer = startClientAndServer(1080);

    // set up mock with a delay of 5 seconds
    mockServer.when(HttpRequest.request().withMethod("GET")
            .withPath("/accounts"))
            .respond(
                    HttpResponse.response()
                            .withContentType(MediaType.APPLICATION_JSON)
                            .withBody("{ \"result\": \"ok\"}")
                            .withDelay(TimeUnit.MILLISECONDS, 5000)
            );

    webClient = WebClient.builder().build();
  }

  private Mono<JsonNode> doGetWithDefaultConnectAndReadTimeOut(URI uri, long timeout) {
    return webClient.get()
            .uri(uri)
            .retrieve()
            .onStatus(HttpStatus::isError, clientResponse -> {
              LOGGER.error("Error while calling endpoint {} with status code {}",
                      uri.toString(), clientResponse.statusCode());
              throw new RuntimeException("Error while calling  accounts endpoint");
            })
            .bodyToMono(JsonNode.class)
            // setting the signal timeout
            .timeout(Duration.ofMillis(timeout));
  }

  @Test
  public void testWebClient() {
    URI uri = UriComponentsBuilder.fromUriString(BASE_URL + "/accounts").build().toUri();
    // do the service call out with 3 seconds of signal timeout
    Mono<JsonNode> result = doGetWithDefaultConnectAndReadTimeOut(uri, 3000);
    StepVerifier.create(result)
            .expectSubscription()
            .assertNext(jsonNode -> assertEquals("ok", jsonNode.get("result").textValue()))
            .verifyComplete();
  }

  @AfterEach
  public void stopMockServer() {
    mockServer.stop();
  }

}

```
这个测试里面使用了publisher（Mono）的timeout方法来配置了一个3s的超时值。当我们运行这个测试时，将会看到如下的错误：
```java
[ERROR] testWebClient  Time elapsed: 5.818 s  <<< FAILURE!
java.lang.AssertionError: expectation "assertNext" failed (expected: onNext(); 
actual: onError(java.util.concurrent.TimeoutException: Did not observe any item or terminal signal 
within 3000ms in 'flatMap' (and no fallback has been configured)))

```
timeout()方法实际上会注册一个错误信号发生器（error signal generator），并且当指定时间内无任何事件时，该错误信号发生器将被触发。

我们可以像处理其他任何错误一样通过添加一个OnError()来处理这个错误：
```java
private Mono<JsonNode> doGetWithDefaultConnectAndReadTimeOut(URI uri, long timeout) {
    return webClient.get()
            .uri(uri)
            .retrieve()
            .onStatus(HttpStatus::isError, clientResponse -> {
              LOGGER.error("Error while calling endpoint {} with status code {}",
                      uri.toString(), clientResponse.statusCode());
              throw new RuntimeException("Error while calling  accounts endpoint");
            })
            .bodyToMono(JsonNode.class)
            // setting the signal timeout
            .timeout(Duration.ofMillis(timeout))
            // detecting the timeout error
            .doOnError(error -> LOGGER.error("Error signal detected", error));
  }

```

## The real HTTP Connect and Read timeouts
现在将信号超时抛开，我们来看看如何配置TCP层的超时。无论是阻塞式或非阻塞客户端，TCP层的工作原理都一样。

请注意，上面的测试类WebClientTimeoutTest中，使用了默认的builder来构建了一个WebClient，并没有任何特定的配置。因此，这个client会使用默认的连接、数据读取超时值（都是30s）。现代应用程序不可能等待30s这么长的时间，这些系统更希望这样的请求尽快失败以便节约珍贵的CPU等资源。

请参考Spring文档[webflux-client-builder-reactor-timeout](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html#webflux-client-builder-reactor-timeout)，这里介绍了如何配置连接超时以及读写超时。

下面是一个配置连接超时和数据读取超时的示例：
```java
private WebClient createWebClientWithConnectAndReadTimeOuts(int connectTimeOutMs, long readTimeOutMs) {
    // create reactor netty HTTP client
    HttpClient httpClient = HttpClient.create()
            .tcpConfiguration(tcpClient -> {
              tcpClient = tcpClient.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, connectTimeOutMs);
              tcpClient = tcpClient.doOnConnected(conn -> conn
                      .addHandlerLast(new ReadTimeoutHandler(readTimeOutMs, TimeUnit.MILLISECONDS)));
              return tcpClient;
            });
    // create a client http connector using above http client
    ClientHttpConnector connector = new ReactorClientHttpConnector(httpClient);
    // use this configured http connector to build the web client
    return WebClient.builder().clientConnector(connector).build();
  }
```

另外，当http数据过大（超过262144字节）时，WebClient这里会产生异常：
```java
org.springframework.core.io.buffer.DataBufferLimitException: Exceeded limit on max bytes to buffer : 262144

```
所以，还可以将上面build WebClient的过程再改一下，如下面这里将缓冲区大小设置为2M：
```java
...

ExchangeStrategies exchangeStrategies = ExchangeStrategies.builder()
            .codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(1024 * 1024 * 2)).build();

return WebClient.builder()
		.baseUrl("http://www.hello.com")
		.clientConnector(connector)
		.defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
		.exchangeStrategies(exchangeStrategies)
		.build();
```

接下来，调用WebClient时就不需要配置信号超时了。当然，这里依然可以设置信号超时，重要的是要理解：不能将这个信号超时来替代TCP超时。

至此，测试用例中应该是期望产生一个ReadTimeoutException ：
```java
private Mono<JsonNode> doGetWithDefaultConnectAndReadTimeOut(URI uri, long timeout) {
    return webClient.get()
            .uri(uri)
            .retrieve()
            .onStatus(HttpStatus::isError, clientResponse -> {
              LOGGER.error("Error while calling endpoint {} with status code {}",
                      uri.toString(), clientResponse.statusCode());
              throw new RuntimeException("Error while calling  accounts endpoint");
            })
            .bodyToMono(JsonNode.class)
            // setting the signal timeout
            // .timeout(Duration.ofMillis(timeout))
            // detecting the timeout error
            .doOnError(error -> LOGGER.error("Error signal detected", error));
  }

  @Test
  public void testWebClient() {
    URI uri = UriComponentsBuilder.fromUriString(BASE_URL + "/accounts").build().toUri();
    // do the service call out with 3 seconds of signal timeout
    Mono<JsonNode> result = doGetWithDefaultConnectAndReadTimeOut(uri, 3000);
    StepVerifier.create(result)
            .expectSubscription()
            // .assertNext(jsonNode -> assertEquals("ok", jsonNode.get("result").textValue()))
            // .verifyComplete();
            .expectError(ReadTimeoutException.class)
            .verify();
  }
```

## 后记
关于WebClient以及WebTestClient的用法，可以参考一下[spring-5-webclient](https://www.baeldung.com/spring-5-webclient)。