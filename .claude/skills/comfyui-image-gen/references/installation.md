# ComfyUI Image Generation Setup

## 1. A running ComfyUI instance

`cli.exe` (and the server-based `generate-comfy.mjs`) talk to a ComfyUI instance over its websocket. ComfyUI must be running and reachable — by default at `127.0.0.1:8188`.

- Point elsewhere with `--comfyui-host ip:port` or the `COMFYUI_HOST` environment variable.
- No persistent wrapper server is needed for `cli.exe` — only ComfyUI itself.

## 2. The cli.exe binary

`cli.exe` is not committed to this repo. Fetch the prebuilt Windows bundle, or run from source on Linux/macOS — see **Getting cli.exe** in `SKILL.md`.

## 3. Workflows (Export (API) format)

Workflow `.json` files live in the `workflows/` folder beside `cli.exe` (`.claude/skills/comfyui-image-gen/workflows/`), or wherever `--workflow-dir` / the `COMFYUI_WORKFLOW_DIR` env var points.

- They **must** be exported from the ComfyUI UI via **Export (API)** (enable Settings → Dev mode if you don't see it) — the normal **Export** produces an incompatible UI-graph format.
- Run `cli.exe --list-workflows` to confirm what's available.
- Workflows depend on the models you have installed locally, so they are not shipped with the binary — supply your own.

## 4. (Optional) The comfyui-api server

The server-based fallback (`generate-comfy.mjs`) needs the **comfyui-api** FastAPI server running on `127.0.0.1:5000`. This is only required if you prefer the server path over `cli.exe`. See [Oratorian/comfyui-api](https://github.com/Oratorian/comfyui-api).
