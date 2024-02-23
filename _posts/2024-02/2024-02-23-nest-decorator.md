---
layout: post
title: "[NestJs] Decorator"
date: 2024-02-23
categories: dev
tags: nestjs
---

> - [1. 설정](#1-설정)
> - [2. 사용 예시](#2-사용-예시)
> - [3. Decorator Factory](#3-decorator-factory)
> - [4. Decorator 합성](#4-decorator-합성)
> - [5. Decorator 역할 요약](#5-decorator-역할-요약)

---

Nest는 Decorator를 적극 활용한다. Decorator를 잘 사용하면 횡단 관심사를 분리하여 관점 지향 프로그래밍을 적용한 코드를 작성할 수 있다.
타입스크립트의 Decorator는 클래스, 메서드, 접근자, 프로퍼티, 매개변수에 적용 가능하다. 각 요소의 선언부 앞에 @로 시작하는 Decorator를 선언하면 구현된 코드가 함께 실행된다.

# 1. 설정

Decorator를 사용하려면 tsconfig.json의 experimentalDecorators 옵션을 true로 활성화해야 한다.

tsconfig.json

```javascript
{
  "compilerOptions": {
    "target": "ES5",
    "experimentalDecorators": true
  }
}
```

# 2. 사용 예시

```javascript
function deco(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor
) {
  console.log("Decorator 가 평가됨");
}

class TestClass {
  @deco
  test() {
    console.log("test 함수 호출");
  }
}

const t = new TestClass();
t.test();
```

```
> Decorator 가 평가됨
> test 함수 호출
```

# 3. Decorator Factory

만약 **Decorator 에 인수를 넘기고 싶다면** Decorator Factory, 즉 **Decorator를 리턴하는 함수를 만들면 된다.**

```javascript
function deco(value: string) {
  console.log("Decorator 평가됨");

  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    console.log(value);
  };
}

class TestClass {
  @deco("Hello")
  test() {
    console.log("함수 호출됨");
  }
}

const t = new TestClass();
t.test();
```

```
> Decorator 평가됨
> Hello
> 함수 호출됨
```

# 4. Decorator 합성

여러 개의 Decorator를 사용한다면 다음과 같이 선언한다.

- **각 표현은 위에서 아래로 평가된다.**
- **결과는 아래에서 위로 함수를 호출한다.**

```javascript
function first() {
  console.log("first(): factory evaluated");
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    console.log("first(): called");
  };
}

function second() {
  console.log("second(): factory evaluated");
  return function (
    target: any,
    propertyKey: string,
    descriptor: PropertyDescriptor
  ) {
    console.log("second(): called");
  };
}

class ExampleClass {
  @first()
  @second()
  method() {}
}
```

```
> first(): factory evaluated
> second(): factory evaluated
> second(): called
> first(): called
```

# 5. Decorator 역할 요약

| Decorator           | 역할                        | 전달 인수                               | 선언 불가능 위치                           |
| ------------------- | --------------------------- | --------------------------------------- | ------------------------------------------ |
| Class Decorator     | 클래스의 정의를 읽거나 수정 | constructor                             | d.ts 파일, declare 클래스                  |
| Method Decorator    | 메서드의 정의를 읽거나 수정 | target, propertyKey, propertyDescriptor | d.ts 파일, declare 클래스, 오버로드 메서드 |
| Accessor Decorator  | 접근자의 정의를 읽거나 수정 | target, propertyKey, propertyDescriptor | d.ts 파일, declare 클래스                  |
| Property Decorator  | 속성의 정의를 읽거나 수정   | target, propertyKey                     | d.ts 파일, declare 클래스                  |
| Parameter Decorator | 매개변수의 정의를 읽음      | target, propertyKey, parameterIndex     | d.ts 파일, declare 클래스                  |
