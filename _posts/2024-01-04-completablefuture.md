---
layout: post
title: Java CompletableFuture로 비동기 적용하기
author: 박지훈
date: 2024-01-04
tags: [java, async, completableFuture]
---

안녕하세요. 11번가 클레임개발팀 박지훈입니다.

중앙 집중식 데이터베이스를 영역별로 분리하는 **탈중앙화를 대비**하여 분리 대상 테이블을 참조하고 있는 **쿼리를 분리하고, 이관하는 작업**을 진행하고 있습니다. 코드를 이관하는 과정에서 가장 중요한 부분은 as-is, to-be 결과를 비교하는 부분일 텐데요. 기존 결과 비교를 위해 이관 전/후 로직을 실행하는 부분이 **순차적으로 실행되다 보니 전체적인 실행시간이 두 배로 증가하는 문제**를 마주하였고, 1초 차이로 조회 결과가 달라지는 경우 이관 전 로직 실행이 완료된 이후 이관 후 로직이 실행되면서 **정상 케이스임에도 결과 비교가 실패하는 문제**를 발견하게 되었습니다.

as-is, to-be 로직을 순차적으로 실행하면서 발생하는 **실행 시간 증가 문제**와 1초 차이로 **조회 결과가 달라지는 문제**를 마주하여 안전한 이관을 위해 개선의 필요성을 느끼게 되었고 이 문제를 **비동기로 해결**하게 되었습니다.

