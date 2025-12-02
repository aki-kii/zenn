---
title: 'AWS CDKの新機能「Mixins」を試してみた'
emoji: '🍸'
type: 'tech'
topics: ['aws', 'cdk', 'iac']
published: true
---

## はじめに

AWS CDK で新機能の「Mixins」が Developers preview 版として利用可能になりました。

> mixins-preview: developer preview of CDK Mixins
>
> [v2.229.0 リリースノート](https://github.com/aws/aws-cdk/releases/tag/v2.229.0)

「ミックスインズ」と読み、語源は OOP(オブジェクト指向プログラミング)の Mixin から来ています。\
OOP の Mixin は、単体では動作せずサブクラスに継承することで機能を利用できるクラスです。\
CDK Mixins は L1 や L2 などの Construct に利用可能な、共通的に機能を提供する仕組みなのです。

詳しい機能の紹介は AWS DevTools Hero の後藤さんが解説してくれています。

https://go-to-k.hatenablog.com/entry/cdk-mixins

本記事では、実際に Mixins を試してみることで使い勝手を紹介しようと思います。

## ざっくり概要

- L1, L2 Construct などの Construct の機能差を埋める仕組み
- Construct tree の特定のスコープ配下のリソースに変更を加えられる
- Mixins はリソースの変更、Aspects はバリデーションを推奨
- リソース固有 Mixins: L1 Construct に必要な抽象化を選択して組み込める
- L1 プロパティ Mixins: L2 Construct でエスケープハッチを利用せず型安全に新しいプロパティを利用できる
- カスタム Mixins を利用すれば自分好みに Mixins を定義できる
- リソース間 Mixins: 複数のサービスに共通している一般的なパターンを一貫して設定できる

## Mixins とは

Mixins を利用すれば L1 や L2 などの Construct の性能差を埋めて、どの Construct を利用して CDK アプリを構成しても CDK の恩恵を最大限受けられます。\
L1 Construct だと抽象化された機能が利用できなかったり L2 Construct だと新しいプロパティは実装待ちになっていたりと、Mixins は抽象化された様々な Construct の機能を提供します。

2025/12/2 時点では Developer preview 版となっているため、機能が限定的で破壊的変更が起こりえることにご注意ください。

## Mixins の対象範囲、Aspects との違い

Mixins は Construct tree の特定のスコープ配下の Construct に作用します。\
これは Aspects と同じ対象範囲です。Aspects も特定のスコープ配下の Construct に作用しリソースのプロパティの上書きや動作の検証を加えられます。

Mixins の RFC によると、Mixins で変更を加えて Aspects では動作の検証を行うとよいとされています。

> We recommend to use Mixins to make changes, and to use Aspects to validate behaviors.
>
> https://github.com/aws/aws-cdk-rfcs/blob/9cba7a86f4c052413d557ba55722606a98d33739/text/0814-cdk-mixins.md#mixins-and-aspects

## やってみた

概要を掴めたので早速 Mixins を使ってみます！

### インストール

Mixins は Developer Preview 版であるためまだ aws-cdk-lib パッケージには組み込まれておらず、利用するためにはインストールする必要があります。

```sh
npm install -D @aws-cdk/mixins-preview
```

### 基本的な使い方

`Mixins.of()`メソッドの引数に Construct を渡すことで L1, L2 Construct それぞれの S3 バケットに対してバージョニングを有効にします。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Bucket, CfnBucket } from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';
import { Mixins } from '@aws-cdk/mixins-preview/core';
import { EnableVersioning } from '@aws-cdk/mixins-preview/aws_s3/mixins';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const cfnBucket = new CfnBucket(this, 'CfnBucket');
    // L1 ConstructのCfnBucketに対してバージョニングを有効化
    Mixins.of(cfnBucket).apply(new EnableVersioning());

    const bucket = new Bucket(this, 'Bucket');
    // L2 ConstructのBucketに対してバージョニングを有効化
    Mixins.of(bucket).apply(new EnableVersioning());
  }
}
```

複数の Mixins を適用することも可能です。

```ts
export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const bucketA = new Bucket(this, 'BucketA');
    // 複数のMixinsを指定できる
    Mixins.of(bucketA).apply(new EnableVersioning(), new AutoDeleteObjects());

    const bucketB = new Bucket(this, 'BucketB');
    // メソッドチェーンを利用することでも複数のMixinsを指定できる
    Mixins.of(bucketB)
      .apply(new EnableVersioning())
      .apply(new AutoDeleteObjects());
  }
}
```

L1, L2 Construct の`with()`メソッドを利用することでも Mixins を適用できます。\
現時点では`@aws-cdk/mixins-preview/with`モジュールのインストールが必要です。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { EnableVersioning } from '@aws-cdk/mixins-preview/aws_s3/mixins';
import '@aws-cdk/mixins-preview/with';
import { Bucket, CfnBucket } from 'aws-cdk-lib/aws-s3';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const cfnBucket = new CfnBucket(this, 'CfnBucket');
    // with メソッドをL1 Constructに適用
    cfnBucket.with(new EnableVersioning());

    const bucket = new Bucket(this, 'Bucket');
    // with メソッドをL2 Constructに適用
    bucket.with(new EnableVersioning());
  }
}
```

