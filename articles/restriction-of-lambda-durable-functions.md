---
title: 'Lambda Durable Functionsの制約'
emoji: '⛓️‍💥'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['aws', 'lambda', 'serverless']
published: true
---

AWS re:Invent 2025 で発表された AWS Lambda の新機能「Durable Functions」、皆さんはもう使ってみましたか？

僕が Durable Functions を利用したところ、思ったように設定できなかったため Durable Functions の制約について調べてみました。
制約のためにハマってしまったポイントも紹介していますので、もしよりよい方法をご存知の方はコメントいただくか X アカウント([@akikii\_\_](https://x.com/akikii__?s=21))までご連絡ください…！

## Lambda Durable Functions とは？

Lambda Durable Functions はマルチステップのワークフローを Lambda 関数の仕組みを拡張することで実現する機能です。
`ステップ`と`待機`によって最大 1 年間の長期実行を可能としています。

- `ステップ`(step): ワークフローの 1 ステップの処理を実行します。ステップで失敗したときに自動的に再実行されます。
- `待機`(wait, callBack): 指定された時間、もしくは API や人間の応答を待って Lambda 関数を中断します。この間は実行料金が発生しません。

他にも`並行処理`(map, parallel)を利用することで分岐や配列の数によって処理を並行に実行できます。

これらの機能は Lambda 関数を作成する時に Durable execution を有効にし、関数の中で Durable execution SDK を利用することによって実装できます。

## Durable Functions の制約

### Lambda 関数 1 回あたりの実行時間は 15 分まで

Durable Functions の特徴の一つに実行時間が最長 1 年間であることが挙げられます。しかし、Lambda 関数が 1 回あたりの実行に使える時間は従来通り 15 分のままです。

Durable Functions の`ステップ`では、処理が完了したら`チェックポイント`を保存して関数の実行を終了します。続いて Durable Functions が再度呼び出されて保存された`チェックポイント`まで処理をスキップする`リプレイ`が行われ、その後に次の`ステップ`が実行されます。この仕組みを繰り返すことで、ワークフロー全体が段階的に進行していきます。

![alt text](/images/restriction-of-lambda-durable-functions/lambda-async-sync-invoke.dio.png)

また、待機している間は関数は実行されておらず、実際にステップが処理されるタイミングでのみ Lambda 関数が起動します。そのため、Lambda 関数の最大実行時間は待機時間に関係なく、従来どおり 15 分のままです。

1 ステップが 15 分を超える Durable Functions を作って試してみます。
Durable Functions の実行タイムアウトは 15 分を超えていればいいため、 30 分に設定します。

![alt text](/images/restriction-of-lambda-durable-functions/execution-timeout-30-min.png)

Durable Functions のソースコードに 15 分を超える処理を書きます。16 分間待機する処理を書いてみました。

```ts
import { withDurableExecution } from '@aws/durable-execution-sdk-js';

export const handler = withDurableExecution(async (event, context) => {
  await context.step('Step #1', async () => {
    await new Promise((resolve) => setTimeout(resolve, 960000)); // NOTE: 16分
  });

  return {
    statusCode: 200,
    body: {},
  };
});
```

Durable Functions を非同期呼び出しでテストします。

![alt text](/images/restriction-of-lambda-durable-functions/async-invoke-function.png)

実行結果を見てみるとステータスが「タイムアウトしました」になっています。

![alt text](/images/restriction-of-lambda-durable-functions/status-timeout.png)

イベント履歴で失敗したステップを確認してみると、900 秒（15 分）のタイムアウト時間を超えたとのエラーメッセージが出ていました。

![alt text](/images/restriction-of-lambda-durable-functions/timeout-error-message.png)

```json
{
  "EventId": 3,
  "EventTimestamp": "2025-12-27T08:56:50.696Z",
  "EventType": "InvocationCompleted",
  "InvocationCompletedDetails": {
    "EndTimestamp": "2025-12-27T08:56:50.690Z",
    "Error": {
      "Truncated": false,
      "Payload": {
        "ErrorType": "Sandbox.Timedout",
        "ErrorMessage": "RequestId: 9ac63323-b549-481c-9dce-d3d014339e9a Error: Task timed out after 900.00 seconds"
      }
    },
    "RequestId": "9ac63323-b549-481c-9dce-d3d014339e9a",
    "StartTimestamp": "2025-12-27T08:41:50.135Z"
  }
}
```

対処方法として、15 分を超える可能性がある`ステップ`を分割することで Lambda 関数の実行時間を短くする方法が有効です。
分割できない単位の処理がどうしても 15 分を超えてしまう場合は、時間のかかる処理を切り出して非同期呼び出しするのもよさそうです(未検証)
(そこまでやるなら感覚的には Step Functions で実装してもよさそうなイメージはあります)

### 修飾 ARN を指定して呼び出す必要がある

Lambda 関数のコンソールからテストで呼び出したときは意識する必要がありませんでしたが、 Durable Functions の呼び出しには修飾 ARN が必須です。修飾 ARN とは、Lambda バージョンや Lambda エイリアスのような Lambda 関数のソースコードが特定できる状態になっている ARN です。

Durable Functions では、修飾 ARN を利用して呼び出さない場合エラーとなってしまいます。例えば SQS キュー の Lambda トリガーに Durable Functions の ARN をそのまま指定すると、指定した ARN が不適合とのエラーメッセージが出ます。

![alt text](/images/restriction-of-lambda-durable-functions/required-unqualified-arn.png)

Durable Functions のドキュメントにもバージョン、エイリアスもしくは`$LATEST` を指定する必要があるとの記載があります。

> Durable functions require qualified identifiers for invocation. You must invoke durable functions using a version number, alias, or $LATEST. You can use either a full qualified ARN or a function name with version/alias suffix. You cannot use an unqualified identifier (without a version or alias suffix).
> （翻訳）Durable Functions の呼び出しには、正規化された識別子が必要です。
> Durable Functions は、バージョン番号、エイリアス、または $LATEST を使用して呼び出す必要があります。
> 完全修飾 ARN、またはバージョン/エイリアス接尾辞付きの関数名を使用できます。
> 正規化されていない識別子（バージョンまたはエイリアス接尾辞なし）は使用できません。
>
> https://docs.aws.amazon.com/lambda/latest/dg/durable-invoking.html#durable-invoking-qualified-arns

Durable Functions を Lambda コンソールのテストタブから実行したときは、実は最新のバージョンである`$LATEST`が自動で呼び出されていました。

![alt text](/images/restriction-of-lambda-durable-functions/invoke-success.png)

キューの Lambda トリガーにエイリアスを含めた Lambda ARN を指定すると、キューから Durable Functions の呼び出しが可能になりました。

![alt text](/images/restriction-of-lambda-durable-functions/alias-invoke-trigger.png)

ちなみに AWS CDK で Lambda エイリアスを作成するときは、1 行だけ追加すればバージョン作成からエイリアスの張り替えまで行ってくれて大変便利です。

```diff ts
const func = new NodejsFunction(this, "Function", {
  entry: path.join(__dirname, "../src/index.ts"),
  runtime: Runtime.NODEJS_24_X,
  durableConfig: {
    executionTimeout: Duration.minutes(30),
  },
  currentVersionOptions: {
    removalPolicy: RemovalPolicy.RETAIN,
  },
});
+func.addAlias("latest");
```

エイリアスを張り替えるときに前のバージョンを削除してしまうので、気になる方は Lambda 関数の`currentVersionOptions.removalPolicy`に`RemovalPolicy.RETAIN`を設定すると削除されません（CDK の管理外になります）

### 呼び出し方法によって最大実行時間が 15 分までになる

先にも述べた通り、Durable Functions は実行時間が最長 1 年間です。しかし、呼び出し方法によっては最大実行時間が 15 分になってしまいます。

Lambda 関数には、同期呼び出し・非同期呼び出し・イベントソースマッピングの 3 種類の呼び出し方法があります。まずは同期・非同期呼び出しの説明をします。

- 同期呼び出し: Lambda 関数の処理が終わってからレスポンスが返ってきます。レスポンスには、Lambda 関数内で処理した結果を含めることができます。
- 非同期呼び出し: Lambda 関数の処理を待たずにレスポンスが返ります。レスポンスには lambda 関数の呼び出しが成功したかが含まれており、処理の結果を含めることはできません。

Durable Functions を同期呼び出ししたときは、実行タイムアウトが最大 15 分に制限されてしまいます。

試しに実行タイムアウトを 30 分に設定した Durable Functions を同期実行してみます。エラーが発生して、実行タイムアウトを 15 分以内にする必要があるとのメッセージが出ました。

![alt text](/images/restriction-of-lambda-durable-functions/sync-invoke-error.png)

Durable Functions を非同期呼び出ししたときは、実行タイムアウトを 30 分にした Durable Functions を呼び出せました。

![alt text](/images/restriction-of-lambda-durable-functions/async-invoke-success.png)

また、イベントソースマッピング（ESM）から呼び出すときも、実行タイムアウトが最大 15 分に制限されてしまいます。

ESM は、ストリームやキューを定期的にポーリングしてイベントを受け取る処理を行なっています。ESM からイベントを受け取ると、そのイベントをリクエストボディとして Lambda 関数を呼び出します。ESM は、SQS キュー や DynamoDB ストリームなどから利用できます。

この制約は、ESM が Durable Functions を同期的に呼び出しているため同期実行と同様の制限が設けられているみたいです。

> Event source mappings invoke durable functions synchronously, waiting for the complete durable execution to finish before processing the next batch or marking records as processed.
>
> （翻訳）イベントソースマッピングは、次のバッチを処理したり、レコードを処理済みとマークしたりする前に、Durable Functions の実行が完了するのを待ってから、Durable Functions を同期的に呼び出します。
>
> https://docs.aws.amazon.com/lambda/latest/dg/durable-invoking-esm.html

試しに SQS キューに対して実行タイムアウトが 30 分の Durable Functions を Lambda トリガーとして設定してみると、エラーが発生して実行タイムアウトを 15 分以下にする必要があるとのメッセージが出てきました。

![alt text](/images/restriction-of-lambda-durable-functions/execution-timeout-over.png)

15 分を超える実行タイムアウトを持つ Durable Functions を ESM から直接呼び出すことはできません。
そこで EventBridge パイプを利用することで、 ESM 経由の呼び出しを非同期化し Durable Functions を実行できます。

![alt text](/images/restriction-of-lambda-durable-functions/pipe-durable-functions.dio.png)

非同期呼び出しなので Durable Functions の実行タイムアウトは最長 1 年まで設定できます。

まとめると、Durable Functions の実行タイムアウトの最大値は、呼び出し方法によって次のように振る舞います。

- 同期呼び出し: 15 分
- 非同期呼び出し: 1 年
- イベントソースマッピング: 15 分

## ハマったポイント

僕がハマったポイントは、SQS キューをイベントソースマッピングで Durable Functions に連携し、メッセージを処理する構成における「再処理」の部分です。
具体的には、処理に失敗してデッドレターキュー（DLQ）に送られたメッセージをリドライブし、再度 Durable Functions で処理させる操作でつまずきました。

![alt text](/images/restriction-of-lambda-durable-functions/dlq-redrive-arch.dio.png)

この構成ではイベントソースマッピング呼び出しの制約で実行タイムアウトを 15 分より長く設定できなかったため、EventBridge パイプを利用して非同期呼び出しにする必要があると考えました。
しかし、EventBridge パイプから Durable Functions を非同期呼び出しした時点でメッセージが処理済みになってしまうため、Durable Functions で処理に失敗してもメッセージがデッドレターキューに送られません。
また、Lambda 関数自体の DLQ を設定できますが、 Lambda 関数の DLQ から 送信元の キューにはメッセージをリドライブできない仕様になっています。

![alt text](/images/restriction-of-lambda-durable-functions/dlq-redrive-with-pipes-arch.dio.png)

この仕組みは関数の再実行を簡単化することが目的でした。
もし Durable Functions が過去の呼び出しを再実行できるようになればこの話も解決するのですが、現時点では SQS + DLQ による再処理を前提とした設計とは相性が悪そうです。

## おわりに

Lambda Durable Functions の制約について紹介しました！
今回紹介したハマりポイントに当たったきっかけは、Athena を利用する関数でクエリ後にポーリングする処理を Durable Functions で実装できないか？と思ったためでした。
Durable Functions の利用用途としてよく Step Functions と比較されます。
個人的には SRE が作るようなオートメーション・オペレーション用のワークフローが Durable Functions に向いているケースだと思っています。特にポーリング処理は、Lambda 関数の処理時間を消費せずに待機できることが嬉しいです！

そして Durable Functions の過去の呼び出しを再実行できるようになることを待ち望んでいます…！
