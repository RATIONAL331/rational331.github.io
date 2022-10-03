---
title:  "리액터 살짝만 발을 더 깊이 - 2"
excerpt: "Hot & Cold 퍼블리셔, 쓰레드 스케쥴링에 관해서"
category: "reactive"

last_modified_at: 2022-10-03T
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
* 기본적으로 Hot을 취급하는 연산자가 아니면 대부분 Cold로 동작합니다.
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

		Flux<String> movieStream3 = Flux.fromStream(() -> getMovie())
		                                .delayElements(Duration.ofMillis(500))
		                                .publish() // return ConnectableFlux
		                                .refCount(2); // 구독자수가 2가 되어야, publisher가 데이터를 방출합니다.

		movieStream3.subscribe(...);
		sleepSeconds(5);
		movieStream3.subscribe(...); // <- 데이터의 생성시점은 이곳임니다.
		sleepSeconds(5);
		movieStream3.subscribe(...); // <- 다음 구독자가 올 때 까지 대기됩니다.
		sleepSeconds(3);

		/**
		 * getMovie()
		 * sub1; Received: Scene 1
		 * sub2; Received: Scene 1
		 * sub1; Received: Scene 2
		 * sub2; Received: Scene 2
		 * sub1; Received: Scene 3
		 * sub2; Received: Scene 3
		 * sub1; Completed
		 * sub2; Completed
		 */ // 세번째 구독자는 데이터를 받지 못하였습니다.

	}

	static Stream<String> getMovie() {
		System.out.println("getMovie()");
		return Stream.of("Scene 1", "Scene 2", "Scene 3");
	}
}
```

* publish() 함수는 ConnectableFlux를 리턴합니다.
* refCount()는 특정 구독자 수가 충족되면 그 때 데이터 생성을 시작합니다.
* refCount()에서 지정한 구독자 수를 충족하지 못한다면 데이터 생성이 시작되지 않습니다.
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
* 세번째 구독자는 바로 구독을 하였지만 데이터를 전혀 받지 못하였습니다.
    * main thread에서 두번째 구독까지 처리 진행하였고 세번째 구독자는 기다렸습니다.
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
    * cache(n) == publish().replay(n)
* cache()에 인자가 들어오지 않으면 Integer.MAX_VALUE개의(대략 21억) 원소를 기억합니다.
* cache()에 숫자뿐만 아니라 Duration 객체를 지정할 수 있습니다.
    * Duration 객체를 지정하면 해당 기간만큼 캐싱이 유지됩니다.

# Schedulers

## 비동기적으로 데이터를 처리하는 방법

* 퍼블리셔 내부에서 데이터를 생성할 때 비동기적으로 생성할 수 도 있어야 합니다.
* 대부분 데이터를 생성할 때 main thread에서 생성하였습니다.
* 다른 워커 쓰레드에서 데이터를 생성하고 처리되게 하는 방법에 대해서 살펴봅시다.

## 개요

* ![hello_reactor_03_02.png](/assets/images/hello_reactor_03/hello_reactor_03_02.png)
* 우리가 여태껏 봐왔던 모델입니다.
    * Subscriber가 Publisher에게 구독하여 Pushbliser가 Subscriber에게 데이터가 전달되는 모습입니다.
    * 이 때 실행되는 Thread를 Current Thread (현재 실행 쓰레드)라고 합니다.
    * 대개 현재 실행 쓰레드는 Main Thread입니다.

```java
class ThreadDemo {
	void threadDemo() {
		Flux<Object> flux = Flux.create(fluxSink -> {
			                        printThreadName("create");
			                        fluxSink.next(1); // 1을 방출하였습니다.
		                        })
		                        // doOnNext는 구독이 완료되고 데이터를 방출될 때 수행합니다.
		                        .doOnNext(i -> printThreadName("doOnNext " + i));
		flux.subscribe(v -> printThreadName("sub " + v));
		/**
		 * create		: Thread: main // 모두 main thread에서 실행됩니다.
		 * doOnNext 1		: Thread: main
		 * sub 1		: Thread: main
		 */

		// 구독을 다른 쓰레드에서 수행
		Runnable runnable = () -> flux.subscribe(v -> printThreadName("new thread " + v));
		for (int i = 0; i < 2; i++) {
			new Thread(runnable).start();
		}

		/**
		 * create		: Thread: Thread-1 // 각기 다른 쓰레드에서 수행
		 * create		: Thread: Thread-0
		 * doOnNext 1		: Thread: Thread-1
		 * doOnNext 1		: Thread: Thread-0
		 * new thread 1		: Thread: Thread-1
		 * new thread 1		: Thread: Thread-0
		 */

		Util.sleepSeconds(1);

	}

