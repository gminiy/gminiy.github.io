---
layout: post
title: "[NestJs] NestJs 장점 및 설치"
date: 2024-02-23
categories: dev
tags: nestjs
---

> - [1. NestJS 장점](#1-nestjs-장점)
> - [2. NestJS 설치](#2-nestjs-설치)
> - [3. NestJS 보일러플레이트](#3-nestjs-보일러플레이트)
> - [4. 서버 실행](#4-서버-실행)

---

# 1. NestJS 장점

NestJS는 **Express 또는 Fastify 프레임워크**를 래핑하여 동작한다. Fastify는 Express보다 벤치마크 결과 2배 빠른 속도를 자랑한다. NestJS는 Fastify의 빠른 속도와 Express의 높은 호환성을 갖고자 한다.

Node.js 의 과도한 유연함으로 인해 발생하는 문제점을 해결하기 위해 **데이터베이스. 객체 관계 매핑(ORM), 설정, 유효성 검사** 등의 기능을 기본으로 제공한다. 그러면서도 필요한 라이브러리를 쉽게 설치하여 사용하는 Node.JS 장점은 그대로 가지고 있다.

**모듈/컴포넌트 기반**으로 프로그램을 작성함으로써 재사용성을 높인다. 또 **제어 반전, 의존성 주입, 관점 지향 프로그래밍** 같은 객체 지향 개념을 도입했다. 기본으로 **타입스크립트**를 채택하고 있다.

---

# 2. NestJS 설치

Mac 버전으로 설명한다.

- NodeJS 먼저 설치한다.

[https://nodejs.org/en](https://nodejs.org/en)

- @nestjs/cli 를 npm으로 설치한다.

```shell
$ npm i -g @nestjs/cli
```

- 프로젝트를 생성한다.

```shell
$ nest new project-name
```

# 3. NestJS 보일러플레이트

```shell
├── README.md
├── nest-cli.json
├── package-lock.json
├── package.json
├── src
│   ├── app.controller.spec.ts
│   ├── app.controller.ts
│   ├── app.module.ts
│   ├── app.service.ts
│   └── main.ts
├── test
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
├── tsconfig.build.json
└── tsconfig.json
```

# 4. 서버 실행

```shell
$ npm i
$ npm run start:dev

[5:51:40 PM] Starting compilation in watch mode...

[5:51:41 PM] Found 0 errors. Watching for file changes.

[Nest] 32419  - 02/25/2023, 5:51:42 PM     LOG [NestFactory] Starting Nest application...
[Nest] 32419  - 02/25/2023, 5:51:42 PM     LOG [InstanceLoader] AppModule dependencies initialized +10ms
[Nest] 32419  - 02/25/2023, 5:51:42 PM     LOG [RoutesResolver] AppController {/}: +4ms
[Nest] 32419  - 02/25/2023, 5:51:42 PM     LOG [RouterExplorer] Mapped {/, GET} route +2ms
[Nest] 32419  - 02/25/2023, 5:51:42 PM     LOG [NestApplication] Nest application successfully started +2ms
```

localhost에서 구동된 것을 확인할 수 있다. 포트는 기본으로 main.ts 에서 3000번으로 설정되어 있다.

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
