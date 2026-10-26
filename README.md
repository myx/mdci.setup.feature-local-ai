# mdci.setup.feature-local-ai

Optional local AI on a machine: speech-to-text (whisper.cpp) and chat models (llama.cpp), served on loopback. Any project may take it. Each part is installed and removed on its own. A project that does not list a part gets none of it.

Maintainer notes: `MAGIC.md`. How the packages compose: `../MAGIC.md`.

## Requires

- Nothing in `Requires:`.
- FreeBSD is the first-class host. macOS (Homebrew) is second. Other systems are not supported yet.
- `curl` on the host. On FreeBSD `setup.machine-freebsd` already puts it in `OS_PACKAGES`.
- Root for the deploy and for `mdci-local-ai install` and `uninstall`.

## Parts

| part | installs | size |
|---|---|---|
| `stt` | package `whisper.cpp`, the service `mdci_ai_stt`, the model `stt-whisper-large-v3-turbo-q5_0` | 547 MB model |
| `chat-engine` | package `llama-cpp` (FreeBSD) or `llama.cpp` (Homebrew), the service `mdci_ai_chat`, no model | package only |
| `chat` | `chat-engine` plus the default chat model of the configured profile | see below |
| `<model-id>` | one model file, nothing else | see `models.tsv` |

Packages (checked against FreshPorts and Homebrew on 2026-10-26):

| engine | FreeBSD | macOS |
|---|---|---|
| llama.cpp (`llama-server`) | `pkg install llama-cpp` (port `misc/llama-cpp`) | `brew install llama.cpp` |
| whisper.cpp (`whisper-server`, `whisper-cli`) | `pkg install whisper.cpp` (port `audio/whisper.cpp`) | `brew install whisper.cpp` |

The FreeBSD `misc/ggml` library the two ports share is built with Vulkan on by default. A machine with no GPU runs the CPU path and works; only a build from ports with `VULKAN=OFF` in `misc/ggml` drops the Vulkan libraries.

### Chat models and the three machine profiles

Set the profile with `MDCI_LOCAL_AI_PROFILE`. The default model of each profile is marked `*`.

| profile | machine | default | other rows in `models.tsv` |
|---|---|---|---|
| `shared-cpu` | normal CPU, constrained or shared RAM | `qwen3.5-4b-q8_0` (4.2 GB) | none |
| `big-cpu` | dedicated or bigger CPU, more RAM | `qwen3.5-9b-q8_0` (8.9 GB) | none |
| `gpu` | the GPU machine (`setup.machine-mgpu`) | `qwen3.8-27b-q8_0` (27.1 GB) | none |

Each chat model in `models.tsv` is the default of one profile.

An install is refused when the host RAM is below the model size plus about 17 percent. `--force` overrides.

## Data and checksums

- Everything is in `/var/db/mdci-local-ai/`, a fixed path. It is never a home folder. To put it on another disk, mount a dataset or link the folder.
- `models/` chat models, `stt/` speech models, `state/` one `<id>.ok` per verified model, `cache/` the prompt cache, `run/` scratch, `local-ai.conf`, `manifest.tsv`, `models.ini`, `models.json`.
- Every model is downloaded over https to `<file>.part`, resumed after an interruption, checked against the SHA-256 in `data/local-ai/models.tsv`, and only then renamed. A mismatch deletes the file and fails the install.
- `mdci-local-ai check --verify` hashes every installed model again.

## Inputs

All optional. Set them in the project that takes this feature.

| variable | default | effect |
|---|---|---|
| `MDCI_LOCAL_AI_PARTS` | empty | parts to install on deploy, space separated, for example `stt chat` |
| `MDCI_LOCAL_AI_REMOVE` | empty | parts to remove on deploy, same names; nothing is removed unless named |
| `MDCI_LOCAL_AI_PROFILE` | `shared-cpu` | `shared-cpu`, `big-cpu` or `gpu` |
| `MDCI_LOCAL_AI_CHAT_ADDR` | `127.0.0.1:8181` | chat listen address; loopback only |
| `MDCI_LOCAL_AI_STT_ADDR` | `127.0.0.1:8182` | speech listen address; loopback only |

## What a deploy does

Steps in `host/install/`, in order:

- `0.uninstall` removes what `MDCI_LOCAL_AI_REMOVE` names.
- `1.install` creates the folders, the user `mdci-ai`, `local-ai.conf`, `/usr/local/sbin/mdci-local-ai`, then installs what `MDCI_LOCAL_AI_PARTS` names. A failed package or download is a warning; the deploy goes on.
- `2.check` (after all other steps) prints the same report as `mdci-local-ai check`. It never stops a deploy.

## Commands

On the host, `/usr/local/sbin/mdci-local-ai`:

| command | does |
|---|---|
| `parts` | the part names |
| `list`, `list --available`, `list --json` | installed models, all known models, the model list file |
| `install [--dry-run] [--force] <part\|model-id>...` | `--dry-run` shows the packages, the download size, the free space and the RAM, and changes nothing |
| `uninstall [--purge] <part\|model-id>...` | `stt` removes package, service and speech models. `chat-engine` removes package and service, keeps the models. `chat` removes the engine and all chat models. `all` removes everything. `--purge` also deletes the data folder |
| `check [--run] [--verify]` | what is installed, the sizes, whether each part runs; `--run` makes a real request, `--verify` rehashes. Exit 1 when an installed part does not run |
| `start`, `stop`, `restart`, `status [stt\|chat]` | FreeBSD services; on macOS run `serve` yourself |
| `serve stt\|chat` | the server in the foreground; the rc scripts run this |
| `refresh` | rewrite `models.json` and `models.ini` |

Services on FreeBSD: `service mdci_ai_chat|mdci_ai_stt start|stop|status`, enabled with `sysrc` when the part is installed, run as `mdci-ai`.

## Contract for the `local` CLI service

Loopback only. Nothing here listens on a non-loopback address; the address variables refuse anything else.

| part | endpoint | API |
|---|---|---|
| chat | `http://127.0.0.1:8181/v1` | OpenAI-compatible: `POST /v1/chat/completions`, `GET /v1/models`; `GET /health` is the liveness probe. The request `model` is a model id from `models.json` |
| stt | `http://127.0.0.1:8182/inference` | whisper-server: `POST` multipart with `file` (a WAV, 16 kHz mono is the safe form) and `response_format=json`; the reply has `text` |

- Unix sockets: not offered. `llama-server` and `whisper-server` are used on loopback TCP only. A wrapper that needs a socket puts its own proxy in front.
- No credentials are used or stored. The loopback bind is the protection; anything beyond loopback is a decision for the project that takes the feature.
- Chat serves every installed chat model from one endpoint (llama.cpp router mode, `--models-preset /var/db/mdci-local-ai/models.ini`) where the installed `llama-server` has it, else the profile default model alone.

### The model list

One file at a fixed path: `/var/db/mdci-local-ai/models.json`, world readable, rewritten by every install, uninstall, start, stop and `refresh`.

```json
{
  "schema": "mdci-local-ai/1",
  "data": "/var/db/mdci-local-ai",
  "profile": "shared-cpu",
  "endpoints": {
    "chat": {"url": "http://127.0.0.1:8181/v1", "health": "http://127.0.0.1:8181/health", "api": "openai", "state": "running"},
    "stt": {"url": "http://127.0.0.1:8182/inference", "api": "whisper-server", "state": "absent"}
  },
  "models": [
    {"id": "qwen3.5-4b-q8_0", "part": "chat", "family": "qwen3.5", "params": "4B", "quant": "q8_0",
     "file": "Qwen3.5-4B-Q8_0.gguf", "path": "/var/db/mdci-local-ai/models/Qwen3.5-4B-Q8_0.gguf",
     "size_bytes": 4482403488, "sha256": "10cc391b...", "profiles": "shared-cpu*", "installed": true}
  ],
  "available": [{"id": "qwen3.5-9b-q8_0", "part": "chat", "size_bytes": 9527502048, "profiles": "big-cpu*"}]
}
```

- `models` holds only installed, checksum-verified files. `available` is what `models.tsv` offers and the host lacks.
- `state` is `absent` (engine not installed), `stopped` or `running` as of the last rewrite. A caller that needs the live state asks `health`.
- A read from outside the host: `ssh <host> cat /var/db/mdci-local-ai/models.json`.

## Reading a model sheet

`data/local-ai/models.tsv`: `id`, `part`, `family`, `params`, `quant`, `profiles` (`*` marks a default), `file`, `size_bytes`, `sha256`, `url`. To add a model add a row with the hash from the publisher.
