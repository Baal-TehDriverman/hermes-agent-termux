# Hermes Agent — Termux/Android Setup

Complete setup for running Hermes Agent on Termux (Android) for NSSP mesh edge nodes.

## Prerequisites

- Termux (F-Droid build recommended, NOT Play Store)
- Android 7+ (API 24+)
- ~4GB free space
- Storage permission: `termux-setup-storage`

## One-Line Bootstrap

```bash
pkg update && pkg upgrade -y && \
pkg install -y python git nodejs-lts rust openssl libffi zlib && \
pip install --upgrade pip && \
pip install --no-deps -r requirements.txt 2>/dev/null || pip install -r requirements.txt
```

## Full Step-by-Step

### 1. Base System

```bash
# Update packages
pkg update && pkg upgrade -y

# Core dependencies
pkg install -y \
  python git nodejs-lts rust \
  openssl libffi zlib \
  clang make cmake pkg-config \
  termux-api termux-tools

# Storage access (run once)
termux-setup-storage
```

### 2. Python Environment

```bash
# Use system python (venv has issues with native deps on arm64)
python -m ensurepip --upgrade
pip install --upgrade pip setuptools wheel

# Install Hermes deps (--no-deps avoids maturin/native build failures)
pip install --no-deps -r requirements.txt 2>/dev/null || pip install -r requirements.txt

# If maturin/cargo build fails, use pre-built wheels or skip:
# pip install --only-binary :all: -r requirements.txt
```

### 3. Node.js / CLI Dependencies

```bash
# For lilith-cli-android and esbuild
npm install -g bun  # or use node directly
```

### 4. Ollama (Local LLM Runtime)

```bash
# Install Ollama binary
curl -fsSL https://ollama.com/install.sh | sh

# Or manual arm64 binary:
# curl -L https://github.com/ollama/ollama/releases/latest/download/ollama-linux-arm64 -o ~/bin/ollama && chmod +x ~/bin/ollama

# Start Ollama daemon (background)
ollama serve &

# Pull models for edge node
ollama pull gemma3:1b       # lightweight edge model
ollama pull nemotron-3-super:cloud  # tool-calling (if space)
ollama pull qwen2.5:1.5b    # alternative edge
```

### 5. Hermes Configuration

```bash
# Create config directory
mkdir -p ~/.hermes/profiles/default

# Minimal config.yaml for Termux
cat > ~/.hermes/profiles/default/config.yaml << 'EOF'
profiles:
  default:
    model:
      provider: ollama
      model: gemma3:1b
      base_url: http://127.0.0.1:11434/v1
    context:
      max_tokens: 65536
    tools:
      enabled: [terminal, file, search, web, delegation]
    delegation:
      max_concurrent_children: 2
    tts:
      provider: edge
EOF
```

### 6. Start Hermes

```bash
# Direct CLI
python -m hermes_agent

# Or install as command (after pip install -e .)
hermes
```

### 7. NSSP Mesh Integration

```bash
# Clone mesh repos
cd ~
git clone https://github.com/Baal-TehDriverman/lilith-nssp-mesh
git clone https://github.com/Baal-TehDriverman/lilith-cli-android
git clone https://github.com/Baal-TehDriverman/vm-ai-gateway

# lilith-cli-android build
cd ~/lilith-cli-android
npm install
npm run build

# nssp-infer.sh helper (already in ~/nssp/)
chmod +x ~/nssp/nssp-infer.sh
```

### 8. Background Services (Termux:Boot)

Create `~/.termux/boot/hermes-services.sh`:

```bash
#!/data/data/com.termux/files/usr/bin/bash
# Start Ollama
ollama serve > ~/ollama.log 2>&1 &

# Start NSSP poller (if configured)
~/nssp/phone-poller.sh > ~/nssp-poller.log 2>&1 &

# Start Hermes gateway (optional)
cd ~/vm-ai-gateway && python -m uvicorn main:app --host 0.0.0.0 --port 8000 > ~/gateway.log 2>&1 &
```

```bash
chmod +x ~/.termux/boot/hermes-services.sh
```

## Known Termux Issues & Workarounds

| Issue | Workaround |
|-------|------------|
| `maturin` / native wheels fail | `pip install --only-binary :all:` or `--no-deps` |
| `ls /storage/emulated/0` → ENOSYS | Use `python3 -c "import os; print(os.listdir(...))"` |
| Symlinks on /sdcard fail | `cp -a` instead of `cp -s`; ignore kernel-source symlink errors |
| `venv` breaks native modules | Use system python directly |
| `proot-distro` kali image missing | Use `danhunsaker/archlinuxarm` + `debian:stable` |
| `vite-plugin-pwa` workbox terser crash | Set `workbox.mode='development'` in vite config |

## Minimal Disk Footprint

If space is tight, skip:
- `kernel-source` (1.3G)
- `NVIDIA-TOOLS` (3G)
- `hermes-agent` venv (use system python)
- Large models (>2GB)

Keep: `lilith-cli-android`, `nssp`, `vm-ai-gateway`, `ollama`, `gemma3:1b`

## Verify Everything Works

```bash
# Test Ollama
curl http://127.0.0.1:11434/api/tags

# Test Hermes
hermes --version
python -c "from hermes_agent import main; print('import ok')"

# Test NSSP
~/nssp/nssp-infer.sh "test prompt"
```

## Repo Locations

| Repo | Purpose |
|------|---------|
| `Baal-TehDriverman/hermes-agent-termux` | This fork |
| `Baal-TehDriverman/lilith-cli-android` | Android CLI (TypeScript) |
| `Baal-TehDriverman/lilith-nssp-mesh` | GitHub-based task queue |
| `Baal-TehDriverman/vm-ai-gateway` | FastAPI gateway |
| `Baal-TehDriverman/nssp-build` | mkosi/Arch image builder |
| `Baal-TehDriverman/nssp-kernel` | Custom Android kernel |

## Credits

Hermes Agent core by Nous Research. Termux/Android adaptations for NSSP mesh by Lilith-Systems.