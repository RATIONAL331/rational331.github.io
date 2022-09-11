---
title:  "리액터에 관하여"
excerpt: "리액티브 스트림을 구현한 리액터에 관하여 알아봅시다."
category: "reactive"

last_modified_at: 2022-09-12T
---

# 리액터

## 리액터 프로젝트

* https://projectreactor.io/
* 리액터는 리액티브 스트림을 구현합니다.
* 리액터의 기본 목표는 비동기 파이프라인을 구축할 때 콜백 지옥과 깊게 중첩된 코드를 생략하는 목적으로 설계되었습니다.
* 그래서 가독성을 높이고, 조합성이라는 특성을 추구합니다.
* 리액터는 ```연산자를 연결해서``` 사용하는 것을 권장합니다.
    * ![hello_reactor_01_01.png](/assets/images/hello_reactor_01/hello_reactor_01_01.png)
