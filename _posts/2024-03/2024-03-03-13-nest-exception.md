---
layout: post
title: "[NestJs] 예외 처리"
date: 2024-03-03
categories: dev
tags: nestjs
---

- [1. 예외 처리](#1-예외처리)
- [2. 예외 필터](#2-예외-필터)
- [3. 유저 서비스에 예외 필터 적용하기](#3-유저-서비스에-예외-필터-적용하기)

---

에러 처리기를 따로 만들어 한곳에서 공통으로 처리하도록 한다.

# 1. 예외 처리

Nest는 프레임워크 내에 예외 레이어를 두고 있다.

```javascript
import { InternalServerErrorException } from '@nestjs/common';

@Controller()
export class AppController {
  @Get('/error')
  error(fooL any): string {
    return foo.bar();
  }
}
```

```
{
  "statusCode": 500,
  "message": "Internal Server Error"
}
```

에러 발생 시 응답을 JSON 형식으로 빠궈서 보내주는데 내장된 전역 예외 필터가 이를 처리한다. HttpException은 Error 객체로부터 파생되어 있다. Nest에서 제공하는 모든 Exception은 HttpException을 상속하고 있다.

# 2. 예외 필터

Nest에서 제공하는 전역 예외 필터 외에 직접 예외 필터 레이어를 두어서 원하는 대로 예외를 다룰 수 있다. 예외가 발생했을 때 로그를 남기거나 응 답 객체를 원하는 대로 변경하고자 하는 등의 요구 사항을 해결하고자 할 때 사용한다.

```javascript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  InternalServerErrorException,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch() // 처리되지 않은 모든 예외를 잡을 때 사용
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: any, host: ArgumentsHost): any {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();

    if (!(exception instanceof HttpException)) {
      exception = new InternalServerErrorException();
    }

    const response = (exception as HttpException).getResponse();

    const log = {
      timestamp: new Date(),
      url: req.url,
      response,
    };

	console.log(log);
    res.status((exception as HttpException).getStatus()).json(response);
  }
}
```

예외 필터는 @UseFilter 데커레이터로 컨트롤러에 직접 적용하거나 전역으로 적용할 수 있다. 예외 필터는 전역 필터를 하나만 가지도록 하는 것이 일반적이다.

- 전역으로 적용

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { HttpExceptionFilter } from "./http-exception.filter";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

부트스트랩에서 전역 필터를 적용하면 필터에 의존성을 주입할 수 없다는 제약이 있다. 의존성 주입을 받고자 한다면 예외 필터를 커스텀 프로바이더로 등록한다.
외부 모듈에서 제공하는 Logger 객체를 사용한다면 다음과 같이 구현한다.

```javascript
...
import { Logger } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';
import { HttpExceptionFilter } from './http-exception.filter';

@Module({
  providers: [
    Logger,
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

```javascript
@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  constructor(private logger: Logger) { }
}
```

# 3. 유저 서비스에 예외 필터 적용하기

앞서 만든 HttpExceptionFilter와 LoggerService를 사용한다. HttpExceptionFilter는 Logger를 주입받아 사용한다. 먼저 예외 처리를 위한 ExceptionModule을 생성한다.

- exception.module.ts

```javascript
import { Logger, Module } from "@nestjs/common";
import { APP_FILTER } from "@nestjs/core";
import { HttpExceptionFilter } from "./http-exception.filter";

@Module({
  providers: [Logger, { provide: APP_FILTER, useClass: HttpExceptionFilter }],
})
export class ExceptionModule {}
```

- http-exception.filter.ts

```javascript
import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  InternalServerErrorException,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  constructor(private logger: Logger) {}
  catch(exception: any, host: ArgumentsHost): any {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();
    const req = ctx.getRequest<Request>();

    if (!(exception instanceof HttpException)) {
      exception = new InternalServerErrorException();
    }

    const response = (exception as HttpException).getResponse();
    const stack = exception.stack;
    const log = {
      timestamp: new Date(),
      url: req.url,
      response,
      stack,
    };

    this.logger.log(log);

    res.status((exception as HttpException).getStatus()).json(response);
  }
}
```

- app.module.ts

```javascript
...
@Module({
  imports: [
    ...
    ExceptionModule,
  ],
  controllers: [],
  providers: [],
})
...
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
