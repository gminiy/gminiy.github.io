---
layout: post
title: "[NestJs] 프로바이더"
date: 2024-02-25
categories: dev
tags: nestjs
---

- [1. 프로바이더](#1-프로바이더)
  - [1.1 프로바이더 등록](#11-프로바이더-등록)
  - [1.2 속성 기반 주입](#12-속성-기반-주입)
- [2. 유저 서비스에 회원 가입 로직 구현](#2-유저-서비스에-회원-가입-로직-구현)
  - [2.1 구현할 기능 목록](#21-구현할-기능-목록)
  - [2.2 UserService 프로바이더 생성](#22-userservice-프로바이더-생성)
  - [2.3 회원 가입](#23-회원-가입)
  - [2.4 회원 가입 이메일 발송](#24-회원-가입-이메일-발송)
  - [2.5 이메일 인증](#25-이메일-인증)
  - [2.6 로그인](#26-로그인)
  - [2.7 유저 정보 조회](#27-유저-정보-조회)

---

# 1. 프로바이더

앱이 제공하고자 하는 핵심 기능, 즉 비즈니스 로직을 수행하는 역하을 하는 것이 프로바이더이다. 컨트롤러가 이 역할을 수행하지 않고 분리함으로써 SRP에 부합하게 만든다.
프로바이더는 service, repository, factory, helper 등 여러 가지 형태로 구현이 가능하다.

Nest에서 제공하는 프로바이더는 따로 라이브러리를 사용하지 않고 의존성을 주입할 수 있다.

- users.controller.ts

```javascript
@Controller('users')
export class UsersController {
    constructor(private readonly usersService: UsersService) { }

    @Delete(':id')
    remove(@Param('id') id: string) {
        return this.usersService.remove(+id);
    }
}
```

컨트롤러는 비즈니스 로직을 직접 수행하지 않는다. UsersController가 UsersService를 생성자를 통해 주입 받아서 멤버 변수에 할당해서 사용한다. 이때 UsersService에는 @Injectable 데코레이터를 선언해서 다른 어떤 Nest 컴포넌트에서도 주입할 수 있는 프로바이더가 된다. 별도의 스코프를 지정해주지 않으면 싱글턴 인스턴스가 된다.

- users.service.ts

```javascript
import { Injectable } from "@nestjs/common";

@Injectable()
export class UsersService {
  remove(id: number) {
    return `This action removes a #${id} user`;
  }
}
```

---

## 1.1 프로바이더 등록

module에서 등록을 해줘야 프로바이더로 사용할 수 있다.

- app.module.ts

```javascript
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { UsersController } from "./users/users.controller";

@Module({
  imports: [],
  controllers: [AppController, UsersController],
  providers: [AppService],
})
export class AppModule {}
```

---

## 1.2 속성 기반 주입

프로바이더를 생성자를 통해 직접 주입받아 사용하지 않고 상속 관계에 있는 자식 클래스를 주입받아 사용하고 싶은 경우에 사용한다.

```javascript
export class BaseService {
  @Inject(ServiceA) private readonly serviceA: ServiceA;

  doSumeFuncFromA(): string {
    return this.serviceA.getHello();
  }
}
```

BaseService 클래스의 ServieA 속성에 @Inject 데코레이터를 선언한다. 프로바이더에 정의된 클래스가 사용된다.

---

# 2. 유저 서비스에 회원 가입 로직 구현

---

## 2.1 구현할 기능 목록

- 회원 가입
- 이메일 인증
- 로그인
- 회원 정보 조회

---

## 2.2 UserService 프로바이더 생성

nest g s Users 명령어로 UsersService 프로바이더를 생성한다. app.module.ts에 UsersService가 자동으로 추가된다.
현재 src 디렉터리 내의 파일 구성은 아래와 같다.

```shell
├── app.controller.ts
├── app.module.ts
├── app.service.ts
├── main.ts
└── users
    ├── UserInfo.ts
    ├── dto
    │   ├── create-user.dto.ts
    │   ├── user-login.dto.ts
    │   └── verify-email.dto.ts
    ├── users.controller.ts
    └── users.service.ts
```

---

## 2.3 회원 가입

회원 가입 요청을 구현한다.
먼저 POST /users 엔드포인트 담당 컨트롤러를 수정한다.
uuid 라이브러리를 설치한다.

```shell
$ npm i uuid
```

- users.controller.ts

```javascript
...
@Controller('users')
export class UsersController {
  constructor(private userService: UsersService) {}
  // 회원 가입
  @Post()
  async createUser(@Body() dto: CreateUserDto): Promise<void> {
    const { name, email, password } = dto;
    await this.userService.createUser(name, email, password);
  }
...
```

UsersService를 생성자를 통해 주입받아 createUser에서 사용한다.

UsersService 다음과 같이 구현한다.

- users.service.ts

```javascript
import { Injectable } from '@nestjs/common';
import * as uuid from 'uuid';

@Injectable()
export class UsersService {
  async createUser(
    name: string,
    email: string,
    password: string,
  ): Promise<void> {
    await this.checkUserExists(email);

    const signupVerifyToken = uuid.v1();

    await this.saveUser(name, email, password, signupVerifyToken);
    await this.sendMemberJoinEmail(email, signupVerifyToken);
  }

  private checkUserExists(email: string) {
    return false; // TODO: DB 연동 후 구현
  }

  private saveUser(
    name: string,
    email: string,
    password: string,
    signupVerifyToken: string,
  ) {
    return; // TODO: DB 연동 후 구현
  }

  private async sendMemberJoinEmail(email: string, signupVerifyToken: string) {
    return; // TODO: EmailService 프로바이더 구현 후 적용
  }
}
```

---

## 2.4 회원 가입 이메일 발송

여기서는 무료 이메일 전송 라이브러리인 nodemailer를 사용한다.

```shell
$ npm i nodemailer
$ npm i -D @types/nodemailer
```

EmailService 프로바이더를 nest g s Email을 통해 생성한다.
UserService에서 EmailService를 주입 받아서 sendMemverJoinEmail에서 emailService의 메소드를 호출한다.

```javascript
import { Injectable } from '@nestjs/common';
import { EmailService } from 'src/email/email.service';
import * as uuid from 'uuid';

@Injectable()
export class UsersService {
  constructor(private emailService: EmailService) {}

  ...

  private async sendMemberJoinEmail(email: string, signupVerifyToken: string) {
    await this.emailService.sendMemberJoinVerification(
      email,
      signupVerifyToken,
    );
  }
}

```

EmailService에서 sendMemberJoinVerification를 구현한다.

```javascript
import Mail from 'nodemailer/lib/mailer';
import * as nodemailer from 'nodemailer';
import { Injectable } from '@nestjs/common';

interface EmailOptions {
  to: string;
  subject: string;
  html: string;
}

@Injectable()
export class EmailService {
  private transporter: Mail;

  constructor() {
    this.transporter = nodemailer.createTransport({
      service: 'Gmail',
      auth: {
        user: 'YOUR_GMAIL',
        pass: 'YOUR_PASSWORD',
      },
    });
  }

  async sendMemberJoinVerification(email: string, signupVerifyToken: string) {
    const baseUrl = 'http://localhost:3000';

    const url = `${baseUrl}/users/email-verify?signupVerifyToken=${signupVerifyToken}`;

    const mailOptions: EmailOptions = {
      to: email,
      subject: '가입 인증 메일',
      html: `
        가입확인 버튼을 누르시면 가입 인증이 완료됩니다.<br/>
        <form action="${url}" method="POST">
          <button>가입확인</button>
        </form>
      `,
    };

    return await this.transporter.sendMail(mailOptions);
  }
}
```

nodemailer는 [구글 앱 비밀번호 로그인](https://support.google.com/accounts/answer/185833?hl=ko) 를 참고하여 앱 비밀번호 설정해서 사용한다.

---

## 2.5 이메일 인증

받은 메일에서 가입확인 버튼을 눌렀을때 컨트롤러가 서비스 로직을 사용하도록 변경한다.

- users.controller.ts

```javascript
...
  // 이메일 인증
  @Post('/email-verify')
  async verifyEmail(@Query() dto: VerifyEmailDto): Promise<void> {
    const { signupVerifyToken } = dto;

    return await this.userService.verifyEmail(signupVerifyToken);
  }
...
```

- users.service.ts

```javascript
...
  async verifyEmail(signupVerifyToken: string) {
    // TODO
    // 1. DB에서 signupVerifyToken으로 회원 가입 처리중인 유저가 있는지 조회하고 없다면 에러처리
    // 2. 바로 로그인 상태가 되도록 JWT 발급
  }
...
```

---

## 2.6 로그인

- users.controller.ts

```javascript
...
  // 로그인
  @Post('login')
  async login(@Body() dto: UserLoginDto): Promise<void> {
    const { email, password } = dto;

    return await this.userService.login(email, password);
  }
...
```

- users.service.ts

```javascript
...
  async login(email, password) {
    // TODO
    // 1. email, password를 가진 유저가 존재하는지 DB에서 확인하고 없다면 에러 처리
    // 2. JWT 발급
  }
...
```

---

## 2.7 유저 정보 조회

- users.controller.ts

```javascript
...
  // 회원 정보 조회
  @Get(':id')
  async getUserInfo(@Param('id') userId: string): Promise<UserInfo> {
    return await this.userService.getUserInfo(userId);
  }
...
```

- users.service.ts

```javascript
...
  async getUserInfo(userId: string): Promise<UserInfo> {
    // TODO
    // 1. UserId를 가진 유저가 존재하는지 DB에서 확인 후 없다면 에러 처리
    // 2. 조회된 데이터를 UserInfo 타입으로 응답

    throw new Error('Method not iplemented');
  }
...
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