	private static void printThreadName(String msg) {
		System.out.println(msg + "\t\t: Thread: " + Thread.currentThread().getName());
	}
}
```

* 일반적으로 구독을 진행하게 되면 Publisher가 데이터를 만들어내고, Subscriber가 데이터를 받아서 처리하는 전 과정은 현재 실행 쓰레드에서 수행됩니다.
* 위 예제에서는 구독을 다른 쓰레드에서 수행하는 과정이 포함 되어있습니다.
    * 데이터 생성과 Subscriber가 받아서 처리하는 것까지 다른 쓰레드에서 수행되었습니다.
    * 이 때 데이터 생성과 Subscriber가 받아서 처리하는 것은 같은 쓰레드에서 수행되었습니다.
* 이제부터 구독 과정, 데이터 생성, 데이터 처리 과정을 각각 다른 쓰레드에서 수행할 수 있도록 조절해보겠습니다.

## Scheduler 종류

| Schedulers Method |             용례              |
|:-----------------:|:---------------------------:|
|  boundedElastic   |   네트워크 또는 IO등으로 시간이 소요될 때   | 
|     parallel      |       CPU 집약적인 일을 할 때       |
|      single       | 하나의 작업 처리를 위한 단 한개의 지정된 쓰레드 |
|     immediate     |           현재 쓰레드            |

* Schedulers는 병렬 데이터 처리가 아닙니다.
* 모든 연산들은 항상 순차적으로 수행됩니다.
* 데이터는 하나씩 쓰레드풀에 의해 처리됩니다.
* 그래서 Schedulers.parallel()는 병렬 실행을 의미하지 않습니다.
    * CPU 집약적 일을 위한 쓰레드 풀입니다.
* ![hello_reactor_03_05.png](/assets/images/hello_reactor_03/hello_reactor_03_05.png)
* 왼쪽의 그림은 데이터가 병렬 처리되는 것을 보여주고 있습니다.
* 오른쪽 그림은 여러 구독자가 있을 때 각각의 쓰레드 풀에서 데이터를 처리하는 것을 보여주고 있습니다.
* 스케쥴러에서 parallel은 왼쪽처럼 데이터를 병렬처리하는 것이 아닙니다.
    * 여러개의 구독자가 있을 때 해당 처리를 쓰레드 풀에서 처리하게 할지 결정하게 하는 것 입니다.

## subscribeOn

* ![hello_reactor_03_03.png](/assets/images/hello_reactor_03/hello_reactor_03_03.png)
* SubscribeOn은 업스트림(구독할 때) 쓰레드를 조절할 수 있습니다.
    * 또한 해당 메서드는 런타임에서 데이터가 전달되는(다운트스림) 쓰레드도 조절이 됩니다.
* 맨 처음 조립단계가 진행됩니다.
* 조립 단계가 종료된 후 구독 단계가 수행됩니다.
* 구독은 아래에서 위로 진행됩니다.
    * Sub이 Op를 구독, Op가 다른 Op를 구독, Op가 Pub을 구독
    * 이 떄 subscribeOn을 만나면 해당 부분에서 구독 단계를 다른 쓰레드에서 수행하도록 조절합니다.

```java
class SubscribeOn() {
	void subscribeOnDemo() {
		Flux<Object> flux = Flux.create(fluxSink -> {
			                        printThreadName("create");
			                        fluxSink.next(1);
		                        })
		                        // doOnNext는 구독이 완료된 후 데이터가 전달될 때 실행됩니다.
		                        .doOnNext(i -> printThreadName("doOnNext " + i));

		// doFirst는 구독 단계가 시작할 때 실행됩니다.
		flux.doFirst(() -> printThreadName("doFirst1"))
		    .subscribeOn(Schedulers.boundedElastic()) // 이 위로 진행되는 구독 단계, 데이터 생성, 전달은 모두 다른 쓰레드에서 수행됩니다.
		    .doFirst(() -> printThreadName("doFirst2"))
		    .doOnNext(i -> printThreadName("doOnNext2 " + i))
		    .subscribe(v -> printThreadName("sub " + v));

		/**
		 * doFirst2		: Thread: main // doFirst는 역순으로 수행됩니다. (upstream)
		 * doFirst1		: Thread: boundedElastic-1
		 * create		: Thread: boundedElastic-1
		 * doOnNext 1		: Thread: boundedElastic-1
		 * doOnNext2 1		: Thread: boundedElastic-1 // 데이터가 방출된 쓰레드는 boundedElastic-1입니다.
		 * sub 1		: Thread: boundedElastic-1 // 최종적으로 구독이 수행되는 쓰레드는 boundedElastic-1입니다.
		 */

		Runnable runnable = () -> flux.doFirst(() -> printThreadName("doFirst1"))
		                              .subscribeOn(Schedulers.boundedElastic())
		                              .doFirst(() -> printThreadName("doFirst2"))
		                              .subscribe(v -> printThreadName("sub " + v));
		for (int i = 0; i < 2; i++) {
			new Thread(runnable).start();
		}

		/**
		 * doFirst2		: Thread: Thread-1
		 * doFirst2		: Thread: Thread-0
		 * doFirst1		: Thread: boundedElastic-1 <- 각기 다른 boundElastic 쓰레드에서 수행되는 것을 관찰할 수 있습니다.
		 * doFirst1		: Thread: boundedElastic-2
		 * create		: Thread: boundedElastic-2
		 * create		: Thread: boundedElastic-1
		 * doOnNext 1		: Thread: boundedElastic-1
		 * doOnNext 1		: Thread: boundedElastic-2
		 * sub 1		: Thread: boundedElastic-1
		 * sub 1		: Thread: boundedElastic-2
		 */
	}

