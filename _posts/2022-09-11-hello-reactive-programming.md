---
title:  "리액티브 프로그래밍은 무엇인가?"
excerpt: "리액티브 프로그래밍은 기존과 무엇이 다른가?"
category: "reactive"

last_modified_at: 2022-09-11T
---

# 리액티브 프로그래밍

## 관찰자 패턴 [옵저버 패턴][1]

[1]: https://ko.wikipedia.org/wiki/%EC%98%B5%EC%84%9C%EB%B2%84_%ED%8C%A8%ED%84%B4

### 뜬금없이 갑자기 패턴?

* 리액티브 프로그래밍을 설명하기 전에 GoF 디자인 패턴 중 하나인 옵저버 패턴에 대해 간단히 설명하고 넘어가겠습니다.
* 관찰자 패턴은 관찰자(Observer)라고 불리는 자손의 리스트를 가지고 있는 주체(Subject)가 있고, 주체는 자신의 메서드 중 하나를 호출하여 관찰자에게 상태 변경을 알립니다.
* 이벤트 처리를 기반으로 시스템을 구현할 때 필수적이며, 대부분의 UI 라이브러리가 내부적으로 이 패턴을 자주 사용합니다.

```java
public interface Subject<T> {
	void registerObserver(Observer<T> o);

	void unregisterObserver(Observer<T> o);

	void notifyObservers(T event); // 주체가 관찰자에게 상태 변경을 알림 (구현하기 나름)
}

public interface Observer<T> {
	void observe(T event);
}
```

* 위의 코드 블럭은 관찰자 패턴을 구현하기 위해서 필요한 인터페이스입니다.

```java
public class SubjectImpl<T> implements Subject<T> {
	// 멀티 쓰레드 환경에서도 안전하게 사용할 수 있도록 동기화 처리
	private final Set<Observer<T>> observers = new CopyOnWriteArraySet<>();

	@Override
	public void registerObserver(Observer<T> o) {
		observers.add(o);
	}

	@Override
	public void unregisterObserver(Observer<T> o) {
		observers.remove(o);
	}

	@Override
	public void notifyObservers(T event) {
		observers.forEach(o -> o.observe(event)); // o.observe 하나를 호출하는데 너무 많은 시간이 걸린다면?
	}
}
```

* 만약에 ```notifyObservers```를 호출할 때 ```observe```라는 함수를 수행하는데 오래 걸린다면 어떻게 될까요?
* 구독자가 적으면 문제가 안되겠지만, 구독자가 많아지면 많아질수록 ```notifyObservers```를 한번 호출하면 너무 많은 시간이 소요될 것입니다.

```java
public class SubjectImpl2<T> implements Subject<T> {
	// 멀티 쓰레드 환경에서도 안전하게 사용할 수 있도록 동기화 처리
	private final Set<Observer<T>> observers = new CopyOnWriteArraySet<>();
	// 쓰레드 풀 생성
	private final ExecutorService executorService = Executors.newCachedThreadPool();

	//...

	@Override
	public void notifyObservers(T event) {
		// 쓰레드 풀을 사용하여 병렬로 이벤트를 전달하여 개선(?) 할 수 있습니다.
		observers.forEach(o -> executorService.submit(() -> o.observe(event)));
	}
}
```

* 여러분이 쓰레드 풀 크기를 깜빡하고 지정하지 않는다면 어떻게 될까요?
    * OutOfMemoryError 발생
* 쓰레드 풀 크기를 지정하였다고 한들, 쓰레드 풀 크기보다 더 많은 관찰자가 있으면 어떻게 될까요?

### jdk.util.Observable is [deprecated][2], jdk.util.Observer is [deprecated][3]

[2]: https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Observable.html

[3]: https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Observer.html

* jdk 1.0부터 제공되었던 유구한 역사를 지닌 인터페이스와 클래스입니다.
* 우선 위 코드처럼 제네릭 형태가 아닌 ```Object``` 타입을 사용하고 있습니다. (타입 안정성 보장 X)
* 관찰자와 주체 사이에 지원하는 이벤트 모델이 제한적입니다. (완료 및 에러 신호 전달 등...)
* 주체가 전달하는 이벤트의 순서를 지정할 수 없습니다.

