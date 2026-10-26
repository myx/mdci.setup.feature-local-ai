# MAGIC.md — setup.feature-local-ai

Maintainer notes for the optional local AI feature. Users read `README.md`. How the packages compose: `../MAGIC.md`.

## What is true of the mechanism

- Parts are tokens of one command, `mdci-local-ai` (`data/local-ai/mdci-local-ai`). The deploy fragments do no work of their own: they call the staged copy at `${MDCI_FILES_PATH}/local-ai/mdci-local-ai`. The installed copy in `/usr/local/sbin` is the same file.
- The feature declares nothing a project must supply and installs nothing by default. `MDCI_LOCAL_AI_PARTS` empty means folders, user, config and the command only; no package, no download.
- Deploys are additive. A part leaves only by `MDCI_LOCAL_AI_REMOVE` or by hand. A part dropped from `MDCI_LOCAL_AI_PARTS` stays installed.
- The project has no `Requires:` (like `setup.feature-monit`). It does not call `myx.common`, so it need not require `standard/setup.feature-myx.common`. It makes the service user with `pw` and installs its own rc scripts from `data/local-ai/rc.d/`, so it does not depend on the floor's `os-services-rc.d` contract having run first.
- `data/local-ai` uses `sync-source-files:*`, the project and its dependents, like the other feature contracts.

## One fixed data folder

- `/var/db/mdci-local-ai` is a constant, not a variable. The model list path in the README is a contract that tooling reads without knowing the install variables, so a variable path would break it. A different disk is a mount or a link.
- Nothing is written to a home folder, and nothing to `/tmp`: downloads go to `<file>.part` beside the target, the probe WAV to `run/`, rewritten files to `*.new` in the same folder and renamed.
- `models.json` and `models.ini` are written whole and renamed, so a reader never sees a half file. Both are derived: only a model with its file present and its `state/<id>.ok` marker counts as installed. The marker is written only after the checksum matched.

## Checksums

- `models.tsv` carries the SHA-256 and size that the publisher lists (the Hugging Face LFS object ids, read 2026-10-26). A changed row needs both fields read again from the publisher; a hash taken from a download is not a reference.
- A mismatch removes the partial file and fails. A `.part` of exactly the expected size is hashed without a new download; a shorter one is resumed.

## Engines and packages

- FreeBSD: `misc/llama-cpp` (build b11669 when read) ships `bin/llama-server` and its own rc script `llama-server`; `audio/whisper.cpp` (1.8.4_2) ships `bin/whisper-server` and `bin/whisper-cli` and no rc script. The port rc script is not used: it runs one model, and our scripts need the dedicated user, the loopback bind and the router preset. Its `llama_server_enable` is never set.
- macOS: Homebrew `llama.cpp` and `whisper.cpp`. Homebrew refuses root, so the command runs `brew` as `SUDO_USER`. No launchd job is shipped; `serve` is run by hand.
- Linux (the Ubuntu floor, `setup.machine-mgpu`): not supported by this command yet. `setup.machine-mgpu` serves Ollama today (see Open).
- Prompt-cache flags (`--cache-reuse`, `--slot-save-path`) and `--models-preset` are passed only when the installed `llama-server --help` lists them, so an older build still starts.

## Services

- `mdci_ai_chat` and `mdci_ai_stt` run `mdci-local-ai serve <part>` under `daemon(8)` as `mdci-ai` (uid 583), restarting after 10 seconds. `serve` exits 78 with a warning when the engine or a model is missing, so an enabled service with nothing to serve retries every 10 seconds and logs the reason, and does not spin.
- The service is enabled by `sysrc` when the engine is installed, and started only when a model of that part exists.
- Sizing comes from the profile at start time (`serve`), not at install time, so a profile change in `local-ai.conf` plus a restart applies. `LAI_CHAT_ARGS` and `LAI_STT_ARGS` in the config are appended last.
- Bind addresses are validated as loopback in `conf_write`. A non-loopback listen needs a code change, on purpose: no credentials exist on these endpoints.

## Check

- `check` exits 1 only when an installed part does not run or a model size differs. A part not installed is not a failure.
- `check --parse` prints the same facts as `key: value` lines on stdout (first line `local-ai: installed | not installed | broken`, then `host.*`, `engine.*` and, for every model of the manifest, `part.<id>.{part,state,disk_bytes,expected_bytes,memory_bytes,running}`). A model is installed only with its file of the manifest size and its marker; the file without the marker, the marker without the file and a wrong size are `broken`. The check fragment prints it when `MDCI_CHECK_PARSE=1`; the human report is unchanged. The console's `--manager-feature-list` reads it, and `install --dry-run --parse` and `uninstall --dry-run` give the plans (`plan.*` lines) it shows before an action. `ask <model-id>` and `transcribe [wav]` make one request to the running part and print the server's JSON; the prompt reaches curl on its stdin, never on a command line. `stt-engine` is a part token of its own (the whisper.cpp package and service, no model). The test is `myx.distro-agents/sh-test/config/AgentsLocalAiCheckParse.test.sh`, which runs a copy of the command with its data folder pointed into a rig.
- `--run` makes a real request: a one-token chat completion to the first installed chat model, and a one-second silent WAV (written by `printf` in the command) to `/inference`. The chat probe loads the model, so it can take long on a shared CPU.

## Naming

- Per `../MAGIC.md`: `mdci`, never a customer. The user, the rc names, the variables and the folder are `mdci-ai`, `mdci_ai_*`, `MDCI_LOCAL_AI_*`, `mdci-local-ai`.
- uid 583 and gid 583 follow the neighbours (`setup.machine-mgpu` 581 and 582, `setup.machine-mdci` 181). Nothing checks that 583 is free on a given host.
- Help pairing: the command has no `sh-lib` entry, so there is no `Help.*.help.md` pair; `mdci-local-ai help` prints its own syntax.
- A fragment comment holds no apostrophe (the generated installer passes through `bash 3.2`); messages follow the grammar in `setup.machine-mdci/MAGIC.md`.

## Verified and not verified

- Checked: `sh -n` and `bash -n` on every script; a dry run of `help`, `list --available`, `install --dry-run` and `check` on a Mac, which wrote nothing. The package names and versions above were read from FreshPorts and Homebrew.
- Not run anywhere: an install, a download, the rc scripts, `daemon -R`, a server answering. Nothing here has run on a FreeBSD host.

## Open

- The router preset keys: only `model = <path>` is written, and whether `-c`, `-np` and `-ngl` given on the command line reach each routed model is not established.
- `whisper-server` request fields (`file`, `response_format`) and `GET /` as a liveness answer are taken from upstream usage, not from the port.
- `daemon -R` (restart delay) is assumed on the target FreeBSD releases.
- Unix socket serving is not provided; whether `llama-server` can bind one is not established.
- `setup.machine-mgpu` runs Ollama on `127.0.0.1:11434` behind its TLS gateway; this feature serves llama.cpp on 8181. Whether the GPU machine takes this feature and drops Ollama is a decision for its owner.
- The LanceDB embedding model (`potion-base-8M`) is a different project and is not here.
- The Slack voice-message integration and the `local` wrapper are other builders; they read `models.json` and call the endpoints.
