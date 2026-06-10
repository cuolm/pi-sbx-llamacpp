# Pi Coding Agent + Docker Sandbox + llama-server: Setup Guide

This guide runs the Pi coding agent inside an isolated Docker Sandbox (sbx)
microVM, with inference served by a local llama-server on the host machine.
The microVM provides hypervisor-level isolation. Pi cannot access host files
outside the mounted workspace, cannot reach the host keychain or SSH keys, and
cannot make network requests beyond what is explicitly permitted. The model
runs on the host GPU at full speed and is not exposed to the network.

Official documentation links:
- sbx: https://docs.docker.com/ai/sandboxes/get-started/
- Pi coding agent: https://pi.dev / https://github.com/earendil-works/pi
- llama.cpp: https://github.com/ggml-org/llama.cpp

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│ HOST (macOS / Linux)                                                 │
│                                                                      │
│  llama-server                                                        │
│  127.0.0.1:8001                                                      │
│                                                                      │
│  sbx host-side proxy                                                 │
│  • allowedDomains enforcement                                        │
│  • forwards allowed traffic to host services                         │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │ microVM (Linux, isolated)                                    │    │
│  │                                                              │    │
│  │  Docker Engine                                               │    │
│  │  ┌────────────────────────────────────────────────────────┐  │    │
│  │  │ Pi coding agent container                              │  │    │
│  │  └────────────────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```
**How Pi reaches llama-server across the VM boundary:**

Inside the microVM, `localhost` refers to the VM itself. 
`host.docker.internal` resolves to your host machine via the sbx gateway.
Requests to it travel through the sbx proxy that runs on the host machine.
Only destinations listed in `allowedDomains` are forwarded, everything else
is blocked.
---

## Step 1 — Install sbx

sbx is Docker's standalone CLI for running AI agents in isolated microVMs. It
does not require Docker Desktop.

**macOS**

```bash
brew install docker/tap/sbx
sbx login
```

**Linux (Ubuntu/Debian)**

```bash
curl -fsSL https://get.docker.com | sudo REPO_ONLY=1 sh
sudo apt-get install docker-sbx
sudo usermod -aG kvm $USER
newgrp kvm
sbx login
```

The `kvm` group membership is required on Linux because sbx uses KVM for
hardware-accelerated microVM isolation.

On first login, sbx prompts you to choose a default network policy. Select
**Balanced** — it allows common development traffic while blocking everything
else by default.

For other Linux distributions or manual binary installation, see:
https://docs.docker.com/ai/sandboxes/get-started/

---

## Step 2 — Install llama.cpp

**macOS**

```bash
brew install llama.cpp
```

**Linux**

Homebrew also works on Linux:

```bash
brew install llama.cpp
```

Alternatively, build from source:

```bash
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build
cmake --build build --config Release
sudo cmake --install build
```

On Linux with an Nvidia GPU, add `-DGGML_CUDA=ON` to the first cmake command
to enable CUDA acceleration. For AMD GPUs use `-DGGML_ROCM=ON`.

For all installation options see: https://github.com/ggml-org/llama.cpp

---

## Step 3 — Start llama-server on the host

llama-server exposes an OpenAI-compatible HTTP API. On macOS it uses Metal
for GPU acceleration. On Linux it uses CUDA or ROCm depending on your GPU.
The microVM has no access to the host GPU, which is why inference runs on the
host machine.

```bash
llama-server \
    --hf-repo unsloth/gemma-4-12B-it-qat-GGUF:UD-Q4_K_XL \
    --no-mmproj \
    --reasoning on \
    --temp 1.0 \
    --top-p 0.95 \
    --top-k 64 \
    --alias "unsloth/gemma-4-12B-it-qat-GGUF" \
    --port 8001
```

---

## Step 4 — Create the kit directory

An sbx kit is a directory containing a `spec.yaml` and an optional `files/`
subtree. Files placed under `files/home/` are injected into the agent's home
directory inside the microVM at sandbox creation time, before any install
commands run.

A kit can be placed anywhere on disk and referenced via the `--kit` flag when
running sbx. This guide places it at `~/.config/docker-sbx/pi-llamacpp`.

```bash
mkdir -p ~/.config/docker-sbx/pi-llamacpp/files/home/.pi/agent
```

Full directory layout:

```
~/.config/docker-sbx/pi-llamacpp/
├── spec.yaml
└── files/
    └── home/
        └── .pi/
            └── agent/
                ├── models.json
                └── settings.json
```

---

## Step 5 — Create spec.yaml

```yaml
schemaVersion: "1"
kind: agent
name: pi

agent:
  image: "docker/sandbox-templates:shell"
  entrypoint:
    run: [pi]

network:
  allowedDomains:
    - "registry.npmjs.org"
    - "host.docker.internal:8001"
    - "localhost:8001"

commands:
  install:
    - command: "npm install -g --ignore-scripts @earendil-works/pi-coding-agent"
```

Both `host.docker.internal:8001` and `localhost:8001` must be included in 
`allowedDomains` to authorize and route proxy traffic to the host.
`registry.npmjs.org` is needed for the npm install during sandbox creation.

---

## Step 6 — Create models.json

Pi reads `~/.pi/agent/models.json` inside the microVM to discover custom model
providers. Without this file it only knows about its built-in providers such as Claude.

Place this at `~/.config/docker-sbx/pi-llamacpp/files/home/.pi/agent/models.json`:

```json
{
  "providers": {
    "llamacpp": {
      "baseUrl": "http://host.docker.internal:8001/v1",
      "api": "openai-completions",
      "apiKey": "local-llama",
      "models": [
        {
          "id": "unsloth/gemma-4-12B-it-qat-GGUF",
          "name": "unsloth/gemma-4-12B-it-qat-GGUF",
          "input": ["text"]
        }
      ]
    }
  }
}
```

The `id` must exactly match the `--alias` value passed to llama-server. The
`baseUrl` uses `host.docker.internal` for the reason explained in the
Architecture section.

---

## Step 7 — Create settings.json

Pi creates `~/.pi/agent/settings.json` on first launch, but only writes
internal state into it (`lastChangelogVersion` and similar). It does not set
`defaultProvider` or `defaultModel`, so without this file in the kit Pi will
prompt you to select a provider and model interactively on every session.

By injecting `settings.json` via the kit's `files/` directory, Pi picks up
the defaults before it runs for the first time and never asks.

Place this at `~/.config/docker-sbx/pi-llamacpp/files/home/.pi/agent/settings.json`:

```json
{
  "defaultProvider": "llamacpp",
  "defaultModel": "unsloth/gemma-4-12B-it-qat-GGUF"
}
```

---

## Step 8 — Run the sandbox

From the project directory you want Pi to work in:

```bash
sbx run --kit ~/.config/docker-sbx/pi-llamacpp pi
```

On first run sbx creates the microVM, injects `files/home/` into `~/` inside
the VM, and runs the npm install. This takes about 30-60 seconds. On
subsequent runs it reattaches to the existing sandbox instantly.

> **Important:** The `files/` injection only happens at sandbox creation time.
> If you modify `models.json` or `settings.json` after the sandbox already
> exists, destroy it and recreate it:
>
> ```bash
> sbx rm <sandbox-name>
> sbx run --kit ~/.config/docker-sbx/pi-llamacpp pi
> ```