## 발행-구독 패턴

* ![hello_reactive_02_01.png](/assets/images/hello_reactive_02/hello_reactive_02_01.png)

### @EventListener, ApplicationEventPublisher

* 스프링 프레임워크는 이벤트처리를 위한 적절한 어노테이션과 클래스를 제공합니다.
* 관찰자와 주체 사이에 직접적인 연결은 사라지고 그 사이에 간접적인 계층이 생겼습니다.
* 관찰자는 이벤트 채널을 알고 있지만, 주체가 누구인지 신경쓰지 않습니다.
* 이벤트 채널이 관찰자에게 이벤트를 전달하기 전에 필터링 작업을 진행할 수 도 있습니다.

#### 서버

```java

@Component
class TemperatureSensor {
	private final ApplicationEventPublisher publisher;
	private final ScheduledExecutorService executor = Executors.newSingleThreadScheduledExecutor();

	@PostConstruct
	public void startProcessing() {
		// 맨 처음에 1초 지연 후에 이벤트 발생 로직 수행
		executor.schedule(this::publishEvent, 1, TimeUnit.SECONDS);

	}

	private void publishEvent() {
		publisher.publishEvent(new Temperature(...)); // 이벤트 발생
		executor.schedule(this::publishEvent, randomSecond, TimeUnit.SECONDS); // 무작위 초 후에 이벤트 발생 로직 수행 예약
	}
}

@RestController
class TemperatureController {
	private final Set<SseEmitter> clients = new CopyOnWriteArraySet<>();

	@GetMapping("/temperature-stream")
	public SseEmitter events() { // 해당 메서드를 호출하면(GET 요청을 날리면) 클라이언트로 등록되고 Sse로 이벤트를 계속 받아볼 수 있다.
		SseEmitter emitter = new SseEmitter();
		clients.add(emitter);

		emitter.onCompletion(() -> clients.remove(emitter));
		emitter.onTimeout(() -> clients.remove(emitter));
		return emitter;
	}

	@Async // 비동기 실행 (@EnableAsync, Executor @Bean(쓰레드 풀) 필요)
	@EventListener // Temperature 에 대한 이벤트를 수신하기 위해서 해당 어노테이션을 사용
	public void handleMessage(Temperature temperature) { // <- publisher.publishEvent(...) 에서 이벤트가 넘어오면 이 메서드가 호출됨
		clients.forEach(emitter -> emitter.send(temperature, MediaType.APPLICATION_JSON));
	}
}
```

#### 클라이언트

```javascript
let eventSource = new EventSource("/temperature-stream");
eventSource.onmessage = function (event) {
    let temperature = JSON.parse(event.data);
    console.log(temperature);
};

eventSource.onopen = (event) => {
    console.log("EventSource connected.");
};

eventSource.onerror = (event) => {
    console.log("EventSource failed.");
};
```

#### 결과

```text
17
15
30
...
```

#### 평가

* 스프링에서 제공하는 발행-구독 메커니즘은 고부하 및 고성능 환경에서 작동하기 위해 설계되지는 않았습니다.
* 현재 하나의 온도 센서가 있지만, 수천개, 수백만개의 개별 스트림이 필요할 때 어떤 일이 발생할까요?
* 스프링은 그러한 부하를 효율적으로 처리하지 못합니다.
* 큰 단점은 해당 접근 방식이 스프링 프레임워크의 내부 메커니즘을 활용합니다.
    * 스프링 컨텍스트를 로드하지 않고 비즈니스 로직 단위 테스트는 어렵습니다.
    * 드문 경우로 프레임워크의 변경이 발생하면 코드를 수정해야 합니다.
    * 또한 @EventListener 어노테이션에서 종료 시그널, 에러 시그널을 알리기 위한 별도의 처리가 어렵습니다.
* 온도 이벤트를 비동기적으로 전파하기 위해서 쓰레드 풀을 사용한다는 점도 단점입니다.
* 클라이언트가 하나도 없을 때도(아무도 구독하고 있지 않음에도) 이벤트는 계속 발생됩니다.

## 다시 패턴 소개

