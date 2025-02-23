---
title: TypeScript を利用した Lambda 関数で依存性を注入する
date: 2025-02-24
author: "@irihit"
tags:
  - AWS
  - AWS Lambda
  - TypeScript
---

TypeScript で依存性の注入 (Dependency Injection) を行う場合、軽量なライブラリの選択肢としては [Inversify](https://inversify.io/) か [TSyringe](https://github.com/microsoft/tsyringe) を検討することができます。どちらもスター数は十分にあるものの、2025 年 2 月次展で TSyringe は 4 年以上更新が停止しているため、Inversify の利用を検討します。

<!--more-->

## 前提

本記事では次の環境で実装を行うものとして想定します。

- AWS Lambda (Node.js 22)
- TypeScript 5.7
- Inversify 6.2

## Inversify の導入

Inversify は npm などの一般的なパッケージ管理でインストールすることができる。npm を利用する場合のインストール方法は次のとおりです。

```bash
npm install --save inversify reflect-metadata
```

Inversify はデコレーターを利用して依存関係の解決を行うため、reflect-metadata も一緒にインストールする必要がある。また、TypeScript を利用する場合は `tsconfig.json` で次の設定を行う必要があります。

```json
{
  "compilerOptions": {
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true
  }
}
```

それぞれの設定がどのように作用するかについては TypeScript の公式ドキュメントを参照します。

- [Experimental Decorators](https://www.typescriptlang.org/tsconfig/#experimentalDecorators)
- [Emit Decorator Metadata](https://www.typescriptlang.org/tsconfig/#emitDecoratorMetadata)

## Inversify を利用して Lambda 関数のハンドラーを実装する

今回は依存性の注入を行うことを主とするため、簡単なサービスクラスを実装します。

```ts
// my-service.ts
import { inject, injectable } from 'inversify';

@injectable()
export class MyService {
  hello() {
    console.log('Hello');
  }
}
```

次にハンドラーから呼び出す実行クラスを実装します。

```ts
// my-runner.ts
import { inject, injectable } from 'inversify';
import { MyService } from './my-service';

@injectable()
export class MyRunner {
  constructor(
    @inject(MyService)
    private readonly service: MyService,
  ) {}

  run() {
    this.service.hello();
  }
}
```

次にバインドの設定を行う。バインドの定義は `inversify.config.ts` で行います。

```ts
// inversify.config.ts
import { Container } from 'inversify';

const container = new Container();
container.bind(MyService).toSelf();
container.bind(MyRunner).toSelf();
```

最後に Lambda 関数のハンドラーを実装する。ハンドラーでは `reflect-metadata` の import を行います。

```ts
// handler.ts
import 'reflect-metadata';
import { container } from './inversify.config';
import { MyRunner } from './my-runner';

export function handler() {
  const runner = container.get<MyRunner>(MyRunner);
  runner.run();
}
```

## 依存性の注入を行うメリット

簡単な処理を行う Lambda 関数では、処理のすべてを一つのファイルにまとめて定義したり、クラスを利用せず単純な関数のみを利用して実装が可能となるため、こういった実装は不要な場合があります。

ただし、より複雑な実装を行う場合には、依存性の注入を行うメリットが大きくなる場合があります。次にその例を示します。

- RDS Proxy 等を利用して Lambda 関数からデータベースに直接接続する場合には、 [シングルトンスコープ](https://inversify.io/docs/api/binding-syntax/#insingletonscope) を利用して接続を共有することができます。
- サービスクラスなどを共有する場合、テストの容易性が向上します。

必要に応じて利用することでコードの保守が効率的になるため、選択肢の一つとして検討するといいでしょう。

## 参考

本記事に関する正確な情報、および最新の情報は、次のページから確認できます。

- [InversifyJS docs](https://inversify.io/)
