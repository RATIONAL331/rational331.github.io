---
title:  "리액터 살짝만 발을 더 깊이 - 2"
excerpt: "Hot & Cold 퍼블리셔, 쓰레드 스케쥴링, 배압 관리, 리액터 컨텍스트에 관해서"
category: "reactive"

last_modified_at: 2022-09-12T
---

# Hot & Cold

## 뜨거운 거, 차가운 거

* ![hello_reactor_03_01.webp](/assets/images/hello_reactor_03/hello_reactor_03_01.webp)
* 위에 거랑은 관계가 없긴 합니다.
* 퍼블리셔는 Cold Publisher, Hot Publisher로 나뉩니다.
* Cold Publisher는 구독자가 나타날 때 마다 해당 구독자에게 모든 데이터가 생성되는 방식으로 동작합니다.
* 또한 구독자가 없으면 데이터가 생성되지 않습니다.

## Cold Publisher

```java
class ColdPublishser {
	void coldPublisher() {
		Flux<String> movieStream = Flux.fromStream(() -> getMovie())
		                               .delayElements(Duration.ofMillis(500));

		movieStream.subscribe(DefaultSubscriber.subscriber("sub1"));
		sleepSeconds(1);
		movieStream.subscribe(DefaultSubscriber.subscriber("sub2"));
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * getMovie() <- [주목!]
		 * sub1; Received: Scene 3
		 * sub1; Completed
		 * sub2; Received: Scene 1
		 * sub2; Received: Scene 2
		 * sub2; Received: Scene 3
		 * sub2; Completed
		 */
	}

	static Stream<String> getMovie() {
		System.out.println("getMovie()");
		return Stream.of("Scene 1", "Scene 2", "Scene 3");
	}
}
```
* 다른 구독자가 나타나서 구독을 한다면 getMovie()가 다시 호출되어 총 두 번 호출 되는 것을 볼 수 있습니다.
* 지금까지 대부분 봤던 Publisher는 Cold Publisher입니다.

## Hot Publisher

* Hot Publisher의 데이터 생성은 구독자 존재 여부에 의존하지 않습니다.
* 따라서 첫번째 구독자가 구독을 하기 전에 원소를 만들어내기 시작할 수 있습니다.
* 또한 구독자가 구독할 때 이전에 생성된 값을 보내지 않고 새롭게 만들어진 값만 보낼 수 있습니다.
* just()는 빌드될 때 값이 한 번만 계산되고 새롭게 구독하면 다시 계산되지 않는 형태의 Hot Publisher를 생성합니다.
  * defer()로 래핑하여 Cold Publisher로 전활할 수 있습니다. 이렇게 되면 초기화 시 값을 생성하더라도, 새 구독을 하면 초기화를 합니다.
  * ```Flux.defer(() -> Flux.just(1, 2, 3))```

### share
```java

```

### refCount
```java

```

### autoConnect
```java

```

### cache
```java

```

# Schedulers
## 비동기적으로 데이터를 처리하는 방법
* 퍼블리셔 내부에서 데이터를 생성할 때 비동기적으로 생성할 수 도 있어야 합니다.
* 대부분 데이터를 생성할 때 main thread에서 생성하였습니다.
* 다른 워커 쓰레드에서 데이터를 생성하고 처리되게 하는 방법에 대해서 살펴봅시다.

## subscribeOn

## publishOn

# Backpressure

## Overflow Strategy

### Drop

### Latest

### Error

### Buffer

# Reactor Context

## ThreadLocal

## contextWrite

## contextUpdate

## Rate Limiting