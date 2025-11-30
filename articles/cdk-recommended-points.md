---
title: 'AWS CDKの推しポイントN選'
emoji: '😻'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'cdk', 'iac']
published: false
---

AWS CDK は AWS の IaC サービスの 1 つで、AWS のリソースをプログラミング言語で定義して管理・デプロイを行うツールです。\
同じ IaC サービスである CloudFormation をラップすることによって抽象化しています。

そんな AWS CDK の推しポイントをカテゴリに分けて紹介します！

- IaC 編
- IDE 編
- コアコンセプト編
- エコシステム編

## IaC 編

AWS リソースをデプロイする方法は他にもいろいろあります。\
例えば、AWS コンソールからデプロイしたり、AWS CLI を利用してコマンドからデプロイ、SDK を使ってプログラムからデプロイしたりできます。

![alt text](/images/cdks-recommended-points/make-aws-resource.dio.png)

IaC では、コードで作成した定義ファイルと現在の状態の差分によってデプロイが行われます。

TODO: IaC のイメージ

AWS CDK が IaC であることによって嬉しいポイントをあげていきます。

### 定義ファイルの変更管理ができる

AWS CDK の定義ファイルはプログラミング言語で書かれているため、Git などのバージョン管理システムを用いることで変更管理が容易に行えます。\
リソースの変更差分を残せる上、最新の状態が一目瞭然です。

![alt text](/images/cdks-recommended-points/change-management-for-git.png)

### リソースの設定を集約できる

AWS コンソールでリソースの関係性を見ようと思った時、迷ったことはありませんか？\
例えば、S3 バケットからイベント通知で Lambda 関数が起動し DynamoDB テーブルに書き込む構成の関連性を調査しようとした時、次のように調べることとなると思います。

1. S3 バケット/プロパティ/イベント通知 → Lambda 関数
2. Lambda 関数/コード/ソースコード or Lambda 関数/設定/アクセス権限　 → DynamoDB テーブル

TODO: 画像化する

CDK では、リソースの設定が Git リポジトリに集約されているので関係性の把握がしやすいです。

### デプロイの安全性が向上する

AWS コンソールや CLI などを用いてデプロイする場合、リソースごとにデプロイして確認する必要があるため手順が複雑になります。\
手順が複雑な場合はクリックミスや設定の確認漏れが発生しやすくなります。

CDK では最小のデプロイ単位であるスタック単位でデプロイするため手順を簡単化できます。\
AWS CDK からデプロイするコマンドの一例です。

```shell
npx cdk deploy
```

また、デプロイ前にレビューすることで意図しない変更を未然に防ぎやすくなります。

### 環境の複製が容易

ソフトウェア開発では開発・検証・本番などの複数の環境を利用します。\
IaC ではインフラリソースをコードで定義するため、そのコードを用いて複数の環境にデプロイできます。

CDK では「デプロイする AWS アカウントを分ける」「スタック名に環境名を含める」ことで実現可能です。\
基本的にはどちらも取り入れることをお勧めします。\
(「スタック名に環境名を含める」理由は、リソース名を自動生成する場合に環境名が含まれるため環境ごとに異なるリソース名になるためです)

設定ファイルを用意することで環境ごとに異なるプロパティを設定できます。\
例えば、開発環境では Lambda のメモリを 512MB、本番環境では 2048MB などの異なる設定にできます。

## IDE 編

Visual Studio Code のような IDE(統合開発環境)では、ソフトウェア開発をサポートする機能が多く組み込まれています。\
AWS CDK は TypeScript や Python などのプログラミング言語で書くため、IDE のサポートを受けられます。

CDK で実装する時に嬉しい IDE のサポート機能について、次のコードをベースに紹介します。\
可視性タイムアウトを 30 秒に設定した SQS キューを作成しています。

```ts
import { Duration } from 'aws-cdk-lib';
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class MyConstruct extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new Queue(this, 'Queue', {
      visivilityTimeout: Duration.seconds(30),
    });
  }
}
```

### コード補完が効く

コードの予測をしてくれる機能です。\
途中まで入力したら予測を出してくれます。

![alt text](/images/cdks-recommended-points/code-completion-suggest.png)

階層構造になっているプロパティをリストアップしてくれます。

![alt text](/images/cdks-recommended-points/code-completion-list.png)

コード補完を利用すれば typo も減り、どのようなプロパティがあるのかをドキュメントなしで探せます。

### ドキュメントがすぐ読める

プロパティに設定する値を確認する時、ドキュメントを読むと思います。\
IDE 上でオブジェクトにホバーすれば、実装に記載されているドキュメントを見れます。

![alt text](/images/cdks-recommended-points/documentation.png)

