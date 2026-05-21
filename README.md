# aholo-3dgs-reconstruction-global

Cursor / OpenClaw Agent Skill for **Aholo Global Open Platform** 3D tasks: world reconstruction and spatial generation via OpenAPI v1.

- **Gateway:** `https://api.aholo3d.com`
- **World APIs:** `/global/world/v1/...` (OUS upload paths `/ous/api/...` are not prefixed with `/global`)
- **API keys:** [labs.aholo3d.com/api-keys](https://labs.aholo3d.com/api-keys)

## Repository layout

| Path | Role |
|------|------|
| `LICENSE` | MIT license (marketplace requirement) |
| `skills/aholo-3dgs-reconstruction-global/SKILL.md` | Agent instructions |
| `skills/aholo-3dgs-reconstruction-global/aholo_reconstruct.py` | CLI: upload, create task, poll |
| `skills/aholo-3dgs-reconstruction-global/requirements.txt` | Python dependencies |

## Install (Cursor)

```bash
git clone https://github.com/xiaohao17501671450-lgtm/aholo-3dgs-reconstruction-global.git
```

Copy the **skill folder** (not the whole repo) into your skills directory:

- Windows: `%USERPROFILE%\.cursor\skills\aholo-3dgs-reconstruction-global` ← copy from `skills/aholo-3dgs-reconstruction-global/`
- macOS/Linux: `~/.cursor/skills/aholo-3dgs-reconstruction-global`

Restart Cursor so the skill is discovered.

## Prerequisites

```bash
cd skills/aholo-3dgs-reconstruction-global
pip install -r requirements.txt
```

```bash
# PowerShell
$env:AHOLO_API_KEY="your_api_key"
```

```bash
# bash
export AHOLO_API_KEY="your_api_key"
```

## Quick test

```bash
cd skills/aholo-3dgs-reconstruction-global
python -u aholo_reconstruct.py '{"action":"status","worldId":"<worldId>"}'
```

See `skills/aholo-3dgs-reconstruction-global/SKILL.md` for full workflows.

## Docs

- [Aholo Labs — Aholo World (EN)](https://labs.aholo3d.com/skills/aholo-3dgs-reconstruction)
- [ClawHub package](https://clawhub.ai/xiaohao17501671450-lgtm/aholo-3dgs-recon-global)

## Submit to Claude Skills Hub

Use this repository URL (no `.git` suffix):

`https://github.com/xiaohao17501671450-lgtm/aholo-3dgs-reconstruction-global`
