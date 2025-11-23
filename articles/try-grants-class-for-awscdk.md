---
title: "AWS CDKのGrantsクラスを試してみた"
emoji: "🪄"
type: "tech"
topics: ['aws', 'cdk', 'iac', 'iam']
published: true
---

AWS CDKでは、特定のAWSリソースから他のAWSリソースに対して`grantXxx`メソッドを利用して権限を付与できます。\
`grantXxx`メソッドを利用すれば、複雑な権限設定をとても簡単に書けます。

例えば、S3バケットに対してread権限を付与したいときは、次のように書けます。

```ts
export class CdkSampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const func = new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, 'lambda', 'index.ts'),
      handler: 'handler',
      runtime: Runtime.NODEJS_22_X,
    });
    const bucket = new Bucket(this, 'Bucket');

    // S3バケットがLambda関数に対してread許可を付与
    bucket.grantRead(func);
  }
}
```

次のようなIAMポリシーが作成され、S3バケットのreadに関する許可設定が付与されています。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:GetBucket*",
                "s3:GetObject*",
                "s3:List*"
            ],
            "Resource": [
                "arn:aws:s3:::cdksample-bucketXXXXXXXXX",
                "arn:aws:s3:::cdksample-bucketXXXXXXXXX/*"
            ],
            "Effect": "Allow"
        }
    ]
}
```

新たに`Grant`クラスを利用できるようになったとのことで調査してみました！

## Grantsクラスが利用できるようになりました

2025年11月21日にCDKのバージョンがv2.227.0に更新され、`S3`・`DynamoDB`・`Step Functions`の`Grants`オブジェクトが追加されています。

> s3: add BucketGrants
> dynamodb: add TableGrants and StreamGrants
> stepfunctions: add StateMachineGrants
> [aws-cdk Releases/v2.227.0](https://github.com/aws/aws-cdk/releases/tag/v2.227.0)

この`Grants`オブジェクトがどのように利用できるのか検証していきます。

## 仕様調査

AWS CDKのAPIリファレンスには、今のところあまり情報が載っていませんでした。

> Collection of grant methods for a Bucket.
> [BucketGrants](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.BucketGrants.html)
>
> A set of permissions to grant on a Table.
> [TableGrants](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_dynamodb.TableGrants.html)
>
> Collection of grant methods for a IStateMachineRef.
> [StateMachineGrants](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_stepfunctions.StateMachineGrants.html)

AWS DevTools Hero 後藤さんによると、`Grants`クラスはL1, L2どちらのConstructにも適用できるみたいです。

https://x.com/365_step_tech/status/1991702573297070374?s=20

L1 Constructにも適用できることで、これまでL1しかなかったリソースや、L1のまま利用していたリソースにも`Grant`が利用できそうです！

また、これまで利用できていた`grantXxx`メソッドは、引き続き利用できるみたいです。
https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html#grantwbrreadidentity-objectskeypattern

これも後藤さんからの情報ですが`grantXxx`メソッドは利用できるものの今後は非推奨になるみたいです。
https://x.com/365_step_tech/status/1991703504285790278?s=20

## やってみた

ということで新しい `Grants` クラスを利用してL1, L2 Constructに対して権限を付与してみます。

### L2 Constructへの適用

まずは今までも`grantXxx`メソッドが利用できていたL2 Constructで`Grants`クラスを利用してみます。

ドキュメントを見るとL2 Constructに`grants`メソッドが実装されています。

> grants
> Type: BucketGrants
> Collection of grant methods for a Bucket.
>
> [Bucket.grants](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.Bucket.html#grants)

`BucketGrants`クラスには、`read`や`delete`、`publicAccess`など、今まで`grantXxx`メソッドとして存在していたメソッドが実装されています。

> Collection of grant methods for a Bucket.
> [BucketGrants](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.BucketGrants.html)

概要がわかったので置き換えてみます。

これまで利用していた`grantXxx`メソッドはこのように書けました。\
（あとで比較できるように、ここでスナップショットテストを保存しています）

```ts
export class CdkSampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const func = new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, 'lambda', 'index.ts'),
      handler: 'handler',
      runtime: Runtime.NODEJS_22_X,
    });
    const bucket = new Bucket(this, 'Bucket');

    // read権限の付与
    bucket.grantRead(func);
  }
}
```

`Grants`クラスに置き換えてみましたが、簡単に変更できました。

```diff
export class CdkSampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const func = new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, 'lambda', 'index.ts'),
      handler: 'handler',
      runtime: Runtime.NODEJS_22_X,
    });
    const bucket = new Bucket(this, 'Bucket');

