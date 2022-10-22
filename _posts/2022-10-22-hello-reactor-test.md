---
title:  "리액터 테스트 & 좀 더 알아보면 좋을 것"
excerpt: "테스트 그리고 못 다한 내용"
category: "reactive"

last_modified_at: 2022-10-22T
---

# 리액터 테스트

## StepVerifier

* `Publisher`를 구독하면서 예상값, 순서를 검증할 수 있습니다.
* 반드시 마지막에는 `verify...`(`StepVerifier.LastStep`)이 호출되어야 합니다. (terminate)

### 데이터 개별 검증

```java
class StepVerifierTest {
	@Test
	void stepVerifierTest() {
		Flux<Integer> just = Flux.just(1, 2, 3); // 1, 2, 3이 순차적으로 발행
		StepVerifier.create(just)
		            .expectNext(1)
		            .expectNext(2)
		            .expectNext(3)
		            .verifyComplete(); // 1, 2, 3이 모두 나오고나서 onComplete

		Flux<Integer> just2 = Flux.just(1, 2, 3);
		just2.as(StepVerifier::create) // as()를 통하여 좀 더 간단하게 표현이 가능합니다.
		     .expectNext(1, 2, 3) // expectNext는 여러개의 값을 한번에 검증할 수 있습니다.
		     .verifyComplete();
	}
}
```

* `StepVerifier.create()`에 `Publisher`를 넣어서 검증을 진행할 수 있습니다.
    * `expectNext()`를 통해 예상값을 검증할 수 있습니다.
    * `verifyComplete()`를 통해 `Publisher`가 완료되었는지 검증할 수 있습니다.
* `as()`를 통해서도 좀 더 간단하게 `StepVerifier`를 생성할 수 있습니다.

### 에러 검증

```java
class ErrorTest {
	@Test
	void errorTest() {
		Flux<Integer> just = Flux.just(1, 2, 3);
		Flux<Object> error = Flux.error(new RuntimeException("error"));
		Flux<Object> concat = Flux.concat(just, error);

		concat.as(StepVerifier::create)
		      .expectNext(1, 2, 3)
		      .verifyError(RuntimeException.class);
	}

	@Test
	void assertionErrorTest() {
		Flux<Integer> just = Flux.just(1, 2, 3);
		Flux<Object> error = Flux.error(new RuntimeException("error"));
		Flux<Object> concat = Flux.concat(just, error);

		// 테스트가 실패하는 것을 검증
		Assertions.assertThrows(AssertionError.class, () -> {
			concat.as(StepVerifier::create)
			      .expectNext(1, 2, 3)
			      .verifyError(IllegalArgumentException.class);
			// RuntimeException이 발행해야하는데 IllegalArgumentException이 발행되었기 때문에 테스트가 실패합니다.
		});
	}
}
```

* `verifyError()`를 통해 에러가 발생했는지 검증할 수 있습니다.

### 데이터 개수 검증, 건너뛰기

```java
class CountTest {
	@Test
	void expectCount() {
		Flux<Integer> just = Flux.range(1, 50);

		just.as(StepVerifier::create)
		    .expectNextCount(50) // 50개의 값이 발행되었는지 검증
		    .verifyComplete();
	}

	@Test
	void consumeWhile() {
		Flux<Integer> just = Flux.range(1, 50);

		just.as(StepVerifier::create)
		    .thenConsumeWhile(i -> i < 100) // 100보다 작은 값이 나올동안 건너뛰기
		    .verifyComplete(); // 1 ~ 50은 100보다 모두 작기 때문에 모두 소모하여서 검증이 완료됩니다.
	}
}
```

* `expectNextCount()`를 통해 발행된 값의 개수를 검증할 수 있습니다.
* `thenConsumeWhile()`을 통해 조건에 맞는 값들을 건너뛸 수 있습니다.

### 지연 검증