通常は Mixins の処理にサポートされていないリソースを対象とした場合に処理がスキップされますが、`mustApply()`メソッドを利用することでバリデーションエラーを発生させることが可能です。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { Mixins } from '@aws-cdk/mixins-preview/core';
import { EnableVersioning } from '@aws-cdk/mixins-preview/aws_s3/mixins';
import { Queue } from 'aws-cdk-lib/aws-sqs';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const queueA = new Queue(this, 'QueueA');
    // `EnableVersioning`はS3 バケットにのみ有効
    // `apply`メソッドを利用すると処理をスキップする
    Mixins.of(queueA).apply(new EnableVersioning());

    const queueB = new Queue(this, 'QueueB');
    // `mustApply`メソッドを利用するとバリデーションエラーが発生
    Mixins.of(queueB).mustApply(new EnableVersioning());
  }
}
```

SQS Queue には`EnableVersioning`を適用できないというエラーメッセージが出ました。

```sh
ValidationError: Mixin EnableVersioning could not be applied to Queue2 but was requested to.
```

### スコープに適用する Mixins

`Mixins.of()`メソッドにスコープを渡せばすべてのリソースに適用されます。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Bucket } from 'aws-cdk-lib/aws-s3';
import { Construct } from 'constructs';
import { Mixins } from '@aws-cdk/mixins-preview/core';
import { EnableVersioning } from '@aws-cdk/mixins-preview/aws_s3/mixins';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Bucket(this, 'BucketA');
    new Bucket(this, 'BucketB');

    // このスタックのスコープ配下全てのリソースにMixinsを適用
    Mixins.of(this).apply(new EnableVersioning());
  }
}
```

### リソース固有の Mixins

S3 バケットや SQS キューなどの特定の AWS サービスに対してL2 Construct で利用できるような抽象化された操作を行う機能です。\
これまで例として挙げていた S3 バケットの`EnableVersioning`もその一つです。

`EnableVersioning`はネストしたプロパティの簡単な抽象化でしたが、`AutoDeleteObject`を利用すれば「S3 バケットを削除するときにバケット内のオブジェクトを自動で削除してくれる」機能を持っているためとても強力です。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { AutoDeleteObjects } from '@aws-cdk/mixins-preview/aws_s3/mixins';
import '@aws-cdk/mixins-preview/with';
import { CfnBucket } from 'aws-cdk-lib/aws-s3';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new CfnBucket(this, 'CfnBucket')
        // S3バケットを削除するときに中身のオブジェクトを削除する処理を追加
        .with(new AutoDeleteObjects());
  }
}
```

これまで L2 Construct のみに提供されていた抽象化の機能がこれからは L1 Construct でも利用できるようになります。\
全ての抽象化を受け入れたくないけど、運用を簡単にするために一部の抽象化メソッドを利用したい場合に便利です。



リソース固有の Mixins の抽象化機能はまだ実装が進んでおらず、まだあまり機能は提供されていません（自分が見つけたのは紹介した 2 つのみです）\
今後は L2 Construct に実装されている抽象化機能を実装されていくのではないでしょうか。


#### L1 プロパティ Mixins

リソースのプロパティを指定して上書きする機能です。\
CloudFormation の仕様に合わせて自動生成されるため、L2 Construct の実装を待たずに利用できます。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { CfnBucketPropsMixin } from '@aws-cdk/mixins-preview/aws_s3/mixins';
import '@aws-cdk/mixins-preview/with';
import { Bucket } from 'aws-cdk-lib/aws-s3';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Bucket(scope, 'Bucket').with(
      // L1 Construct のプロパティを指定
      new CfnBucketPropsMixin({
        versioningConfiguration: { status: 'Enabled' },
      })
    );
  }
}
```

エスケープハッチを利用してL1 プロパティを設定する場合は間違えても気付けませんが、L1 プロパティMixinsでは型サポートが得られるため安全に書けます。

<!-- この操作は対象リソースのプロパティと競合するので 2 つのマージ戦略から解決策を選べます。

- MERGE: リソースのプロパティに指定したプロパティをマージ(既に設定されていた場合は上書き)。デフォルト設定
- OVERRIDE: 指定したプロパティでリソースのプロパティを置き換えます

// TODO: 試す

