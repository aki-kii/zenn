---
title: 'AWS CDK スナップショットテストのプラクティス集'
emoji: '📷'
type: 'tech'
topics: ['aws', 'cdk', 'iac']
published: true
---

本記事は『AWS CDK Advent Calendar 2025』の 4 日目の記事です。

https://qiita.com/advent-calendar/2025/aws-cdk

## はじめに

AWS CDK には変更差分を確認するスナップショットテストというテスト手法があります。\
取り入れるのは簡単ですが、効果は絶大なのでやっておいて損はありません。

本記事ではスナップショットテストの効率化や、より効果的に実施するためのプラクティスを紹介します！

## ざっくり概要

- スナップショットテストを利用すれば意図しない変更を防げる
- リファクタリングや CDK のバージョンアップ時にもリソースの変更がないか確かめられる
- Jest を Vitest に置き換えることでテストの高速化・開発体験向上が見込める
- アセットのハッシュ値を比較しないことで無駄に変更差分を出さない
- アプリケーションコードのバンドルをスキップすることでテストを効率化できる
- アセットのステージングをスキップすることでテストを効率化できる
- 機能フラグを渡すことで正確なテストを実施できる
- 全ての環境でテストすることで安全性を高める

## スナップショットテストとは？

CDK が合成した CloudFormation テンプレートを以前取得したスナップショットと差分があるかを確認するテスト手法です。\
差分が発生した場合はテストに失敗しますが、意図した差分であればスナップショットを更新することでスナップショットテストに差分を取り込みます。

主なユースケースは次のようなものです。

- 変更した差分が意図したものであるか確認する
- リファクタリングや CDK のバージョンアップなどで差分が発生しないことを確認する
- CloudFormation テンプレートの合成に成功するか確認する

## 基本のスナップショットテスト

基本的なスナップショットテストを実行できる環境を構築します。\
`cdk init` コマンドで　 CDK アプリを初期構築した状態から始めます。

```sh
% npx cdk init -l ts
```

なお、本記事ではスナップショットテストを実施することを目的としており、簡単化のために`bin`ディレクトリにある CDK アプリのエントリーポイントには触れません。

Stack に SQS Queue を追加します。

```ts:lib/sample-stack.ts
import { Duration, Stack, StackProps } from 'aws-cdk-lib';
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class SampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Queue(this, 'Queue', {
      visibilityTimeout: Duration.seconds(300)
    });
  }
}
```

次にテストファイルを作成します。\
基本のスナップショットテストは次の流れで実施されます。

1. CDK アプリをインスタンス化
1. テスト対象の Stack をインスタンス化
1. Stack からテンプレートを合成
1. 前回のスナップショット(`test/snapshot`に保存されている)と比較

```ts:test/sample-stack.test.ts
import { App } from "aws-cdk-lib";
import { SampleStack } from "../lib/cdk-snapshot-sample-stack";
import { Template } from "aws-cdk-lib/assertions";

test('Snapshot test', () => {
    // CDK アプリをインスタンス化
    const app = new App()
    // Stack をインスタンス化
    const stack = new SampleStack(app, 'SampleStack')
    // テンプレートを作成して前回のスナップショット(`test/snapshot`に保存されている)と比較する
    expect(Template.fromStack(stack)).toMatchSnapshot()
});
```

テストを実行すると成功しています。\
前回のスナップショットがないため現在のテンプレートがスナップショットとして残ります。

```sh
% npm run test

> cdk-snapshot-sample@0.1.0 test
> jest

ts-jest[config] (WARN) message TS151002: Using hybrid module kind (Node16/18/Next) is only supported in "isolatedModules: true". Please set "isolatedModules: true" in your tsconfig.json. To disable this message, you can set "diagnostics.ignoreCodes" to include 151002 in your ts-jest config. See more at https://kulshekhar.github.io/ts-jest/docs/getting-started/options/diagnostics
ts-jest[config] (WARN) message TS151002: Using hybrid module kind (Node16/18/Next) is only supported in "isolatedModules: true". Please set "isolatedModules: true" in your tsconfig.json. To disable this message, you can set "diagnostics.ignoreCodes" to include 151002 in your ts-jest config. See more at https://kulshekhar.github.io/ts-jest/docs/getting-started/options/diagnostics
 PASS  test/cdk-snapshot-sample.test.ts
  ✓ Snapshot test (109 ms)

 › 1 snapshot written.
Snapshot Summary
 › 1 snapshot written from 1 test suite.

Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
Snapshots:   1 written, 1 total
Time:        3.53 s
Ran all test suites.
```

差分を検出するために設定されているプロパティの値を変えます。\
Queue の visibilityTimeout(可視性タイムアウト)の値を 300→600 に更新しています。

```ts diff:lib/sample-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class SampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new Queue(this, 'Queue', {
      visibilityTimeout: cdk.Duration.seconds(600),
    });
  }
}
```

スナップショットテストを再度実行すると想定した差分を検出してくれました。

