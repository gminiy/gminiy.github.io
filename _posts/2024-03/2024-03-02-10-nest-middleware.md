---
layout: post
title: "[NestJs] 미들웨어"
date: 2024-03-02
categories: dev
tags: nestjs
---

- [1. 미들웨어](#1-미들웨어)
- [2. Logger 미들웨어](#2-logger-미들웨어)
- [3. MiddlewareConsumer](#3-middlewareconsumer)
- [4. 전역으로 적용하기](#4-전역으로-적용하기)

---

# 1. 미들웨어

라우트 핸들러가 클라이언트 요청을 처리하기 전에 수행되는 컴포넌트를 말한다. Nest 미들웨어는 Express와 동일하며 다음 동작을 수행할 수 있다.

- 어떤 형태의 코드라도 수행할 수 있다.
- 요청과 응답에 변형을 가할 수 있다.
- 요청/응답 주기를 끝낼 수 있다.
- 여러 개의 미들웨어를 사용한다면 next()로 호출 스택상 다음 미들웨어에 제어권을 전달한다.

요청 응답 주기를 끝낸다는 것은 응답을 보내거나 에러 처리를 한다는 뜻이다. 미들웨어가 응답 주기를 끝내지 않을 것이라면 next()를 호출해야 한다.

미들웨어를 활용하여 다음과 같은 작업을들 수행한다.

- 쿠키 파싱 : 쿠키를 파싱하여 변경하여 핸들러에 전달
- 세션 관리 : 세션 상태를 조회해서 요청에 세션 정보를 추가한다.
- 인증/인가 : 사용자가 서비스에 접근 가능한 권한이 있는지 확인한다. Nest는 가드를 이용하도록 권장한다.
- 본문 파싱 : JSON, 파일 스트림 같은 데이터를 변환하여 매개변수에 넣는 작업을 한다.

커스텀 미들웨어도 구현할 수 있다. 예를 들어 데이터베이스 트랜잭션이 필요한 요청이 있을 때 트랜잭션을 걸고 동작 수행 완료 후 커밋하는 미들웨어를 사용할 수 있다. 비슷한 개념으로는 인터셉터가 있다.

# 2. Logger 미들웨어

미들웨어는 함수로 작성하거나 NestMiddleware 인터페이스를 구현한 클래스로 작성할 수 있다.

- logger.middleware.ts

```javascript
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log("Request...");
    next();
  }
}
```

- app.module.ts

```javascript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes("/users");
  }
}
```

# 3. MiddlewareConsumer

configure 메서드에 인수로 전달된 MiddlewareConsumer 객체를 이용해서 미들웨어를 적용할 라우트를 관리할 수 있다.
apply 메서드에 미들웨어를 콤마로 나열하면 미들웨어가 나열된 순서대로 적용된다.

forRoutes 의 인수는 문자열 형식의 경로를 직접주거나, 컨트롤러 클래스 이름을 주어도 되고, RouteInfo 객체를 넘길 수도 있다. 보통은 클래스를 주어서 동작하도록 한다.

```javascript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes(UsersController);
  }
}
```

exclude 는 미들웨어를 적용하지 않을 라우팅 경로를 설정한다.

```javascript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .exclude({ path: "/users", method: RequestMethod.GET })
      .forRoutes(UsersController);
  }
}
```

# 4. 전역으로 적용하기

미들웨어를 모든 모듈에 적용하기 위해서는 main.ts에서 NestFactory.create의 use() 메서드를 사용하여 미들웨어를 설정한다.

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "@nestjs/common";
import { LoggerMiddleware } from "./logger/logger.middleware";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ transform: true }));
  app.use(LoggerMiddleware);
  await app.listen(3000);
}
bootstrap();
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
