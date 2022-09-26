---
title:  "리액터 살짝만 발을 더 깊이 - 1"
excerpt: "리액터 Sink와 리액터 수명주기에 대해서"
category: "reactive"

last_modified_at: 2022-09-12T
---

# Flux.generate, Flux.create, Flux.push

## 프로그래밍적으로(programmatically) 데이터를 방출해보기

* generate, create, push는 개발자가 직접 데이터를 방출할 수 있도록 해줍니다.

### generate

* ![hello_reactor_02_02.png](/assets/images/hello_reactor_02/hello_reactor_02_02.png)

```java
class FluxGenerate {
	void generate() {
		// generate method doesn't need to loop
		Flux.generate(synchronousSink -> {
			    System.out.println("emit");
			    synchronousSink.next("Something" + new Random().nextInt(100));
		    })
		    .take(5) // 5개만 받기
		    .subscribe(...);

		System.out.println("============================================================");

		Flux.generate(synchronousSink -> {
			    System.out.println("emit");
			    synchronousSink.next("Something" + new Random().nextInt(100));
			    synchronousSink.complete(); // complete를 호출하면 더이상 데이터를 방출하지 않음
		    })
		    .take(5)
		    .subscribe(...);

		System.out.println("============================================================");

		Flux.generate(synchronousSink -> {
			    System.out.println("emit");
			    int a = new Random().nextInt(100);
			    if (a < 50) {
				    synchronousSink.next("Something" + a);
			    } else {
				    synchronousSink.complete();
			    }
		    })
		    .take(5)
		    .subscribe(...);

		System.out.println("============================================================");

		Flux.generate(synchronousSink -> {
			    System.out.println("emit");
			    int a = new Random().nextInt(100);
			    if (a < 50) {
				    synchronousSink.next("Something" + a);
			    } else {
				    synchronousSink.complete();
			    }
		    })
		    .take(5)
		    .subscribe(...);

		System.out.println("============================================================");

		Flux.generate(synchronousSink -> {
			    System.out.println("emit");
			    synchronousSink.next("Something" + new Random().nextInt(100));
			    synchronousSink.next("Something" + new Random().nextInt(100)); // synchronousSink permit only 1 item emit
		    })
		    .subscribe(...);

		System.out.println("============================================================");

		Flux.generate(() -> 1, // initial state
		              (counter, sink) -> {
			              int a = new Random().nextInt(100);
			              String name = "Something" + a;
			              sink.next(name);
			              if (counter >= 10 || a >= 50) {
				              sink.complete();
			              }
			              return counter + 1;
		              })
		    .subscribe(...);
	}
}
```

```text
emit
Received: Something41
emit
Received: Something67
emit
Received: Something3
emit
Received: Something86
emit
Received: Something74
Completed
============================================================
emit
Received: Something33
Completed
============================================================
emit
Received: Something41
emit
Received: Something31
emit
Received: Something28
emit
Completed // 50이상이면 바로 complete
============================================================
emit
Received: Something27
ERROR: java.lang.IllegalStateException: More than one call to onNext
============================================================	
Received: Something36
Received: Something46
Received: Something26
Received: Something54
Completed
```

* generate 함수는 synchronousSink를 인자로 받습니다. (Consumer<SyncrhonousSink<T>>)
* 인자로 받은 synchronousSink consumer를 반복해서 호출하게 됩니다.
* synchronousSink의 next 메소드를 반드시 한번만 호출하여 데이터를 무한적으로 방출할 수 있습니다.
* synchronousSink의 complete, error 메소드를 호출하면 더 이상 데이터를 방출하지 않습니다.
* synchronousSink가 데이터를 방출하는 동안 Flux.generate는 블록킹됩니다.
* next가 무한적으로 데이터를 방출하므로 take를 사용하여 데이터를 제한할 수 있습니다.
* generate 함수에 처음 상태를 consumer 형태로 전달할 수 있습니다.

### create

