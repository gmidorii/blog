---
title: "Aws Lamgda"
date: 2017-12-27T22:02:03+09:00
draft: true
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

## ロギング
### メトリクス
| Metrics           | 意味                                                                                       |
| :-------------    | :----                                                                                      |
| Invocations       | Lambdaの呼び出し回数<br> スロットリングは含まれない                                        |
| Dulation          | 所要時間<br> 初期化処理やグローバルスコープの実行時間は含まれない                          |
| Errors            | 実行した場合のエラー数<br> 下記は含まれない<br>- 同時実行数上限<br> - 関数の実行自体の失敗 |
| Throttles         | スロットリングされた回数                                                                   |
| Dead Letter Error | デッドレターキューエラー                                                                   |
| Iterater Age      | ストリームに書き込まれてから読み出されるまでの時間                                         |