	private static void printThreadName(String msg) {
		System.out.println(msg + "\t\t: Thread: " + Thread.currentThread().getName());
	}
}
```

* doFirst() 함수는 구독 단계가 시작할 때 수행하는 메서드입니다.
    * 조립 단계를 다시 한번 생각해보면 가장 마지막에 래핑된 Publisher가 노출됩니다.
    * 그래서 가장 마지막에 있는 doFirst()는 구독 단계가 시작할 때 가장 먼저 수행됩니다.
* subscribeOn() 위와 아래를 관찰해봅시다.
    * subscribeOn() 위에 있는 doFirst()는 boundedElastic-1 쓰레드에서 수행됩니다.
    * subscribeOn() 아래에 있는 doFirst()는 main 쓰레드에서 수행됩니다.
    * subscribeOn()은 위에 작성되어있는 체이닝 부터 데이터가 방출되는 쓰레드를 조절하게 됩니다.
* 수행되는 쓰레드가 각기 다를 때 각기 다른 boundElastic 쓰레드에서 수행되는 것을 관찰할 수 있습니다.

### multi-subscribeOn

```java
class MultiSubscribeOn {
	void multiSubscribeOn() {
		Flux<Object> flux2 = Flux.create(fluxSink -> {
			                         printThreadName("create");
			                         fluxSink.next(1);
		                         })
		                         // create, next(1), 데이터가 전달되는 과정은 parallel 쓰레드에서 수행됩니다.
		                         .subscribeOn(Schedulers.newParallel("newParallel")) // <- notice
		                         .doOnNext(i -> printThreadName("doOnNext " + i));

		flux2.doOnNext(i -> printThreadName("doOnNext#1 " + i))
		     .doFirst(() -> printThreadName("doFirst1"))
		     .subscribeOn(Schedulers.boundedElastic())
		     .doOnNext(i -> printThreadName("doOnNext#2 " + i))
		     .doFirst(() -> printThreadName("doFirst2"))
		     .doOnNext(i -> printThreadName("doOnNext#3 " + i))
		     .subscribe(v -> printThreadName("sub " + v));

		/**
		 * doFirst2		: Thread: main
		 * doFirst1		: Thread: boundedElastic-1
		 * create		: Thread: newParallel-1 // subscribeOn이 다시 작성되면 해당 쓰레드에서 수행되도록 재조절 합니다.
		 * doOnNext 1		: Thread: newParallel-1 // 마지막이 parallel 이므로 데이터가 전달되는 과정은 모두 parallel에서 수행됩니다.
		 * doOnNext#1 1		: Thread: newParallel-1
		 * doOnNext#2 1		: Thread: newParallel-1
		 * doOnNext#3 1		: Thread: newParallel-1
		 * sub 1		: Thread: newParallel-1
		 */

		Flux<Object> flux3 = Flux.create(fluxSink -> {
			                         for (int i = 0; i < 5; i++) {
				                         fluxSink.next(i);
			                         }
			                         fluxSink.complete();
		                         })
		                         .subscribeOn(Schedulers.boundedElastic());
		// parallel로 수행하려 했는데 boundedElastic에 의해 덮였습니다.
		flux3.subscribeOn(Schedulers.parallel())
		     .doOnNext(i -> printThreadName("doOnNext#1 " + i))
		     .map(i -> i + "a")
		     .doOnNext(i -> printThreadName("doOnNext#2 " + i))
		     .subscribe(v -> printThreadName("sub " + v));

		/**
		 * doOnNext#1 0		: Thread: boundedElastic-1
		 * doOnNext#2 0a		: Thread: boundedElastic-1
		 * sub 0a		: Thread: boundedElastic-1
		 * doOnNext#1 1		: Thread: boundedElastic-1
		 * doOnNext#2 1a		: Thread: boundedElastic-1
		 * sub 1a		: Thread: boundedElastic-1
		 * doOnNext#1 2		: Thread: boundedElastic-1
		 * doOnNext#2 2a		: Thread: boundedElastic-1
		 * sub 2a		: Thread: boundedElastic-1
		 * doOnNext#1 3		: Thread: boundedElastic-1
		 * doOnNext#2 3a		: Thread: boundedElastic-1
		 * sub 3a		: Thread: boundedElastic-1
		 * doOnNext#1 4		: Thread: boundedElastic-1
		 * doOnNext#2 4a		: Thread: boundedElastic-1
		 * sub 4a		: Thread: boundedElastic-1
		 */
	}

