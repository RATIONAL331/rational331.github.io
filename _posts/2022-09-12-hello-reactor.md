---
title:  "리액터에 관하여"
excerpt: "리액티브 스트림을 구현한 리액터에 관하여 알아봅시다."
category: "reactive"

last_modified_at: 2022-09-12T
---

# 리액터

## 리액터 프로젝트

* https://projectreactor.io/
* 리액터는 리액티브 스트림을 구현한 라이브러리입니다..
* 리액터의 기본 목표는 비동기 파이프라인을 구축할 때 콜백 지옥과 깊게 중첩된 코드를 생략하는 목적으로 설계되었습니다.
* 리액터는 ```연산자를 연결해서``` 사용하는 것을 권장합니다.
    * ![hello_reactor_01_01.png](/assets/images/hello_reactor_01/hello_reactor_01_01.png)

## Mono

* Mono는 0 또는 1개의 데이터를 emit(방출)하는 Publisher입니다.

|             |      ```Object```      |
|:-----------:|:----------------------:|
|    Java     |       ```null```       | 
| Java Stream | ```Optional<Object>``` |
|   Reactor   |   ```Mono<Object>```   |

### 잠깐 Stream을 잠시 살펴봅시다.

```java
class StreamExample {
	Stream<Integer> integerStream = Stream.of(1)
	                                      .map(i -> {
		                                      try {
			                                      Thread.sleep(1000);
		                                      } catch (InterruptedException e) {
			                                      throw new RuntimeException(e);
		                                      }
		                                      return i * 2;
	                                      });
}
```

* 위 코드는 1초를 기다리지 않습니다.
* ```integerStream.forEach(...)```과 같은 terminate operation을 호출해야지만 1초를 기다리게 됩니다.
* ```Stream```은 ```Lazy```한 특성을 가지고 있습니다.

### Mono 예제

#### just

```java
public class MonoJust {
	Mono<Integer> just = Mono.just(1);
	just.subscribe(i ->System.out.println("Received: "+i));
}
```

* Publisher에서도 Stream과 비슷한 규칙중 하나는 ```구독(subscribe)하기 전까지 아무일도 일어나지 않습니다.```
* 위 코드에서 ```just.subscribe(...)``` 를 호출해야지만 1이 출력됩니다.

#### subscribe

```java
class MonoSubscribe {
	void subscribe() {
		// 1. 단순 데이터 방출
		Mono<String> ball = Mono.just("ball");
		ball.subscribe(System.out::println);
		System.out.println("==============================");

		// 2. onNext, onError, onComplete
		// re-subscribe
		ball.subscribe(item -> System.out.println(item),
		               err -> System.out.println(err.getMessage()), // no error
		               () -> System.out.println("completed"));

		System.out.println("==============================");

		// 3. on error
		Mono<Integer> ball2 = Mono.just("ball2")
		                          .map(String::length)
		                          .map(len -> len / 0);
		ball2.subscribe(item -> System.out.println(item),
		                err -> System.out.println(err.getMessage()), // error
		                () -> System.out.println("completed"));

		System.out.println("==============================");

		// 4. explict empty
		Mono<String> empty = Mono.empty();
		empty.subscribe(item -> System.out.println(item),
		                err -> System.out.println(err.getMessage()), // no error
		                () -> System.out.println("completed"));

		System.out.println("==============================");

		// 5. explict error
		Mono<String> error = Mono.error(new IllegalStateException());
		error.subscribe(item -> System.out.println(item),
		                err -> System.out.println(err), // error
		                () -> System.out.println("completed"));
	}
}
```

``` text
ball
==============================
ball
completed
==============================
/ by zero
==============================
completed
==============================
ERROR: java.lang.IllegalArgumentException
```

* subscribe 함수는 3가지 파라미터를 받을 수 있습니다.
    * ```onNext```: 데이터를 받고 나서 호출
    * ```onError```: 에러 시그널을 발았을 때 호출
    * ```onComplete```: complete 시그널을 받았을 때 호출

#### fromSupplier, fromCallable [Supplier, Callable 차이][1]

