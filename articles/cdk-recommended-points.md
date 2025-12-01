---
title: 'AWS CDKの推しポイントを紹介します'
emoji: '😻'
type: 'tech'
topics: ['aws', 'cdk', 'iac']
published: true
---

本記事は『AWS CDK Advent Calendar 2025』の1日目の記事です。

https://qiita.com/advent-calendar/2025/aws-cdk

AWS CDK は AWS の IaC サービスの 1 つで AWS のリソースをプログラミング言語で定義して管理・デプロイを行うツールです。\
同じ AWS の IaC サービスである CloudFormation の抽象化レイヤーとして動作し、CDK アプリをCloudFormationテンプレートへ変換します。

CDKは「インフラリソースの設定を抽象化して定義する」という点で他のデプロイ方法やIaCサービスにはない魅力を持っていると感じます。\
そんな AWS CDK の推しポイントをカテゴリに分けて紹介します！

- IaC 編
- IDE 編
- コアコンセプト編

## IaC 編

AWSへのデプロイ方法はIaCの他にも主なところでAWS コンソールやAWS CLI、SDKがあります。

![alt text](/images/cdks-recommended-points/make-aws-resource.dio.png)

IaC ではコードで作成した定義ファイルと現在の状態の差分によってデプロイが行われます。

![alt text](/images/cdks-recommended-points/whats-iac.dio.png)

AWS CDK が IaC であることによる推しポイントをあげていきます。

### 1. 定義ファイルの変更管理ができる

AWS CDK の定義ファイル(コード)はプログラミング言語で書かれているため、Git などのバージョン管理システムを用いることで変更管理が容易に行えます。\
リソースの変更差分を残せる上、最新の状態が一目瞭然です。

![alt text](/images/cdks-recommended-points/diff-for-github.png)

### 2. リソースの設定を集約できる

AWS コンソールでリソースの関係性を見ようと思った時に迷ったことはありませんか？

例えば S3 バケットからイベント通知で Lambda 関数が起動し DynamoDB テーブルに書き込む構成の関連性を調査しようとした時、次のように調べることとなると思います。

![alt text](/images/cdks-recommended-points/resource-trace.dio.png)

（Lambda 関数のコードから依存リソースを追うこともできます）

CDK ではリソースの設定が Git リポジトリに集約されているため、リソースの関係性を把握しやすいです。

### 3. デプロイの安全性が向上する

AWS コンソールや CLI などを用いてのデプロイでは、リソースごとにデプロイして確認する必要があるため手順が複雑になります。\
手順が複雑になるとクリックミスや設定の確認漏れが発生しやすくなります。

CDK のデプロイはリソースを集めた「スタック」の単位で実行されるため、手順が簡素となりデプロイの属人性を排除できます。

![alt text](/images/cdks-recommended-points/deploy-safety.dio.png)

AWS CDK デプロイコマンドの一例です。

```shell
npx cdk deploy
```

CDK のデプロイコマンド実行後は CloudFormation によってデプロイされるため、人間が操作する時間を減らせます。

### 4. 環境の複製が容易

ソフトウェア開発では開発・検証・本番などの複数の環境を利用することが一般的です。\
IaC ではインフラリソースをコードで定義するため、そのコードを用いて複数の環境にデプロイできます。

![alt text](/images/cdks-recommended-points/multi-environment.dio.png)

CDK では「デプロイ対象の AWS アカウントを分ける」「スタック名に環境名を含める」ことで複数環境へのデプロイを実現可能です。\
基本的にはどちらも取り入れることをお勧めします。\
(「スタック名に環境名を含める」理由は、リソースの物理名を自動生成する場合に環境名が含まれるので名前の衝突が発生しないためです)

設定ファイルを用意することで環境ごとに異なるプロパティを設定できます。\
例えば、開発環境では Lambda のメモリを 512MB、本番環境では 2048MB などの異なる設定にできます。

## IDE 編

Visual Studio Code のような IDE(統合開発環境)では、ソフトウェア開発をサポートする機能が多く組み込まれています。\
AWS CDK は TypeScript や Python などのプログラミング言語で書くため、IDE のサポートを受けられます。

