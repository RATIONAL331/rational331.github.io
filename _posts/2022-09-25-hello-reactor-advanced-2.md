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

```java
class FluxJust {
	void just() {
		Flux<Integer> just1 = Flux.just(new Random().nextInt(1000),
		                                new Random().nextInt(1000),
		                                new Random().nextInt(1000),
		                                new Random().nextInt(1000),
		                                new Random().nextInt(1000));
		just1.subscribe(System.out::println);
		just1.subscribe(System.out::println);
		/**
		 995
		 959
		 432
		 188
		 476
		 995 <- [주목!]
		 959 <- [주목!]
		 432 <- [주목!]
		 188 <- [주목!]
		 476 <- [주목!]
		 */
	}
}

```

* defer()로 래핑하여 Cold Publisher로 전활할 수 있습니다. 이렇게 되면 초기화 시 값을 생성하더라도, 새 구독을 하면 초기화를 합니다.
* ```Flux.defer(() -> Flux.just(1, 2, 3))```

### share

* share() 연산자를 사용하면 Cold Publisher를 Hot Publisher로 쉽게 전환할 수 있습니다.

```java
class HotShare {
	void hotShare() {
		Flux<String> movieStream = Flux.fromStream(() -> getMovie())
		                               .delayElements(Duration.ofMillis(500))
		                               .share(); // convert cold to hot

		sleepSeconds(1);
		movieStream.subscribe(...); // 첫번째 구독자
		sleepSeconds(1);
		movieStream.subscribe(...); // 두번째 구독자는 첫번째 구독 후 1초 뒤 구독
		sleepSeconds(3);

		/**
		 * getMovie() <- [주목!] 한번만 호출되고 있습니다.
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub2; Received: Scene 3 <- notice (1.5초에 생성되는 Scene 3만 전달 받음)
		 * sub1; Completed
		 * sub2; Completed <- sub1, sub2 모두 completed 신호를 받고 종료되었습니다.
		 */

		Flux<String> movieStream2 = Flux.fromStream(() -> getMovie())
		                                .delayElements(Duration.ofMillis(500))
		                                .share(); // convert cold to hot

		sleepSeconds(1);
		movieStream2.subscribe(...); // 첫번째 구독자
		sleepSeconds(3);
		movieStream2.subscribe(...); // 두번째 구독자는 첫번째 구독 후 3초 뒤 구독
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub1; Completed
		 * getMovie() <- [주목!] 데이터 생성을 다시 시도합니다.
		 * sub2; Received: Scene 1
		 * sub2; Received: Scene 2
		 * sub2; Received: Scene 3
		 * sub2; Completed
		 */
	}

	private static Stream<String> getMovie() {
		System.out.println("getMovie()");
		return Stream.of("Scene 1", "Scene 2", "Scene 3");
	}
}
```

* 위에서 보는 예제에서 Scene 1, Scene 2, Scene 3이 500ms 간격으로 생성됩니다.
* sub1은 1초 후에 구독을 진행합니다. 이 때 처음 구독자에게는 Scene 1, Scene 2, Scene 3 모두가 전달됩니다.
* sub2는 sub1이 구독하고 나서 1초 후에 구독을 합니다. 이 때는 Scene 3만 전달됩니다.
* 만약에 sub2가 sub1이 completed 신호를 받고나서 구독을 한다면 데이터 생성부터 다시 시작하는 것을 볼 수 있습니다.

### refCount

* 구독자 수를 자동으로 추적하여 실행하게 할 수 있습니다.

```java
class HotRefCount() {
	void refCount() {
		Flux<String> movieStream = Flux.fromStream(() -> getMovie())
		                               .delayElements(Duration.ofMillis(500))
		                               .publish() // return ConnectableFlux
		                               .refCount(1);
		// publish().refCount(1) == share()

		movieStream.subscribe(...);
		sleepSeconds(1);
		movieStream.subscribe(...);
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub2; Received: Scene 3 <- notice
		 * sub1; Completed
		 * sub2; Completed
		 */

		Flux<String> movieStream2 = Flux.fromStream(() -> getMovie())
		                                .delayElements(Duration.ofMillis(500))
		                                .publish() // return ConnectableFlux
		                                .refCount(2); // 구독자수가 2가 되어야, publisher가 데이터를 방출합니다.

		movieStream2.subscribe(...);
		sleepSeconds(5);
		movieStream2.subscribe(...); // <- 데이터의 생성시점은 이곳임니다.
		movieStream2.subscribe(...); // 세번째 구독자도 데이터가 발행될 동안 같이 데이터를 받습니다.
		sleepSeconds(3);

		/**
		 * getMovie() // <- 5초 후에야 데이터를 생성하기 시작합니다.
		 * sub1; Received: Scene 1
		 * sub2; Received: Scene 1 // 두번째 구독자도 같이 Scene1을 받습니다.
		 * sub3; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub2; Received: Scene 2
		 * sub3; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub2; Received: Scene 3
		 * sub3; Received: Scene 3
		 * sub1; Completed
		 * sub2; Completed
		 */
	}

	static Stream<String> getMovie() {
		System.out.println("getMovie()");
		return Stream.of("Scene 1", "Scene 2", "Scene 3");
	}
}
```

