# Python下moviepy加速的环境配置

## 1.安装CUDA12.4版本

具体要看Pytorch支持到哪个版本。

## 2.安装Python

版本为Python 3.12.7。

## 3.升级pip

```bash
pip install --upgrade pip
```

## 4.安装moviepy

```bash
pip install moviepy
```

## 5.安装ImageMagick

并且将convert.exe拷贝至安装目录：[convert.zip](https://attach.xcopter.cc/convert.zip)

## 6.安装pytorch

## 7.安装cupy

## 8.安装openai whisper

```bash
pip install openai-whisper
```

## 9.安装whisperx

```bash
pip install whisperx
```

## 10.安装chardet

```bash
pip install chardet
```

## 11.安装pydub

```bash
pip install pydub
```

## 12.升级pyannote.audio

```bash
pip install --upgrade pyannote.audio
```

## 13.替换Moviepy的Cupy加速版

[moviepy](https://attach.xcopter.cc/moviepy.zip)

## 正常运行的各类依赖的版本

```
C:\Users\Administrator>pip list
Package Version
----------------------- ------------
aiohappyeyeballs 2.4.0
aiohttp 3.10.5
aiosignal 1.3.1
alembic 1.13.2
antlr4-python3-runtime 4.9.3
asteroid-filterbanks 0.4.0
attrs 24.2.0
audioread 3.0.1
av 11.0.0
certifi 2024.8.30
cffi 1.17.1
chardet 5.2.0
charset-normalizer 3.3.2
click 8.1.7
colorama 0.4.6
coloredlogs 15.0.1
colorlog 6.8.2
contourpy 1.3.0
ctranslate2 4.4.0
cupy-cuda12x 13.3.0
cycler 0.12.1
decorator 4.4.2
docopt 0.6.2
einops 0.8.0
faster-whisper 1.0.1
fastrlock 0.8.2
filelock 3.16.0
flatbuffers 24.3.25
fonttools 4.53.1
frozenlist 1.4.1
fsspec 2024.9.0
greenlet 3.1.0
huggingface-hub 0.24.7
humanfriendly 10.0
HyperPyYAML 1.2.2
idna 3.10
imageio 2.35.1
imageio-ffmpeg 0.5.1
Jinja2 3.1.4
joblib 1.4.2
julius 0.2.7
kiwisolver 1.4.7
lazy_loader 0.4
librosa 0.10.2.post1
lightning 2.4.0
lightning-utilities 0.11.7
llvmlite 0.43.0
Mako 1.3.5
markdown-it-py 3.0.0
MarkupSafe 2.1.5
matplotlib 3.9.2
mdurl 0.1.2
more-itertools 10.5.0
moviepy 1.0.3
mpmath 1.3.0
msgpack 1.1.0
multidict 6.1.0
networkx 3.3
nltk 3.9.1
numba 0.60.0
numpy 1.26.4
omegaconf 2.3.0
onnxruntime 1.19.2
openai-whisper 20231117
optuna 4.0.0
packaging 24.1
pandas 2.2.2
pillow 10.4.0
pip 24.2
platformdirs 4.3.3
pooch 1.8.2
primePy 1.3
proglog 0.1.10
protobuf 5.28.1
pyannote.audio 3.1.1
pyannote.core 5.0.0
pyannote.database 5.1.0
pyannote.metrics 3.2.1
pyannote.pipeline 3.0.1
pycparser 2.22
pydub 0.25.1
Pygments 2.18.0
pyparsing 3.1.4
pyreadline3 3.5.2
python-dateutil 2.9.0.post0
pytorch-lightning 2.4.0
pytorch-metric-learning 2.6.4
pytz 2024.2
PyYAML 6.0.2
regex 2024.9.11
requests 2.32.3
rich 13.8.1
ruamel.yaml 0.18.6
ruamel.yaml.clib 0.2.8
safetensors 0.4.5
scikit-learn 1.5.2
scipy 1.14.1
semver 3.0.2
sentencepiece 0.2.0
setuptools 75.1.0
shellingham 1.5.4
six 1.16.0
sortedcontainers 2.4.0
soundfile 0.12.1
soxr 0.5.0.post1
speechbrain 1.0.1
SQLAlchemy 2.0.35
sympy 1.13.2
tabulate 0.9.0
tensorboardX 2.6.2.2
threadpoolctl 3.5.0
tiktoken 0.7.0
tokenizers 0.15.2
torch 2.4.1+cu124
torch-audiomentations 0.11.1
torch-pitch-shift 1.2.4
torchaudio 2.4.1
torchmetrics 1.4.2
torchvision 0.19.1+cu124
tqdm 4.66.5
transformers 4.39.3
typer 0.12.5
typing_extensions 4.12.2
tzdata 2024.1
urllib3 2.2.3
whisperx 3.1.5
yarl 1.11.1
```
