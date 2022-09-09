---
title:  "왜 리액티브인가?"
excerpt: "리액티브가 추구하고자 하는 것은?"
category: "reactive"

last_modified_at: 2022-09-10T
---

# 왜 리액티브인가?

## 리액티브 프로그래밍의 정의 [위키피디아][1]

* <cite>리액티브 프로그래밍은 데이터 스트림 및 변경의 전파와 관련된 선언적 프로그래밍 패러다임 입니다.</cite>
* 예를 들어, 명령형 프로그래밍에서 ```a := b + c``` 와 같은 문장에서 ```b + c```식이 평가되어 나온 결과가 ```a```에 할당되는 것을 의미합니다. 나중에 ```b```와 ```c```가 바뀌어도 ```a```에 영향을
  미치지 얺습니다.
* 반면에 리액티브 프로그래밍에서는 ```a```는 할당된 값을 결정하기 위해 ```a := b + c```라는 문장을 명시적으로 재수행하지 않고도 ```b```또는 ```c```가 바뀔 때 자동적으로 바뀝니다.

## 리액티브 프로그래밍의 정의는 알겠는데 그래서...

* 사실 리액티브를 검색할 때 가장 관련이 높은 단어는 ```리액태브 프로그래밍``` 이고, 프로그래밍 모델 종류(명령형, 함수형, 객체지향형...) 중 하나입니다.
* 그러나 리액티브의 유일한 의미는 아니이며, 리액티브 말 뒤에는 강력한 시스템을 구축하기 위한 기본 설계 원칙이 숨어있습니다.

[1]: https://en.wikipedia.org/wiki/Reactive_programming

## 그래서 왜 리액티브인가?

### 전통적인 쓰레드 기반 요청 처리

* ![hello_reactive_01_01.png](/assets/images/hello_reactive_01/hello_reactive_01_01.png)
* 시간 당 평균 1000개의 요청을 처리하는 서버가 있다고 가정해봅시다.
* 톰캣 웹 서버를 실행하고, 500개의 쓰레드로 톰캣 쓰레드 풀을 구성했다고 가정합시다.
* 사용자 평균 응답 시간은 약 250ms 라고 가정합시다.
  * 단순 계산시 초당 약 2000개의 사용자 요청을 처리할 수 있다고 생각할 수 있습니다.
* 그런데 2000개 보다 많은 사용자의 요청이 오게 되면 어떻게 될까요?
  * 쓰레드 풀에 사용자 요청을 처리할 쓰레드가 남아 있지 않게 됩니다.
  * 결과적으로 사용차 요청이 블록킹되고, 응답시간이 증가합니다.
    * 서비스가 중단됩니다.
    * 고객이 항의를 합니다.
* 우리는 이를 해결하기 위해서 수직 확장 또는 수평 확장을 통해 이를 해결할 수 있습니다.
    * 수직 확장은 서버의 성능을 향상시키는 방법입니다.
    * 수평 확장은 서버의 수를 늘려서 서버의 성능을 향상시키는 방법입니다.

### 마이크로서비스 & 서버 스케일링(수평 확장)

* ![hello_reactive_01_02.png](/assets/images/hello_reactive_01/hello_reactive_01_02.png)

```java

@RequestMapping("/resource")
@RequiredArgsConstructor
public class ResourceController {
	private final RestTemplate restTemplate;

	@GetMapping
    public Example processRequest() {
      Example example = restTemplate.getObject("something", Example.class); // (1) 다른 마이크로 서비스로 요청을 보냅니다. (이 때 쓰레드가 블록됩니다.)
      process(example); // (2) 다른 마이크로 서비스에 있는 내용을 받아와서 가공을 하는 로직
		return example;
	}
}
```

* 우리는 서버의 수를 늘려서 여러 개의 요청을 한번 더 받을 수 있습니다.
* 그런데 만약 서비스 하고 있는 서버가 다른 서버의 요청을 호출하게 된다면 어떻게 될까요?
    * 만약 다른 서비스 요청이 시간이 많이 걸리는 작업이라면 무슨 상황이 일어날까요?
* 여전히 우리는 다시 위에서 겪었던 문제에 도달하게 됩니다.

```java
interface ShoppingCardService {
	Output calculate(Input input); // HTTP 요청 또는 DB 쿼리와 같은 시간이 걸리는 I/O 작업이라고 생각해봅시다.
}

class OrderService {
	private final ShoppingCardService shoppingCardService;

	void process() {
      Input input = new Input();
      Output output = shoppingCardService.calculate(input); // 이 때 쓰레드가 블록됩니다.
      something(); // 위의 문장이 끝날 때 까지 기다림
	}
}
```

### 잠깐 Block/Non-Block & Sync/Async 비교

#### Block/Non-Block

* 호출된 함수가 끝날 때 까지 기다리는지, 기다리지 않는지에 대한 여부
* Block 은 호출된 함수가 끝날 때 까지 호출한 함수가 기다린다는 의미
* Non-Block 은 호출된 함수가 바로 리턴하여 호출한 함수에게 제어권을 넘겨 받음.

