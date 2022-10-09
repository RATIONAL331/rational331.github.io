---
title:  "리액터 살짝만 발을 더 깊이 - 3"
excerpt: "배압 관리, 리액터 컨텍스트"
category: "reactive"

last_modified_at: 2022-10-09T
---

# Backpressure

* ![hello_reactor_04_01.png](/assets/images/hello_reactor_04/hello_reactor_04_01.png)
* 위 그림은 Subscriber가 처리할 수 있는 속도보다 Publisher가 더 빠르게 데이터를 전달하는 상황입니다.
* Publisher는 Subscriber에게 모두 데이터를 전달할 수는 있을까요?
* 모두 전달하게 하려면 어떻게 해야할까요?
    * 무제한 큐를 활용해야 할 것이지만 현실적으로 이러한 방법은 활용할 수 없습니다.
* 모두 전달할 수 없다면 어떠한 방식으로 처리해야할까요?

## Overflow Strategy

```java
class BackpressureExample {
	void backpressure() {
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500; i++) {
				    fluxSink.next(i); // 500개의 데이터를 아주 빠르게 전달
				    System.out.println("emit " + i);
			    }
			    fluxSink.complete();
		    })
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10)) // 느리게 소비
		    .subscribe(...);

		sleepSeconds(5);

		/**
		 * emit 0
		 * emit 1
		 * ...
		 * emit 499
		 * subscribe; Received: 0
		 * subscribe; Received: 1
		 * ...
		 * subscribe; Received: 499
		 * subscribe; Completed // 느리게 소비하였지만 500개 모두 전달 받았습니다.
		 * // 참고로 전달하는 숫자가 많이 커지게 되면 구독자도 모두 받지 못하게 됩니다.
		 */
	}
}
```

* 기본적으로 리액터는 위와 같은 상황에서 Buffer 전략을 기본적으로 사용하게 됩니다.
* Buffer 전략은 Subscriber가 데이터를 느리게 소비할 동안 다음 전달할 내용을 버퍼(메모리)에 저장해둡니다.
    * 이 때 오버플로우가 발생하였을 때 기본 전략은 최신 발행된 데이터를 버립니다.
* Backpressure 전략은 Buffer만 있지 않고 다른 것도 존재합니다.
    * https://itsallbinary.com/reactor-basics-with-example-backpressure-overflow-drop-error-latest-ignore-buffer-good-for-beginners/

|   전략   |            행동            |
|:------:|:------------------------:|
| Buffer |    사이즈가 정해진 큐에 저장합니다.    | 
|  Drop  |   큐가 가득차면 새로운 것을 버립니다.   |
| Latest |   큐가 가득차면 오래된 것을 버립니다.   |
| Error  | 큐가 가득차면 다운스트림에 에러를 보냅니다. |

### Drop

```java
class BackpressureDropDefaultExample {
	void dropDefault() {
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500; i++) {
				    fluxSink.next(i); // 500개의 데이터를 아주 빠르게 전달
				    System.out.println("emit " + i);
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureDrop() // <- [주목!]
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10)) // 느리게 소비
		    .subscribe(...);

		sleepSeconds(5);

		/**
		 * emit 0
		 * emit 1
		 * ...
		 * emit 499
		 * subscribe; Received: 0
		 * subscribe; Received: 1
		 * ...
		 * subscribe; Received: 255
		 * subscribe; Completed // 느리게 소비하였고 256개만 전달 받았습니다.
		 */
	}
}
```

* onBackpressureLatest()는 기존에 있는 값을 유지하려고합니다.
    * 데이터가 전달되면 새롭게 전달된 데이터를 버리고 기존 데이터를 유지합니다.
* 위의 예제를 확인해보면 총 500개의 데이터를 방출하였고 256개의 데이터만 전달 받았습니다.
* 어느 환경에서든 항상 256개만 전달 받으며 해당 설정은 다음 파일에 설정되어 있습니다.
    * reactor.util.concurrent.Queues#SMALL_BUFFER_SIZE
    * 위에서 기본값으로 256이 설정되어있습니다.

