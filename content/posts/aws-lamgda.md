---
title: "Aws Lamgda"
date: 2017-12-27T22:02:03+09:00
draft: false
---

# AWS Lambda
著: 西谷圭介

## Lambda概要
- コンテナ技術を利用して実行されている
- プロセスをspwan可能
	- 子プロセス生成+コマンド実行
- スレッド作成可能
- /tmpへIO可能
	- 容量512MB
- メモリ使用量は最大3GB
	- メモリとCPU能力は比例している
- 呼び出しタイプ
	- RequestResponse: 同期
	- Event: 非同期
- DynamoDB
	- 各テーブルに対して `Amazon DynamoDB Streams` を有効にする必要あり
	- RequestResponse
- イベントソース
	- ストリームベース
		- 対象をポーリングして変更等をトリガーに実行する
			- DynamoDB
			- Kinesis
	- [サンプル](http://docs.aws.amazon.com/ja_jp/lambda/latest/dg/eventsources.html#eventsources-ddb-update)
- ポリシー
	- アイデンティティベース
		- IAMポリシー
		- Lambda関数が利用可能な権限
	- リソースベース
		- Lambda関数を呼び出す権限
		- CLIを利用して付与

### 料金
- 対象
	- Lambda関数のリクエスト数
	- Lambda関数の実行時間
- $0.00001667 = 1GM 1s のLambda料金を基準にして計算される

### バージョニング
- バージョン指定可能
- エイリアス指定可能
	- エイリアスのバージョンを変更することでリリース
	- 切り戻どしもエイリアス変更で完了
- 運用
	- Prod: v1.0
	- Dev: $LATEST

```sh
# version発行
aws lambda publish-version --function-name <関数名>

# alias発行
aws lambda create-alias \
		--function-name <関数名> \
		--name Prod \
		--function-version 3

aws lambda create-alias \
		--function-name <関数名> \
		--name Dev \
		--function-version '$LATEST'

# alias一覧
aws lambda list-alias --function-name <関数名>

# alias更新
aws lambda update-alias \
		--function-name <関数名> \
		--name Prod \
		--function-version 4
```

### デプロイパッケージの作成
- デプロイパッケージ
	- サードパーティのライブラリ等を利用する場は必須
	- zip化してアップロードする
	- S3にアップロードしたファイルを設定で読ませることも可能
- Node.js
	- npmで落としたパッケージごとzip
	- package.json含む
	- **ディレクトリ配下のみをzip化する**
		- `zip -r package.zip ./*`

```sh
aws lambda update-function-code \
		--function-name <関数名> \
		--zip-file fileb://package.zip
```

### VPCアクセス
- サブネット+セキュリティグループをLambdaに設定
- 発火時にENIをVPC内に自動で作成する
	- 作成のためレイテンシが発生
	- 10~30秒程度
- VPCアクセスする際は、インターネットに出れなくなる
	- NAT Gatewayを利用して外への通路を開ける必要がある
- 必要ポリシー(AWSLambdaVPCAccess)
	- ec2:CreateNetworkInteface
	- ec2:DescribeNetworkInteface
	- ec2:DeleteNetworkInteface
- IP固定
	- グローバルIPアドレス固定が可能
	- NAT Gatewayを利用する必要あり
- アクセス元
	- 固定するBest PracticeはSGの設定

### 環境変数
- Node.jsでのアクセス方法
	- `process.env.<環境変数名>`
- 合計4KB以下にしなければならない
- 予約済みの環境変数有り
- バージョニングにて設定が固定される
	- コード
	- 環境変数
- KMSで暗号化される
	- 独自キーを利用するとKMS分の料金がかかる
	- Policy: `kms:Decrypt` が必要
	- 独自キーの利用がおすすめされている

### エラーハンドリング
- イベントソースによって異なる
- ストリームベース
	- Kinesis or DynamoDB
	- 有効期限が切れるまでリトライされ続ける
	- 成功するまで次の処理は実行されない
- 同期呼び出し
	- 呼び出し元へエラーを返す
	- リトライの集中による2次障害を避けるよう実装する
		- Exponential Backoff
		- Jitter(ゆらぎ)をいれる
		- [参考](http://yoshidashingo.hatenablog.com/entry/2014/08/17/135017)
- 非同期呼び出し
	- 2回自動呼び出しされる
	- デッドレターキューが構築
		- 3回失敗するとSQS or SNSに通知される
		- エラーを取りこぼさなくてすむようになっている

#### デッドレターキュー
- 非同期処理に失敗した場合にフォローする仕組み
- Amazon SQS/SNSを登録しておく
- 関数単位で設定可能

#### 同時実行数
- 同時実行数はイベントソースによって異なる
- ストリームベースの場合
	- 1シャード1Lambda
	- シャード数が最大同時実行数となる
- それ以外
	- `同時実行数 = 1秒あたりのイベント数 * 関数の実行時間`
	- 例
		- 10 件/秒 * (実行時間) 3秒 = (同時実行数) 30回
	- 最大数で計算するため
- 標準で1000
- 超えた場合は、スロットリングに入る(= CloudWatchに入る)
- スロットリング後の処理
	- 同期: 429エラー
	- 非同期: 6時間再実行

## メトリクスとログ
### メトリクス
| Metrics           | 意味                                                                                       |
| :-------------    | :----                                                                                      |
| Invocations       | Lambdaの呼び出し回数<br> スロットリングは含まれない                                        |
| Dulation          | 所要時間<br> 初期化処理やグローバルスコープの実行時間は含まれない                          |
| Errors            | 実行した場合のエラー数<br> 下記は含まれない<br>- 同時実行数上限<br> - 関数の実行自体の失敗 |
| Throttles         | スロットリングされた回数                                                                   |
| Dead Letter Error | デッドレターキューエラー                                                                   |
| Iterater Age      | ストリームに書き込まれてから読み出されるまでの時間                                         |

### ロギング
- CloudWatch Logsへ全てのログは吐かれる
- ロググループ
	- aws/lambda/<関数名>
	- 関数内: context.logGroupName
- ログストリーム名
	- 関数内: context.logStreamName

## テスト
### ユニットテスト
- 特別なことをする必要はない
- おすすめ
	- 「ユニットテストはローカル、それ以外はクラウド」

#### ローカル実行
- 3点
	- Local Entry Point
	- Event 
	- Context Object
- Entry Point

```js
exports.handler = (event, context, callback) => {
	callback(null, "Hello!")
}

event = ""
context = ""
callback = (error, result) => {
	if (error != null) {
		console.log(error)
		process.exit(1)
	}
	console.log(result)
	process.exit(0)
}

this.handler(event, context, callback)
```

- Event
	- JSONファイルとして静的に作成する
	- 実行時にファイル読み込みを実施する
- Context Object
	- 利用しないならエミュレート不要


### Node.js
基本形
```js
// strict modeで実行する
'use strict'

```

- callback
	- 全ての非同期呼び出し
	- `callback(Error error, Object result)`
	- error時は、errorに値が入る
		- 正常時はnull
- context
	- `getRemaingingTimeInMillis()` : 残りの実行時間

### Best Practice
#### ステートはバイブに保存して、Lambda関数はステートレスにする
- (補足) ステートレスとは?
	- Client/Server間に状態を持たない
	- リクエスト1つ1つに必要な情報が全て全て含まれること

#### なるべく非同期を利用
- 実行数条件に引っかかる場合が多い
- 非同期
	- バースト許容
	- **エラーになって処理が行われないことを防げる**
- できるだけ非同期で実行することが上手く制御できるコツ
- API GW + 更新系
	- APIGWをサービスProxyとして利用
	- SQS/Kinesisへ突っ込む
	- ストリームベースで非同期実行する

#### コールドスタートとウォームスタート
- コールドスタート
	- コンテナ作成
	- コードロード/展開
	- 初期化処理
		- グローバルスコープ
		- コンストラクタ
	- (VPCアクセスの場合) ENI作成
- ウォームスタート
	- 作成されたコンテナを利用して実行される
- コールドスタート対策
	- 定期ポーリング(5分)
	- メモリ使用量を増やす(CPU能力UP)
	- VPC内リソースへアクセスしない
	- パッケージサイズを小さくする
	- グローバル変数の遅延ロード
		- グローバル変数として宣言
		- ハンドラ内で初期化/変数の格納

#### 冪等性はアプリ側で担保
- 2回実行される場合がまれに発生する
- 冪等であるべきな場合は、アプリ側で保証する

## サーバーレスとは？
- 仮想化
	- ハードウェアを抽象化
- コンテナ
	- OSが抽象化
- サーバーレス
	- コンテナレベルで抽象化
	- コードにのみ集中
- サーバーレスで特別なこと
	- イベントドリブン
	- 関数同士が疎結合でステートレス

## 疑問点
- Lambda Functionの分割方法
	- CRUDでは分ける必要はないのか


## 参考
- [API Gateway + Lambdaステージとエイリアスを使ってバージョン管理してみた](https://dev.classmethod.jp/cloud/aws/version-management-with-api-gateway-and-lambda/)