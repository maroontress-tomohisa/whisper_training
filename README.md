# OpenVINO によるモデルの学習・実行

## はじめに

このドキュメントでは、環境構築とともに OpenVINO を使った学習済みモデルの活用方法を学びます。
動作環境としては、**Intel Core Ultra シリーズ搭載の Windows 11** を想定しています。
今回は音声認識モデル「Whisper」を扱います。

## セットアップ

コードエディタとして [VS Code][] の使用を推奨します。
インストール後、拡張機能から「Japanese Language Pack for Visual Studio Code」を検索・導入することで日本語化できます。

実装の中心は Python ですが、インタプリタだけでなく、いくつかのサードパーティ製ライブラリも必要です。
これらのライブラリは、ホストマシンではなく専用のディレクトリ内にインストールすることを推奨します。
この役割を担うのがパッケージマネージャであり、[Poetry][] や [uv][] が代表例です。
今回は、軽量かつ多機能な **uv** を使用します。

以下のコマンドを PowerShell で実行してください:

```powershell
powershell -ExecutionPolicy Bypass -Command "irm https://astral.sh/uv/install.ps1 | iex"
```

> [!NOTE]
> - 環境変数 `PATH` を適宜修正し、`uv` コマンドが実行できるようにしてください。
> - 以降のコマンドは、**PowerShell** でも **コマンドプロンプト** でも利用可能です。

作業しやすいよう、デスクトップに `whisper_training` という新しいプロジェクトディレクトリを作成しましょう。
以下を実行してください:

```powershell
cd Desktop
uv init whisper_training
```

プロジェクトフォルダが作成されたら、VS Code で開いてください。
初期状態の `hello.py` はサンプルファイルなので、削除して構いません。

---

### 必要なライブラリのインストール

`pyproject.toml` を以下の内容で上書きしてください:

```toml
[project]
name = "whisper-training"
version = "0.1.0"
description = "whisper-training for OpenVINO"
readme = "README.md"
requires-python = ">=3.11"
dependencies = [
    "pyannote-audio ==3.3.2",
    "torch ==2.6.0",
    "torchaudio ==2.6.0",
    "transformers ==4.36.2",
    "openvino ==2024.1.0",
    "optimum-intel ==1.22.0",
]
```

とくに `dependencies` の項目が重要で、Whisper を OpenVINO 上で実行するために必要なライブラリが記載されています。

以下のコマンドを実行し、`.venv` ディレクトリが作成されることを確認してください:

```
cd whisper_training
uv sync
```

---

### モデルのダウンロードと変換

Whisper のモデルおよびトークナイザーを変換し、保存します。
以下のコードを `dlmodel.py` という名前で保存してください:

```python
from optimum.intel.openvino import OVModelForSpeechSeq2Seq
from transformers import AutoProcessor

model_id = "openai/whisper-large-v3-turbo"
model = OVModelForSpeechSeq2Seq.from_pretrained(model_id, export=True)
model.save_pretrained('./assets')
processor = AutoProcessor.from_pretrained(model_id)
processor.save_pretrained('./assets')
```

以下を実行してください:

```powershell
uv run dlmodel.py
```

`assets` ディレクトリが作成され、その中にファイルが生成されていれば成功です。

---

### 音声認識の実行

無料のサンプル音源を使用します。
以下の音声ファイルを `sample.wav` の名前でプロジェクトフォルダに保存してください:

- https://www.hke.jp/products/voice/wav/audition/01.femal.wav

次に、以下のコードを `main.py` という名前で保存します:

```python
from optimum.intel.openvino import OVModelForSpeechSeq2Seq
from transformers import AutoProcessor, pipeline
import torch
import torchaudio
import time

# モデルとトークナイザーの読み込み
model = OVModelForSpeechSeq2Seq.from_pretrained("./assets").to('GPU')
model.compile()
processor = AutoProcessor.from_pretrained("./assets")

# 音声ファイルの読み込みとテンソルへの変換
waveform, sample_rate = torchaudio.load("sample.wav")

# サンプリングレートの確認と変換（必要な場合）
if sample_rate != 16000:
    resampler = torchaudio.transforms.Resample(orig_freq=sample_rate, new_freq=16000)
    waveform = resampler(waveform)

# 入力テンソルの整形
inputs = processor(waveform.squeeze(), sampling_rate=16000, return_tensors="pt")

# 実行時間の計測開始
start_time = time.time()

# モデルによる音声認識
with torch.no_grad():
    generated_ids = model.generate(inputs["input_features"], language="ja")

# 認識結果のテキスト化
text = processor.batch_decode(generated_ids, skip_special_tokens=True)[0]

# 実行時間と結果を表示
print(time.time() - start_time)
print(text)
```

以下のコマンドで実行します:

```powershell
uv run main.py
```

実行時間と認識結果が表示されれば成功です。

[VS Code]: https://code.visualstudio.com/
[Poetry]: https://python-poetry.org/
[uv]: https://docs.astral.sh/uv/