	private static void printThreadName(String msg) {
		System.out.println(msg + "\t\t: Thread: " + Thread.currentThread().getName());
	}
}
```

* subscribeOn()이 다시 작성된다면 구독 단계에서 수행되는 쓰레드를 다시 재조절할 수 있습니다.
* 각 스케쥴러에 의해 수행되는 쓰레드가 달라지기 전까지 하나의 쓰레드에서 동일하게 수행되는 것도 주목하시기 바랍니다.
    * 한번 boundElastic-1에서 수행되었다면 subscribeOn, publishOn등으로 바꾸지 않는 이상 계속 boundElastic-1에서 수행됩니다.

## publishOn

* ![hello_reactor_03_04.png](/assets/images/hello_reactor_03/hello_reactor_03_04.png)
* publishOn()은 다운스트림(데이터 전달) 쓰레드를 조절할 수 있습니다.
    * publishOn()은 다운스트림만 조절됩니다. 업스트림에는 영향을 미치지 못합니다.

```java
class PublishOn() {
	void publishOnDemo() {
		Flux<Object> flux = Flux.create(fluxSink -> {
			                        printThreadName("create");
			                        for (int i = 0; i < 5; i++) {
				                        fluxSink.next(i);
			                        }
			                        fluxSink.complete();
		                        })
		                        .doOnNext(i -> printThreadName("main next " + i));

		flux.publishOn(Schedulers.boundedElastic()) // for downstream
		    .doOnNext(i -> printThreadName("bound next " + i))
		    .subscribe(v -> printThreadName("sub " + v));

		/**
		 * create		: Thread: main
		 * main next 0		: Thread: main
		 * main next 1		: Thread: main
		 * main next 2		: Thread: main
		 * main next 3		: Thread: main
		 * main next 4		: Thread: main
		 * bound next 0		: Thread: boundedElastic-1
		 * sub 0		: Thread: boundedElastic-1 <- notice this is boundedElastic thread
		 * bound next 1		: Thread: boundedElastic-1
		 * sub 1		: Thread: boundedElastic-1
		 * bound next 2		: Thread: boundedElastic-1
		 * sub 2		: Thread: boundedElastic-1
		 * bound next 3		: Thread: boundedElastic-1
		 * sub 3		: Thread: boundedElastic-1
		 * bound next 4		: Thread: boundedElastic-1
		 * sub 4		: Thread: boundedElastic-1
		 */

		Flux<Object> flux2 = Flux.create(fluxSink -> {
			                         printThreadName("create");
			                         for (int i = 0; i < 5; i++) {
				                         fluxSink.next(i);
			                         }
			                         fluxSink.complete();
		                         })
		                         .doOnNext(i -> printThreadName("main next " + i));

		flux2.publishOn(Schedulers.boundedElastic()) // downstream
		     .doOnNext(i -> printThreadName("bound next " + i))
		     .publishOn(Schedulers.parallel())
		     .doOnNext(i -> printThreadName("parallel next " + i))
		     .subscribe(v -> printThreadName("sub " + v));

		/**
		 * create		: Thread: main
		 * main next 0		: Thread: main
		 * main next 1		: Thread: main
		 * main next 2		: Thread: main
		 * main next 3		: Thread: main
		 * bound next 0		: Thread: boundedElastic-1
		 * main next 4		: Thread: main
		 * bound next 1		: Thread: boundedElastic-1
		 * bound next 2		: Thread: boundedElastic-1
		 * bound next 3		: Thread: boundedElastic-1
		 * parallel next 0		: Thread: parallel-1
		 * bound next 4		: Thread: boundedElastic-1
		 * sub 0		: Thread: parallel-1 <- notice this is parallel thread
		 * parallel next 1		: Thread: parallel-1
		 * sub 1		: Thread: parallel-1
		 * parallel next 2		: Thread: parallel-1
		 * sub 2		: Thread: parallel-1
		 * parallel next 3		: Thread: parallel-1
		 * sub 3		: Thread: parallel-1
		 * parallel next 4		: Thread: parallel-1
		 * sub 4		: Thread: parallel-1
		 */
	}