```java
class BackpressureDropExample {
	void drop() {
		System.setProperty("reactor.bufferSize.small", "16"); // <- [주목!] 시스템 설정 변경
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500; i++) {
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1); // <- 500개의 데이터를 Subscriber보다 10배 빠르게 데이터 전달
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureDrop()
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);

		sleepSeconds(5);

		/**
		 * emit 0
		 * emit 1
		 * ...
		 * emit 8
		 * subscribe; Received: 0
		 * emit 9
		 * ...
		 * emit 16
		 * subscribe; Received: 1
		 * emit 17
		 * ...
		 * emit 26
		 * subscribe; Received: 2
		 * emit 27
		 * ...
		 * emit 113
		 * emit 114
		 * subscribe; Received: 11 <- 버퍼 사이즈(16개)의 75%(12개)가 전달될 때 [주목!]
		 * emit 115 <- 75%가 소비되고 그 다음 전달된 데이터부터 75%가 큐에 채워집니다.
		 * emit 116
		 * ...
		 * emit 150
		 * emit 151
		 * subscribe; Received: 15 <- 버퍼 마지막에 들어있는 값
		 * emit 152
		 * ...
		 * emit 161
		 * subscribe; Received: 115 <- [주목!] 15가 전달되고 그 다음은 115가 오는 것을 전달되는 것을 확인
		 * emit 162
		 * emit 163
		 * emit 164
		 * ...
		 * subscribe; Received: 116
		 * emit ...
		 * ...
		 */

		System.setProperty("reactor.bufferSize.small", "16");

		List<Object> list = new ArrayList<>();
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500; i++) {
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1); // <- 구독자보다 10배 빠르게 데이터 전달
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureDrop(list::add) // <- 버려질 때 행동을 정의할 수 있습니다.
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> Util.sleepMillis(10))
		    .subscribe(...);

		System.out.println("print drop list");
		System.out.println(list);

		/**
		 * ...
		 * subscribe; Received: 453
		 * subscribe; Completed
		 * print drop list
		 * [16, 17, 18, 19, 20, 21, 22, ...] // <- 버러진 것들을 확인할 수 있습니다.
		 */
	}
}
```

* 프로퍼티 설정을 바꾸어 실행한 값을 수정하고 난 예제입니다.
* 버퍼 사이즈를 16으로 설정하였고 데이터를 전달할 때 구독자의 10배 빠른속도로 데이터를 전합니다.
* 버퍼 사이즈의 75%를 전달 받고 그 다음 데이터의 전달 데이터를 버퍼안에 있는 모든 내용이 나오고 난 뒤 보이는 것을 확인할 수 있습니다.
* replenishing optimization[1]에 대한 정보를 찾아보시면 좋습니다.
    * https://beer1.tistory.com/18
* onBackpressureDrop()에서 버려지는 원소에 대한 정의를 추가할 수 있습니다.

[1]: https://projectreactor.io/docs/core/release/reference/#_on_backpressure_and_ways_to_reshape_requests

### Latest

```java
class BackpressureLatestExample {
	void latest() {
		System.setProperty("reactor.bufferSize.small", "16");
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500; i++) {
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1); // <- 10배 빠르게 데이터 전달
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureLatest()
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);
		/**
		 * emit 0
		 * ...
		 * emit 14
		 * subscribe; Received: 0
		 * emit 15
		 * ...
		 * emit 22
		 * subscribe; Received: 1
		 * emit 23
		 * ...
		 * emit 30
		 * subscribe; Received: 2
		 * emit 31
		 * ...
		 * emit 40
		 * subscribe; Received: 3
		 * emit 41
		 * ...
		 * emit 110 <- 75%가 소비되기 직전 데이터부터 75%가 큐에 채워집니다. [주목!]
		 * subscribe; Received: 11 <- 버퍼 사이즈(16개)의 75%(12개)가 전달될 때 [주목!]
		 * emit 111
		 * emit 112
		 * ...
		 * emit 146
		 * emit 147
		 * subscribe; Received: 15
		 * emit 148
		 * ...
		 * emit 158
		 * subscribe; Received: 110 <- [주목!] 15가 전달되고 그 다음은 110가 오는 것을 전달되는 것을 확인
		 * emit 159
		 * ...
		 * subscribe; Received: 499 // 반드시 마지막으로 전달된 값을 받습니다.
		 * subscribe; Completed
		 */
	}
}
```