```ts
declare const bucket: s3.CfnBucket;

// MERGE:
Mixins.of(bucket).apply(
  new CfnBucketPropsMixin(
    { versioningConfiguration: { status: 'Enabled' } },
    { strategy: PropertyMergeStrategy.MERGE }
  )
);

// OVERRIDE: Replaces existing property values
Mixins.of(bucket).apply(
  new CfnBucketPropsMixin(
    { versioningConfiguration: { status: 'Enabled' } },
    { strategy: PropertyMergeStrategy.OVERRIDE }
  )
);
```

// TODO: 結果を書く -->


### カスタム Mixins

Mixins を自分好みにカスタマイズできます。\
企業のポリシーに合わせた Mixins や設定の集合を作ることや、独自の抽象化機能を定義するのもよさそうです。

カスタム Mixins を定義するには`IMixin`インターフェースを実装する必要があります。\
すでに`Mixin`抽象クラスが`IMixin`の一部を実装しているため、`Mixin`を継承するのがよいでしょう。\

S3 バケットの EnableVersioning をカスタム Mixins で再実装してみます。

実装が必要なのは次のメソッドです。

- supports: Mixins を適用したいリソースタイプを決定します。適用されない場合は処理をスキップします
- applyTo: Mixins で適用する処理を書きます

```ts
import { Mixin } from '@aws-cdk/mixins-preview/core';
import { CfnBucket } from 'aws-cdk-lib/aws-s3';
import { IConstruct } from 'constructs';

export class EnableVersioning extends Mixin {
  // カスタムMixinsを適用したいリソースタイプか判断する
  supports(construct: IConstruct): boolean {
    return construct instanceof CfnBucket;
  }

  // カスタムMixinsが適用する処理
  applyTo(bucket: IConstruct): IConstruct {
    // 渡されたS3バケットのバージョニングを有効化
    (bucket as CfnBucket).versioningConfiguration = {
      status: 'Enabled',
    };
    return bucket;
  }
}
```

### サービス間 Mixins

複数のリソースタイプに適用できる一般なパターンの Mixins です。

RFC のサンプルでは暗号化を適用するパターンが紹介されており、S3 バケット・ロググループ・DynamoDB テーブルに対してサービス間 Mixins を適用しています。

```ts
// Same mixin works across different resource types
const bucket = new s3.CfnBucket(scope, 'Bucket').with(new EncryptionAtRest());

const logGroup = new logs.CfnLogGroup(scope, 'LogGroup').with(
  new EncryptionAtRest()
);

const table = new dynamodb.CfnTable(scope, 'Table').with(
  new EncryptionAtRest()
);
```

しかし執筆時点で最新の v2.231.0 の段階ではまだ実装されていないのでカスタム Mixins で擬似的に実装してみます。

S3 バケットと DynamoDB テーブルに 暗号化を適用するカスタム Mixins です。

```ts
import { Mixin } from '@aws-cdk/mixins-preview/core';
import { CfnTable } from 'aws-cdk-lib/aws-dynamodb';
import { CfnBucket } from 'aws-cdk-lib/aws-s3';
import { IConstruct } from 'constructs';

export class EncryptionAtRest extends Mixin {
  supports(construct: IConstruct): boolean {
    return construct instanceof CfnBucket || construct instanceof CfnTable;
  }

  applyTo(construct: IConstruct): IConstruct {
    if (construct instanceof CfnBucket) {
      construct.bucketEncryption = {
        serverSideEncryptionConfiguration: [
          {
            serverSideEncryptionByDefault: {
              sseAlgorithm: 'AES256',
            },
          },
        ],
      };
    }

    if (construct instanceof CfnTable) {
      construct.sseSpecification = {
        sseEnabled: true,
        sseType: 'AES256',
      };
    }

    return construct;
  }
}
```

利用するときはスコープに対して適用します。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import '@aws-cdk/mixins-preview/with';
import { Bucket } from 'aws-cdk-lib/aws-s3';
import { AttributeType, Table } from 'aws-cdk-lib/aws-dynamodb';
import { Mixins } from '@aws-cdk/mixins-preview/core';
import { EncryptionAtRest } from './mixins/encryption-at-rest';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Bucket(this, 'Bucket');
    new Table(this, 'Table', {
      partitionKey: {
        name: 'id',
        type: AttributeType.STRING,
      },
    });
    Mixins.of(this).apply(new EncryptionAtRest());
  }
}
```

<!-- ### その他 Mixins の仕様 -->

## まとめ

CDK の新しい機能 Mixins を試してみました。

L1, L2 Construct の境界を近づけることで組織のポリシーによってConstructを使い分ける未来がみえました。
特にL1 Construct は Mixins 以外にも Grants クラスや IXxxReference クラスが実装されたためどんどん抽象化のための機能が拡充されています。

まだDeveloper Preview版なので今後の機能拡充が楽しみです！\
特にサービス間Mixinsにどんな機能が追加されるのか注目してます👀