비동기는 Java8에 등장한 [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 클래스를 활용하게 되었는데요. 비동기를 처음 적용하다 보니 관련 내용을 학습하면서 나중에 다른 분들도 쉽게 비동기를 적용하실 수 있도록 기본적인 학습 내용을 공유하면 좋겠다는 생각을 시작으로 글을 작성하게 되었습니다.

## Contents

- [Contents](#contents)
- [비동기 처리](#비동기-처리)
- [CompletableFuture](#CompletableFuture)
    - [CompletableFuture 인스턴스](#completableFuture-인스턴스)
    - [순차적으로 연산 처리하기](#순차적으로-연산-처리하기)
    - [연산 결합하기](#연산-결합하기)
    - [thenApply or thenCompose](#thenApply-or-thenCompose)
    - [병렬처리](#병렬처리)
    - [비동기 메서드](#비동기-메서드)
    - [예외 처리](#예외-처리)
    - [Timeout](#timeout)
- [마무리](#마무리)

---

## 비동기 처리

비동기 처리는 **특정 작업이 다른 작업과 독립적으로 동작**하도록 하여 다음 단계의 작업이 이전 작업의 완료를 기다리지 않고 **동시에 실행**할 수 있도록 하거나, 특정 작업의 **완료를 기다리는 동안 다른 작업을 처리**할 수 있는 장점이 있습니다.

흔히 사용하는 방식은 **동기적 처리**로 **한 작업이 완료되기를 기다렸다가 다음 작업을 순차적으로 실행**하도록 구현하는 방식이고, 함께 알아볼 **비동기 처리**는 여러 작업이 동시에 실행될 수 있고, 다른 작업의 완료를 기다리지 않고 실행하여 시스템의 자원을 최대한 활용할 수 있는 방식입니다.  

비동기 처리를 통해 **성능 향상**, **시스템 활용도 증가**, **동시성 관리**, **자원 활용** 등의 장점을 누릴 수 있지만,<br/>
**복잡성 증가**, **디버깅의 어려움**, **가독성 감소**와 같은 단점을 함께 고려해야 할 필요가 있습니다.

## CompletableFuture

`java5`부터 [Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html) 인터페이스는 비동기 연산을 위해 추가되었지만, 몇 가지 문제점을 가지고 있었습니다.
- 여러 연산을 결합하기 어려운 문제
- 비동기 처리 중에 발생하는 예외를 처리하기 어려운 문제

...

이러한 `Future` 인터페이스의 문제를 개선한 `CompletableFuture` 클래스가 `java8`에 등장하게 되었습니다.<br/>
`CompletableFuture` 클래스는 java5에 추가된 [Future](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Future.html) 인터페이스와 [CompleteStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) 인터페이스를 구현하고 있습니다.
- `Future`: java5에서 **비동기 연산을 위해 추가**된 인터페이스
- `CompleteStage`: 여러 연산을 결합할 수 있도록 연산이 완료되면 다음 단계의 작업을 수행하거나 값을 연산하는 **비동기식 연산 단계를 제공**하는 인터페이스

[CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 클래스는 `Future` 인터페이스의 문제를 개선하기 위해 등장한 만큼 **여러 연산을 결합한 비동기 연산 처리**, **예외 처리** 등을 위한 50여 가지의 다양한 메서드을 제공하고 있습니다. (이 글에서는 50여 가지의 메서드를 모두 소개해 드리기에는 많은 양이다 보니 자주 사용될 수 있는 기본적이 메서드 위주로 예제와 함께 작성해 보았습니다.)<br/>
Future와 CompletableFuture가 무엇인지 대략 알게 되었으니, 둘의 차이를 간략하게 보고 넘어가면 좋을 것 같습니다.

**Future vs CompletableFuture**

| Future                             | CompletableFuture                   |
|------------------------------------|-------------------------------------|
| Blocking                           | Non-blocking                        |
| 여러 연산을 함께 연결하기 어려움                 | 여러 연산을 함께 연결                        |
| 여러 연산 결과를 결합하기 어려움                 | 여러 연산 결과를 결합                        |
| 연산 성공 여부만 확인할 수 있고 예외 처리의 어려움 | exceptionally(), handle()을 통한 예외 처리 |

...

이제 본격적으로 `CompletableFuture` 클래스와 함께 놀아볼 시간입니다.🧸

---

### CompletableFuture 인스턴스

먼저 `CompletableFuture` 클래스와 함께 놀기 위해 `CompletableFuture` 인스턴스가 필요하겠죠?!<br/>
`CompletableFuture` 클래스의 정적 메서드인 [supplyAsync()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#supplyAsync-java.util.function.Supplier-) 메서드를 통해 `CompletableFuture` 인스턴스를 생성할 수 있습니다.

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(ASYNC_POOL, supplier);
}
```

`Supplier`를 인수로 `supplyAsync()`를 호출하면 `ForkJoinPool.commonPool()`에서 전달된 `Supplier`를 비동기적으로 호출한 뒤 `CompleteableFuture` 인스턴스를 반환하게 됩니다.<br/>
여기서 참고로 [get()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#get--) 메서드를 호출하면 `Supplier`의 비동기 작업을 기다리다가 작업이 완료되면 결과를 반환하게 됩니다.

```java
@Test
@DisplayName("주문 정보를 조회하는 예시입니다.")
void supplyAsync() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo));
    
    assertEquals("iPhone 15", orderInfoFuture.get()); // CompletableFuture.get() 호출로 비동기 작업이 시작되고 2초 뒤 결과 반환
}

private String getOrderInfo(String orderNo) {
    try {
            Thread.sleep(2000); // orderInfoRepository.findByOrderNo(orderNo);
    } catch (InterruptedException e) {
            // ..
    }
    return "iPhone 15";
}
```

---

### 순차적으로 연산 처리하기

리턴 타입에 따라 적절한 메서드를 호출하여 순차적으로 비동기 연산을 처리할 수 있습니다.<br/>
여기서 "순차적으로 연산이 처리되는 거라면 비동기를 적용하지 않아도 되지 않을까?"라는 의문이 들었었는데, 단순히 순차적으로 연산이 수행되는 것이 아니라 **비동기로 처리된다는 것**을 간과하고 있었답니다..😅

복잡한 연산을 비동기로 순차적으로 처리해야 할 경우 유용하게 사용할 수 있을 것 같습니다.

```java
/**
 * 인자로 받은 Function을 사용하여 다음 연산 처리
 * Function의 반환 값을 가지고 있는 CompletableFuture<U> 반환
 */
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

/**
 * Consumer를 인자로 받고, 결과를 CompletableFuture<Void> 로 반환
 * get() 호출 시 연산을 처리하고 Void 유형의 인스턴스를 반환
 */
public CompletableFuture<Void> thenAccept(Consumer<? super T> action) {
    return uniAcceptStage(null, action);
}

/**
 * Runnable을 인자로 받고, 결과를 CompletableFuture<Void> 로 반환
 * get() 호출 없이 연산을 처리
 */
public CompletableFuture<Void> thenRun(Runnable action) {
    return uniRunStage(null, action);
}
```

.

**thenApply()**

먼저 thenApply() 메서드를 살펴볼까요?<br/>
[thenApply()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenApply-java.util.function.Function-) 메서드는 이전 단계의 결괏값을 인수로 사용하고, 전달한 `Function`을 다음 연산으로 사용합니다.<br/>
실행 결과로 `Function`의 반환 값을 가지고 있는 `CompletableFuture<U>`을 반환합니다.

```java
@Test
@DisplayName("주문 정보 조회가 완료되면 결제 정보를 조회하는 예시입니다.")
void thenApply() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo));
    CompletableFuture<String> paymentInfoFuture = orderInfoFuture.thenApply(s -> getPaymentInfo(s)); // 이전 연산 완료 후(2초 후) 다음 연산 처리
    
    assertEquals("iPhone 15 / 결제 정보: 신용카드 1,200,000원", paymentInfoFuture.get());
}

private String getPaymentInfo(String s) {
    try {
            Thread.sleep(2000);
    } catch (InterruptedException e) {
            // ..
    }
    return s + " / 결제 정보: 신용카드 1,200,000원";
}
```

**thenAccept()**

[thenAccept()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenAccept-java.util.function.Consumer-) 메서드는 `Consumer`를 인자로 받고, 결과를 `CompletableFuture<Void>`를 반환합니다.<br/>
리턴 타입이 없는 로직을 호출할 때 사용할 수 있습니다.

```java
@Test
@DisplayName("주문 정보 조회가 완료되면 주문 정보를 업데이트하는 예시입니다.")
void thenAccept() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo));
    CompletableFuture<Void> thenAccept = orderInfoFuture.thenAccept(s -> updateOrderInfo(s));
    
    thenAccept.get(); // Completed update: iPhone 15 출력
}

private void updateOrderInfo(String s) {
        System.out.println("Completed update: " + s);
}
```

**thenRun()**

[thenRun()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenRun-java.lang.Runnable-) 메서드는 `Runnable`를 인자로 받고, `thenAccept()` 메서드와 동일하게 결과를 `CompletableFuture<Void>`로 반환합니다.<br/>
댠, `thenAccept()` 메서드와 다르게 `CompletableFuture<Void>.get()` 호출 없이 연산이 실행됩니다.

```java
@Test
@DisplayName("주문 정보 조회가 완료되면 로그를 남기는 예시입니다.")
void thenRun() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo));
    orderInfoFuture.thenRun(() -> writeLog(orderNo));
}

private void writeLog(String orderNo) {
    System.out.println("Completed query: " + orderNo);
}
```

---

### 연산 결합하기

연산 결합하기는 java5 `Future` 인터페이스에서 처리하기 어려웠던 기능입니다.

**thenCompose()**

[thenCompose()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCompose-java.util.function.Function-) 메서드를 통해 `CompletableFuture` 인스턴스를 결합하여 연산을 처리할 수 있습니다.

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(null, fn);
}
```

`thenCompose()` 메서드는 두 개의 Future를 순차적으로 연결합니다.<br/>
이전 단계의 결과(CompletionStage)를 다음 CompletableFuture 안에서 사용하게 됩니다.

```java
    @Test
@DisplayName("주문 정보 조회가 완료되면 결제 정보를 조회하는 예시입니다.")
void thenCompose() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo))
            .thenCompose(s -> CompletableFuture.supplyAsync(() -> getPaymentInfo(s)));
    
    assertEquals("iPhone 15 / 결제 정보: 신용카드 1,200,000원", orderInfoFuture.get());
}
```

**thenCombine()**

[thenCombine()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenCombine-java.util.concurrent.CompletionStage-java.util.function.BiFunction-) 메서드로는 두 개의 독립적인 Future를 처리하고 두 결과를 결합하여 추가적인 작업을 수행할 수 있습니다.

```java
public <U,V> CompletableFuture<V> thenCombine(CompletionStage<? extends U> other, BiFunction<? super T,? super U,? extends V> fn) {
    return biApplyStage(null, other, fn);
}
```

`thenCombine()` 메서드는 `Future`와 Functional Interface인 `BiFunction`를 파라미터로 받아서 두 결과를 결합한 추가적인 처리가 가능합니다.

```java
@Test
@DisplayName("주문 정보 조회가 완료되면 옵션 정보를 조회하고 결과를 결합하는 예시입니다.")
void thenCombine() throws Exception {
    String orderNo = "1234567890";
    String optionNo = "112532";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo))
        .thenCombine(CompletableFuture.supplyAsync(() -> getProductInfo(optionNo)), (s1, s2) -> combineOrderProductInfo(s1, s2));
    
    assertEquals("iPhone 15. [옵션] AppleCare+", orderInfoFuture.get());
}

private String getProductInfo(String optionNo) {
    return "AppleCare+";
}

private String combineOrderProductInfo(String s1, String s2) {
    return s1 + ". [옵션] " + s2;
}
```

**thenAcceptBoth()**

`thenCombine()` 메서드와 유사하지만 [thenAcceptBoth()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#thenAcceptBoth-java.util.concurrent.CompletionStage-java.util.function.BiConsumer-) 메서드는 결괏값을 전달할 필요가 없으면 간단하게 사용할 수 있습니다. 

```java
public <U> CompletableFuture<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action) {
    return biAcceptStage(null, other, action);
}
```

`thenCombine()` 메서드는 데이터 조회 후 추가적인 정제 작업이 필요할 경우 사용할 수 있고, `thenAcceptBoth()` 메서드는 인서트, 업데이트 작업에 유용하게 사용할 수 있을 것 같습니다.

```java
@Test
@DisplayName("주문 정보 조회가 완료되면 배송 정보를 조회하고 주문 정보를 업데이트하는 예시입니다.")
void thenAcceptBoth() throws Exception {
    String orderNo = "1234567890";
    String shippingNo = "484567";
    CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo))
        .thenAcceptBoth(CompletableFuture.supplyAsync(() -> getShippingInfo(shippingNo)), (s1, s2) -> updateOrderInfo(s1, s2)); // Completed update: iPhone 15 -> 배송중 출력
}

private String getShippingInfo(String shippingNo) {
    return "배송중";
}

private void updateOrderInfo(String orderInfo, String shippingInfo) {
    System.out.println("Completed update: " + orderInfo + " -> " + shippingInfo);
}
```

---

### thenApply or thenCompose

보다 보니 `thenApply()` 메서드와 `thenCompose()` 메서드가 무슨 차이가 있는지 의문이 들기 시작했었는데요.<br/>
두 메서드의 특징을 비교해 보았습니다.


| thenApply()                   | thenCompose()                            |
|-------------------------------|------------------------------------------|
| 새로운 CompleteStage 반환          | 새로운 CompleteStage 반환                     |
| 이전 단계의 **결괏값**을 인수로 사용        | 이전 단계의 **CompletionStage**를 인수로 사용       |
| `CompletableFuture` 호출 결과를 반환 | 최종 결과가 포함된 `CompletableFuture`를 평면화하여 반환 |


**thenApply()**

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn) {
    return uniApplyStage(null, fn);
}

@Test
void thenApply() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo));
    CompletableFuture<String> paymentInfoFuture = orderInfoFuture.thenApply(s -> getPaymentInfo(s));
    
    assertEquals("iPhone 15 / 결제 정보: 신용카드 1,200,000원", paymentInfoFuture.get());
}
```

**thenCompose()**

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(null, fn);
}

@Test
void thenCompose() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfo(orderNo))
        .thenCompose(s -> CompletableFuture.supplyAsync(() -> getPaymentInfo(s)));
    
    assertEquals("iPhone 15 / 결제 정보: 신용카드 1,200,000원", orderInfoFuture.get());
}
```

