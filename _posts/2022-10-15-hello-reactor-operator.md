---
title:  "리액터 연산자들"
excerpt: "연산자 그리고 퍼블리셔 합쳐보기"
category: "reactive"

last_modified_at: 2022-10-16T
---

# 리액터 연산자들

* 모든 연산자들을 다룰 수는 없습니다.
* 많이 사용하는 연산자들에 대해서만 살펴봅시다.
* 더욱 많은 연산자들은 아래를 참고하세요.
* https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html
* https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html
* https://projectreactor.io/docs/core/release/reference/#which-operator

## map

* 각각의 데이터에 동기 함수를 적용하여 각각의 데이터를 변환합니다.
* map체이닝이 많아지면 동기 함수를 여러번 적용하여 오버헤드가 발생할 수 있습니다.
* map을 여러개 사용하면 IDE에서 알려줍니다.
* map을 여러번 사용하는 대신 flatMap을 사용하는 것이 좋습니다.
  * https://youtu.be/I0zMm6wIbRI?t=1074 (NHN Forward 2020 - NHN GLOBAL 김병부 수석님)
* ![hello_reactor_05_01.png](/assets/images/hello_reactor_05/hello_reactor_05_01.png)

## flatMap

* 각 데이터에 비동기 함수를 적용하여 각각의 퍼블리셔로 변환합니다. 그 후 평탄화 작업을 진행하여 하나의 퍼블리셔로 만듭니다.
* 이 때 끼워넣기가 허용됩니다. (interleave)
    * 따라서 원래 순서가 유지되지 않을 수 있습니다.

## flatMapSequential

* flatMap과 유사하나 끼워넣기가 허용되지 않습니다.
  * 따라서 원래 순서를 유지합니다.
  * 순서를 유지하기 위하여 내부적으로 큐를 사용합니다.

## concatMap

* flatMapSequential과 유사하여 원래 순서를 유지합니다. (끼워넣기 허용 안함)
* flatMap, flatMapSequential과 결정적으로 다른 부분은 하나의 데이터가 완전히 처리될 때 까지 기다립니다. (속도면에서 불리합니다.)
  * flatMap, flatMapSequential은 하나의 데이터가 완전히 처리되지 않아도 다음 데이터를 처리합니다. (eagerly subscribing)
  * concatMap은 첫번째 데이터가 완전히 처리되어 complete 시그널이 도착해야 그 다음 두번째 데이터가 방출되어 처리되기 시작합니다. (순차 변환)
  * flatMapSequential은 순서를 유지하기 위하여 내부적으로 큐를 사용했지만, concatMap은 그럴 필요가 없습니다.

```java
class FlatConcatMap {
	void flatConcatMap() {
		Flux.just(1, 2, 3)
		    .flatMap(this::doSomethingAsync)
		    .doOnNext(n -> log.info("Done {}", n))
		    .blockLast();

		/**
		 * main - Executing 1
		 * main - Done 1
		 * main - Executing 2
         * main - Executing 3
         * main - Done 3 // 3이 먼저 나오게 됨(순서 보장X)
         * parallel-1 - Done 2 // delayElement를 사용하게 되면 다른 쓰레드로 수행되게 전환됩니다..
		 */

		Flux.just(1, 2, 3)
		    .flatMapSequential(this::doSomethingAsync)
		    .doOnNext(n -> log.info("Done {}", n))
		    .blockLast();

		/**
		 * main - Executing 1
		 * main - Done 1
		 * main - Executing 2
		 * main - Executing 3
         * parallel-1 - Done 2 // delayElement를 사용하게 되면 다른 쓰레드로 수행되게 전환됩니다.
         * parallel-1 - Done 3 // 3은 2다음 나옴(순서 보장)
         */

      Flux.just(1, 2, 3)
          .concatMap(this::doSomethingAsync)
          .doOnNext(n -> log.info("Done {}", n))
          .blockLast();

      /**
       * main - Executing 1
       * main - Done 1
       * main - Executing 2
       * parallel-1 - Done 2 // 2번이 끝나야 
       * parallel-1 - Executing 3 // 3번이 방출됩니다.
       * parallel-1 - Done 3 
       */

    }

  private Mono<Integer> doSomethingAsync(Integer number) {
    // 두번째 아이템은 1초 지연되어 방출됩니다.
    return number == 2 ? Mono.just(number).doOnNext(n -> log.info("Executing {}", n)).delayElement(Duration.ofSeconds(1))
            : Mono.just(number).doOnNext(n -> log.info("Executing {}", n));
  }
}
```

