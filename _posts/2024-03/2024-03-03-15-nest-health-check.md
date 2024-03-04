---
layout: post
title: "[NestJs] 헬스 체크"
date: 2024-03-03
categories: dev
tags: nestjs
---

- [1. Terminus 적용](#1-terminus-적용)
- [2. 헬스 체크](#2-헬스-체크)
- [3. TypeOrm 헬스 체크](#3-typeorm-헬스-체크)
- [4. 커스텀 상태 표시기](#4-커스텀-상태-표시기)

---

장애는 어느 레이어에스든 발생할 수 있다. 현재 서비스 상태를 체크할 장치가 필요하고 이를 헬스 체크라고 부른다.
서버는 HTTP, DB, 메모리, 디스크 상태 등을 체크하는 헬스 체크 장치가 있어야 한다. 만약 서버 상태에 문제가 있다면 이를 사내 메신저 등을 통해 즉시 담당자에게 알려야 한다.
Nest 는 Terminus(@nestjs/terminus) 헬스 체크 라이브러리를 제공한다. 이는 다양한 상태 표시기를 제공하며 필요하면 직접 만들어서 사용할 수도 있다.

# 1. Terminus 적용

패키지를 설치한다.

```shell
$ npm i @nestjs/terminus
```

상태 확인은 특정 라우터 엔드포인트로 요청을 보내고 응답을 확인하는 방법을 사용한다. 이를 위해 HealthCheckController 를 생성한다. TerminusModule과 생성한 컨트롤러를 실행할 수 있도록 준비한다.

```shell
$ nest g controller health-check
```

- app.module.ts

```javascript
...
import { TerminusModule } from '@nestjs/terminus';

@Module({
  imports: [
    ,,,
    TerminusModule,
  ],
  controllers: [HealthCheckController],
  providers: [],
})
...
```

# 2. 헬스 체크

HttpHealthIndicator는 동작 과정에서 @nestjs/axios가 필요하므로 설치한다.

```shell
$ npm i @nestjs/axios
```

- app.module.ts

```javascript
...
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';

@Module({
  imports: [
    ,,,
    TerminusModule,
    HttpModule
  ],
  controllers: [HealthCheckController],
  providers: [],
})
...
```

컨트롤러에 HTTP 헬스 체크 코드를 구현한다.

- health-check.controller.ts

```javascript
import { Controller, Get } from '@nestjs/common';
import { HealthCheckService, HttpHealthIndicator } from '@nestjs/terminus';

@Controller('health-check')
export class HealthCheckController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'), // 서비스가 제공하는 다른 서버가 잘 동작하는지 확인
    ]);
  }
}
```

# 3. TypeOrm 헬스 체크

TypeOrmHealthIndicator는 DB 가 잘 살아 있는지 확인한다.

```javascript
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HttpHealthIndicator,
  TypeOrmHealthIndicator,
} from '@nestjs/terminus';

@Controller('health-check')
export class HealthCheckController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
      () => this.db.pingCheck('database'),
    ]);
  }
}
```

# 4. 커스텀 상태 표시기

@nestjs/terminus에서 제공하지 않는 상태 표시기가 필요하다면 HealthIndicator를 상속받는 상태 표시기를 직접 만든다.

```javascript
export declare abstract class HealthIndicator {
    /**
     * Generates the health indicator result object
     * @param key The key which will be used as key for the result object
     * @param isHealthy Whether the health indicator is healthy
     * @param data Additional data which will get appended to the result object
     */
    protected getStatus(key: string, isHealthy: boolean, data?: {
        [key: string]: any;
    }): HealthIndicatorResult;
}
```

예를 들어 강아지들의 상태를 알려주는 DogHealthIndicator라는 상태 표시기를 구현해보자.

```javascript
import { Injectable } from '@nestjs/common';
import {
  HealthCheckError,
  HealthIndicator,
  HealthIndicatorResult,
} from '@nestjs/terminus';

export interface Dog {
  name: string;
  type: string;
}

@Injectable()
export class DogHealthIndicator extends HealthIndicator {
  private dogs: Dog[] = [
    { name: 'Fido', type: 'goodboy' },
    { name: 'Rex', type: 'badboy' },
  ];

  async isHealthy(key: string): Promise<HealthIndicatorResult> {
    const badboys = this.dogs.filter((dog) => dog.type === 'badboy');
    const isHealthy = badboys.length === 0;
    const result = this.getStatus(key, isHealthy, { badboys: badboys.length });

    if (isHealthy) {
      return result;
    }

    throw new HealthCheckError('Dogcheck failed', result);
  }
}
```

- app.module.ts

```javascript
...
import { DogHealthIndicator } from './health-check/dog-health.indicator';
@Module({
  ...
  providers: [DogHealthIndicator],
})
...
```

- health-check.controller.ts

```javascript
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheckService,
  HttpHealthIndicator,
  TypeOrmHealthIndicator,
} from '@nestjs/terminus';
import { DogHealthIndicator } from './dog-health.indicator';

@Controller('health-check')
export class HealthCheckController {
  constructor(
    private health: HealthCheckService,
    private http: HttpHealthIndicator,
    private db: TypeOrmHealthIndicator,
    private dogHealth: DogHealthIndicator,
  ) {}

  @Get()
  check() {
    return this.health.check([
      () => this.http.pingCheck('nestjs-docs', 'https://docs.nestjs.com'),
      () => this.db.pingCheck('database'),
      () => this.dogHealth.isHealthy('dog'),
    ]);
  }
}
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