* ![hello_reactor_02_01.png](/assets/images/hello_reactor_02/hello_reactor_02_01.png)

```java
class FluxCreate {
	void create() {
		Flux.create(fluxSink -> {
			// emit what you provided
			fluxSink.next(1);
			fluxSink.next(2);
			fluxSink.complete();
		}).subscribe(...);

		System.out.println("============================================================");

		Flux.create(fluxSink -> {
			String country;
			int a;
			do {
				a = new Random().nextInt(100);
				country = "Something" + a;
				fluxSink.next(country);
			} while (a < 50);
			fluxSink.complete();
		}).subscribe(...);

		System.out.println("============================================================");

		Flux.create(fluxSink -> {
			    String country;
			    int a;
			    do {
				    a = new Random().nextInt(100);
				    country = "Something" + a;
				    fluxSink.next(country);
			    } while (a < 50); // -> doesn't check cancel;
			    fluxSink.complete();
		    })
		    .take(3) // <- notice
		    .subscribe(...);

		System.out.println("============================================================");

		Flux.create(fluxSink -> {
			    String country;
			    int a;
			    do {
				    a = new Random().nextInt(100);
				    country = "Something" + a;
				    System.out.println("emitting: " + country);
				    fluxSink.next(country);
			    } while (a < 50 && !fluxSink.isCancelled()); // check cancel manually!
			    fluxSink.complete();
		    })
		    .take(3)
		    .subscribe(...);
	}
}
```

```text
Received: 1
Received: 2
Completed
============================================================
Received: Something29
Received: Something23
Received: Something92
Completed
============================================================
emitting: Something42
Received: Something42
emitting: Something23
Received: Something23
emitting: Something48
Received: Something48
Completed // <- 3개를 온전히 받고 끝남
emitting: Something30 // <- 파이프라인에서는 계속 데이터 생성중 (fluxSink.next가 계속 호출은 되고 있는 중)
emitting: Something24
emitting: Something36
emitting: Something45
emitting: Something42
emitting: Something44
emitting: Something12
emitting: Something13
emitting: Something82 // <- 50 이상이여야지 next호출이 종료됨
============================================================
emitting: Something29
Received: Something29
emitting: Something8
Received: Something8
emitting: Something26
Received: Something26
Completed // <- 3개를 온전히 받고 끝남; 파이프라인도 종료 (fluxSink.next 더 이상 호출 X)
```

* create 함수는 FluxSink를 인자로 받습니다. (Consumer<FluxSink<T>>)
* 인자로 받은 fluxSink consumer는 한번만 호출됩니다.
* FluxSink의 next 메소드를 호출하여 데이터를 방출할 수 있습니다.
* complete, error를 따로 호출하지 않아도 됩니다. (complete, error 시그널이 전달이 안될 뿐)
* generate와 다르게 무한적으로 데이터를 방출하려면 create 안에서 무한히 데이터를 직접 만들어서 방출해야 합니다.
* generate와 다르게 take를 사용하여 데이터를 제한한다면 create 안에서 cancel 여부를 직접 확인해야 합니다.
* Flux.create는 Flux.generate와 다르게 오버플로우 처리를 할 수 있는 인자를 추가로 설정할 수 있습니다.
    * 설정하지 않으면 기본적으로 BUFFER 전략을 사용합니다.

### push

```java
class FluxPush {
	void push() {
		NameProducer nameProducer = new NameProducer();

//      create 또는 push 둘 중에 하나를 주석 및 주석 해제하고 실행하였을 때 결과가 다릅니다.
//      Flux.create(nameProducer)
//          .subscribe(...);

		Flux.push(nameProducer) // push는 싱글 쓰레드에서만 동작하기를 기대합니다.
		    .subscribe(...);

		Runnable runnable = () -> nameProducer.produce();
		for (int i = 0; i < 10; i++) {
			new Thread(runnable).start();
		}

		sleepSeconds(1);
	}

	static class NameProducer implements Consumer<FluxSink<String>> {
		private FluxSink<String> fluxSink;

		@Override
		public void accept(FluxSink<String> stringFluxSink) {
			this.fluxSink = stringFluxSink;
		}

		public void produce() {
			String fullName = "Something" + new Random().nextInt(100);
			String threadName = Thread.currentThread().getName();
			this.fluxSink.next(fullName + ":" + threadName);
		}
	}
}
```