## switchMap

* 데이터가 방출될 될때마다 기존 변환을 취소하고 그 다음 데이터를 퍼블리셔로 변환합니다.
* 항상 마지막 데이터는 온전하게 처리되는 것을 보장해야할 때 사용할 수 있습니다.
* switchMap이 적절한 예제를 다음 주소에서 확인할 수 있습니다.
  * https://medium.com/@elizabethveprik/rxjava-flatmap-vs-switchmap-85cd7e2c791c
  * 기존 데이터 변환에 더 이상 신경쓰지 않아야 하는 상황에서 유용하게 사용할 수 있습니다.
* 또 다른 예제로 포털사이트에서 검색하는 시나리오가 있습니다.
* ![hello_reactor_05_01.png](/assets/images/hello_reactor_05/hello_reactor_05_02.png)
* 검색어를 입력하면 검색어를 포함하는 데이터를 검색하는 API를 호출합니다.
* 이 때 어떤 한 응답이 늦게 도착하는 상황이라면 어떻게 될까요?
  * 엉뚱한 결과가 리스트에 늦게 반영이 될 수 있습니다.
  * 해당 현상을 막기 위해 switchMap을 사용할 수 있습니다.

```java
class SwitchMap {
	void switchMap() {
		Flux.just(1, 2, 3)
		    .switchMap(this::doSomethingAsync)
		    .doOnNext(n -> log.info("Done {}", n))
		    .blockLast();

		/**
		 * main - Executing 1
		 * main - Done 1
		 * main - Executing 2
		 * main - Executing 3
		 * main - Done 3 // 2번은 처리될 동안 3번 데이터가 방출되어 2번 데이터 변환이 취소되어 나오지 않습니다.
		 */
	}

	private Mono<Integer> doSomethingAsync(Integer number) {
		// 두번째 아이템은 1초 지연되어 방출됩니다.
		return number == 2 ? Mono.just(number).doOnNext(n -> log.info("Executing {}", n)).delayElement(Duration.ofSeconds(1))
				: Mono.just(number).doOnNext(n -> log.info("Executing {}", n));
	}
}
```

## transform

* 퍼블리셔를 다른 퍼블리셔로 변환합니다.

```java
class Transform {
	void transform() {
		getXXX().transform(applyFilterMap()) // 퍼블리셔를 다른 퍼블리셔로 변환합니다. (flux -> mono)
		        .subscribe(...);


		static Flux<String> getXXX () {
			return Flux.range(1, 10)
			           .map(i -> ...); // 문자열로 변환
		}

		static Function<Flux<Person>, Mono<Integer>> applyFilterMap () {
			return flux -> flux.filter(...).map(...).next(); // 퍼블리셔(Flux<String>)를 다른 퍼블리셔(Mono<Integer>)로 변환합니다.
		}
	}
}
```

## defaultIfEmpty

* 퍼블리셔가 비어있을 경우(아무런 데이터를 방출하지 못하고 complete 하는 경우) 기본값을 방출합니다.

## switchIfEmpty

* 퍼블리셔가 비어있을 경우(아무런 데이터를 방출하지 못하고 complete 하는 경우) 다른 퍼블리셔를 방출합니다.

