---
name: local-llm-serving
description: Use when downloading, serving or calling a local LLM - model weights and quantization, vllm/sglang/llama.cpp/ollama options, no-GPU constraints, llama.cpp server integration from Node/TypeScript, offline fallback design.
---

Facts verified in the `bigass_ai` (download + launchers) and `storyteller`
(llama.cpp server + Node client + offline design) projects; paths below are
relative to those project roots. Hardware numbers are measurements from one
CPU-only host — a baseline to compare against, not a requirement.

## Weights on disk

- `bigass_ai/models/Huihui-Qwen3.8-27B-abliterated/` — huihui-ai/Huihui-Qwen3.8-27B-abliterated,
  base `Qwen/Qwen3.8-27B`, arch `Qwen3_5ForConditionalGeneration`, bf16, 55.6 GB in 18
  safetensors shards (`model-000NN-of-00018.safetensors`, largest ~3.98 GB) plus
  `model.safetensors.index.json`, `chat_template.jinja`, `tokenizer.json`.
  Disk: 156 GB free before, ~100 GB after.
- `storyteller/models/` (untracked; names fixed in `src/model/server.ts`):
  - `LFM2.5-1.2B-Instruct-Q4_K_M.gguf` — 730,895,168 B (~697 MB), default, RSS 0.77 GB.
  - `qwen2.5-3b-instruct-q4_k_m.gguf` — 2,104,932,768 B (~1.96 GB), RSS 2.17 GB.

## Downloading (bigass_ai)

`scripts/download.sh <repo_id> <dest_dir> [jobs]` (jobs default 4): lists
`https://huggingface.co/api/models/<repo>/tree/main?recursive=1` via python3 (skips
`.gitattributes`), then per file `curl -fL --retry 10 --retry-delay 3 --retry-all-errors -C - -o
"$out" "$BASE/<f>?download=true"` through `xargs -P $JOBS`. A local file is only skipped when
`stat -c%s` equals the API size exactly — truncated files are redone.

## Serving options actually offered — `bigass_ai/scripts/run.sh {vllm|sglang|llama|ollama}`

```bash
vllm)    exec vllm serve "$MODEL_DIR" --served-model-name uncensored --max-model-len 131072
sglang)  exec python3 -m sglang.launch_server --model-path "$MODEL_DIR" --host 0.0.0.0 --port 30000
llama)   exec llama-server -m "$MODEL_DIR" --ctx-size 131072 --n-gpu-layers 99
ollama)  echo "see NOTES.md for the GGUF conversion step"
```

## Hardware / quantization constraints (recorded in bigass_ai/NOTES.md)

- CPU-only host with **no GPU and 7.7 GB RAM** (measured): it cannot run the 27B
  model at all.
- bf16 ~56 GB weights → 80 GB A100/H100, or 5090 + 96 GB system RAM offload, or 2x RTX 4090.
- GGUF Q4_K_M ~16 GB → single 24 GB card; needs llama.cpp with `qwen3_5` support — check
  `llama-server --list-models | grep qwen3_5` first.
- GGUF Q8_0 ~29 GB → 32 GB card or 64 GB Mac Studio.
- vLLM needs ≥70 GB VRAM for bf16 (comment in run.sh); architecture is `Qwen3_5ForConditionalGeneration`,
  so vLLM/SGLang are the safer path than llama.cpp.
- For Ollama: take a prebuilt GGUF from the HF quantisation tab (`huihui-ai/...-GGUF`), don't convert.
- Example host (storyteller): MacBook Pro 2012, i5-2435M 2c/4t, **no AVX/AVX2, no GPU**, 7.7 GB RAM,
  4 GB swap. llama.cpp nightly **b11249** (v0.5.0-dev) CPU build, vendored at
  `storyteller/bin/llama-b11249/` (auto-picks `libggml-cpu-sandybridge.so`).
- Playability floor is speed, not size: ~5 chars/s; Qwen2.5-3B Q4 does 0.49 chars/s there.
  LFM2.5-1.2B: 2.7-3.5 t/s gen, warm turn 13-15 s; 3B: 1.1-1.3 t/s, 28-33 s.
- Budgets: 150 chars/turn, client gives up after 90 s. Probe real speed:

```bash
curl -s http://127.0.0.1:8080/completion -H 'Content-Type: application/json' \
  -d '{"prompt":"<|im_start|>user\nZeg een zin.<|im_end|>\n<|im_start|>assistant\n","n_predict":40}' \
  | python3 -c "import sys,json; t=json.load(sys.stdin)['timings']; print(round(t['predicted_per_second'],2),'tekens/s')"
```

