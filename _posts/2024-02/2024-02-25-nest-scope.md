- [1. 스코프](#1-스코프)
  - [1.1 프로바이더에 스코프 적용](#11-프로바이더에-스코프-적용)
  - [1.2 컨트롤러에 스코프 적용](#12-컨트롤러에-스코프-적용)

---

# 1. 스코프

Node.JS는 멀티 스레드 상태 비저장 모델을 따르지 않기 때문에 싱글턴 인스턴스를 사용하는 것이 권장된다.
하지만 요청별로 캐싱을 한다거나 요청 추적, 또는 멀티테넌시를 지원하기 위해서는 요청 기반으로 생명주기를 제한해야 한다.

아래는 스코프 종류이다.

- Default
  싱글턴 인스턴스가 전체 애플리케이션에 공유됨
  인스턴스 수명은 애플리케이션 생명주기와 같다.
  애플리케이션이 부팅되면 모든 싱글턴 프로바이더의 인스턴스가 만들어진다.

- Request
  들어오는 요청마다 별도의 인스턴스가 생성됨
  요청을 처리하고 나면 인스턴스는 garbage-collected 됨

- Transient
  인스턴스는 공유되지 않는다.
  이 프로바이더를 주입하는 각 컴포넌트는 새로 생성된 전용 인스턴스를 주입받는다.

---

## 1.1 프로바이더에 스코프 적용

@Injectable 데코레이터에 scope속성을 주는 방법

```javascript
import { Injectable, Scope } from "@nestjs/common";

@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

---

## 1.2 컨트롤러에 스코프 적용

@Controller 데코레이터는 ControllerOptions를 인수로 받을 수 있다. ContorllerOptions는 ScopeOptions를 상속한다.

```javascript
@Constroller({
  path: "cats",
  scope: "Scope.REQUEST",
})
export class CatsController {}
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