	private static void printThreadName(String msg) {
		System.out.println(msg + "\t\t: Thread: " + Thread.currentThread().getName());
	}
}
```

* publishOn()은 subscribeOn()보다 명확합니다.
* publishOn() 그 다음 체이닝에 대해서 데이터 처리를 어떻게 처리할 지 명시할 수 있습니다.
* subscribeOn()과 마찬가지로 publishOn()도 여러번 쓰게 되면 해당 다음 체이닝부터 처리할 쓰레드가 전환됩니니다.

### publishOn & SubscribeOn

```java
class PubOnSubOn {
	void pubOnSubOn() {
		Flux<Object> flux = Flux.create(fluxSink -> {
			                        printThreadName("create");
			                        for (int i = 0; i < 5; i++) {
				                        fluxSink.next(i);
			                        }
			                        fluxSink.complete();
		                        })
		                        .doOnNext(i -> printThreadName("next#1 " + i)); // boundedElastic

		flux.publishOn(Schedulers.parallel()) // downstream
		    .doOnNext(i -> printThreadName("next#2 " + i))
		    .subscribeOn(Schedulers.boundedElastic()) // upstream
		    .subscribe(v -> printThreadName("sub " + v));

		/**
		 * create		: Thread: boundedElastic-1
		 * next#1 0		: Thread: boundedElastic-1
		 * next#1 1		: Thread: boundedElastic-1
		 * next#1 2		: Thread: boundedElastic-1
		 * next#1 3		: Thread: boundedElastic-1
		 * next#2 0		: Thread: parallel-1
		 * next#1 4		: Thread: boundedElastic-1
		 * sub 0		: Thread: parallel-1
		 * next#2 1		: Thread: parallel-1
		 * sub 1		: Thread: parallel-1
		 * next#2 2		: Thread: parallel-1
		 * sub 2		: Thread: parallel-1
		 * next#2 3		: Thread: parallel-1
		 * sub 3		: Thread: parallel-1
		 * next#2 4		: Thread: parallel-1
		 * sub 4		: Thread: parallel-1
		 */
	}