```text
push=> Received: Something52:Thread-1
push=> Received: Something7:Thread-5
push=> Received: Something79:Thread-7
push=> Received: Something84:Thread-9 // <- push는 10번을 호출했지만 4개밖에 나오질 않음

create=> Received: Something82:Thread-9
create=> Received: Something1:Thread-2
create=> Received: Something21:Thread-1
create=> Received: Something46:Thread-3
create=> Received: Something81:Thread-4
create=> Received: Something86:Thread-0
create=> Received: Something11:Thread-7
create=> Received: Something90:Thread-6
create=> Received: Something88:Thread-5
create=> Received: Something87:Thread-8 // <- create는 10번 호출하면 반드시 10개가 나옵니다.
```

* push는 FluxSink를 인자로 받습니다. (Consumer<FluxSink<T>>)
* push는 Flux.create와 유사하지만 싱글 쓰레드에서만 동작하기를 기대합니다.
* push가 싱글 쓰레드 처리만 한다는 단점이 있지만, 싱글 쓰레드에 맞춰 더 빠르게 동작하도록 최적화되어 있습니다.

# Sink

* Sink는 Subscriber, Publisher의 성격을 동시에 지니고 있습니다.
    * 그러면 Processor인가요?
* Processor는 맞는데 Processor의 대안으로 아주 쉽고 유용하게 사용할 수 있습니다. [1]
    * Processor를 직접 사용하게 된다면 리액티브 스트림 스펙 사양과 관련된 외부 동기화와 관련해서 매우 주의해서 사용하여야 합니다.
    * 또한 Reactor에서 Processor는 Deprecated 되어있습니다. (직접 사용을 권장하지 않습니다.)
        * Reactor 3.5.0 버전부터 FluxProcessor, MonoProscessor는 아예 삭제됩니다.
* 만약 정말 Processor를 사용할 일이 있다면 연산자(Operator)의 조합으로 대체할 수 없는지, Sink를 사용할 수 없는지(generate, create, push) 검토하기를 바랍니다.
    * 그렇다 하더라도 Reactor 3.5.0 버전부터는 Processor 직접 사용이 매우 제한되므로 사용하지 않는 것이 좋습니다.

* ![hello_reactor_02_03.png](/assets/images/hello_reactor_02/hello_reactor_02_03.png)

[1]: https://projectreactor.io/docs/core/release/reference/#processors

## Sink Type

|       Type       | Behavior | Pub:Sub |
|:----------------:|:--------:|:-------:|
|       one        |   Mono   |   1:N   |
|  many - unicast  |   Flux   |   1:1   |
| many - multicast |   Flux   |   1:N   |
|  many - replay   |   Flux   |   1:N   |

## Sink One

