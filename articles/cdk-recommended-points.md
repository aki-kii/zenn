---
title: 'AWS CDKの推しポイントN選'
emoji: '😻'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'cdk', 'iac']
published: false
---

AWS CDKはAWSのIaCサービスの1つで、AWSのリソースをプログラミング言語で定義して管理・デプロイを行うツールです。\
同じIaCサービスであるCloudFormationをラップすることによって抽象化しています。

そんなAWS CDKの推しポイントをカテゴリに分けて紹介します！

- IaC編
- IDE編
- コアコンセプト編
- エコシステム編

## IaC 編

AWS リソースをデプロイする方法は他にもいろいろあります。\
例えば、AWS コンソールからデプロイしたり、AWS CLI を利用してコマンドからデプロイ、SDK を使ってプログラムからデプロイしたりできます。

![alt text](/images/cdks-recommended-points/make-aws-resource.dio.png)

IaC では、コードで作成した定義ファイルと現在の状態の差分によってデプロイが行われます。

TODO: IaCのイメージ

AWS CDK が IaC であることによって嬉しいポイントをあげていきます。

### 定義ファイルの変更管理ができる

AWS CDK の定義ファイルはプログラミング言語で書かれているため、Git などのバージョン管理システムを用いることで変更管理が容易に行えます。\
リソースの変更差分を残せる上、最新の状態が一目瞭然です。

![alt text](/images/cdks-recommended-points/change-management-for-git.png)

### リソースの設定を集約できる

AWS コンソールでリソースの関係性を見ようと思った時、迷ったことはありませんか？\
例えば、S3バケットからイベント通知でLambda関数が起動しDynamoDBテーブルに書き込む構成の関連性を調査しようとした時、次のように調べることとなると思います。

1. S3バケット/プロパティ/イベント通知 → Lambda関数
2. Lambda関数/コード/ソースコード or Lambda関数/設定/アクセス権限　→ DynamoDBテーブル

TODO: 画像化する

CDKでは、リソースの設定が Git リポジトリに集約されているので関係性の把握がしやすいです。

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
IaCではインフラリソースをコードで定義するため、そのコードを用いて複数の環境にデプロイできます。

CDKでは「デプロイするAWSアカウントを分ける」「スタック名に環境名を含める」ことで実現可能です。\
基本的にはどちらも取り入れることをお勧めします。\
(「スタック名に環境名を含める」理由は、リソース名を自動生成する場合に環境名が含まれるため環境ごとに異なるリソース名になるためです)

設定ファイルを用意することで環境ごとに異なるプロパティを設定できます。\
例えば、開発環境ではLambdaのメモリを512MB、本番環境では2048MBなどの異なる設定にできます。

## IDE編

Visual Studio Code のようなIDE(統合開発環境)では、ソフトウェア開発をサポートする機能が多く組み込まれています。\
AWS CDK は TypeScript や Python などのプログラミング言語で書くため、IDE のサポートを受けられます。

CDK で実装する時に嬉しい IDE のサポート機能について、次のコードをベースに紹介します。\
可視性タイムアウトを30秒に設定したSQSキューを作成しています。

```ts
import { Duration } from 'aws-cdk-lib'
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class MyConstruct extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new Queue(this, 'Queue', {
        visivilityTimeout: Duration.seconds(30),
      }
    );
  }
}
```

### コード補完が効く

コードの予測をしてくれる機能です。\
途中まで入力したら予測を出してくれます。

![alt text](/images/cdks-recommended-points/code-completion-suggest.png)

階層構造になっているプロパティをリストアップしてくれます。

![alt text](/images/cdks-recommended-points/code-completion-list.png)

コード補完を利用すればtypoも減り、どのようなプロパティがあるのかをドキュメントなしで探せます。

### ドキュメントがすぐ読める

プロパティに設定する値を確認する時、ドキュメントを読むと思います。\
IDE 上でオブジェクトにホバーすれば、実装に記載されているドキュメントを見れます。

![alt text](/images/cdks-recommended-points/documentation.png)

ブラウザでドキュメントを開かずとも説明が読めるので時間を短縮できます。

### 構文エラーがわかる

CDKの構文を間違えた時に、構文エラーとしてエラー内容が表示されます。

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

CDKはCloudFormationを抽象化したラッパーツールです。\
抽象化するにあたって実装された概念がコアコンセプトとしてデベロッパーガイドに記載されています。\
https://docs.aws.amazon.com/ja_jp/cdk/v2/guide/core-concepts.html

コアコンセプトに関する推しポイントをあげていきます。

### L2 Constructの抽象化

L2 ConstructはCloudFormationテンプレートのプロパティを利用者にとって扱いやすいように抽象化しています。\
この抽象化技術によりIaCを用いた開発を簡易化でき、コードの可読性も高まります。

L2 で行われている抽象化として、適切なデフォルトプロパティやベストプラクティスに沿ったセキュリティポリシー・関連リソースの作成の他、リソースの特性に沿ったヘルパーメソッドが用意されています。\
抽象化によってリソースの設定がブラックボックスになるわけではなく、L2 Constructにプロパティを渡すことでCloudFormationと変わらない設定もできます。

Amazon VPCを作成するL2 Constructの実装例を見てみます。

TODO: 例を作成

### Grantsで簡単に権限を付与

AWSリソースから他のAWSリソースを呼び出す際は、サービスが利用するIAM Roleに権限を付与する必要があります。\
L2 Constructには権限設定を簡単に行える「Grants」を利用できます。

Lambda FunctionからS3 Bucketに対して読み取り権限を付与する例を見てみます。

### バリデーションで設定ミスを事前に検査できる



### テストで設定ミスを事前に検査できる
