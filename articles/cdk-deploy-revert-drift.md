---
title: 'AWS CDKのドリフトを元に戻す--revert-driftオプションがcdk deployに追加されました'
emoji: '🎣'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'cdk', 'cloudformation']
published: true
---

AWS CDKでデプロイしたAWSリソースにドリフトが発生した時、元に戻すのが大変だった経験はありませんか？
`aws-cdk@v2.1110.0` から`cdk deploy`コマンドに`--revert-drift`オプションが追加されました。
このオプションを指定することで、CDKでデプロイしたリソースにドリフトが発生していた場合でも、合成したテンプレートに合わせてドリフトを解消してくれるようになります！

この機能はCloudFormationで既に実装されている「ドリフト対応変更セット」をCDKでも扱えるようにしたものです。

https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html

## ドリフトとは

AWS CDKはプログラミング言語でAWSリソースを構築するツールです。
ドリフトはCDKで定義したAWSリソースがCDKのソースコードとズレてしまっている状態を指します。
もう少し詳しく説明すると、CDKデプロイによって作成されたCloudFormationスタック管理下のAWSリソースと実際のAWSリソースの設定がズレている状態です。

よくある原因としては、AWSコンソールからリソースの設定を手動で行うことで発生します。
急いでいたから・ソースコードを変更するのが面倒だったから...など発生してしまう理由は様々ですが、ドリフトを発生させないのがベストです。
CloudFormationのベストプラクティスにも記載があります。

> Don't make changes to stack resources outside of CloudFormation. Doing so can create a mismatch between your stack's template and the current state of your stack resources, which can cause errors if you update or delete the stack.
>
> https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html#donttouch

## CDKデプロイでドリフトを元に戻す方法

CDKデプロイではCDKのソースコードとデプロイ済みのCloudFormationスタックとの差分を変更セットとして抽出し、その変更部分のみデプロイします。（厳密には`--method`オプションがデフォルトの場合）
CloudFormationスタックの状態はAWSリソースの設定が直接変更されても変わらないため、CDKのソースコードとCloudFormationスタックの間に差分が発生しません。
差分が検出されないことには変更セットが作成されず、デプロイも行われません。

これまでCDKのコード定義の状態にAWSリソースを戻すには、次のいずれかの方法を取る必要がありました。

1. AWSコンソールなどからドリフトが発生しているリソースを直接修正する
2. CDKのソースコードを一時的に書き換えてデプロイし、その後元の状態に戻して再デプロイする

どちらも面倒な上に、設定を間違えてしまう可能性があるため安全ではありません。
そこで`--revert-drift`オプションを利用することで、ドリフトが発生したリソースを簡単かつ安全に元の設定に戻すことができるようになりました。

## 使ってみた

次のユースケースにて`--revert-drift`を利用してドリフトを元に戻してみました。

1. 意図的に一時的なドリフトを発生させる
2. ドリフトを自動で元に戻すスケジュールジョブ

### 意図的に一時的なドリフトを発生させる

CloudFormationではドリフトを発生させないのがベストプラクティスでした。

しかし、一時的にドリフトを発生させたいケースはたまにあります。
値を変更してどうなるか試したい、設定を今だけ無効化したい、スケジュールを今すぐ起動したい...などなど

今回はEventBridgeスケジューラのスケジュールを変更して即時実行するケースを試します。
Lambda関数とEventBridgeスケジューラを実装するスタックをデプロイします。

```ts:lib/sandbox-stack.ts
export class SandboxStack extends Stack {
  constructor(scope: Construct, id: string, props?: StackProps) {
    super(scope, id, props);

    const func = new NodejsFunction(this, 'Function', {
      entry: path.join(__dirname, '../src/lambda/index.ts'),
    });
    new Schedule(this, 'Schedule', {
      target: new LambdaInvoke(func),
      schedule: ScheduleExpression.cron({
        hour: '14',
        minute: '00',
        weekDay: '2-6',
        timeZone: TimeZone.ASIA_TOKYO,
      }),
    });
  }
}
```

平日の14時にLambda関数をスケジュール実行するEventBridge Schedulerが作成されました。
![alt text](images/cdk-deploy-revert-drift/eventbridge-scheduelr.png)

このままAWSコンソールから1回限りのスケジュールに変更します。
![alt text](images/cdk-deploy-revert-drift/change-once-schedule.png)

余談ですが、1回限りのスケジュールで過去の時刻に設定した場合はスケジュールがすぐ実行されて便利です。

スケジューラが起動してLambda関数が実行されました。
![alt text](images/cdk-deploy-revert-drift/invoked-lambda-function.png)

満足したのでSchedulerの設定を元のcron式に戻したくなってきました。

まずはドリフトが発生しているか確認するため`cdk drift`コマンドを実行してドリフト検出すると、`ScheduleExpression` プロパティにドリフトが発生していました。

```shell
> cdk drift

✨  Synthesis time: 1.89s
Stack Sandbox
Modified Resources
[~] AWS::Scheduler::Schedule Schedule Schedule83A77FD1
 └─ [~] /ScheduleExpression
     ├─ [-] cron(00 14 ? * 2-6 *)
     └─ [+] at(2026-01-01T00:00:00)

1 resource has drifted from their expected configuration

✨  Number of resources with drift: 1 (1 unchecked)
```

一時的な変更なので元に戻したいですが、コンソールから元に戻そうとするとcron式を指定しなきゃいけなくて面倒です。
そのまま`cdk deploy`しようとしても変更セットに差分がない状態なのでデプロイできません。