* onBackpressureLatest()는 최신의 값을 유지하려고합니다.
    * 데이터가 전달되면 기존에 오래된 데이터를 버리고 새로운 데이터를 취하려 합니다.

### Error

```java
class BackpressureErrorExample {
	void error() {
		System.setProperty("reactor.bufferSize.small", "16");
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500; i++) {
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1);
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureError()
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);

		/**
		 * ...
		 * emit 133
		 * emit 134
		 * emit 135
		 * emit 136
		 * subscribe; Received: 15
		 * subscribe; Error: reactor.core.Exceptions$OverflowException: The receiver is overrun by more signals than expected (bounded queue...)
		 * emit 137 <- 계속 데이터가 전달됨을 주목하세요!
		 * emit 138
		 * ...
		 */

		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500 && !fluxSink.isCancelled(); i++) { // <- [주목!] isCancelled()을 확인합니다.
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1);
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureError()
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);

		/**
		 * ...
		 * emit 13
		 * emit 14
		 * emit 15
		 * emit 16 // [주목!] 16까지만 전달되었습니다.
		 * subscribe; Received: 1
		 * subscribe; Received: 2
		 * subscribe; Received: 3
		 * subscribe; Received: 4
		 * subscribe; Received: 5
		 * subscribe; Received: 6
		 * subscribe; Received: 7
		 * subscribe; Received: 8
		 * subscribe; Received: 9
		 * subscribe; Received: 10
		 * subscribe; Received: 11
		 * subscribe; Received: 12
		 * subscribe; Received: 13
		 * subscribe; Received: 14
		 * subscribe; Received: 15
		 * subscribe; Error: reactor.core.Exceptions$OverflowException: The receiver is overrun by more signals than expected (bounded queue...)
		 */
	}
}
```

* onBackpressureError()는 더 이상 큐 사이즈가 충분하지 않으면 에러를 발생시킵니다.
    * 이 때 cancel 신호가 전달되기 때문에, 데이터 전달할 때 isCancelled()을 확인해야 합니다.

### Buffer

```java
class BackpressureBufferExample {
	void buffer() {
		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500 && !fluxSink.isCancelled(); i++) { // <- [주목!] isCancelled()을 확인합니다.
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1);
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureBuffer(20) // 명시적으로 버퍼 사이즈 지정
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);

		/**
		 * ...
		 * emit 14
		 * emit 15
		 * emit 16
		 * subscribe; Received: 0
		 * emit 17
		 * emit 18
		 * ...
		 * subscribe; Received: 20
		 * subscribe; Received: 21
		 * subscribe; Error: reactor.core.Exceptions$OverflowException: The receiver is overrun by more signals than expected (bounded queue...)
		 */

		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500 && !fluxSink.isCancelled(); i++) { // <- [주목!] isCancelled()을 확인합니다.
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1);
			    }
			    fluxSink.complete();
		    })
		    .onBackpressureBuffer(20, BufferOverflowStrategy.DROP_LATEST) // 명시적으로 버퍼 사이즈 지정 및 버퍼 오버 플로우 발생시 전략 설정
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);

		/**
		 * ...
		 * emit 14
		 * emit 15
		 * emit 16
		 * subscribe; Received: 0
		 * emit 17
		 * emit 18
		 * ...
		 * subscribe; Received: 20
		 * ...
		 * emit 498
		 * emit 499
		 * subscribe; Received: 52
		 * subscribe; Received: 53
		 * ...
		 * subscribe; Received: 322
		 * subscribe; Completed
		 */

		Flux.create(fluxSink -> {
			    for (int i = 0; i < 500 && !fluxSink.isCancelled(); i++) {
				    fluxSink.next(i);
				    System.out.println("emit " + i);
				    sleepMillis(1);
			    }
			    fluxSink.complete();
		    }, FluxSink.OverflowStrategy.DROP) // <- Flux.create에 명시적으로 backpressure 전략을 지정할 수 있습니다.
		    .publishOn(Schedulers.boundedElastic())
		    .doOnNext(i -> sleepMillis(10))
		    .subscribe(...);
	}
}
```

