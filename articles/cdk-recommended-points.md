---
title: 'AWS CDKの推しポイントN選'
emoji: '😻'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'cdk', 'iac']
published: false
---

## AWS CDK とは

AWS CDK は、AWS の IaC サービスの一つです。\
IaC（Infrastructure as Code）とは、インフラをコードで定義してデプロイ・管理を行うためのツールです。\
AWS CDK では、AWS リソースをプログラミング言語で定義してデプロイ・管理を行うサービスです。

AWS CDK は、IaC サービスの AWS CluodFormation をプログラミング言語で利用できるよう拡張したサービスです。\
今回お話しする`推しポイント`には、CloudFormatoin のよさも含めます！

## IaC のよさ

AWS リソースをデプロイする方法は他にもいろいろあります。\
例えば、AWS コンソールから作成したり、AWS CLI を利用してコマンドから作成したり、SDK を使ってプログラムから作成したりできます。

![alt text](/images/cdks-recommended-points/make-aws-resource.dio.png)

IaC では、コードで作成した定義ファイルと現在の状態の差分によってデプロイが行われます。\
AWS CDK が IaC であることによって嬉しいポイントをあげていきます。

### 定義ファイルの変更管理ができる

AWS CDK の定義ファイルはプログラミング言語で書かれているため、Git などのバージョン管理システムを用いることで変更管理が容易に行えます。\
リソースの変更差分を残せる上、最新の状態が一目瞭然です。

![alt text](/images/cdks-recommended-points/change-management-for-git.png)

### リソースの設定を集約できる

AWS コンソールでリソースの関係性を見ようと思った時、迷ったことはありませんか？\
CDK では一つのリポジトリの中に AWS リソースの設定をまとめて定義するので、リポジトリを見れば全てがわかる状態となります。

### デプロイの手順が簡単になる

AWS コンソールや CLI などを用いて AWS リソースをデプロイするとき、手数が多かったり複雑な手順だった経験はありませんか？\
CDK では、1 つのデプロイ単位で複数の AWS リソースの設定を定義できるため、ワンアクションでデプロイが完了します。

AWS CDK からデプロイするコマンドの一例です。

```shell
npx cdk deploy
```

設定する箇所が少なくなるので、手作業によるミスも抑えられます。

## IDE（統合開発環境）のサポート

Visual Studio Code のようなアプリケーションを開発するための IDE（統合開発環境）には、開発をサポートする機能が多く組み込まれています。\
AWS CDK は TypeScript や Python などのプログラミング言語で書くため、IDE のサポートが受けられます。

ここでは、CDK で実装する時に嬉しい IDE のサポート機能について次のコードをベースに紹介します。

```ts
import { Queue } from 'aws-cdk-lib/aws-sqs';
import { Construct } from 'constructs';

export class MyConstruct extends Construct {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new Queue(this, 'Queue');
  }
}
```

これは SQS Queue を作成するための設定を CDK のデフォルト値で定義しているコードです。

### コード補完が効く

コードの予測をしてくれる機能です。\
途中まで入力したら予測を出してくれたり、階層構造になっているプロパティの一覧を出せたりもします。

![alt text](/images/cdks-recommended-points/code-completion-suggest.png)

![alt text](/images/cdks-recommended-points/code-completion-list.png)

コード補完を利用すれば、打ち間違いも少なくなるしどんなプロパティがあるのかをドキュメントを見ずに探せます。

### ドキュメントが読める

プロパティにどんな値を指定すればいいかわからない時、ドキュメントを見に行くと思います。\
IDE 上でオブジェクトにホバーすれば、ドキュメントが見れるのです。

![alt text](/images/cdks-recommended-points/documentation.png)

わざわざドキュメントに行かずとも説明が読めるので、書く時間を短縮できます。\
それでもわからなければ、実装まで読めば確実です。

### プロパティを型検査できる/構文エラーがわかる

