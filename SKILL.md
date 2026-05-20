---
name: aholo-3dgs-reconstruction-global
description: "Aholo OpenAPI v1 global 3D tasks (reconstruction/generation): upload assets, create task (WorldAsyncOperation / worldId), poll status and fetch PLY/SPZ/SOG; gateway api.aholo3d.com, world APIs under /global/world/v1. One POST create per user task round. Not for 2D image output."
---

# Aholo 3D Reconstruction Skill — Global (OpenAPI v1)

> **Scope:** **Aholo global Open Platform** (`api.aholo3d.com`) 3D tasks — not 2D text-to-image.

## When to use / not use (decide first)

### Use this skill when

- The user wants **3D reconstruction** (model / space)
- The user wants a **3D world** (`worldId`, online preview, PLY / SPZ / SOG downloads)
- The user asks to **query `worldId` status or keep polling**

### Do not use this skill when

- The user only wants **2D renders / concept art / still images**
- The user did not ask for a **3D outcome** (worldId, model files, online 3D preview)
- The user wants general image editing or text-to-image (use image-generation capabilities)

### Ambiguous requests (clarify first)

When the user says something like “generate a room in this style from a reference image”, ask:

```text
Which do you want?
1) 2D room render (single image)
2) 3D room task (returns worldId and can be polled)
```

Enter this skill’s flow only after the user clearly picks **2)**.

Upload images or video and create 3D tasks:

- **reconstruction**
- **generation** / spatial gen

When complete, model download URLs (PLY / SPZ / SOG) may be returned.

## Prerequisites

Environment variable:

- `AHOLO_API_KEY` — API Key from [labs.aholo3d.com/api-keys](https://labs.aholo3d.com/api-keys)

Auth header: `Authorization: <API Key>` (**no** `Bearer` prefix).

### When `AHOLO_API_KEY` is missing (agent behavior)

- Tell the user to set `AHOLO_API_KEY` in the terminal or system environment, then reply **“continue”** or describe the same 3D task again.
- **Do not** ask the user to run `python ... aholo_reconstruct.py` as the main path; the agent runs the script by default.
- Local debugging by the user is optional.

## TLS verification (skipped by default)

- SSL verification is off by default to avoid `CERTIFICATE_VERIFY_FAILED` on corporate/self-signed networks.
- Set `AHOLO_FORCE_SSL_VERIFY=1` to force verification on.
- `AHOLO_INSECURE_SKIP_VERIFY` still skips verify unless explicitly `0` / `false` / `no` / `off`.

## Service endpoints

- API gateway: `https://api.aholo3d.com`
- Upload host: from `GET /global/world/v1/asset/token` → `globalDomain` (OUS direct upload; response uses OUS V2 `c` / `m` / `d` envelope)
- Typical `globalDomain` example: `https://ous-sg.kujiale.com` (always use the token response value)

## Open Platform APIs

| Method | Path | Success body |
|--------|------|--------------|
| GET | `/global/world/v1/asset/token` | `ousToken`, `globalDomain`, `blockSize` |
| POST | `/global/world/v1/reconstructions` | `WorldAsyncOperation`: `{"worldId":"<encrypted id>"}` |
| POST | `/global/world/v1/generations` | Same as above |
| GET | `/global/world/v1/{worldId}` | World detail (`status`, `assets`, etc.) |
| POST | `/global/world/v1/list` | Paginated list (not wrapped in script) |

OUS upload paths remain `/ous/api/...` (no `/global` prefix on OUS).

## Response conventions

- **Success:** HTTP 200, business object returned directly (not OUS `c` / `m` / `d`).
  - Create: JSON `WorldAsyncOperation` with field **`worldId`** only. Script also accepts legacy plain-text `worldId` if present.
- **Failure:** HTTP 4xx/5xx, **`ApiError`**: `code`, `message`, `status`, `details.metaData.bizCode` (e.g. auth **401**, bizCode **10004**).

## Supported actions

| Action | Purpose |
|--------|---------|
| `create` | Unified entry; set `workflow` to `reconstruction` or `generation` |
| `create-reconstruction` | Reconstruction only |
| `create-generation` | Generation only |
| `poll` | Poll by `worldId` until terminal state (recommended) |
| `status` | Single status query |

## Agent hard constraints (must follow)

0. 2D image requests → **do not** call create / status / poll from this skill.
1. Reconstruction without confirmed `scene` and `taskQuality` → **no** create.
2. Confirm with the user first:
   - `scene`: `model` or `space`
   - `taskQuality`: `low` / `normal` / `high`
3. **Do not** substitute defaults (e.g. `model` or `high`) without user choice.
4. Call create only after explicit user choice (and obey rule 9).
5. After `worldId`, **ask** whether to wait for polling:
   - **Will wait:** run synchronous `poll` in-session until done
   - **Will not wait:** return `worldId` and viewer link only
6. **Do not** background-poll and promise “notify when done” — the session cannot notify later.
7. If 3D intent is unclear, clarify 2D vs 3D before creating.
8. **Image directories:** when the user points at a folder of images, use **`imageDir`** so all images upload — **never** upload only a subset (e.g. first 20).
9. **One create POST per user task round** (high cost):
   - At most **one** `POST /global/world/v1/reconstructions` or `POST /global/world/v1/generations` per user’s stated order in this conversation round.
   - On failure, timeout, or missing `worldId` locally, **do not** create again unless the user explicitly starts a **new** order.
   - Exception: if create POST was **never sent** (auth, missing assets, etc.), one first create is allowed after fixes.
   - If the server may have charged but `worldId` is missing: guide the user to the platform task list or use `status` / `list` — **not** another create.
   - If create was already sent this round, pass `forbidCreate: true` or skip create actions.
10. **`projectName`:** omit unless the user explicitly requests a project name.

## Required parameter interaction

1. Prefer `AskQuestion` for `scene` × `taskQuality` (six combinations).
2. Ask both in one step when possible.
3. No create until the user has chosen.
4. If the user writes free text (e.g. “high quality space”), normalize, confirm once, then proceed.

### Recommended options

| Option | scene | taskQuality | Note |
|--------|-------|-------------|------|
| 1 | model | low | Object, faster |
| 2 | model | normal | Object, balanced |
| 3 | model | high | Object, recommended |
| 4 | space | low | Scene, faster |
| 5 | space | normal | Scene, balanced |
| 6 | space | high | Scene, recommended |

**After create — ask whether to wait:**

```text
Task created, worldId: {worldId}

When complete, view at: https://www.aholo3d.com/3dgs-model/{worldId}
(Link may not work until the task finishes.)

Reconstruction usually takes minutes to tens of minutes.

Wait until complete?
- "wait" / "yes" — I poll in this session until done
- "no" — use the link later or ask me to poll/status
```

## Recommended flow

1. Confirm `reconstruction` vs `generation`.
2. For reconstruction: confirm `scene`, `taskQuality`.
3. **One** create for the whole round.
4. Record `worldId`; ask whether to poll.
5. Afterwards only `poll` / `status` / `list` unless the user starts a new task.

## Execution logging (default on)

Each step should log: start, end, duration (seconds). Suggested steps: token, upload, create, status, each poll tick.

## Polling strategy

After `worldId`, **ask** before polling.

**User will wait** — synchronous poll (use `-u`):

```bash
python -u aholo_reconstruct.py '{"action":"poll","worldId":"xxx","intervalSeconds":60,"timeoutSeconds":14400}'
```

**User will not wait** — return `worldId` and `https://www.aholo3d.com/3dgs-model/{worldId}`.

Suggested: `intervalSeconds=60`, `timeoutSeconds=14400`.

**Forbidden:** background poll with a promise to notify later; polling without asking.

## Task type rules

### reconstruction

- Input: `videoPaths` **or** `imagePaths` / `imageDir` (one of)
- `videoPaths`: 1–3 files
- `imagePaths` / `imageDir`: at least 20 images
- Required: `scene` (`model` / `space`), `taskQuality` (`low` / `normal` / `high`)

### generation

- `imagePaths` only, at most 1 image
- `prompt` and `imagePaths` cannot both be empty
- No `videoPaths`

## Parameters

### General

| Parameter | Type | Description |
|-----------|------|-------------|
| `action` | enum | `create` / `create-reconstruction` / `create-generation` / `status` / `poll` |
| `workflow` | enum | For `create` only: `reconstruction` or `generation` |
| `projectName` | string | Optional → request `name`; omit unless user asked |
| `coverPath` | string | Local cover image (optional) |
| `cover` | string | Existing cover URL (optional) |
| `worldId` | string | Required for `status` / `poll` |
| `forbidCreate` | bool | If `true`, block any `create*` in this run |

### reconstruction

| Parameter | Type | Rule |
|-----------|------|------|
| `videoPaths` | string[] | 1–3 video paths |
| `imagePaths` | string[] | ≥20 image paths |
| `imageDir` | string | **Preferred** for folders — scans all images |
| `scene` | enum | `model` / `space` (required) |
| `taskQuality` | enum | `low` / `normal` / `high` (required) |

### generation

| Parameter | Type | Rule |
|-----------|------|------|
| `imagePaths` | string[] | At most 1 |
| `prompt` | string | Optional; cannot be empty together with images |

## Examples

Agent should run the script; users only need `AHOLO_API_KEY` set.

```bash
python -u .cursor/skills/aholo-3dgs-reconstruction-global/aholo_reconstruct.py '{
  "action": "create",
  "workflow": "reconstruction",
  "imageDir": "D:/images",
  "scene": "space",
  "taskQuality": "high"
}'

python -u .cursor/skills/aholo-3dgs-reconstruction-global/aholo_reconstruct.py '{
  "action": "create-reconstruction",
  "videoPaths": ["D:/videos/angle1.mp4"],
  "scene": "model",
  "taskQuality": "high"
}'

python -u .cursor/skills/aholo-3dgs-reconstruction-global/aholo_reconstruct.py '{
  "action": "create-generation",
  "imagePaths": ["D:/images/seed.jpg"],
  "prompt": "modern minimal interior"
}'

python -u .cursor/skills/aholo-3dgs-reconstruction-global/aholo_reconstruct.py '{
  "action": "status",
  "worldId": "A1b2C3d4E5"
}'

python -u .cursor/skills/aholo-3dgs-reconstruction-global/aholo_reconstruct.py '{
  "action": "poll",
  "worldId": "A1b2C3d4E5",
  "intervalSeconds": 60,
  "timeoutSeconds": 14400
}'
```

## Windows PowerShell (optional local debug)

```powershell
$env:AHOLO_API_KEY="your_api_key"
```

```powershell
$env:AHOLO_FORCE_SSL_VERIFY="1"
```

## Terminal status values

`SUCCEEDED` | `FAILED` | `CANCELED` | `TIMEOUT` | `REJECTED`

## Viewer URL after successful create

`https://www.aholo3d.com/3dgs-model/{worldId}`
