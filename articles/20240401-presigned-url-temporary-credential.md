---
title: "S3 Presigned URL 有効期限の注意点"
emoji: "🫢"
type: "tech"
topics:
  - "aws"
  - "s3"
  - "sso"
  - "cloudshell"
published: true
published_at: "2024-04-08 11:00"
---

# はじめに
S3 の Presigned URL を手動で発行するときありませんか？
私はたまに、Presigned URL を利用して資料の共有をしています。
今回は私は AWS SSO でログインした AWS CLI で発行した Presigned URL の有効期限が設定したものより短い現象に遭遇しましたので、Presigned URL の有効期限について注意点を共有します。

# Presigned URL
Presigned URL は S3 Object への一時的なアクセス権を付与するために使用されるものです。
通常、Presigned URL には有効期限が設定され、その期間中に URL を使って S3 Object にアクセスできます。

# Presigned URL の有効期限
実は、Presigned URL の有効期限は
1. Presigned URL を発行するときに指定する有効期限
2. Presigned URL を発行したクレデンシャルの有効期限

という2つのものと関係しています。どちらか一つが切れたら、S3 Object にアクセスできなくなります。

## 発行時に指定する有効期限について
Presigned URL はコンソールで発行する際には、最大 12 時間の有効期限を指定することが可能。
プログラム的に CLI もしくは CDK で発行する際には、最大 7 日間の有効期限を指定することが可能。

### コンソールで発行する方法
![コンソールで S3 Presigned URL を発行する方法](/images/s3-presigned-url/01_console.png)
![S3 Presigned URL 有効期限の指定](/images/s3-presigned-url/02_console_expire_time.png)
最大 7 日間の有効期限を指定可能

### CLI で発行する方法
```bash
aws s3 presign s3://{bucket_name}/{file_name} --expires-in 604800
```
`--expires-in` のオプションで有効期限を指定する。最大は 604800 秒、つまり 7 日間まで指定可能。

## クレデンシャルの有効期限
もう一つ関連するのは発行時に使うクレデンシャルの有効期限です。
発行時に使うクレデンシャルが無効になったら、設定した Presigned URL の有効期限に達しているかにかかわらず URL は無効になります。

例：
- Cloudshell で CLI を利用している場合
- ローカルで SSO で CLI にログインしている場合

などは、IAM ユーザーのような永続的なクレデンシャルを使っているわけではなく、数時間しか有効ではない一時的なクレデンシャルが利用される。
その場合、Cloudshell のセッションが終了した、またはローカルの SSO セッションが修了した場合、Presigned URL の期限を 7 日間に設定したにもかかわらず数時間で無効になる現象が起こります。

それは、SDK でプログラム的に発行するときも同じです。ただ、SDK でアプリケーション内で組み込んで利用する場合は短い有効期限を設定して、利用するたびに発行するケースが多いためあまり問題になることは少ないでしょう。

私は今回ローカルで SSO で CLI にログインして発行していましたが、思ったよりすぐ期限切れになり `ExpiredToken error` となりました。
参考：SSO CLI ログインについて
https://dev.classmethod.jp/articles/aws-cli-for-iam-identity-center-sso/

# まとめ
Presigned URL の有効期限は、発行時に設定した有効期限と関連するだけでなく、発行時に利用したクレデンシャルの有効期限にも関連します。
一時的なクレデンシャルを利用して発行する際には有効期限には注意しないといけないですね。

- コンソールで最大 12 時間の Presigned URL を発行
- IAM ユーザのクレデンシャルで CLI 等で発行する

数時間以上有効など、長い有効期間を設定したい場合は上記の方法を検討しましょう。