プロパティに渡す値に`string`や`int`などとは違う独自の`型`を渡すことができるものがあります。\
例えば`Duration`型には、期間を指定できます。
次の例では "30 秒" を表しています。

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

デプロイする前に事前に気づけるので、安全性が増します。

## AWS CDK のコンセプト

### Construct

Construct は CDK の基本的な構成要素です

1 つ以上の AWS リソースとその設定が含まれています。
CDK アプリケーションでは、Consturct を組み合わせて構築します。

Construct は抽象度により 3 つのレベルに分類されます。

L1 Construct
CloudFormation で作成できる AWS リソース（以下、CFn リソース）とプロパティレベルで一致しており、抽象化はされません。

L2 Construct
利用者にとって扱いやすいように CFn リソースのプロパティが高度に抽象化されています。
L1 Construct では必要なプロパティを設定するだけでも迷ってしまいますが、L2 Construct では適切なデフォルトプロパティやセキュリティポリシーが設定されるため、

L2 Construct の中で CFn リソースへマッピングされます。

### Grant

### バリデーション

### テスト

---

### Construct とは

Construct は CDK の基本的な構成要素です

1 つ以上の AWS リソースとその設定が含まれています。
CDK アプリケーションでは、Consturct を組み合わせて構築します。

Construct は抽象度により 3 つのレベルに分類されます。

L1 Construct
CloudFormation で作成できる AWS リソース（以下、CFn リソース）とプロパティレベルで一致しており、抽象化はされません。

L2 Construct
利用者にとって扱いやすいように CFn リソースのプロパティが高度に抽象化されています。
L1 Construct では必要なプロパティを設定するだけでも迷ってしまいますが、L2 Construct では適切なデフォルトプロパティやセキュリティポリシーが設定されるため、

L2 Construct の中で CFn リソースへマッピングされます。

### プロパティが抽象化されている

### 関連するリソースを作成してくれる

自分で指定できることも書く

### 直感的な権限設定（grant メソッド）

### テストで検証

## プログラミング言語で書ける（IDE に特化したほうがいいかな？）

### 構文エラーの検出

### 型の恩恵を受けれる（TypeScript）

## 設定を早い段階で検証できる（プログラミング言語で書けるを分割？）

### 型や内部実装でバリデーションしてる

### cdk-nag

## Construct の構造化による恩恵

### 1 つのスタックにまとめやすい

### Construct Tree View でリソースを追いやすい

## 再利用性が高い

## メモ

### 構成

この仕組みについて説明するよ → その中で推しポイントはこれ！みたいな流れにするといいかも

### 内容

L2 Construct はすごいよ
AWS リソースの設定や関連するリソースが抽象化されているんだもん
少ない設定でいっぱい設定できて嬉しいなあ
けどもちろん細かい設定はオプションになっているだけで、自分でも設定できるよ
IAM Role の設定は自分たちで持っておきたい組織もあるよねぇ
他にも IAM Role へ直感的に権限設定できる Grantable メソッドや、SG を直感的に繋げる Connectable メソッドもあるよ
プログラミング言語で書けることによるメリットもあるよ
プロパティが IDE の補完機能で使えるよ
構文のエラーも検出できるし、TypeScript だと型の恩恵も得られるからプロパティを勘違いしづらいね
検証しやすいこともメリットだよ
バリデーションが入っているから入力時点で合わないプロパティはエラーになる
cdk-nag を利用すればセキュリティチェックも事前に通せるよ
Construct Tree view も便利
1 つのプロジェクトで利用しているインフラを階層構造に分けてみれるのがとてもいいね
テストも書けるよ
絶対に守りたい設定に書くのがおすすめ
他にもプログラミング言語で書くから作られるリソースが不安なところにも書けて安心できる
再利用性が高いのもいいね
この組み合わせでリソースを作るよって箇所が何ヶ所もあれば、Construct を自前でも実装することで実現できるよ
Construct でテストを実装していれば展開しても安心だね