### 반복자 패턴 [위키백과][4]

[4]: https://ko.wikipedia.org/wiki/%EB%B0%98%EB%B3%B5%EC%9E%90_%ED%8C%A8%ED%84%B4

```java
public interface Iterator<T> {
	boolean hasNext();

	T next();
}
```

* 반복자 패턴의 next()는 하나씩 항목을 검색할 수 있게 합니다.
* hasNext()는 시퀀스의 끝을 알려줄 수 있게 합니다.

### 관찰자 패턴 + 반복자 패턴 = 리액티브

```java
public interface RxObserver<T> {
	void onNext(T item);

	void onCompleted();
}
```

* RxJava 1.x는 한동한 리액티브 프로그래밍은 위한 표준 라이브러리였습니다.
* 위의 코드 블럭을 보면 반복자 패턴과 매우 비슷하지만 RxObserver는 onNext()에 의해 새로운 값이 통지되고, onComplete()에 의해 스트림의 끝을 알립니다.
* 해당 코드에서 에러를 처리해주는 메커니즘이 있다면 좋을 것 입니다.

```java
// https://github.com/ReactiveX/RxJava/blob/1.x/src/main/java/rx/observers/Observers.java
public interface RxObserver<T> {
	void onNext(T item);

	void onCompleted();

	void onError(Exception e);
}
```

### RxJava 1.x의 Observable, Observer 간의 관계

* ![hello_reactive_02_02.png](/assets/images/hello_reactive_02/hello_reactive_02_02.png)
* Observable은 0~N개의 이벤트를 Observer 에게 보내고 완료 또는 오류를 시그널을 Observer에게 알립니다.
* 각 Observer 는 onNext()를 여러 번 호출한 다음, onCompleted() 또는 onError()를 호출합니다.
    * 이 때 둘이 동시에 호출되지는 않고, onComplete(), onError()가 호출되고 onNext()가 호출되지 않습니다.

```java

@Component
class TemperatureSensor {
	private final Observable<Temperature> dataStream = Observable.range(0, Integer.MAX_VALUE)
	                                                             .map(i -> new Temperature(/* anything */))
	                                                             .delay(1, TimeUnit.SECONDS)
	                                                             .share(); // 하나의 데이터 스트림을 여러 구독자와 공유
}

@RestController
class TemperatureController {
	private final TemperatureSensor sensor;

	@GetMapping("/temperature-stream")
	public SseEmitter events() { // 해당 메서드를 호출하면(GET 요청을 날리면) 클라이언트로 등록되고 Sse로 이벤트를 계속 받아볼 수 있다.
		SseEmitter emitter = new SseEmitter();

		sensor.getDataStream()
		      .subscribe(new Observer<Temperature>() { // 실제로는 Observer가 아닌 Subscriber가 와야하지만 이해의 편의를 위해서 Observer로 임의 표현하였습니다.
			      @Override
			      public void onNext(Temperature temperature) {
				      emitter.send(temperature);
			      }

			      @Override
			      public void onCompleted() {
				      emitter.send("complete");
			      }

			      @Override
			      public void onError(Exception e) {
				      emitter.send(e)
			      }
		      }); // 리턴값은 Subscription이다. 이를 통해 구독을 해지할 수 있다.
		return emitter;
	}
}
```

* SseEmitter 클라이언트들을 관리하지 않으며 동기화에도 신경쓰지 않을 수 있습니다.
* 더 이상 ```@EventListener, @Async, @EnableAsync, Executor @Bean(쓰레드 풀)```을 사용하지 않아도 됩니다.
* 참고로 ```Observable#subscribe```하고 난 결과는 ```Subscription```이라는 객체를 반환합니다.

```java
interface Subscription {
	void unsubscribe();

	boolean isUnsubscribed();
}
```

* unsubscribe()를 호출하면 Observable에게 더 이상 새 이벤트를 보낼 필요가 없음을 알릴 수 있습니다.

## 리액티브 표준

### RxJava 1.x 문제점

#### API 문제

