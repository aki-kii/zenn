---
title: 'cdkd（CDK Direct）でAWSリソースの爆速デプロイを試してみました'
emoji: '🏎️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'cdk', 'iac', 'oss']
published: true
---

CDKデプロイツールの`cdkd`（CDK Direct）がリリースされました。

`cdkd`はAWS DevTools Heroで、AWS CDKのCommunity Reviewer・Top Contributorとして活躍されている後藤さん([@365_step_tech](https://x.com/365_step_tech))により開発されたOSSです。

https://x.com/365_step_tech/status/2041857536337572173

GitHubリポジトリはこちらです。

https://github.com/go-to-k/cdkd

CDKアプリのソースコードそのままで爆速デプロイできるらしいです。速さは正義！！！

https://x.com/365_step_tech/status/2046919133628170259?s=20

ただし、`cdkd`はあくまでも実験的/教育的に作成されているプロジェクトなので、本番利用には適さないとREADMEに書かれています。
自己責任で利用しましょうとのことです。

> ⚠️ WARNING: NOT PRODUCTION READY
>
> This project is in early development and is NOT suitable for production use. Features are incomplete, APIs may change without notice, and there may be bugs that could affect your AWS infrastructure. Use at your own risk in development/testing environments only.
>
> Note: This is an experimental/educational project exploring alternative deployment approaches for AWS CDK. It is not intended to replace the official AWS CDK CLI, but rather to experiment with direct SDK provisioning as a learning exercise and proof of concept.

## セットアップ

`cdkd`はnpmパッケージとして公開されています。
ここではpnpmプロジェクトにインストールして利用します。

:::message
本記事の動作確認には`@go-to-k/cdkd@0.0.3`を使用しています。

一部の検証は`@go-to-k/cdkd@0.3.1`で追記しています。
:::

```sh
> pnpm add -D @go-to-k/cdkd
```

READMEのQuick Startセクションを見ると、AWS CDKと同様にbootstrapが必要とのことです。
`cdkd bootstrap`のヘルプを見ると、状態管理のためにS3バケットが作られるみたいです。

```shell
> pnpm cdkd bootstrap --help
Usage: cdkd bootstrap [options]

Bootstrap cdkd by creating required S3 bucket for state management
...
```

AWS CDKではCloudFormationのスタックで状態を管理していますが、cdkdでは作成したS3バケットで状態を管理しているのですね。

東京リージョンを指定して`cdkd bootstrap` します。

```shell
> pnpm cdkd bootstrap --region ap-northeast-1
Starting cdkd bootstrap...
No --state-bucket specified, resolving default bucket name...
Using default state bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Creating S3 bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1 in region ap-northeast-1
✓ Created S3 bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
✓ Enabled bucket versioning
✓ Enabled bucket encryption (AES-256)
✓ Set bucket policy (deny external access)

✓ Bootstrap completed successfully

State bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Region: ap-northeast-1

You can now use cdkd deploy with:
  --state-bucket cdkd-state-xxxxxxxxxxxx-ap-northeast-1
  --region ap-northeast-1
```

`cdkd-state-*`から始まる名前のS3 バケットが作成されました。

```shell
> aws s3 ls --bucket-name-prefix cdkd
2026-04-23 23:02:22 cdkd-state-xxxxxxxxxxxx-ap-northeast-1
```

## デプロイしてみた

cdkdはAWS CDKより速くデプロイできるとのことです。
新しくスタックを作成するときのデプロイ時間を比較してみます。

簡単なリソースとしてLambda関数を [NodejsFunction](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_lambda_nodejs.NodejsFunction.html)からデプロイします。

```ts
export class SandboxStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, '../src/lambda/index.ts'),
    });
  }
}
```

### cdk deploy

まずはAWS CDKでデプロイします。

```shell
> time pnpm cdk deploy --yes
Bundling asset Sandbox/Function/Code/Stage...

  ...3e80afdfb2c233cf01ad735d31ff97ba2ffbe-building/index.js  1.3kb

⚡ Done in 9ms

✨  Synthesis time: 2.25s

Sandbox: start: Building Sandbox Template
Sandbox: success: Built Sandbox Template
Sandbox: start: Publishing Sandbox Template (current_account-current_region-9dcc36f6)
Sandbox: success: Published Sandbox Template (current_account-current_region-9dcc36f6)
Sandbox: deploying... [1/1]
Sandbox: creating CloudFormation changeset...

 ✅  Sandbox

✨  Deployment time: 37.2s

Stack ARN:
arn:aws:cloudformation:ap-northeast-1:xxxxxxxxxxxx:stack/Sandbox/004363f0-36d0-11f1-b0df-0e4076ab47bd

✨  Total time: 39.45s

pnpm cdk deploy --yes  4.62s user 0.69s system 12% cpu 41.831 total
```

CDK CLIではデプロイ中のリソースが表示されてデプロイ完了した時点で非表示となります。
デプロイのトータルは42秒ほどでした。

### cdkd deploy

続いては cdkd でデプロイします。

```shell
> time pnpm cdkd deploy --region ap-northeast-1
State bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Synthesizing CDK app...
Bundling asset Sandbox/Function/Code/Stage...
...91df574c855f191fd6b3ae134856076f0348e71c7c60fe93-building/index.js  1.3kb

⚡ Done in 7ms

Deploying stack: Sandbox
Changes: 3 to create, 0 to update, 0 to delete
Level 1/3 (1 resources)
[1/3] ✅ FunctionServiceRole675BB04A (AWS::IAM::Role) created
Level 2/3 (1 resources)
[2/3] ✅ Function76856677 (AWS::Lambda::Function) created
Level 3/3 (1 resources)
[3/3] ✅ FunctionLogGroup55B80E27 (AWS::Logs::LogGroup) created

Deployment Summary:
  Stack: Sandbox
  Created: 3
  Updated: 0
  Deleted: 0
  Unchanged: 0
  Duration: 13.17s

✓ Deployment completed successfully
pnpm cdkd deploy --region ap-northeast-1  5.31s user 0.99s system 33% cpu 19.062 total
```

デプロイが完了したリソースがリアルタイムで積み上がっていきます。
かかった時間は19秒程度と、CDKデプロイの半分以下の時間でデプロイできています！爆速ですね！！！

## スタックを削除する

作成したスタックを削除します。
`cdk destroy` コマンドはCloudFormationスタックを削除する動作ですが、`cdkd destroy`ではデプロイと同様にAWS SDKを利用して削除するのでしょう。

時間を計測しているので、インタラクティブなやり取りをスキップする `--force`オプションをつけて`destroy`コマンドを実行します。

```shell
> time pnpm cdkd destroy --force --region ap-northeast-1
State bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Starting stack destruction...
Bundling asset Sandbox/Function/Code/Stage...
...91df574c855f191fd6b3ae134856076f0348e71c7c60fe93-building/index.js  1.3kb

⚡ Done in 6ms
Found 1 stack(s) to destroy: Sandbox

Preparing to destroy stack: Sandbox

Resources to be deleted (3):
  - FunctionServiceRole675BB04A (AWS::IAM::Role)
  - Function76856677 (AWS::Lambda::Function)
  - FunctionLogGroup55B80E27 (AWS::Logs::LogGroup)

Acquiring lock for stack Sandbox...
Building dependency graph...
  ✅ FunctionLogGroup55B80E27 (AWS::Logs::LogGroup) deleted
  ✅ Function76856677 (AWS::Lambda::Function) deleted
  ✅ FunctionServiceRole675BB04A (AWS::IAM::Role) deleted

✓ Stack Sandbox destroyed (3 deleted, 0 errors)
pnpm cdkd destroy --force --region ap-northeast-1  3.53s user 0.60s system 77% cpu 5.318 total
```

5秒ちょっとで削除できました。リソースの削除も爆速です！

## 少し複雑なスタックのデプロイ時間を比較してみる

静的Webサイトホスティングとバックエンドを持つWebアプリケーションのスタックがあったので、デプロイ時間を比較してみます。

スタックの構成は以下の通りです。

- CloudFront + S3（静的Webサイトホスティング）
- API Gateway + Lambda（API）
- DynamoDB
- Custom Resource（DynamoDBへ初期データを投入）

AWS CDKでデプロイします。

```shell
> time pnpm cdk deploy --yes
...
pnpm cdk deploy --yes 9.01s user 1.19s system 3% cpu 5:01.28 total
```

cdkdでデプロイします。

```shell
> time pnpm cdkd deploy --region ap-northeast-1
...
pnpm cdkd deploy --region ap-northeast-1 5.40s user 0.98s system 2% cpu 4:24.03 total
```

~~5分1秒 → 4分24秒 と、デプロイ時間が40秒近くも縮まりました！~~

:::message
2026/04/29 追記

cdkdのデプロイ速度が向上されたとのことでv0.3.1で再度試しました。同じスタックで試したところ、3分44秒 でした。AWS CDKと比較してなんと1分20秒近くの短縮です！

https://x.com/365_step_tech/status/2049191934837866666?s=20

```shell
> time pnpm exec cdkd deploy --yes --region ap-northeast-1
State bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Synthesizing CDK app...
...

pnpm exec cdkd deploy --yes --region ap-northeast-1  5.00s user 0.88s system 2% cpu 3:44.73 total
```

:::

AWS CDKはデプロイ時間が長いのがネックであるため、時間が短縮できると嬉しいですね。

## `--no-wait`オプションをつけてデプロイしてみる

`cdkd`デプロイにはリソースを非同期で作成する`--no-wait`オプションがあります。

> This can significantly speed up deployments with CloudFront (which takes 3-15 minutes to deploy to edge locations). The resource is fully functional once AWS finishes the async deployment.

CloudFrontやRDSなどの作成に時間がかかるリソースを待たずにデプロイできるとのことです。

先の静的Webサイトホスティング+バックエンドのスタックではちょうどCloudFrontをデプロイしているので長い待ち時間の短縮が望めそうです。

ではデプロイします。

```shell
> time pnpm cdkd deploy --no-wait --region ap-northeast-1
State bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Synthesizing CDK app...

...
pnpm cdkd deploy --no-wait --region ap-northeast-1  4.72s user 0.83s system 5% cpu 1:36.86 total
```

~~1分36秒 ...！？~~
~~CDKデプロイからは3分25秒、`--no-wait`オプションをつけない状態からは2分48秒も縮まっています...！~~

:::message
2026/04/29 追記

こちらも`v0.3.1`で試したところ、1分22秒でデプロイできました。AWS CDKからは3分40秒ほど、`--no-wait`オプションをつけない状態からは2分20秒も縮まっています...！

```shell
> time pnpm exec cdkd deploy --no-wait --yes --region ap-northeast-1
State bucket: cdkd-state-xxxxxxxxxxxx-ap-northeast-1
Synthesizing CDK app...
...

pnpm exec cdkd deploy --no-wait --yes --region ap-northeast-1  4.64s user 0.82s system 6% cpu 1:22.69 total
```

:::

開発環境で少しの変更加えるために毎度5分ほど待つのはしんどかったので、非同期デプロイができる`--no-wait`オプションは重宝しそうですね。

## Stateの確認

cdkdの状態はbootstrapで作成したS3バケットで管理されているのでした。

READMEによるとスタック名のprefixにある`state.json`でリソースの状態が管理されているとのこと。

```txt
s3://{state-bucket}/
  └── {prefix}/                     # Default: "cdkd" (configurable via --state-prefix)
      ├── MyStack/
      │   ├── state.json            # Resource state
      │   └── lock.json             # Exclusive deploy lock
      └── AnotherStack/
          ├── state.json
          └── lock.json
```

> https://github.com/go-to-k/cdkd#state-management

デプロイが完了した今、どのようにリソースが管理されているのか、NodejsFunctionのスタックの`state.json`を見てみます。（スタックの削除前に検証しています）

```json
{
  "version": 1,
  "region": "ap-northeast-1",
  "stackName": "Sandbox",
  "resources": {
    "FunctionServiceRole675BB04A": {
      "physicalId": "Sandbox-FunctionServiceRole675BB04A",
      "resourceType": "AWS::IAM::Role",
      "properties": {
        "AssumeRolePolicyDocument": {
          "Statement": [
            {
              "Action": "sts:AssumeRole",
              "Effect": "Allow",
              "Principal": {
                "Service": "lambda.amazonaws.com"
              }
            }
          ],
          "Version": "2012-10-17"
        },
        "ManagedPolicyArns": [
          "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
        ]
      },
      "attributes": {
        "Arn": "arn:aws:iam::xxxxxxxxxxxx:role/Sandbox-FunctionServiceRole675BB04A",
        "RoleId": "AROAUVXGUSUOWHC64J3GP"
      }
    },
    "Function76856677": {
      "physicalId": "Sandbox-Function76856677",
      "resourceType": "AWS::Lambda::Function",
      "properties": {
        "Code": {
          "S3Bucket": "cdk-hnb659fds-assets-xxxxxxxxxxxx-ap-northeast-1",
          "S3Key": "a9a27596ac4e797a73c3c4f8622945f2a04afe3b6b651901f543fbec2cc9dd5a.zip"
        },
        "Handler": "index.handler",
        "Role": "arn:aws:iam::xxxxxxxxxxxx:role/Sandbox-FunctionServiceRole675BB04A",
        "Runtime": "nodejs22.x"
      },
      "attributes": {
        "Arn": "arn:aws:lambda:ap-northeast-1:xxxxxxxxxxxx:function:Sandbox-Function76856677",
        "FunctionName": "Sandbox-Function76856677"
      },
      "dependencies": ["FunctionServiceRole675BB04A"]
    },
    "FunctionLogGroup55B80E27": {
      "physicalId": "/aws/lambda/Sandbox-Function76856677",
      "resourceType": "AWS::Logs::LogGroup",
      "properties": {
        "LogGroupName": "/aws/lambda/Sandbox-Function76856677",
        "RetentionInDays": 731
      },
      "attributes": {
        "Arn": "arn:aws:logs:ap-northeast-1:xxxxxxxxxxxx:log-group:/aws/lambda/Sandbox-Function76856677:*"
      },
      "dependencies": ["Function76856677"]
    }
  },
  "outputs": {},
  "lastModified": 1776955631391
}
```

見た感じ、ほぼCloudFormationのテンプレートのように見えます。

アセットファイルがどこに管理されているのか疑問に感じたので、Lambda関数のCodeプロパティを見ると、AWS CDK（≠cdkd）のbootstrapで作られたアセットを配置するS3バケットに保存されていました！

```json
"Function76856677": {
  "resourceType": "AWS::Lambda::Function",
  "properties": {
    "Code": {
        "S3Bucket": "cdk-hnb659fds-assets-xxxxxxxxxxxx-ap-northeast-1",
        "S3Key": "a9a27596ac4e797a73c3c4f8622945f2a04afe3b6b651901f543fbec2cc9dd5a.zip"
    },
    ...
},
```

実はcdkdドキュメントのPrerequisitesにも事前にAWS CDKのbootstrapをしておく必要があると書いてあります。

> AWS CDK Bootstrap: You must run cdk bootstrap before using cdkd. cdkd uses CDK's bootstrap bucket (cdk-hnb659fds-assets-\*) for asset uploads (Lambda code, Docker images).
>
> https://github.com/go-to-k/cdkd#prerequisites

うまくAWS CDKに相乗りしてる...！

## まとめ

`cdkd`を利用してリソースのデプロイを試してみました。

AWS CDKは他のIaCツールであるTerraformと比較したときにデプロイに時間がかかるという特徴があります。
`cdkd`はCloudFormationを介さずAWS SDKでデプロイするため、素早くデプロイできる嬉しいツールです。

ただし、現時点の`cdkd`は実験的/教育的であり、後藤さんお一人で開発・メンテナンスされているツールです。
未対応のリソースも存在するとのことです。

https://x.com/365_step_tech/status/2047237532665192570

そのため、今の`cdkd`をプロダクションに持ち込むのは難しいと思っています（開発環境限定で利用するのはありかも）

今後`cdkd`がどのように舵を切っていくかはわかりませんが、個人的には`cdk8s`や`CDK Terrain`（`CDKTF`）のようにAWS CDKの兄弟のような立ち位置になっていくと嬉しいなと思います。
