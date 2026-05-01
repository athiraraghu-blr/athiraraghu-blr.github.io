Title: Setting Up Dev Tools on Ubuntu - llama.cpp, Slack, Git and JupyterLab
Date: 2026-04-16
Category: Linux
Tags: Ubuntu, llama.cpp, Slack, Git, JupyterLab, LangChain, Linux, DevTools
Slug: setting-up-dev-tools-ubuntu-llamacpp-slack-git-jupyterlab

A practical walkthrough of setting up a local AI development environment on Ubuntu — covering llama.cpp installation via Homebrew, running local LLMs with LangChain, fixing common Slack conflicts, Git branch management, and JupyterLab. Hard-won lessons from real terminal errors.

## Installing llama.cpp via Homebrew on Ubuntu

**Homebrew on Linux** — Homebrew is primarily a macOS package manager but works on Linux too. Install it with the official script and add it to your PATH via `~/.zshrc` using `eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"`.

**brew install llama.cpp** — Installs llama.cpp and its dependencies including ggml (the tensor/ML backend) and openssl@3. The binaries land in `/home/linuxbrew/.linuxbrew/bin/` and won't be accessible until Homebrew is added to PATH.

**PATH fix is mandatory** — After install, running `llama-cli` gives `command not found` because `/home/linuxbrew/.linuxbrew/bin/` is not in PATH by default. Add `eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"` to `~/.zshrc` and source it.

**Available binaries** — llama.cpp ships with 35+ tools including `llama-cli` (inference), `llama-server` (OpenAI-compatible REST API), `llama-bench` (benchmarking), `llama-quantize` (model quantization), `llama-tts` (text-to-speech), and `llama-diffusion-cli` (diffusion models).

## Running Local LLMs with llama-server and LangChain

**llama-server** — Starts a local OpenAI-compatible REST API server on `localhost:8080`. Point any OpenAI SDK or LangChain app at it by setting `BASE_URL=http://localhost:8080/v1` in your `.env` file.

**LangChain + ChatOpenAI** — LangChain's `ChatOpenAI` class works with local llama-server by overriding `base_url`. The `api_key` is just a placeholder — llama.cpp doesn't validate it. Temperature range is `0.0` to `2.0`; setting it to `5` causes gibberish or errors.

**GGUF models are required** — llama-server needs a real `.gguf` model file. Download models via `huggingface-cli download`. Good starter models: Phi-3.5-mini-Q4 (~2.2GB for 4GB RAM), Mistral-7B-Q4 (~4.1GB for 16GB RAM).

**Port conflict fix** — If llama-server fails with `couldn't bind HTTP server socket`, port 8080 is already in use. Use `sudo lsof -i :8080` to find the process and `sudo kill -9 <PID>` to free it, or start the server on a different port with `--port 8081`.

**lsof -i explained** — `lsof` means "List Open Files." The `-i` flag filters results to show only internet/network connections. `lsof -i :8080` shows exactly which process is occupying a specific port.

## The Slack Name Conflict on Ubuntu

**Two different programs named slack** — Ubuntu has a sysadmin configuration management tool called `slack` at `/usr/sbin/slack`. It tries to rsync config from a server called `slack-master` and fails with a DNS error. This is NOT the Slack chat app.

**snap install is the cleanest route** — `sudo snap install slack` installs Slack chat app version 4.49.81 to `/snap/bin/slack`. Always launch via `snap run slack` or add `/snap/bin` before `/usr/sbin` in PATH to avoid calling the wrong binary.

**Outdated versions get blocked** — Slack versions older than a certain threshold show "This version of the app is no longer supported." The `.deb` at version 4.38 or 4.40 will be blocked on login. Use snap for the latest version automatically.

**deb conflict with sysadmin slack** — Installing `slack-desktop` via `.deb` fails with `trying to overwrite '/usr/share/lintian/overrides/slack'` because the sysadmin `slack` package owns that file. Fix with `sudo dpkg -i --force-overwrite` followed by `sudo apt install -f`.

## Git Branch Management

**git init before anything** — Running `git add` or `git status` outside a Git repo gives `fatal: not a git repository`. Always run `git init` first in the project root.

**Nested .git folders break git add** — If a subfolder has its own `.git` directory, Git treats it as a submodule and refuses to add it: `error: 'folder/' does not have a commit checked out`. Fix by removing the nested `.git` with `rm -rf subfolder/.git`.

**master vs main mismatch** — Git still defaults to `master` as the initial branch name. GitHub defaults to `main`. Push fails with `src refspec main does not match any` if you push to the wrong branch name. Fix: `git branch -m master main` then `git push -u origin main`.

**--set-upstream explained** — When you create a new local branch, Git doesn't know where to push it. `git push --set-upstream origin branch-name` (or `-u`) creates the branch on GitHub and links the local branch to track it. After that, plain `git push` works with no extra flags.

**--set-upstream vs --set-remote** — These are different. `--set-remote` changes which server (GitHub vs GitLab) a branch points to. `--set-upstream` links a local branch to a specific branch on a remote server. Remote = the post office. Upstream = the specific PO box.

## JupyterLab and Conda

**jupyter lab not python lab** — The correct command to launch JupyterLab is `jupyter lab`, not `python lab` or `pythonlab`. It opens a browser interface at `http://localhost:8888` automatically.

**conda activate first** — Always run `conda activate <env-name>` before launching JupyterLab to ensure the correct Python environment and installed packages are used inside notebooks.

## Ubuntu Dock and Font Customization

**Move dock to horizontal** — Go to Settings → Appearance → Dock → Position on screen → Bottom. Or via terminal: `gsettings set org.gnome.shell.extensions.dash-to-dock dock-position BOTTOM`.

**Increase font size** — Settings → Accessibility → Large Text toggle, or install `gnome-tweaks` (`sudo apt install gnome-tweaks`) and adjust Fonts → Scaling Factor. Terminal shortcut: `gsettings set org.gnome.desktop.interface text-scaling-factor 1.5`. Valid range is 1.0 (default) to 2.0.