```java
class MonoFromXXX {
	void fromXXX() {
		// 'just' 함수는 데이터가 이미 존재하는 상태일 때 쓰는 것이 유용합니다.
		Mono<String> just = Mono.just(getName());
		System.out.println("==============================");

		// 필요에 의해서 호출합니다.
		Mono<String> stringMono = Mono.fromSupplier(() -> getName()); // <- getName() 호출 안 됨
		System.out.println("==============================");

		Mono<String> stringMono2 = Mono.fromSupplier(() -> getName());
		stringMono2.subscribe(item -> System.out.println(item),
		                      err -> System.out.println(err.getMessage()),
		                      () -> System.out.println("completed"));
		System.out.println("==============================");

		Supplier<String> stringSupplier = () -> getName();
		Mono.fromSupplier(stringSupplier)
		    .subscribe(item -> System.out.println(item),
		               err -> System.out.println(err.getMessage()),
		               () -> System.out.println("completed"));
		System.out.println("==============================");

		Callable<String> stringCallable = () -> getName();
		Mono.fromCallable(stringCallable)
		    .subscribe(item -> System.out.println(item),
		               err -> System.out.println(err.getMessage()),
		               () -> System.out.println("completed"));
		System.out.println("==============================");
		System.out.println("################################");
		for (int i = 0; i < 5; i++) {
			getName(); // 단순히 파이프라인만 생성(시간 소요 매우 짧음)
		}
		System.out.println("==============================");

		getName().subscribe(item -> System.out.println(item)); // 구독하고 나서 3초 후에 출력
		System.out.println("==============================");

		getName().subscribeOn(Schedulers.boundedElastic()) // async
		         .subscribe(item -> System.out.println(item)); // 구독하고 나서 3초 후에 출력
		System.out.println("quickly already print");
		sleepSeconds(4); // 4초간 멈추기
		System.out.println("==============================");
	}

	private String getName() {
		System.out.println("Generate name..."); // 호출 주목
		return "Something";
	}

	private Mono<String> getMonoName() {
		System.out.println("Entered getMonoName method.");
		return Mono.fromSupplier(() -> {
			System.out.println("Generate name...");
			sleepSeconds(3); // 3초간 멈추기
			return "Something";
		}).map(String::toUpperCase);
	}
}
```

```text
Generate name...
==============================
==============================
Generate name...
Something
completed
==============================
Generate name...
Something
completed
==============================
Generate name...
Something
completed
==============================
################################
Entered getMonoName method.
Entered getMonoName method.
Entered getMonoName method.
Entered getMonoName method.
Entered getMonoName method. <- 5번이 매우 빠르게 호출
==============================
Entered getMonoName method.
Generate name... <- 이 곳에서 3초간 블록
SOMETHING
==============================
Entered getMonoName method.
quickly already print
Generate name... <- 이 곳에서 3초간 블록
SOMETHING
==============================
```

[1]: https://stackoverflow.com/questions/52215917/callable-vs-supplier-interface-in-java

* ```fromSupplier```, ```fromCallable```은 데이터를 생성하는 시점을 지연시킵니다.
* 구독이 해야지만 해당 파이프라인 내부를 수행하기 시작합니다. 구독이 이루어지지 않으면 아무런 동작도 하지 않습니다.
* ```fromFuture```, ```fromRunnable``` 등의 메소드도 있습니다.
    * ```fromFuture```
        * ```Mono.fromFuture(CompletableFuture.supplyAsync(() -> "Hello"));```
        * 마찬가지로 구독이 이루어지지 않으면 아무런 동작도 하지 않습니다.
        * fromFuture는 future가 시간이 걸리는 작업이더라도 바로 다음 줄을 수행합니다.

``` java
public class MonoFromFuture {
    void fromFuture() {
        Mono<String> fromFuture = Mono.fromFuture(getName());
        fromFuture.subscribe(item -> System.out.println(item));
        System.out.println("Already print"); <- 먼저 출력됨
        sleepSeconds(1); // <- 수행하지 않으면 name 출력 X
    }

    private static CompletableFuture<String> getName() {
        return CompletableFuture.supplyAsync(() -> {
            return "name";
        });
    }
}
```

