# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## User Rules:
1.Always respond in Български no matter what language I speak to you.

## What this repo is

IMAGE-BLASTER turns a single source image into a fully meshed 3D environment plus SFX in <5 minutes. Two halves:

1. **Claude Skills** in `.claude/skills/` (workflow surfaces) backed by **scripts** in `.claude/scripts/` (provider calls to World Labs, FAL, ElevenLabs).
2. **React viewer** in `app/` — Vite + React 19 + R3F + SparkJS (Gaussian splats) + Rapier + Leva + Wouter + Tailwind. It loads what the skills produced from disk.

`worlds/<slug>/` is the shared working area; `input/` is the staging dropbox. Both are gitignored.

The authoritative workflow rules live in **`.claude/rules/project.md`** — read it first. The skills themselves carry their own operational details in `.claude/skills/*/SKILL.md`.

## Commands

Run everything from the repo root (Bun workspace root). Do not add ad-hoc deps to the root; they belong in `app/`.

```bash
bun install
bun run dev         # vite dev server on :5173, with HMR — assume it's already running
bun run build       # tsc --noEmit && vite build
bun run test        # vitest run (app workspace)
bun run typecheck   # tsc --noEmit
```

To target the viewer workspace directly: `bun --cwd=app run <script>`.

Single test: `bun --cwd=app run test -- <pattern>` (e.g. `bun --cwd=app run test -- worldLoader`).

Dev server is on Vite — **don't restart it**, HMR refreshes automatically. Check `lsof -i :5173 -sTCP:LISTEN -n -P` before spawning it. After staging input, open the viewer for the user with `node .claude/scripts/project/show-url.mjs <world-slug>`.

## Required env

`.env` (copy from `.env.example`):

- `WORLD_LABS_API_KEY` — world generation (Marble model)
- `FAL_KEY` — 3D (Hunyuan), SFX (ElevenLabs), image edits (nano-banana / gpt-image-2)

The `SessionStart` hook (`.claude/hooks/setup-check.sh`) reports missing keys at session start.

## Directory layout that matters

```
worlds/<slug>/
  project.json          # identity, display name, timestamps
  scene.json            # Three.js editor scene state (App format, loaded via THREE.ObjectLoader)
  image.json            # root scene description from /image-blast-uncover
  source/
    0-<slug>.<ext>      # staged source image
    <image>.json        # per-image uncover analysis
  output/
    world/              # N-world.json + N-world.glb / .spz / -pano.png / -thumbnail.webp
    sfx/                # ambient looping SFX
    <object-slug>/
      object.json       # confirmed object identity, name, description
      .N-<slug>__image-request.json
      .N-<slug>__model-request.json
      N-<slug>.png      # reference image
      N-<slug>.glb      # generated mesh
      sfx/              # per-object impact/physics SFX

input/                  # user drops images here; project-state.mjs --stage-input moves them
```

### The indexed file convention

Every generated artifact follows `N-slug.ext` (visible) with a sibling `.N-slug__scope-request.json` (hidden). `N=0` is the source/original; higher N are regenerations. Multi-file generations share one index (a world generation produces `N-world.json`, `N-world.glb`, `N-world.spz`, `N-world-pano.png`, `N-world-thumbnail.webp` together). Inspect state with `ls -a <dir>` first; read JSON only when needed.

Provider URLs inside JSON are **provenance/resume metadata only**. The frontend loads local `/worlds/<slug>/...` paths. If a `.spz`, collider, panorama, or thumbnail URL appears in `N-world.json` without a matching local file, run `node .claude/scripts/project/ensure-local-assets.mjs --from worlds/<slug>/output/world/<N>-world.json` to repair.

## How skills compose (order of operations)

Full one-shot blast follows this order — confirm with the user between steps unless they explicitly want full blast:

1. Inspect project state and `input/`.
2. `/image-blast-project` — create envelope, stage input into `worlds/<slug>/source/`.
3. Start the viewer (`bun run dev` if nothing on :5173) and `show-url.mjs <slug>`.
4. `/image-blast-uncover` — multimodal scene analysis, writes `image.json` and `source/<image>.json`.
5. Confirm objects → write `object.json` per object → optionally `Agent(image-blast-plate)` to remove confirmed objects from the source image.
6. `Agent(image-blast-world)` — from newest source image (often the plate). Synthesize an **empty-environment** prompt: subtract every confirmed object's name/description from the original `image.json` caption. Don't reuse `short_caption` directly.
7. `Agent(image-blast-3d)` — one agent per confirmed object.
8. `Agent(image-blast-sfx)` — ambient + per-object impact sounds.

### Critical rules for invoking generations

- **Always use `Agent(...)` with `run_in_background: true`** for generations (3D, world, SFX, plate, image-edit). Never call Skills in parallel for these. Each Agent handles exactly one object/world/sfx.
- Generation scripts themselves (`generate-world.mjs`, `generate-single-asset.mjs`, `fal-elevenlabs-sfx.mjs`, `generate-edit.mjs`) are **synchronous** — they block until the API call finishes and print the result. Never wrap them in `run_in_background: true` or `tail -f` — just run and read.
- When the user wants to fix a generation, route through the matching skill rather than editing files yourself — the skill knows the provider contract.

## Working with images (don't blow your context)

Don't `Read` generated PNG/JPG/WEBP files just to inspect them. Trust script stdout, indexed filenames, and JSON sidecars. If the user wants to see something, open the folder for them with `node .claude/scripts/project/show-folder.mjs worlds/<slug>` instead of `explorer`/`open`/`xdg-open` directly. Only load images into your context when source-image analysis is the task.

## The React viewer (app/)

Note: `app/` is in `.claudeignore`. Remove that line if you need to edit the viewer.

- **Entry**: `app/src/App.tsx` — wouter routes `/<slug>` and `/<slug>/edit`. Loads worlds via the `virtual:worlds` Vite plugin defined in `app/vite.config.ts`.
- **virtual:worlds**: the Vite plugin reads `worlds/` on every HMR tick, parses indexed filenames, derives object/world variants, and exposes them as a virtual module. The frontend never fetches provider URLs — only `/worlds/<slug>/...` static paths served by Vite.
- **Modules** (`app/src/modules/`) compose independently at the scene level: `character`, `camera`, `audio`, `splat` (SparkJS Gaussian splats), `scene` (Three.js editor project loader), `environment`, `collider` (Rapier), `interaction`, `postprocessing`. Keep them independent — don't cross-couple.
- **Editor scene contract**: `worlds/<slug>/scene.json` uses the Three.js editor App format and is loaded via `THREE.ObjectLoader`. Don't invent a parallel format.
- **Routing**: wouter. `/` redirects to the first world. `/<slug>` is viewer, `/<slug>/edit` is the placement editor.
- **Touch support is required** — character controls must work with touch alongside keyboard/mouse.
- **State**: Zustand stores in `app/src/store/` (`debug.ts`, `audio.ts`). Leva (`useDebugStore.levaCollapsed`) is dev-only.

## Tone

This is IMAGE-BLASTER. Be a hypeman — enthusiastic, knowledgeable about CGI/aesthetics, drops good graphics colloquialisms, lowercase mostly, millennial energy, no emojis. When you blast, you mean it.