#### Sync/Async

* 호출되는 함수의 작업 완료 여부를 알아야 하는지 여부
* Sync 는 호출된 함수의 작업이 완료되어야(결과물이 나와야) 호출한 함수가 다음 작업을 진행할 수 있다는 의미
* Async 는 호출된 함수의 작업이 완료되지 않아도(결과물을 몰라도) 호출한 함수가 다음 작업을 진행할 수 있다는 의미

### 멀티 쓰레드 기반 처리

#### 콜백 기반

* 콜백은 자바스크립트에서 자주 보았습니다.

```javascript
userRepository.getUser(username, (user) => {
  let userId = user.getUserId();
  console.log(userId);
});
```

* 콜백을 이용하게 되면 우리는 쓰레드가 블록킹되지 않게 할 수 있습니다.

```java
interface ShoppingCardService {
  void calculate(Input input, Consumer<Output> consumer); // 인자가 두개로 바뀌었고, 리턴타입은 void, 콜백을 받는 인자가 추가되었습니다.
}

class SyncShoppingCardService implements ShoppingCardService {
  @Override
  public void calculate(Input input, Consumer<Output> consumer) {
    // 블록킹 미호출
    Output output = new Output();
    consumer.accept(output);
  }
}

class AsyncShoppingCardService implements ShoppingCardService {
  @Override
  public void calculate(Input input, Consumer<Output> consumer) {
    new Thread(() -> {
      // 블록킹 호출
      Output output = restTemplate.getForObject("...", Output.class);
      consumer.accept(output);
    }).start();
  }
}

class OrderService {
  private final ShoppingCardService shoppingCardService;

  void process() {
    Input input = new Input();
		shoppingCardService.calculate(input, output -> {
			// ...
		});
		something(); // 즉시 수행 됨
	}
}
```

* 그러나 콜백은 콜백 지옥을 만들어내고, 자바스크립트에서 자주 언급되는 내용이기는 하나, 자바라고 예외는 아닙니다.

```javascript
userRepository.getUser(username, (user) => {
    let userId = user.getUserId();
    orderService.getOrders(userId, (orders) => {
        paymentService.getStatus(orders, (result) => {
            // ...
        });
    });
});
```

#### AsyncRestTemplate

```java
class AsyncTemplate {
	AysncRestTemplate template = new AsyncRestTemplate();
	SuccessCallback onSuccess = r -> {
		// ...
	};
	FailureCallback onFailure = e -> {
		// ...
	};
	ListenableFuture<String> response = template.getForEntity("...", String.class);
	response.addCallback(onSuccess,onFailure);
}
```

#### CompletableFuture

* 서비스 구현체에서는 ```@Async``` 어노테이션을 사용합니다.
* Configuration 으로 ```@EnableAsync``` 및 Executor ```@Bean``` 을 만들어 주어야 합니다.

```java
interface ShoppingCardService {
	CompletableFuture<Output> calculate(Input input);
}

class OrderService {
	private final ShoppingCardService shoppingCardService;

	void process() {
		Input input = new Input();
		shoppingCardService.calculate(input)
		                   .thenApply(output -> {
			                   // ...
		                   })
		                   .thenCombine(output -> {
			                   // ...
		                   })
		                   .thenAccept(output -> {
			                   // ...
		                   });
		something(); // 즉시 수행 됨
	}
}
```

* 기본적으로 위와 같은 멀티쓰레딩 디자인은 그들의 작업을 동시에 실행하기 위해서 하나의 CPU를 공유할 수 있다고 가정하였습니다.
* 이 때 컨텍스트 스위칭이 발생하게 되어 결과적으로는 성능이 떨어지게 됩니다.
* 즉 적은 수의 CPU에 많은 수의 쓰레드를 사용하는 것은 비효율적입니다.
* 또한 쓰레드를 무한정 늘릴 수 없습니다. (메모리 부족)
* 쓰레드 풀을 사용한다 하더라도 쓰레드를 모두 사용한 상태에서는 쓰레드가 반납될 때 까지 새로운 요청이 오면 대기해야 합니다.

## 리액티브가 결국 하고 싶은 일은?

### 강력한 시스템을 구축하기 위한 기본 설계 원칙 [리액티브 메니페스토][2]

* ![hello_reactive_01_04.png](/assets/images/hello_reactive_01/hello_reactive_01_04.svg)

[2]: https://www.reactivemanifesto.org/ko

### 리액티브 프로그래밍은 리엑티브 시스템을 만들기에 적합한 좋은 기술입니다.

* 비동기 데이터 처리
    * 단순한 요청/응답(request-response) 처리
    * 연속적인 데이터의 스트림 처리 등
* 배압 지원
* 논블록킹
* 함수형/선언적 프로그래밍

## 그보다 리액티브 프로그래밍이 뭔지 모르겠는데요?