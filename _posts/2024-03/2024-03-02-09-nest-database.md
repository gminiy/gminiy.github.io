---
layout: post
title: "[NestJs] 영속화 - 데이터베이스 다루기"
date: 2024-03-02
categories: dev
tags: nestjs
---

- [1. MySQL 설정](#1-mysql-설정)
- [2. TypeORM으로 데이터베이스 연결](#2-typeorm으로-데이터베이스-연결)
- [3. 회원 가입 요청한 유저의 정보 저장](#3-회원-가입-요청한-유저의-정보-저장)
- [4. 트랜잭션 적용](#4-트랜잭션-적용)
  - [4.1 QueryRunner 사용](#41-queryrunner-사용)
  - [4.2 transaction 함수 이용](#42-transaction-함수-이용)
- [5. 마이그레이션](#5-마이그레이션)

---

이번 포스트에서는 MySQL과 TypeORM을 사용하여 데이터를 다루어보도록 한다.

# 1. MySQL 설정

MySQL 설치 및 실행은 아래 블로그를 확인하기 바란다.
https://velog.io/@yeawonbong/MySQL

# 2. TypeORM으로 데이터베이스 연결

typeorm 라이브러리를 설치한다.

```shell
$ npm i typeorm @nestjs/typeorm mysql2
```

TypeOrmModule을 이용하여 DB에 연결한다.

- app.module.ts

```javascript
import { Module } from "@nestjs/common";
import { UsersModule } from "./users/users.module";
import { ConfigModule } from "@nestjs/config";
import emailConfig from "./config/emailConfig";
import { validationSchema } from "./config/validationSchema";
import { TypeOrmModule } from "@nestjs/typeorm";

@Module({
  imports: [
    UsersModule,
    ConfigModule.forRoot({
      envFilePath: [`${__dirname}/config/env/.${process.env.NODE_ENV}.env`],
      load: [emailConfig],
      isGlobal: true,
      validationSchema,
    }),
    // TypeOrmModule을 동적 모듈로 가지고 온다.
    TypeOrmModule.forRoot({
      type: "mysql", // 데어터베이스 타입
      host: "localhost",
      port: 3306,
      username: "root",
      password: "test",
      database: "test",
      entities: [__dirname + "/**/*.entity{.ts,.js}"], // 소스 코드 내에서 TypeORM이 구동될 때 인식하도록 할 엔티티 클래스 경로
      synchronize: true, // 서비스 구동 시 소스 코드 기반으로 데이터베이스 스키마를 동기화할지 여부. 로컬에서만 true로 설정.
    }),
  ],
  controllers: [],
  providers: [],
})
export class AppModule {}
```

forRoot에 전달되는 TypeOrmModuleOptions 시그니처

```javascript
export type TypeOrmModuleOptions = {
  /**
   * Number of times to retry connecting
   * Default: 10
   */
  retryAttempts?: number,
  /**
   * Delay between connection retry attempts (ms)
   * Default: 3000
   */
  retryDelay?: number,
  /**
   * Function that determines whether the module should
   * attempt to connect upon failure.
   *
   * @param err error that was thrown
   * @returns whether to retry connection or not
   */
  toRetry?: (err: any) => boolean,
  /**
   * If `true`, entities will be loaded automatically.
   */
  autoLoadEntities?: boolean,
  /**
   * If `true`, connection will not be closed on application shutdown.
   * @deprecated
   */
  keepConnectionAlive?: boolean,
  /**
   * If `true`, will show verbose error messages on each connection retry.
   */
  verboseRetryLog?: boolean,
  /**
   * If `true` database initialization will not be performed during module initialization.
   * This means that database connection will not be established and migrations will not run.
   * Database initialization will have to be performed manually using `DataSource.initialize`
   * and it will have to implement own retry mechanism (if necessary).
   */
  manualInitialization?: boolean,
} & Partial<DataSourceOptions>;
```

- retryAttempts : 연결 시 재시도 횟수, 기본은 10
- retryDelay : 재시도 간 지연시간, 기본 3000 ms
- toRetry : 에러 시 연결을 시도할 지 판단하는 함수. 콜백으로 받은 인수 err 을 이용하여 연결 여부를 판단하는 함수를 구현
- autoLoadEntities : entity 를 자동으로 로드할 지 여부
- keepConnectionAlive: 애플리케이션 종료 후 연결을 유지할 지 여부
- verboseRetryLog : 연결 재시도 시 verbose 레벨로 에러 메시지를 보여줄 지 여부

TypeOrmModuleOptions는 위의 타입과 DataSourceOptions 타입을 교차한 타입. 이 옵션 외의 세부 옵션은 MysqlConnectionOptions를 살펴보아야 한다.

TypeOrmModuleOptions 객체를 환경 변수에서 값을 읽어도록 변경.

- /config/env/.development.env

```
...
DATABASE_HOST=localhost
DATABASE_USERNAME=root
DATABASE_PASSWORD=test
DATABASE_SYNCHRONIZE=true
```

- app.module.ts

```javascript
...
@Module({
  imports: [
    ...
    // TypeOrmModule을 동적 모듈로 가지고 온다.
    TypeOrmModule.forRoot({
      type: 'mysql', // 데어터베이스 타입
      host: process.env.DATABASE_HOST,
      port: 3306,
      username: process.env.DATABASE_USERNAME,
      password: process.env.DATABASE_PASSWORD,
      database: 'test',
      entities: [__dirname + '/**/*.entity{.ts,.js}'], // 소스 코드 내에서 TypeORM이 구동될 때 인식하도록 할 엔티티 클래스 경로
      synchronize: process.env.DATABASE_SYNCHRONIZE === 'true', // 서비스 구동 시 소스 코드 기반으로 데이터베이스 스키마를 동기화할지 여부. 로컬에서만 true로 설정.
    }),
  ],
...
```

# 3. 회원 가입 요청한 유저의 정보 저장

유저 엔티티 정의

- src/users/entity/user.entity.ts

```javascript
import { Column, Entity, PrimaryColumn } from "typeorm";

@Entity("User")
export class UserEntity {
  @PrimaryColumn()
  id: string;

  @Column({ length: 30 })
  name: string;

  @Column({ length: 60 })
  email: string;

  @Column({ length: 30 })
  password: string;

  @Column({ length: 60 })
  signupVerifyToken: string;
}
```

데이터베이스 사용하도록 함수 구현

- users.module.ts

```javascript
import { Module } from "@nestjs/common";
import { UsersController } from "./users.controller";
import { UsersService } from "./users.service";
import { EmailModule } from "src/email/email.module";
import { TypeOrmModule } from "@nestjs/typeorm";
import { UserEntity } from "./entity/user.entity";

@Module({
  imports: [EmailModule, TypeOrmModule.forFeature([UserEntity])], // UserEntity Repository 등록
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

- users.service.ts

```javascript
...
@Injectable()
export class UsersService {
  constructor(
    private emailService: EmailService,
    @InjectRepository(UserEntity)
    private userRepository: Repository<UserEntity>, // inject repository
  ) {}
...
```

- users.service.ts

```javascript
...
  async createUser(
    name: string,
    email: string,
    password: string,
  ): Promise<void> {
    const userExist = await this.checkUserExists(email);

    if (userExist) {
      throw new UnprocessableEntityException(
        '해당 이메일로는 가입할 수 없습니다.',
      );
    }

    const signupVerifyToken = uuid.v1();

    await this.saveUser(name, email, password, signupVerifyToken);
    await this.sendMemberJoinEmail(email, signupVerifyToken);
  }
...
  private async checkUserExists(email: string) {
    const user = await this.userRepository.findOne({
      where: { email },
    });

    return user != undefined; // TODO: DB 연동 후 구현
  }
...
  private async saveUser(
    name: string,
    email: string,
    password: string,
    signupVerifyToken: string,
  ) {
    const user = new UserEntity();
    user.id = uuid.v1();
    user.name = name;
    user.email = email;
    user.password = password;
    user.signupVerifyToken = signupVerifyToken;

    await this.userRepository.save(user);
  }
...
```

# 4. 트랜잭션 적용

트랜잭션은 요청을 처리하는 과정에서 데이터베이스에 변경이 일어나는 요청을 분리해서 에러 발생시 이전 상태로 되돌리게 하기 위해 데이터베이스에서 제공하는 기능이다.
TypeORM에서 트랜잭션은 2가지 방법으로 사용할 수 있다.

- QueryRunner를 이용해서 DB 커넥션 상태 생성 및 관리
- transaction 함수 사용

## 4.1 QueryRunner 사용

- user.service.ts

```javascript
...
  private async saveUserUsingQueryRunner(
    name: string,
    email: string,
    password: string,
    signupVerifyToken: string,
  ) {
    const queryRunner = this.dataSource.createQueryRunner(); // QueryRunner 생성

    await queryRunner.connect(); // 연결
    await queryRunner.startTransaction(); // 트랜잭션 시작

    try {
      const user = new UserEntity();
      user.id = uuid.v1();
      user.name = name;
      user.email = email;
      user.password = password;
      user.signupVerifyToken = signupVerifyToken;

      await queryRunner.manager.save(user); // 저장

      await queryRunner.commitTransaction(); // 커밋
    } catch (e) {
      await queryRunner.rollbackTransaction(); // 에러 발생시 롤백
    } finally {
      await queryRunner.release(); // 생성한 QueryRunner는 해제한다.
    }
  }
...
```

## 4.2 transaction 함수 이용

- user.service.ts

```javascript
...
  private async saveUserUsingTransaction(
    name: string,
    email: string,
    password: string,
    signupVerifyToken: string,
  ) {
    await this.dataSource.transaction(async (manager) => {
      const user = new UserEntity();
      user.id = uuid.v1();
      user.name = name;
      user.email = email;
      user.password = password;
      user.signupVerifyToken = signupVerifyToken;

      await manager.save(user);
    });
  }
...
```

# 5. 마이그레이션

데이터베이스 스키마 변경시 마이그레이션을 한다. TypeORM에서 제공하는 마이그레이션을 사용하면 SQL 문을 직접 작성하지 않아도 되고 롤백 작업도 쉽게 수행할 수 있다.

먼저 ts-node 패키지를 글로벌 환경으로 설치해서 typeorm cli를 실행할 수 있게 한다.

```shell
$ npm i -g ts-node
```

pacakage.json 에 typeorm 실행 스크립트를 작성한다.

```json
"scripts": {
  ...
  "typeorm:create": "ts-node -r ts-node/register ./node_modules/typeorm/cli.js",
  "typeorm:generate": "ts-node -r ts-node/register ./node_modules/typeorm/cli.js -d ormconfig.ts"
  ...
```

root에 ormconfig.ts 파일을 생성한다.

- ormconfig.ts

```javascript
import { DataSource } from "typeorm";

export const AppDataSource = new DataSource({
  type: "mysql",
  host: "localhost",
  port: 13306,
  username: "root",
  password: "password",
  database: "test",
  entities: [__dirname + "/**/*.entity{.ts,.js}"],
  synchronize: false,
  migrations: [__dirname + "/**/migrations/**/*{.ts,.js}"],
  migrationsTableName: "migrations",
});
```

- tsconfig.json

```json
{
  ...
  "include": ["src/**/*"]
  ...
}
```

- app.module.ts

```javascript
...
    TypeOrmModule.forRoot({
      type: 'mysql', // 데어터베이스 타입
      host: process.env.DATABASE_HOST,
      port: 3306,
      username: process.env.DATABASE_USERNAME,
      password: process.env.DATABASE_PASSWORD,
      database: 'test',
      entities: [__dirname + '/**/*.entity{.ts,.js}'], // 소스 코드 내에서 TypeORM이 구동될 때 인식하도록 할 엔티티 클래스 경로
      // synchronize: process.env.DATABASE_SYNCHRONIZE === 'true', // 서비스 구동 시 소스 코드 기반으로 데이터베이스 스키마를 동기화할지 여부. 로컬에서만 true로 설정.
      synchronize: false, // 마이그레이션 테스트를 위해 변경
      migrationsRun: false, // 서버 구동 시 자동으로 마이그레이션 하지 않도록 설정. cli로만 직접 입력한다.
      migrations: [__dirname + '/**/migrations/*.js'], // 마이그레이션 스크립트 경로
      migrationsTableName: 'migrations', // 마이그레이션 이력 기록되는 테이블 이름
    }),
...
```

User 테이블 삭제한 후 서버를 재시동한다.

- migration:create: 수행할 마이그레이션 내용이 비어 있는 파일 생성
- migration:generate: 현재 소스 코드와 migrations 테이블에 기록된 이력을 기반으로 마이그레이션 파일 자동 생성

```shell
$ npm run typeorm migration:create src/migrations/CreateUserTable
```

아래와 같은 파일이 생성된다.

```javascript
import { MigrationInterface, QueryRunner } from "typeorm";

export class CreateUserTable1709375389995 implements MigrationInterface {
	// migration:run 수행 시 실행
    public async up(queryRunner: QueryRunner): Promise<void> {
    }
	// migration:revert 수행 시 실행
    public async down(queryRunner: QueryRunner): Promise<void> {
    }

}
```

migration 코드를 cli로 생성해보자.

```shell
$ npm run typeorm:generate migration:generate src/migrations/CreateUserTable
```

아래와 같은 파일이 생성된다.

```javascript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class CreateUserTable1709376266709 implements MigrationInterface {
  name = 'CreateUserTable1709376266709';

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(
      `CREATE TABLE \`User\` (\`id\` varchar(255) NOT NULL, \`name\` varchar(30) NOT NULL, \`email\` varchar(60) NOT NULL, \`password\` varchar(30) NOT NULL, \`signupVerifyToken\` varchar(60) NOT NULL, PRIMARY KEY (\`id\`)) ENGINE=InnoDB`,
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE \`User\``);
  }
}
```

마이그레이션을 실행해보자.

```shell
$ npm run typeorm:generate migration:run
```

마이그레이션을 되돌려보자.

```shell
$ npm run typeorm:generate migration:revert
```

---

_본 포스트는 한용재 저자의 **NestJS로 배우는 백엔드 프로그래밍**을 기반으로 스터디하며 정리한 내용들입니다._

- [NestJS로 배우는 백엔드 프로그래밍](http://www.yes24.com/Product/Goods/115850682)
