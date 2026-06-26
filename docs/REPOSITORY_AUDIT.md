# Repository Audit Report

Audit mode: static source review only. The project was not installed, built, or executed.

## 1. Tong quan du an

This repository is a local subtitle/video processing application named `subtitles-generator` / "One-Click Subtitles Generator". It combines a React frontend, an Express backend, Electron desktop packaging, a Remotion video renderer, and Python services for narration/transcription engines.

Main capabilities observed from source:

- Accept video inputs from uploads, YouTube, Douyin/TikTok, or other yt-dlp-supported sites.
- Download/cache videos and scan video quality with `yt-dlp`.
- Generate, translate, clean, split, and render subtitles.
- Use Google Gemini APIs for transcription/translation/analysis/narration/image-related flows.
- Use TTS/narration engines including Gemini audio, gTTS, Edge TTS, Chatterbox, F5-TTS, and Parakeet/ONNX ASR wrappers.
- Render subtitled videos via Remotion and local backend services.

Technologies used:

- JavaScript/Node.js: Express backend, scripts, Electron main/preload, React frontend.
- TypeScript: `video-renderer` Remotion/server code and `promptdj-midi` components.
- Python: FastAPI/Flask-like services, narration service, Chatterbox API, Parakeet wrapper.
- Frontend framework/runtime: React 18, Create React App (`react-scripts`), Vite in `promptdj-midi`.
- Desktop runtime: Electron.
- Package managers/runtimes: npm workspaces, `uv` for Python virtual environments.
- Media tooling: `ffmpeg`, Remotion, `yt-dlp`, `fluent-ffmpeg`, `pydub`, `torch/torchaudio`.

Build/run commands if considered safe:

- `npm install` triggers `postinstall`, which runs workspace builds. Review `package.json` and workspace manifests first.
- `npm run dev` starts generated React env setup, version generation, and `dev-server.js`.
- `npm run server:start` starts the Express backend.
- `npm run python:start` starts the Python narration service via `uv`.
- `npm run dist:*` builds Electron packages after copying/preparing Python resources.

Do not run the installer scripts directly on a primary machine until the checklist in section 7 is followed.

## 2. Cau truc thu muc

Important top-level structure:

```text
.
├── app.js                         Express app configuration and routes
├── server.js                      Backend startup, port cleanup, WebSocket init
├── package.json                   Main npm workspace manifest and Electron build config
├── package-lock.json              npm lockfile
├── electron/                      Electron main/preload and Python service process manager
├── server/                        Backend routes, controllers, services, model manager, engines
├── src/                           React frontend components, services, styles, i18n, utilities
├── video-renderer/                Remotion renderer workspace and local render server
├── promptdj-midi/                 Vite/React workspace
├── chatterbox-fastapi/            FastAPI service for Chatterbox TTS
├── parakeet_wrapper/              FastAPI/ONNX ASR wrapper with Python package metadata
├── scripts/                       Build, setup, cache, install/uninstall helper scripts
├── public/                        Static frontend assets and sample audio
├── readme_assets/                 README images
├── additional_docs/               Extra documentation
├── models/                        Model-related local files
└── docs/                          Audit/report documentation
```

Key files:

- `package.json`: npm scripts, dependencies, workspaces, Electron packaging config.
- `server/config.js`: local ports and generated working directories such as `videos`, `subtitles`, and `narration`.
- `server/services/shared/ytdlpUtils.js`: yt-dlp executable resolution, ffmpeg args, optional browser cookie support.
- `src/services/gemini/*`: Gemini API client, key manager, translation/streaming/file APIs.
- `src/services/googleAuthService.js`: YouTube OAuth client credential and token storage in browser localStorage.
- `OSG_installer.sh`, `OSG_installer_Windows.bat`: system-level installers.
- `scripts/OSG_uninstaller_fortesting.*`: system/application uninstall helpers.