```java
class SinkOne {
	void sinkOne() {
		Sinks.One<Object> sink = Sinks.one();
		Mono<Object> objectMono = sink.asMono();
		objectMono.subscribe(...);
		sink.tryEmitValue("hi"); // <- 구독 후 값을 발행하는 것에 대해 유의하기
		sink.tryEmitValue("hello"); // <- 데이터 방출 실패

		/**
		 * subscribe; Received: hi
		 * subscribe; Completed
		 */

		Sinks.One<Object> sink2 = Sinks.one();
		Mono<Object> objectMono2 = sink2.asMono();
		objectMono2.subscribe(...);
		sink2.emitValue("hi", ((signalType, emitResult) -> {
			System.out.println("SIG TYPE: " + signalType.name()); // 해당 콜백은 실패했을 때만 처리되기 때문에 수행되지 않습니다.
			System.out.println("RES: " + emitResult.name());
			return false; // <- 재시도 여부를 결정합니다. 
		}));
		sink2.emitValue("hello", ((signalType, emitResult) -> { // Sink.One은 하나의 데이터만 방출할 수 있기 때문에 에러가 발생합니다.
			System.out.println("SIG TYPE: " + signalType.name());
			System.out.println("RES: " + emitResult.name());
			// return true; <- 만약 재시도를 한다면 무한히 재시도를 하게 됩니다. (하나의 데이터만 방출할 수 있기 때문에)
			return false;
		}));

		/**
		 * subscribe2; Received: hi
		 * subscribe2; Completed
		 * SIG TYPE: ON_NEXT
		 * RES: FAIL_TERMINATED
		 */

		Sinks.One<Object> sink3 = Sinks.one();
		Mono<Object> objectMono3 = sink3.asMono(); // subscribe to value
		objectMono3.subscribe(...); // <- 첫번째 구독자
		objectMono3.subscribe(...); // <- 두번째 구독자
		sink3.tryEmitValue("hi");
		/**
		 * subscribe; Received: hi
		 * subscribe; Completed
		 * subscribe2; Received: hi // <- 두번째 구독자에게도 데이터가 전달됨
		 * subscribe2; Completed
		 */
	}
}
```

* Sink.One은 하나의 데이터만 방출할 수 있습니다.
* tryEmitValue는 값 발행 후 EmitResult를 반환합니다.
    * Sinks.EmitResult Enum을 자세히 확인하시기 바랍니다.
* emitValue는 값 발행을 하고 나서 만약 실패하였을 때 처리에 대한 콜백을 받습니다.
    * 콜백의 반환값은 재시도 여부를 결정합니다.

## Sink Many

### Unicast

```java
class SinkManyUnicast {
	void sinkManyUnicast() {
		Sinks.Many<Object> sink = Sinks.many().unicast().onBackpressureBuffer();

		// handle through which subscribers will receive items
		Flux<Object> flux = sink.asFlux();
		flux.subscribe(...); // 첫번째 구독자
		flux.subscribe(...); // [주목!] 두번째 구독자

		sink.tryEmitNext("hi");
		sink.tryEmitNext("hi2");
		sink.tryEmitNext("hi3");

		/**
		 * subscriber2; Error: java.lang.IllegalStateException: UnicastProcessor allows only a single Subscriber
		 * subscriber; Received: hi
		 * subscriber; Received: hi2
		 * subscriber; Received: hi3
		 */
	}
}
```

* unicast는 하나의 구독자만 받을 수 있습니다.
* 첫번째 구독자가 complete 시그널을 받더라도 두번째 구독자가 구독을 할 수 없습니다.

### Multicast

