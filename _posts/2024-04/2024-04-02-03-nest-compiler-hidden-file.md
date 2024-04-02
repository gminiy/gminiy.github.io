---
layout: post
title: "[NestJs] compilerOption, assets 숨김 파일 처리"
date: 2024-04-02
categories: dev
tags: nest
---

습관처럼 파일 이름을 작성하다가 컴파일 이슈가 발생했다.

## 문제

env 파일을 사용하기 위해 compilerOptions에 assets로 파일을 등록했지만 컴파일에서 제외되었다.

## 원인

env 파일 이름을 ".local.env"와 같이 앞에 . 을 붙여서 숨김 파일로 설정했다. 와일드카드(\*) 사용시 숨김 파일빌드 프로세스에서 자동으로 무시된다.

## 해결

두 가지 방법이 있다.

1. "."을 제거
2. 파일 이름을 컴파일러에 명시

```json
{
  "$schema": "https://json.schemastore.org/nest-cli",
  "collection": "@nestjs/schematics",
  "sourceRoot": "src",
  "compilerOptions": {
    "assets": [
      {
        "include": "./config/env/.local.env",
        "outDir": "./dist"
      }
    ],
    "deleteOutDir": true
  }
}
```

굳이 숨김 파일을 설정할 이유가 없고 와일드카드를 쓰고 싶어서 파일 이름을 변경했다.