No `.github/workflows`, Dockerfile, or docker-compose file was found in the static scan.

## 3. Luong xu ly chinh

Typical flow from input to output:

1. User provides a video URL or local media file in the React UI.
2. Frontend calls the local Express backend on localhost ports, mainly `3031`.
3. Backend downloads or reads media. YouTube/all-site downloads use `yt-dlp`; uploaded files are cached under local project directories.
4. Backend/Python services extract audio, normalize media, transcribe or generate narration.
5. Gemini/Genius/Google APIs may be called when the selected feature needs AI analysis, translation, lyrics lookup, or YouTube OAuth/search.
6. Subtitles/narration/audio are cached under local folders such as `videos`, `subtitles`, and `narration`.
7. Remotion/video-renderer composes and renders the final subtitled video.
8. Frontend previews or downloads generated output.

## 4. Dependency audit

Main Node dependencies and purpose:

- `react`, `react-dom`, `react-scripts`: main web UI.
- `electron`, `electron-builder`: desktop app packaging/runtime.
- `express`, `cors`, `multer`, `ws`: local backend API, uploads, WebSocket progress.
- `axios`, `cheerio`, `googleapis`, `@google/genai`, `@react-oauth/google`: external HTTP/API integrations.
- `@remotion/*`, `remotion`, `fluent-ffmpeg`, `@ffmpeg-installer/ffmpeg`: video rendering/media processing.
- `puppeteer-core`: site-specific extraction/fallback flows.
- `jszip`, `adm-zip`: archive handling.

Python dependencies:

- `fastapi`, `uvicorn`, `flask`, `flask-cors`: local Python APIs.
- `torch`, `torchaudio`, `vocos`, `soundfile`, `numpy`, `scipy`, `pydub`: audio/TTS/transcription processing.
- `onnx-asr`, `onnxruntime`/`onnxruntime-directml`: Parakeet ASR wrapper.
- `yt-dlp`, `edge-tts`, `gtts`: installed by setup scripts into `.venv`.

Dependencies/risk points to note:

- `postinstall` exists in `package.json`, but it runs only `npm run workspace:build`, which builds `video-renderer` and `promptdj-midi` (`package.json:44`, `package.json:57`).
- Installer/setup scripts install packages from npm/PyPI/GitHub/OS package managers. This is normal for this app type but should be run in isolation first.
- `setup-narration.js` clones/downloads external engine/plugin code and installs Python packages. Review exact refs before running.
- `yt-dlp` and browser-cookie integration are sensitive because they can read browser profiles when enabled.

## 5. Security audit

### Environment variables

- Reads non-secret service configuration such as ports: `FRONTEND_PORT`, `BACKEND_PORT`, etc. in `server/config.js`.
- Reads runtime flags such as `NODE_ENV`, `VERBOSE`, `ELECTRON_*`, `VENV_PATH`.
- I did not find code that enumerates all environment variables for exfiltration. Some spawned child processes inherit `process.env`, which is common but means secrets in the shell environment could be visible to child tools.

### Sensitive local storage and API keys

- Gemini keys are stored in browser `localStorage` by `src/services/gemini/keyManager.js`.
- YouTube OAuth client ID, client secret, and OAuth tokens are stored in browser `localStorage` by `src/services/googleAuthService.js:5-7`, `src/services/googleAuthService.js:67-68`.
- Backend controllers read `localStorage.json` in the project root for Gemini/Genius keys:
  - `server/controllers/geminiController.js:24-38`
  - `server/controllers/geniusController.js:77-95`
- This is not clear malware, but it is a security concern: localStorage and a JSON file are not hardened secret stores.

### Reading files outside the project

- Browser profile paths are checked and may be used for cookie extraction when cookie downloads are enabled:
  - Windows Chrome/Edge/Firefox paths: `server/services/shared/ytdlpUtils.js:108-114`
  - macOS browser paths: `server/services/shared/ytdlpUtils.js:115-122`
  - Linux browser paths: `server/services/shared/ytdlpUtils.js:123-129`