```java
class IfEmpty {
	void ifEmpty() {
		Flux.range(1, 10)
		    .filter(i -> i > 10) // empty
		    .defaultIfEmpty(-100) // 기본 '값'을 설정.
		    .subscribe(...);

		/**
		 * onNext: -100
		 * onComplete
		 */

		Flux.range(1, 10)
		    .filter(i -> i > 10) // empty
		    .switchIfEmpty(Flux.range(20, 5)) // 기본 '퍼블리셔'를 설정.
		    .subscribe(...);

		/**
		 * onNext: 20
		 * onNext: 21
		 * onNext: 22
		 * onNext: 23
		 * onNext: 24
		 * Completed
		 */
	}
}
```

## handle

* 방출되는 데이터를 직접 제어할 수 있습니다.

```java
class Handle {
	void handle() {
		Flux.range(1, 5)
		    .handle(((integer, synchronousSink) -> {
			    if (integer % 2 == 0) {
				    synchronousSink.next(integer);
			    } else {
				    synchronousSink.next(integer + " is odd"); // map
			    }
		    }))
		    .subscribe(...);
		/**
		 * Received: 1 is odd
		 * Received: 2
		 * Received: 3 is odd
		 * Received: 4
		 * Received: 5 is odd
		 * Completed
		 */

		Flux.range(1, 5)
		    .handle(((integer, synchronousSink) -> {
			    if (integer == 4) synchronousSink.complete(); // until
			    else synchronousSink.next(integer);
		    }))
		    .subscribe(...);

		/**
		 * Received: 1
		 * Received: 2
		 * Received: 3
		 * Completed
		 */
	}
}
```

* handle 내부에서 synchronousSink.next()는 최대 한번만 불려야합니다.
* handle 내부에서 synchronousSink.complete()를 호출하면 더이상 데이터를 방출하지 않습니다.

## do...

* doFirst: 조립 단계 후에(구독 단계에서) 처음으로 호출됩니다.
* doFinally: 어떠한 이유로 끝나면 호출됩니다. (에러, 완료, 취소)
* doOnSubscribe: 런타임 단계에서 구독자가 퍼블리셔에게 실제로 구독을 할 때 호출됩니다.
* doOnNext: 런타임 단계에서 데이터가 발행될 때 호출됩니다.
* doOnComplete: 퍼블리셔가 완료될 때 호출됩니다.
* doOnCancel: 구독이 취소될 때 호출됩니다.
* doOnDiscard: 구독자가 데이터를 받지 않고 버릴 때 호출됩니다.

```java
class DoCallback {
	void doCallback() {
		Flux.create(fluxSink -> {
			    System.out.println("inside create");
			    for (int i = 0; i < 5; i++) {
				    fluxSink.next(i);
			    }
			    fluxSink.complete();
			    System.out.println("create completed");
		    })
		    .doFirst(() -> System.out.println("doFirst 1"))
		    .doFinally(signal -> System.out.println("doFinally 1" + signal)) // cancel
		    .doFirst(() -> System.out.println("doFirst 2"))
		    .doOnComplete(() -> System.out.println("doOnComplete1"))
		    .doOnNext(i -> System.out.println("doOnNext: " + i))
		    .doOnSubscribe(subscription -> System.out.println("doOnSubscribe 1" + subscription))
		    .doFirst(() -> System.out.println("doFirst 3"))
		    .doOnSubscribe(subscription -> System.out.println("doOnSubscribe 2" + subscription)) //
		    .doOnCancel(() -> System.out.println("doOnCancel"))
		    .doFinally(signal -> System.out.println("doFinally 2" + signal)) // cancel
		    .doOnDiscard(Object.class, o -> System.out.println("doOnDiscard " + o))
		    .take(2) // doOnDiscard 2, 3, 4
		    .doOnComplete(() -> System.out.println("doOnComplete2"))
		    .doFinally(signal -> System.out.println("doFinally 3" + signal)) // complete
		    .subscribe(...);

		/**
		 * doFirst 3 (doFirst는 런타임 단계에서 수행되지 않습니다; 역순으로 수행됩니다.)
		 * doFirst 2
		 * doFirst 1
		 * doOnSubscribe 1reactor.core.publisher.FluxPeekFuseable$PeekConditionalSubscriber@7ff2a664 (doOnSubscribe는 런타임 단계에서 수행되어 정방향으로 수행됩니다.)
		 * doOnSubscribe 2reactor.core.publisher.FluxPeekFuseable$PeekConditionalSubscriber@525b461a
		 * inside create
		 * doOnNext: 0
		 * Received: 0
		 * doOnNext: 1
		 * Received: 1
		 * doOnCancel
		 * doFinally 1cancel
		 * doFinally 2cancel
		 * doOnComplete2 // doOnComplete1은 호출되지 않은 것에 주목하세요. (take(2) 때문에 수행될 수 없습니다.)
		 * Completed
		 * doFinally 3onComplete
		 * doOnDiscard 2
		 * doOnDiscard 3
		 * doOnDiscard 4
		 * create completed
		 */
	}
}
```

