---
title: Nest.jsのORMにPrismaを導入してみる
tags: NestJS TypeScript prisma チュートリアル JavaScript
author: kikikikimorimori
slide: false
---
### はじめに
Nest.jsでアプリを作る際に、`typeORM`ではなく`Prisma`を使いたかったので、その導入方法についてと簡単な使い方

### Prismaとは
Prismaについてテキトーに公式ドキュメントから引用しておきます。

>Prisma is an open-source ORM for Node.js and TypeScript. It is used as an alternative to writing plain SQL, or using another database access tool such as SQL query builders (like knex.js) or ORMs (like TypeORM and Sequelize). Prisma currently supports PostgreSQL, MySQL, SQL Server, SQLite, MongoDB and CockroachDB (Preview).

>PrismaはNode.jsとTypeScriptのためのオープンソースのORMです。SQLクエリビルダ（knex.jsなど）やORM（TypeORMやSequelizeなど）などのデータベースアクセスツールの代替として使用されます。Prismaは現在、PostgreSQL、MySQL、SQL Server、SQLite、MongoDB、CockroachDB (Preview)をサポートしています。

> While Prisma can be used with plain JavaScript, it embraces TypeScript and provides a level to type-safety that goes beyond the guarantees other ORMs in the TypeScript ecosystem.

>PrismaはプレーンなJavaScriptで使用できますが、TypeScriptを採用し、TypeScriptエコシステムの他のORMの保証を超えるレベルの型安全性を提供します。

typeORMとの比較についてはこちら↓

https://www.prisma.io/docs/concepts/more/comparisons/prisma-and-typeorm

### Nest.jsにPrismaを導入する

まずはnestのプロジェクトのディレクトリに移動して、`Primsa`をインストールします。

```
$ cd nest-app
$ npm install prisma --save-dev
```

下記のように、npxをプレフィックスとしてローカルで`Prisma CLI`を起動し、
initコマンドを使用して、`Prisma`の初期設定を作成します。

```
$ npx prisma
$ npx prisma init
```
initコマンドを実行するとprismaディレクトリと、その配下に`schema.prisma`というファイルが作成されます。

```:prisma/schema.prisma
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

データベース接続は、`schema.prisma`ファイルの`datasource`ブロックで設定します。

デフォルトでは、`sqlite`に設定されています。

今回は`mysql`を使います。

```:prisma/schema.prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

```:.env
DATABASE_URL="mysql://USER:PASSWORD@HOST:PORT/DATABASE"
```
※USERやPASSWORDなど大文字の部分は自分のものに変更してください

### `schema.prisma`にUserモデルを作成

続いて`schema.prisma`にモデルを定義します。

今回は試しにUserモデルを作成します。

```:prisma/schema.prisma
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique 
　　　　name  String?
}
```

モデルを定義したら下記コマンドでマイグレートします。

```
$ npx prisma migrate dev --name init
```
上記のコマンドを実行すると、下記のように`migrations`ディレクトリとその配下にマイグレーションファイルが作成されます。

```
prisma
├── migrations
│   └── 20220602083421_init
│       └── migration.sql
└── schema.prisma
```

```sql:prisma/migrations/20220602083421_init/migration.sql
-- CreateTable
CREATE TABLE `User` (
    `id` INTEGER NOT NULL AUTO_INCREMENT,
    `email` VARCHAR(191) NOT NULL,
    `name` VARCHAR(191) NULL,

    UNIQUE INDEX `User_email_key`(`email`),
    PRIMARY KEY (`id`)
) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### `Prisma　Client`のインストール

続いて、下記コマンドで`Prisma　Client`をインストールします。

`Prisma Client`は、Prismaのモデル定義から生成されるタイプセーフのデータベースクライアントです。

```
$ npm install @prisma/client
```

インストール時に、Prismaは自動的に`prisma generate`コマンドを呼び出します。
なので、次回以降Prismaモデルを変更するたびにこのコマンドを実行し、生成されたPrismaクライアントを更新する必要があります。

### PrismaServiceの作成

続いて、`PrismaClient`のインスタンス化とデータベースへの接続を行う`PrismaService`を作成します。

srcディレクトリ内に`prisma.service.ts`というファイルを新規に作成し、下記のコードを追加します。

```ts:src/prisma.service.ts
import { INestApplication, Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }

  async enableShutdownHooks(app: INestApplication) {
    this.$on('beforeExit', async () => {
      await app.close();
    });
  }
}
```

### UsersServiceの作成

`Prisma`スキーマから、先程作成したUserモデルのデータベースを呼び出すために使用する`UsersService`を作成します。

srcディレクトリの中にusersディレクトリと`users.service.ts`というファイルを新規に作成し、下記のコードを追加します。

下記コマンドでも作成できます。
```
$ nest g service users
```

```ts:src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { User, Prisma } from '@prisma/client';

@Injectable()
export class UsersService {
  constructor(private prisma: PrismaService) {}

  async user(
    id: number,
  ): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id }
    });
  }

  async users(): Promise<User[]> {
    return this.prisma.user.findMany();
  }

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }
}

```

### ルーティングとコントローラを追加

続いて、先程作成したサービスを使用して、ルーティングとコントローラを定義します。

usersディレクトリの中に`users.controller.ts`というファイルを新規に作成し、下記のコードを追加します。

下記コマンドでも作成できます。
```
$ nest g controller users
```

```ts:src/users/users.controller.ts
import {
  Controller,
  Get,
  Param,
  Post,
  Body,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from '@prisma/client';

@Controller('users')
export class UsersController {
  constructor(
    private readonly usersService: UsersService,
  ) {}

  @Get(':id')
  async findUserById(
  @Param('id') id: string,
  ): Promise<User> {
    return this.usersService.user(Number(id));
  }

  @Get()
  async users(): Promise<User[]> {
    return this.usersService.users();
  }

  @Post()
  async createUser(
    @Body() userData: { name?: string; email: string },
  ): Promise<User> {
    return this.usersService.createUser(userData);
  }
}
```

### モジュールを追加

最後に`UsersModule`を作成し、`AppModule`に追加します。

users配下に`users.module.ts`というファイルを作成し、下記コードを追加します。

下記コマンドでも作成できます。
```
$ nest g module users
```

```ts:src/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { PrismaService } from '../prisma.service';

@Module({
  controllers: [UsersController,],
  providers: [UsersService, PrismaService]
})
export class UserModule {}
```
```ts:src/app.module.ts
import { Module } from '@nestjs/common';
import { UsersModule } from './users/users.module';

@Module({
  imports: [UsersModule],
})
export class AppModule {}
```

これでNest.jsにPrismaを導入し、アプリケーションからDBを操作できるようになりました

curlコマンドなどで試してみてください！

```
$ curl -H "content-type: application/json" -X POST -d'{"name":"田中太郎", "email":"tanaka@sample.com"}' http://localhost:3000/users
=> {"id":1,"name":"田中太郎","email":"tanaka@sample.com"}

$ curl -X GET http://localhost:3000/users/1
=> {"id":1,"name":"田中太郎","email":"tanaka@sample.com"}

$ curl -X GET http://localhost:3000/users
=> [{"id":1,"name":"田中太郎","email":"tanaka@sample.com"}]
```


### 参考
https://docs.nestjs.com/recipes/prisma#prisma


