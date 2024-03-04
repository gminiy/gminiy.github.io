---
layout: post
title: "[NestJs] 파이프와 유효성 검사"
date: 2024-03-02
categories: dev
tags: nestjs
---

- [1. 파이프](#1-파이프)
- [2. 파이프의 내부 구현 이해하기](#2-파이프의-내부-구현-이해하기)
- [3. 유효성 검사 파이프 만들기](#3-유효성-검사-파이프-만들기)
- [4. 유저 서비스에 유효성 검사 적용](#4-유저-서비스에-유효성-검사-적용)
  - [4.1 유저 생성 body 유효성 검사](#41-유저-생성-body-유효성-검사)
  - [4.2 class-transformer 활용](#42-class-transformer-활용)
  - [4.3 커스텀 유효성 검사기](#43-커스텀-유효성-검사기)

---

---

# 1. 파이프

파이프는 요청이 라우터 핸들러로 전달되기 전에 요청 객체를 변환하거나 검사할 수 있게 한다. 미들웨어와 비슷하지만 파이프는 모든 context에서 사용할 수 있다.
@nest/common 패키지에는 여러 내장 파이프가 있다.

- ValidationPipe
- ParseIntPipe
- ParseBoolPipe
- ParseArrayPipe
- ParseUUIDPipe
- DefaultValuePipe

예를 들어 /users/user/:id 엔드포인트에 전달된 id는 문자열 타입이다. 코드에서는 이를 정수로 사용할때 @Param 데커레이터를 통해 실행 context에 바인딩할 수 있다.

```javascript
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  return this.userService.findOne(id);
}
```

** id 에 정수 파싱이 안되는 문자를 전달하면 400 에러가 발생한다. **

클래스를 전달하지 않고 파이프 객체를 직접 생성해서 전달할 수 있다. 예를 들어 에러에서 상태코드를 변경하고자 할때 사용할 수 있다.

```javascript
@Get(':id')
findOne(@Param('id', new ParseIntPipe({ errorHttpStatusCode: HttpStatus.NOT_ACCEPTABLE })) id: number) {
  return this.userService.findOne(id);
}
```

DefaultValuePipe는 인수에 기본값을 설정할 때 사용한다.

```javascript
@Get()
findAll(
  @Query('offset', new DefaultValuePipe(0), ParseIntPipe) offset: number,
  @Query('limit', new DefaultValuePipe(10), ParseIntPipe) limit: number,
) {
    console.log(offset, limit);
    return this.userService.findAll();
}
```

# 2. 파이프의 내부 구현 이해하기

Nest에서 ValidationPipe를 제공하지만 여기서 ValidationPipe를 직접 만들어본다.
커스텀 파이프는 PipeTransform 인터페이스를 상속받은 클래스에 @Injectable 데코레이터를 붙여준다.

- validation.pipe.ts

```javascript
import { PipeTransform, Injectable, ArgumentMetadata } from "@nestjs/common";

@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    console.log(metadata);
    return value;
  }
}
```

- value: 현재 파이프에 전달된 인수
- metadata: 현재 파이프에 전달된 인수의 메타데이터

ArgumentMetadata 시그니처

```javascript
export interface ArgumentMetadata {
    /**
     * Indicates whether argument is a body, query, param, or custom parameter
     */
    readonly type: Paramtype;
    /**
     * Underlying base type (e.g., `String`) of the parameter, based on the type
     * definition in the route handler.
     */
    readonly metatype?: Type<any> | undefined;
    /**
     * String passed as an argument to the decorator.
     * Example: `@Body('userId')` would yield `userId`
     */
    readonly data?: string | undefined;
}
```

- type: 파이프에 전달된 인수가 본문인지 쿼리 매개변수인지, 경로 매개변수인지 커스텀 매개변수인지 나타낸다.
- metadata: 라우트 핸들러에 정의된 인수의 타입을 알려준다.
- data: 데커레이터에 전달된 문자열, 즉, 매개변수의 이름이다.

예를 들어 아래 라우터 핸들러를 구현했다면, transform 함수에 전달되는 인수는 다음과 같은 객체가 된다.

```javascript
@Get(':id')
findOne(@Param('id', ValidationPipe) id: number) {
  return this.userService.findOne(id);
}
```

```
{ metadata: [Function: Number], type: 'param', data: 'id' }
```

# 3. 유효성 검사 파이프 만들기

Nest 공식 문서에는 @UsePipes와 joi 를 이용하여 커스텀 파이프를 바인딩하는 방법을 설명한다. joi는 널리 사용되지만 class-validator와 비교하면 문법이 번거롭다. class-validator를 이용하여 유효성 검사 파이프를 구현해보자.

```shell
$ npm i --save class-validator class-transformer
```

- dto/create-user.dto

```javascript
import { IsEmail, IsString, MaxLength, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @MinLength(1)
  @MaxLength(20)
  readonly name: string;

  @IsEmail()
  email: string;
}
```

위에서 정의한 dto 객체를 받아서 유효성 검사를 하는 파이프(ValidationPipe)를 직접 구현해보자.

```javascript
import {
  ArgumentMetadata,
  BadRequestException,
  Injectable,
  PipeTransform,
} from '@nestjs/common';
import { validate } from 'class-validator';
import { plainToClass } from 'class-transformer';

@Injectable()
export class ValidationPipe implements PipeTransform<any> {
  async transform(value: any, { metatype }: ArgumentMetadata) {
    // metatype 이 지원하는 타입인지 검사
    if (!metatype || !this.toValidate(metatype)) {
      return value;
    }
    // 순수 자바스크립트 객체를 클래스의 객체로 변환, class-validator의 유효성 검사는 타입이 필요하므로 타입 지정 과정을 plainToClass로 수행한다.
    const object = plainToClass(metatype, value);

    const errors = await validate(object);

    if (errors.length > 0) {
      throw new BadRequestException('Validation failed~');
    }

    return value;
  }

  private toValidate(metatype: Function): boolean {
    const types: Function[] = [String, Boolean, Number, Array, Object];
    return !types.includes(metatype);
  }
}
```

ValidationPipe 적용

```javascript
@Post()
create(@Body(ValidationPipe) createUserDto: CreateUserDto) {
  return this.userService.create(createUserDto)
}
```

잘못된 데이터 전달하면 에러가 발생한다.

```jso
{
  "statusCode": 400,
  "message": "Validation failed",
  "error": "Bad Request"
}
```

ValidationPipe를 전역 설정하기 위해서는 bootstrap에서 적용하면 된다.

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "./validation.pipe";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  await app.listen(3000);
}
bootstrap();
```

# 4. 유저 서비스에 유효성 검사 적용

## 4.1 유저 생성 body 유효성 검사

Nest 에서 제공하는 ValidationPipe를 전역으로 적용한다. 이때 class-tranformer를 적용하기 위해 transform 속성을 true로 설정한다.

- main.ts

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import { ValidationPipe } from "@nestjs/common";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe({ transform: true }));
  await app.listen(3000);
}
bootstrap();
```

DTO를 수정한다.

- create-user.dto.ts

```javascript
import {
  IsEmail,
  IsString,
  Matches,
  MaxLength,
  MinLength,
} from 'class-validator';

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  readonly name: string;

  @IsString()
  @IsEmail()
  @MaxLength(60)
  readonly email: string;

  @IsString()
  @Matches(/^[A-Za-z\d!@#$%^&*()]{8,30}$/)
  readonly password: string;
}
```

## 4.2 class-transformer 활용

@Transform 데커레이터는 transformFn를 인수로 받는다. transformFn은 value와 obj 등을 인수로 받아 속성을 변형한 후 리턴하는 함수이다.

```javascript
export declare function Transform(transformFn: (params: TransformFnParams) => any, options?: TransformOptions): PropertyDecorator;


export interface TransformFnParams {
  value: any;
  key: string;
  obj: any;
  type: TransformationType;
  options: ClassTransformOptions;
}
```

name 속성에 @Transform 적용하여 값을 확인해보자.

- create-user.dto.ts

```javascript
import { Transform } from 'class-transformer';
import {
  IsEmail,
  IsString,
  Matches,
  MaxLength,
  MinLength,
} from 'class-validator';

export class CreateUserDto {
  @Transform((params) => {
    console.log(params);
    return params.value;
  })
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  readonly name: string;

  ...

```

```
{
  value: 'ga',
  key: 'name',
  obj: { name: 'ga', email: 'gmini.y@gmail.com', password: 'pasdfqwer' },
  type: 0,
  options: {
    enableCircularCheck: false,
    enableImplicitConversion: false,
    excludeExtraneousValues: false,
    excludePrefixes: undefined,
    exposeDefaultValues: false,
    exposeUnsetFields: true,
    groups: undefined,
    ignoreDecorators: false,
    strategy: undefined,
    targetMaps: undefined,
    version: undefined
  }
}
```

name에 앞뒤 공백을 제거하는 로직을 추가하자.

- create-user.dto.ts

```javascript
...

export class CreateUserDto {
  @Transform((params) => params.value.trim())
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  readonly name: string;

...
```

TransformFnParams의 obj는 CreateUserDto 객체를 갖는다. 이를 통해 password는 name과 동일한 문자열을 포함할 수 없도록 구현할 수 있다.

@ create-user.dto.ts

```javascript
...
export class CreateUserDto {
  ...
  @Transform(({ value, obj }) => {
    if (obj.password.includes(obj.name.trim())) {
      throw new BadRequestException('password can not include string of name');
    }

    return value.trim();
  })
  @IsString()
  @Matches(/^[A-Za-z\d!@#$%^&*()]{8,30}$/)
  readonly password: string;
}
...
```

## 4.3 커스텀 유효성 검사기

직접 필요한 유효성 검사를 수행하는 데커레이터를 만들어서 활용할 수 있다.

- not-in.ts

```javascript
import {
  ValidationOptions,
  registerDecorator,
  ValidationArguments,
} from 'class-validator';

// 데커레이터 인수는 참조하고자 하는 다른 속성의 이름과 ValidationOptions를 받는다.
export function NotIn(property: string, validationOptions?: ValidationOptions) {
  // registerDecorator를 호출하는 함수를 리턴한다.
  // eslint-disable-next-line @typescript-eslint/ban-types
  return (object: Object, propertyName: string) => {
    // registerDecorator 함수는 ValidationDecoratorOptions 객체를 인수로 받는다.
    registerDecorator({
      name: 'NotIn', // 데커레이터 이름
      target: object.constructor, // 객체 생성 시 적용
      propertyName,
      options: validationOptions, // 데커레이터 인수로 받은 옵션을 적용한다.
      constraints: [property], // 속성에 적용되도록 제약을 준다.
      validator: {
        // 유효성 검사 규칙이 기술된다.
        validate(value: any, args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          const relatedValue = (args.object as any)[relatedPropertyName];

          return (
            typeof value === 'string' &&
            typeof relatedValue === 'string' &&
            !relatedValue.includes(value)
          );
        },
      },
    });
  };
}
```

- create-user.dto.ts

```javascript
...
import { NotIn } from './not-in';

export class CreateUserDto {
  @Transform((params) => params.value.trim())
  @NotIn('password', { message: 'password can not include name' })
  @IsString()
  @MinLength(2)
  @MaxLength(20)
  readonly name: string;
  ...
}
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