	private static void printThreadName(String msg) {
		System.out.println(msg + "\t\t: Thread: " + Thread.currentThread().getName());
	}
}
```

* publishOn()과 subscribeOn()을 같이 사용하였을 때 어떻게 동작하는지 확인해봅니다.
* 참고로 doOnNext는 데이터가 전달될 때 수행되므로 downstream 영역입니다.

1. 우선 조립단계가 끝나고나서 구독단계에 돌입할 때 subscribeOn()이 적용됩니다.
2. 따라서 create()는 boundedElastic에서 수행됩니다.
3. 그 다음 바로 아래 doOnNext도 boundedElastic에서 수행됩니다.
4. 이제 publishOn()으로 쓰레드를 parallel로 전환합니다.
5. 그 다음 doOnNext도 parallel에서 수행됩니다.
6. 마지막으로 subscribe()는 parallel에서 수행됩니다.

## parallel

* 논의할 내용은 Schedulers.parallel()이 아닙니다.
* parallel()은 데이터를 병렬로 처리하는 연산자입니다.
* 해당 연산자는 하위 스트림에 대하여 흐름 분할, 분할된 흐름 간 균형 조정을 합니다.

```java
class Parallel {
	void parallel() {
		Flux<Integer> flux = Flux.range(1, 3);

		flux.parallel() // ParallelFlux 
		    .doOnNext(i -> printThreadName("next " + i))
		    .subscribe(v -> printThreadName("sub " + v));
		/**
		 * next 1		: Thread: main
		 * sub 1		: Thread: main
		 * next 2		: Thread: main
		 * sub 2		: Thread: main
		 * next 3		: Thread: main
		 * sub 3		: Thread: main
		 */

		Flux<Integer> flux2 = Flux.range(1, 3);

		flux2.parallel() // ParallelFlux 
		     .runOn(Schedulers.parallel()) // 각각의 Flux가 동작할 쓰레드를 지정
		     .doOnNext(i -> printThreadName("next " + i))
		     .subscribe(v -> printThreadName("sub " + v));
		/**
		 * next 1		: Thread: parallel-1
		 * next 2		: Thread: parallel-2
		 * next 3		: Thread: parallel-3
		 * sub 2		: Thread: parallel-2
		 * sub 1		: Thread: parallel-1
		 * sub 3		: Thread: parallel-3
		 */

		Flux<Integer> flux3 = Flux.range(1, 10);
		flux3.parallel(2) // 병렬 갯수를 지정할 수 있습니다.
		     .runOn(Schedulers.boundedElastic())
		     .doOnNext(i -> printThreadName("next " + i))
		     .subscribe(v -> printThreadName("sub " + v));
		/**
		 * next 1		: Thread: boundedElastic-10 // 2개로만 병렬 처리하고 있는 모습
		 * next 2		: Thread: boundedElastic-7
		 * sub 1		: Thread: boundedElastic-10
		 * sub 2		: Thread: boundedElastic-7
		 * next 4		: Thread: boundedElastic-7
		 * sub 4		: Thread: boundedElastic-7
		 * next 3		: Thread: boundedElastic-10
		 * next 6		: Thread: boundedElastic-7
		 * sub 6		: Thread: boundedElastic-7
		 * sub 3		: Thread: boundedElastic-10
		 * next 8		: Thread: boundedElastic-7
		 * sub 8		: Thread: boundedElastic-7
		 * next 10		: Thread: boundedElastic-7
		 * next 5		: Thread: boundedElastic-10
		 * sub 10		: Thread: boundedElastic-7
		 * sub 5		: Thread: boundedElastic-10
		 * next 7		: Thread: boundedElastic-10
		 * sub 7		: Thread: boundedElastic-10
		 * next 9		: Thread: boundedElastic-10
		 * sub 9		: Thread: boundedElastic-10
		 */

		Flux<Integer> flux4 = Flux.range(1, 10);
		flux4.parallel()
		     .runOn(Schedulers.boundedElastic())
		     // doOnNext는 여러개의 스레드에서 수행
		     .doOnNext(i -> printThreadName("next " + i))
		     .sequential()
		     // subscribe는 next가 이루어진 대로 하나의 스레드에서 수행
		     .subscribe(v -> printThreadName("sub " + v));

		/**
		 * next 1		: Thread: boundedElastic-3
		 * next 10		: Thread: boundedElastic-12
		 * next 8		: Thread: boundedElastic-6
		 * sub 1		: Thread: boundedElastic-3
		 * sub 8		: Thread: boundedElastic-3
		 * sub 10		: Thread: boundedElastic-3
		 * next 7		: Thread: boundedElastic-5
		 * next 3		: Thread: boundedElastic-8
		 * next 6		: Thread: boundedElastic-2
		 * next 2		: Thread: boundedElastic-9
		 * next 5		: Thread: boundedElastic-7
		 * next 9		: Thread: boundedElastic-1
		 * next 4		: Thread: boundedElastic-4
		 * sub 7		: Thread: boundedElastic-5
		 * sub 2		: Thread: boundedElastic-5
		 * sub 3		: Thread: boundedElastic-5
		 * sub 4		: Thread: boundedElastic-5
		 * sub 5		: Thread: boundedElastic-5
		 * sub 6		: Thread: boundedElastic-5
		 * sub 9		: Thread: boundedElastic-5
		 */
	}
}
```

* parallel()을 호출하면 ParallelFlux를 반환합니다.
* 그 다음 runOn()을 호출하여 publishOn을 내부 Flux에 적용할 수 있습니다.
* sequential()를 호출하면 ParallelFlux를 Flux로 변환합니다.

## interval

```java
class Interval {
	void interval() {
		Flux.interval(Duration.ofMillis(100)) // <- basic return parallel /* return interval(period, Schedulers.parallel()); */
		    .doFirst(() -> printThreadName("doFirst#1"))
		    .doOnNext(i -> printThreadName("next#1" + i)) // execution in parallel because Flux#interval execute in parallel
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> printThreadName("next#2" + i))
		    .doFirst(() -> printThreadName("doFirst#2"))
		    .subscribeOn(Schedulers.boundedElastic())
		    .subscribe(v -> printThreadName("sub " + v));

