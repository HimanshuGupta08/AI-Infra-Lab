# AI Infra Lab — Local Setup

## 0. Prereqs — Podman (recommended for this hardware)
- Install **Podman Desktop** for Windows (uses WSL2 under the hood, lighter VM than Docker Desktop).
- It bundles `podman` CLI. Also install `podman-compose` for compose-file support:
```bash
pip install podman-compose --break-system-packages
```
  (or use `podman compose` directly — recent Podman versions detect and proxy to podman-compose/docker-compose automatically if present)
- Init + start the Podman machine (Windows needs a small VM, same as Docker, just leaner):
```bash
podman machine init
podman machine start
```
- Confirm: `podman --version` and `podman-compose --version`.
- No memory cap step needed like Docker Desktop — no persistent daemon hogging RAM when idle. The `podman machine` VM itself is small; default is usually fine, but if you want to set it explicitly: `podman machine init --memory 6144` (MB) on first init.

<details>
<summary>If you'd rather use Docker Desktop instead</summary>

- Docker Desktop for Windows, WSL2 backend (not Hyper-V — lighter on this hardware).
- In Docker Desktop settings: cap memory at ~10-11GB (leave headroom for Windows itself), CPU = all 4 threads.
- Confirm: `docker --version` and `docker compose version`.
- Swap `podman-compose`/`podman` for `docker compose`/`docker` in every command below — same compose file works for both.
</details>

## 1. Bring up the stack
```bash
cd ai-infra-lab
podman-compose up -d
podman ps      # all 4 should show "running"
```

## 2. Pull small models (CPU-appropriate)
```bash
podman exec -it ollama ollama pull llama3.2:3b
podman exec -it ollama ollama pull qwen2.5:3b
podman exec -it ollama ollama pull nomic-embed-text   # for RAG
```
Stick to 1B-3B for anything you want to actually chat with at usable speed. Skip 7B+ unless you're fine with slow generation just to see it work once.

## 3. Verify Ollama directly
```bash
curl.exe http://localhost:11434/api/generate -d '{"model":"llama3.2:3b","prompt":"say hi in 5 words"}'
```

## 4. Verify Qdrant
```bash
curl.exe http://localhost:6333/collections
```
Should return empty list, not an error.

## 5. Verify LiteLLM
```bash
curl.exe http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-local-master-changeme" \
  -d '{"model":"local-llama","messages":[{"role":"user","content":"hi"}]}'
```

## 6. Open WebUI
- Go to `http://localhost:3000`, sign up — first account = admin automatically.
- Settings → Connections: Ollama connection should already be live (`http://ollama:11434` via env var).
- Optional second connection: add LiteLLM as an "OpenAI API" connection — Base URL `http://litellm:4000/v1`, API key `sk-local-master-changeme`. Note: from inside the open-webui container, use the container name `litellm`, not `localhost`.
- Settings → Documents: set embedding model to `nomic-embed-text` (pulled in step 2).
- Workspace → Knowledge → create a KB, upload a short doc, ask it a question only that doc would know the answer to. Confirms RAG is wired through Qdrant, not just the base model guessing.

## 7. Resource discipline (important on this hardware)
- Don't load two models in Ollama at once — `ollama ps` to check what's resident, it'll auto-unload idle ones after a few minutes but watch RAM if you're impatient.
- If things feel sluggish: `podman stats` to see which container is actually eating CPU/RAM before assuming it's the model.
- When not actively using this, `podman-compose stop` (not `down` — keeps volumes/models) to give the laptop its RAM back.

## 8. Langfuse — skip self-hosting here
Self-hosted Langfuse needs Postgres + ClickHouse + Redis running alongside everything above — too heavy for this box. Use Langfuse Cloud free tier instead:
1. Sign up at cloud.langfuse.com, create a project, grab public/secret keys.
2. Uncomment `success_callback: ["langfuse"]` in `litellm_config.yaml`.
3. Add to the `litellm` service in `docker-compose.yml`:
```yaml
    environment:
      - LANGFUSE_PUBLIC_KEY=pk-...
      - LANGFUSE_SECRET_KEY=sk-...
```
4. `podman-compose up -d litellm` to restart with the new env. Every call routed through LiteLLM now traces to Langfuse Cloud — zero extra local load.

## 9. When you move to a real server/VPS later
Same compose file works as-is. Differences at that point:
- Add `--gpus=all` (or device_requests) to Ollama if the box has a GPU.
- Self-host Langfuse instead of cloud, since you'll have the RAM to spare.
- Put Nginx in front of Open WebUI / LiteLLM if exposing beyond localhost, same pattern as your Drawitz stack — separate compose, don't touch this one's networking.
- Swap `WEBUI_SECRET_KEY` and `master_key` for actual secrets — these are placeholder values for local-only use.