```sh
% npm run test

> cdk-snapshot-sample@0.1.0 test
> jest

ts-jest[config] (WARN) message TS151002: Using hybrid module kind (Node16/18/Next) is only supported in "isolatedModules: true". Please set "isolatedModules: true" in your tsconfig.json. To disable this message, you can set "diagnostics.ignoreCodes" to include 151002 in your ts-jest config. See more at https://kulshekhar.github.io/ts-jest/docs/getting-started/options/diagnostics
ts-jest[config] (WARN) message TS151002: Using hybrid module kind (Node16/18/Next) is only supported in "isolatedModules: true". Please set "isolatedModules: true" in your tsconfig.json. To disable this message, you can set "diagnostics.ignoreCodes" to include 151002 in your ts-jest config. See more at https://kulshekhar.github.io/ts-jest/docs/getting-started/options/diagnostics
 FAIL  test/cdk-snapshot-sample.test.ts
  ✕ Snapshot test (113 ms)

  ● Snapshot test

    expect(received).toMatchSnapshot()

    Snapshot name: `Snapshot test 1`

    - Snapshot  - 1
    + Received  + 1

    @@ -8,11 +8,11 @@
        },
        "Resources": {
          "Queue5EE69D51": {
            "DeletionPolicy": "Delete",
            "Properties": {
    -         "VisibilityTimeout": 300,
    +         "VisibilityTimeout": 600,
            },
            "Type": "AWS::SQS::Queue",
            "UpdateReplacePolicy": "Delete",
          },
        },

       6 |     const app = new App()
       7 |     const stack = new SampleStack(app, 'SampleStack')
    >  8 |     expect(Template.fromStack(stack)).toMatchSnapshot()
         |                                       ^
       9 | });
      10 |

      at Object.<anonymous> (test/cdk-snapshot-sample.test.ts:8:39)

 › 1 snapshot failed.
Snapshot Summary
 › 1 snapshot failed from 1 test suite. Inspect your code changes or run `npm test -- -u` to update them.

Test Suites: 1 failed, 1 total
Tests:       1 failed, 1 total
Snapshots:   1 failed, 1 total
Time:        3.257 s, estimated 4 s
```

スナップショットテストを更新します。

```sh
% npm run test -- -u
```

## テスティングフレームワークを Vitest に置き換える

CDK を初期構築するとテスティングフレームワークとして`Jest`がインストールされています。\
これを`Jest`から`Vitest`に置き換えます。

`Vitest`はテストの実行が`Jest`と比較して高速で、ソースコードやテストの変更を検知して関連するテストを再実行してくれるので開発体験がよいです。\
書き方も`Jest`とほとんど同じなので置き換えもスムーズにできます。

それでは`Vitest`の Getting started を参考に置き換えを進めます。

https://vitest.dev/guide/

まずパッケージをインストールします。

```sh
% npm install -D vitest
```

package.json の設定を変更してテストを実行するスクリプトに`Vitest`を利用するよう変更します。

```json diff:package.json
  "scripts": {
    ...
-    "test": "jest",
+    "test": "vitest",
    ...
  },
```

`Vitest`の設定ファイルを作成します。

```ts:vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  root: ".",
  test: {
    root: ".",
    globals: true,
    environment: "node",
    include: ["**/test/**/*.{spec,test}.ts"]
  },
});
```

`tsconfig.json`を次のように変更して Vitest をグローバルに利用できるよう変更します。\
この設定を入れることで`Jest`とほぼ変わらない書き方が実現できます。

```json diff:tsconfig.json
{
  "compilerOptions": {
    // ...
    "typeRoots": ["./node_modules/@types", "./node_modules"],
    "types": ["vitest/globals"]
  }
}
```

それではいよいよ`Vitest` でスナップショットテストを実行します。\
先ほど変更した`test`コマンドを実行するとテストが成功しました。

```sh
RERUN  rerun all tests x25

 ✓ test/cdk-snapshot-sample.test.ts (1 test) 69ms
   ✓ Snapshot test 69ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  09:55:38
   Duration  180ms

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

もう一度値を変更してみます。

```ts diff:lib/sample-stack.ts
import * as cdk from 'aws-cdk-lib';
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class SampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new Queue(this, 'Queue', {
      visibilityTimeout: cdk.Duration.seconds(900),
    });
  }
}
```

保存しただけでテストが実行されてますね！コマンドを毎回実行するのは意外と負担になるので嬉しいところです。\
そして今度はテストが失敗してます。ここで`u`を押すことでスナップショットテストを簡単に更新できます。便利です！

```sh diff
% npm run test

> cdk-snapshot-sample@0.1.0 test
> vitest

 ❯ test/cdk-snapshot-sample.test.ts (1 test | 1 failed) 51ms
   × Snapshot test 51ms

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯

 FAIL  test/cdk-snapshot-sample.test.ts > Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

@@ -8,11 +8,11 @@
    },
    "Resources": {
      "Queue5EE69D51": {
        "DeletionPolicy": "Delete",
        "Properties": {
-         "VisibilityTimeout": 600,
+         "VisibilityTimeout": 900,
        },
        "Type": "AWS::SQS::Queue",
        "UpdateReplacePolicy": "Delete",
      },
    },

 ❯ test/cdk-snapshot-sample.test.ts:8:39
      6|     const app = new App()
      7|     const stack = new SampleStack(app, 'SampleStack')
      8|     expect(Template.fromStack(stack)).toMatchSnapshot()
       |                                       ^
      9| });
     10|

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  17:58:41
   Duration  295ms (transform 17ms, setup 0ms, import 182ms, tests 51ms, environment 0ms)

 FAIL  Tests failed. Watching for file changes...
       press u to update snapshot, press h to show help
```

## アセットのハッシュ値を置換する

アセットは CDK アプリをデプロイするときに CloudAssembly（synthesize されるときに作成されるファイル群）にコピーされる資材です。ファイルは zip 圧縮されて S3 へ、コンテナイメージは ECR へプッシュされます。

例えばコンソールから Lambda 関数をデプロイするときに zip ファイルを S3 バケットに配置してデプロイできますよね。\
それと同様に、CDK デプロイするときは、synthesize 時に Lambda 関数のソースコードのアセットを作成し、デプロイする時にアセットとして S3 へ配置します。

どのように行われるかを見ていきます。\
まずは準備のために Stack に Lambda 関数を定義します。簡単化のために SQS Queue は消しちゃいます！

```ts:lib/sample-stack.ts
import * as cdk from 'aws-cdk-lib';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Construct } from 'constructs';
import path from 'path';