```java
class DelayTest {
	@Test
	void delay() {
		Mono<BookOrder> mono = Mono.fromSupplier(BookOrder::new).delayElement(Duration.ofSeconds(3));
		mono.as(StepVerifier::create)
		    .assertNext(b -> Assertions.assertNotNull(b.getAuthor()))
		    .verifyComplete(); // 검증까지 완료되는데 3초 걸림
	}

	@Test
	void verifyDuration() {
		Mono<BookOrder> mono = Mono.fromSupplier(BookOrder::new).delayElement(Duration.ofSeconds(3));
		mono.as(StepVerifier::create)
		    .assertNext(b -> Assertions.assertNotNull(b.getAuthor()))
		    .expectComplete()
		    .verify(Duration.ofSeconds(5)); // 5초 안에 위의 검증이 모두 이루어지는지 확인합니다. (검증은 총 3초 걸림)
	}

	@Test
	void verifyFail() {
		Mono<BookOrder> mono = Mono.fromSupplier(BookOrder::new).delayElement(Duration.ofSeconds(3));

		Assertions.assertThrows(AssertionError.class, () -> {
			mono.as(StepVerifier::create)
			    .assertNext(b -> Assertions.assertNotNull(b.getAuthor()))
			    .expectComplete()
			    .verify(Duration.ofSeconds(1)); // 1초 안에 complete 안됨 따라서 실패
		});
	}

	@Getter
	static class BookOrder {
		private final String title;
		private final String author;

		public BookOrder() {
			this.title = "title";
			this.author = "author";
		}
	}
}
```

* `verify(Duration)`을 통해 검증이 완료되는데 걸리는 시간을 제한할 수 있습니다.
* 위의 검증에서는 실제로 데이터가 발행되는데 3초가 걸리기 때문에 검증이 완료되는데 3초가 걸립니다.

### 지연 검증(가상 시간 활용)

```java
class VirtualTimeTest {
	@Test
	void realTime() {
		Flux<String> stringFlux = timeConsumingFlux();
		stringFlux.as(StepVerifier::create)
		          .expectNext("1a", "2a", "3a", "4a")
		          .verifyComplete(); // 검증까지 총 4초 소요
	}

	@Test
	void virtualTime1() {
		// 가상 시간을 사용하려면 아쉽게도 as 메서드를 사용할 수 없습니다.
		StepVerifier.withVirtualTime(this::timeConsumingFlux)
		            .thenAwait(Duration.ofSeconds(30)) // 30초를 가상으로 기다림
		            .expectNext("1a", "2a", "3a", "4a") // 30초가 지나있으면 4개의 값이 발행되어 있음
		            .verifyComplete(); // 매우 빠르게 검증이 완료됨
	}

	private Flux<String> timeConsumingFlux() {
		return Flux.range(1, 4)
		           .delayElements(Duration.ofSeconds(1))
		           .map(i -> i + "a");
	}
}
```

* `StepVerifier.withVirtualTime(Supplier<Publisher>>)`를 사용하여 가상 시간을 사용할 수 있습니다.
* `thenAwait(Duration)`을 통해 가상 시간을 기다릴 수 있습니다.

### Context 검증

```java
class ContextTest {
	@Test
	public void test1() {
		getWelcomeMessage().as(StepVerifier::create)
		                   .verifyError(RuntimeException.class); // <- Mono.error 발생
	}

	@Test
	public void test2() {
		// 초기 Context를 설정하고 검증을 진행합니다.
		StepVerifierOptions stepVerifierOptions = StepVerifierOptions.create().withInitialContext(Context.of("user", "sam"));

		getWelcomeMessage().as(publisher -> StepVerifier.create(publisher, stepVerifierOptions))
		                   .expectNext("Welcome sam")
		                   .verifyComplete();
	}

	private static Mono<String> getWelcomeMessage() {
		return Mono.deferContextual(context -> {
			if (context.hasKey("user")) {
				return Mono.just("Welcome " + context.get("user"));
			}
			return Mono.error(new RuntimeException("unauthenticated"));
		});
	}
}
```

# 더 이야기 해볼 것들

## Webflux

* 적은 수의 스레드로 동시성을 처리하고 더 적은 하드웨어 리소스로 확장할 수 있는 Non-Blocking I/O를 지원하는 Reactive Web Framework입니다.
* 서블릿 3.1 에서는 Non-Blocking I/O를 지원하나
    * Filter, Servlet과 같은 동기식(synchronous), getParameter, getPart과 같은 Blocking의 사용이 자제됩니다.
* 그래서 새로운 웹 프레임워크 스택의 필요성이 생겼습니다.
* 좀 더 필요한 정보는 [이곳][1]에서 살펴봅시다

[1]: https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html

## Reactor-Netty 맛만 보기