CDK におけるIDE のサポート機能についての推しポイントを、次のコードをベースに紹介します。\
可視性タイムアウトを 30 秒に設定した SQS キューを作成しています。

```ts
import { Duration } from 'aws-cdk-lib';
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class MyConstruct extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new Queue(this, 'Queue', {
      visibilityTimeout: Duration.seconds(30),
    });
  }
}
```

### 5. コード補完が利用できる

コードの予測をしてくれる機能です。\
途中まで入力したら予測を出してくれます。

![alt text](/images/cdks-recommended-points/code-completion-suggest.png)

階層構造になっているプロパティもリストアップしてくれます。

![alt text](/images/cdks-recommended-points/code-completion-list.png)

コード補完を利用すれば typo も減り、どのようなプロパティがあるのかをドキュメントなしで探せます。

### 6. ドキュメントがすぐ読める

プロパティに設定する値を確認する時、ドキュメントを読むと思います。\
IDE 上でオブジェクトにホバーすれば、実装に記載されているドキュメントを見れます。

![alt text](/images/cdks-recommended-points/documentation.png)

ブラウザでドキュメントを開かずとも説明が読めるので時間を短縮できます。

### 7. 構文エラーがわかる

CDK の構文を間違えた時に構文エラーとしてエラー内容が表示されます。

構文エラーの中でも型エラーを紹介します。

プロパティにはプリミティブ型(`string`や`int`など)とは違う独自の`型`を渡すことができるものがあります。\
例えば`Duration`型には、期間を指定でき、次の例では "30 秒" を表しています。

```ts
import { Duration } from 'aws-cdk-lib';

Duration.seconds(30);
```

本来`Duration`型を渡さないといけないのに、`number`型を渡すとエラーとなります。

![alt text](/images/cdks-recommended-points/syntax-error-code.png)

本来は`Duration`型を渡す必要があるのに`number`型を渡していると言われてます。

![alt text](/images/cdks-recommended-points/syntax-error-message.png)

コーディング中に気づけるので設定間違いを減らせます。

## AWS CDK のコアコンセプト

AWS CDK は CloudFormation の抽象化レイヤーとして動作し CDK アプリを CloudFormation テンプレートへ変換してデプロイします。\
抽象化するにあたって実装された概念がコアコンセプトとしてデベロッパーガイドに記載されています。\
https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/core-concepts.html

コアコンセプトに関する推しポイントをあげていきます。

### 8. L2 Construct

L2 Construct は CloudFormation テンプレートのプロパティを利用者にとって扱いやすいように抽象化しています。\
この抽象化技術により IaC を用いた開発を簡易化でき、コードの可読性も高まります。

L2 で行われている抽象化として、適切なデフォルトプロパティやベストプラクティスに沿ったセキュリティポリシー・関連リソースの作成の他、リソースの特性に沿ったヘルパーメソッドが用意されています。

![alt text](/images/cdks-recommended-points/l2abstruct.dio.png)

抽象化によってリソースの設定がブラックボックスになるわけではなく、L2 Construct にプロパティを渡すことで CloudFormation と変わらない設定もできます。

Amazon VPC を作成する L2 Construct の実装例を見てみます。

![alt text](/images/cdks-recommended-points/vpc-architecture.dio.png)

CloudFormation の Resources ブロックに書くと 174 行もかかります。\
作るのに手間がかかりますし、視認性も良くありません。

:::details VPC の CloudFormation テンプレート