요약하자면,<br/>
**각 CompletableFuture의 호출 결과가 필요할 경우**, `thenApply()` 메서드를 사용하는 것이 적합하고,<br/> 
각 CompletableFuture의 결과를 결합한 **최종 연산 결과만 필요한 경우**, `thenCompose()` 메서드를 사용하는 것이 적합할 것 같습니다.

---

### 병렬처리

[allOf()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#allOf-java.util.concurrent.CompletableFuture...-) 정적 메서드를 사용하면 여러 Future를 병렬로 처리할 수 있습니다.<br/>
`var-arg`로 제공되는 모든 `Future`의 처리를 대기하다가 모두 완료되면 `CompletableFuture<Void>`를 반환합니다.

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs) {
    return andTree(cfs, 0, cfs.length - 1);
}
```

다만, 병렬 처리는 가능하지만, 모든 `Future`의 결과를 결합한 결괏값을 반환할 수 없는 한계가 있습니다.<br/>
`get()` 메서드와 유사한 [join()](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#join--) 메서드를 활용하면 `allOf()` 메서드의 한계를 극복할 수 있지만, Future가 정상적으로 완료되지 않을 경우 확인되지 않은 예외가 발생할 수 있는 단점이 있다는 점을 고려해야 합니다.

```java
@Test
@DisplayName("주문 정보, 옵션 정보, 배송 정보를 병렬로 조회하고 결과를 결합하는 예시입니다.")
void allOf() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> {
        System.out.println("future1: " + Thread.currentThread().getName()); // ForkJoinPool.commonPool-worker-5
        return getOrderInfo(orderNo);
    });

    String optionNo = "112532";
    CompletableFuture<String> optionInfoFuture = CompletableFuture.supplyAsync(() -> {
        System.out.println("future1: " + Thread.currentThread().getName()); // ForkJoinPool.commonPool-worker-9
        return getProductInfo(optionNo);
    }).thenCompose(s -> CompletableFuture.supplyAsync(() -> addOptionTag(s)));
    
    String shippingNo = "484567";
    CompletableFuture<String> shippingInfoFuture = CompletableFuture.supplyAsync(() -> {
        System.out.println("future1: " + Thread.currentThread().getName()); // ForkJoinPool.commonPool-worker-19
        return getShippingInfo(shippingNo);
    }).thenCompose(s -> CompletableFuture.supplyAsync(() -> addShippingTag(s)));;
    
    CompletableFuture<Void> combinedFuture = CompletableFuture.allOf(orderInfoFuture, optionInfoFuture, shippingInfoFuture);
    combinedFuture.get();
    
    assertTrue(orderInfoFuture.isDone());
    assertTrue(optionInfoFuture.isDone());
    assertTrue(shippingInfoFuture.isDone());
    
    String combined = Stream.of(orderInfoFuture, optionInfoFuture, shippingInfoFuture)
        .map(CompletableFuture::join)
        .collect(Collectors.joining(" "));
    
    assertEquals("iPhone 15 . [옵션] AppleCare+ (배송중)", combined);
}
```

---

### 비동기 메서드

`CompletableFuture` 클래스가 제공하는 메서드를 보면 Async 접미사가 붙은 `supplyAsync`, `thenApplyAsync`와 같은 형태의 메서드를 발견할 수 있습니다. 이러한 메서드는 일반적으로 **다른 스레드를 사용하여 비동기 연산을 수행**하고, Async 접미사가 없는 메서드는 **현재 스레드를 사용하여 연산을 수행**하게 됩니다. 각 Future들을 비동기로 동작시키기 위해서 Async 접미사가 붙은 메서드들을 잘 활용해야 하겠죠?!😉<br/>
Async 접미사가 붙은 메서드를 자세히 들여다보면 다른 스레드를 사용하기 위해 Executor를 직접 제공하거나 기본 Executor를 사용할 수 있도록 선택권을 제공하고 있습니다.

먼저 Executor 인수가 없는 메서드를 보면 common pool 사용 여부에 따라 Executor를 얻어오게 됩니다.<br/>
아래 코드를 보면 `CommonPoolParallelism` 값을 기준으로 1보다 클 경우 `ForkJoinPool.commonPool()`에서 common pool 인스턴스를 획득하게 되고, `ForkJoinPool.commonPool()`이 병렬화를 지원할 수 없는 경우 `ThreadPerTaskExecutor` 인스턴스를 사용하게 됩니다.<br/>
Fork/Join Framework에 대한 자세한 내용은 [Guide to the Fork/Join Framework in Java](https://www.baeldung.com/java-fork-join)를 참고하면 좋을 것 같습니다.

```java
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn) {
    return uniApplyStage(defaultExecutor(), fn);
}

