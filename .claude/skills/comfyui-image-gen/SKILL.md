---
name: comfyui-image-gen
description: Generate images by talking directly to a running ComfyUI instance via cli.exe (default, no server), or via the comfyui-api server.
---

# Image Generation

```bash
.claude/skills/comfyui-image-gen/cli.exe --workflow <workflow>.json --out images/generated/<name>.png --prompt "<prompt>"
```

Run from the project root. Set Bash timeout to **120000ms** (the **first** generation in a session is slow while the model loads; later ones are fast).

There are no built-in workflows. Before generating, **discover what workflows exist and let the user pick one** — see [Choosing a workflow](#choosing-a-workflow).

This is the default generator: **`cli.exe`** talks **directly to a running ComfyUI** instance (default `127.0.0.1:8188`) and writes straight to a file — **no persistent server is required, only ComfyUI itself must be running.**

> **Note:** `cli.exe` collects images from the workflow's `SaveImage` or `PreviewImage` nodes. If a workflow has only a `PreviewImage` node (no `SaveImage`), add `--preview` so the image can be collected — otherwise the run finishes with no output. (See *Auto-retry with `--preview`* under **Important**.)

One fallback remains:
- **`generate-comfy.mjs`** — the server-based path, for when the **comfyui-api** FastAPI server (`127.0.0.1:5000`) is already running. See [Server-based alternative](#server-based-alternative-generate-comfymjs).

## Getting cli.exe

`cli.exe` is **not committed to this repo** (it's a build artifact, gitignored). Its source lives at **[Oratorian/comfyui-api](https://github.com/Oratorian/comfyui-api)**, where CI publishes a prebuilt Windows bundle as a release asset: **`comfyui-api-cli-windows.zip`** (it contains `cli.exe` and an empty `workflows/` folder — workflows are not shipped, since they depend on locally installed models).

**Windows — fetch the prebuilt binary if missing.** If `.claude/skills/comfyui-image-gen/cli.exe` is absent, download the CLI zip from the latest release and extract `cli.exe` from it. The orchestrator may do this automatically when the binary is missing — it will always show the source URL first.

```bash
# from project root
mkdir -p .claude/skills/comfyui-image-gen
curl -L -o /tmp/comfyui-api-cli.zip \
  https://github.com/Oratorian/comfyui-api/releases/latest/download/comfyui-api-cli-windows.zip

# (optional) verify the zip — only works once CI publishes the matching .sha256 asset:
#   curl -L -o /tmp/comfyui-api-cli.zip.sha256 \
#     https://github.com/Oratorian/comfyui-api/releases/latest/download/comfyui-api-cli-windows.zip.sha256
#   echo "$(cat /tmp/comfyui-api-cli.zip.sha256)  /tmp/comfyui-api-cli.zip" | sha256sum -c -

# extract just cli.exe into the skill folder
unzip -o -j /tmp/comfyui-api-cli.zip cli.exe -d .claude/skills/comfyui-image-gen/
```

PowerShell equivalent:

```powershell
Invoke-WebRequest -Uri "https://github.com/Oratorian/comfyui-api/releases/latest/download/comfyui-api-cli-windows.zip" -OutFile "$env:TEMP\comfyui-api-cli.zip"
# (optional) verify once a .sha256 asset exists:
#   $expected = (Invoke-WebRequest -Uri "https://github.com/Oratorian/comfyui-api/releases/latest/download/comfyui-api-cli-windows.zip.sha256").Content.Trim().Split()[0]
#   if ((Get-FileHash "$env:TEMP\comfyui-api-cli.zip" -Algorithm SHA256).Hash.ToLower() -ne $expected) { throw "checksum mismatch — do not run" }
Expand-Archive -Path "$env:TEMP\comfyui-api-cli.zip" -DestinationPath "$env:TEMP\comfyui-api-cli" -Force
Copy-Item "$env:TEMP\comfyui-api-cli\cli.exe" ".claude\skills\comfyui-image-gen\cli.exe" -Force
```

**Checksum:** verify the downloaded zip with SHA256 when a `comfyui-api-cli-windows.zip.sha256` asset is available (the verify lines above, currently commented out). If the checksum fails, the download is corrupt or tampered — stop and re-fetch, do not run it.

**Linux / macOS — run from source.** There is no `.exe` for non-Windows. Clone [Oratorian/comfyui-api](https://github.com/Oratorian/comfyui-api), install its requirements, and call the Python entrypoint directly — it takes the same flags as `cli.exe`:

```bash
python cli.py --workflow <workflow>.json --out images/generated/<name>.png --prompt "<prompt>"
```

Everywhere `cli.exe` appears below, substitute `python cli.py` on Linux/macOS. The flags, defaults, and behavior are identical.

## Important

**Always run image generation directly via Bash.** Never delegate to a background agent (subagent) — the interactive approval workflow (generate → show → approve/retry) requires direct orchestrator control. Each `cli.exe` run is an independent subprocess that writes its own `--out` file, so concurrent runs are safe as long as each uses a distinct `--out` path.

**Auto-retry with `--preview` on a no-image result.** If a generation yields no usable image — `cli.exe` exits non-zero with *"produced no images"* and writes no file, **or** the written file is zero-byte / not a PNG / an all-black square — retry the exact same command with `--preview` added. The usual cause is a workflow whose only image output is a `PreviewImage` node (no `SaveImage`); `--preview` lets that image be collected. If it still fails with `--preview`, the workflow has no usable image-output node and the prompt/workflow itself needs attention — do not keep retrying blindly.

## Options (cli.exe — default)

| Flag | Required | Description |
|------|----------|-------------|
| `--prompt` | Yes | Positive prompt. |
| `--out` | No | Output file (default `output.png`). For portraits, write into `images/generated/<name>.png`. With `--all`, an index is inserted (`out.png` → `out_0.png`, `out_1.png`, …). |
| `--workflow` | No | Workflow file: a name resolved inside the workflows folder, or a path. Default `base_workflow.json`. The prompt format (Natural Language vs. Danbooru/Gelbooru tags vs. both) depends on the model the workflow loads — **ask the user** (see [Choosing a workflow](#choosing-a-workflow)). Must be **Export (API)** format. |
| `--workflow-dir` | No | Folder to resolve `--workflow` names against. Defaults to the `workflows/` folder beside `cli.exe` (`.claude/skills/comfyui-image-gen/workflows/`), or the `COMFYUI_WORKFLOW_DIR` env var if set. The explicit flag takes precedence over the env var. |
| `--list-workflows` | No | List the available workflow files (in `--workflow-dir` / the default workflows folder) and exit. Does not require `--prompt`. Prints a `workflow dir: <path>` header followed by the `.json` file names; errors with `workflow dir ... does not exist` if the folder is missing, or `... contains no .json workflow files` if it has none. |
| `--negative` | No | Negative prompt (default empty). Supply a painterly-leaning quality negative for the house look. |
| `--width` / `--height` | No | Output size override (px). Omit to use the workflow's own values. |
| `--batch` | No | Images per run override. |
| `--img2img` | No | Source image → image-to-image (the workflow needs a `LoadImage` node). |
| `--index` | No | Which image to save, in ComfyUI execution order. `-1` (default) = the last/final image. Ignored with `--all`. |
| `--all` | No | Save every collected image (with an index suffix), not just one. |
| `--preview` | No | Also collect images from `PreviewImage` nodes. Needed when a workflow's only image output is a `PreviewImage` node (no `SaveImage`). On its own it does not change selection — `--index`/`--all` still pick which collected image(s) to save. |
| `--comfyui-host` | No | ComfyUI host as `ip:port` (default `COMFYUI_HOST` env or `127.0.0.1:8188`). |

## Output (cli.exe)

- Writes the PNG to the `--out` path you give. For portraits, target `images/generated/<name>.png` so the upload/cleanup scripts can find it.
- **No auto-increment and no metadata JSON** — pick a unique `--out` name yourself (e.g. add a numeric suffix per variant). This is a difference from `generate-comfy.mjs`.
- It prints nothing automatically — use the **Read** tool on the `--out` path to view the result.

## Choosing a workflow

There are **no built-in workflows** — they depend on locally installed models, so the user supplies them. Never assume a workflow name. Before any generation:

1. **List what's available** with `--list-workflows`:

   ```bash
   .claude/skills/comfyui-image-gen/cli.exe --list-workflows
   ```

   This prints the `workflow dir` path followed by the `.json` files it contains.

2. **Ask the user which workflow to use.** Present the listed files and let them choose — do not pick for them.

3. **Ask the user what prompt format the model supports.** This cannot be inferred from the workflow name — it depends on the model the workflow loads. Ask explicitly whether the model supports:
   - **Natural Language** — descriptive prose sentences.
   - **Danbooru / Gelbooru tags** — comma-separated booru tags (`1girl, silver hair, armor, dramatic lighting, ...`).
   - **Both** — the model accepts either; you may mix prose with tags.

   Write the prompt in the format the user specifies. Do not assume — a mismatch (prose into a tag-only model, or tags into a prose-only model) produces poor results.

4. **Handle an empty or missing dir.** If `--list-workflows` reports the folder is missing (`workflow dir ... does not exist`) or contains no workflows (`... contains no .json workflow files`), tell the user — generation cannot proceed until a workflow exists. Instruct them to:
   - In ComfyUI, enable **dev mode**: Settings → enable **"Dev mode"** (this exposes the API export option).
   - Build or load the workflow they want, then export it via **File → Export (API)**.
   - Save the resulting `.json` into the workflows folder (`.claude/skills/comfyui-image-gen/workflows/`, or wherever `--workflow-dir` / `COMFYUI_WORKFLOW_DIR` points).
   - Re-run `--list-workflows` to confirm it appears, then ask which to use.

   Plain **Save** does **not** work — `cli.exe` requires the **Export (API)** format, which only appears once dev mode is enabled.

## Style & Workflow Notes

- Workflow files live in the **`workflows/` folder beside `cli.exe`** (`.claude/skills/comfyui-image-gen/workflows/`) unless overridden with `--workflow-dir` or the `COMFYUI_WORKFLOW_DIR` env var. They must be exported from ComfyUI in **Export (API)** format. Reference one by bare name (`--workflow <name>.json`) or by path. If the folder is missing, `cli.exe` reports it — create it or point `--workflow-dir`/`COMFYUI_WORKFLOW_DIR` at your workflows. Run `--list-workflows` to see what's available (see [Choosing a workflow](#choosing-a-workflow)).
- **Match the prompt to the model's format.** A model supports Natural Language, Danbooru/Gelbooru tags, or both — this can't be guessed from the workflow name. **Ask the user** which (see [Choosing a workflow](#choosing-a-workflow)) before writing the prompt: prose for natural language, comma-separated booru tags (`1boy, black hair, painterly, dramatic lighting, ...`) for tag models, either/mixed if both.
- **Name the style explicitly** when the workflow won't infer one. For the world's house look, ask for a **semi-realistic painterly RPG character splash art**: "painted brushstrokes, dramatic chiaroscuro, warm rim light, oil-painting texture, loose abstract painterly background, moody cinematic color." Avoid flat-anime phrasing unless you want flat anime.

## Character Portrait Workflow

When the user asks for a portrait of an NPC:

1. Read `tabs/npcs.json` and find the NPC entry
2. If the user wants to change appearance details before generating, update `basicInfo` in `tabs/npcs.json` first (with user approval via AskUserQuestion). The prompt is always curated from the current basicInfo — never invent appearance details directly in the prompt that aren't in basicInfo.
3. Curate `basicInfo` down to **visual/appearance details only**. Keep gender and NPC type as they set visual tone. Drop narrative role, relationships, and lore. Include **all physical features** (ears, tails, horns, wings, skin texture, missing limbs, etc.) even if they extend beyond the frame — the model handles composition. Always include the primary weapon. Be selective with other accessories — pick **1-2 signature items**, not every item mentioned. Too many props create visual clutter.
   - **Keep:** `[gender], [type], [build], [hair], [eyes], [fangs/claws/horns/ears/tails/wings/skin], [clothing + 1-2 items]`
   - **Drop:** `serving as [role]`, `bound as one of [master]'s [group]`, `who commands [unit]`
4. Extract the **single keyword** before the colon from each personality trait (e.g. "Enigmatic" from "Enigmatic: No one knows where the act ends"). **Present the trait options to the user** and let them choose — do not pick for them.
5. **Pick the workflow and confirm the prompt format** ([Choosing a workflow](#choosing-a-workflow)) — list available workflows, ask the user which to use, and ask whether the model supports Natural Language, Danbooru/Gelbooru tags, or both. Then assemble the prompt in that format. For a natural-language model, lead with the **style**, then the curated appearance:

```
A semi-realistic painterly digital portrait, RPG character splash art, painted brushstrokes, dramatic chiaroscuro with warm rim light, tight head-and-shoulders close-up, [framing/expression]. [curated appearance]. Loose abstract painterly background, moody cinematic color, oil-painting texture.
```

   Keep the curated appearance to visual facts only (build, hair, eyes, distinctive features, clothing + 1–2 signature items, primary weapon). Name the **style explicitly** when the model won't infer one. For a tag-based model, rewrite the whole thing as Danbooru/Gelbooru comma tags instead.

6. **Never modify the approved prompt without flagging the change.** If a previous generation didn't capture a detail (e.g., build reads too thin), explain the issue and propose a prompt change before regenerating. Do not silently add words to the prompt.
7. Run cli.exe with `--workflow <chosen>.json --out images/generated/[name-lowercase]-000.png --prompt "<prompt>"`. Add `--width 832 --height 1216` to force portrait size. Match the prompt style to the chosen workflow (natural language vs. tags). For each retry/variant, increment the `--out` suffix (`-001`, `-002`, …) yourself — cli.exe does not auto-increment.
8. Show the image and **wait for user approval**. If touch-ups are needed, use `--img2img <path>` to edit the existing image rather than regenerating from scratch. When the user specifies a base image for img2img edits, use that base for all subsequent retries unless the user explicitly changes it.
9. Only once approved, host it for a stable URL (the local file lives under `images/generated/`, but a hosted link is what goes into the config):

```bash
node .claude/skills/comfyui-image-gen/scripts/upload-image.mjs -n [name-lowercase] images/generated/<file>.png
```

This uploads (to catbox.moe), prints the URL, and moves the file to `images/uploaded/{name}-{hash}.png`. **Note:** catbox may return `412 Precondition Failed` when anonymous uploads are blocked/rate-limited — set `CATBOX_USERHASH`, use a different host, or paste a URL the user provides. If the user already has a hosted image URL, skip this step.

10. Add the returned URL as `"portraitUrl"`:
    - **NPCs** → the NPC entry in `tabs/npcs.json`
    - **Premade characters** (e.g. the fixed protagonist) → the character entry in `tabs/premade-characters.json`
    Then rebuild with `node .claude/scripts/build.js` and validate with `node .claude/scripts/validate.js`.

### Batch Portrait Workflow

When generating portraits for multiple NPCs at once:

1. Curate appearance text and collect personality keywords for all NPCs upfront
2. Run all `cli.exe` calls concurrently via background Bash, each with its own distinct `--out images/generated/[name]-000.png` path. Note ComfyUI processes **one prompt at a time** (extra runs queue), so "concurrent" here means submitted together, not rendered in parallel — they still complete one after another. Retry any that error.
4. Show all variants to the user and collect picks
5. For each picked variant, upload and write `portraitUrl`
6. Clean up unpicked variants:

```bash
node .claude/skills/comfyui-image-gen/scripts/cleanup-variants.mjs -n [name] -k [picked-number]
```

This moves all variants of `[name]` except the picked one to `images/generated/unused/` (along with any sidecar metadata JSON, which only `generate-comfy.mjs` produces — `cli.exe` writes none).

## Prompt Tips

Be specific: not "a tavern" but "a dimly lit medieval tavern with smoke curling from a stone hearth, amber candlelight, rough oak tables stained with ale."

## Server-based alternative (generate-comfy.mjs)

When the **comfyui-api** FastAPI server (`127.0.0.1:5000`) is already running, you can use the original Node.js script instead of `cli.exe`. It submits a job, polls, and downloads from the server's in-memory store:

```bash
node .claude/skills/comfyui-image-gen/scripts/generate-comfy.mjs -n <name> [options] <prompt>
```

Key differences from `cli.exe`: it uses `-n <name>` (auto-increments `name-000.png`, `name-001.png`, …) and writes a metadata JSON to `images/generated/json/<name>.json`; `-i` is its img2img flag; `--host` points at the `:5000` server. It requests preview frames by default (`--no-preview` to skip, **not recommended** — the server keeps images in memory only). Concurrent runs with the same `-n` name are safe (atomic file creation). Prefer `cli.exe` unless the server is the only thing available.