```yaml
Resources:
  NetworkingVpc6B5E6F44:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: '10.0.0.0/16'
      EnableDnsHostnames: 'true'
      EnableDnsSupport: 'true'
      InstanceTenancy: 'default'
  NetworkingVpcpublicSubnet1SubnetEC29329E:
    Type: 'AWS::EC2::Subnet'
    Properties:
      AvailabilityZone:
        Fn::Select:
          - '0'
          - Fn::GetAZs: ''
      CidrBlock: '10.0.0.0/24'
      MapPublicIpOnLaunch: 'true'
      Tags:
        - Key: 'aws-cdk:subnet-name'
          Value: 'public'
        - Key: 'aws-cdk:subnet-type'
          Value: 'Public'
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/publicSubnet1'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcpublicSubnet1RouteTable2E3445F2:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      Tags:
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/publicSubnet1'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcpublicSubnet1RouteTableAssociation97DD2CA1:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId:
        Ref: 'NetworkingVpcpublicSubnet1RouteTable2E3445F2'
      SubnetId:
        Ref: 'NetworkingVpcpublicSubnet1SubnetEC29329E'
  NetworkingVpcpublicSubnet1DefaultRoute90D9E52F:
    Type: 'AWS::EC2::Route'
    Properties:
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId:
        Ref: 'NetworkingVpcIGW21218DAB'
      RouteTableId:
        Ref: 'NetworkingVpcpublicSubnet1RouteTable2E3445F2'
    DependsOn:
      - 'NetworkingVpcVPCGW12E561D8'
  NetworkingVpcpublicSubnet2SubnetB1EC0FB5:
    Type: 'AWS::EC2::Subnet'
    Properties:
      AvailabilityZone:
        Fn::Select:
          - '1'
          - Fn::GetAZs: ''
      CidrBlock: '10.0.1.0/24'
      MapPublicIpOnLaunch: 'true'
      Tags:
        - Key: 'aws-cdk:subnet-name'
          Value: 'public'
        - Key: 'aws-cdk:subnet-type'
          Value: 'Public'
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/publicSubnet2'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcpublicSubnet2RouteTable8AA4DF78:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      Tags:
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/publicSubnet2'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
      aws:cdk:path: 'CdkSample/Networking/Vpc/publicSubnet2/RouteTable'
  NetworkingVpcpublicSubnet2RouteTableAssociationCD7AC5EA:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId:
        Ref: 'NetworkingVpcpublicSubnet2RouteTable8AA4DF78'
      SubnetId:
        Ref: 'NetworkingVpcpublicSubnet2SubnetB1EC0FB5'
  NetworkingVpcpublicSubnet2DefaultRoute5F0CBE68:
    Type: 'AWS::EC2::Route'
    Properties:
      DestinationCidrBlock: '0.0.0.0/0'
      GatewayId:
        Ref: 'NetworkingVpcIGW21218DAB'
      RouteTableId:
        Ref: 'NetworkingVpcpublicSubnet2RouteTable8AA4DF78'
    DependsOn:
      - 'NetworkingVpcVPCGW12E561D8'
  NetworkingVpcprivateSubnet1Subnet7D313D7C:
    Type: 'AWS::EC2::Subnet'
    Properties:
      AvailabilityZone:
        Fn::Select:
          - '0'
          - Fn::GetAZs: ''
      CidrBlock: '10.0.2.0/24'
      MapPublicIpOnLaunch: 'false'
      Tags:
        - Key: 'aws-cdk:subnet-name'
          Value: 'private'
        - Key: 'aws-cdk:subnet-type'
          Value: 'Isolated'
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/privateSubnet1'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcprivateSubnet1RouteTableBA07DE42:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      Tags:
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/privateSubnet1'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcprivateSubnet1RouteTableAssociation325591CC:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId:
        Ref: 'NetworkingVpcprivateSubnet1RouteTableBA07DE42'
      SubnetId:
        Ref: 'NetworkingVpcprivateSubnet1Subnet7D313D7C'
  NetworkingVpcprivateSubnet2Subnet390F8998:
    Type: 'AWS::EC2::Subnet'
    Properties:
      AvailabilityZone:
        Fn::Select:
          - '1'
          - Fn::GetAZs: ''
      CidrBlock: '10.0.3.0/24'
      MapPublicIpOnLaunch: 'false'
      Tags:
        - Key: 'aws-cdk:subnet-name'
          Value: 'private'
        - Key: 'aws-cdk:subnet-type'
          Value: 'Isolated'
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/privateSubnet2'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcprivateSubnet2RouteTable6A66AAB8:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      Tags:
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc/privateSubnet2'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
  NetworkingVpcprivateSubnet2RouteTableAssociation0175E56E:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      RouteTableId:
        Ref: 'NetworkingVpcprivateSubnet2RouteTable6A66AAB8'
      SubnetId:
        Ref: 'NetworkingVpcprivateSubnet2Subnet390F8998'
  NetworkingVpcIGW21218DAB:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: 'Name'
          Value: 'CdkSample/Networking/Vpc'
  NetworkingVpcVPCGW12E561D8:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      InternetGatewayId:
        Ref: 'NetworkingVpcIGW21218DAB'
      VpcId:
        Ref: 'NetworkingVpc6B5E6F44'
```