```java
class SinkManyMulticast {
	void sinkManyMulticast() {
		// handle through which we would push items
		Sinks.Many<Object> sink = Sinks.many().multicast().onBackpressureBuffer();

		// handle through which subscribers will receive items
		Flux<Object> flux = sink.asFlux();

		flux.subscribe(...);
		flux.subscribe(...);

		sink.tryEmitNext("hi");
		sink.tryEmitNext("hi2");
		sink.tryEmitNext("hi3");

		/**
		 * subscriber; Received: hi
		 * subscriber2; Received: hi
		 * subscriber; Received: hi2
		 * subscriber2; Received: hi2
		 * subscriber; Received: hi3
		 * subscriber2; Received: hi3
		 */

		Sinks.Many<Object> sink2 = Sinks.many().multicast().onBackpressureBuffer();
		Flux<Object> flux2 = sink2.asFlux();

		flux2.subscribe(...);

		sink2.tryEmitNext("hi");
		sink2.tryEmitNext("hi2");

		flux2.subscribe(...); // 두번째 구독이 약간 늦음
		sink2.tryEmitNext("hi3");

		/**
		 * subscriber; Received: hi
		 * subscriber; Received: hi2
		 * subscriber; Received: hi3
		 * subscriber2; Received: hi3 <- 두번째 구독자는 hi, hi2를 받지 못함
		 */

		Sinks.Many<Object> sink3 = Sinks.many().multicast().onBackpressureBuffer();
		Flux<Object> flux3 = sink3.asFlux();

		sink3.tryEmitNext("hi");
		sink3.tryEmitNext("hi2"); // <- 구독전에 데이터가 발행되었음

		flux3.subscribe(...); // <- [주목] 첫번째 구독자 전에 데이터가 발행되었다면 어떻게 될까요?
		flux3.subscribe(...);
		sink3.tryEmitNext("hi3");
		flux3.subscribe(...);
		sink3.tryEmitNext("hi4");

		/**
		 * subscriber; Received: hi <- 첫번째 구독자는 hi를 전달 받았습니다.(왜냐하면 Publisher는 Subscriber가 존재할때만 데이터를 방출하기 때문입니다.)
		 * subscriber; Received: hi2
		 * subscriber; Received: hi3
		 * subscriber2; Received: hi3 <- 두번째 구독자는 hi, hi2를 전달 받지 못했습니다.
		 * subscriber; Received: hi4
		 * subscriber2; Received: hi4
		 * subscriber3; Received: hi4
		 */

		Sinks.Many<Object> sink4 = Sinks.many().multicast().directAllOrNothing(); // <- 멀티캐스트 스펙이 directAllOrNothing으로 설정되었습니다.
		Flux<Object> flux4 = sink4.asFlux();

		sink4.tryEmitNext("hi");
		sink4.tryEmitNext("hi2");

		flux4.subscribe(...);
		flux4.subscribe(...);
		sink4.tryEmitNext("hi3");
		flux4.subscribe(...);
		sink4.tryEmitNext("hi4");

		/**
		 * subscriber; Received: hi3 <- 첫번째 구독자가 hi, hi2를 전달받지 못하였습니다.
		 * subscriber2; Received: hi3
		 * subscriber; Received: hi4
		 * subscriber2; Received: hi4
		 * subscriber3; Received: hi4
		 */
	}
}
```

* multicast는 여러개의 구독자를 허용합니다.

#### Multicast Spec [2]

[2]: https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Sinks.MulticastSpec.html

* onBackpressureBuffer
    * Subscriber가 없을 때 발행된 이벤트들에 대해서 그 다음 구독하는 Subscriber에게 전달합니다.
* directAllOrNothing, directBestEffort
    * Subscriber는 자신이 구독한 시점에서부터의 이벤트만 받습니다.

```java
class MulticastSpec {
	void multicastSpec() {
		// reactor.util.concurrent.Queues.SMALL_BUFFER_SIZE를 조절
		System.setProperty("reactor.bufferSize.small", "16");

		Sinks.Many<Object> sink = Sinks.many().multicast().directAllOrNothing(); // <- directAllOrNothing 
		Flux<Object> flux = sink.asFlux();

		flux.subscribe(...);
		flux.delayElements(Duration.ofMillis(200)).subscribe(...); // <- 처리하는 속도가 매우 느린 구독자

		for (int i = 0; i < 100; i++) {
			sink.tryEmitNext(i);
		}

		/**
		 * subscriber; Received: 0
		 * ...
		 * subscriber; Received: 30
		 * subscriber; Received: 31 <- 첫번째 구독자가 32~99 이벤트를 받지 못함; (왜냐하면 두번째 구독자가 0~31까지 처리할 수 밖에 없기 때문에)
		 * subscriber2; Received: 0 <- too slow!
		 * ...
		 * subscriber2; Received: 31
		 */

		sleepSeconds(10);

		// handle through which we would push items
		Sinks.Many<Object> sink2 = Sinks.many().multicast().directBestEffort(); // <- directBestEffort
		Flux<Object> flux2 = sink2.asFlux();

		flux2.subscribe(...);
		flux2.delayElements(Duration.ofMillis(200)).subscribe(...);

		for (int i = 0; i < 100; i++) {
			sink2.tryEmitNext(i);
		}

		/**
		 * subscriber; Received: 0
		 * ...
		 * subscriber; Received: 98
		 * subscriber; Received: 99 <- 첫번째 구독자는 모든 이벤트를 받았습니다.
		 * subscriber2; Received: 0 <- too slow!
		 * ...
		 * subscriber2; Received: 31
		 */

		sleepSeconds(10);
	}
}
```

