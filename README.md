# ollama-searxng

Run [Open WebUI](https://openwebui.com/) backed by a **local, GPU-accelerated
LLM** and a private [SearXNG](https://github.com/searxng/searxng) metasearch
instance, orchestrated with [Lando](https://lando.dev/).

> [!NOTE]
> Despite the repo name, the LLM is **no longer served by Ollama**. It is served
> by a custom, source-built [`llama.cpp`](https://github.com/ggml-org/llama.cpp)
> fork ([`charlie12345/rocmfp4-llama`](https://github.com/charlie12345/rocmfp4-llama),
> branch `mtp-rocmfp4-strix`) that loads the community **ROCmFP4** quantised
> tensors and runs **MTP** (multi-token-prediction) self-speculative decoding on
> an **AMD Strix Halo** APU (gfx1151). It is built from
> [`tools/Dockerfile.rocmfp4`](tools/Dockerfile.rocmfp4); benchmarks live in
> [`docs/`](docs/).

## What is included?

| Name | Description | Image / build |
|------|-------------|---------------|
| [Open WebUI](https://openwebui.com/) | UI for chatting with the LLM | [`ghcr.io/open-webui/open-webui:main`](https://github.com/open-webui/open-webui/pkgs/container/open-webui) |
| [SearXNG](https://github.com/searxng/searxng) | Private metasearch engine | [`docker.io/searxng/searxng:latest`](https://hub.docker.com/r/searxng/searxng) |
| **llama.cpp (ROCmFP4 + MTP)** | OpenAI-compatible LLM server | **built** from [`tools/Dockerfile.rocmfp4`](tools/Dockerfile.rocmfp4) → `llama-rocmfp4-strix:latest` |
| [Valkey](https://github.com/valkey-io/valkey) | In-memory store for SearXNG | [`docker.io/valkey/valkey:8-alpine`](https://hub.docker.com/r/valkey/valkey) |
| [Firecrawl](https://github.com/firecrawl/firecrawl) | Self-hosted scrape/extract API (`:3002`, `https://firecrawl.lndo.site`) | [`ghcr.io/firecrawl/firecrawl:latest`](https://github.com/firecrawl/firecrawl/pkgs/container/firecrawl) + Playwright, Redis, RabbitMQ, `nuq-postgres` |

Firecrawl is what Hermes uses for `web_extract`. Keep SearXNG for search. Point Hermes at the published port (do **not** `lando rebuild` the whole app — that rebuilds `llama-cpp` from source):

```yaml
# ~/.hermes/config.yaml
web:
  backend: searxng
  extract_backend: firecrawl
```

```bash
# ~/.hermes/.env
FIRECRAWL_API_URL=http://localhost:3002
```

Rebuild only Firecrawl services after changing them:

```sh
lando rebuild -s firecrawl -s firecrawl-playwright -s firecrawl-redis -s firecrawl-rabbitmq -s firecrawl-postgres -y
```

## The LLM service (`llama-cpp`)

The `llama-cpp` Lando service **builds** the ROCmFP4 fork from source instead of
pulling a stock image, because the `Q4_0_ROCMFP4` / `Q4_0_ROCMFP4_FAST` tensor
types used by the [plunderstruck](https://huggingface.co/plunderstruck) GGUFs
**cannot load in upstream llama.cpp**. The image (`ubuntu:26.04` + TheRock ROCm
7.13) compiles both the HIP and Vulkan backends; at runtime the math runs on
**Vulkan** (`-dev Vulkan0`, RADV) and ROCm is present only so the HIP-linked
binary can initialise.

Default model:
[`plunderstruck/Qwen3.6-35B-A3B-MTP-ROCmFP4-GGUF`](https://huggingface.co/plunderstruck/Qwen3.6-35B-A3B-MTP-ROCmFP4-GGUF)
— a 35B-A3B MoE with a built-in MTP head, served with MTP self-speculative
decoding on. It exposes an OpenAI-compatible API on port **8395**
(`/v1/...`, alias `qwen3.6-35b-a3b-mtp`).

### Requirements

- An **AMD Strix Halo** (Ryzen AI Max/Max+) APU, ISA `gfx1151`, with `/dev/kfd`
  and `/dev/dri` present. This image is **not** portable to other GPUs/arches.
- [Docker](https://docs.docker.com/engine/install/) and [Lando](https://lando.dev/download/) v3.
- The ROCmFP4 GGUF downloaded into your Hugging Face cache (see below).
- ~128 GB unified RAM recommended; weights and KV are pinned into GTT
  (`GGML_HIP_ENABLE_UNIFIED_MEMORY` / `GGML_VK_PREFER_HOST_MEMORY`), not the
  small VRAM carve-out.

> [!WARNING]
> The first `lando start`/`lando rebuild` **builds the fork from source and
> pulls a pinned TheRock ROCm SDK** — this takes a while and produces a multi-GB
> image. Subsequent starts reuse the cached image.

## How to use it

1. [Install Lando](https://lando.dev/download/) and Docker.
2. Get ollama-searxng:
   ```sh
   cd /usr/local
   git clone https://github.com/thewunder/ollama-searxng.git
   cd ollama-searxng
   ```
3. Download the model into your Hugging Face cache:
   ```sh
   pip install -U "huggingface_hub[cli]"
   huggingface-cli download plunderstruck/Qwen3.6-35B-A3B-MTP-ROCmFP4-GGUF
   ```
   This lands under
   `~/.cache/huggingface/hub/models--plunderstruck--Qwen3.6-35B-A3B-MTP-ROCmFP4-GGUF/`
   with a `snapshots/<revision>/` folder and a shared `blobs/`.
4. Configure the LLM. Copy the template and edit it:
   ```sh
   cp .env.example .env
   ```
   Then reconcile the values with `.lando.base.yml` (see [Configuration](#configuration)
   below) — in particular the **snapshot revision hash**, your **GPU group IDs**,
   and the **models directory**.
5. Generate the SearXNG secret key:
   ```sh
   sed -i "s|ultrasecretkey|$(openssl rand -hex 32)|g" searxng/settings.yml
   ```
   On a Mac: `sed -i '' "s|ultrasecretkey|$(openssl rand -hex 32)|g" searxng/settings.yml`
6. Edit [`searxng/settings.yml`](https://github.com/searxng/searxng-docker/blob/master/searxng/settings.yml) to taste.
7. Start the stack (first run builds the LLM image, see the warning above):
   ```sh
   lando start
   ```
8. Point Open WebUI at the model: open the UI, then **Settings → Connections →
   OpenAI API** and add `http://llama-cpp:8395/v1` with any (non-empty) API key.
   The `OLLAMA_BASE_URL` in `.lando.base.yml` is a leftover and points at a
   service that no longer exists.

> [!NOTE]
> Windows users can generate the SearXNG secret key with:
> ```powershell
> $randomBytes = New-Object byte[] 32
> (New-Object Security.Cryptography.RNGCryptoServiceProvider).GetBytes($randomBytes)
> $secretKey = -join ($randomBytes | ForEach-Object { "{0:x2}" -f $_ })
> (Get-Content searxng/settings.yml) -replace 'ultrasecretkey', $secretKey | Set-Content searxng/settings.yml
> ```

## Configuration

The `.env` / [`.env.example`](.env.example) files document every knob of the LLM
container. **Lando does not read `.env`** — the values are baked into the
`llama-cpp` service in [`.lando.base.yml`](.lando.base.yml) (its `build.args`,
`command`, `volumes`, and `group_add`). `.env` is the source of truth if you
build/run `tools/Dockerfile.rocmfp4` directly with plain `docker`. When you
change a value, change it in **both** places.

Things you will likely need to change for your machine:

| What | `.env` key | Where in `.lando.base.yml` |
|------|-----------|----------------------------|
| Model / template path (the `snapshots/<hash>/` revision differs per download — check `ls ~/.cache/huggingface/hub/models--plunderstruck--*/snapshots/`) | `ROCMFP4_MODEL`, `ROCMFP4_CHAT_TEMPLATE` | `command:` (`--model`, `--chat-template-file`) |
| GPU group IDs (`getent group render` / `getent group video`) | `RENDER_GID`, `VIDEO_GID` | `group_add:` |
| Models directory mounted at `/models` | `MODELS_DIR` | `volumes:` |
| Context window, alias, port | `ROCMFP4_CTX`, `ROCMFP4_ALIAS`, `ROCMFP4_PORT` | `command:` / `ports:` |

To serve a **different** ROCmFP4 model (e.g. the dense 27B builds in
[`docs/qwen3.6-27b-mtp-rocmfp4.md`](docs/qwen3.6-27b-mtp-rocmfp4.md)), download
it, then point `--model` / `--chat-template-file` in the `command:` at the new
GGUF and re-run `lando rebuild`.

Vision (the Qwen3-VL projector) is off by default; enable it by adding
`--mmproj /models/snapshots/<hash>/mmproj-F32.gguf` to the `command:`.

### Verify the model is in GTT (not VRAM)

After the server is healthy, confirm the weights landed in unified RAM:

```sh
LLM_PORT=8395 scripts/verify-gtt.sh --min-gtt-mib 18000
```

## Start with systemd

You can skip this if you don't use systemd. The template is a **user** unit
(runs as your account, not root) so Lando can reach your Docker socket.

1. Copy the service template into your user systemd directory:
   ```sh
   mkdir -p ~/.config/systemd/user
   cp open-webui.service.template ~/.config/systemd/user/ollama-searxng.service
   ```
2. Edit `WorkingDirectory` in that file if the repo is not at
   `~/IdeaProjects/ollama-searxng`.
3. Enable lingering (starts the unit at boot without a graphical login) and
   enable the unit:
   ```sh
   loginctl enable-linger "$USER"
   systemctl --user daemon-reload
   systemctl --user enable --now ollama-searxng.service
   ```

Check status with `systemctl --user status ollama-searxng.service`. Logs:
`journalctl --user -u ollama-searxng.service -f`.

## Update

Run a Lando rebuild to update Open WebUI, SearXNG, and Valkey, and to rebuild
the LLM image against the pinned fork/ROCm versions:

```sh
lando rebuild
```