## limitRate

* 구독자가 퍼블리셔에게 요청하는 데이터의 개수를 제한합니다.
* limitRate(n)을 설정하면 맨 처음 n만큼 데이터를 요청하고 나서 그 다음부터는 replenishing optimization에 의하여 n의 75%만큼 요청합니다.
* limitRate(high, low)를 설정하면 맨 처음 high만큼 데이터를 요청하고 나서 그 다음부터는 low만큼 요청합니다.

```java
class LimitRate {
	void limitRate() {
		Flux.range(1, 1000)
		    .log()
		    .limitRate(100) // constantly 75%
		    //.limitRate(100, 99) // 99%
		    //.limitRate(100, 100) // 75% => 만약 low가 high보다 크거나 같다면, limitRate(n)과 동일합니다.
		    //.limitRate(100, 0) // 100% => low값이 0보다 작거나 같으면 항상 high만큼 요청합니다.
		    .subscribe(...);

		/**
		 * request(100) // 맨 처음에 100개 요청
		 * onNext(1)
		 * Received: 1
		 * onNext(2)
		 * Received: 2
		 * ...
		 * onNext(75)
		 * received: 75 // 75%까지 소비 후 그 다음 75% 요청
		 * request(75)
		 * onNext(76)
		 * Received: 76
		 * ...
		 * request(75)
		 * onNext(976)
		 * Received: 976
		 * ...
		 * onNext(1000)
		 * Received: 1000
		 * onComplete()
		 */
	}
}
```

## delay

* 각각의 데이터를 방출할 때 지연시간을 설정합니다.
* 이 때 기본적으로 parallel 스케쥴러에서 수행됩니다.
* 비어있거나, 에러가 전달되면 지연되지 않습니다.
* 기본적으로 버퍼 사이즈는 32입니다.

```java
package reactor.util.concurrent

public final class Queues {
	public static final int CAPACITY_UNSURE = Integer.MIN_VALUE;
	public static final int XS_BUFFER_SIZE = Math.max(8, Integer.parseInt(System.getProperty("reactor.bufferSize.x", "32"))); // <- delay의 기본 버퍼 사이즈
	public static final int SMALL_BUFFER_SIZE = Math.max(16, Integer.parseInt(System.getProperty("reactor.bufferSize.small", "256")));
}
```

```java
class Delay {
	void delay() {

		Flux.range(1, 100)
		    .log()
		    .delayElements(Duration.ofMillis(100)) // 100ms 간격으로 데이터를 방출합니다. parallel 스케쥴러에서 수행됩니다.
		    .subscribe(...);

		/**
		 * request(32) // <- 맨 처음 32개 요청
		 * onNext(1)
		 * ..
		 * onNext(32)
		 */
		sleepSeconds(20);
		/**
		 * received: 1
		 * received: 2
		 * ...
		 * received: 23
		 * request(24) [주목!] <- replenishing optimization에 의해 24(32의 75%)개 요청
		 * onNext(33)
		 * ...
		 * onNext(56)
		 * received: 24
		 * ...
		 * received: 47
		 * request(24) [주목!] <- replenishing optimization에 의해 24(32의 75%)개 요청
		 * onNext(57)
		 * ...
		 * received: 48
		 * ...
		 */


	}
}
```