-   bucket.grantRead(func);
+   bucket.grants.read(func);
  }
}
```

スナップショットテストを実行してみましたが、変更なしでした。

```shell
> vitest

...

 ✓ test/cdk-sample.test.ts (1 test) 119ms
   ✓ Snapshot test 119ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
    ...
```

### L1 Constructへの適用

L1へ適用するために`BucketGrants`クラスのドキュメントをしばらく覗いていましたが、それらしき記述はありませんでした。

https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_s3.BucketGrants.html

`BucketGrants`クラスの実装を見てみると、`_fromBucket`というstaticメソッドが実装されていました。

```ts
/**
 * Creates grants for an IBucketRef
 *
 * @internal
 */
static _fromBucket(bucket: IBucketRef): BucketGrants;
```

`IBucketRef`を受け取って`BucketGrants`メソッドを返すので、このメソッドを利用すれば権限が付与できそうです。

しかし、internalメソッドであるため、外部利用は本来想定されていないようです。\
もし他に利用できる方法があれば教えてください！

外部利用は想定されていなさそうですが、利用できそうなので試してみます。

まず、S3バケットのL1を利用して、S3バケットread権限がついた状態を実装します。\
（ここでスナップショットテストも更新してます）

```ts
export class CdkSampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const cfnBucket = new CfnBucket(this, 'Bucket');

    const role = new Role(this, 'FunctionServiceRole', {
      assumedBy: new ServicePrincipal('lambda.amazonaws.com'),
      managedPolicies: [
        ManagedPolicy.fromAwsManagedPolicyName(
          'service-role/AWSLambdaBasicExecutionRole',
        ),
      ],
    });
    role.addToPolicy(
      new PolicyStatement({
        actions: ['s3:GetObject*', 's3:GetBucket*', 's3:List*'],
        resources: [cfnBucket.attrArn, `${cfnBucket.attrArn}/*`],
      }),
    );

    const func = new NodejsFunction(this, 'Function', {
      entry: 'lib/lambda/index.ts',
      handler: 'handler',
      runtime: Runtime.NODEJS_22_X,
      // read権限の付与
      role,
    });
  }
}
```

`BucketGrants`クラスの`_fromBucket`メソッドを利用して実装します。\
こちらはかなりスッキリしましたね！

```ts
export class CdkSampleStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const cfnBucket = new CfnBucket(this, 'Bucket');
    const func = new NodejsFunction(this, 'Function', {
      entry: 'lib/lambda/index.ts',
      handler: 'handler',
      runtime: Runtime.NODEJS_22_X,
    });

    const grants = BucketGrants._fromBucket(cfnBucket);
    // read権限の付与
    grants.read(func);
  }
}
```

スナップショットテストを実行してみると、変更がないことがわかります。

```shell
> vitest

...

 ✓ test/cdk-sample.test.ts (1 test) 119ms
   ✓ Snapshot test 119ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   ...
```

## まとめ

新しく実装された`Grants`クラスを利用して権限の付与を試してみました。\
L2での使い勝手はほぼ変わらず、L1でも使える様になりました！

L1の`Grants`クラスは、今後`_fromXxx`メソッドが公開される形になるのでしょうか？\
いずれにせよ続報を待つこととします！
