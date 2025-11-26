---
layout: post
title: 'Java proxy setting guide'
author: 김보배
date: 2021-09-07
tags: ['java', 'spring', 'proxy']
---

안녕하세요. 
11번가 Platform Engineering 팀의 서버 개발자 김보배 입니다.

서버 구성에서 보안적인 이유 등으로 Proxy 서버를 중간에 두고 허용된 요청만을 처리하도록 하는 경우가 많습니다. 
<br/>그런 경우, 애플리케이션 단의 요청에서 Proxy 서버를 거쳐 갈 수 있도록 설정을 따로 해주어야만 합니다.


그래서 이번 글에서는 Java 언어/Spring 환경에서 Proxy 설정을 어떻게 하는지 다뤄보고자 합니다.
<br/>특히, Java에서 대표적인 Http client인 `URLConnection`, `Apache HttpClient`, `RestTemplate`, `Feign` 에서의 설정법을 다뤄보겠습니다.

---

**목차**
- [들어가며](#들어가며)
- [System Property를 통한 Proxy 설정](#system-property를-통한-proxy-설정)
  - [System Property 란](#system-property-란)
  - [System Property 설정](#system-property-설정)
  - [다양한 Http Client Proxy 적용](#다양한-http-client-proxy-적용)
    - [HttpURLConnection](#httpurlconnection)
    - [Apache Http Client](#apache-http-client)
    - [RestTemplate](#resttemplate)
    - [Feign](#feign)
- [Code를 통한 Proxy 설정](#code를-통한-proxy-설정)
  - [다양한 Http Client에서 Proxy 적용](#다양한-http-client에서-proxy-적용)
    - [HttpURLConnection](#httpurlconnection-1)
    - [Apache Http Client](#apache-http-client-1)
    - [RestTemplate](#resttemplate-1)
    - [Feign](#feign-1)

---

# 들어가며

Proxy 설정 방법은 크게 2가지 방법으로 나눌 수 있습니다.
1. System property를 통한 Proxy 설정
   - System property를 설정해 Proxy를 적용하는 방법입니다.
   - 전역적으로 설정되는 경우가 있기에 대부분의 요청이 Proxy 서버를 거쳐가야 한다면, 이 방법을 사용하는게 좋습니다.
2. Code를 통한 Proxy 설정
   - Proxy 설정 코드를 직접 작성하여 Proxy를 적용하는 방법입니다.
   - 개별적으로 Proxy 적용이 가능하나, 어떤 Http client를 사용하냐에 따라 적용법이 달라집니다.

이제 좀 더 자세히 알아볼까요?

# System Property를 통한 Proxy 설정

## System Property 란

Java 공식문서의 [Network System Property](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/net/doc-files/net-properties.html#Proxies)에서 Proxy에 대한 System property를 정의해두고 있습니다.

|System Property|설명|기본 값|
|---|---|---|
|http.proxyHost|HTTP에 대한 Proxy hostname, address|none|
|http.proxyPort|HTTP에 대한 Proxy server port|`80`|
|http.nonProxyHosts|Proxy 설정에서 제외할 hostname, address| `localhost|127.*|[::1]`|
|https.proxyHost|HTTPS에 대한 Proxy hostname, address|none|
|https.proxyPort|HTTPS에 대한 Proxy server port|`443`|

이외에도 FTP, SOCKS에 대한 Proxy 설정을 지원하고 있습니다.


<br/> `http.nonProxyHosts` 의 경우, 다른 설정들과 다르게 특징이 있습니다.
- `|` 문자(Vertical bar)를 통해서 여러 호스트들을 설정할 수 있습니다.
- `*` 문자(Asterisk)를 통해서 Pattern 매칭을 할 수 있습니다.
- 예를 들어 ```-Dhttp.nonProxyHosts="*.foo.com|localhost"``` 로 설정한 경우, foo.com 도메인의 모든 호스트와 localhost 로의 요청은 Proxy로 향하지 않고, 곧바로 해당 주소로 요청이 가게 됩니다.
  * nonProxyHosts 패턴에 관련된 코드는 [여기](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/sun/net/spi/DefaultProxySelector.java#L276)를 참조해주세요.
- 일반적으로 `localhost|127.*|[::1]` 는 설정하는 편(default)이며, 추가적으로 Proxy가 필요하지 않은 경우 주소의 등록이 필요합니다.


`http.nonProxyHosts` 설정이 잘 적용되는지 다음과 같이 Test code로 작성해볼 수 있습니다.
```java
private static final String SYSTEM_PROPERTY_HTTP_PROXY_HOST = "http.proxyHost";
private static final String SYSTEM_PROPERTY_HTTP_PROXY_PORT = "http.proxyPort";
private static final String SYSTEM_PROPERTY_NON_PROXY_HOSTS = "http.nonProxyHosts";
 
@BeforeAll
static void beforeAll() {
    System.setProperty(SYSTEM_PROPERTY_HTTP_PROXY_HOST, "proxy.11st.com");
    System.setProperty(SYSTEM_PROPERTY_HTTP_PROXY_PORT, "1111");
}
 
@AfterAll
static void afterAll() {
    System.clearProperty(SYSTEM_PROPERTY_HTTP_PROXY_HOST);
    System.clearProperty(SYSTEM_PROPERTY_HTTP_PROXY_PORT);
    System.clearProperty(SYSTEM_PROPERTY_NON_PROXY_HOSTS);
}
 
@DisplayName("http.nonProxyHosts에 해당하는 host는 proxy가 설정되지 않아야한다.")
@Test
void testNonProxyHosts() {
    // given
    var nonProxyHosts =
            String.join("|", "172.16.*", "172.17.*", "172.18.*", "172.19.*", "172.20.*");
    System.setProperty(SYSTEM_PROPERTY_NON_PROXY_HOSTS, nonProxyHosts);
    var uri = URI.create("http://172.20.0.1:1111/test");
 
    // when
    var select = ProxySelector.getDefault().select(uri);
 
    // then
    assertThat(select).containsOnly(NO_PROXY);
}
 
@DisplayName("http.nonProxyHosts에 해당하지 않는 host는 proxy가 설정되어야만 한다.")
@Test
void testProxyHosts() {
    // given
    var nonProxyHosts = String.join("|", "172.16.*", "172.17.*", "172.18.*", "172.19.*");
    System.setProperty(SYSTEM_PROPERTY_NON_PROXY_HOSTS, nonProxyHosts);
    var uri = URI.create("http://172.20.0.1:1111/test");
 
    // when
    var select = ProxySelector.getDefault().select(uri);
 
    // then
    assertThat(select).containsOnly(new Proxy(Type.HTTP, new InetSocketAddress("proxy.11st.com", 1111)));
}
```

<br/>
## System Property 설정
System Property를 설정하기 위해선 2가지 방법이 있습니다.
- 실행 시 Parameter를 통한 System Property 설정
  - `java -Dhttp.proxyHost=PROXY_HOST_ADDRESS -Dhttp.proxyPort=PROXY_SERVER_PORT -jar 11st.jar` 와 같이 Parameter를 전달함으로서 설정
- Code를 통해 System Property 설정
  ```java
  System.setProperty("http.proxyHost", "proxy.11st.com");
  System.setProperty("http.proxyPort", "1111");
  ``` 

<br/>
## 다양한 Http Client Proxy 적용
그럼 이제 각 Http Client에서 System Property를 어떻게 활용하여 Proxy를 적용하는지 알아보겠습니다.

<br/>
### HttpURLConnection
기본적으로 System property를 사용해 Proxy를 지원하고 있으므로, 위 Proxy System property를 설정하셨다면 자동으로 Proxy 설정됩니다.

<br/>
### Apache Http Client
보통 HttpClient 의 구현체로 `CloseableHttpClient` 를 사용합니다.
<br/>`CloseableHttpClient`를 생성하기 위해선 `HttpClientBuilder`를 사용하거나 또는 Factory class인 `HttpClients`를 활용합니다.

위 방법을 통해 별 다른 설정없이 생성된 `CloseableHttpClient` 는 System property를 사용하지 않습니다.
```java
// Using HttpClientBuilder
HttpClientBuilder.create().build();
 
// Using HttpClients
HttpClients.createDefault();
 
// HttpClients.createDefault()
public static CloseableHttpClient createDefault() {
    return HttpClientBuilder.create().build();
}
```

System property를 사용하기 위해선 `HttpClientBuilder`의 [`useSystemProperties()`](https://github.com/apache/httpcomponents-client/blob/4.5.x/httpclient/src/main/java/org/apache/http/impl/client/HttpClientBuilder.java#L784-L787)를 사용하면 됩니다.
<br/>`useSystemProperties()`를 통해 `http.proxyHost`, `http.proxyPort` 등 System Property로 Proxy를 사용할 수 있습니다.
- `useSystemProperties()`는 Proxy 뿐 아니라 더 많은 설정 값을 적용시킬 수 있습니다. 더 자세한 내용은 [javadoc](https://www.javadoc.io/doc/org.apache.httpcomponents/httpclient/latest/org/apache/http/impl/client/HttpClientBuilder.html)을 참조해주세요.
- `HttpClientBuilder` build 시, `useSystemProperties()` 호출 유무에 따라 [`SystemDefaultRoutePlanner`](https://github.com/apache/httpcomponents-client/blob/4.5.x/httpclient/src/main/java/org/apache/http/impl/conn/SystemDefaultRoutePlanner.java#L54)를 생성해 적용하게 되는데요. 이를 통해 System property를 이용하여 Proxy를 설정하게 됩니다.

```java
// Using HttpClientBuilder
HttpClientBuilder.create().useSystemProperties().build();

// Using HttpClients
HttpClients.createSystem();

// HttpClients.createSystem()
public static CloseableHttpClient createSystem() {
    return HttpClientBuilder.create().useSystemProperties().build();
}
```

Spring framework의 [`HttpComponentsClientHttpRequestFactory`](https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/http/client/HttpComponentsClientHttpRequestFactory.java#L78-L80)의 경우, default로 `HttpClients.createSystem()` 을 사용합니다. 즉, System property를 사용하는 HttpClient가 만들어집니다.

<br/>
### RestTemplate
`RestTemplate`의 경우, default로 `HttpURLConnection`을 사용하고 있습니다.
<br/>즉, 별 다른 설정없이 System property가 활용되어 Proxy가 적용됩니다.
```java
new RestTemplate();

/*
RestTemplate.doExecute(...)
    -> HttpAccessor.createRequest(...)
    -> (requestFactory default) SimpleClientHttpRequestFactory.createRequest(...)
    -> HttpURLConnection
*/
```

만약 [`ClientHttpRequestFactory`를 사용해 HttpClient로 RestTemplate을 사용 중](https://github.com/spring-projects/spring-framework/blob/main/spring-web/src/main/java/org/springframework/web/client/RestTemplate.java#L206-L209)이라면 `HttpComponentsClientHttpRequestFactory`를 사용하면 됩니다. 위에서 언급했듯, `HttpComponentsClientHttpRequestFactory`는 기본적으로 System property를 사용하고 있기 때문입니다.

```java
var factory = new HttpComponentsClientHttpRequestFactory();

new RestTemplate(factory);

// OR
new RestTemplateBuilder().requestFactory(() -> factory).build();

// OR
var restTemplate = new RestTemplate();
restTemplate.setRequestFactory(factory);
```

<br/>
### Feign
Feign의 경우도 default로 [`HttpURLConnection`을 사용](https://github.com/OpenFeign/feign/blob/master/core/src/main/java/feign/Client.java#L150-L170)합니다.
그렇기에 위와 마찬가지로 System property만 제대로 설정되어졌다면 Proxy가 적용됩니다.

<br/>

Spring cloud OpenFeign을 사용하면, Feign 내부 client로 여러 Http client를 사용할 수 있습니다. 그 예로 앞에서 나왔던 Apache Http Client도 사용할 수 있습니다.
- `io.github.openfeign:feign-httpclient` 의존성을 추가하게 되면, default로 Feign client를 `CloseableHttpClient`(Apache http client)로 사용하게 되기 때문입니다.
- `FeignAutoConfiguration`의 [`HttpClientFeignConfiguration`](https://github.com/spring-cloud/spring-cloud-openfeign/blob/3.1.x/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignAutoConfiguration.java#L161-L166)과 [`feignClient` Bean 등록](https://github.com/spring-cloud/spring-cloud-openfeign/blob/3.1.x/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/FeignAutoConfiguration.java#L206-L210) 부분을 참고해주세요.

<br/>

결국 위와 같이 한다면, Feign 내부에서 Apache http client를 사용하게 됩니다. 그런데 Apache http client에서 System property를 통해 Proxy 설정을 하기 위해선 따로 *useSystemProperties()* 와 같은 설정이 필요했습니다. 그럼 여기서도 해주어야 하는 걸까요?
<br/>다행히 설정 코드를 따로 작성하지 않아도 System property만 제대로 설정되어있다면 Proxy 적용이 됩니다.

`FeignAutoConfiguration`의 `HttpClient`가 Bean으로 등록되는 부분을 살펴봅시다.
```java
@Bean
public CloseableHttpClient httpClient(ApacheHttpClientFactory httpClientFactory,
        HttpClientConnectionManager httpClientConnectionManager,
        FeignHttpClientProperties httpClientProperties) {
    RequestConfig defaultRequestConfig = RequestConfig.custom()
            .setConnectTimeout(httpClientProperties.getConnectionTimeout())
            .setRedirectsEnabled(httpClientProperties.isFollowRedirects()).build();
    this.httpClient = httpClientFactory.createBuilder().setConnectionManager(httpClientConnectionManager)
            .setDefaultRequestConfig(defaultRequestConfig).build();
    return this.httpClient;
}
```
`ApacheHttpClientFactory`를 주입받아 `createBuilder()` 통해 httpClient를 생성하고 있습니다. 
<br/>`ApacheHttpClientFactory`는 [`ApacheHttpClientConfiguration`](https://github.com/spring-cloud/spring-cloud-commons/blob/main/spring-cloud-commons/src/main/java/org/springframework/cloud/commons/httpclient/HttpClientConfiguration.java#L38)에서 구현체인 `DefaultApacheHttpClientFactory`가 Bean으로 등록되고 있습니다.
```java
@Bean
@ConditionalOnMissingBean
public ApacheHttpClientFactory apacheHttpClientFactory(HttpClientBuilder builder) {
    return new DefaultApacheHttpClientFactory(builder);
}
```

`DefaultApacheHttpClientFactory`의 `createBuilder()`에서는 `useSystemProperties()`를 사용하고 있습니다.
```java
public HttpClientBuilder createBuilder() {
    return this.builder.disableContentCompression().disableCookieManagement().useSystemProperties();
}
```

즉, Spring cloud OpenFeign에서 Apache http client를 사용한다면 System property를 통한 Proxy 설정이 자동으로 수행되게 됩니다.

<br/>

이 외에도 Feign Client는 OkHttpClient, ApacheHC5 를 사용할 수 있습니다.
- 관련된 공식문서는 [여기](https://docs.spring.io/spring-cloud-openfeign/docs/current/reference/html/#spring-cloud-feign-overriding-defaults)를 참조해주세요.

<br/>

추가적으로 11번가에서는 Feign과 함께 Spring Cloud LoadBalancer를 사용하고 있습니다.
- 이에 따라 `org.springframework.cloud:spring-cloud-starter-loadbalancer` 의존성이 추가되었습니다.
- Feign 공식문서에 따르면 Spring Cloud LoadBalancer가 classpath 있을 경우, Feign은 내부 client로 `FeignBlockingLoadBalancerClient`를 사용하게 됩니다.
> Client feignClient: If Spring Cloud LoadBalancer is on the classpath, FeignBlockingLoadBalancerClient is used. If none of them is on the classpath, the default feign client is used.
- `FeignBlockingLoadBalancerClient`의 경우, Apache Http Client를 사용하며 `useSystemProperties`를 사용하고 있어 System property 만으로 Proxy 설정이 가능합니다.
  * `HttpClientFeignLoadBalancerConfiguration` - [`feignClient`](https://github.com/spring-cloud/spring-cloud-openfeign/blob/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/loadbalancer/HttpClientFeignLoadBalancerConfiguration.java#L59-L63)
  * `HttpClientFeignConfiguration` - [`httpClient`](https://github.com/spring-cloud/spring-cloud-openfeign/blob/main/spring-cloud-openfeign-core/src/main/java/org/springframework/cloud/openfeign/clientconfig/HttpClientFeignConfiguration.java#L93-L98)


<br/>

이상으로 System property를 통해 Proxy를 설정하는 법을 알아보았습니다.

만약 모든 요청이 Proxy를 적용해야 한다면, 위와 같은 방식으로 설정하여 활용할 수 있을텐데요. 하지만 Proxy 적용이 필요하지 않는 요청도 있다면, System property 중  `http.nonProxyHosts`를 잘 설정하여 사용하여야합니다.
<br/>정말 소수의 요청에만 Proxy를 설정해야 한다면, System property를 활용하는 것보다 Code를 통해 Proxy를 설정하는 편이 나은 방법일 수 있습니다. System property는 설정만 하면 자동으로 적용되는 경우가 있고, 그 적용이 Library 내부에 있어 쉽게 알지 못하는 경우가 있기 때문입니다.

---

# Code를 통한 Proxy 설정

## 다양한 Http Client에서 Proxy 적용
각 Http Client에서 Code를 통해 Proxy를 어떻게 적용하는지 알아보겠습니다.

<br/>
### HttpURLConnection
`Proxy` 클래스를 생성해 `openConnection` 메서드의 파라미터로 넣어주면 됩니다.
```java
var proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress(properties.getProxyHost(), properties.getProxyServerPort()));
var url = new URL(properties.getTargetHttpURI());
var con = (HttpURLConnection) url.openConnection(proxy);
```

<br/>
### Apache Http Client
`Proxy` 클래스를 생성해 `ProxyRoutePlanner`를 생성합니다.
<br/>생성된 RoutePlnanner를 `HttpClient`의 `setRoutePlanner` 메서드를 통해 Proxy를 적용할 수 있습니다.

```java
var proxy = new HttpHost(properties.getProxyHost(), properties.getProxyServerPort());
var routePlanner = new DefaultProxyRoutePlanner(proxy);
return HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
```

<br/>

`HttpClient`의 `setProxy`를 활용해 Proxy를 적용할 수도 있습니다.
<br/>`setProxy`는 내부적으로 RouterPlanner가 설정되지 않았을 경우, `DefaultProxyRoutePlanner`를 통해 설정합니다. 즉, 위 방법과 동일하게 동작합니다.

```java
var proxy = new HttpHost(properties.getProxyHost(), properties.getProxyServerPort());
return HttpClients.custom()
        .setProxy(proxy)
        .build();
```

<br/>
### RestTemplate
RestTemplate은 default로 HttpURLConnection 를 사용하고 있습니다. 하지만 System property 설정에서 보았듯 Apache Http Client 도 사용할 수 있습니다.
<br/>앞서 HttpURLConnection, Apache Http Client 두 경우 다 code로 Proxy 설정을 할 수 있었는데, 설정을 똑같이 해서 RestTemplate으로 넘겨주면 됩니다.

- HttpURLConnection를 사용하는 경우
  * `SimpleClientHttpRequestFactory`를 통해 HttpURLConnection Factory를 만들고, Proxy를 설정해줍니다.


```java
var proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress(properties.getProxyHost(), properties.getProxyServerPort()));
var requestFactory = new SimpleClientHttpRequestFactory();
requestFactory.setProxy(proxy);
return new RestTemplate(requestFactory);
 
 
// SimpleClientHttpRequestFactory에선 request 시에 아래와 같이 동작하게 됩니다.
protected HttpURLConnection openConnection(URL url, @Nullable Proxy proxy) throws IOException {
    URLConnection urlConnection = (proxy != null ? url.openConnection(proxy) : url.openConnection());
    if (!(urlConnection instanceof HttpURLConnection)) {
        throw new IllegalStateException(
                "HttpURLConnection required for [" + url + "] but got: " + urlConnection);
    }
    return (HttpURLConnection) urlConnection;
}
```


- Apache Http Client를 사용하는 경우
  * `HttpComponentsClientHttpRequestFactory`를 통해 Proxy 설정이 된 HttpClient를 등록합니다.

```java
var proxy = new HttpHost(properties.getProxyHost(), properties.getProxyServerPort());
var routePlanner = new DefaultProxyRoutePlanner(proxy);
var httpClient = HttpClients.custom()
        .setRoutePlanner(routePlanner)
        .build();
 
var factory = new HttpComponentsClientHttpRequestFactory();
factory.setHttpClient(httpClient);
 
 
return new RestTemplate(factory);
 
// OR
new RestTemplateBuilder().requestFactory(() -> factory).build();
 
// OR
var restTemplate = new RestTemplate();
restTemplate.setRequestFactory(factory);
```

<br/>
### Feign

System Property 부분에서 살펴보았듯, Feign 내부 Client로 다양한 Http client를 사용할 수 있습니다.

또한 Spring cloud OpenFeign과 함께 `feign-httpclient`, `Spring cloud LoadBalancer`를 사용한 예를 살펴보았는데요.
<br/>이 두 방식 다 `CloseableHttpClient`를 주입받아 Client를 생성하고 있습니다.

그럼 `CloseableHttpClient`에 Proxy를 설정하여 Bean으로 등록하면 되는 것일까요?
<br/>Feign을 사용하는 모든 요청에 Proxy이 필요하다면 상관없지만, Feign 요청 중에서도 개별적으로 Proxy 설정을 해야한다면 이렇게 해서는 불가능합니다.

<br/>
**개별 Feign에 Proxy를 설정하고 싶은 경우**

RoutePlanner를 활용해 Proxy 설정을 원하지 않는 HttpHost인 경우 Proxy를 타지 않도록 하고, Proxy 설정을 해야하는 HttpHost인 경우 Proxy를 타도록 하면 됩니다.

이 설정을 하기 위해서는 `HttpRoutePlanner`의 `determineRoute`를 구현하면 됩니다. `HttpRoutePlanner`의 구현체인 `DefaultRoutePlanner`를 상속받아 `determineRoute`를 다음과 같이 구현했습니다.
```java

public class CustomProxyRoutePlanner extends DefaultRoutePlanner {

    private final HttpHost proxy;
    private final List<HttpHost> noProxyHttpHosts;

    public CustomProxyRoutePlanner(final HttpHost proxy, final SchemePortResolver schemePortResolver, final List<String> noProxyHosts) {
        super(schemePortResolver);
        this.proxy = Args.notNull(proxy, "Proxy host");
        this.noProxyHttpHosts = noProxyHosts.stream()
                .map(HttpHost::new)
                .collect(Collectors.toList());
    }

    public CustomProxyRoutePlanner(final HttpHost proxy, final List<String> noProxyHosts) {
        this(proxy, null, noProxyHosts);
    }

    ...
    
    @Override
    protected HttpHost determineProxy(
            final HttpHost target,
            final HttpRequest request,
            final HttpContext context) throws HttpException {

        if (!noProxyHttpHosts.isEmpty() && noProxyHttpHosts.contains(target)) {
            return null;
        }
        return proxy;
    }

}

```

이러한 `CustomProxyRoutePlanner`를 사용한 `HttpClientBuilder`를 생성해 Bean으로 등록해줍니다.
```java
@Bean
public HttpClientBuilder apacheHttpClientBuilder() {
    var proxy = new HttpHost(clientProperties.getProxyHost(), clientProperties.getProxyServerPort());
    var customRoutePlanner = new CustomProxyRoutePlanner(proxy, clientProperties.getNoProxy());
    return HttpClientBuilder.create().setRoutePlanner(customRoutePlanner);
}
```

지금까지의 내용이 도움이 되셨길 바라며, 이상 Proxy 설정하는 법을 마치겠습니다.

감사합니다.