		/**
		 * doFirst#2		: Thread: boundedElastic-1
		 * doFirst#1		: Thread: boundedElastic-1
		 * next#1 0		: Thread: parallel-1 <- Flux.interval은 기본적으로 parallel에서 수행합니다.
		 * next#2 0		: Thread: boundedElastic-2 <- publishOn으로 boundedElastic로 변경
		 * sub 0		: Thread: boundedElastic-2
		 * next#1 1		: Thread: parallel-1 <- Flux.interval은 기본적으로 parallel에서 수행합니다. 
		 * next#2 1		: Thread: boundedElastic-2
		 * sub 1		: Thread: boundedElastic-2
		 * next#1 2		: Thread: parallel-1 <- Flux.interval은 기본적으로 parallel에서 수행합니다.
		 * next#2 2		: Thread: boundedElastic-2
		 * sub 2		: Thread: boundedElastic-2
		 * next#1 3		: Thread: parallel-1 <- Flux.interval은 기본적으로 parallel에서 수행합니다.
		 * next#2 3		: Thread: boundedElastic-2
		 * sub 3		: Thread: boundedElastic-2
		 */
	}
}
```

* interval은 0부터 주기적으로 1씩 증가하는 데이터를 발행하는 Flux를 생성합니다.
* 해당 Flux는 기본적으로 Scheduler.parallel()에서 동작합니다.
    * return interval(period, Schedulers.parallel());