* RxJava 1.x 는 시간에 따라 빠르게 매우 빠르게 변화했기 때문에, RxJava 1.x에 의존하는 다른 라이브러리를 사용하면 원치 않는 문제가 발생하였습니다.
* 또한 RxJava 1.x는 커스터마이징이 표준화 되어 있지 않았습니다.
    * Observable 커스터마이징, 변환단계 커스터마이징 등.

#### 순수 푸시 모델 방식

* 리액티브 초기 단계에서는 대부분 라이브러리의 데이터 흐름은 Observable에서 Observer에게 순수하게 푸시되는 방식(컨슈머의 처리 성능 상관X)이었습니다.
* 순수 푸시 모델은 요청하는 횟수를 최소화하여 전체 시간이 최적화되기 때문에, RxJava 1.x를 비롯한 유사 라이브러리들은 데이터 푸시를 위해 설계되었습니다.

##### 빠른 프로듀서, 느린 컨슈머

* ![hello_reactive_02_03.gif](/assets/images/hello_reactive_02/hello_reactive_02_03.gif)
* 컨슈머의 부하가 심해져 치명적인 오류가 발생할 수 있습니다.
* 이러한 경우 직관적인 솔루션은 원소를 큐에 수집하는 것이고, 큐는 프로듀서에 둘 수 있고, 컨슈머에 둘 수도 있습니다.

###### 무제한 큐

* 장점은 모든 메시지가 반드시 컨슈머에게 전달이 된다는 점
* 단점은 메모리 한도에 도달하면 시스템이 손상

###### 크기가 제한된 드롭 큐(신규 유입을 버리거나, 오래된 유입을 버리거나, 우선순위에 따라 버리거나)

* 큐의 메시지의 중요성이 낮을 때 고려
* 결제 같은 중요성이 높은 메시지는 선택하지 못함

###### 크기가 제한된 블록킹 큐

* 큐가 꽉 차면 새로운 메시지가 들어올 때까지 블록
* 사실상 비동기 동작을 모두 무효화하기 때문에 사용하지 못하는 모델

#### 적합한 제어를 추가하지 않은 순수 푸시 모델은 다양한 부작용이 있다.

* 리액티브 메니페스토에서 배압 제어 메커니즘의 중요성을 언급한 이유입니다.
* RxJava 1.x는 배압을 관리하는 표준화된 기능을 제공하지 않습니다.
    * window, buffer와 같은 연산자가 있지만 모든 서비스가 배치 작업을 지원하지는 않아 적용범위가 제한적입니다.

### 리액티브 스트림 표준

```java
package org.reactivestreams;

interface Publisher<T> {
	public void subscribe(Subscriber<? super T> s);
}

interface Subscriber<T> {
	public void onSubscribe(Subscription s); // 구독 했음을 알린다.

	public void onNext(T t);

	public void onError(Throwable t);

	public void onComplete();
}

interface Subscription {
	public void request(long n); // Publisher가 보내줘야 하는 데이터 크기를 알려주어 배압을 관리한다.

	public void cancel();
}
```

* ![hello_reactive_02_04.png](/assets/images/hello_reactive_02/hello_reactive_02_04.png)
* 순수 푸시 모델과는 다르게 배압을 적절하게 제어할 수 있습니다.

#### 사실 하나 더 있습니다.

```java
package org.reactivestreams;

interface Processor<T, R> extends Subscriber<T>, Publisher<R> {
}
```

* Processor는 Publisher와 Subscriber 사이에 몇 가지 처리 단계를 추가하도록 설계되었습니다.

#### Publisher/Processor의 구현은 매우 어렵습니다. 만약에 직접 구현하려면 다음 문헌을 참고하세요

* https://medium.com/@olehdokuka/mastering-own-reactive-streams-implementation-part-1-publisher-e8eaf928a78c
* https://github.com/reactive-streams/reactive-streams-jvm/tree/master/tck
    * Publisher/Subscriber 등의 구현을 검증하는 테스트 케이스가 있습니다.

### JDK9

* 위에서 정의된 모든 인터페이스는(```org.reactivestreams.*```) ```java.util.concurrent.Flow``` [Flow][5] 패키지에 정의되어 있습니다.
* 자바9 부터는 표준으로 제공되는 리액티브 스트림을 사용할 수 있습니다.

[5]: https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/Flow.html