---
title: "SageMaker Jumpstart を使って Whisper でリアルタイム文字起こし"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","sagemaker","whisper","文字起こし","Mac"]
published: true
---


# はじめに
この記事は下記の内容について説明します。
1. SageMaker JumpStart で Whisper をほぼワンクリックでホスティング
2. PC の音声をリアルタイムで 10 秒ごとに文字起こしさせる
3. Blackhole を使って、Macbook で PC からの出力とマイクからの入力を両方文字起こしさせる方法

# Whisper を SageMaker JumpStart でセルフホストする
Whisper で文字起こししたいが、外部の API を使いたくないのでセルフホストしたいときがあります。ただ、自分で設定すると依存関係の解決がかなり大変です。
色々試してみましたが、おそらく Sagemaker JumpStart で Whisper をセルフホストするのが一番簡単なのでその方法を紹介します。便利すぎてびっくりです。

## Sagemaker JumpStart について
SageMaker Jumpstart は SageMaker Studio の機能の一つであり、事前トレーニング済みのモデルをワンクリックでデプロイできます。
モデルのラインアップが豊富であり、多くの有名なオープンソースのモデルが使えるようになってます。
![sagemaker-jumpstart-model-line-up-1](/images/transcription/image.png)
![sagemaker-jumpstart-model-line-up-2](/images/transcription/image-1.png)

SageMaker Studio のセットアップなどはここで省略するがこちらの記事は割と詳しくあるのでご参照ください。
https://zenn.dev/kento_mm_ninw/articles/07576364700096

## Whisper をホストする
やり方は簡単。
1. Jumpstart の画面で Whisper を検索する
![search-whisper](/images/transcription/image-2.png)
2. Whisper Large V3 を選択してデプロイボタンを押す
![alt text](/images/transcription/image-3.png)
3. 設定を確認し、デプロイボタンを押す
![alt text](/images/transcription/image-4.png)
終わりです。

作り終わったら、左の Deployment -> Endpoints の部分で作成されたエンドポイントを確認できます。
In Service になったら使える状態です、
そこの Endpoint Name をメモっておきましょう。

# ローカルからの音声をリアルタイムで文字起こしさせる
ここからは、ローカルの音声をリアルタイムで文字起こしをしたいと思います。

## Whipser のモデルの制限
注意点としては、Whisper はいくつか制限があります。
- 音声の長さは 1 min のみ
- 音声のサイズは 25 MB 以下
- 音声のフォーマットは .wav のみ

そのため、リアルタイムで細切れした音声を送るか、長い音声の文字起こしは1分ごとにする処理を自前で実装する必要があります。
私はせっかちで文字起こしの結果をすぐみたいので、リアルタイムで 10 秒ごとに推論させることにしました。

## マイクからの音声のリアルタイムで文字起こしさせる
デバイスに入力した音声を 10 秒ごとにリアルタイムで SageMaker エンドポイントに送り、推論結果をターミナルに出力、終わったら文字起こし全文を txt に保存するようにしています。また、このあと述べる、PC の入力と出力を一つにまとめてエンドポイントに送っている都合上、デバイスを指定するようにしています。

```py
import pyaudio
import boto3
import json
from pydub import AudioSegment
from io import BytesIO
from datetime import datetime
import os

def find_device_index_and_info(audio, name):
    device_count = audio.get_device_count()
    for i in range(device_count):
        device_info = audio.get_device_info_by_index(i)
        if name in device_info['name']:
            return i, device_info['maxInputChannels'], int(device_info['defaultSampleRate'])
    return None, None, None

client = boto3.client('runtime.sagemaker')
audio = pyaudio.PyAudio()
device_name = "Input" # 文字起こししてほしい入力デバイスを設定する

transcript_name = input("Enter the name for the transcript: ")

device_index, CHANNELS, RATE = find_device_index_and_info(audio, device_name)
if device_index is None:
    raise Exception("Audio device not found")

FORMAT = pyaudio.paInt16
CHUNK = 4096
RECORD_SECONDS = 10
ENDPOINT_NAME = "your-sagemaker-endpoint-name" # 先程作った Endpoint Name

stream = audio.open(format=FORMAT, channels=CHANNELS, rate=RATE, input=True, frames_per_buffer=CHUNK, input_device_index=device_index)

current_time = datetime.now().strftime("%Y%m%d_%H%M%S")
file_path = f"{current_time}_{transcript_name}.txt"

with open(file_path, "w") as f:
    try:
        print("start speaking...")
        while True:
            frames = [stream.read(CHUNK, exception_on_overflow=False) for _ in range(int(RATE / CHUNK * RECORD_SECONDS))]
            audio_segment = AudioSegment(data=b''.join(frames), sample_width=audio.get_sample_size(FORMAT), frame_rate=RATE, channels=CHANNELS)
            wav_io = BytesIO()
            audio_segment.export(wav_io, format="wav")
            response = client.invoke_endpoint(EndpointName=ENDPOINT_NAME, ContentType='audio/wav', Body=wav_io.getvalue())
            transcription = json.loads(response['Body'].read().decode())['text']
            print("Transcription:", transcription)
            f.write(" ".join(transcription) + "\n")
    except KeyboardInterrupt:
        print("Speaking ends.")
    finally:
        print(f"transcription saved: {file_path}")

stream.stop_stream()
stream.close()
audio.terminate()
```