public Executor defaultExecutor() {
    return ASYNC_POOL;
}

private static final Executor ASYNC_POOL = USE_COMMON_POOL ? 
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();

private static final boolean USE_COMMON_POOL = (ForkJoinPool.getCommonPoolParallelism() > 1);
```

반대로, Executor 인수가 있는 메서드를 보면 전달된 Executor를 사용하여 비동기 연산 단계를 실행할 수 있습니다.

```java
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor) {
    return uniApplyStage(screenExecutor(executor), fn);
}

static Executor screenExecutor(Executor e) {
    if (!USE_COMMON_POOL && e == ForkJoinPool.commonPool())
        return ASYNC_POOL;
    if (e == null) throw new NullPointerException();
    return e;
}
```

제가 as-is, to-be 로직을 비동기로 호출하기 위해 비동기 메서들를 활용하게 되었는데요.<br/>
실제로는 더 복잡한 로직이 포함되어 있지만 아래 간단한 예시와 같이 asIs, toBe 로직을 비동기로 호출하고 두 Future 실행이 모두 완료되면 결과를 비교하도록 구현하였습니다.<br/>
두 Future 실행 시 사용되는 스레드 정보를 보면 서로 다른 스레드에서 작업을 처리하고 있는 것을 확인할 수 있습니다.

```java
@Test
void supplyAsync() throws Exception {
    CompletableFuture<String> asIsFuture = CompletableFuture.supplyAsync(() -> {
        System.out.println("asIsFuture: " + Thread.currentThread().getName()); // ForkJoinPool.commonPool-worker-23
        return getHello();
    });
    CompletableFuture<String> toBeFuture = CompletableFuture.supplyAsync(() -> {
        System.out.println("asIsFuture: " + Thread.currentThread().getName()); // ForkJoinPool.commonPool-worker-19
        return getHelloDirect();
    });
    
    String asIs = asIsFuture.get();
    String toBe = toBeFuture.get();
    
    assertTrue(toBe.equals(asIs));
}
```

---

### 예외 처리

**handle()**

[handle()](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletableFuture.html#handle-java.util.function.BiFunction-) 메서드는 예외를 잡는 대신 CompletableFuture 클래스를 사용하여 별도의 메서드에서 예외 처리가 가능합니다.<br/>
또한 연산 결과(성공적으로 완료된 경우)와 발생한 예외(정상적으로 완료되지 않은 경우)를 매개 변수로 받을 수 있습니다.

```java
public <U> CompletableFuture<U> handle(BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(null, fn);
}
```

`handle()` 메서드로 예외를 처리하려면 일반적인 방법과 유사하게 throw/catch 구문을 적용하게 됩니다.

```java
@Test
@DisplayName("주문 정보 조회 시 발생할 수 있는 예외를 다루는 예시입니다.")
void handle() throws Exception {
    String orderNo = "1234567890";
    // do something..
    
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> {
        if (orderNo == null) {
            throw new IllegalArgumentException("The orderNo must not be null!");
        }
        return getOrderInfo(orderNo);
    }).handle((s, t) -> { // s: Future 실행 완료 후 결괏값, t: Future 실행 중 발생한 예외 
        /**
        * 연산이 성공적으로 완료된 경우
        * s = iPhone 15
        * t = null
        *
        * 연산이 정상적으로 완료되지 않고 예외가 발생한 경우
        * s = null
        * t = java.util.concurrent.CompletionException: java.lang.IllegalArgumentException: The orderNo must not be null!
        */
        return t == null ? s : "Default value";
    });
    
    assertEquals("iPhone 15", orderInfoFuture.get());
}
```

**completeExceptionally()**

[completeExceptionally()](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletableFuture.html#completeExceptionally-java.lang.Throwable-) 메서드는 연산이 정상적으로 완료되지 않을 경우 예외를 정의하여 비동기 처리를 완료시킬 수 있습니다.

```java
public boolean completeExceptionally(Throwable ex) {
    if (ex == null) throw new NullPointerException();
    boolean triggered = internalComplete(new AltResult(ex));
    postComplete();
    return triggered;
}
```

특정 상황에 예외가 발생하도록 Future에 예외를 지정할 수 있고, 특정 상황에 맞는 동작을 하도록 지정할 수 있어서 동적으로 행동이나 예외를 지정해야 할 경우 사용될 수 있을 것 같습니다. 

```java
@Test
@DisplayName("Future가 특정 상황에 맞는 예외를 던지게 하거나 특정 형태로 Future가 완료되도록 지정하는 예시입니다.")
void completeExceptionally_exception() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = new CompletableFuture<>();
    
    if (orderNo == null) {
        orderInfoFuture.completeExceptionally(new IllegalArgumentException("The orderNo must not be null!"));
    }
    
    // do something..
    
    String shippingNo = null;
    if (shippingNo == null) {
        orderInfoFuture.completeExceptionally(new IllegalArgumentException("The ShippingNo must not be null!"));
    }
    
    if (orderNo != null && shippingNo != null) {
        orderInfoFuture.complete(getOrderInfo(orderNo));
    }
    
    ExecutionException executionException = Assertions.assertThrows(ExecutionException.class, () -> {
        orderInfoFuture.get();
    });
    Assertions.assertTrue(executionException.getMessage().contains("The ShippingNo must not be null!"));
}