- `yt-dlp --cookies-from-browser` is added when `useCookies` is true: `server/services/shared/ytdlpUtils.js:347-370` and later in that function.
- `extractCookiesToFile()` can export cookies to a temp file for caching: `server/services/shared/ytdlpUtils.js:262-338`.
- I did not find code reading SSH keys, password manager databases, browser "Login Data" passwords, or arbitrary home-directory files. Cookie access is the primary sensitive filesystem behavior.

### Network calls / external services

Expected external endpoints found:

- Google Gemini API: `https://generativelanguage.googleapis.com/...` in `server/controllers/geminiController.js:37-38` and additional frontend Gemini services.
- Genius API and Genius web pages: `server/controllers/geniusController.js:90-113`.
- Hugging Face model downloads: `server/model_manager/download_helpers.py`.
- Google Fonts in video renderer.
- YouTube/Douyin/TikTok/other video sites through `yt-dlp`, Puppeteer, or site-specific downloaders.
- GitHub downloads/clones for setup, engine, and yt-dlp plugin install flows.
- NodeSource, Homebrew, Astral uv installer in shell installer scripts.

I did not find evidence of data being sent to an unknown attacker-controlled domain. Network calls appear tied to declared app functionality. However, user media/subtitle text/API requests may be sent to third-party services depending on selected features.

### Obfuscation/minified code

- No obvious obfuscated malware-like code was found in reviewed source.
- Many generated/static assets and large UI files exist, but I did not see packed eval-heavy payloads or suspicious minified blobs in source directories reviewed.

### Install/build scripts

- Main `package.json` has `postinstall`: `npm run workspace:build` (`package.json:44`), which resolves to TypeScript/Vite builds (`package.json:57`).
- `install:all` runs `npm install && node install-base.js` (`package.json:43`).
- `install:yt-dlp` runs `node install-yt-dlp.js` (`package.json:42`), which creates `.venv` and installs `yt-dlp`, `edge-tts`, and `gtts`.
- `OSG_installer.sh` uses privileged/system install operations:
  - Homebrew installer via `curl | bash`: `OSG_installer.sh:99-102`
  - `sudo apt update/install`: `OSG_installer.sh:148-166`
  - NodeSource setup via `curl | sudo bash`: `OSG_installer.sh:163-166`
- Uninstaller scripts remove app/system components and caches. They include broad removals such as `rm -rf "$HOME/.npm"` and package manager uninstall commands. Treat them as destructive maintenance scripts, not safe audit commands.

### Detailed installer review: `OSG_installer.sh`

Static review result: **CAUTION**.

What it does:

- Must be run from the existing repository root. It checks for `package.json` and `server.js` before showing the menu (`OSG_installer.sh:719-739`).
- On Linux, asks for sudo immediately before showing the menu (`OSG_installer.sh:686-711`).
- Installs prerequisites:
  - macOS: Homebrew via `curl` piped into `/bin/bash`, then `brew install git/node/ffmpeg` (`OSG_installer.sh:98-137`).
  - Linux: `sudo apt update`, `sudo apt install git`, NodeSource setup via `curl | sudo bash`, `sudo apt install nodejs`, and `sudo apt install ffmpeg` (`OSG_installer.sh:148-176`).
  - Cross-platform uv install via `curl ... | sh` (`OSG_installer.sh:187-190`).
- Clean install removes only project-local/generated paths: `node_modules`, `.venv`, `chatterbox/chatterbox`, and `package-lock.json` (`OSG_installer.sh:244-270`).
- Installation then runs `node setup-workspaces.js`, `npm run install:all`, `npm run install:yt-dlp`, and starts `npm run dev` (`OSG_installer.sh:391-431`).
- Update flow runs `git reset --hard origin/main`, `git pull`, optional `uv pip install --upgrade yt-dlp`, and optional `npm install` (`OSG_installer.sh:496-544`).
- Uninstall asks confirmation, then changes to the parent directory and runs `rm -rf "$PROJECT_NAME"` where `PROJECT_NAME` is the basename of current working directory (`OSG_installer.sh:630-679`).

