# PrismTrack Current Project Context

## Snapshot

- Date: 2026-05-13
- Branch: `260504-feat-prismtrack-spleeter-migration`
- Pull request: `https://github.com/valenbine/PrismTrack/pull/1`
- Product name: `PrismTrack`
- Main app directory: `PrismTrack-Spleeter/`

## Current Scope

This branch migrates PrismTrack to a Spleeter-backed web and Windows desktop project. The app keeps the existing Web UI approach while adding a minimal Electron host and a reproducible Windows packaging workflow.

## Key Decisions

1. Use Spleeter instead of Demucs for source separation.
2. Keep the Electron desktop layer minimal: start `server.js`, wait for readiness, open the local Web UI, and delegate external links to the system browser.
3. Do not commit local Windows runtime binaries to GitHub.
4. Hydrate the Windows runtime in CI from `SpleeterGUI 2.9.4.0` before building the installer.
5. Do not bundle Spleeter model weights in the installer. Download them on demand with Lazy Model Fetch.
6. Store desktop runtime data under Electron `userData`, not under the installation directory.

## Runtime Layout

The Windows installer expects these runtime files during packaging:

```text
PrismTrack-Spleeter/
  python/
    python.exe
    python37.dll
    Lib/
    Scripts/
  ffmpeg.exe
  ffprobe.exe
  ffplay.exe
```

These files are required in the packaged app, but they are intentionally ignored in Git. GitHub Actions downloads and injects them during packaging.

## Desktop Runtime Paths

- Logs: `%APPDATA%/prismtrack-spleeter/logs/desktop.log`
- Runtime temp data: `%APPDATA%/prismtrack-spleeter/.runtime/`
- Model cache: `%APPDATA%/prismtrack-spleeter/pretrained_models/`
- Model download temp archives: `%APPDATA%/prismtrack-spleeter/.runtime/model-downloads/`

## Readiness Protocol

The desktop host should not wait on deep `/api/health` checks before opening the window.

1. Start local `server.js`.
2. Probe `GET /api/ready`.
3. If `/api/ready` is unavailable, probe `/`.
4. Open the window when the local Web UI is reachable.
5. Use `/api/health` only for deeper runtime diagnostics.

## Model Download Behavior

- Models are downloaded on first use.
- Download progress is surfaced in the existing Web status area.
- Downloading does not count toward separation processing timeout.
- `.tar.gz.part` files support resume with HTTP Range requests.
- If Node `fetch` fails on Windows because of certificate-chain issues, the backend falls back to system `curl.exe` with `--continue-at -` and `--ssl-no-revoke`.

## CI Packaging

Workflow file: `.github/workflows/build-windows.yml`

The workflow:

1. Runs on `windows-latest`.
2. Uses `PrismTrack-Spleeter/` as the working directory.
3. Downloads `SpleeterGUI_2.9.4.0.zip`.
4. Extracts `python/`, `ffmpeg.exe`, `ffprobe.exe`, and `ffplay.exe`.
5. Runs `npm ci`.
6. Runs `npm run dist:win`.
7. Verifies packaged Node dependencies including `archiver`, `archiver-utils`, and `zip-stream`.
8. Uploads `dist/**` artifacts.

The workflow is triggered by `push` and `workflow_dispatch`, not by `pull_request`.

## Files That Should Stay Ignored

- `.runtime/`
- `PrismTrack-Spleeter/.runtime/`
- `PrismTrack-Spleeter/python/`
- `PrismTrack-Spleeter/ffmpeg.exe`
- `PrismTrack-Spleeter/ffprobe.exe`
- `PrismTrack-Spleeter/ffplay.exe`
- `PrismTrack-Spleeter/pretrained_models/`
- `PrismTrack-Spleeter/node_modules/`

## Verification Status

- The latest pushed branch contains the Windows desktop host, runtime injection workflow, Lazy Model Fetch, resume support, Windows TLS fallback, and documentation updates.
- Local workspace should remain clean after committing `.monkeycode` updates.
- Remote synchronization should be verified with `git fetch origin` and `git rev-parse HEAD origin/260504-feat-prismtrack-spleeter-migration`.
