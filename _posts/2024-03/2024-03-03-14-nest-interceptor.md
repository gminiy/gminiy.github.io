---
layout: post
title: "[NestJs] 인터셉터"
date: 2024-03-03
categories: dev
tags: nestjs
---

- [1. 인터셉터](#1-인터셉터)
- [2. 응답과 예외 매핑](#2-응답과-예외-매핑)
- [3. 유저 서비스에 인터셉터 적용하기](#3-유저-서비스에-인터셉터-적용하기)

---

# 1. 인터셉터

인터셉터는 요청과 응답을 가로채서 변형을 할 수 있는 컴포넌트이다. 이를 이용하여 다음고 같은 기능을 수행할 수 있다.

- 매서드 실행 전/후 추가 로직 바인딩
- 함수에서 반환된 결과를 변환
- 함수에서 던져진 예외를 변환
- 기본 기능의 동작을 확장
- 특정 조건에 따라 기능을 완전히 재정의(예: 캐싱)

인터셉터는 미들웨어와 비슷하지만 수행 시점이 다르다. 미들웨어는 라우트 핸들러 전달 전에 동작하고 인터셉터는 라우트 핸들러의 전/후 호출된다.
라우트 핸들러가 요청 처리 전후 어떤 로그를 남기는 요구 사항이 있다면 LogginInterceptor를 만들어 처리할 수 있다.

```javascript
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable, tap } from "rxjs";

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(
    context: ExecutionContext,
    next: CallHandler<any>
  ): Observable<any> | Promise<Observable<any>> {
    console.log("Before...");

    const now = Date.now();
    return next
      .handle()
      .pipe(tap(() => console.log(`After... ${Date.now() - now} ms`)));
  }
}
```

특정 컨트롤러나 메서드에 적용하고 싶다면 @UseInterceptors()를 이용하면 된다. 여기서는 전역으로 적용한다.

- main.ts

```javascript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { LoggingInterceptor } from './logging.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  ...
  app.useGlobalInterceptors(new LoggingInterceptor());
  await app.listen(3000);
}
bootstrap();
```

NestInterceptor 인터페이스

```javascript
export interface NestInterceptor<T = any, R = any> {
  /**
   * Method to implement a custom interceptor.
   *
   * @param context an `ExecutionContext` object providing methods to access the
   * route handler and class about to be invoked.
   * @param next a reference to the `CallHandler`, which provides access to an
   * `Observable` representing the response stream from the route handler.
   */
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>
  ): Observable<R> | Promise<Observable<R>>;
}

/**
 * Interface providing access to the response stream.
 *
 * @see [Interceptors](https://docs.nestjs.com/interceptors)
 *
 * @publicApi
 */
export interface CallHandler<T = any> {
  /**
   * Returns an `Observable` representing the response stream from the route
   * handler.
   */
  handle(): Observable<T>;
}
```

CallHandler는 handle() 메서드를 구현해야한다. 이 메서드는 라우트 핸들러에서 전달된 응답 스트림을 리턴하고 RxJS의 Observable로 구현되어 있다. 인터셉터가 handle 메서드를 호출하지 않으면 라우터 핸들러가 동작을 하지 않는다.

# 2. 응답과 예외 매핑

전달받은 응답을 변형해보자. 라우터 핸들러에서 전달한 응답을 객체로 감싸서 전달하도록 하는 TransforInterceptor를 만든다.

```javascript
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";

export interface Response<T> {
  data: T;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler<T>
  ): Observable<Response<T>> | Promise<Observable<Response<T>>> {
    return next.handle().pipe(
      map((data) => {
        return { data };
      })
    );
  }
}
```

TransforInterceptor를 전역으로 적용해보자. useGlobalInterceptors에 인터셉터 객체를 추가한다.

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { LoggingInterceptor } from "./logging.interceptor";
import { TransformInterceptor } from "./transform.initerceptor";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalInterceptors(
    new LoggingInterceptor(),
    new TransformInterceptor()
  );
  await app.listen(3000);
}
bootstrap();
```

라우트 핸들링 도중 던져진 예외를 잡아서 변환해보자.

```javascript
import {
  BadRequestException,
  CallHandler,
  ExecutionContext,
  Injectable,
  NestInterceptor,
} from "@nestjs/common";
import { catchError, Observable, throwError } from "rxjs";

@Injectable()
export class ErrorInterceptor implements NestInterceptor {
  intercept(
    context: ExecutionContext,
    next: CallHandler<any>
  ): Observable<any> | Promise<Observable<any>> {
    return next
      .handle()
      .pipe(catchError((err) => throwError(() => new BadRequestException())));
  }
}
```

```javascript
@UseInterceptors(ErrorInterceptor)
@Get(':id')
findOne(@Param('id') id: string) {
  throw new InternalServerErrorException();
}
```

# 3. 유저 서비스에 인터셉터 적용하기

유저 서비스에는 LoggingInterceptor를 변형하여 들어온 요청과 응답을 로그로 남겨본다.

- logging.interceptor.ts

```javascript
import {
  CallHandler,
  ExecutionContext,
  Injectable,
  Logger,
  NestInterceptor,
} from '@nestjs/common';
import { Observable, tap } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  constructor(private logger: Logger) {}
  intercept(
    context: ExecutionContext,
    next: CallHandler<any>,
  ): Observable<any> | Promise<Observable<any>> {
    const { method, url, body } = context.getArgByIndex(0);
    this.logger.log(`Request to ${method} ${url}`);

    return next.handle().pipe(
      tap((data) => {
        this.logger.log(
          `Response from ${method} ${url} \n response: ${JSON.stringify(data)}`,
        );
      }),
    );
  }
}
```

LoggingInterceptor를 LoggingModule로 분리하여 적용한다.

- logger.module.ts

```javascript
import { Logger, Module } from "@nestjs/common";
import { APP_INTERCEPTOR } from "@nestjs/core";
import { LoggingInterceptor } from "src/logger/logging.interceptor";

@Module({
  providers: [
    Logger,
    { provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
  ],
})
export class LoggerModule {}
```

- app.module.ts

```javascript
...
@Module({
  imports: [
    ...
    LoggerModule,
  ],
  controllers: [],
  providers: [],
})
...
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