Risk notes:

- `curl | bash/sh` is high trust: it executes remote installer code without local inspection (`OSG_installer.sh:101`, `OSG_installer.sh:165`, `OSG_installer.sh:190`).
- It requests sudo at startup even if the user later chooses a non-install option (`OSG_installer.sh:686-711`).
- Update uses `git reset --hard origin/main`, which destroys local changes without confirmation (`OSG_installer.sh:516`).
- Uninstall deletes the entire current repository directory after confirmation (`OSG_installer.sh:668`). Because it validates repo structure first, this is scoped, but still destructive.
- No evidence found in this script of credential theft, reading browser profiles, exfiltration, or hidden network calls beyond package/repository installer sources.

### Detailed installer review: `OSG_installer_Windows.bat`

Static review result: **CAUTION / HIGH CAUTION**.

What it does:

- Immediately requests Administrator privileges and relaunches itself elevated if needed (`OSG_installer_Windows.bat:30-32`).
- Has a self-update mechanism:
  - Checks GitHub latest release API (`OSG_installer_Windows.bat:34-45`).
  - If user accepts, downloads `OSG_installer_Windows.bat` from GitHub release assets (`OSG_installer_Windows.bat:79-89`).
  - Validation is weak: only checks file exists, size > 5000, and contains `PROJECT_FOLDER_NAME` (`OSG_installer_Windows.bat:83-85`). No signature/hash verification.
- Install flow:
  - Installs prerequisites via `winget` for Git, Node.js, FFmpeg and via `irm https://astral.sh/uv/install.ps1 | iex` for uv (`OSG_installer_Windows.bat:510-524`).
  - Runs `git clone` from `https://github.com/nganlinh4/oneclick-subtitles-generator.git` into `%SCRIPT_DIR%\oneclick-subtitles-generator` (`OSG_installer_Windows.bat:152-153`).
  - Runs `node setup-workspaces.js`, `npm run install:all`, `npm run install:yt-dlp`, and then `npm run dev` (`OSG_installer_Windows.bat:168-194`).
- Clean install deletes the existing `%SCRIPT_DIR%\oneclick-subtitles-generator` folder without an extra confirmation after the user selects Install (`OSG_installer_Windows.bat:386-403`).
- Update flow runs `git reset --hard origin/main`, `git pull`, `uv pip install --upgrade yt-dlp`, then `npm install` (`OSG_installer_Windows.bat:216-246`).
- It changes PowerShell execution policy to `RemoteSigned` for CurrentUser (`OSG_installer_Windows.bat:373-374`).
- It attempts to enable Windows Hardware-accelerated GPU scheduling by writing `HKLM:\SYSTEM\CurrentControlSet\Control\GraphicsDrivers\HwSchMode = 2` (`OSG_installer_Windows.bat:415-443`).
- Uninstall asks confirmation and deletes `%PROJECT_PATH%` via `RMDIR /S /Q` (`OSG_installer_Windows.bat:283-313`).

Risk notes:

- Running the whole script as Administrator increases blast radius for every later command, including self-update, `npm`, `git`, and dependency install steps.
- Self-update downloads executable batch code from GitHub release and replaces the local installer without cryptographic verification. This is not evidence of malware, but it is a supply-chain risk.
- `irm ... | iex` executes remote PowerShell code for uv installation (`OSG_installer_Windows.bat:524`).
- Clean install can silently delete an existing project folder on option 1 (`OSG_installer_Windows.bat:149`, `OSG_installer_Windows.bat:393`).
- Update uses `git reset --hard origin/main`, which destroys local edits (`OSG_installer_Windows.bat:217`).
- It modifies user PowerShell policy and HKLM graphics registry settings; those are system/user configuration changes beyond normal project setup.
- No evidence found in this script of stealing tokens/passwords/cookies or sending private files to unknown servers. Network calls are to GitHub, winget sources, and Astral uv installer.