:::

CDK で書くと 25 行で済みます。\
どのようなリソースが作成されるのか一目でわかります。

```ts
import { Stack, StackProps } from 'aws-cdk-lib';
import { Construct } from 'constructs';
import { IpAddresses, SubnetType, Vpc } from 'aws-cdk-lib/aws-ec2';

export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Vpc(this, 'Vpc', {
      ipAddresses: IpAddresses.cidr('10.0.0.0/16'),
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: 'public',
          subnetType: SubnetType.PUBLIC,
        },
        {
          cidrMask: 24,
          name: 'private',
          subnetType: SubnetType.PRIVATE_ISOLATED,
        },
      ],
    });
  }
}
```

ただし、書かれているリソース以外にも Route Table や Internet Gateway など、色々なリソースが作られています。\
リソースを勝手に作られるのが嫌な場合は、指定することもできます。

### 9. Grants

AWS リソースから他の AWS リソースを呼び出す際は、サービスが利用する IAM Role に権限を付与する必要があります。\
L2 Construct には権限設定を簡単に行える「Grants」を利用できます。

次の例では、Lambda Function から S3 Bucket に対して読み取り権限を付与しています。\

```ts
const bucket = new Bucket(this, 'Bucket');

const func = new NodejsFunction(this, 'Function', {
  entry: path.join(__dirname, '../lambda/index.ts'),
  handler: 'handler',
  runtime: Runtime.NODEJS_22_X,
});

// S3 Bucketに対してLambda Functionの読み取り権限を付与
bucket.grants.read(func);
```

作成された IAM ロールには、S3 Bucket に対して読み取り権限が付与されています。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": ["s3:GetBucket*", "s3:GetObject*", "s3:List*"],
      "Resource": [
        "arn:aws:s3:::cdksample-bucketXXXXXX",
        "arn:aws:s3:::cdksample-bucketXXXXXX/*"
      ],
      "Effect": "Allow"
    }
  ]
}
```

### 10. バリデーション

L2 Construct ではデプロイ時に失敗するコードを検証する仕組みを持っています。\
CloudFormation テンプレートの合成時にバリデーションコードが実行されるため、デプロイ時だけではなくスナップショットテストの実行時や Synthesize 実行時にも検証できます。

例えば SQS Queue を作成するときの例を見てみます。

FIFO Queue の名前を指定するときはリソース名のサフィックスに`.fifo` と付ける必要があります。\
CDK の L2 Construct 「Queue」を利用すれば、テンプレートを構成したタイミングでエラーに気づけます。

```ts
export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Queue(this, 'FifoQueue', {
      // FIFO Queueの名前はサフィックスに `.fifo`とつける必要がある
      queueName: 'my-fifo-queue',
      fifo: true,
    });
  }
}
```

このコードを synthesize すると Validation エラーが発生します。

```shell
% npx cdk synth
/Users/xxx/cdk-sample/lib/akikii-cdk-sample-stack.ts:9
    new Queue(this, 'FifoQueue', {
    ^
  ValidationError: FIFO queue names must end in '.fifo'
  ...
```

自己レビューの段階で気づけるため手戻りが少なくなります。\
デプロイのタイミングで気づくことを考えると恐ろしいですね...

### 11. Aspects

Aspects を利用することで複数の AWS リソースに対して操作を適用できます。

CDK は App クラスをルートとして Construct tree を形成します。

![alt text](/images/cdks-recommended-points/construct-tree.dio.png)

https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/apps.html#apps-tree

Construct tree の特定のスコープ対して Aspects を適用すると、そのスコープ内のリソース全てに操作を適用できます。\
スコープの中でも特定のリソースタイプのみに適用することも可能です。

![alt text](/images/cdks-recommended-points/aspects.dio.png)

主なユースケースは次の通りです。

- Tags クラスを利用してリソースにタグを設定する(プロジェクト名や環境名など)
- RemovalPolicies クラスを利用してリソースに削除ポリシーを設定する(開発環境は削除保護しないなど)
- リソースが会社のポリシーに適応しているか検証する(全ての S3 Bucket がバージョニングを適用しているか？など)

### 12. Snapshot test

CDK の単体テスト手法のひとつです。\
CloudFormation テンプレートを合成して以前のテンプレートと比較したときに差分があるかをテストします。\
リソースやプロパティを変更した場合はもちろん差分が出るので、これが適切であると判断すれば Snapshot test を更新します。\
リファクタリング時には差分が出ないことを確認できます。

デフォルトの S3 Bucket に対して事前にスナップショットを取得した上でスナップショットテストを実施します。

```ts
export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new Bucket(this, 'Bucket');
  }
}
```

変更差分がないためテストが成功しています。

```shell
> vitest