@Test
void completeExceptionally_complete() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = new CompletableFuture<>();
    
    if (orderNo == null) {
        orderInfoFuture.completeExceptionally(new IllegalArgumentException("The orderNo must not be null!"));
    }
    
    // do something..
    
    String shippingNo = "484567";
    if (shippingNo == null) {
        orderInfoFuture.completeExceptionally(new IllegalArgumentException("The ShippingNo must not be null!"));
    }
    
    if (orderNo != null && shippingNo != null) {
        orderInfoFuture.complete(getOrderInfo(orderNo));
    }
    
    assertEquals("iPhone 15", orderInfoFuture.get());
}
```

---

### Timeout

[get(long timeout, TimeUnit unit)](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html#get-long-java.util.concurrent.TimeUnit-)

java8에서는 `CompletableFuture.get(long timeout, TimeUnit unit)` 메서드에서만 timeout 설정이 가능했었습니다.

```java
public T get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    long nanos = unit.toNanos(timeout);
    Object r;
    if ((r = result) == null)
        r = timedGet(nanos);
    return (T) reportGet(r);
}
```

`get()` 메서드 호출 시 전달한 timeout 설정보다 실행 시간이 더 오래 걸린다면 TimeoutException 예외를 발생시키게 됩니다.

```java
@Test
@DisplayName("주문 정보 조회 시 타입아웃이 발생하면 예외를 발생시키는 예시입니다.")
void get_timeout() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfoDelay(orderNo));
    
    Assertions.assertThrows(TimeoutException.class, () -> {
        orderInfoFuture.get(2000, TimeUnit.MILLISECONDS);
    });
}

