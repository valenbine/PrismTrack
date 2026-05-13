# PrismTrack Windows Packaging

## Current Status

This repository now includes a minimal Electron desktop host at `desktop/main.cjs`.

Its responsibilities are intentionally narrow:

- start the local `server.js` process
- wait for `GET /api/ready`, with `/` as a fallback readiness check
- open the Web UI inside an Electron window
- open external links in the default Windows browser
- write startup diagnostics to the Electron `userData` log directory

## Required Runtime Layout

The CI workflow hydrates the Windows runtime from the `SpleeterGUI 2.9.4.0` release before packaging. A local build needs the same runtime layout under `PrismTrack-Spleeter/`:

```text
PrismTrack-Spleeter/
  ffmpeg.exe
  ffprobe.exe
  ffplay.exe
  python/
    python.exe
    python37.dll
    Lib/
    Scripts/
```

The desktop host passes these paths to `server.js` through `SPLEETER_PYTHON`, `FFMPEG`, `FFPROBE`, and `SPLEETER_WRAPPER`.

## Installer Behavior

The current `electron-builder` configuration is set up for:

- product name: `PrismTrack`
- target: `nsis`
- selectable install directory
- unpacked application files (`asar: false`)
- model weights excluded from the installer

## Important Notes

1. The repository does not commit the Windows Python runtime.
2. The repository does not commit `ffmpeg.exe`, `ffprobe.exe`, or `ffplay.exe`.
3. GitHub Actions downloads and injects those runtime files during packaging.
4. Local `npm run dist:win` requires the runtime files to exist locally first.
5. Model weights are intentionally not bundled. They are downloaded on demand at runtime.

## Model Download Behavior

The desktop build stores model weights under Electron `userData`:

```text
%APPDATA%\prismtrack-spleeter\pretrained_models
```

Temporary model archives are stored under:

```text
%APPDATA%\prismtrack-spleeter\.runtime\model-downloads
```

Downloads use stable `.tar.gz.part` files for resume support. If Node `fetch` fails on Windows because of TLS certificate chain issues, the backend falls back to system `curl.exe` with `--continue-at -` and `--ssl-no-revoke`.

## Diagnostics

Startup logs are written to:

```text
%APPDATA%\prismtrack-spleeter\logs\desktop.log
```

The log records runtime roots, resolved Python/ffmpeg paths, server stdout/stderr, readiness strategy, and model download failures.

## Recommended Next Steps

1. Copy the validated Windows runtime from the chosen reference baseline for local builds, or use GitHub Actions for CI builds.
2. Install project dev dependencies with `npm ci`.
3. Run `npm run desktop` for local desktop verification.
4. Run `npm run dist:win` to produce the installer.
