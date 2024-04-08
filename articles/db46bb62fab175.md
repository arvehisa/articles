---
title: "ffmpeg を Docker Lambda で使う方法"
emoji: "🎧"
type: "tech"
topics:
  - "aws"
  - "docker"
  - "lambda"
  - "dockerfile"
  - "ffmpeg"
published: true
published_at: "2024-04-05 21:26"
---

# はじめに
動画処理を行う際、ffmpeg は非常に強力なツールです。 S3 PUT をトリガー Lambda を起動させて動画や音声のフォーマット変換もよくあるユースケースでしょう。
しかし、Lambda (python) で ffmpeg を使う際には、pip install や yum install が使えないため Lambda レイヤーや Docker Lambda を利用する必要があります。
この記事では Docker Lambda で ffmpeg をインストールする方法、注意点、デプロイ方法について説明します。

# Lambda で ffmpeg をインストールするためのオプション
ffmpeg を AWS Lambda で使うためには、大きく分けて2つの方法があります。

- Lambda Layer を使う
- Docker Image Lambda を使う

今回は、Docker Image Lambda を選択しました。
理由は、利用経験があるのと、CDK で DockerImageFunction を使ったデプロイをしたいためです。

# 最新の Lambda ベースイメージは Amazon Linux 2023
2023 年 11 月から、最新の Lambda Docker Image は、 Amazon Linux 2023 を利用することになりました。
Amazon Linux 2023 で利用しているパッケージ管理ツールは yum ではなく dnf になりました。また、Lambda の Amazon Linux 2023 ベースのイメージは microdnf を利用します。
Dockerfile はその影響も反映しています。
https://aws.amazon.com/jp/blogs/compute/introducing-the-amazon-linux-2023-runtime-for-aws-lambda/

# Dockerfile
以下は、ffmpeg をインストールするための Dockerfile です。
```Dockerfile
FROM public.ecr.aws/lambda/python:3.12

RUN microdnf install -y tar gzip xz

RUN curl -O https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz && \
    tar -xJf ffmpeg-release-amd64-static.tar.xz && \
    mv ffmpeg-*/ffmpeg /usr/local/bin/ && \
    mv ffmpeg-*/ffprobe /usr/local/bin/ && \
    rm -rf ffmpeg-*

COPY main.py ./

CMD ["main.lambda_handler"]
```
Dockerfile では、以下の手順で ffmpeg をインストールしています。
1. curl コマンドで ffmpeg の静的ビルド版をダウンロード
2. tar コマンドでダウンロードしたアーカイブを解凍
3. 解凍した ffmpeg バイナリを /usr/local/bin/ に移動
4. 不要になったファイルを削除



# Python Lambda ファンクションの作成と実行
この例では、S3 PUT OBJECT のイベントをトリガーに、Lambda が mp4 から m4a に変換しています。
やっていることは、S3 PUT イベントから、ソースバケットとキーを取得してファイルを Lambda の /tmp 領域にダウンロードする。ffmpeg でフォーマット変換をして上で、Output の S3バケットにアップロードしています。

```python
import os
import boto3
import subprocess
from urllib.parse import unquote_plus

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    key = unquote_plus(event['Records'][0]['s3']['object']['key'])
    
    download_path = f'/tmp/{os.path.basename(key)}'
    output_path = f'/tmp/{os.path.splitext(os.path.basename(key))[0]}.m4a'

    output_bucket = '｛your_output_bucket｝'
    
    s3_client.download_file(source_bucket, key, download_path)
    
    convert_mp4_to_m4a(download_path, output_path)
    
    s3_client.upload_file(output_path, output_bucket, os.path.basename(output_path))


def convert_mp4_to_m4a(input_file, output_file):
    command = ['ffmpeg', '-i', input_file, '-vn', '-acodec', 'copy', output_file]
    subprocess.run(command, check=True)
```

# ephemeral storage (/tmp)
Lambda の ephemeral storage (/tmp) に上限があります。
デフォルトでは 512 MB に設定されていますが、最大 10GB までいけるようになってます。
大きいファイルを処理する必要があるときには Lambda の ephemeral storage 設定も忘れずに行ってください。
![](https://storage.googleapis.com/zenn-user-upload/53ddab20c5df-20240407.png)

# デプロイ
私の場合、インフラを CDK でデプロイしていますので参考までにコードを貼ります。

- DockerImageCode を利用して直接 Dockerfile があるディレクトリを指定したらローカルでビルドしてデプロイしてくれます。
- Platform を AMD64 を指定することで Macbook で ARM のチップを使っていても x86 にビルドします。
- 5 分の Timeout を設定
- Memory Size を 512 MB に設定
- Lambda のアーキテクチャを x86 に設定
- ephemeralStorageSize を 4GB に設定

```ts
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as s3 from 'aws-cdk-lib/aws-s3';
import { Platform } from 'aws-cdk-lib/aws-ecr-assets';
// ...
const convertToAudio = new lambda.DockerImageFunction(this, 'convertToAudio', {
  functionName: `${resourceName}-convertToAudio`,
  code: lambda.DockerImageCode.fromImageAsset('./{your_code_directory}',{ platform: Platform.LINUX_AMD64 }),
  timeout: cdk.Duration.seconds(300),
  memorySize: 512,
  architecture: lambda.Architecture.X86_64,
  role: lambdaRole,
  ephemeralStorageSize: cdk.Size.mebibytes(4096)
});
```

# まとめ
AWS Lambda で ffmpeg を Docker Lambda を使ってインストールする方法について書きました。ご参考になれますと幸いです。