* ```fromRunnable```
    * ```Mono.fromRunnable(() -> { System.out.println("Hello"); });```
    * ```Runnable``` 객체는 파라미터가 없고 리턴값이 없는 함수형 인터페이스입니다.
    * fromRunnable 과는 다르게 내부가 블록되면 구독을 하는 곳도 블록됩니다.

``` java
public class MonoFromRunnable {
    void fromRunnable() {
        Mono<Void> mono = Mono.fromRunnable(timeConsumingProcess());
        mono.subscribe(item -> System.out.println(item), // no data; no print
                       err -> System.out.println(err.getMessage()), // no error; no print
                       () -> System.out.println("process is done. notify emails."));
        // 데이터가 없기 때문에 onNext는 수행되지 않고, onComplete만 수행됩니다.
        // 또한 내부에서 3초 기다리는 동안 실제로 구독을 하는 곳도 블록됩니다.
        System.out.println("Print later"); // 구독이 끝나고 complete를 하고 나서야 출력됩니다.
    }

    private Runnable timeConsumingProcess() {
        return () -> {
            sleepSeconds(3);
            System.out.println("Process Done!");
        };
    }
}
```

## Flux

* Flux는 0 부터 N개의 데이터를 emit(방출)하는 Publisher입니다.

|             |     ```Object```     |
|:-----------:|:--------------------:|
|    Java     |  ```List<Object>```  |
| Java Stream | ```Stream<Object>``` |
|   Reactor   |  ```Flux<Object>```  |

### Flux 예제

#### just

```java
public class FluxJust {
	void just() {
		Flux<String> flux = Flux.just("A", "B", "C");
		flux.subscribe(item -> System.out.println(item),
		               err -> System.out.println(err.getMessage()),
		               () -> System.out.println("Done"));
	}
}
```

* ```Mono```, ```Flux``` 모두 ```Publisher```이기 때문에, 대부분의 메소드는 Mono와 유사한점들이 많다.
* Mono를 사용하는 이유는 둘 이상의 결과를 기대하지 않는 것이 명확해집니다.

#### 여러 개의 구독자

``` java
class MultipleSubscriber {
    void multipleSubscirber() {
        Flux<Integer> just = Flux.just(1, 2, 3, 4, 5);
        Flux<Integer> even = just.filter(i -> i % 2 == 0); // filter로 감싸진 publisher 리턴
        just.subscribe(i -> System.out.println("Subscriber 1: " + i));
        just.subscribe(i -> System.out.println("Subscriber 2: " + i)); // re-subscribe
        even.subscribe(i -> System.out.println("Subscriber 3: " + i));
    }
}
```

``` text
Subscriber 1: 1
Subscriber 1: 2
Subscriber 1: 3
Subscriber 1: 4
Subscriber 1: 5
Subscriber 2: 1
Subscriber 2: 2
Subscriber 2: 3
Subscriber 2: 4
Subscriber 2: 5
Subscriber 3: 2
Subscriber 3: 4
```

* operator를 적용하면 publisher가 변경되는 것이 아닌 새로운 Publisher를 리턴합니다.
* 구독을 다시 하면 Publisher는 이벤트를 처음부터 다시 발생을 시도합니다.

#### fromStream

``` java
class FluxFromStream {
    void fluxFromStream() {
        List<Integer> integers = List.of(1, 2, 3, 4, 5);
        Stream<Integer> stream = integers.stream();
        stream.forEach(System.out::println); // 스트림이 닫힙니다.
//        stream.forEach(System.out::println); // IllegalStateException: stream has ALREADY been operated upon or closed

        List<Integer> integerList = List.of(1, 2, 3, 4, 5);
        Stream<Integer> integerStream = integerList.stream();
        Flux<Integer> integerFlux = Flux.fromStream(integerStream);
        integerFlux.subscribe(i -> System.out.println("Subscriber 1: " + i)); // stream closed
//        integerFlux.subscribe(i -> System.out.println("Subscriber 2: " + i)); // IllegalStateException: stream has ALREADY been operated upon or closed

        Flux<Integer> reusableFlux = Flux.fromStream(() -> integerList.stream());
        reusableFlux.subscribe(i -> System.out.println("Subscriber 1: " + i));
        reusableFlux.subscribe(i -> System.out.println("Subscriber 2: " + i));
    }
}
```

