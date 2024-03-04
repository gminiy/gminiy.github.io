---
layout: post
title: "[TypeScript] 타입스크립트 설정"
date: 2024-03-02
categories: dev
tags: typescript
---

- tsconfig.json을 통해 타입스크립트 옵션을 설정해야 동료들과 다른 도구들이 알 수 있다.

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2021",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "skipLibCheck": true,
    "strictNullChecks": false,
    "noImplicitAny": false,
    "strictBindCallApply": false,
    "forceConsistentCasingInFileNames": false,
    "noFallthroughCasesInSwitch": false
  },
  "include": ["src/**/*"]
}
```

- **noImplicitAny**는 변수들이 미리 정의된 타입을 가져야 하는지 여부를 제어한다. **되도록이면 true**로 설정해서 'implicit any'를 제거한다.
- **strictNullChecks**는 null과 undefined가 모든 타입에서 허용되는 지 확인한다.
  - const x: number = null 은 strictNullChecks가 true면 오류가 발생한다.
    - const x: number | null = null 의도를 명시적으로 드러냄으로써 오류를 해결한다.
    - null과 undefined 관련된 오류를 잡아 내는 데 도움이 되지만 코드 작성이 어렵다. 익숙한 사용자는 true를 사용하는 것이 좋지만 false로 설정해도 괜찮다.
    - strictNullChecks 는 noImplictAny가 true이어야 한다.
- 타입스크립트에서 엄격한 체크를 하고 싶다면 strict 설정을 고려해보아야 한다.

---

_본 포스트는 댄 밴더캄 저자의 **이펙티브 타입스크립트**를 기반으로 스터디하며 정리한 내용들입니다._

- [이펙티브 타입스크립트](https://product.kyobobook.co.kr/detail/S000001033114)