* publish() 함수는 ConnectableFlux를 리턴합니다.
* ConnectableFlux는 데이터 생성 준비가 완료되는 대로 일부 구독자에게 전달하고
    * 각 구독자를 위해 중복된 데이터를 생성하지는 않으려고 합니다.
* refCount()는 특정 구독자 수가 충족되면 그 때 데이터 생성을 시작합니다.
* publisher().refCount(1)는 share()와 같습니다.
    * refCount()에 인자를 주지 않으면 refCount(1)과 같습니다.

```java
class HotCreate {
	void hotCreate() {
		Flux<Object> flux = Flux.create(fluxSink -> {
			                        System.out.println("created");
			                        for (int i = 0; i < 5; i++) {
				                        fluxSink.next(i);
			                        }
			                        fluxSink.complete();
		                        })
		                        .publish()
		                        .refCount(2);
		flux.subscribe(...);
		flux.subscribe(...); // <- 데이터의 생성 시점
		flux.subscribe(...);
		/**
		 * created
		 * sub1; Received: 0
		 * sub2; Received: 0
		 * sub1; Received: 1
		 * sub2; Received: 1
		 * sub1; Received: 2
		 * sub2; Received: 2
		 * sub1; Received: 3
		 * sub2; Received: 3
		 * sub1; Received: 4
		 * sub2; Received: 4
		 * sub1; Completed
		 * sub2; Completed
		 */ // 세번째 구독자는 데이터를 전혀 받지 못하였습니다.


		Flux<Object> flux2 = Flux.create(fluxSink -> {
			                         System.out.println("created");
			                         for (int i = 0; i < 5; i++) {
				                         fluxSink.next(i);
			                         }
			                         fluxSink.complete();
		                         })
		                         .publish()
		                         .refCount(2);
		flux2.subscribe(...);
		flux2.subscribe(...); // <- 데이터의 생성 시점
		flux2.subscribe(...);
		flux2.subscribe(...); // <- 데이터의 생성 시점
		/**
		 * created
		 * sub1; Received: 0
		 * sub2; Received: 0
		 * sub1; Received: 1
		 * sub2; Received: 1
		 * sub1; Received: 2
		 * sub2; Received: 2
		 * sub1; Received: 3
		 * sub2; Received: 3
		 * sub1; Received: 4
		 * sub2; Received: 4
		 * sub1; Completed
		 * sub2; Completed
		 * created
		 * sub3; Received: 0
		 * sub4; Received: 0
		 * sub3; Received: 1
		 * sub4; Received: 1
		 * sub3; Received: 2
		 * sub4; Received: 2
		 * sub3; Received: 3
		 * sub4; Received: 3
		 * sub3; Received: 4
		 * sub4; Received: 4
		 * sub3; Completed
		 * sub4; Completed
		 */
	}
}
```

* Flux.create로 직접 프로그래밍적으로 데이터를 방출하는 예제를 살펴봅시다.
* refCount(2)이기 때문에 두 개의 구독자가 구독을 해야지만 데이터 생성이 시작됩니다.
* 세번째 구독자는 구독을 하였지만 데이터를 전혀 받지 못하였습니다.
    * fromStream() 하고는 다른 모습입니다. (세번째 구독자는 중간 데이터를 받았습니다.)
* 네번째 구독자가 등장할 때 데이터 생성이 다시 시작되었습니다.
    * 앞에 두개의 구독이 모두 끝난 후 데이터 생성이 시작되었습니다.

### autoConnect

* autoConnect는 refCount와 유사하게 구독자 수를 자동으로 추적합니다.
* 다른 점을 아래 예제로 살펴봅시다.