## Starting the server (storyteller)

- Only `src/cli/serve.ts` spawns it: `npm run model:serve` (`node src/cli/serve.ts`),
  `--model 1b|3b` (default `1b`), `--port 8080`. If `GET /health` already answers, it exits
  ("llama-server draait al"). Frontends never start a model themselves.
- Spawn args (`src/model/server.ts`): `llama-server --model <file> --port 8080 --threads 4
  --ctx-size 2048` (1b) `| 4096` (3b) `--cache-reuse 256 --parallel 1`; stderr piped to a log
  file. `--cache-reuse 256` is what keeps warm turns at 13 s instead of 48 s (prefix cache:
  253 prompt tokens → 4).
- Readiness: poll `http://127.0.0.1:<port>/health` (800 ms timeout, 500 ms interval, 120 s
  limit), then stop = SIGTERM, SIGKILL only after 5 s.
- Manual/bench invocation (`bench/09-lfm-bench.sh`), port from `${LFM_PORT:-8098}`:

```bash
bin/llama-b11249/llama-server -m models/LFM2.5-1.2B-Instruct-Q4_K_M.gguf \
  -c 4096 -t 4 --host 127.0.0.1 --port "$PORT" --no-webui
```

- Raw numbers: `bin/llama-b11249/llama-bench -m models/qwen2.5-3b-instruct-q4_k_m.gguf
  -p 512,1024 -n 48 -r 1 -t 4` (`bench/01-llama-bench.sh`). Never pipe llama-bench through grep
  (buffering); always `curl --max-time` and `trap cleanup`; never two bench scripts at once.

## Client: Node → llama.cpp (src/model/client.ts)

- Base URL: `http://${STORYTELLER_MODEL_HOST ?? 127.0.0.1}:${STORYTELLER_MODEL_PORT ?? 8080}`.
- `GET /health` (1500 ms) = `healthy()`; `warmup()` does one 1-token completion.
- `POST /completion`, JSON, **not streaming** (`stream: false`), 90 s abort:

```json
{"prompt":"...", "n_predict":150, "temperature":0.85, "top_p":0.9,
 "cache_prompt":true, "stream":false, "grammar":"<GBNF JSON skeleton>"}
```

  Response read as `{ content, timings }`. Defaults in `complete()`: `n_predict` 60, temp 0.8.
  The GBNF grammar constrains **JSON structure only** (`{"narration":string,"mood":enum}`), no
  whitespace between fields — a prose grammar hung the server at 133% CPU for 8+ minutes.
- Errors: HTTP != 200 → `ModelError`; abort → "het model reageerde niet binnen de tijdlimiet";
  connection refused → "geen verbinding met llama-server op <base>. Start hem met: npm run model:serve".
- Requirement: the model **must** be llama.cpp `llama-server` (`/health` + `/completion`);
  Ollama and OpenAI-compatible APIs do not have those paths.
- Remote model: `STORYTELLER_MODEL_HOST=192.168.1.20 STORYTELLER_MODEL_PORT=8080 ./bin/storyteller`,
  preferably behind `ssh -N -L 8080:127.0.0.1:8080 user@other-machine` (llama-server has no password).

## Degrading without a model (by design, not a debug mode)

- Engine deps take `useModel: false`; free-text turns then return `describe(state, adventure)`
  (the generator's room text) with `usedModel: false`. Parser intents (move/take/drop/look/
  wait/help/inventory/attack) never touch the model.
- On `ModelError` (or any throw) the turn continues: narration = `describe(...)`, `mood: 'calm'`,
  `fallbackReason` set.
- `validateTurn` fallback (`src/engine/validate.ts`): pick the location's `room`/`template`
  fragment, else any generic `template` fragment, else `'Nothing comes of it.'`.
- Output is post-validated: max 45 words / two sentences, chat-prefix stripped, uppercase or
  player-action-echoing text replaced by a template.

## Tests without a live server

- `npm run check` = `tsc --noEmit && node --test tests/*.test.ts` — all green before handing back.
- Tests inject `useModel: false` and a stub `client` whose `complete` throws
  (`{ complete: async () => { throw new Error('mag niet'); } } as unknown as LlamaClient`);
  `tests/bridge.test.ts` asserts `oog.model.aanwezig === false` when no server runs.
- Related env vars: `STORYTELLER_BRIDGE_PORT` (bridge, default 3131), `STORYTELLER_SAVES`,
  `STORYTELLER_SNAPSHOT`, `STORYTELLER_INVOER`, `LFM_PORT` (bench).
