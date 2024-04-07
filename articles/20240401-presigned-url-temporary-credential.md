---
title: "AWS CLI SSOとPresigned URLを使用する際の注意点"
emoji: "🫢"
type: "tech"
topics:
  - "aws"
  - "s3"
  - "sso"
published: false
published_at: "2024-04-01 00:44"
---

# はじめに
AWS SSO はシングルサインオンを実現して、AWS リソースへのアクセス管理を容易にする便利な機能です。
AWS CLI v2 では、SSO で一時的な認証情報でログインすることも可能です。
ただ、一時的な認証情報を使用することになるため、S3 Presigned URL を発行するときには、認証情報が切れたら発行された URL も失効する問題に遭遇したので説明します。

# AWS IAM Identity Center (SSO)
AWS SSOは、一度の認証でさまざまなAWSアカウントやサービスにアクセスできるシングルサインオンの仕組みを提供します。IAM Identity Center（SSO）を使用して一時的な認証情報でログインすることで、アクセスキーとシークレットキーの管理が不要になり、アクセス管理とセキュリティが向上します。

AWS CLI を SSO で認証させるのに興味がある方は下記見てみると良いかもです。
https://qiita.com/oiz-y/items/d01a193be93789b3ab43
https://dev.classmethod.jp/articles/aws-cli-for-iam-identity-center-sso/

# Presigned URL
Presigned URLは、AWS S3オブジェクトへの一時的なアクセス権を付与するために使用されます。通常、Presigned URLには有効期限が設定され、その期間中はURLを使ってオブジェクトにアクセスできます。

会社の規定上 S3 バケットを Public にしてはいけないため、大きいファイルを共有するときには S3 にアップロードして S3 Presigned URL で 7日間有効で共有したいと思いました。

ローカルで Presigned URL を下記のコマンドで発行した。

```
aws s3 presign s3://{bucket_name}/{file_name} --expires-in 604800
```

本来これで発行した URL は 7 日間有効になるはずだが、数時間後には `ExpiredToken error` というのになってしまった。
理由を調べたら、Presigned URL を発行するときに使うクレデンシャルが失効すれば、Presigned URLの残りの有効期限に関係なく即座に失効するとのことらしい。

CLI は SSO でログインしていたため一時的な認証情報を利用していた。そのため、すぐにクレデンシャルが失効してしまったのです。

結果的に、AWS CLI では IAM ユーザーで発行したアクセスキーとシックレートキーで認証し直して Presigned URL を発行した。

# 終わりに
こういう問題があるので、Presigned URL を発行するとき、セッション有効期限と Presigned URL の有効期限を両方気にしないといけないのです。
ローカルでPresigned URL発行する際にはこのような注意点があることを認識するのも大事です。

基本これからは
- セッションの有効期限を考慮し、Presigned URLの有効期限を設定する（どうしても短い期間になる）
- マネジメントコンソールで Presigned URL を作成する（最大12時間有効）
- ローカルで IAM ユーザーを利用して長期的に有効なクレデンシャルを利用して最大 7 日間有効の Presigned URL を作成する

でいきたいと思います。
