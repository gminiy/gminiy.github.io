---
layout: post
title: "[NestJs] ConfigModule 비동기로 사용하기"
date: 2024-03-22
categories: dev
tags: nestjs
---

## 문제

Nest에서 ConfigModule을 이용해 환경변수를 전역으로 사용하는데 JwtModule 생성시 ConfigModule을 통해 환경변수를 읽어오지 못하는 이슈가 있었다.

- app.module.ts

```typescript
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AuthModule } from "./auth/infrastructure/in/auth.module";
import { ConfigModule } from "@nestjs/config";

@Module({
  imports: [
    ConfigModule.forRoot({
      envFilePath: [`${__dirname}/config/env/.${process.env.NODE_ENV}.env`],
      isGlobal: true,
    }),
    AuthModule,
  ],
  controllers: [AppController],
  providers: [],
})
export class AppModule {}
```

- auth.module.ts

```typescript
import { Module } from "@nestjs/common";
import { AuthController } from "./controller/auth.controller";
import { JwtModule } from "@nestjs/jwt";

@Module({
  imports: [
    TypeOrmModule.forFeature([UserEntity]),
    JwtModule.register({
      secret: process.env.JWT_SECRET, // undefined
      signOptions: {
        expiresIn: process.env.JWT_ACCESS_TOKEN_EXPIRES_IN, // undefind
      },
    }),
  ],
  controllers: [AuthController],
})
export class AuthModule {}
```

AuthModule에서 JwtModule을 import할때 환경 변수들을 ConfigModule을 통해 받기를 기대했지만 undefined로 읽어오게 된다.

## 원인

ConfigModule은 모듈 로딩시 비동기로 수행된다. 즉, 다른 모듈들이 ConfigModule을 사용할때 ConfigModule의 로딩이 완료되었다는 보장을 하지 않는다.

## 해결

Async 메소드를 통해 모듈의 준비 상태를 확인하고 사용할 수 있다. 이번 경우에는 ConfigService를 주입함으로써 ConfigModule이 초기화되고 ConfigService가 준비될때까지 기다린 후 모듈을 생성하게 만들 수 있다.

- auth.module.ts

```typescript
import { Module } from "@nestjs/common";
import { AuthController } from "./controller/auth.controller";
import { JwtModule } from "@nestjs/jwt";
import { ConfigModule, ConfigService } from "@nestjs/config";

@Module({
  imports: [
    TypeOrmModule.forFeature([UserEntity]),
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        secret: configService.get("JWT_SECRET"),
        signOptions: {
          expiresIn: configService.get("JWT_ACCESS_TOKEN_EXPIRES_IN"),
        },
      }),
    }),
  ],
  controllers: [AuthController],
})
export class AuthModule {}
```