* directAllOrNothing
    * 구독자중 하나라도 이벤트를 처리하지 못하는 경우(direct all or nothing; 전부 or 전무) 호출자에게 Sinks.EmitResult.FAIL_OVERFLOW 알림
    * 모든 구독자의 관점에서는 데이터가 누락되었습니다.
* directBestEffort
    * 구독자중 모두 이벤트를 처리하지 못하는 경우 호출자에게 Sinks.EmitResult.FAIL_OVERFLOW 알림
    * 그렇지 않으면 느린 구독자를 무시하고 최선의 노력으로(best effort) 빠른 구독자에게 요소를 전달합니다.
    * 느린 구독자의 관점에서 데이터가 누락되었습니다.

### Replay

```java
class SinkManyReplay {
	void sinkManyReplay() {
		Sinks.Many<Object> sink = Sinks.many().replay().all(/* if empty => Queues.SMALL_BUFFER_SIZE */);
		Flux<Object> flux = sink.asFlux();

		sink.tryEmitNext("hi");
		sink.tryEmitNext("hi2");

		flux.subscribe(...); // 첫번째 구독자
		flux.subscribe(...); // 두번째 구독자
		sink.tryEmitNext("hi3");

		flux.subscribe(...); // 세번째 구독자
		sink.tryEmitNext("hi4");

		/**
		 * subscriber; Received: hi
		 * subscriber; Received: hi2
		 * subscriber2; Received: hi
		 * subscriber2; Received: hi2
		 * subscriber; Received: hi3
		 * subscriber2; Received: hi3
		 * subscriber3; Received: hi <- 세번째 구독자도 hi, hi2를 전달받았습니다. 
		 * subscriber3; Received: hi2 <- 세번째 구독자도 hi, hi2를 전달받았습니다.
		 * subscriber3; Received: hi3
		 * subscriber; Received: hi4
		 * subscriber2; Received: hi4
		 * subscriber3; Received: hi4
		 */
	}
}
```

* replay는 여러개의 구독자를 허용하고, 이전에 발행된 이벤트들에 대해서 기억을 해두고 추가로 구독하는 Subscriber에게 전달합니다.
* all 뿐만 아니라 limit, latest등의 다양한 옵션을 제공하고 limit은 갯수 뿐만 아니라 시간을 지정할 수 도 있습니다.

### Sink가 가지고 있는 메서드들은 쓰레드-세이프 하지 않습니다.

```java
class SinkIsNotThreadSafe() {
	void sinkIsNotThreadSafe() {
		Sinks.Many<Object> sink = Sinks.many().multicast().onBackpressureBuffer();
		Flux<Object> flux = sink.asFlux();
		List<Object> list = new ArrayList<>();

		flux.subscribe(list::add); // 구독을 하면 list에 추가
		for (int i = 0; i < 1000; i++) {
			int finalI = i;
			CompletableFuture.runAsync(() -> sink.tryEmitNext("hi" + finalI)); // tryEmitNext를 여러개의 쓰레드에서 호출중
		}

		sleepSeconds(3);
		System.out.println(list.size()); // <- 1000이 아님
	}
}
```

* Sink가 가지는 메소드들은 쓰레드-세이프 하지 않기 때문에 여러개의 쓰레드에서 동시에 호출하면 문제가 발생할 수 있습니다.