* Buffer전략은 Flux.create의 기본전략입니다.
* onBackpressureBuffer()로 명시적으로 버퍼 사이즈를 지정할 수 있습니다.
    * 명시적으로 버퍼 사이즈를 지정하면, 버퍼 오버 플로우 발생시 에러가 발생하게 됩니다.
    * 버퍼 사이즈를 지정하지 않았을 때에는 버퍼 오버 플로우 발생시 에러가 발생하지 않습니다.
    * 버퍼 사이즈를 지정하면서, 버퍼 오버 플로우에 대한 전략을 설정할 수 있습니다.
* Flux.create를 호출할 때 명시적으로 backpressure 전략을 지정할 수 있습니다.

# Reactor Context

* ![hello_reactor_04_02.png](/assets/images/hello_reactor_04/hello_reactor_04_02.png)
* 리액터 컨텍스트는 런타임 단계에서 필요한 컨텍스트 정보에 엑세스할 수 있도록 하는 기능입니다.
* 예를 들어 세션키 값을 받아 로그인한 사용자의 정보를 조회하는 경우, 세션키 값을 컨텍스트에 저장하고, 컨텍스트에서 세션키 값을 가져와 사용자 정보를 조회할 수 있습니다.

## ThreadLocal

* 우리는 위 역할을 하기 위해 ThreadLocal를 떠올릴 법도 합니다.
* 대부분의 프레임워크는 ThreadLocal에 SecurityContext를 전달하고 사용자의 엑세스가 적절한지 확인합니다.
* 하지만 ThreadLocal은 단일 스레드를 이용할 때에만 제대로 동작합니다.
* 비동기 처리 방식을 사용하면 ThreadLocal을 사용할 수 있는 구간이 매우 짧아집니다.

```java
class ThreadLocalHasProblemOnAsync() {
	void threadLocal() {
		ThreadLocal<Map<Object, Object>> threadLocal = new ThreadLocal<>();
		threadLocal.set(new HashMap<>());

		Flux.range(0, 10)
		    .doOnNext(i -> threadLocal.get().put(i, new Random(i).nextGaussian())) // <- [주목!] ThreadLocal에 값을 넣고 있습니다. (main 스레드)
		    .publishOn(Schedulers.parallel()) // <- [주목!] 비동기 처리를 하고 있습니다. 이 다음 downstream에서는 다른 쓰레드에서 수행됩니다.
		    .map(i -> threadLocal.get().get(i)) // <- [문제 지점!] ThreadLocal에서 값을 꺼내고 있습니다. (ThreadLocal에서 값을 넣은 곳과는 다른 쓰레드에서 수행됩니다.)
		    .blockLast();
	}
}
```

* 위 코드는 컴파일이 잘 됩니다.
* publishOn 다음 map에서 NullPointException이 발생합니다.
    * 메인 쓰레드의 값을 다른 쓰레드에서 사용할 수 없기 때문입니다.
* 멀티쓰레드 완경에서는 ThreadLocal을 사용은 매우 위험하여 예기치 않은 동작을 할 수 있습니다.
* ThreadLocal 데이터를 다른 쓰레드로 전송할 수 있지만, 모든 곳에서 일관된 전송을 보장하지 않습니다.

## contextWrite

```java
class ContextUsing() {
	void context() {
		Flux.range(0, 10)
		    .flatMap(k -> Mono.deferContextual(ctx -> {
			    Map<Object, Object> randoms = ctx.get("randoms");
			    randoms.put(k, new Random(k).nextGaussian());
			    return Mono.empty();
		    }).thenReturn(k))
		    .publishOn(Schedulers.parallel())
		    .flatMap(k -> Mono.deferContextual(ctx -> {
			    Map<Object, Object> randoms1 = ctx.get("randoms");
			    return Mono.just(randoms1.get(k));
		    }))
		    .contextWrite(ctx -> ctx.put("randoms", new HashMap<>())) // 마치 쓰레드 로컬 초기화와 같은 역할을 합니다.
		    .doOnNext(System.out::println)
		    .blockLast();
	}
}
```

* deferContextual()를 사용하면 현재 스트림의 Context 인스턴스를 가져올 수 있습니다.
* contextWrite()를 통해서 해당 스트림 안에서 사용할 Context를 설정할 수 있습니다.

## Context

* Context는 본질적으로 Immutable 객체여서 새로운 요소를 추가하게 되면 Context는 새로운 인스턴스로 변경됩니다.
    * Context는 스트림의 다른 지점에서 동일한 객체가 아닐 수 있습니다.