# Publisher 합치기

* 역시 모든 것을 다룰 수는 없습니다.
* 많이 사용하는 것들에 대해서만 살펴봅시다.

## startWith

* 퍼블리셔가 데이터를 방출하기 전에 데이터를 추가합니다.

```java
class StartWith {
  public startWith() {
    Flux<String> flux = Flux.just("a", "b", "c")
                            .delayElements(Duration.ofMillis(500));

    Flux.just("a2", "b2", "c2")
        .startWith(flux)
        .subscribe(System.out::println);

    /**
     * a
     * b
     * c
     * a2
     * b2
     * c2
     */

    sleepSeconds(2);
  }
}
```

* startWith는 Iterable, T..., Publisher를 인자로 받을 수 있습니다.
* Publisher로 인자로 받을 때에는, 인자로 넘어온 Publisher가 데이터를 모두 방출하고 나서야 데이터를 방출합니다.

## concat

* 퍼블리셔를 구독한 다음 완료될 때 까지 기다린 후 완료되면 그 다음 퍼블리셔를 순차적으로 구독하여 연결합니다.
  * 따라서 무한의 퍼블리셔가 concat 앞에 존재하면 그 다음 퍼블리셔가 구독이 되지 않습니다.

```java
class Concat {
  public concat() {
    Flux<Long> just = Flux.just(-1L, -2L, -3L);
    Flux.interval(Duration.ofMillis(500)) // interval은 인자로 넘긴 시간 간격으로 0부터 1씩 증가하는 데이터를 방출합니다.
        .concatWith(just) // concatWith는 해당 인자를 다음 퍼블리셔로 연결합니다.
        .subscribe(s -> System.out.println("Received: " + s));
    sleepSeconds(5);

    /**
     * Received: 0
     * Received: 1
     * Received: 2
     * Received: 3
     * Received: 4 
     * ... // -1, -2, -3은 절대로 못나옴
     */

    Flux<String> pub1 = Flux.just("a", "b", "c");
    Flux<String> pub2 = Flux.just("d", "e", "f");
    Flux<String> error = Flux.error(new RuntimeException());

    Flux<String> stringFlux = Flux.concat(pub1, pub2, error);

    stringFlux.subscribe(...);

    /**
     * subscriber1; Received: a
     * subscriber1; Received: b
     * subscriber1; Received: c
     * subscriber1; Received: d
     * subscriber1; Received: e
     * subscriber1; Received: f
     * subscriber1; Error: java.lang.RuntimeException
     */

    Flux<String> stringFlux2 = Flux.concat(pub1, error, pub2); // 에러가 중간에 발생하면 그 다음 퍼블리셔는 구독되지 않습니다.
    stringFlux2.subscribe(...);
    /**
     * subscriber2; Received: a
     * subscriber2; Received: b
     * subscriber2; Received: c
     * subscriber2; Error: java.lang.RuntimeException
     */

    Flux<String> stringFlux3 = Flux.concatDelayError(pub1, error, pub2); // 에러가 발생해도 그 다음 퍼블리셔를 구독합니다. (에러를 늦춤)
    stringFlux3.subscribe(...);

    /**
     * subscriber3; Received: a
     * subscriber3; Received: b
     * subscriber3; Received: c
     * subscriber3; Received: d <- [주목!] 에러가 발생해도 그 다음 퍼블리셔를 구독합니다.
     * subscriber3; Received: e
     * subscriber3; Received: f
     * subscriber3; Error: java.lang.RuntimeException
     */
  }
}
```

* concat, concatWith는 Publisher를 순차적으로 연결합니다.
  * 이 때 각각의 구독이 완료되고 나서야 그 다음 퍼블리셔를 순차적으로 구독합니다.
  * 에러가 발생하면 그 다음 퍼블리셔는 구독되지 않습니다.
  * concatDelayError는 에러가 발생해도 그 다음 퍼블리셔를 구독합니다.

## merge