```java
class SinkIsNotThreadSafeFix() {
	void sinkIsNotThreadSafeFix() {
		Sinks.Many<Object> sink = Sinks.many().multicast().onBackpressureBuffer();
		Flux<Object> flux = sink.asFlux();
		List<Object> list = new ArrayList<>();

		flux.subscribe(list::add); // 구독을 하면 list에 추가
		for (int i = 0; i < 1000; i++) {
			int finalI = i;
			CompletableFuture.runAsync(() -> {
				sink.emitNext("hi" + finalI, (sig, res) -> {
					System.out.println("emitNext: " + sig + " " + res); // emitNext: onNext FAIL_NON_SERIALIZED
					return true; // <- [주목!] 재시도 하고 있음
				}); 
			});
		}

		sleepSeconds(3);
		System.out.println(list.size()); // 1000
	}
}
```

* sink의 emitNext를 이용하여 삽입 실패시 재시도 하게끔 수정한 코드
* 실패 signal은 onNext, Sinks.EmitResult는 FAIL_NON_SERIALIZED 입니다.

# 리액티브 스트림 수명 주기

## 조립 단계 (Assembly time)

* 리액터는 복잡한 처리 흐름을 구현할 수 있는 연쇄형(chaining; 체이닝) API를 제공합니다.
* 마치 빌더 패턴과 유사한데 일반적인 빌더 패턴과는 다르게 불변성(immutability) 입니다.
* 각각의 연산자가 새로운 객체를 생성합니다.

```java
class ReactorHasChain {
	void eachOperatorHasReturn() {
		// 각각의 연산자들은 새로운 객체를 생성합니다.
		Flux<Integer> source = Flux.just(1, 20, 300, 4000);
		Flux<String> mapFlux = source.map(String::valueOf); // 새로운 객체
		Flux<String> filterFlux = mapFlux.filter(s -> s.length() > 1); // 새로운 객체
	}

	void reactorHasChain() {
		// 체이닝 API를 사용하는 경우
		Flux.just(1, 20, 300, 4000)
		    .map(String::valueOf) // 새로운 객체
		    .filter(s -> s.length() > 1) // 새로운 객체
	}
}

class ReactorNoChain {
	void reactorNoChain() {
		// 체이닝 API 미제공 가정 코드
		Flux<Integer> source = new FluxJust(1, 20, 300, 4000);
		Flux<String> mapFlux = new FluxMap(source, String::valueOf);
		Flux<String> filterFlux = new FluxFilter(mapFlux, s -> s.length() > 1);
	}
}
```

* 만약 체이닝 API를 사용하지 않는다고 생각해봅시다.
* 위의 코드는 Publisher는 다음과 같이 조립됩니다. (수도 코드)

```text
FluxFilter(
    FluxMap(
        FluxJust(1, 20, 300, 4000),
    )
)
```

* Just -> Map -> Filter 순서로 연산자를 적용하면 Filter -> Map -> Just 순으로 감싸지는 것을 확인할 수 있습니다.

## 구독 단계 (Subscription time)

* 특정 Publisher를 구독(subscribe)하면 발생합니다.
* 위 조립 단계에서 가장 마지막 단계인 filterFlux를 구독한다고 생각합시다.
* 구독 단계 동안 Subscriber 체인을 통해 Subscriber가 전파되는 방식을 관찰해봅시다.

```text
filterFlux.subscribe(new Subscriber) {
    // 인자에서 전달받은 Subscriber를 FilterSubscriber로 감싸서 전달
    mapFlux.subscribe(new FilterSubscriber(Subscriber)) {
        // 인자에서 전달받은 FilterSubscriber(Subscriber)를 MapSubscriber로 감싸서 전달
        source.subscribe(new MapSubscriber(FilterSubscriber(Subscriber))) {
            // 인자에서 전달받은 MapSubscriber(FilterSubscriber(Subscriber))를 JustSubscriber로 감쌈
            // 실제 데이터를 송신하기 시작하는 부분입니다.
            new JustSubscriber(MapSubscriber(FilterSubscriber(Subscriber))).onSubscribe(...);
        }
    }
}
```

