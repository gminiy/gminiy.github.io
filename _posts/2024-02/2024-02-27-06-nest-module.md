---
layout: post
title: "[NestJs] 모듈 설계"
date: 2024-02-27
categories: dev
tags: nestjs
---

- [1. 모듈](#1-모듈)
  - [1.1 모듈 다시 내보내기](#11-모듈-다시-내보내기)
  - [1.2 전역 모듈](#12-전역-모듈)
- [2. 유저 서비스 모듈 분리](#2-유저-서비스-모듈-분리)
  - [2.1 UserModule 분리](#21-usermodule-분리)
  - [2.2 EmailModule 분리](#22-emailmodule-분리)

---

# 1. 모듈

모듈이란 여러 컴포넌트를 조합하여 더 큰 작업을 수행할 수 있게 하는 단위를 말한다.
Nest 애플리케이션이 실행되기 위해서는 하나의 루트 모듈이 존재하고 이 루트 모듈은 다른 모듈들로 구성된다. 이를 통해 책임을 나누고 응집도를 높일 수 있다.
모듈을 어떻게 나눌 것인지는 명확한 기준은 없지만 유사한 기능끼리 모듈로 묶어서 설계하게 될 것이다.

모듈은 @Module 데커레이터를 사용한다. @Module 은 ModuleMetadata를 인수로 받으며 ModuleMetadata는 아래와 같이 정의된다.

```javascript
export declare function Module(metadata: ModuleMetadata): ClassDecorator;

export interface ModuleMetadata {
  imports?: Array<Type<any> | DynamicModule | Promise<DynamicModule> | ForwardReference>;
  controllers?: Type<any>[];
  providers?: Provider[];
  exports?: Array<DynamicModule | Promise<DynamicModule> | string | symbol | Provider | ForwardReference | Abstract<any> | Function>;
}
```

- import : 이 모듈에서 사용하기 위한 프로바이더를 가지고 있는 다른 모듈을 가지고 온다.
- controllers / providers : 컨트롤러와 프로바이더를 모듈에서 사용할수 있도록 Nest가 객체를 생성하고 주입할 수 있게 해준다.
- exports : 이 모듈에서 제공하는 컴포넌트를 다른 모듈에서 import 해서 사용하고자 한다면 export를 해야한다. export로 선언했다는 뜻은 어디서든 쓸 수 있으므로 public 인터페이스 또는 API로 간주된다.

---

## 1.1 모듈 다시 내보내기

가져온 모듈은 다시 내보내기가 가능하다. 예를 들어 AppModule이 CoreModule과 CommonModule의 기능이 모두 필요하다면 AppModule은 모두를 가지고 오는 것이 아니라 CoreModule만을 가져오고, CoreModule에서는 CommonModule을 내보내면 AppModule에서 CommonModule을 가지고 오지 않아도 사용할 수 있다.

- CommonModule.ts

```javascript
@Module({
  providers: [CommonService],
  exports: [CommonService], // CommonService 제공
})
export class CommonModule {}
```

- CommonService.ts

```javascript
@Injectable()
export class CommonService {
  hello(): string {
    return "Common Hello~";
  }
}
```

- CoreModule.ts

```javascript
@Module({
  imports: [CommonModule],
  exports: [CommonModule],
})
export class CoreModule {}
```

- AppModule.ts

```javascript
@Module({
  imports: [CoreModule],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

- AppController.ts

```javascript
@Controller()
export class AppController {
  constructor(private readonly commonService: CommonService) {}

  @Get('/common-hello')
  getCommonHello(): string {
    return this.commonService.hello();
  }
}
```

---

## 1.2 전역 모듈

Nest는 모듈 범위 내에서 프로바이더를 캡슐화한다. 따라서 어떤 모듈에 있는 프로바이더를 사용하려면 모듈을 먼저 import 해야한다.
하지만 헬퍼나 DB 연결 등의 전역적으로 쓸 수 있는 프로바이더가 필요한 경우 전역 모듈로 제공할 수 있다.
전역 모듈은 @Global 데커레이터를 선언한다. 전역 모듈은 루트 모듈이나 코어 모듈에서 한 번만 등록해야 한다.

```javascript
@Global()
@Module({
  providers: [CommonService],
  exports: [CommonService],
})
export class CommonModule {}
```

---

# 2. 유저 서비스 모듈 분리

만들고 있는 유저 서비스는 루트 모듈인 AppModule만 존재한다. 유저 관리 기능을 UserModule로, 이메일 기능은 EmailModule로 분리하는 작업을 진행하자.

---

## 2.1 UserModule 분리

UserModule을 아래 cli 를 통해 생성한다.

```shell
$ nest g mo Users
```

생성된 UserModule에 UserController와 UserService를 추가한다. EmailService를 사용하므로 함께 추가한다.

- user.module.ts

```javascript
import { Module } from "@nestjs/common";
import { UsersController } from "./users.controller";
import { UsersService } from "./users.service";
import { EmailService } from "src/email/email.service";

@Module({
  imports: [],
  controllers: [UsersController],
  providers: [UsersService, EmailService],
})
export class UsersModule {}
```

AppModule에 UserModule이 자동으로 import 되어 있고 AppModule에서 UsersController, UsersService, EmailService를 참조할 필요가 없으므로 제거한다.

- app.module.ts

```javascript
import { Module } from "@nestjs/common";
import { UsersModule } from "./users/users.module";

@Module({
  imports: [UsersModule],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

---

## 2.2 EmailModule 분리

이메일 모듈을 생성한다.

```shell
$ nest g mo Email
```

생성된 EmailModule에서 EmailService를 UserModule에서 사용할수 있도록 exports 해준다.

- email.module.ts

```javascript
import { Module } from "@nestjs/common";
import { EmailService } from "./email.service";

@Module({
  providers: [EmailService],
  exports: [EmailService],
})
export class EmailModule {}
```

UsersModule에서 EmailModule을 import해주고 EmailService는 삭제한다.

- users.module.ts

```javascript
import { Module } from "@nestjs/common";
import { UsersController } from "./users.controller";
import { UsersService } from "./users.service";
import { EmailModule } from "src/email/email.module";

@Module({
  imports: [EmailModule],
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