そんな時、`cdk deploy`コマンドに`--revert-drift`オプションを指定すれば、変更を元の状態に戻せます！
さっそく実行してみると、変更セットに差分が見つかりデプロイが実行されました。

```shell
> cdk deploy --revert-drift

Bundling asset Sandbox/Function/Code/Stage...

  cdk.out/bundling-temp-f2312d8dc7d4d13bb90f79dc2593e80afdfb2c233cf01ad735d31ff97ba2ffbe-building/index.js  1.3kb

⚡ Done in 9ms

✨  Synthesis time: 2.6s

Sandbox: deploying... [1/1]
Sandbox: creating CloudFormation changeset...

 ✅  Sandbox

✨  Deployment time: 28.1s

Stack ARN:
arn:aws:cloudformation:ap-northeast-1:xxxxxxxxxxxx:stack/Sandbox/004363f0-36d0-11f1-b0df-0e4076ab47bd

✨  Total time: 30.7s
```

AWSコンソールからEventBridgeスケジューラ を確認するとCDKで設定した値に戻っていました。
![alt text](images/cdk-deploy-revert-drift/eventbridge-scheduelr.png)

今回はEventBridgeスケジューラの例を紹介しましたが、他にもLambda関数の環境変数やIAMポリシーの追加など、一時的にリソースを変更したくなるケースはたくさんありそうです！

### ドリフトを自動で元に戻すスケジュールジョブ

ドリフトを一時的に発生させて戻す...たまに忘れそうですよね。
しかも自分一人だけじゃなくチームで開発しているとそのリスクも人の人数だけ増えます。

そこで、ドリフトを放置せず自動で元に戻すスケジュールジョブをGitHub Actionsで作ってみました。

```yaml:.github/workflow/revert-drift.yml
name: Revert Drift Job

on:
  schedule:
    - cron: '0 12 * * 1-5' # 平日 JST 21:00
  workflow_dispatch:

permissions:
  contents: read
  id-token: write

defaults:
  run:
    shell: bash

jobs:
  run:
    runs-on: ubuntu-latest
    timeout-minutes: 30

    strategy:
      matrix:
        include:
          - branch: main
            role_secret: AWS_ROLE_ARN

    steps:
      - uses: actions/checkout@v6
        with:
          ref: ${{ matrix.branch }}

      - uses: pnpm/action-setup@v5

      - uses: actions/setup-node@v6
        with:
          node-version: '24'
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: ${{ secrets[matrix.role_secret] }}
          aws-region: ap-northeast-1

      # ドリフトが発生してた時だけデプロイを実行
      - run: pnpm exec cdk drift --fail || pnpm exec cdk deploy --revert-drift --yes
```

実行するためにはAWSアカウントへアクセスするための権限が必要です。
自分はGitHubのOIDCプロバイダーを利用して権限を委譲しました。

https://github.com/aws-actions/configure-aws-credentials#oidc-configuration-details

もう一度EventBridgeスケジューラの設定にドリフトを起こしてワークフローを実行しました。
`Run pnpm exec cdk drift --fail || pnpm exec cdk deploy --revert-drift --yes`セクションの結果にて、ドリフトの差分が表示された後にデプロイが行われています。

```shell
Run pnpm exec cdk drift --fail || pnpm exec cdk deploy --revert-drift --yes
✨  Synthesis time: 10.11s
Stack Sandbox
Modified Resources
[~] AWS::Scheduler::Schedule Schedule Schedule83A77FD1
 └─ [~] /ScheduleExpression
     ├─ [-] cron(00 14 ? * 2-6 *)
     └─ [+] at(2026-04-01T09:09:00)
1 resource has drifted from their expected configuration
✨  Number of resources with drift: 1 (1 unchecked)
Bundling asset Sandbox/Function/Code/Stage...

  ...9906fc59df09d8bae33c0a04f1f1b43039d9f56b287f50ba-building/index.js  1.3kb

⚡ Done in 3ms

✨  Synthesis time: 5.92s

Sandbox: deploying... [1/1]
Sandbox: creating CloudFormation changeset...
Sandbox | 0/3 | 7:02:04 AM | UPDATE_IN_PROGRESS   | AWS::CloudFormation::Stack | Sandbox User Initiated
Sandbox | 0/3 | 7:02:16 AM | UPDATE_IN_PROGRESS   | AWS::Scheduler::Schedule   | Schedule (Schedule83A77FD1)
Sandbox | 1/3 | 7:02:18 AM | UPDATE_COMPLETE      | AWS::Scheduler::Schedule   | Schedule (Schedule83A77FD1)
Sandbox | 2/3 | 7:02:19 AM | UPDATE_COMPLETE_CLEA | AWS::CloudFormation::Stack | Sandbox
Sandbox | 3/3 | 7:02:20 AM | UPDATE_COMPLETE      | AWS::CloudFormation::Stack | Sandbox

 ✅  Sandbox

✨  Deployment time: 36.52s

Stack ARN:
arn:aws:cloudformation:ap-northeast-1:xxxxxxxxxxxx:stack/Sandbox/004363f0-36d0-11f1-b0df-0e4076ab47bd

✨  Total time: 42.44s
```

## まとめ

CDKデプロイの`--revert-drift`オプションを試してみました。
これまではドリフトを元に戻すのに面倒な手順が必要だったものが簡単に戻せるようになりました。

試すために設定値を変更したい、一時的に設定値を変更したいケースはよくあると思うので、使う機会はそこそこあるんじゃないでしょうか？

実を言うとこのオプションは僕がコントリビュートしました...！
https://github.com/aws/aws-cdk-cli/releases/tag/aws-cdk%40v2.1110.0

便利な機能を追加できたと思うので、ぜひ使ってみてください！