* ![hello_reactor_06_01.png](/assets/images/hello_reactor_06/hello_reactor_06_01.png)
* 두 개의 이벤트 루프 그룹을 사용합니다.
    * ```Boss Thread Group```: 클라이언트의 연결을 수락합니다.
        * 연결을 담당합니다. (리슨 소켓을 관리합니다.)
        * 예를 들어 80, 443 포트를 사용하는 서버는 2개의 Boss 쓰레드를 사용하게 할 수 있습니다.
            * 한 개의 스레드로도 관리하게 할 수 있습니다.
    * ```Worker Thread Group```: 연결된 클라이언트의 요청을 처리합니다.
        * 연결된 클라이언트와의 송수신을 담당합니다. (연결된 소켓을 관리합니다.)
        * 실제로 클라이언트의 요청을 처리하고 응답을 내보냅니다.
* 좀 더 필요한 정보는 [이곳][2]에서 살펴봅시다

[2]: https://projectreactor.io/docs/netty/release/reference/index.html

## RSocket

* TCP 계층과 같은 계층에서 Reactive Streams를 지원하는 프로토콜입니다.
* Netflix에서 HTTP를 대체하기 위하여 만들어졌습니다.

### 기존 방식의 문제점

1. HTTP는 기본적으로 3-way Handshake 과정이 필수적으로 진행된다.
2. 또한 비연결성 특성으로 다시 연결해주어야 한다. 그리고 실시간적이지 못하다. (서버측에서 무언가가 바뀌었을 때 클라이언트가 요청해야 알 수 있음)
3. 서비스간에 통신을 할 때 직렬화 & 역직렬화 하는 과정이 필요하다. (항상 CPU 시간이 필요하다.)

### RSocket을 사용하면 좋은점

1. HTTP통신을 하지 않는다. TCP 레이어와 같은 계층(5/6 Layer)에서 작동한다.
2. 연결을 끊지 않는다. 또한 무언가 바뀌었을 때 클라이언트가 인지할 수 있게 할 수 있다.
3. 서비스간에 통신은 바이너리(이진) 형태로 통신한다. (기계 친화적이긴 하지만, 사용자 친화적이진 않음)

|       Spring WebFlux        |                                    Spring RSocket                                    |
|:---------------------------:|:------------------------------------------------------------------------------------:|
| Non-blocking & Asynchronous |                             Non-blocking & Asynchronous                              |
|     Request - Response      | Request - Response<br/>Request - Stream<br/>Bi-directional Streaming<br/>Fire&Forget |
|            HTTP             |                              TCP, WebSocket, UDP(Aeron)                              |
|           Layer 7           |                                      Layer 5, 6                                      |
|            JSON             |                                        Binary                                        |
|        클라이언트가 초기에 요청        |                                       발행 & 구독                                        |
|              -              |                                       스트림 재수행                                        |
|              -              |                                        역압 지원                                         |
|      많은 도구가 있고 증명되어있음       |                                 새로움, 도구 많이 없음, 증명안됨                                  |

* 좀 더 필요한 정보는 [이곳][3]과 [이곳][4]에서 살펴봅시다.

[3]: https://rsocket.io/

[4]: https://docs.spring.io/spring-framework/docs/current/reference/html/rsocket.html

## Java19 Virtual Thread Preview

* 자바 19에서 Preview로 새롭게 추가된 쓰레드 모델입니다.
* 기존 사용하던 쓰레드는 platform 쓰레드라고 불리우고, 이는 OS의 쓰레드와 1:1로 매핑되어 있습니다.
* virtual 쓰레드는 JVM에 의해 특정 platform 쓰레드에서 실행되도록 예약되며 platform 쓰레드는 동시에 하나의 virtual 쓰레드만 실행합니다.
* 단순히 쓰레드 모델이 추가된 것이고, Reactive Stream 모델은 아니지만 기존의 쓰레드 모델과 비교하여 잠재력이 높아보입니다.
* 좀 더 필요한 정보는 [이곳][5]과 [이곳][6]과 [이곳][7]에서 살펴봅시다.

[5]: https://openjdk.org/jeps/425

[6]: https://medium.com/javarevisited/how-to-use-java-19-virtual-threads-c16a32bad5f7

[7]: https://spring.io/blog/2022/10/11/embracing-virtual-threads

