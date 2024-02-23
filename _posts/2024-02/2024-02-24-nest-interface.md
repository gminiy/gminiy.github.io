---
layout: post
title: "[NestJs] 인터페이스"
date: 2024-02-24
categories: dev
tags: nestjs
---

> - [1. 컨트롤러](#1-컨트롤러)
>   - [1.1 라우팅](#11-라우팅)
>   - [1.2 요청 객체](#12-요청-객체)
>   - [1.3 응답](#13-응답)
>   - [1.4 헤더](#14-헤더)
>   - [1.5 리디렉션](#15-리디렉션)
>   - [1.6 라우트 매개변수](#16-라우트-매개변수)
>   - [1.7 하위 도메인 라우팅](#17-하위-도메인-라우팅)
>   - [1.9 페이로드 다루기](#19-페이로드-다루기)
> - [2. 유저 서비스의 인터페이스](#2-유저-서비스의-인터페이스)

---

# 1. 컨트롤러

Nest의 컨트롤러는 MVC 패턴에서 말하는 그 컨트롤러를 말한다. 컨트롤ㄹ러는 들어오는 요청을 받고 처리된 결과를 응답으로 돌려주는 인터페이스 역할을 한다.

컨트롤러를 작성해보자.

- 프로젝트 생성

```shell
$ nest new my-project
```

- cli를 통한 컨트롤러 생성

```shell
$ nest g controller Users
```

AppModule 에서 생성한 users.controller.ts를 import 하는 부분이 추가된다.

- cli를 통해 보일러 플레이트를 한번에 만들 수 있다.
  module, controller, service, entity, dto, test 코드 등이 한번에 만들어진다.

```shell
$ nest g resource Users
? What transport layer do you use? REST API
? Would you like to generate CRUD entry points? Yes
CREATE src/users/users.controller.spec.ts (566 bytes)
CREATE src/users/users.controller.ts (894 bytes)
CREATE src/users/users.module.ts (248 bytes)
CREATE src/users/users.service.spec.ts (453 bytes)
CREATE src/users/users.service.ts (609 bytes)
CREATE src/users/dto/create-user.dto.ts (30 bytes)
CREATE src/users/dto/update-user.dto.ts (169 bytes)
CREATE src/users/entities/user.entity.ts (21 bytes)
UPDATE package.json (1990 bytes)
UPDATE src/app.module.ts (389 bytes)
✔ Packages installed successfully.
```

---

## 1.1 라우팅

- app.controller.ts

```javascript
import { Controller, Get } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

@Controller 데코레이터를 통해 해당 클래스는 컨트롤러 역할을 하게 된다.

getHello 함수는 @GET 데코레이터를 통해 라우팅된다. 라우팅 경로는 @Get 데코레이터의 인수로 관리된다. @Controller 데코레이터에도 인수를 전달하여 라우팅 경로의 접두어를 지정할 수 있다.
예를 들어 @Controller('app')이라고 하면 http://localhost:3000/app/hello 로 접근된다.

라우팅 패스는 와일드 카드를 이용하여 작성할 수 있다. @Get(‘/he*lo’) 로 설정하면 hello, helo, he**\_**o 등으로 접근가능하고, * 외 ?, +, () 역시 정규 표현식에서의 와일드 카드와 동일하게 동작한다. 단, -, . 은 문자열로 취급한다.

---

## 1.2 요청 객체

Nest는 요청과 함께 전달되는 데이터를 핸들러가 다룰 수 있는 객체로 변환한다. 이 객체는 @Req 데코레이터로 다룰 수 있다.

- app.controller.ts

```javascript
import { Controller, Get, Req } from '@nestjs/common';
import { AppService } from './app.service';

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(@Req() req: Request): string {
    console.log(req);
    return this.appService.getHello();
  }
}
```

---

## 1.3 응답

요청의 성공 응답 코드는 POST일 경우 201이고, 나머지는 모두 200이다. Nest는 응답을 어떤 방식으로 처리할지 미리 정의해두었다. 객체를 리턴한다면 직렬화를 통해 JSON으로 자동 변환해준다.
@Res 데코레이터를 이용해서 응답 객체를 다룰 수 있다.

```javascript
@Get()
findAll(@Res() res) {
    const users = this.usersService.findAll();
    return res.status(200).send(users);
}
```

응답 코드를 변경하고 싶을때는 @HttpCode로 변경할수 있다.

```javascript
import { HttpCode } from '@nestjs/common';

@HttpCode(202)
@Get()
findAll() {
    return this.usersService.findAll();
}
```

예외 처리로 에러 코드를 보낼때는 아래처럼 정의된 Exception 객체들을 사용해서 처리한다.

```javascript
  @Get(':id')
  findOne(@Param('id') id: string) {
    if (+id < 1) {
      throw new BadRequestException('id error');
    }
    return this.usersService.findOne(+id);
  }
```

---

## 1.4 헤더

Nest는 헤더를 자동으로 구성해준다. 만약 커스텀 헤더를 추가하고 싶다면 @Header 데코레이터를 사용하면 된다. 인수로 헤더 이름과 값을 받는다.

```javascript
  @Header('Custom', 'Test Header')
  @Get(':id')
  findOne(@Param('id') id: string) {
    if (+id < 1) {
      throw new BadRequestException('id error');
    }
    return this.usersService.findOne(+id);
  }
```

---

## 1.5 리디렉션

클라이언트를 다른 페이지로 이동하고 싶은 경우 @Redirect 데코레이터를 사용해 구현할수 있다. 두번째 인자는 상태 코드이다.

```javascript
  @Redirect('https://google.com', 301)
  @Get(':id')
  findOne(@Param('id') id: string) {
    if (+id < 1) {
      throw new BadRequestException('id error');
    }
    return this.usersService.findOne(+id);
  }
```

요청 결과에 따라 동적으로 리디렉트할 때는 응답으로 다음 객체를 리턴하면 된다.

```javasciprt
{
	"url": string,
    "statusCode": number
}
```

예를 들어 다음처럼 구현할 수 있다.

```javascript
@Get('redirect/docs')
@Redirect('https://docs.nestjs.com', 302)
getDocs(@Query('version') version) {
  if (version && version === '5') {
    return { url: 'https://docs.nestjs.com/v5/'};
  }
}
```

---

## 1.6 라우트 매개변수

전달받은 매개변수는 함수 인수에 @Param 데코레이터 주입받을 수 있다.

```javascript
@Delete(':userId/memo/:memoId')
deleteUserMemo(
@Param('userId') userId: string,
@Param('memoId') memoId: string,
) {
	return `userId: ${userId}, memoId: ${memoId}`;
}
```

---

## 1.7 하위 도메인 라우팅

http://example.com, http://api.example.com 각 들어온 요청을 다르게 처리하고 싶을때 사용한다.

먼저 새로운 컨트롤러를 생성한다.

```shell
$ nest g co Api
```

ApiController가 먼저 처리되도록 라우팅 순서를 변경한다.

```javascript
@Module({
  imports: [UsersModule],
  controllers: [ApiController, AppController],
  providers: [AppService],
})
export class AppModule {}
```

@Controller 데코레이터는 ControllerOptions 객체를 인수로 받는데, host 속성에 하위 도메인을 기술하면 된다.

```javascript
@Controller({ host: "api.localhost" }) // 하위 도메인 요청 처리 설정
export class ApiController {
  @Get() // 같은 루트 경로
  index(): string {
    return "Hello Api"; // 다른 응답
  }
}
```

@HostParam 데코레이터를 통해 서브 도메인을 변수로 받을 수 있다.

```javascript
@Controller({ host: ":version.api.localhost" })
export class ApiController {
  @Get()
  index(@HostParam("version") version: string): string {
    return `Hello, API ${version}`;
  }
}
```

---

## 1.9 페이로드 다루기

NestJS에서는 데이터 전송 객체 (DTO)가 구현되어 있어 body를 쉽게 다룰 수 있다.

회원 가입 처리를 위해 이름과 이메일을 추가해보자.

- create-user.dto.ts

```javascript
export class CreateUserDto {
  name: string;
  email: string;
}
```

- users.controller.ts

```javascript
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    const { name, email } = createUserDto;

    return `Created User. name: ${name}, Email: ${email}`;
  }
```

---

# 2. 유저 서비스의 인터페이스

이 과정에서는 다음과 같이 요청에 대한 인터페이스를 정의한다.

| 기능           | end-point                | body(json)                                                       | path parameter | response        |
| :------------- | :----------------------- | :--------------------------------------------------------------- | :------------- | :-------------- |
| 회원 가입      | POST /users              | { "name": "assu", email": "email@test.com", "password": "abcd" } |                | 201             |
| 이메일 인증    | POST /users/email-verify | { "signupVerifyToken": "fdsafdsa"                                |                | 201 AccessToken |
| 로그인         | POST /users/login        | { "email": "email@test.com", "password": "fdsafd" }              |                | 201 AccessToken |
| 회원 정보 조회 | GET /users/:id           |                                                                  | id             | 200 회원 정보   |

먼저 컨트롤러를 만든다.

```shell
$ nest g co users
```

회원 가입 인터페이스 구현

- users.controller.ts

```javascript
import { Body, Controller, Post } from "@nestjs/common";
import { CreateUserDto } from "./dto/create-user.dto";

@Controller("users")
export class UsersController {
  @Post()
  async createUser(@Body() dto: CreateUserDto): Promise<void> {
    console.log(dto);
  }
}
```

- dto/create-user.dto.ts

```javascript
export class CreateUserDto {
  readonly name: string;
  readonly email: string;
  readonly password: string;
}
```

나머지 인터페이스 구현

- users.controller.ts

```javascript
import { Body, Controller, Get, Param, Post, Query } from "@nestjs/common";
import { CreateUserDto } from "./dto/create-user.dto";
import { VerifyEmailDto } from "./dto/verify-email.dto";
import { UserLoginDto } from "./dto/user-login.dto";
import { UserInfo } from "./UserInfo";

@Controller("users")
export class UsersController {
  // 회원 가입
  @Post()
  async createUser(@Body() dto: CreateUserDto): Promise<void> {
    console.log("createUser dto: ", dto);
  }

  // 이메일 인증
  @Post("/email-verify")
  async verifyEmail(@Query() dto: VerifyEmailDto): Promise<string> {
    console.log("verifyEmail dto: ", dto);
    return;
  }

  // 로그인
  @Post("login")
  async login(@Body() dto: UserLoginDto): Promise<string> {
    console.log("login dto: ", dto);
    return;
  }

  // 회원 정보 조회
  @Get(":id")
  async getUserInfo(@Param("id") userId: string): Promise<UserInfo> {
    console.log("getUserInfo userId: ", userId);
    return;
  }
}
```

- dto/verify-email.dto.ts

```javascript
export class VerifyEmailDto {
  signupVerifyToken: string;
}
```

- dto/user-login.dto.ts

```javascript
export class UserLoginDto {
  email: string;
  password: string;
}
```

- UserInfo.ts
  유저 정보는 다음과 같은 인터페이스로 구성된다.

```javascript
export interface UserInfo {
  id: string;
  name: string;
  email: string;
}
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
