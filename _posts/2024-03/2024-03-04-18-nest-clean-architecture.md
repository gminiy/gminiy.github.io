---
layout: post
title: "[NestJs] 클린 아키텍처"
date: 2024-03-04
categories: dev
tags: nestjs
---

- [1. 클린 아키텍처](#1-클린-아키텍처)
- [2. SOLID 객체 지향 설계 원칙](#2-solid-객체-지향-설계-원칙)
- [3. 유저 서비스에 클린 아키텍처 적용하기](#3-유저-서비스에-클린-아키텍처-적용하기)
  - [3.1 도메인 레이어](#31-도메인-레이어)
  - [3.2 어플리케이션 레이어](#32-어플리케이션-레이어)
  - [3.3 인터페이스 레이어](#33-인터페이스-레이어)
  - [3.4 인프라 레이어](#34-인프라-레이어)
  - [3.5 이메일 모듈](#35-이메일-모듈)

---

# 1. 클린 아키텍처

클린 아키텍처는 소프트웨어를 여러 동심원 레이어로 나누고 각 레이어에 있는 컴포넌트는 안쪽 원에 있는 컴포넌트에만 의존성을 가지게 한다. 따라서 안쪽 컴포넌트는 바깥 원에 독립적이다.
이 책에서는 클린 아키텍처를 4개의 레이어로 분류했다.

- Infrastructure : 외부에서 가져다 쓰는 컴포넌트를 배치한다. 예를 들어 데이터베이스, 이메일 전송, 다른 서비스와의 통신 프로토콜 구현체 등 외부에서 제공하는 인터페이스나 라이브러리를 이용하여 우리 서비스에 맞게 구현한 구현체가 포함된다.
- Interface : 서비스가 제공하는 인터페이스가 구현되는 레이어. 컨트롤러의 외부 요청 형식, 나가는 데이터의 형식 등을 정의한다. 마찬가지로 게이트웨이나 프레젠터 같은 컴포넌트도 외부와의 인터페이스를 담당한다.
- Application : 비즈니스 로직이 구현되는 레이어이다. 서비스 로직들이 여기 존재한다.
- Domain : 애플리케이션의 핵심 도메인을 구현하는 레이어이다. 도메인 레이어는 다른 레이어에 의존하지 않는다. 따라서 애플리케이션이 가져야 하는 핵심 요소만 가진다.

클린 아키텍처 작성시 의존성 역전 원칙을 이해해야 한다. 각 레이어는 의존성이 안쪽 원으로 향하는데 구현할때 안쪽 원에서 바깥쪽 원의 구현체가 필요한 경우가 많다. 그때 안쪽 레이어에서는 인터페이스를 정의하고 인터페이스를 구현한 구현체는 바깥 레이어에 둠으로써 해결할 수 있다.

# 2. SOLID 객체 지향 설계 원칙

SOLID는 로버트 C. 마틴이 객체 지향 언어로 소프트웨어를 설계할 때의 방법론을 정리한 것이다. SOLID를 적용하면 유지 보수와 확장이 쉬운 시스템을 만들 수 있다.

- SRP: 단일 책임 원칙
  - 이 원칙의 더 정확한 정의는 클래스를 변경하는 이유는 오직 한 가지뿐이어야 한다는 것이다. 클래스를 비대하게 설계하지 말고 크기가 작고 적은 책임을 가지도록 작성해서 변경에 유연하게 대처할 수 있도록 한다.
- OCP : 개방-폐쇄 원칙
  - 확장에는 열려있고 변경에는 닫혀 있어야 한다. 소프트웨어의 요구 사항이 추가되었다고 해서 기존의 소스 코드를 고쳐야 하면 안된다. 즉, 기능의 추가가 기존 코드에는 영향을 끼치지 않도록 하는 구조가 필요하다. 이는 인터페이스를 활용하여 달성할 수 있다. 기능이 구현체가 아닌 인터페이스에 의존하도록 한다. 추가 기능은 인터페이스를 추가한다.
- LSP : 리스코프 치환 원칙
  - 프로그램의 객체는 프로그램의 정확성을 깨뜨리지 않으면서 하위 타입의 인스턴스로 변경 가능해야 한다. 상속 관계에서 자식 클래스의 인스턴스는 부모 클래스로 선언된 함수의 인수로 전달할 수 있다. 실제 동작하는 인스턴스는 인터페이스가 제공하는 기능을 구현한 객체이지만 인터페이스를 사용하는 다른 객체에도 전달할 수 있어야 한다. 따라서 실제 구현체인 자식 인스턴스는 언제든지 부모 또는 인터페이스가 제공해야 하는 기능을 제공하는 다른 구현체로 바꿀 수 있다.
- ISP : 인터페이스 분리 원칙
  - 특정 클라이언트를 위한 인터페이스 여러 개가 범용 인터페이스 하나보다 낫다. 하나의 인터페이스에 의존하게 되면 인터페이스에 기능이 추가될 때 인터페이스를 구현하는 모든 클래스를 수정해야 한다. 이보다는 인터페이스를 기능별로 잘게 쪼개어 특정 클라이언트용 인터페이스로 모아 사용하는 것이 변경에 대해 의존성을 낮추고 유연하게 대처할 수 있는 방법이다.
- DIP : 의존관계 역전 원칙
  - 프로그래머는 추상화에 의존해야지 구체화에 의존하면 안된다. 클린 아키텍처를 구현하기 위해서는 의존관계 역전이 발생하기 마련이고 이를 해소하기 위해 DI를 이용해야 한다. 이는 추후에 살펴본다.

# 3. 유저 서비스에 클린 아키텍처 적용하기

지금까지 레이어 없이 기능별 역할에 따라 Module로 분리했다. 이제 우리가 작성하고 있는 유저 서비스에서 유저 모듈로 그 범위를 좁혀 클린 아키텍처를 적용해보자.
먼저 다음과 같이 4개의 레이어와 모든 레이어에서 공통으로 사용하는 컴포넌트를 작성할 common 디렉터리를 만든다.

```
/src/users
├── application
├── common
├── domain
├── infra
├── interface
```

## 3.1 도메인 레이어

가장 안쪽 레이어인 domain 레이어에는 도메인 객체와 그 도메인 객체의 상태 변화에 따라 발생되는 이벤트가 존재한다. UserModule은 User 도메인 객체를 갖는다.

- domain/user.ts

```javascript
export class User {
  constructor(
    private id: string,
    private name: string,
    private email: string,
    private password: string,
    private signupVarificationToken: string,
  ) {}
}
```

User 객체를 생성할 대 UserCreatedEvent 를 발송해야 한다. 이 도메인 이벤트를 발송하는 주체는 User의 생성자가 되어야 하는데 User 클래스는 new 키워드로 생성하므로 EventBus를 주입받을 수가 없다. 따라서 User를 생성하는 팩터리 클래스인 UserFactory를 구현해서 이를 프로바이더로 제공한다.

- domain/user.factory.ts

```javascript
import { EventBus } from '@nestjs/cqrs';
import { UserCreateEvent } from '../event/user-create.event';
import { User } from './user';

export class UserFactory {
  constructor(private eventBus: EventBus) {}

  create(
    id: string,
    name: string,
    email: string,
    signupVarificationToken: string,
    password: string,
  ) {
    const user = new User(id, name, email, password, signupVarificationToken);
    this.eventBus.publish(new UserCreateEvent(email, signupVarificationToken));
    return user;
  }
}
```

- users.module.ts

```javascript
...
import { UserFactory } from './domain/user.factory';

@Module({
  ...
  providers: [
	...
    UserFactory,
  ],
})
export class UsersModule {}
```

UserCreatedEvent는 도메인 레이어에 위치시킨다.

```javacript
├── domain
│   ├── cqrs-event.ts
│   ├── user-create.event.ts
│   ├── user.factory.ts
│   └── user.ts
```

## 3.2 어플리케이션 레이어

비즈니스 로직이 구현되는 application 레이어를 구현한다. UserEventsHandler를 어플리케이션 레이어로 이동하고 TestHandler는 삭제한다.

- application/user-event.handler.ts

```javascript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { UserCreateEvent } from '../domain/user-create.event';
import { EmailService } from 'src/email/email.service';

@EventsHandler(UserCreateEvent)
export class UserEventsHandler implements IEventHandler<UserCreateEvent> {
  constructor(private emailService: EmailService) {}

  async handle(event: UserCreateEvent) {
    switch (event.name) {
      case UserCreateEvent.name: {
        console.log('UserCreatedEvent!');
        const { email, signupVerifyToken } = event as UserCreateEvent;
        await this.emailService.sendMemberJoinVerification(
          email,
          signupVerifyToken,
        );
        break;
      }
      default:
        break;
    }
  }
}
```

커맨드 핸들러도 application 레이어로 이동한다. 이때 커맨드와 이벤트 소스들을 따로 관리하고 싶다면 디렉터리를 따로 만드는 것도 좋다.

```
├── application
│   ├── command
│   │   ├── create-user.command.ts
│   │   ├── create-user.handler.ts
│   │   ├── login.command.ts
│   │   ├── login.handler.ts
│   │   ├── verify-email.command.ts
│   │   └── verify-email.handler.ts
│   ├── event
│   │   └── user-event.handler.ts
│   └── query
│       ├── get-user-info.handler.ts
│       └── get-user-info.query.ts
```

## 3.3 인터페이스 레이어

UsersController와 관계된 소스코드와 userInfo, DTO 관련 클래스들을 모두 interface 디렉터리로 이동한다.

```
├── interface
│   ├── UserInfo.ts
│   ├── dto
│   │   ├── create-user.dto.ts
│   │   ├── user-login.dto.ts
│   │   └── verify-email.dto.ts
│   └── users.controller.ts
```

## 3.4 인프라 레이어

유저 모듈에서 쓰는 외부 컴포넌트가 포함되도록 한다. 데이터베이스와 이메일 관련 로직이 대상이다. 먼저 엔티티 클래스를 infra/db/entity로 이동한다. UserEntity는 application 레이어에 있는 핸들러가 사용하고 있으므로 DIP를 적용해서 의존관계를 역전한다. 먼저 데이터베이스에 유저 정보를 다루는 인터페이스인 IUserRepository를 선언한다. 이는 domain 레이어에 작성한다.

- domain/repository/iuser.repository.ts

```javascript
import { User } from "../user";

export interface IUserRepository {
  findByEmail: (email: string) => Promise<User | null>;
  save: (
    name: string,
    email: string,
    password: string,
    signupVerifyToken: string
  ) => Promise<void>;
}
```

IUserRepository의 구현체인 UserRepository는 infra 레이어에서 구현한다.

- infra/db/repository/user.repository.ts

```javascript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { IUserRepository } from 'src/users/domain/repository/iuser.repository';
import { Connection, Repository } from 'typeorm';
import { UserEntity } from '../entity/user.entity';
import { UserFactory } from 'src/users/domain/user.factory';
import { User } from 'src/users/domain/user';

@Injectable()
export class UserRepository implements IUserRepository {
  constructor(
    private connection: Connection,
    @InjectRepository(UserEntity)
    private userRepository: Repository<UserEntity>,
    private userFactory: UserFactory,
  ) {}

  async findByEmail(email: string): Promise<User | null> {
    const userEntity = await this.userRepository.findOne({
      where: { email },
    });

    if (!userEntity) {
      return null;
    }

    const { id, name, signupVerifyToken, password } = userEntity;

    // create에는 UserCreateEvent 가 발생하기 때문에 이벤트가 제외된 reconstitute 를 생성하여 사용한다.
    return this.userFactory.reconstitute(
      id,
      name,
      email,
      signupVerifyToken,
      password,
    );
  }

  async save(
    id: string,
    name: string,
    email: string,
    password: string,
    signupVerifyToken: string,
  ): Promise<void> {
    // CreateUserHandler에 있던 로직을 이관한다.
    await this.connection.transaction(async (manager) => {
      const user = new UserEntity();
      user.id = id;
      user.name = name;
      user.email = email;
      user.password = password;
      user.signupVerifyToken = signupVerifyToken;

      await manager.save(user);
    });
  }
}
```

이제 다른 레이어에서 IUserRepository를 이용해 데이터를 다룰 수 있다. CreateUserHandler를 수정하자.

- application/event/create-user.handler.ts

```javascript
import {
  Inject,
  Injectable,
  UnprocessableEntityException,
} from '@nestjs/common';
import { CommandHandler, ICommandHandler } from '@nestjs/cqrs';
import { CreateUserCommand } from '../command/create-user.command';
import * as uuid from 'uuid';
import { IUserRepository } from 'src/users/domain/repository/iuser.repository';
import { UserFactory } from 'src/users/domain/user.factory';

@Injectable()
@CommandHandler(CreateUserCommand)
export class CreateUserHandler implements ICommandHandler<CreateUserCommand> {
  constructor(
  	// UserRepository 토큰을 이용하여 구현된 클래스를 주입받는다.
    @Inject('UserRepository') private userRepository: IUserRepository,
    private userFactory: UserFactory,
  ) {}

  async execute(command: CreateUserCommand) {
    const { name, email, password } = command;
    // userRepository로 user를 찾는다.
    const userExist = await this.userRepository.findByEmail(email);

    if (userExist) {
      throw new UnprocessableEntityException(
        '해당 이메일로는 가입할 수 없습니다.',
      );
    }

    const id = uuid.v1();
    const signupVerifyToken = uuid.v1();

    await this.userRepository.save(
      id,
      name,
      email,
      password,
      signupVerifyToken,
    );

    this.userFactory.create(id, name, email, signupVerifyToken, password);
  }
}
```

- users.module.ts

```javascript
...
import { UserRepository } from './infra/db/repository/user.repository';

@Module({
  ...
  providers: [
    ...
    // 커스텀 프로바이더로 주입한다.
    { provide: 'UserRepository', useClass: UserRepository },
  ],
})
export class UsersModule {}
```

## 3.5 이메일 모듈

이메일 모듈과 유저 모듈의 결합을 인터페이스로 느슨하게 변경한다. 이메일 모듈은 외부 시스템이기 때문에 infra 에 구현체를 위치시킨다. 또 이 구현체를 사용하는 곳은 UserEventsHandler인데 application 레이어에 존재한다. 따라서 application 레이어에 IEmailService를 정의한다.

- application/adapter/iemail.service.ts

```javascript
export interface IEmailService {
  sendMemberJoinVerification: (
    email: string,
    signupVerifyToken: string
  ) => Promise<void>;
}
```

infra 레이어에 인터페이스의 구현체를 작성한다.

- infra/adapter/email.service.ts

```javascript
import { Injectable } from '@nestjs/common';
import { IEmailService } from 'src/users/application/adapter/iemail.service';
import { EmailService as ExternalEmailService } from 'src/email/email.service';

@Injectable()
export class EmailService implements IEmailService {
  constructor(private emailService: ExternalEmailService) {}

  async sendMemberJoinVerification(
    email: string,
    signupVerifyToken: string,
  ): Promise<void> {
    this.emailService.sendMemberJoinVerification(email, signupVerifyToken);
  }
}
```

이제 UserEventHandler에 IEmailService를 주입받아 사용하도록 변경한다.

- application/event/user-event.handler.ts

```javascript
import { EventsHandler, IEventHandler } from '@nestjs/cqrs';
import { UserCreateEvent } from '../../domain/user-create.event';
import { Inject } from '@nestjs/common';
import { IEmailService } from '../adapter/iemail.service';

@EventsHandler(UserCreateEvent)
export class UserEventsHandler implements IEventHandler<UserCreateEvent> {
  constructor(@Inject('EmailService') private emailService: IEmailService) {}

  async handle(event: UserCreateEvent) {
    switch (event.name) {
      case UserCreateEvent.name: {
        console.log('UserCreatedEvent!');
        const { email, signupVerifyToken } = event as UserCreateEvent;
        await this.emailService.sendMemberJoinVerification(
          email,
          signupVerifyToken,
        );
        break;
      }
      default:
        break;
    }
  }
}
```

- users.module.ts

```javascript
...
import { UserRepository } from './infra/db/repository/user.repository';

@Module({
  ...
  providers: [
    ...
    // 커스텀 프로바이더로 주입한다.
    { provide: 'UserRepository', useClass: UserRepository },
    { provide: 'EmailService', useClass: EmailService },
  ],
})
export class UsersModule {}
```

나머지 핸들러도 동일하게 수정한다.

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
