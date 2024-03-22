---
layout: post
title: "[TypeScript] 코드 생성과 타입이 관계 없음을 이해하기"
date: 2024-03-20
categories: dev
tags: typescript
---

큰 그림에서 타입스크립트 컴파일러는 두 가지 역할을 하는데 이 두 역할은 완전히 독립적이다. - 최신 타입스크립트/자바스크립트를 브라우저에서 동작할 수 있도록 구버전의 자바스크립트로 Transpile(Translate + Compile) - 코드 타입 오류 체크

### 1. 타입 오류가 있는 코드도 컴파일이 가능하다.

컴파일은 타입 체크와 독립적으로 동작하기 때문에, 타입 오류가 있는 코드도 컴피일이 가능하다.

```shell
$ cat test.ts
let x = 'hello';
x = 1234;

$ tsc test.ts
test.ts:2:1 - error TS2322: Type 'number' is not assignable to type 'string'.

2 x = 1234;

$ cat test.js
var x = 'hello';
x = 1234;
```

타입스크립트 오류는 Warning과 비슷하다. 문제가 될 만한 부분을 알려 주지만, 빌드를 멈추지는 않는다.

타입 오류가 있어도 컴파일을 하기 때문에 오류를 수정하지 않더라도 애플리케이션의 다른 부분을 테스트할 수 있다.
타입 오류가 있을 때 컴파일하지 않으려면, tsconfig.json에 noEmitOnError를 설정한다.

### 2. 런타임에는 타입 체크가 불가능하다.

```typescript
interface Squere {
  width: number;
}

interface Rectangle extends Squere {
  height: number;
}

type Shape = Squere | Rectangle;

function calculateArea(shape: Shape) {
  if (shape instanceof Rectangle) {
    return shape.width * shape.height;
  } else {
    return shape.width * shape.width;
  }
}
```

instanceof 체크는 런타임에 실행되지만 Rectangle은 타입이기 때문에 런타임 시점에 아무런 역할을 할 수 없다.'

타입 정보를 유지하는 방법 중 태그를 다는 방법이 있다.

```typescript
interface Squere {
  kind: "square";
  width: number;
}

interface Rectangle {
  kind: "rectangle";
  height: number;
  width: number;
}

type Shape = Squere | Rectangle;

function calculateArea(shape: Shape) {
  if (shape.kind === "rectangle") {
    return shape.width * shape.height;
  } else {
    return shape.width * shape.width;
  }
}
```

또는 타입(런타임 접근 불가)과 값(런타임 접근 가능)을 둘 다 사용하는 기법도 있다. 타입을 클래스로 만든다.

```typescript
class Squere {
  constructor(public width: number) {}
}

class Rectangle extends Squere {
  constructor(public width: number, public height: number) {
    super(width);
  }
}

type Shape = Squere | Rectangle;

function calculateArea(shape: Shape) {
  if (shape instanceof Rectangle) {
    return shape.width * shape.height;
  } else {
    return shape.width * shape.width;
  }
}
```

### 타입 연산은 런타임에 영향을 주지 않습니다.

```typescript
function asNumber(val: number | string): number {
  return val as number;
```

이 코드는 타입 체커는 통과하지만 number로 값이 정제되지 않는다. as number는 타입 연산이고 런타임 동작에는 아무런 영향을 주지 않는다. 이를 위해서는 런타임의 타입을 체크하고 자바스크립트 연산을 통해 변환해야한다.

```typescript
function asNumber(val: number | string): number {
  return typeof(val) === 'string' ? Number(val) : val;
```

### 런타임 타입은 선언된 타입과 다를 수 있습니다.

다음 함수에서 default는 죽은(dead) 코드이다. 여기서는 타입스크립트가 죽은 코드를 찾아내지 못한다.

```typescript
function setLightSwitch(value: boolean) {
  switch (value) {
    case (true):
      turnLightOn();
      break;
    case false:
      turnLightOff();
      break;
    default:
      console.log('실행되지 않습니다.')
```

: boolean이 런타임에 제거된다.

```typescript
interface LightApiResponse {
  lightSwitchvalue: boolean;
}
async function setLight() {
  const res = await fetch('/light');
  const result: LightApiResponse = await response.json();
  setLightSwitch(result.lightSwitchValue);
```

api에서 받아온 lightSwitchValue가 boolean이라는 보장이 없다. 즉, 문자열이 될 수도 있는 것이다.
타입스크립트에서는 런타임 타입과 선언된 타입이 다를 수 있다는 것을 항상 명심해야한다.

### 타입스크립트 타입으로는 함수를 오버로드할 수 없습니다.

동일한 이름에 매개변수만 다른 여러 버전의 함수를 만드는 '오버로딩'을 타입스크립트는 할 수 없다. 런타임의 동작과 타입 체크는 무관하기 때문이다.

타입스크립트가 함수 오버로딩 기능을 지원하지만 타입 수준에서만 동작한다. 즉, 선언문을 작성할 수 있지만 구현체는 하나다.

```typescript
// 중복된 함수 구현
function add(a: number, b: number): number;
function add(a: string, b: string): string;

function add(a, b) {
  return a + b;
}

const three = add(1, 2); // type number
const twelve = add("1", "2"); // type string
```

### 타입스크립트 타입은 런타임 성능에 영향을 주지 않는다.

- 타입과 타입 연산자는 컴파일시 제거되기 때문에 런타임 성능에 영향을 주지 않는다.
- 런타임 오버헤드가 없는 대신, 타입스크립트 컴파일러는 '빌드타임' 오버헤드가 있다. 그렇지만 타입스크립트 컴파일은 일반적으로는 빠른 편이다. 오버헤드가 커지면, 빌드 도구에서 'transpile only'를 설정하여 타입 체크를 건너뛸 수 있다.
- 타입스크립트는 코드 컴파일 시 호환성과 성능 중 하나를 선택하게 되는 문제에 맞닥뜨릴 수도 있다. 예를 들어 제너레이터 함수가 ES5타깃으로 컴파일되려면, 타입스크립트 컴파일러는 호환성을 위한 특정 헬퍼 코드를 추가할 것이다.

---

_본 포스트는 댄 밴더캄 저자의 **이펙티브 타입스크립트**를 기반으로 스터디하며 정리한 내용들입니다._

- [이펙티브 타입스크립트](https://product.kyobobook.co.kr/detail/S000001033114)
