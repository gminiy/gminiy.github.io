---
layout: post
title: "[NestJs] 로깅"
date: 2024-03-03
categories: dev
tags: nestjs
---

- [1. 내장 로거](#1-내장-로거)
  - [1.1 로깅 비활성화](#11-로깅-비활성화)
  - [1.2 로그 레벨 지정](#12-로그-레벨-지정)
- [2. 커스텀 로거](#2-커스텀-로거)
  - [2.1 커스텀 로거 주입](#21-커스텀-로거-주입)
  - [2.2 커스텀 로거 전역 사용](#22-커스텀-로거-전역-사용)
- [3. 유저 서비스에 winston 외부 로거 적용하기](#3-유저-서비스에-winston-외부-로거-적용하기)
  - [3.1 nest-winston 적용](#31-nest-winston-적용)
  - [3.2 내장 로거 대체하기](#32-내장-로거-대체하기)
  - [3.3 부트스래핑까지 포함하여 내장 로거 대체](#33-부트스래핑까지-포함하여-내장-로거-대체)

---

Nest 로깅 옵션을 조절하면 다음과 같은 로깅 시스템의 동작을 제어할 수 있다.

- 로깅 비활성화
- 로그 레벨 지정: log, error, warn, debug, verbose
- 로거의 타임스탬프 재정의
- 기본 로거 재정의
- 기본 로거 확장을 통한 커스텀 로거
- 의존성 주입을 통한 로거 주입 및 테스트 모듈 제공

# 1. 내장 로거

내장 로거 인스턴스를 직접 생성하여 사용할 수 있다.

```javascript
import { Injectable, Logger } from '@nestjs/common';

@Injectable()
export class AppService {
  // 로거 인스턴스 생성 시 클래스명을 컨텍스트로 설정하여 로그 메시지 앞에 클래스명이 함께 출력되도록 함.
  private readonly logger = new Logger(AppService.name);
  getHello(): string {
    this.logger.error('this is error');
    this.logger.warn('this is warn');
    this.logger.log('this is log');
    this.logger.verbose('this is verbose');
    this.logger.debug('this is debug');

    return 'Hello World!';
  }
}
```

## 1.1 로깅 비활성화

아래와 같이 logger 옵션을 false로 하면 로그가 출력되지 않는다.

- main.ts

```javascript
...
async function bootstrap() {
  //const app = await NestFactory.create(AppModule);
  // 로그 비활성화
  const app = await NestFactory.create(AppModule, {
    logger: false,
  });
  await app.listen(3000);
}
...
```

## 1.2 로그 레벨 지정

프로덕션 환경에서는 debug 로그가 남지 않도록 한다. 이를 동적으로 지정한다.

```javascript
async function bootstrap() {
  //const app = await NestFactory.create(AppModule);
  // 로그 비활성화
  const app = await NestFactory.create(AppModule, {
    logger:
      process.env.NODE_ENV === "prod" ? ["error", "warn", "log"] : ["debug"],
  });
  await app.listen(3000);
}
```

# 2. 커스텀 로거

로그를 저장하는 기능을 내장 로거는 제공하지 않는다. 이를 위해서는 커스텀 로거를 만들어야 한다.
커스텀 로거는 LoggerService 인터페이스를 구현한다.

```javascript
export interface LoggerService {
    log(message: any, ...optionalParams: any[]): any;
    error(message: any, ...optionalParams: any[]): any;
    warn(message: any, ...optionalParams: any[]): any;
    debug?(message: any, ...optionalParams: any[]): any;
    verbose?(message: any, ...optionalParams: any[]): any;
    setLogLevels?(levels: LogLevel[]): any;
}
```

커스텀 MyLogger를 만들어보자.

```javascript
import { LoggerService } from '@nestjs/common';

export class MyloggerService implements LoggerService {
  debug(message: any, ...optionalParams: any[]): any {
    console.log(message);
  }

  error(message: any, ...optionalParams: any[]): any {
    console.log(message);
    this.doSomething();
  }

  private doSomething() {
    // DB 저장과 같은 부가 로직 추가
  }

  log(message: any, ...optionalParams: any[]): any {
    console.log(message);
  }

  verbose(message: any, ...optionalParams: any[]): any {
    console.log(message);
  }

  warn(message: any, ...optionalParams: any[]): any {
    console.log(message);
  }
}
```

## 2.1 커스텀 로거 주입

로거를 매번 생성해서 사용하는 것이 아니라 모듈로 만들어서 생성자에서 주입바아 사용한다.
먼저 LoggerModule을 만들고 AppModule에 가지고 온다.

```javascript
import { Module } from "@nestjs/common";
import { MyloggerService } from "./mylogger.service";

@Module({
  providers: [MyloggerService],
  exports: [MyloggerService],
})
export class LoggerModule {}
```

```javascript
...
import { LoggerModule } from './logging/logger.module';

@Module({
  imports: [LoggerModule],
  ...
})
export class AppModule {}
```

프로바이더로 MyLogger를 주입받아 사용한다.

```javascript
import { Injectable } from '@nestjs/common';
import { MyloggerService } from './logging/mylogger.service';

@Injectable()
export class AppService {
  constructor(private myLogger: MyloggerService) {}

  getHello(): string {
    this.myLogger.error('test');

    return 'Hello World!';
  }
}
```

## 2.2 커스텀 로거 전역 사용

- main.ts

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { MyloggerService } from "./logging/mylogger.service";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useLogger(app.get(MyloggerService));
  await app.listen(3000);
}
bootstrap();
```

# 3. 유저 서비스에 winston 외부 로거 적용하기

nodejs 의 winston 을 nest 용으로 만든 패키지인 nest-winston을 사용한다.

## 3.1 nest-winston 적용

nest-winston 을 설치한다.

```shell
$ npm i nest-winston winston
```

AppModule에 WinstonModule을 import 한다. 이때 winston 옵션을 줄 수 있다.

- app.module.ts

```javascript
...
import { WinstonModule, utilities } from 'nest-winston';
import * as winston from 'winston';
...
@Module({
  imports: [
    ...
    WinstonModule.forRoot({
      transports: [
        new winston.transports.Console({
          level: process.env.NODE_ENV === 'production' ? 'info' : 'silly',
          format: winston.format.combine(
            winston.format.timestamp(), // 로그 시각 표시
            utilities.format.nestLike('MyApp', { // 어디에서 로그를 남겼는지 구분하는 appName('MyApp') 설정
              prettyPrint: true,
            }),
          ),
        }),
      ],
    }),
...

```

winston이 지원하는 로그 레벨은 다음과 같다. 설정된 로그 레벨보다 레벨이 높은 로그는 함께 출력된다.

```javascript
{
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  verbose: 4,
  debug: 5,
  silly: 6
}
```

WINSTON_MODULE_PROVIDER 토큰으로 Logger 객체를 주입받을 수 있다.

- users.controller.ts

```javascript
...
import { Logger as WinstonLogger } from 'winston';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
...
export class UsersController {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: WinstonLogger,
    private userService: UsersService,
  ) {}
  ...
    private printWinstonLog(dto) {
    this.logger.error('error: ', dto);
    this.logger.warn('warn: ', dto);
    this.logger.info('info: ', dto);
    this.logger.http('http: ', dto);
    this.logger.verbose('verbose: ', dto);
    this.logger.debug('debug: ', dto);
    this.logger.silly('silly: ', dto);
  }
...

```

```
[MyApp] Error   2024. 3. 3. 오후 5:43:45 error:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
[MyApp] Warn    2024. 3. 3. 오후 5:43:45 warn:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
[MyApp] Info    2024. 3. 3. 오후 5:43:45 info:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
[MyApp] Http    2024. 3. 3. 오후 5:43:45 http:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
[MyApp] Verbose 2024. 3. 3. 오후 5:43:45 verbose:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
[MyApp] Debug   2024. 3. 3. 오후 5:43:45 debug:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
[MyApp] Silly   2024. 3. 3. 오후 5:43:45 silly:  - { name: 'ga', email: 'gmin2i.y@gmail.com', password: '123456788hfds' }
```

## 3.2 내장 로거 대체하기

Nest가 시스템 로깅을 할 때 출력하는 로그와 직접 출력하고자 하는 로깅의 형식을 동일하게 할 수 있다.

먼저 main.ts에 전역 로거로 설정한다.

- main.ts

```javascript
...
import { WINSTON_MODULE_NEST_PROVIDER } from 'nest-winston';

async function bootstrap() {
  ...
  app.useLogger(app.get(WINSTON_MODULE_NEST_PROVIDER));
  ...
}
bootstrap();
```

로깅을 하고자 하는 곳에서 LoggerServicefmf WINSTON_MODULE_NEST_PROVIDER 토큰으로 주입받는다.

- users.controller.ts

```javascript
...
export class UsersController {
  constructor(
    @Inject(WINSTON_MODULE_NEST_PROVIDER)
    private readonly logger: WinstonLogger,
    private userService: UsersService,
  ) {}
  ...
    // 회원 가입
  @Post()
  async createUser(@Body() dto: CreateUserDto): Promise<void> {
    this.printLoggerServiceLog(dto);
    const { name, email, password } = dto;
    await this.userService.createUser(name, email, password);
  }
  ...
  private printLoggerServiceLog(dto) {
    try {
      throw new InternalServerErrorException('test');
    } catch (e) {
      this.logger.error('error: ' + JSON.stringify(dto), e.stack);
    }
    this.logger.warn('warn: ', JSON.stringify(dto));
    this.logger.log('log: ', JSON.stringify(dto));
    this.logger.verbose('verbose: ', JSON.stringify(dto));
    this.logger.debug('debug: ', JSON.stringify(dto));
  }
}
```

```
[MyApp] Error   2024. 3. 3. 오후 5:54:07 error: {"name":"ga","email":"gmin22i.y@gmail.com","password":"123456788hfds"} - {
  stack: [
    'InternalServerErrorException: test\n' +
      '    at UsersController.printLoggerServiceLog (/Users/jimimyeon/Documents/workspace/nest-basic-project/src/users/users.controller.ts:82:13)\n' +
      '    at UsersController.createUser (/Users/jimimyeon/Documents/workspace/nest-basic-project/src/users/users.controller.ts:39:10)\n' +
      '    at /Users/jimimyeon/Documents/workspace/nest-basic-project/node_modules/@nestjs/core/router/router-execution-context.js:38:29\n' +
      '    at processTicksAndRejections (node:internal/process/task_queues:95:5)\n' +
      '    at /Users/jimimyeon/Documents/workspace/nest-basic-project/node_modules/@nestjs/core/router/router-execution-context.js:46:28\n' +
      '    at /Users/jimimyeon/Documents/workspace/nest-basic-project/node_modules/@nestjs/core/router/router-proxy.js:9:17'
  ]
}
[MyApp] Warn    2024. 3. 3. 오후 5:54:07 [{"name":"ga","email":"gmin22i.y@gmail.com","password":"123456788hfds"}] warn:
[MyApp] Info    2024. 3. 3. 오후 5:54:07 [{"name":"ga","email":"gmin22i.y@gmail.com","password":"123456788hfds"}] log:
[MyApp] Verbose 2024. 3. 3. 오후 5:54:07 [{"name":"ga","email":"gmin22i.y@gmail.com","password":"123456788hfds"}] verbose:
[MyApp] Debug   2024. 3. 3. 오후 5:54:07 [{"name":"ga","email":"gmin22i.y@gmail.com","password":"123456788hfds"}] debug:
```

nest-winston 모듈이 적용되어 MyAPP 태그가 붙어있다.

## 3.3 부트스래핑까지 포함하여 내장 로거 대체

이 부분은 책보다 간단하게 설정하는 방법은 NestFactory.create 에 bufferLogs를 true 로 주면 된다. ([블로그 참고](https://assu10.github.io/dev/2023/03/26/nest-logging))

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "@nestjs/common";
import { WINSTON_MODULE_NEST_PROVIDER } from "nest-winston";

async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });
  app.useGlobalPipes(new ValidationPipe({ transform: true }));
  app.useLogger(app.get(WINSTON_MODULE_NEST_PROVIDER));
  await app.listen(3000);
}
bootstrap();
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