* 구독 단계에서 Subscriber는 Publisher 체인을 따라 위쪽 방향으로 Subscriber가 래핑되는 것을 확인하였습니다.
* 이 때 추가 Context 객체를 전달하기 위해서 CoreSubscriber라는 특정 인터페이스를 사용합니다.

```java
interface CoreSubscriber<T> extends Subscriber<T> {
	default Context currentContext() {
		return Context.empty();
	}
}
```

* currentContext()를 통해서 현재 Context를 가져올 수 있습니다.
* 아래 예제에서는 컨텍스트가 추가될 때 마다 새로운 Context 인스턴스로 변경되는 예제입니다.

```java
class ContextChange {
	void contextChange() {
		printCurrentContext("top")
				.contextWrite(Context.of("top", "context"))
				.flatMap((x) -> printCurrentContext("middle"))
				.contextWrite(Context.of("middle", "context"))
				.flatMap((x) -> printCurrentContext("bottom"))
				.contextWrite(Context.of("bottom", "context"))
				.flatMap((x) -> printCurrentContext("init"))
				.block();

		/**
		 * top : Context3{bottom=context, middle=context, top=context}
		 * middle : Context2{bottom=context, middle=context}
		 * bottom : Context1{bottom=context}
		 * init : Context0{}
		 */
	}

	private Mono<ContextView> printCurrentContext(String id) {
		return Mono.deferContextual(ctx -> {
			System.out.println(id + " : " + ctx);
			return Mono.just(ctx);
		});
	}
}
```

* 위의 예제에서 보듯이 top 해당 부분에서 사용할 수 있는 Context를 살펴보면 전체 Context가 포함되어있습니다.
* middle에서는 정의한 컨텍스트와 bottom에서 정의한 컨텍스트가 포함되어있습니다.
* 가장 마지막에 있는 컨텍스트는 비어있는 것을 확인할 수 있습니다.

### Context authentication example

* 아래 예제에서는 Context를 활용해서 인증을 하는 예제입니다.

```java
class ContextAuthenticate() {
	void authenticateExample() {
		Mono<String> welcomeMessage = getWelcomeMessage(); // 컨텍스트 설정 X
		Mono<String> welcomeMessageWithContext = getWelcomeMessage().contextWrite(Context.of("user", "sub")); // 컨텍스트 user 설정 O

		welcomeMessage.subscribe(...);
		System.out.println("===============================");
		welcomeMessageWithContext.subscribe(...);
		/**
		 * subscriber; Error: java.lang.RuntimeException: unauthenticated
		 * ===============================
		 * subscriber; Received: Welcome sub
		 * subscriber; Completed
		 */

		Mono<String> welcomeMessageWithContext2 = getWelcomeMessage().contextWrite(Context.of("user", "sub1"))
		                                                             .contextWrite(Context.of("user", "sub2"));

		welcomeMessageWithContext2.subscribe(...);

		/**
		 * subscriber; Received: Welcome sub1 <- [주목!] 맨 처음에 user2로 설정되었지만, sub1으로 덮어씌워짐
		 * subscriber; Completed
		 */

		Mono<String> welcomeMessageWithContext3 = getWelcomeMessage().contextWrite(ctx -> ctx.put("user", ctx.get("user").toString().toUpperCase())) // 컨텍스트 키 수정 
		                                                             .contextWrite(Context.of("user", "sub2"));


		welcomeMessageWithContext3.subscribe(new DefaultSubscriber("subscriber"));
		/**
		 * subscriber; Received: Welcome SUB2 <- [주목!] 맨 처음에 user2로 설정되었고, user2가 대문자로 수정됨
		 * subscriber; Completed
		 */
	}

	private Mono<String> getWelcomeMessage() {
		return Mono.deferContextual(context -> {
			if (context.hasKey("user")) {
				return Mono.just("Welcome " + context.get("user"));
			}
			return Mono.error(new RuntimeException("unauthenticated"));
		});
	}
}
```

* Context는 스프링 프레임워크 내에서 광범위하게 사용되며 특히 스프링 시큐리티에서 더욱 많이 사용중 입니다.
* 리액터의 Context에 대한 자세한 내용은 다음을 살펴보세요
    * https://projectreactor.io/docs/core/release/reference/#context