...

 ✓ test/cdk-sample.test.ts (1 test) 119ms
   ✓ Snapshot test 119ms

 Test Files  1 passed (1)
      Tests  1 passed (1)
   ...
```

S3 Bucket のバージョニングを有効にします。

```diff ts
export class CdkSampleStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

-   new Bucket(this, 'Bucket');
+   new Bucket(this, 'Bucket', {
+     versioned: true,
+   });
  }
}
```

スナップショットテストを実行するとテストが失敗し、変更点が表示されます。\
想定通りバージョニングが有効になっています。

```diff shell
> Snapshot test
Error: Snapshot `Snapshot test 1` mismatched

- Expected
+ Received

@@ -7,10 +7,15 @@
      },
    },
    "Resources": {
      "BucketHASH_REPLACED": {
        "DeletionPolicy": "Retain",
+       "Properties": {
+         "VersioningConfiguration": {
+           "Status": "Enabled",
+         },
+       },
        "Type": "AWS::S3::Bucket",
        "UpdateReplacePolicy": "Retain",
      },
    },
    "Rules": {

 ❯ test/akikii-cdk-sample.test.ts:21:20
     19|
     20|   expect.addSnapshotSerializer(serializer);
     21|   expect(template).toMatchSnapshot();
       |                    ^
     22| });
     23|

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


  Snapshots  1 failed
 Test Files  1 failed (1)
      Tests  1 failed (1)
   Start at  17:37:19
   Duration  299ms (transform 18ms, setup 0ms, collect 158ms, tests 85ms, environment 0ms, prepare 2ms)

 FAIL  Tests failed. Watching for file changes...
```

### 13. Fine-grained Assertion test

CDK の単体テスト手法のひとつです。
合成した CloudFormation テンプレートのリソースやプロパティが正しいかをテストします。\
ただし、設定した全てのリソースおよびプロパティに対して確認するのは現実的ではなく、二重管理のようになってしまいます。\
そのため、特に保証したいプロパティやプログラミング言語の特性を活かした箇所などに限定してテストするとよいでしょう。

こちらの記事では CDK の単体テストについて、実例を交えて詳しく解説されています。\
https://aws.amazon.com/jp/builders-flash/202411/learn-cdk-unit-test/

## まとめ

AWS CDKの推しポイントを紹介しました。\
紹介したポイントに共感したり初めて知った機能を見つけて使いたくなったりしたのではないでしょうか？

この記事で特に紹介したかったのはL2 Constructの抽象化技術についてです。\
CDKの肝とも言える抽象化を利用すればインフラのデプロイを爆速かつ安全にできます！\
僕はそこに魅力を感じてCDKを使い続けるのです…！