ブラウザでドキュメントを開かずとも説明が読めるので時間を短縮できます。

### 構文エラーがわかる

CDK の構文を間違えた時に、構文エラーとしてエラー内容が表示されます。

構文エラーの中でも型エラーを紹介します。

プロパティにはプリミティブ型(`string`や`int`など)とは違う独自の`型`を渡すことができるものがあります。\
例えば`Duration`型には、期間を指定でき、次の例では "30 秒" を表しています。

```ts
import { Duration } from 'aws-cdk-lib';

Duration.seconds(30);
```

本来`Duration`型を渡さないといけないのに、`int`型を渡したとしましょう。\
そうするとエラーとなります。

```ts
import { Queue } from 'aws-cdk-lib/aws-sqs';

new Queue(this, 'Queue', {
  // Type 'number' is not assignable to type 'Duration'.
  visibilityTimeout: 30,
});
```

デプロイの事前に気づけるので、安全性が増します。

## AWS CDK のコアコンセプト

CDK は CloudFormation を抽象化したラッパーツールです。\
抽象化するにあたって実装された概念がコアコンセプトとしてデベロッパーガイドに記載されています。\
https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/core-concepts.html

コアコンセプトに関する推しポイントをあげていきます。

### L2 Construct の抽象化

L2 Construct は CloudFormation テンプレートのプロパティを利用者にとって扱いやすいように抽象化しています。\
この抽象化技術により IaC を用いた開発を簡易化でき、コードの可読性も高まります。

L2 で行われている抽象化として、適切なデフォルトプロパティやベストプラクティスに沿ったセキュリティポリシー・関連リソースの作成の他、リソースの特性に沿ったヘルパーメソッドが用意されています。\
抽象化によってリソースの設定がブラックボックスになるわけではなく、L2 Construct にプロパティを渡すことで CloudFormation と変わらない設定もできます。

Amazon VPC を作成する L2 Construct の実装例を見てみます。

TODO: 例を作成

### Grants で簡単に権限を付与

AWS リソースから他の AWS リソースを呼び出す際は、サービスが利用する IAM Role に権限を付与する必要があります。\
L2 Construct には権限設定を簡単に行える「Grants」を利用できます。

Lambda Function から S3 Bucket に対して読み取り権限を付与する例を見てみます。

TODO: 例を作成

### バリデーションでデータの正当性を検査できる

L2 Construct ではデプロイ時に失敗するコードを検証する仕組みを持っています。\
CloudFormation テンプレートの合成時にバリデーションコードか実行されるため、デプロイ時だけではなくスナップショットテストの実行時や Synthesize 実行時にも検証できます。

例えば SQS Queue を作成するときの例を見てみます。

FIFO Queue を作成するときはリソース名のサフィックスに.fifo と付ける必要があります。

CloudFormation でデプロイする時はデプロイ時にエラーが発生します。

CDK の L2 Construct 「Queue」を利用すれば、テンプレートを構成したタイミングでエラーに気づけます。

TODO: 例を作成

### Aspects

Aspects を利用することで複数の AWS リソースに対して操作を適用できます。

CDK は App クラスをルートとして Construct tree を形成します。

https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/apps.html#apps-tree

Construct tree の特定のスコープ対して Aspects を適用すると、そのスコープ内のリソース全てに操作を適用できます。\
スコープの中でも特定のリソースタイプのみに適用することも可能です。

主なユースケースは次の通りです。

- Tags クラスを利用してリソースにタグを設定する(プロジェクト名や環境名など)
- RemovalPolicies クラスを利用してリソースに削除ポリシーを設定する(開発環境は削除保護しないなど)
- リソースが会社のポリシーに適応しているか検証する(全ての S3 Bucket がバージョニングを適用しているか？など)

### Snapshot test

CDK の単体テスト手法のひとつです。\
CloudFormation テンプレートを合成して以前のテンプレートと比較したときに差分があるかをテストします。\
リソースやプロパティを変更した場合はもちろん差分が出るので、これが適切であると判断すれば Snapshot test を更新します。\
リファクタリング時には差分が出ないことを確認できます。

### Fine-grained Assertion test

CDK の単体テスト手法のひとつです。
合成した CloudFormation テンプレートのリソースやプロパティが正しいかをテストします。\
ただし、設定した全てのリソースおよびプロパティに対して確認するのは現実的ではなく、二重管理のようになってしまいます。\
そのため、特に保証したいプロパティやプログラミング言語の特性を活かした箇所などに限定してテストするとよいでしょう。

こちらの記事では CDK の単体テストについて、実例を交えて詳しく解説されています。\
https://aws.amazon.com/jp/builders-flash/202411/learn-cdk-unit-test/

### Integ test