* 위 수도 코드에서 Subscriber 형태만 살펴봅시다.

```text
JustSubscriber(
    MapSubscriber(
        FilterSubscriber(
            Subscriber
        )
    )
)
```

* 조립 단계에서는 Just는 가장 내부에 있었지만 구독 단계에서는 가장 바깥에 존재합니다. (조립 단계와는 역피마리드 형태)

## 런타임(실행) 단계 (Runtime)

* 이 단계에서는 Publisher와 Subscriber 간에 실제 신호가 교환됩니다.
* 둘 간의 첫 신호는 onSubscribe(), request() 시그널입니다.
* onSubscribe()가 호출되는 과정을 수도 코드로 살펴봅시다.

```text
JustSubscriber(MapSubscriber(FilterSubscriber(Subscriber))).onSubscribe(new Subscription()) {
    // Subscription을 JustSubscription으로 감싸서 전달
    MapSubscriber(FilterSubscriber(Subscriber)).onSubscribe(new JustSubscription(Subscription)) {
        // JustSubscription(Subscription)을 MapSubscription으로 감싸서 전달
        FilterSubscriber(Subscriber).onSubscribe(new MapSubscription(JustSubscription(Subscription))) {
            // MapSubscription(JustSubscription(Subscription))을 FilterSubscription으로 감싸서 전달
            Subscriber.onSubscribe(new FilterSubscription(MapSubscription(JustSubscription(Subscription)))) {
                // 실제 요청 데이터
            }
        }
    }
}
```

* 위 수도 코드에서 Subscription 형태만 살펴봅시다.

```text
FilterSubscription(
    MapSubscription(
        JustSubscription(
            Subscription
        )
    )
)
```

* 조립 단계와 피라미드 구조가 유사합니다. (구독 단계와는 역피라미드 형태)
* 이제 실제 데이터를 요청하는 request() 시그널을 살펴봅시다. (수도 코드)

```text
FilterSubscription(MapSubscription(JustSubscription(Subscription))).request(10) {
    MapSubscription(JustSubscription(Subscription)).request(10) {
        JustSubscription(Subscription).request(10) {
            Subscription.request(10) {
                // 실제 데이터를 요청
            }
        }
    }
}
```

* 실제 Subscriber에게 데이터가 전달되는 과정을 살펴봅시다. (수도 코드)

```text
Subscription.request(10) {
    JustSubscriber(MapSubscriber(FilterSubscriber(Subscriber))).onNext(...) {
          MapSubscriber(FilterSubscriber(Subscriber)).onNext(1) { // 1, 20, 300, 4000 순으로 onNext()
              // 숫자 1이 문자열 "1"로 변환
              FilterSubscriber(Subscriber).onNext("1") {
                  // 한글자이기 때문에 걸러짐 -> 추가 데이터 요청
                  MapSubscription(JustSubscription(Subscription)).request(1) {...}
              }
          }
          MapSubscriber(FilterSubscriber(Subscriber)).onNext(20) {
              // 숫자 20이 문자열 "20"로 변환
              FilterSubscriber(Subscriber).onNext("20") {
                  // 두글자이기 때문에 걸러지지 않음
                  Subscriber.onNext("20") {...} // 다음 구독자에게 전달 (여기에서는 실제 구독자에게 전달)
              }
          }
    }
}
```

* 데이터는 소스로부터 각 Subscriber 체인을 거쳐 단계마다 다른 기능을 수행하게 됩니다.
    * MapSubscriber는 데이터를 변환합니다.
    * FilterSubscriber는 데이터를 걸러냅니다. (걸러내지 않으면 다음 Subscriber에게 전달, 걸러낸다면 추가 요청)
* 위와 같은 조립 단계, 구독 단계, 런타임을 거쳐서 데이터가 전달되는 과정을 통해 리액티브 스트림의 동작 방식을 이해할 수 있기를 바랍니다.