* concat과 유사하나 가장 크게 다른 점은 각각의 구독자를 모두 구독합니다. (eagerly subscribe)
  * concat은 구독자가 각각의 구독을 완료하고 나서야 다음 퍼블리셔를 순차적으로 구독합니다.

```java
class Merge {
  void merge() {
    Flux<Long> just = Flux.just(-1L, -2L, -3L);
    Flux.interval(Duration.ofMillis(500)) // interval은 인자로 넘긴 시간 간격으로 0부터 1씩 증가하는 데이터를 방출합니다.
        .mergeWith(just) // concat -> merge로만 바뀌었습니다.
        .subscribe(s -> System.out.println("Received: " + s));
    sleepSeconds(5);
  }

  /**
   * Received: -1
   * Received: -2
   * Received: -3
   * Received: 0 // 500ms 후에 나오기 때문에 이 전에 -1, -2, -3이 먼저 나옵니다.
   * Received: 1
   * Received: 2
   * Received: 3
   * Received: 4 
   * ...
   */
}
```

## zip

* 여러개의 Publisher를 하나로 합쳐서 하나의 Publisher로 만듭니다.
  * 각각의 Publisher가 방출하는 데이터를 하나로 합쳐서 방출합니다.
  * 각각의 Publisher가 방출하는 데이터의 개수가 다르면 더 적은 데이터를 가진 Publisher가 방출하는 데이터의 개수만큼만 방출합니다.

```java
class Zip {
  void zip() {
    Flux.zip(getBody(), getTire(), getEngine())
        .subscribe(...); // 총 2개의 데이터를 방출합니다.

    sleepSeconds(10);

    private static Flux<String> getBody () {
      return Flux.range(1, 5).delayElements(Duration.ofMillis(500)).map(i -> "body" + i);
    }

    private static Flux<String> getEngine () {
      return Flux.range(1, 2).delayElements(Duration.ofMillis(1000)).map(i -> "engine" + i);
    }

    private static Flux<String> getTire () {
      return Flux.range(1, 6).delayElements(Duration.ofMillis(1500)).map(i -> "tire" + i);
    }
  }
  /**
   * subscriber; Received: [body1,tire1,engine1]
   * subscriber; Received: [body2,tire2,engine2]
   * subscriber; Completed
   */
}
```

## combineLatest

* 여러개의 Publisher를 하나로 합쳐서 하나의 Publisher로 만든다는 면에서 zip과 유사합니다.
* zip과 다른 점은 zip은 각각의 Publisher가 방출하는 데이터를 하나로 합쳐서 방출하지만 combineLatest는 각각의 Publisher가 방출하는 데이터 중 가장 최근에 방출한 데이터를 하나로
  합쳐서 방출합니다.
* 또한 zip은 최소의 데이터 개수를 가진 Publisher가 방출하는 데이터의 개수만큼만 방출하지만 combineLatest는 그렇지 않습니다.

```java
class Zip {
  void zip() {
    Flux.combineLatest(getBody(), getTire(), getEngine(), (all) -> all[0].toString() + all[1].toString() + all[2].toString())
        .subscribe(...);

    sleepSeconds(5);

    private static Flux<String> getBody () {
      return Flux.range(1, 5).delayElements(Duration.ofMillis(500)).map(i -> "body" + i);
    }

    private static Flux<String> getEngine () {
      return Flux.range(1, 2).delayElements(Duration.ofMillis(1000)).map(i -> "engine" + i);
    }

    private static Flux<String> getTire () {
      return Flux.range(1, 6).delayElements(Duration.ofMillis(1500)).map(i -> "tire" + i);
    }
  }
  /**
   * subscriber; Received: body2tire1engine1 // 1500ms
   * subscriber; Received: body3tire1engine1 // 1500ms
   * subscriber; Received: body3tire1engine2 // 2000ms
   * subscriber; Received: body4tire1engine2 // 2000ms
   * subscriber; Received: body5tire1engine2 // 2500ms
   * subscriber; Received: body5tire2engine2 // 3000ms
   * subscriber; Received: body5tire3engine2 // 4500ms
   */
}
```