* ```Stream```의 성질 중 하나는 한번 사용하면 닫힙니다. (재사용 불가)
* 따라서 ```fromStream(Stream<? extends T> s)``` 를 사용하면 구독하고 나서 s는 재사용 불가합니다.
* 그러므로 ```fromStream(Supplier<Stream<? extends T>> streamSupplier)``` 를 사용하면 구독할 때마다 새로운 ```Stream```을 생성하므로 재구독에 안전합니다.

#### fromIterable, fromArray

``` java
class FluxFromXXX {
    void fluxFromXXX() {
        List<String> strings = Arrays.asList("a", "b", "c", "d", "e");
        Flux.fromIterable(strings)
            .subscribe(System.out::println);

        Integer[] arr = {1, 2, 3, 4, 5};
        Flux.fromArray(arr)
            .subscribe(System.out::println);
    }
}
```

#### range

```java
class FluxRange {
	void fluxRange() {
		Flux<Integer> range = Flux.range(3, 5)
		                          .log();

		range.map(i -> "Received: " + i) // subscribe too
		     .log()
		     .subscribe(System.out::println,
		                System.out::println,
		                () -> System.out.println("Completed"));

		System.out.println("==============================================================");

		Flux.range(3, 1000)
		    .map(i -> i / (i - 4)) // 4 divide by 0 and 5 not emit
		    .subscribe(System.out::println,
		               System.out::println,
		               () -> System.out.println("Completed"));
	}
}
```
``` text
[ INFO] (main) | onSubscribe([Synchronous Fuseable] FluxRange.RangeSubscription)
[ INFO] (main) | onSubscribe([Fuseable] FluxMapFuseable.MapFuseableSubscriber)
[ INFO] (main) | request(unbounded)
[ INFO] (main) | request(unbounded)
[ INFO] (main) | onNext(3)
[ INFO] (main) | onNext(Received: 3)
Received: 3
[ INFO] (main) | onNext(4)
[ INFO] (main) | onNext(Received: 4)
Received: 4
[ INFO] (main) | onNext(5)
[ INFO] (main) | onNext(Received: 5)
Received: 5
[ INFO] (main) | onNext(6)
[ INFO] (main) | onNext(Received: 6)
Received: 6
[ INFO] (main) | onNext(7)
[ INFO] (main) | onNext(Received: 7)
Received: 7
[ INFO] (main) | onComplete()
[ INFO] (main) | onComplete()
Completed
==============================================================
-3
java.lang.ArithmeticException: / by zero
```

* range 함수는 (시작값, 개수)를 받습니다.
  * (3, 5)라면 3, 4, 5, 6, 7을 발행합니다.
* log 함수는 subscriber가 어떤식으로 처리하고 있는지 보여줍니다.
* Flux가 발행도중 에러가 발생한다면 에러를 출력하고 종료합니다.
  * onComplete는 출력되지 않습니다.

#### interval
```java
class FluxInterval {
    void interval() {
        Flux.interval(Duration.ofSeconds(1)) // infinite
            .subscribe(System.out::println);
		sleepSeconds(5);
    }
}
```
* interval 함수는 0부터 명시한 시간동안 1씩 증가하는 값을 발행합니다.

#### from, next
```java
class FluxFromNext {
    void fromNext() {
        Mono<String> mono = Mono.just("a");
        Flux<String> from = Flux.from(mono); // mono to flux
        from.subscribe(System.out::println);

        System.out.println("==============================================================");

        Flux<Integer> range = Flux.range(1, 10);
        range.next() // flux to mono (first item)
             .subscribe(System.out::println);
    }
}
```
* from 함수는 Publisher를 Flux로 반환합니다. (위의 예제에서는 Mono를 Flux로 변환)
* next 함수는 Flux의 첫번째 요소를 Mono로 반환합니다. (Flux가 비어있는 경우, Mono역시 비어있습니다.)
  * single 과 유사하지만 single은 반드시 하나가 존재해야하며, 비어있는 Publisher일 경우 에러를 발생시킵니다.

