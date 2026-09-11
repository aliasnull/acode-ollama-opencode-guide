# Acode + Alpine Linux + Ollama Cloud + OpenCode

Run OpenCode on Android inside Acode's built-in terminal using Ollama as
a local bridge to an Ollama Cloud model.

This guide documents the complete working setup:

Acode Terminal → Alpine Linux → OpenCode → Ollama → Ollama Cloud →
gemma4:31b-cloud

It includes the problems encountered during setup and their fixes: -
Acode Alpine repository issue - OpenCode ARM64/musl compatibility
issue - Ollama glibc compatibility issue - OpenCode provider
configuration - Automatic Ollama startup

------------------------------------------------------------------------

# 1. Architecture

``` text
Android
└── Acode
    └── Acode Terminal
        └── Alpine Linux (aarch64 / musl)
            ├── OpenCode
            │   └── Ollama provider
            │       └── http://127.0.0.1:11434/v1
            │
            └── Ollama
                └── Ollama Cloud
                    └── gemma4:31b-cloud
```

OpenCode communicates with the local Ollama server. Ollama handles the
Cloud model connection.

------------------------------------------------------------------------

# 2. Prepare Acode Terminal

## Check environment

``` sh
cat /etc/alpine-release
uname -m
```

Expected architecture:

``` text
aarch64
```

------------------------------------------------------------------------

## Fix Acode Alpine repositories

Acode Alpine may not have correct repositories configured.

Check:

``` sh
cat /etc/apk/repositories
```

If missing or incorrect:

``` sh
cat > /etc/apk/repositories <<'EOF'
https://dl-cdn.alpinelinux.org/alpine/v3.21/main
https://dl-cdn.alpinelinux.org/alpine/v3.21/community
EOF
```

Update:

``` sh
apk update
```

Install required packages:

``` sh
apk add zstd file gcompat ninja-build go
```

------------------------------------------------------------------------

# 3. Install OpenCode

Install:

``` sh
npm install -g opencode-ai
```

Check:

``` sh
opencode --version
```

## Alpine musl issue

The normal OpenCode ARM64 binary may fail on Alpine because Alpine uses
musl instead of glibc.

Install the musl package:

``` sh
npm pack opencode-linux-arm64-musl@1.18.30
```

Extract:

``` sh
mkdir -p /tmp/opencode-test
tar -xzf opencode-linux-arm64-musl-1.18.30.tgz -C /tmp/opencode-test
```

Test:

``` sh
/tmp/opencode-test/package/bin/opencode --version
```

Copy permanently:

``` sh
cp /tmp/opencode-test/package/bin/opencode /usr/local/bin/opencode
chmod +x /usr/local/bin/opencode
```

Verify:

``` sh
opencode --version
```

------------------------------------------------------------------------

# 4. Install Ollama

Install:

``` sh
curl -fsSL https://ollama.com/install.sh | sh
```

## Alpine glibc issue

Running Ollama directly on Alpine can show:

``` text
Error relocating /usr/local/bin/ollama:
fcntl64: symbol not found
```

This happens because Ollama uses glibc while Alpine uses musl.

------------------------------------------------------------------------

# 5. Add glibc compatibility

Download ARM64 glibc:

``` sh
wget -O /tmp/glibc-2.43.apk \
https://github.com/dalet-oss/alpine-glibc/releases/download/2.43-arm64/glibc-2.43.apk
```

Extract:

``` sh
mkdir -p /tmp/glibc
tar -xzf /tmp/glibc-2.43.apk -C /tmp/glibc
```

Copy permanently:

``` sh
mkdir -p /usr/local/lib/glibc-2.43
cp -a /tmp/glibc/usr/glibc-compat/. /usr/local/lib/glibc-2.43/
```

------------------------------------------------------------------------

# 6. Create Ollama wrapper

Move original binary:

``` sh
mv /usr/local/bin/ollama /usr/local/lib/ollama-bin
```

Create wrapper:

``` sh
cat > /usr/local/bin/ollama <<'EOF'
#!/bin/sh
exec /usr/local/lib/glibc-2.43/lib/ld-linux-aarch64.so.1 \
  --library-path /usr/local/lib/glibc-2.43/lib:/lib:/usr/lib:/usr/local/lib/ollama \
  /usr/local/lib/ollama-bin "$@"
EOF
```

Make executable:

``` sh
chmod +x /usr/local/bin/ollama
```

Test:

``` sh
ollama --version
```

Expected:

``` text
Warning: could not connect to a running Ollama instance
Warning: client version is 0.34.0
```

------------------------------------------------------------------------

# 7. Start Ollama

Run:

``` sh
ollama serve
```

Verify:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

Expected:

``` json
{"version":"0.34.0"}
```

------------------------------------------------------------------------

# 8. Setup Ollama Cloud

Run:

``` sh
ollama run gemma4:31b-cloud
```

Complete Ollama authentication if requested.

Test:

``` sh
ollama list
```

------------------------------------------------------------------------

# 9. Configure OpenCode

Create directory:

``` sh
mkdir -p ~/.config/opencode
```

Create:

``` text
~/.config/opencode/opencode.jsonc
```

Add:

``` jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama Cloud via Local Ollama",
      "options": {
        "baseURL": "http://127.0.0.1:11434/v1"
      },
      "models": {
        "gemma4:31b-cloud": {
          "name": "Gemma 4 31B Cloud"
        }
      }
    }
  }
}
```

Important:

For OpenCode 1.18.x use:

``` text
provider
```

not:

``` text
providers
```

------------------------------------------------------------------------

# 10. Test OpenCode

Check model:

``` sh
opencode models ollama
```

Expected:

``` text
ollama/gemma4:31b-cloud
```

Run:

``` sh
opencode run -m ollama/gemma4:31b-cloud "Say hello in one short sentence."
```

Expected:

``` text
Hello!
```

------------------------------------------------------------------------

# 11. Automatic Ollama startup

Create startup script:

``` sh
cat > /usr/local/bin/start-ollama <<'EOF'
#!/bin/sh
if ! pgrep -x ollama >/dev/null 2>&1; then
  nohup ollama serve >/tmp/ollama.log 2>&1 &
fi
EOF
```

Make executable:

``` sh
chmod +x /usr/local/bin/start-ollama
```

Test:

``` sh
start-ollama
```

Verify:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

------------------------------------------------------------------------

# 12. Add to Bash startup

Check:

``` sh
cat ~/.bashrc
```

Add:

``` sh
printf '\n# Start Ollama Cloud bridge\nstart-ollama\n' >> ~/.bashrc
```

Test new shell:

``` sh
bash -c 'sleep 1; curl -s http://127.0.0.1:11434/api/version'
```

Now opening a new Acode terminal automatically starts Ollama.

------------------------------------------------------------------------

# Troubleshooting

## Provider not found: ollama

Check that config uses:

``` json
"provider": {
```

not:

``` json
"providers": {
```

------------------------------------------------------------------------

## Ollama not reachable

Check:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

Check process:

``` sh
pgrep -af ollama
```

Logs:

``` sh
cat /tmp/ollama.log
```

------------------------------------------------------------------------

## llama-server binary warning

If Ollama shows:

``` text
llama-server binary not found
```

but also shows:

``` text
Listening on 127.0.0.1:11434
```

and the API responds, Ollama is running correctly.

------------------------------------------------------------------------

# Final Checklist

``` text
[✓] Acode terminal working
[✓] Alpine repositories fixed
[✓] Packages installed
[✓] OpenCode installed
[✓] OpenCode musl binary working
[✓] Ollama installed
[✓] glibc compatibility added
[✓] Ollama wrapper working
[✓] Ollama server running
[✓] Ollama Cloud model working
[✓] OpenCode connected to Ollama
[✓] Automatic Ollama startup enabled
```

Setup complete.