```java
class HotAutoConnect {
	void hotAutoConnect() {
		Flux<String> movieStream = Flux.fromStream(() -> getMovie())
		                               .delayElements(Duration.ofMillis(500))
		                               .publish()
		                               .autoConnect(1); // <- refCount가 autoConnect로만 바뀌었습니다.

		sleepSeconds(1);
		movieStream.subscribe(...);
		sleepSeconds(1);
		System.out.println("join sub2");
		movieStream.subscribe(...);
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * join sub2
		 * sub1; Received: Scene 3
		 * sub2; Received: Scene 3
		 * sub1; Completed
		 * sub2; Completed
		 */ // refCount와는 다를 바가 없습니다.

		Flux<String> movieStream2 = Flux.fromStream(() -> getMovie())
		                                .delayElements(Duration.ofMillis(500))
		                                .publish()
		                                .autoConnect(1);

		sleepSeconds(1);
		movieStream2.subscribe(...);
		sleepSeconds(3); // <- [주목!] 데이터가 다 소진되고 나서 구독을 시작합니다.
		System.out.println("join sub2");
		movieStream2.subscribe(...);
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub1; Completed
		 * join sub2
		 */ // sub2는 아무런 데이터도 받지 못했습니다.

		Flux<String> movieStream3 = Flux.fromStream(() -> getMovie())
		                                .delayElements(Duration.ofMillis(500))
		                                .publish()
		                                .autoConnect(0); // 구독자가 없어도 곧장 데이터를 방출합니다. -> Real Hot Publisher

		sleepSeconds(1);
		movieStream3.subscribe(...);
		sleepSeconds(2);
		System.out.println("join sub2");
		movieStream3.subscribe(...);
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 2 // <- [주목!] 첫번째 구독자가 Scene 1을 받지 못했습니다.
		 * sub1; Received: Scene 3
		 * sub1; Completed
		 * join sub2
		 */

		private static Stream<String> getMovie () {
			System.out.println("getMovie()");
			return Stream.of("Scene 1", "Scene 2", "Scene 3");
		}
	}
}
```

* autoConnect()는 refCount()와 유사하게 구독자 수를 추적합니다.
* refCount()는 만약 데이터가 모두 소진되고 나서 재구독이 이루어진다면 다시 데이터를 생성합니다.
    * refCount(0)은 불가능합니다.
* autoConnect()는 데이터가 모두 소진되고 나서 재구독을 하여도 데이터를 생성하지 않습니다.
    * autoConnect(0)은 가능합니다.

### cache

* cache()는 데이터가 발행된 것을 기억하게 합니다.

```java
class HotCache() {
	void hotCache() {
		Flux<String> movieStream = Flux.fromStream(() -> getMovie())
		                               .delayElements(Duration.ofMillis(300))
		                               .cache(); // cache INTEGER.MAX_VALUE element

		movieStream.subscribe(...);
		sleepSeconds(3); // 3초 후에 두번째 구독이 시작됩니다.
		System.out.println("join sub2");
		movieStream.subscribe(...);
		sleepSeconds(1);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1 (300ms)
		 * sub1; Received: Scene 2 (300ms)
		 * sub1; Received: Scene 3 (300ms)
		 * sub1; Completed
		 * join sub2 // 이 다음 줄은 매우 빠르게 수행됩니다.
		 * sub2; Received: Scene 1 [ this process really quick! ] <- notice
		 * sub2; Received: Scene 2 [ this process really quick! ]
		 * sub2; Received: Scene 3 [ this process really quick! ]
		 * sub2; Completed
		 */

		Flux<String> movieStream2 = Flux.fromStream(() -> getMovie())
		                                .delayElements(Duration.ofMillis(300))
		                                .cache(2); // 마지막 2개의 원소만 기억

		movieStream2.subscribe(DefaultSubscriber.subscriber("sub1"));
		sleepSeconds(3);
		System.out.println("join sub2");
		movieStream2.subscribe(DefaultSubscriber.subscriber("sub2"));
		sleepSeconds(1);
		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub1; Completed
		 * join sub2
		 * sub2; Received: Scene 2 <- [주목!] Scene 1은 받지 못하고 2부터 빠르게 받았습니다.
		 * sub2; Received: Scene 3
		 * sub2; Completed
		 */

		private static Stream<String> getMovie () {
			System.out.println("getMovie()");
			return Stream.of("Scene 1", "Scene 2", "Scene 3");
		}

		Flux<Integer> flux = Flux.create(fluxSink -> {
			System.out.println("created");
			for (int i = 0; i < 5; i++) {
				fluxSink.next(i);
			}
			fluxSink.complete();
		});
		Flux<Integer> cache = flux.filter(i -> i > 1).cache(1);

		cache.subscribe(DefaultSubscriber.subscriber("sub1"));
		cache.subscribe(DefaultSubscriber.subscriber("sub2"));

		/**
		 * created
		 * sub1; Received: 2
		 * sub1; Received: 3
		 * sub1; Received: 4
		 * sub1; Completed
		 * sub2; Received: 4 <- [주목!] cache(1)이므로 마지막 원소 하나만 기억
		 * sub2; Completed
		 */
	}
}
```

* cache()는 publish().replay()와 같습니다.
* cache()에 인자가 들어오지 않으면 Integer.MAX_VALUE개의 원소를 기억합니다.
* cache()에 숫자뿐만 아니라 Duration 객체를 지정할 수 있습니다.
    * Duration 객체를 지정하면 해당 기간만큼 캐싱이 유지됩니다.

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