#### subscribeWith
```java
class FluxSubscribeWith {
    void subscribeWith() {
        AtomicReference<Subscription> subscriptionAtomicReference = new AtomicReference<>();
        Flux.range(1, 20)
            .log()
            .subscribeWith(new Subscriber<Integer>() {
                // publisher give subscription to subscriber
                @Override
                public void onSubscribe(Subscription subscription) {
                    System.out.println("Receiving subscription: " + subscription);
                    subscriptionAtomicReference.set(subscription); // subscription을 바깥에서 확인하기 위해 저장
                }

                @Override
                public void onNext(Integer integer) {
                    System.out.println("Receiving next: " + integer);
                }

                @Override
                public void onError(Throwable throwable) {
                    System.out.println("Receiving error: " + throwable);
                }

                @Override
                public void onComplete() {
                    System.out.println("Receiving complete");
                }
            });
        Subscription subscription = subscriptionAtomicReference.get();
        subscription.request(2);
        System.out.println("============================================================");
        sleepSeconds(1);
        System.out.println("subscription.cancel()");
        subscription.cancel();
        System.out.println("============================================================");
        sleepSeconds(1);
        subscription.request(1);
        System.out.println("nothing happen");
    }
}
```
```text
[ INFO] (main) | onSubscribe([Synchronous Fuseable] FluxRange.RangeSubscription)
Receiving subscription: reactor.core.publisher.StrictSubscriber@2eafffde
[ INFO] (main) | request(2)
[ INFO] (main) | onNext(1)
Receiving next: 1
[ INFO] (main) | onNext(2)
Receiving next: 2
============================================================
subscription.cancel()
[ INFO] (main) | cancel()
============================================================
nothing happen
```
* subscribeWith 함수는 Subscriber를 인자로 받아서 처리합니다.
* 위 예제에서는 Publisher가 전달한 Subscription을 바깥에서 확인할 수 있게끔 하였습니다.
* subscription.request(2)를 통해 Publisher에게 2개의 요소를 요청하여 전달받았습니다.
* subscription.cancel()을 통해 Publisher에게 더이상 요소를 요청하지 않겠다고 알렸습니다.
* subscription.request(1)을 통해 Publisher에게 요소를 요청하였지만, 이미 cancel을 통해 요청을 거부했기 때문에 아무런 동작이 일어나지 않았습니다.

#### Flux vs List
```java
class FluxVsList {
    void fluxVsList() {
        List<String> names = getNames(5); // 5초 후에 한꺼번에 반환
        System.out.println(names);

        Flux<String> nameFlux = getNameFlux(5); // 결과가 생성되는대로 반환
        nameFlux.subscribe(System.out::println);
    }

    private List<String> getNames(int count) {
        return IntStream.range(0, count)
                        .mapToObj(FluxVsList::getName)
                        .collect(Collectors.toList());
    }

    private Flux<String> getNameFlux(int count) {
        return Flux.range(0, count)
                   .map(FluxVsList::getName);
    }

    private String getName(long i) {
        System.out.println("Generate name...");
        sleepSeconds(1);
        return "name" + i;
    }
}
```
```text
Generate name... (1초에 한번씩)
Generate name... (1초에 한번씩)
Generate name... (1초에 한번씩)
Generate name... (1초에 한번씩)
Generate name... (1초에 한번씩)
[name0, name1, name2, name3, name4] (5초에 한꺼번에 반환)
Generate name... (1초에 한번씩)
name0 (1초에 한번씩)
Generate name... (1초에 한번씩)
name1 (1초에 한번씩)
Generate name... (1초에 한번씩)
name2 (1초에 한번씩)
Generate name... (1초에 한번씩)
name3 (1초에 한번씩)
Generate name... (1초에 한번씩)
name4 (1초에 한번씩)
```

* Flux의 경우 결과가 생성되는데로 반환합니다.
* List의 경우에는 생성되는데로 반환하는 것이 아니라, 모든 결과가 생성되면 한꺼번에 반환합니다.