export class SampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, 'lambda/index.ts'),
    });
  }
}
```

実行するコードが必要なので Lambda 関数のコードを作成します。

```ts:lib/lambda/index.ts
exports.handler = async () => ({
  statusCode: 200,
  body: 'Hello!',
});
```

スナップショットテストを実行するとスナップショットのリソースが Lambda 関数に書きかわっています。\
ここでスナップショットテストを更新しておいてください。

::: details スナップショットテストの結果

```sh
 FAIL  test/cdk-snapshot-sample.test.ts > Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

@@ -5,17 +5,67 @@
        "Description": "Version of the CDK Bootstrap resources in this environment, automatically retrieved from SSM Parameter Store. [cdk:skip]",
        "Type": "AWS::SSM::Parameter::Value<String>",
      },
    },
    "Resources": {
-     "Queue5EE69D51": {
-       "DeletionPolicy": "Delete",
-       "Properties": {
-         "VisibilityTimeout": 300,
-       },
-       "Type": "AWS::SQS::Queue",
-       "UpdateReplacePolicy": "Delete",
+     "Function76856677": {
+       "DependsOn": [
+         "FunctionServiceRole675BB04A",
+       ],
+       "Properties": {
+         "Code": {
+           "S3Bucket": {
+             "Fn::Sub": "cdk-hnb659fds-assets-${AWS::AccountId}-${AWS::Region}",
+           },
+           "S3Key": "09fa2d70dd1d13608cfae499101d1d49a4b206f30f7176194bf2c708f8817540.zip",
+         },
+         "Environment": {
+           "Variables": {
+             "AWS_NODEJS_CONNECTION_REUSE_ENABLED": "1",
+           },
+         },
+         "Handler": "index.handler",
+         "Role": {
+           "Fn::GetAtt": [
+             "FunctionServiceRole675BB04A",
+             "Arn",
+           ],
+         },
+         "Runtime": "nodejs16.x",
+       },
+       "Type": "AWS::Lambda::Function",
+     },
+     "FunctionServiceRole675BB04A": {
+       "Properties": {
+         "AssumeRolePolicyDocument": {
+           "Statement": [
+             {
+               "Action": "sts:AssumeRole",
+               "Effect": "Allow",
+               "Principal": {
+                 "Service": "lambda.amazonaws.com",
+               },
+             },
+           ],
+           "Version": "2012-10-17",
+         },
+         "ManagedPolicyArns": [
+           {
+             "Fn::Join": [
+               "",
+               [
+                 "arn:",
+                 {
+                   "Ref": "AWS::Partition",
+                 },
+                 ":iam::aws:policy/service-role/AWSLambdaBasicExecutionRole",
+               ],
+             ],
+           },
+         ],
+       },
+       "Type": "AWS::IAM::Role",
      },
    },
    "Rules": {
      "CheckBootstrapVersion": {
        "Assertions": [

 ❯ test/cdk-snapshot-sample.test.ts:8:39
      6|     const app = new App()
      7|     const stack = new SampleStack(app, 'SampleStack')
      8|     expect(Template.fromStack(stack)).toMatchSnapshot()
       |                                       ^
      9| });
     10|

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  18:06:40
   Duration  499ms

 FAIL  Tests failed. Watching for file changes...
       press u to update snapshot, press h to show help
```

:::

スナップショットをよく見ると Lambda 関数に S3Key というプロパティがあります。\
Lambda 関数のコードをアセットとして zip 化して S3 バケットに配置したときのファイル名になります。

```
"S3Key": "09fa2d70dd1d13608cfae499101d1d49a4b206f30f7176194bf2c708f8817540.zip",
```

Lambda 関数のアセットのファイル名はソースコードをハッシュ化したものです。\
そのため Lambda 関数のコードを書き換えるとアセットのファイル名も変わります。

```ts diff:lib/lambda/index.ts
exports.handler = async () => ({
  statusCode: 200,
-  body: 'Hello!',
+  body: 'Hello!!!!!!!',
});
```

スナップショットを見てみるとアセットのハッシュ値が書き換わっていることがわかります。

```diff
 FAIL  test/cdk-snapshot-sample.test.ts > Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

@@ -14,11 +14,11 @@
        "Properties": {
          "Code": {
            "S3Bucket": {
              "Fn::Sub": "cdk-hnb659fds-assets-${AWS::AccountId}-${AWS::Region}",
            },
-           "S3Key": "09fa2d70dd1d13608cfae499101d1d49a4b206f30f7176194bf2c708f8817540.zip",
+           "S3Key": "4d924ae403f9bc881186247d2e9d1102a5189e813a370a9fe04c8294fac386e7.zip",
          },
          "Environment": {
            "Variables": {
              "AWS_NODEJS_CONNECTION_REUSE_ENABLED": "1",
            },

 ❯ test/cdk-snapshot-sample.test.ts:8:39
      6|     const app = new App()
      7|     const stack = new SampleStack(app, 'SampleStack')
      8|     expect(Template.fromStack(stack)).toMatchSnapshot()
       |                                       ^
      9| });
     10|

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  18:08:52
   Duration  321ms

 FAIL  Tests failed. Watching for file changes...
       press u to update snapshot, press h to show help
```

Lambda 関数のコードを変えるタイミングと CDK のソースコードを変更するタイミングは必ずしも一致するものではありません。\
もしスナップショットテストを CI に組み込んでいる場合は、Lambda 関数のコードを書き換えるだけで CDK のスナップショットテストが失敗してしまいます。\
これはよくありませんね。

そこで、アセットのハッシュ値を置換するために Vitest の Custom Serializer を利用します。

https://vitest.dev/guide/snapshot#custom-serializer

Custom Serializer はスナップショットのシリアライズ方法をカスタムできる機能で、生成したスナップショットに対して変更を加えられます。

まずは基本の Custom Serializer を作成します。\
このコードは処理を加えていないので何も起きないはずです。

```ts:snapshot-serializer.ts
import { SnapshotSerializer } from "vitest";

export default {
    serialize(val: string) {
      return val;
    },
    test(val: unknown) {
      return typeof val === "string";
    },
} satisfies SnapshotSerializer;
```

Custom Serializer を`Vitest`で利用できるように`vitest.config.ts`に設定します。

```ts diff:vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  root: ".",
  test: {
    root: ".",
    globals: true,
    environment: "node",
+    snapshotSerializers: ["snapshot-serializer.ts"],
    include: ["**/test/**/*.{spec,test}.ts"]
  },
});
```

変更はないはずですが、スナップショットテストを実行すると大量の差分が出ています。\
文字列の表現方法が変わってしまうらしく、ダブルクォートが外されています。\
大事な差分がわからなくなってしまうので、ここで`u`を押してテストを更新します。

::: details スナップショットテスト実行結果

```diff
FAIL  test/cdk-snapshot-sample.test.ts > Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

  {
-   "Parameters": {
+   Parameters: {
-     "BootstrapVersion": {
+     BootstrapVersion: {
-       "Default": "/cdk-bootstrap/hnb659fds/version",
+       Default: /cdk-bootstrap/hnb659fds/version,
-       "Description": "Version of the CDK Bootstrap resources in this environment, automatically retrieved from SSM Parameter Store. [cdk:skip]",
+       Description: Version of the CDK Bootstrap resources in this environment, automatically retrieved from SSM Parameter Store. [cdk:skip],
-       "Type": "AWS::SSM::Parameter::Value<String>",
+       Type: AWS::SSM::Parameter::Value<String>,
      },
    },
-   "Resources": {
+   Resources: {
-     "Function76856677": {
+     Function76856677: {
-       "DependsOn": [
+       DependsOn: [
-         "FunctionServiceRole675BB04A",
+         FunctionServiceRole675BB04A,
        ],
-       "Properties": {
+       Properties: {
-         "Code": {
+         Code: {
-           "S3Bucket": {
+           S3Bucket: {
-             "Fn::Sub": "cdk-hnb659fds-assets-${AWS::AccountId}-${AWS::Region}",
+             Fn::Sub: cdk-hnb659fds-assets-${AWS::AccountId}-${AWS::Region},
            },
-           "S3Key": "09fa2d70dd1d13608cfae499101d1d49a4b206f30f7176194bf2c708f8817540.zip",
+           S3Key: 4d924ae403f9bc881186247d2e9d1102a5189e813a370a9fe04c8294fac386e7.zip,
          },
-         "Environment": {
+         Environment: {
-           "Variables": {
+           Variables: {
-             "AWS_NODEJS_CONNECTION_REUSE_ENABLED": "1",
+             AWS_NODEJS_CONNECTION_REUSE_ENABLED: 1,
            },
          },
-         "Handler": "index.handler",
+         Handler: index.handler,
-         "Role": {
+         Role: {
-           "Fn::GetAtt": [
+           Fn::GetAtt: [
-             "FunctionServiceRole675BB04A",
+             FunctionServiceRole675BB04A,
-             "Arn",
+             Arn,
            ],
          },
-         "Runtime": "nodejs16.x",
+         Runtime: nodejs16.x,
        },
-       "Type": "AWS::Lambda::Function",
+       Type: AWS::Lambda::Function,
      },
-     "FunctionServiceRole675BB04A": {
+     FunctionServiceRole675BB04A: {
-       "Properties": {
+       Properties: {
-         "AssumeRolePolicyDocument": {
+         AssumeRolePolicyDocument: {
-           "Statement": [
+           Statement: [
              {
-               "Action": "sts:AssumeRole",
+               Action: sts:AssumeRole,
-               "Effect": "Allow",
+               Effect: Allow,
-               "Principal": {
+               Principal: {
-                 "Service": "lambda.amazonaws.com",
+                 Service: lambda.amazonaws.com,
                },
              },
            ],
-           "Version": "2012-10-17",
+           Version: 2012-10-17,
          },
-         "ManagedPolicyArns": [
+         ManagedPolicyArns: [
            {
-             "Fn::Join": [
+             Fn::Join: [
-               "",
+               ,
                [
-                 "arn:",
+                 arn:,
                  {
-                   "Ref": "AWS::Partition",
+                   Ref: AWS::Partition,
                  },
-                 ":iam::aws:policy/service-role/AWSLambdaBasicExecutionRole",
+                 :iam::aws:policy/service-role/AWSLambdaBasicExecutionRole,
                ],
              ],
            },
          ],
        },
-       "Type": "AWS::IAM::Role",
+       Type: AWS::IAM::Role,
      },
    },
-   "Rules": {
+   Rules: {
-     "CheckBootstrapVersion": {
+     CheckBootstrapVersion: {
-       "Assertions": [
+       Assertions: [
          {
-           "Assert": {
+           Assert: {
-             "Fn::Not": [
+             Fn::Not: [
                {
-                 "Fn::Contains": [
+                 Fn::Contains: [
                    [
-                     "1",
+                     1,
-                     "2",
+                     2,
-                     "3",
+                     3,
-                     "4",
+                     4,
-                     "5",
+                     5,
                    ],
                    {
-                     "Ref": "BootstrapVersion",
+                     Ref: BootstrapVersion,
                    },
                  ],
                },
              ],
            },
-           "AssertDescription": "CDK bootstrap stack version 6 required. Please run 'cdk bootstrap' with a recent version of the CDK CLI.",
+           AssertDescription: CDK bootstrap stack version 6 required. Please run 'cdk bootstrap' with a recent version of the CDK CLI.,
          },
        ],
      },
    },
  }

 ❯ test/cdk-snapshot-sample.test.ts:8:39
      6|     const app = new App()
      7|     const stack = new SampleStack(app, 'SampleStack')
      8|     expect(Template.fromStack(stack)).toMatchSnapshot()
       |                                       ^
      9| });
     10|

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  20:06:56
   Duration  320ms

 FAIL  Tests failed. Watching for file changes...
       press u to update snapshot, press h to show help
```

:::

Custom Serializer にアセットのハッシュ値を置換する処理を追加します。

```ts diff:
import { SnapshotSerializer } from "vitest";

export default {
    serialize(val: string) {
-      return val;
+      return val
        // ハッシュ値を`hashed`に置き換え
+       .replace(/[A-Fa-f0-9]{64}/, "hashed");
    },
    test(val: unknown) {
      return typeof val === "string";
    },
} satisfies SnapshotSerializer;
```

スナップショットテストを見るとアセットのハッシュ値が`hashed`に置き換わってます！

```diff
 FAIL  test/cdk-snapshot-sample.test.ts > Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

@@ -12,11 +12,11 @@
          FunctionServiceRole675BB04A,
        ],
        Properties: {
          Code: {
            S3Bucket: cdk-hnb659fds-assets-111111111111-ap-northeast-1,
-           S3Key: d68fdb2bf75b9c067552a37325f200037b3becc94bc3a0970075d61911c0bdbe.zip,
+           S3Key: hashed.zip,
          },
          Handler: index.handler,
          Role: {
            Fn::GetAtt: [
              FunctionServiceRole675BB04A,

 ❯ test/cdk-snapshot-sample.test.ts:43:22
     41|   test('Snapshot test', () => {
     42|     const template = getTemplate('dev');
     43|     expect(template).toMatchSnapshot();
       |                      ^
     44|   });
     45| // });

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  14:31:38
   Duration  340ms

 FAIL  Tests failed. Watching for file changes...
       press u to update snapshot, press h to show help
```

再びコードを変えてみます。

```ts
exports.handler = async () => ({
  statusCode: 200,
  body: 'Hello!!!!!!!????????????',
});
```

コードを変えたのにスナップショットテストの差分がなかったです！成功！！

```sh
Bundling asset SampleStack/Function/Code/Stage...

  ...1aad1469dd1721af86b2318cc7512c63ad0348bfab7ed840-building/index.js  129b

⚡ Done in 1ms
 ✓ test/cdk-snapshot-sample.test.ts (1 test) 347ms
   ✓ Snapshot test  346ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  20:09:37
   Duration  455ms

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

## バンドルをスキップする

TypeScript で書かれた コードは Node.js のランタイムで実行できないため JavaScript へバンドルする必要があります。（要検証ですが Lambda 関数ランタイムの Nodejs 24 では動作するかもしれません）\
本来は`tsc`を利用してバンドルしたファイルをアセットとして利用できるようにする必要がありますが、CDK の`NodejsFunction`Construct を利用すれば意識せずにバンドルできます。

しかし、スナップショットテスト時に Lambda 関数のコードがバンドルできることを検証する必要があるでしょうか？\
本来は CDK のテストがしたいのであってコードの検証をする必要はありません。

実はこれまで利用していた Lambda 関数にも`NodejsFunction`が使われています。\
スナップショットテストの最初にバンドルが行われている様子が見えます。

```sh
Bundling asset SampleStack/Function/Code/Stage...

  ...dced3d9839c7326fec6b7d01e26b35be8864ddaa43583992-building/index.js  129b

⚡ Done in 1ms
```

バンドルは `esbuild` というパッケージを利用して行われます。\
`esbuild`がプロジェクトにインストールされていない場合は、Docker にてバンドルが行われます。\
（その他にも`bundling`の`forceDockerBundling`が true の場合も Docker でバンドルされます）

Docker でバンドルする場合はスキップできないため、`esbuild`をインストールします。

```sh
npm install --save-dev esbuild
```

続いて、テストにバンドルをスキップするコンテキストを渡します。\
`App`に対して`context`を渡すことで、CDK の振る舞いを変えられるコンテキストが数多くあります。\
その中の`BUNDLING_STACKS`に空配列を渡すことで、バンドルがスキップできます。

```ts diff:test/sample-stack.test.ts
import { App } from "aws-cdk-lib";
import { SampleStack } from "../lib/cdk-snapshot-sample-stack";
import { Template } from "aws-cdk-lib/assertions";
+import { BUNDLING_STACKS } from "aws-cdk-lib/cx-api";

test('Snapshot test', () => {
-    const app = new App()
+    const app = new App({
+    context: {
+      [BUNDLING_STACKS]: [],
+    },
+  })
    const stack = new SampleStack(app, 'SampleStack')
    expect(Template.fromStack(stack)).toMatchSnapshot()
});
```

スナップショットテストを実行すると`Bundling`の表示がなくなっておりスキップされていることがわかります。

```sh
 RERUN  rerun all tests x25

 ✓ test/cdk-snapshot-sample.test.ts (1 test) 69ms
   ✓ Snapshot test 69ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   Start at  09:55:38
   Duration  180ms

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

## アセットのステージングをスキップする

synthesize して CloudAssembly を生成するとき、アセットをステージングする処理が行われます。\
ステージングは CDK アプリが終了した後もアセットを利用できるようにするために、ファイルシステム上からステージングディレクトリへコピーする処理です。

アセットのステージングもスナップショットテストには不要です。\
`DISABLE_ASSET_STAGING_CONTEXT`に`true`をセットし、context に渡すことでスキップできます。

```ts
import { App } from 'aws-cdk-lib';
import { SampleStack } from '../lib/cdk-snapshot-sample-stack';
import { Template } from 'aws-cdk-lib/assertions';
-import { BUNDLING_STACKS, DISABLE_ASSET_STAGING_CONTEXT } from 'aws-cdk-lib/cx-api';
+import {
  BUNDLING_STACKS,
  DISABLE_ASSET_STAGING_CONTEXT,
} from 'aws-cdk-lib/cx-api';

test('Snapshot test', () => {
  const app = new App({
    context: {
      [BUNDLING_STACKS]: [],
+      [DISABLE_ASSET_STAGING_CONTEXT]: true,
    },
  });
  const stack = new SampleStack(app, 'SampleStack');
  expect(Template.fromStack(stack)).toMatchSnapshot();
});
```

## cdk.json の機能フラグを渡す

CDK ではたびたび破壊的変更を伴うアップデートがありますが、既存のユーザーに影響を与えないために`機能フラグ`という仕組みが用意されています。\
`機能フラグ`は`cdk.json`に保存されており CDK を初期化する時に取り込まれます。\
利用したい機能だけ有効にすることも可能なので影響度合いを考えながら柔軟に新しい変更を取り込むことができます。

```json:cdk.json
{
  ...
  "context": {
    "@aws-cdk/aws-lambda:recognizeLayerVersion": true,
    "@aws-cdk/core:checkSecretUsage": true,
    "@aws-cdk/core:target-partitions": [
      "aws",
      "aws-cn"
    ],
    "@aws-cdk-containers/ecs-service-extensions:enableDefaultLogDriver": true,
    "@aws-cdk/aws-ec2:uniqueImdsv2TemplateName": true,
    "@aws-cdk/aws-ecs:arnFormatIncludesClusterName": true,
    "@aws-cdk/aws-iam:minimizePolicies": true,
    "@aws-cdk/core:validateSnapshotRemovalPolicy": true,
    "@aws-cdk/aws-codepipeline:crossAccountKeyAliasStackSafeResourceName": true,
    "@aws-cdk/aws-s3:createDefaultLoggingPolicy": true,
    "@aws-cdk/aws-sns-subscriptions:restrictSqsDescryption": true,
    "@aws-cdk/aws-apigateway:disableCloudWatchRole": true,
    "@aws-cdk/core:enablePartitionLiterals": true,
    "@aws-cdk/aws-events:eventsTargetQueueSameAccount": true,
    "@aws-cdk/aws-ecs:disableExplicitDeploymentControllerForCircuitBreaker": true,
    "@aws-cdk/aws-iam:importedRoleStackSafeDefaultPolicyName": true,
    "@aws-cdk/aws-s3:serverAccessLogsUseBucketPolicy": true,
    "@aws-cdk/aws-route53-patters:useCertificate": true,
    "@aws-cdk/customresources:installLatestAwsSdkDefault": false,
    "@aws-cdk/aws-rds:databaseProxyUniqueResourceName": true,
    "@aws-cdk/aws-codedeploy:removeAlarmsFromDeploymentGroup": true,
    "@aws-cdk/aws-apigateway:authorizerChangeDeploymentLogicalId": true,
    "@aws-cdk/aws-ec2:launchTemplateDefaultUserData": true,
    "@aws-cdk/aws-secretsmanager:useAttachedSecretResourcePolicyForSecretTargetAttachments": true,
    "@aws-cdk/aws-redshift:columnId": true,
    "@aws-cdk/aws-stepfunctions-tasks:enableEmrServicePolicyV2": true,
    "@aws-cdk/aws-ec2:restrictDefaultSecurityGroup": true,
    "@aws-cdk/aws-apigateway:requestValidatorUniqueId": true,
    "@aws-cdk/aws-kms:aliasNameRef": true,
    "@aws-cdk/aws-kms:applyImportedAliasPermissionsToPrincipal": true,
    "@aws-cdk/aws-autoscaling:generateLaunchTemplateInsteadOfLaunchConfig": true,
    "@aws-cdk/core:includePrefixInUniqueNameGeneration": true,
    "@aws-cdk/aws-efs:denyAnonymousAccess": true,
    "@aws-cdk/aws-opensearchservice:enableOpensearchMultiAzWithStandby": true,
    "@aws-cdk/aws-lambda-nodejs:useLatestRuntimeVersion": true,
    "@aws-cdk/aws-efs:mountTargetOrderInsensitiveLogicalId": true,
    "@aws-cdk/aws-rds:auroraClusterChangeScopeOfInstanceParameterGroupWithEachParameters": true,
    "@aws-cdk/aws-appsync:useArnForSourceApiAssociationIdentifier": true,
    "@aws-cdk/aws-rds:preventRenderingDeprecatedCredentials": true,
    "@aws-cdk/aws-codepipeline-actions:useNewDefaultBranchForCodeCommitSource": true,
    "@aws-cdk/aws-cloudwatch-actions:changeLambdaPermissionLogicalIdForLambdaAction": true,
    "@aws-cdk/aws-codepipeline:crossAccountKeysDefaultValueToFalse": true,
    "@aws-cdk/aws-codepipeline:defaultPipelineTypeToV2": true,
    "@aws-cdk/aws-kms:reduceCrossAccountRegionPolicyScope": true,
    "@aws-cdk/aws-eks:nodegroupNameAttribute": true,
    "@aws-cdk/aws-ec2:ebsDefaultGp3Volume": true,
    "@aws-cdk/aws-ecs:removeDefaultDeploymentAlarm": true,
    "@aws-cdk/custom-resources:logApiResponseDataPropertyTrueDefault": false,
    "@aws-cdk/aws-s3:keepNotificationInImportedBucket": false,
    "@aws-cdk/core:explicitStackTags": true,
    "@aws-cdk/aws-ecs:enableImdsBlockingDeprecatedFeature": false,
    "@aws-cdk/aws-ecs:disableEcsImdsBlocking": true,
    "@aws-cdk/aws-ecs:reduceEc2FargateCloudWatchPermissions": true,
    "@aws-cdk/aws-dynamodb:resourcePolicyPerReplica": true,
    "@aws-cdk/aws-ec2:ec2SumTImeoutEnabled": true,
    "@aws-cdk/aws-appsync:appSyncGraphQLAPIScopeLambdaPermission": true,
    "@aws-cdk/aws-rds:setCorrectValueForDatabaseInstanceReadReplicaInstanceResourceId": true,
    "@aws-cdk/core:cfnIncludeRejectComplexResourceUpdateCreatePolicyIntrinsics": true,
    "@aws-cdk/aws-lambda-nodejs:sdkV3ExcludeSmithyPackages": true,
    "@aws-cdk/aws-stepfunctions-tasks:fixRunEcsTaskPolicy": true,
    "@aws-cdk/aws-ec2:bastionHostUseAmazonLinux2023ByDefault": true,
    "@aws-cdk/aws-route53-targets:userPoolDomainNameMethodWithoutCustomResource": true,
    "@aws-cdk/aws-elasticloadbalancingV2:albDualstackWithoutPublicIpv4SecurityGroupRulesDefault": true,
    "@aws-cdk/aws-iam:oidcRejectUnauthorizedConnections": true,
    "@aws-cdk/core:enableAdditionalMetadataCollection": true,
    "@aws-cdk/aws-lambda:createNewPoliciesWithAddToRolePolicy": false,
    "@aws-cdk/aws-s3:setUniqueReplicationRoleName": true,
    "@aws-cdk/aws-events:requireEventBusPolicySid": true,
    "@aws-cdk/core:aspectPrioritiesMutating": true,
    "@aws-cdk/aws-dynamodb:retainTableReplica": true,
    "@aws-cdk/aws-stepfunctions:useDistributedMapResultWriterV2": true,
    "@aws-cdk/s3-notifications:addS3TrustKeyPolicyForSnsSubscriptions": true,
    "@aws-cdk/aws-ec2:requirePrivateSubnetsForEgressOnlyInternetGateway": true,
    "@aws-cdk/aws-s3:publicAccessBlockedByDefault": true,
    "@aws-cdk/aws-lambda:useCdkManagedLogGroup": true
  }
}
```

CDK のスナップショットテストでは`cdk.json`の機能フラグが利用されずに実行されます。\
CDKアプリをデプロイするときは`cdk.json`の機能フラグを利用するので、スナップショットテストと実際のテンプレートが異なってしまいます。\
これはまずそうですね。

ということで、`cdk.json`の機能フラグを利用する処理を追加します。

```ts diff:lib/sample-stack.ts
import { App } from 'aws-cdk-lib';
import { SampleStack } from '../lib/cdk-snapshot-sample-stack';
import { Template } from 'aws-cdk-lib/assertions';
import {
  BUNDLING_STACKS,
  DISABLE_ASSET_STAGING_CONTEXT,
} from 'aws-cdk-lib/cx-api';
+import * as fs from 'fs';
+import path from 'path';

// cdk.jsonの機能フラグを取得する関数
+const getContext = (): Record<string, any> => {
+  const cdkJsonPath = path.join(__dirname, '../cdk.json');　// cdk.jsonのパス
+  const cdkJson = JSON.parse(fs.readFileSync(cdkJsonPath, 'utf-8'));
+  return cdkJson.context || {};
+};

test('Snapshot test', () => {
  const app = new App({
    context: {
      // Appのcontextとして渡す
+      ...getContext(),
      [BUNDLING_STACKS]: [],
      [DISABLE_ASSET_STAGING_CONTEXT]: true,
    },
  });
  const stack = new SampleStack(app, 'SampleStack');
  expect(Template.fromStack(stack)).toMatchSnapshot();
});
```

テストを実行すると差分がわらわら出てきました。これが本来の姿だったのです。

```diff
 FAIL  test/cdk-snapshot-sample.test.ts > Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

@@ -16,25 +16,39 @@
            S3Bucket: {
              Fn::Sub: cdk-hnb659fds-assets-${AWS::AccountId}-${AWS::Region},
            },
            S3Key: hashed.zip,
          },
-         Environment: {
+         Handler: index.handler,
+         Role: {
-           Variables: {
+           Fn::GetAtt: [
-             AWS_NODEJS_CONNECTION_REUSE_ENABLED: 1,
+             FunctionServiceRole675BB04A,
+             Arn,
-           },
+           ],
          },
-         Handler: index.handler,
-         Role: {
-           Fn::GetAtt: [
-             FunctionServiceRole675BB04A,
-             Arn,
+         Runtime: nodejs18.x,
+       },
+       Type: AWS::Lambda::Function,
+     },
+     FunctionLogGroup55B80E27: {
+       DeletionPolicy: Retain,
+       Properties: {
+         LogGroupName: {
+           Fn::Join: [
+             ,
+             [
+               /aws/lambda/,
+               {
+                 Ref: Function76856677,
+               },
+             ],
            ],
          },
-         Runtime: nodejs16.x,
+         RetentionInDays: 731,
        },
-       Type: AWS::Lambda::Function,
+       Type: AWS::Logs::LogGroup,
+       UpdateReplacePolicy: Retain,
      },
      FunctionServiceRole675BB04A: {
        Properties: {
          AssumeRolePolicyDocument: {
            Statement: [

 ❯ test/cdk-snapshot-sample.test.ts:23:39
     21|   })
     22|     const stack = new SampleStack(app, 'SampleStack')
     23|     expect(Template.fromStack(stack)).toMatchSnapshot()
       |                                       ^
     24| });
     25|

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  10:13:25
   Duration  207ms

 FAIL  Tests failed. Watching for file changes...
```

`cdk.json`の機能フラグを渡していないと間違った結果を取得してしまうこともあるので、正しい機能フラグを利用するようにしましょう。

## 全ての環境でスナップショットテストを実行する

プロダクト開発では複数の環境に分けて開発を行うことが多いです。\
`dev`環境、`stg`環境、`prod`環境を利用している会社が多いのではないでしょうか（名称は揺れあり）

CDK のスナップショットテストを行うべきは全ての環境です。\
実装時にいちばん近い開発環境でのみスナップショットテストを行うというケースはよく聞きますが、実は本番環境のパラメータのみ影響がある変更があった場合は検知できずデプロイが失敗してしまうかもしれません。\
安全性のためにも全ての環境で行うとよいでしょう。

複数の環境に設定を分けるために設定ファイル（`config.ts`）を実装します。\
`dev`環境、`prod`環境を用意しており、Lambda 関数のタイムアウト値を設定しています。

```ts:lib/config.ts
import { Duration } from "aws-cdk-lib";

export interface Config {
  env: {
    account: string;
    region: string;
  },
  timeout: Duration;
}

// 環境名を渡すと設定が取得できる関数
export const getConfig = (env: string): Config => {
  switch (env) {
    case 'dev':
      return {
        env: {
          account: '111111111111',
          region: 'ap-northeast-1',
        },
        // dev環境のLambda関数のタイムアウトは10分
        timeout: Duration.minutes(10),
      };
    case 'prod':
      return {
        env: {
          account: '111111111111',
          region: 'ap-northeast-1',
        },
        // dev環境のLambda関数のタイムアウトは10分
        timeout: Duration.minutes(15),
      };
    default:
      throw new Error(`Invalid environment: ${env}`);
  }
}
```

Stack では timeout を受け取り Lambda 関数の`timeout`プロパティに渡します。

```ts diff:lib/sample-stack.ts
import { Duration, Stack, StackProps } from 'aws-cdk-lib';
import { NodejsFunction } from 'aws-cdk-lib/aws-lambda-nodejs';
import { Construct } from 'constructs';
import path from 'path';

+interface SampleStackProps extends StackProps {
+  timeout: Duration;
+}

export class SampleStack extends Stack {
-  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
+  constructor(scope: Construct, id: string, props: SampleStackProps) {
    super(scope, id, props);

    new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, 'lambda/index.ts'),
+      timeout: props.timeout,
    });
  }
}
```

スナップショットテストでは、

```ts diff:test/sample-stack.test.ts
import { App } from 'aws-cdk-lib';
import { SampleStack } from '../lib/cdk-snapshot-sample-stack';
import { Template } from 'aws-cdk-lib/assertions';
import {
  BUNDLING_STACKS,
  DISABLE_ASSET_STAGING_CONTEXT,
} from 'aws-cdk-lib/cx-api';
import * as fs from 'fs';
import path from 'path';
import { getConfig } from '../lib/config';

const getContext = (): Record<string, any> => {
  const cdkJsonPath = path.join(__dirname, '../cdk.json');
  const cdkJson = JSON.parse(fs.readFileSync(cdkJsonPath, 'utf-8'));
  return cdkJson.context || {};
};

// 全ての環境名を用意
const parameters = [
  {
    env: 'dev',
  },
  {
    env: 'prod',
  },
];
const getTemplate = (env: string): Template => {
  const app = new App({
    context: {
      ...getContext(),
      [BUNDLING_STACKS]: [],
      [DISABLE_ASSET_STAGING_CONTEXT]: true,
    },
  });
  // 環境ごとのパラメータを取得
  const config = getConfig(env);
  const stack = new SampleStack(app, 'SampleStack', config); // Stackにパラメータを渡す

  return Template.fromStack(stack);
};

describe.each(parameters)('testing environment: %s', (parameter) => {
  test('Snapshot test', () => {
    const template = getTemplate(parameter.env);
    expect(template).toMatchSnapshot();
  });
});
```

スナップショットテストを実行すると用意した全ての環境でテストを行なっていることがわかります。\
（このテストは更新済みのものです）

```sh
 RERUN  rerun all tests x94

 ✓ test/cdk-snapshot-sample.test.ts (2 tests) 109ms
   ✓ testing environment: { env: 'dev' } (1)
     ✓ Snapshot test 86ms
   ✓ testing environment: { env: 'prod' } (1)
     ✓ Snapshot test 23ms

 Test Files  1 passed (1)
      Tests  2 passed (2)
   Start at  10:40:56
   Duration  225ms

 PASS  Waiting for file changes...
       press h to show help, press q to quit
```

## まとめ

CDK のスナップショットテストのプラクティスを紹介しました！\
スナップショットテストをもっと効果的にできるようにできることはたくさんあるので、まだ取り入れていない方はぜひ参考にしてみてください！

## 参考

https://aws.amazon.com/jp/builders-flash/202411/learn-cdk-unit-test/

https://go-to-k.hatenablog.com/entry/cdk-stage-and-dynamic-static-stack

https://go-to-k.hatenablog.com/entry/cdk-unit-tests-tips

https://zenn.dev/yamaren/articles/e9d46231c07b08

https://vitest.dev/
