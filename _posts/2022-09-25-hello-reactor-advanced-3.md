---
title:  "리액터 살짝만 발을 더 깊이 - 3"
excerpt: "배압 관리, 리액터 컨텍스트"
category: "reactive"

last_modified_at: 2022-10-03T
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

## ThreadLocal

## contextWrite

## contextUpdate

## Rate Limiting