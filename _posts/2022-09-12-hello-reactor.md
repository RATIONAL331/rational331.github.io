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
		Mono<String> ball1 = Mono.just("ball1");
		ball1.subscribe(item -> System.out.println(item),
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
ball1
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
			getName(); // 단순히 파이프라인만 생성(시간 소요 X)
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

* fromSupplier, fromCallable은 데이터를 생성하는 시점을 지연시킵니다.
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