private String getOrderInfoDelay(String orderNo) {
    try {
        Thread.sleep(4000);
    } catch (InterruptedException e) {
        // ..
    }
    return "iPhone 15";
}
```

...

java9부터는 timeout을 위한 `orTimeout()`, `completeOnTimeout()` 메서드가 추가되었습니다.

[orTimeout()](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletableFuture.html#orTimeout-long-java.util.concurrent.TimeUnit-)

```java
public CompletableFuture<T> orTimeout(long timeout, TimeUnit unit) {
    if (unit == null)
        throw new NullPointerException();
    if (result == null)
        whenComplete(new Canceller(Delayer.delay(new Timeout(this), timeout, unit)));
    return this;
}
```

지정된 시간까지 작업이 완료되지 않은 경우, ExecutionException 예외를 발생시키게 됩니다.

```java
@Test
@DisplayName("주문 정보 조회 시 타입아웃이 발생하면 예외를 발생시키는 예시입니다.")
void orTimeout() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfoDelay(orderNo))
        .orTimeout(2, TimeUnit.SECONDS);
    
    Assertions.assertThrows(ExecutionException.class, () -> {
        orderInfoFuture.get();
    });
}
```

.

[completeOnTimeout](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletableFuture.html#completeOnTimeout-T-long-java.util.concurrent.TimeUnit-)

```java
public CompletableFuture<T> completeOnTimeout(T value, long timeout, TimeUnit unit) {
    if (unit == null)
        throw new NullPointerException();
    if (result == null)
        whenComplete(new Canceller(Delayer.delay(
            new DelayedCompleter<T>(this, value),
            timeout, unit)));
    return this;
}
```

지정된 시간까지 작업이 완료되지 않은 경우, 지정된 기본값으로 Future를 완료시킵니다.

```java
@Test
@DisplayName("주문 정보 조회 시 타입아웃이 발생하면 기본값을 제공하는 예시입니다.")
void completeOnTimeout() throws Exception {
    String orderNo = "1234567890";
    CompletableFuture<String> orderInfoFuture = CompletableFuture.supplyAsync(() -> getOrderInfoDelay(orderNo))
            .completeOnTimeout("default value", 2, TimeUnit.SECONDS);
    
    String result = orderInfoFuture.get();
    assertEquals("default value", result);
}
```

---

## 마무리

비동기 기술에 대해 간접적으로만 들어왔었는데 따로 학습하며 직접 실무에 적용해 볼 수 있었던 유익한 시간이었습니다.<br/>
비동기 처리를 실무에 적용하기 위해 학습했던 내용들을 공유한 글이다 보니 잘못 알고 있던 부분이나 내용이 추가되면 좋을 것 같은 부분이 많을 것으로 생각합니다.<br/>
읽으시면서 궁금하신 사항이나 개선 사항이 보이신다면 언제든 아래 코멘트 부탁드립니다.<br/>
글을 읽어주신 모든 분께 감사드립니다. 🙇🏻‍

## Reference

- [java8 CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
- [java9 CompletableFuture](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/CompletableFuture.html)
- [java11 CompletableFuture](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)
- [Guide To CompletableFuture](https://www.baeldung.com/java-completablefuture)