# whisper-cpp

Container images with the command-line tools of [whisper.cpp](https://github.com/ggml-org/whisper.cpp), a C/C++ implementation of OpenAI's Whisper speech recognition model. `whisper-cli` transcribes speech, or translates it into English, and writes text, subtitles or JSON. It runs on the CPU and needs a model file, which is not in the image. The default image also contains FFmpeg. The images are rebuilt when whisper.cpp publishes a release and when the base image changes, for `linux/amd64` and `linux/arm64`.

This is an unofficial build, not affiliated with or endorsed by the whisper.cpp project or OpenAI. Report problems with the image in this repository and problems with whisper.cpp itself [upstream](https://github.com/ggml-org/whisper.cpp/issues).

## Quick start

Download a model into `models/`, then transcribe a file in the current directory:

```sh
mkdir -p models
curl -fL -o models/ggml-base.en.bin \
  https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-base.en.bin
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/whisper-cpp -m models/ggml-base.en.bin -f input.wav
```

whisper-cli prints the transcript with timestamps. `-osrt -of input` also writes it to `input.srt`; `-ovtt`, `-otxt`, `-ocsv`, `-olrc` and `-oj` (JSON) write the other formats, and several can be given at once:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/whisper-cpp -m models/ggml-base.en.bin -f input.wav -osrt -of input
```

`ggml-base.en.bin` is an English-only model. For other languages, use a multilingual model such as `ggml-base.bin` and name the language with `-l`, or pass `-l auto` to detect it. `-tr` translates the speech into English:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" \
  ghcr.io/randomcontainers/whisper-cpp -m models/ggml-base.bin -l de -tr -f input.mp3 -otxt -of input.en
```

whisper-cli reads WAV, MP3, FLAC and Ogg Vorbis files and resamples them to 16 kHz itself. For video files and other audio formats such as AAC and Opus, the default image has FFmpeg: convert the audio to a WAV stream and pipe it to whisper-cli, which reads it with `-f -`:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD:/work" --entrypoint sh \
  ghcr.io/randomcontainers/whisper-cpp -c \
  'ffmpeg -loglevel error -i input.mp4 -vn -ar 16000 -ac 1 -f wav - | whisper-cli -m models/ggml-base.en.bin -f - -osrt -of input'
```

`whisper-server` does the same over HTTP. It has no authentication, and any client can make it load another model file through `/load`, so publish its port on localhost only. With `--convert`, which needs the default image, it converts each upload with FFmpeg first:

```sh
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD/models:/models:ro" -p 127.0.0.1:8080:8080 \
  --entrypoint whisper-server ghcr.io/randomcontainers/whisper-cpp \
  -m /models/ggml-base.en.bin --host 0.0.0.0 --port 8080 --convert
curl -F file=@input.m4a -F response_format=srt http://127.0.0.1:8080/inference
```

## Models

The images contain no model. The whisper.cpp model files, converted from OpenAI's Whisper models, are on Hugging Face in [ggerganov/whisper.cpp](https://huggingface.co/ggerganov/whisper.cpp/tree/main). Upstream's [models/README.md](https://github.com/ggml-org/whisper.cpp/blob/master/models/README.md) lists them with their size and SHA-1; check a download with `sha1sum`. Models with `.en` in the name are English only. Larger models are more accurate and slower. The quantized files, whose names end in `-q5_0`, `-q5_1` or `-q8_0`, are smaller and faster than the full ones, with a small loss of accuracy.

Without `-m`, whisper-cli loads `models/ggml-base.en.bin` from the working directory. A model directory can also be mounted on its own and read-only, for example `-v "$HOME/whisper-models:/models:ro"` with `-m /models/ggml-base.en.bin`.

## What is in the image

| Area | Details |
|---|---|
| Tools | `whisper-cli`, `whisper-server`, `whisper-bench`, `whisper-quantize`, `whisper-vad-speech-segments`, and `parakeet-cli` and `parakeet-quantize` for NVIDIA Parakeet models |
| Audio input | WAV, MP3, FLAC and Ogg Vorbis, decoded by miniaudio and stb_vorbis and resampled to 16 kHz; `-f -` reads from stdin |
| Output | Text, SRT, VTT, CSV, LRC and JSON |
| CPU backend | ggml's CPU backend, compiled for each x86-64 level from the baseline to AVX-512 and AMX, and for Armv8.0 to Armv9.2 with dot product, SVE and SME instructions. The tools load the build that suits the CPU when they start, and run on OpenMP threads (`-t`, at most 4 by default) |
| Default image | FFmpeg (`ffmpeg` and `ffprobe`), for other audio and video formats and for `whisper-server --convert` |

Not included: models, GPU backends such as CUDA, Vulkan, Metal and OpenVINO, the tools that record from a microphone through SDL2 (`whisper-stream`, `whisper-command`, `whisper-talk-llama`), the model download scripts, and the headers and CMake files of the libraries. The commit and the list of CPU backend builds are in `/usr/local/share/randomcontainers/whisper-cpp/buildinfo`.

## Default or slim

Use the default image (`latest`) for general work. It adds FFmpeg, so video files and audio formats that whisper-cli cannot read, such as AAC, M4A and Opus, are converted and transcribed in one container (see [Quick start](#quick-start)), and `whisper-server --convert` works. `slim` has the whisper.cpp tools and the libraries they need, without FFmpeg. Use it to build your own image, or when your audio is already WAV, MP3, FLAC or Ogg Vorbis.

The default image includes FFmpeg, which is licensed under the GNU General Public License (GPL-3.0-or-later). Use `slim` if your policy excludes GPL. The default image is also published as `ghcr.io/randomcontainers/whisper-cpp-ffmpeg`, built in the [whisper-cpp-ffmpeg](https://github.com/randomcontainers/whisper-cpp-ffmpeg) repository with the same contents and a different digest.

## Tags

`<version>` is a whisper.cpp release such as `1.9.4`. `<minor>` and `<major>` are its shorter forms, `1.9` and `1`, and follow the newest release in that series.

| Default (with FFmpeg) | Slim | Base |
|---|---|---|
| `latest`, `<version>`, `<minor>`, `<major>` | `slim`, `<version>-slim`, `<minor>-slim`, `<major>-slim` | Ubuntu |
| `ubuntu`, `<version>-ubuntu`, `<minor>-ubuntu`, `<major>-ubuntu` | `slim-ubuntu`, `<version>-slim-ubuntu`, `<minor>-slim-ubuntu`, `<major>-slim-ubuntu` | Ubuntu |
| `<version>-ubuntu26.04` | `<version>-slim-ubuntu26.04` | Ubuntu 26.04 |
| `alpine`, `<version>-alpine`, `<minor>-alpine`, `<major>-alpine` | `slim-alpine`, `<version>-slim-alpine`, `<minor>-slim-alpine`, `<major>-slim-alpine` | Alpine |
| `<version>-alpine3.24` | `<version>-slim-alpine3.24` | Alpine 3.24 |

The images are currently built on Ubuntu 26.04 and Alpine 3.24. Tags without a distro version move to the next distro release when the project does; tags ending in `ubuntu26.04` or `alpine3.24` stay on that release and are no longer rebuilt once the project moves to the next one. Every tag of the current whisper.cpp version, including the exact version, is rebuilt in place (see [Updates](#updates)), so pin a digest when you need the same bytes every time.

## Platforms

`linux/amd64` and `linux/arm64`, for both Ubuntu and Alpine. Both are compiled natively on GitHub-hosted runners, without emulation. The images use the CPU only. Since the CPU backend is built for every instruction set level, they run on any x86-64 or 64-bit Arm CPU and use AVX2, AVX-512, dot product or SVE instructions where the CPU has them.

## Files and permissions

The working directory is `/work`. The image runs as UID 1000, and any other UID works too: `HOME` is then `/`, and caches go to `/cache`, which anyone can write to. How to get output files owned by you depends on how you run containers:

| Runtime | Flag |
|---|---|
| Docker on Linux (rootful), GitHub Actions | `--user "$(id -u):$(id -g)"` |
| Rootless Podman | `--userns=keep-id` |
| Rootless Docker | `--user 0:0` (root in the container is your user on the host) |
| Docker Desktop on macOS or Windows | none, file ownership is mapped for you |

ggml loads its CPU backend from `/usr/local/lib`, and it also loads `libggml-*.so` files that it finds in the working directory. Do not run the tools in a directory that holds shared libraries you do not trust.

## Extending the slim image

Use a `slim` tag as the base for your own image. It has no FFmpeg, so FFmpeg updates do not rebuild it. The packages whisper.cpp needs are listed in `/usr/local/share/randomcontainers/whisper-cpp/runtime-deps`. This image carries a model and passes it to every run:

```dockerfile
FROM ghcr.io/randomcontainers/whisper-cpp:slim-ubuntu@sha256:...
COPY models/ggml-base.en.bin /models/ggml-base.en.bin
ENTRYPOINT ["tini", "--", "whisper-cli", "-m", "/models/ggml-base.en.bin"]
```

The image runs as UID 1000, so switch to `USER root` to install distro packages, then back to `USER 1000:1000`. The entrypoint of the slim image is `["tini", "--", "whisper-cli"]`. To pick up new whisper.cpp releases and base image fixes, let Dependabot or Renovate update the digest in your `FROM` line.

## Verifying

Each image has a build provenance attestation from this repository's GitHub Actions run, signed by the shared build workflow in `randomcontainers/ci`:

```sh
gh attestation verify oci://ghcr.io/randomcontainers/whisper-cpp:latest \
  --repo randomcontainers/whisper-cpp --signer-repo randomcontainers/ci
```

Images from `ghcr.io/randomcontainers/whisper-cpp-ffmpeg` are built in that repository, so verify them with `--repo randomcontainers/whisper-cpp-ffmpeg` and the same `--signer-repo`.

Each platform image also carries an SPDX SBOM that lists every distro package with its version:

```sh
docker buildx imagetools inspect ghcr.io/randomcontainers/whisper-cpp:latest --format '{{ json .SBOM }}'
```

whisper.cpp publishes no release tarballs, checksums or signed tags, so each version is pinned to the commit of its tag. The build clones the `v<version>` tag of [ggml-org/whisper.cpp](https://github.com/ggml-org/whisper.cpp) and stops unless the tag points to the commit recorded in `package.yml`, so a tag that is moved after the version was picked up fails the build. The commit is also in `/usr/local/share/randomcontainers/whisper-cpp/buildinfo`.

## Updates

The project checks the `v<version>` tags of [ggml-org/whisper.cpp](https://github.com/ggml-org/whisper.cpp) every 15 minutes. A release is picked up once its tag is 24 hours old. The new version and the commit of its tag are then committed to `package.yml` and the images are rebuilt. Only the newest release is built; tags of older versions stay as they were last built.

The images of the current version are also rebuilt when the Ubuntu or Alpine base image changes, the default ones when a new FFmpeg image is published, and all of them at least every 7 days, so distro security fixes reach the current tags.

## Building

```sh
docker build -f Dockerfile.ubuntu --target slim \
  --build-arg VERSION=<version> \
  --build-arg SOURCE_COMMIT=<commit from package.yml> \
  -t whisper-cpp:local .
```

Use `Dockerfile.alpine` for the Alpine image. `--build-arg JOBS=<n>` limits the number of parallel compile jobs. Most of the build time goes into compiling the CPU backend once for each instruction set level. The default image is generated from the `combos` entry in `package.yml` by [randomcontainers/ci](https://github.com/randomcontainers/ci).

## Licenses

whisper.cpp and the ggml library it includes are released under the [MIT license](https://github.com/ggml-org/whisper.cpp/blob/master/LICENSE). The tools also contain code that whisper.cpp bundles: miniaudio (MIT No Attribution or public domain, SPDX `MIT-0 OR Unlicense`), stb_vorbis (MIT or public domain, used here under MIT), and in `whisper-server` cpp-httplib and nlohmann/json (both MIT). The license texts are in `/usr/local/share/randomcontainers/whisper-cpp/licenses/`, and links to the source are in `/usr/local/share/randomcontainers/whisper-cpp/source`.

Models are not part of the images and come with their own licenses. OpenAI released the Whisper models under the MIT license; check the terms of any other model you use.

The default image adds FFmpeg, licensed under GPL-3.0-or-later; see [randomcontainers/ffmpeg](https://github.com/randomcontainers/ffmpeg#licenses) for its sources and license files. libstdc++, libgomp and the other libraries from Ubuntu or Alpine keep their own licenses. The SBOM lists them.

The files in this repository are available under the MIT license, see [LICENSE](LICENSE).

## Requesting a tool

To suggest another tool, use the [Request a tool](https://github.com/randomcontainers/.github/issues/new?template=tool-request.yml) form.
