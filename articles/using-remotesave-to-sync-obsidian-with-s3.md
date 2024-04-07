---
title: "Remotely Save で Obsidian を S3 経由で同期"
emoji: "🔄"
type: "tech"
topics:
  - "s3"
  - "obsidian"
published: true
published_at: "2023-09-24 00:59"
---

Remotely Save で Obsidian の同期を S3 経由で設定する

https://github.com/remotely-save/remotely-save
Community Plugin の Remotely Save を使って Obsidian の同期を S3 経由でやってみる。
設定方法だけメモる。
Obsidian Plugin 自体は初めて使うので戸惑った…

## AWS 側

1. Obsidian sync 用の S3 Bucket を作る



2. IAM ユーザーを作り、作成したバケットの権限のみ付与する IAM ポリシーをつける

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Action": [
				"s3:*"
			],
			"Resource": [
				"arn:aws:s3:::<your_bucket_name>",
				"arn:aws:s3:::<your_bucket_name>/*"
			]
		}
	]
}
```

3. IAM ユーザーのクレデンシャルとして Access key ID と Secret access key を取得する

## Obsidian 側
※ PC 側とスマホ側は同じ設定をする必要がある。

#### 1. Community Plugin を有効にして、Remotely Save をインストールする

#### 2. Remotely Save を有効化する

#### 3．下記の設定をする
- Choose A Remote Service: S3 or compatiable
- Endpoint: s3.ap-northeast-1.amazonaws.com
- Region: ap-northeast-1
- Access Key ID：先程取得した Access key ID を記入する
- Secret Access Key: 先程取得した Secret access key を記入する
- Bucket Name: 先程作成したバケット名を記入する
- S3 URL style: Path-style
- Bypass CORS Issue Local: Enable

残りはデフォルト設定にして、
Check Connection を押して、接続できるか確認する。

特に保存する必要がないので、Window を閉じて設定画面終了する。

#### 4. 左上の Sync ボタンを押して、実際 Sync できるかを試してみる。

問題なくスマホと PC で Sync できた。