### System command execution

System command execution is common in this repo and mostly supports media processing/setup:

- `child_process.spawn/exec/execSync` used for `yt-dlp`, `ffmpeg`, port cleanup, Electron service startup, Python services, build/setup scripts.
- Python uses `subprocess.run` for ffmpeg/media operations.
- I did not find `os.system` or `shell=True` patterns in the reviewed Python results.
- Risk: any route that passes user-provided URLs/paths into command arguments must continue using `spawn(..., args)` style. The reviewed yt-dlp downloaders generally use argument arrays, which is safer than shell string interpolation.

### File write/delete behavior

Expected writes/deletes:

- Creates project-local `videos`, `subtitles`, `narration`, album art, lyrics, generated env/config files.
- Deletes temporary extracted/download files and partial downloads.
- Build scripts remove/copy venv and build artifacts.
- Installer/uninstaller scripts can delete `node_modules`, `.venv`, `F5-TTS`, caches, and sometimes global user caches.

No evidence was found of silent destructive deletion outside expected install/cache/uninstall paths, but the installer/uninstaller scripts are powerful and should not be run casually.

## 6. Ket luan an toan

Classification: **CAUTION**

Reasoning:

- I did not find clear evidence of malware, backdoor, hidden exfiltration, SSH key/password harvesting, arbitrary credential theft, or unknown command-and-control endpoints.
- The repo does intentionally handle sensitive data: Gemini API keys, Genius token, YouTube OAuth tokens/client secret, browser cookies for yt-dlp, local media files, and user subtitle/video content.
- Browser cookie use is especially sensitive. The code detects browser profile locations and passes `--cookies-from-browser` to yt-dlp when the cookie option is enabled. This can be legitimate for downloading restricted/high-quality videos, but it should be opt-in and tested in isolation.
- Installer scripts make system-level changes with `sudo`, `curl | bash`, package managers, Python package installs, GitHub downloads, and destructive cleanup commands.
- Network traffic is expected for core features and goes to recognized providers, but using the app means sending selected content to third-party APIs.

There is not enough evidence to classify this as `RISKY`, but it is not `SAFE` for direct first run on a primary machine with real tokens/browser profiles.

## 7. Khuyen nghi truoc khi chay tren Ubuntu

Checklist:

- Run first in a disposable VM or container, not on your main user account.
- Do not mount `$HOME`, browser profiles, `.ssh`, password stores, or real project secrets into the sandbox.
- Use a throwaway Linux user if running outside Docker.
- Do not pass real Gemini/Google/Genius API keys on the first run.
- Keep `use_cookies_for_download` disabled for first tests.
- If testing browser-cookie download, use a separate browser profile with a throwaway account.
- Block or monitor network first: run with a local firewall, `tcpdump`, `mitmproxy`, or Docker network controls.
- Avoid `OSG_installer.sh` on the host until reviewed; prefer manual package installation inside a VM/container.
- Avoid uninstaller scripts on a real machine because they can remove system packages and user caches.
- Use least privilege: do not run app/server as root.
- Prefer explicit commands over all-in-one install scripts:
  - review `package.json`
  - run `npm ci --ignore-scripts` first if only inspecting dependencies
  - then run workspace build scripts manually only after review
  - create `.venv` inside the project or container only
- Keep real tokens out of browser localStorage during audit; use dummy tokens or a restricted API key with low quota and no sensitive permissions.
- Inspect generated files after first run: `.env.local`, `localStorage.json`, `videos/`, `subtitles/`, `narration/`, `.venv/`, and temp cookie files under `/tmp`.
