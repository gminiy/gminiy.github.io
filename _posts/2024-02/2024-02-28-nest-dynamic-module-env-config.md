---
layout: post
title: "[NestJs] 동적 모듈을 활용한 환경 변수 구성"
date: 2024-02-28
categories: dev
tags: nestjs
---

- [1. 동적 모듈](#1-동적-모듈)
- [2. dotenv를 이용한 Config 설정](#2-dotenv를-이용한-config-설정)
- [3. Nest에서 제공하는 Config 패키지](#3-nest에서-제공하는-config-패키지)
- [4. 유저 서비스에 환경 변수 구성하기](#4-유저-서비스에-환경-변수-구성하기)
  - [4.1 커스텀 Config 파일 작성](#41-커스텀-config-파일-작성)
  - [4.2 동적 ConfigModule 등록](#42-동적-configmodule-등록)

---

# 1. 동적 모듈

동적 모듈 (Dynamic Module) 은 모듈이 생성될 때 동적으로 변수들이 정해진다.
실행 환경에 따라 환경 변수를 관리하는 모듈인 Config 모듈이 대표적인 예다.

---

# 2. dotenv를 이용한 Config 설정

NodeJs의 dotenv 라이브러리를 통해 환경 변수를 설정한다. dotenv는 .env확장자를 가진 파일을 읽어서 환경 변수를 설정한다.

dotenv를 설치하자.

```shell
$ npm i dotenv
$ npm i -D @types/dotenv
```

dotenv는 루트 디렉터리에 있는 .env확장자 파일을 읽는다. DTABASE_HOST라는 환경 변수를 각 파일에 저자한다.

- .development.env

```javascript
DATABASE_HOST = local;
```

- .stage.env

```javascript
DATABASE_HOST = stage - reader.dextto.com;
```

- .production.env

```javascript
DATABASE_HOST = prod - reader.dextto.com;
```

package.json 을 수정해서 NODE_ENV를 설정되도록 한다.

- package.json

```json
	"prebuild": "rimraf dist",
    "start:dev": "npm run prebuild && NODE_ENV=local nest start --watch",
```

env 파일을 읽어서 사용하도록 변경한다.

- main.ts

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";
import * as dotenv from "dotenv";
import * as path from "path";

dotenv.config({
  path: path.resolve(
    process.env.NODE_ENV === "production"
      ? ".production.env"
      : process.env.NODE_ENV === "stage"
      ? ".stage.env"
      : ".local.env"
  ),
});

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  await app.listen(3000);
}

bootstrap();
```

---

# 3. Nest에서 제공하는 Config 패키지

앞에서 dotenv 패키지를 직접 사용했는데 Nest는 dotenv를 내부적으로 활용하는 @nestjs/config 패키지를 제공한다. 이를 이용해서 ConfigModule을 동적으로 생성할 수 있다.

```shell
$ npm i @nestjs/config
```

이 패키지에는 ConfigModule이 이미 존재한다. 이 모듈을 동적 모듈로 가지고 온다.

- app.module.ts

```javascript
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { ConfigModule } from "@nestjs/config";

@Module({
  imports: [ConfigModule.forRoot()],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}
```

forRoot는 DynamicModule을 리턴하는 정적 메서드로 ConfigModuleOptions를 인수로 받는다.

- forRoot()

```javascript
static forRoot(options?: ConfigModuleOptions): DynamicModule;
```

ConfigModule은 소비 모듈이 원하는 옵션값을 전달하여 원하는 대로 동적으로 ConfigModule을 생성한다.

- ConfigModule

```javascript
export interface ConfigModuleOptions {
  /**
   * If "true", values from the process.env object will be cached in the memory.
   * This improves the overall application performance.
   * See: https://github.com/nodejs/node/issues/3104
   */
  cache?: boolean;
  /**
   * If "true", registers `ConfigModule` as a global module.
   * See: https://docs.nestjs.com/modules#global-modules
   */
  isGlobal?: boolean;
  /**
   * If "true", environment files (`.env`) will be ignored.
   */
  ignoreEnvFile?: boolean;
  /**
   * If "true", predefined environment variables will not be validated.
   */
  ignoreEnvVars?: boolean;
  /**
   * Path to the environment file(s) to be loaded.
   */
  envFilePath?: string | string[];
  /**
   * Environment file encoding.
   */
  encoding?: string;
  /**
   * Custom function to validate environment variables. It takes an object containing environment
   * variables as input and outputs validated environment variables.
   * If exception is thrown in the function it would prevent the application from bootstrapping.
   * Also, environment variables can be edited through this function, changes
   * will be reflected in the process.env object.
   */
  validate?: (config: Record<string, any>) => Record<string, any>;
  /**
   * Environment variables validation schema (Joi).
   */
  validationSchema?: any;
  /**
   * Schema validation options.
   * See: https://joi.dev/api/?v=17.3.0#anyvalidatevalue-options
   */
  validationOptions?: Record<string, any>;
  /**
   * Array of custom configuration files to be loaded.
   * See: https://docs.nestjs.com/techniques/configuration
   */
  load?: Array<ConfigFactory>;
  /**
   * A boolean value indicating the use of expanded variables, or object
   * containing options to pass to dotenv-expand.
   * If .env contains expanded variables, they'll only be parsed if
   * this property is set to true.
   */
  expandVariables?: boolean | DotenvExpandOptions;
}
```

루트 디렉터리에 있는 .env 파일을 환경 변수로 등록한다.

- app.module.ts

```javascript
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";
import { ConfigModule } from "@nestjs/config";

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath:
        process.env.NODE_ENV === "production"
          ? ".production.env"
          : process.env.NODE_ENV === "stage"
          ? ".stage.env"
          : ".local.env",
    }),
  ],
  controllers: [AppController],
  providers: [AppService, ConfigService],
})
export class AppModule {}
```

ConfigModule은 환경 변수 값을 가져오는 프로바이더인 ConfigService가 있다. 이를 원하는 컴포넌트에서 주입해서 사용하면 된다.

```javascript
import { Controller, Get } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Controller()
export class AppController {
  constructor(
          private readonly configService: ConfigService,
  ) {}

  @Get('config')
  getConfig(): string {
    return this.configService.get('DATABASE_HOST');
  }
}
```

---

# 4. 유저 서비스에 환경 변수 구성하기

@nest/config 와 환경 변수 유효성 검사를 위한 joi를 설치한다.

```shell
$ npm i @nestjs/config
$ npm i joi
```

- .development.env

```
EMAIL_SERVICE=Gmail
EMAIL_AUTH_USER=YOUR-GAMIL
EMAIL_AUTH_PASSWORD=YOUR-GMAIL-PASSWORD
EMAIL_BASE_URL=http://localhost:3000
```

---

## 4.1 커스텀 Config 파일 작성

모든 환경 변수가 .env 파일에 선언되어 있지만, 가져다 쓸 대는 DatabaseConfig, EmailConfig와 같이 의미 있는 단위로 묶고 싶다면 ConfigModule을 이용해 구현할수 있다.

src/config에 이메일 관련 환경 변수를 관리하는 emailConfig.ts를 작성한다.
email 토큰으로 ConfigFactory를 등록하는 코드이다.

- emailConfig.ts

```javascript
import { registerAs } from "@nestjs/config";

export default registerAs("email", () => ({
  service: process.env.EMAIL_SERVICE,
  auth: {
    user: process.env.EMAIL_AUTH_USER,
    pass: process.env.EMAIL_AUTH_PASSWORD,
  },
  baseUrl: process.env.EMAIL_BASE_URL,
}));
```

---

## 4.2 동적 ConfigModule 등록

.env 파일을 루트 경로가 아니라 src/config/env 디렉터리에 모아서 관리한다.

- nest-cli.json

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "assets": [
      {
        "include": "./config/env/*.env",
        "outDir": "./dist"
      }
    ],
    "watchAssets": true // watch 모드에서 asset을 탐색하게 한다.
  }
}
```

AppModule에 ConfigModule을 등록한다.
config는 전역모듈로 설정해서 모든 모듈에서 사용할 수 있게 한다. 유효성 검사를 위해 유효성 검사 객체를 작성한다.

- app.module.ts

```javascript
import { Module } from "@nestjs/common";
import { UsersModule } from "./users/users.module";
import { ConfigModule } from "@nestjs/config";
import emailConfig from "./config/emailConfig";
import { validateSchema } from "./config/validateSchema";

@Module({
  imports: [
    UsersModule,
    ConfigModule.forRoot({
      envFilePath: [`${__dirname}/config/env/.${process.env.NODE_ENV}.env`],
      load: [emailConfig],
      isGlobal: true,
      validationSchema,
    }),
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

- config/validateSchema.ts

```javascript
import * as Joi from "joi";

export const validateSchema = Joi.object({
  EMAIL_SERVICE: Joi.string().required(),
  EMAIL_AUTH_USER: Joi.string().required(),
  EMAIL_AUTH_PASSWORD: Joi.string().required(),
  EMAIL_BASE_URL: Joi.string().required(),
});
```

- email.service.ts

```javascript
import Mail from 'nodemailer/lib/mailer';
import * as nodemailer from 'nodemailer';
import { Inject, Injectable } from '@nestjs/common';
import emailConfig from 'src/config/emailConfig';
import { ConfigType } from '@nestjs/config';

...

@Injectable()
export class EmailService {
  private transporter: Mail;

  constructor(
    @Inject(emailConfig.KEY) private config: ConfigType<typeof emailConfig>,
  ) {
    this.transporter = nodemailer.createTransport({
      service: config.service,
      auth: {
        user: config.auth.user,
        pass: config.auth.pass,
      },
    });
  }

  async sendMemberJoinVerification(email: string, signupVerifyToken: string) {
    const baseUrl = this.config.baseUrl;

    ...
}
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