このコードは、指定されたオーディオ入力デバイスから音声をキャプチャし、Amazon SageMakerのエンドポイントを使用してリアルタイムで文字起こしを行うものです。

1. `find_device_index_and_info`関数は、指定されたデバイス名に一致するオーディオデバイスのインデックス、最大入力チャンネル数、およびデフォルトのサンプルレートを取得します。
2. `boto3`クライアントを使用してSageMakerのエンドポイントに接続し、`pyaudio`を使用してオーディオストリームを設定します。
3. ユーザーに文字起こしファイルの名前を入力させ、その名前でファイルを作成します。
4. 指定されたデバイスからオーディオをキャプチャし、10秒ごとにSageMakerエンドポイントに送信して文字起こしを取得します。
5. 取得した文字起こしをファイルに保存し、リアルタイムでコンソールに出力します。
6. キーボード割り込み（Ctrl+C）で録音を停止し、ファイルに保存された文字起こしのパスを表示します。

# Blackhole を使って Macbook からの音声とマイクの音声両方を文字起こしさせる
ここからは余談ですが、普通に録音すると自分のマイクの音声しか入りません。PC からの出力もキャプチャーするためには、工夫が必要です。

オンライン会議など、こちらが入力した音声とPCから出力された音声両方をキャプチャして、両方結合した状態で文字起こしさせたいときはあります。

その場合、仮想オーディオドライバーの Blackhole を使って実現可能です。

## Blackhole
Blackhole は、PC からの出力音声をキャプチャーするために使うものです。OBS で配信することや、PC 音声付きで録画したいとかをする場合は結構使われるかと思います。
https://existential.audio/blackhole/

今回は Blackhole を使って、PC からの出力音声をキャプチャーして、更に入力デバイスとセットを作り、両方の音声が合わさった状態で SageMaker Endpoint に投げる。
Blackhole のダウンロードとインストール手順は省略します。

## やること
Mac の Audio MIDI でオーディオ装置を設定します。

概要としては、PC の Output を利用の出力デバイスと仮想オーディオドライバーの Blackhole 両方に流すことで、PC 出力を Blackhole でキャプチャーし、そしてマイクの入力と Blackhole の音声を束ねて一つのデバイスに結合することです。
https://support.apple.com/en-us/102171

### 1. 複数出力装置 (Multi output device) の設定
目的は、PC 出力音声を同時にヘッドホンもしくはスピーカーと Blackhole 両方におくるため。
私の場合、下記のように Output という複数出力装置の設定をした。
![multi-output-device](/images/transcription/image-6.png)
![multi-output-device](/images/transcription/image-5.png)

### 2. 仮想デバイス機器セット (Aggregated Device)の作成
目的は Blackhole（PC 出力音声）とマイクの入力音声を一つのデバイスにまとめるため。
私の場合、入力と出力を Input という機器セットに設定した。
![aggregated-device](/images/transcription/image-7.png)

### 3. SageMaker Endpoint への入力
前述したコードで、実際コンピューターで利用しているデバイスと関係なく、デバイスを `Input` に設定して音声をキャプチャーして SageMaker Endpoint に入力している。

コードを実行すると、きれいに両方の音声が文字起こしされました。
# おわりに
このように、Macbook でマイクからの入力と PC からの出力音声を束ねて SageMaker JumpStart でセルフホストした Whisper で文字起こしすることができました。

Whisper の文字起こし自体の感想として、私の場合は10秒に一回の文字起こしのため、かなり限られた文脈でしか文字起こしできないので、精度がとても高いとは言えない。
また言語は自動判別をしているので、途切れた文章だとほかの言語だと判断される場合もある。
10秒間無言のときは、`thanks for watching` とかが出力されることも多いので、トレーニングで使われそうなデータを想像しながら眺めてました。
