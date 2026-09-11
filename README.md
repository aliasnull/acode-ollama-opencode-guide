# Acode + Alpine Linux + Ollama Cloud + OpenCode

Run **OpenCode on Android inside Acode's built-in terminal**, using
**Ollama locally as the bridge to an Ollama Cloud model**.

This guide documents the complete setup from a fresh Acode terminal
through a working OpenCode + `gemma4:31b-cloud` setup, including the
Alpine/musl compatibility problem, permanent fixes, automatic Ollama
startup, configuration, verification, and troubleshooting.

> **Important:** This guide does not use `proot`, `chroot`, a direct
> Ollama Cloud API key inside OpenCode, or locally downloaded 31B model
> weights.

------------------------------------------------------------------------

## 1. What this setup does

The final architecture is:

``` text
Android
└── Acode
    └── Acode Terminal
        └── Alpine Linux 3.21 (aarch64 / musl)
            ├── OpenCode
            │   └── custom Ollama provider
            │       └── http://127.0.0.1:11434/v1
            │
            └── Ollama
                └── Ollama Cloud
                    └── gemma4:31b-cloud
```

The important distinction is that **OpenCode talks to local Ollama**,
while Ollama handles the Cloud model connection.

The Cloud model itself is not downloaded as a 31B model onto the phone.

------------------------------------------------------------------------

## 2. Tested environment

This guide was built and tested with:

  Component             Tested value
  --------------------- ---------------------
  Android environment   Acode
  Terminal OS           Alpine Linux 3.21.6
  Architecture          `aarch64` / ARM64
  libc                  musl
  Node.js               `v22.23.2`
  npm                   `10.9.1`
  Ollama                `0.34.0`
  OpenCode              `1.18.30`
  Model                 `gemma4:31b-cloud`
  Ollama endpoint       `127.0.0.1:11434`

Your versions may differ. The important parts are ARM64 Alpine, a
working Node/npm environment, Ollama, and OpenCode.

------------------------------------------------------------------------

# Part 1 --- Prepare Acode Terminal

## 3. Verify the Alpine environment

Run:

``` sh
cat /etc/alpine-release
```

Then:

``` sh
uname -m
```

Expected architecture:

``` text
aarch64
```

Update package indexes:

``` sh
apk update
```

------------------------------------------------------------------------

## 4. Install basic packages

The exact package list can vary depending on the existing Acode image.

For the troubleshooting/build environment used in this project:

``` sh
apk add zstd file gcompat ninja-build go
```

You may also need common build tools later:

``` sh
apk add gcc g++ make cmake
```

Do not install `proot`. It is not required for this setup.

------------------------------------------------------------------------

# Part 2 --- Install OpenCode

## 5. Install OpenCode

``` sh
npm install -g opencode-ai
```

Check:

``` sh
opencode --version
```

### Alpine/musl issue

On Alpine, the normal OpenCode ARM64 binary may be a glibc build and can
fail with relocation errors.

The working solution is to install/use the **musl OpenCode package**:

``` sh
npm pack opencode-linux-arm64-musl@1.18.30
```

Extract it:

``` sh
mkdir -p /tmp/opencode-test
tar -xzf opencode-linux-arm64-musl-1.18.30.tgz -C /tmp/opencode-test
```

Test the binary:

``` sh
/tmp/opencode-test/package/bin/opencode --version
```

Expected:

``` text
1.18.30
```

### Make OpenCode permanent

Do not leave the executable under `/tmp`.

Copy it:

``` sh
cp /tmp/opencode-test/package/bin/opencode /usr/local/bin/opencode
chmod +x /usr/local/bin/opencode
```

Verify:

``` sh
opencode --version
```

------------------------------------------------------------------------

# Part 3 --- Install Ollama

## 6. Install Ollama

Official Linux installation:

``` sh
curl -fsSL https://ollama.com/install.sh | sh
```

Official Ollama Linux installation documentation:
https://ollama.com/download/linux

### Alpine problem

The official Ollama ARM64 binary is glibc-based.

On Alpine/musl, running it directly can produce:

``` text
Error relocating /usr/local/bin/ollama:
fcntl64: symbol not found
```

Do **not** switch to proot just to solve this.

Instead, provide a glibc runtime for the Ollama executable.

------------------------------------------------------------------------

# Part 4 --- Add glibc compatibility without proot

## 7. Download ARM64 glibc

The working ARM64 glibc package used in this setup was:

``` text
https://github.com/dalet-oss/alpine-glibc/releases/download/2.43-arm64/glibc-2.43.apk
```

Download:

``` sh
wget -O /tmp/glibc-2.43.apk \
  https://github.com/dalet-oss/alpine-glibc/releases/download/2.43-arm64/glibc-2.43.apk
```

Extract it:

``` sh
mkdir -p /tmp/glibc
tar -xzf /tmp/glibc-2.43.apk -C /tmp/glibc
```

Verify the loader:

``` sh
ls -l /tmp/glibc/usr/glibc-compat/lib/ld-linux-aarch64.so.1
```

------------------------------------------------------------------------

## 8. Test Ollama through the glibc loader

The important runtime is:

``` sh
/tmp/glibc/usr/glibc-compat/lib/ld-linux-aarch64.so.1 \
  --library-path /tmp/glibc/usr/glibc-compat/lib:/lib:/usr/lib:/usr/local/lib/ollama \
  /usr/local/bin/ollama --version
```

Expected:

``` text
Warning: could not connect to a running Ollama instance
Warning: client version is 0.34.0
```

The connection warning is normal if the server is not running yet.

------------------------------------------------------------------------

# Part 5 --- Make glibc permanent

## 9. Copy glibc to a persistent location

Do not rely on `/tmp`.

``` sh
mkdir -p /usr/local/lib/glibc-2.43
cp -a /tmp/glibc/usr/glibc-compat/. /usr/local/lib/glibc-2.43/
```

Verify:

``` sh
ls -l /usr/local/lib/glibc-2.43/lib/ld-linux-aarch64.so.1
```

------------------------------------------------------------------------

# Part 6 --- Create the permanent Ollama wrapper

## 10. Protect the original Ollama executable

The original installer places the glibc Ollama executable at:

``` text
/usr/local/bin/ollama
```

Move it out of the normal command path:

``` sh
mv /usr/local/bin/ollama /usr/local/lib/ollama-bin
```

Now `/usr/local/bin/ollama` can safely become a wrapper.

Create:

``` sh
cat > /usr/local/bin/ollama <<'EOF'
#!/bin/sh
exec /usr/local/lib/glibc-2.43/lib/ld-linux-aarch64.so.1 \
  --library-path /usr/local/lib/glibc-2.43/lib:/lib:/usr/lib:/usr/local/lib/ollama \
  /usr/local/lib/ollama-bin "$@"
EOF
```

Make it executable:

``` sh
chmod +x /usr/local/bin/ollama
```

Verify:

``` sh
ollama --version
```

Expected:

``` text
Warning: could not connect to a running Ollama instance
Warning: client version is 0.34.0
```

The warning only means the server is not currently running.

------------------------------------------------------------------------

# Part 7 --- Start Ollama

## 11. Start the Ollama server

``` sh
ollama serve
```

A successful server should report something similar to:

``` text
Ollama cloud disabled: false
Listening on 127.0.0.1:11434
version 0.34.0
```

The server listens locally on:

``` text
http://127.0.0.1:11434
```

Verify from another terminal:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

Expected:

``` json
{"version":"0.34.0"}
```

------------------------------------------------------------------------

# Part 8 --- Sign in to Ollama Cloud

## 12. Authenticate Ollama

Run the Cloud model through Ollama:

``` sh
ollama run gemma4:31b-cloud
```

If Ollama asks you to sign in, complete the browser authentication.

After authentication, you should see something similar to:

``` text
Connecting to 'gemma4:31b-cloud' on 'ollama.com'
```

This is the important point:

-   OpenCode does not need a direct Ollama Cloud API key for this
    architecture.
-   OpenCode communicates with local Ollama.
-   Ollama manages the Cloud model connection.

Ollama documents Cloud models as models that run on Ollama's cloud
infrastructure while remaining usable through the normal Ollama
workflow.

------------------------------------------------------------------------

# Part 9 --- Configure OpenCode

## 13. Create the OpenCode config directory

``` sh
mkdir -p ~/.config/opencode
```

Create:

``` text
~/.config/opencode/opencode.jsonc
```

For OpenCode 1.18.x, use the **singular `provider` key** and the `npm` +
`options.baseURL` format:

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

### Important version note

OpenCode has different configuration formats across generations.

For the tested OpenCode `1.18.30`, the working configuration is:

``` text
provider
  └── ollama
      ├── npm
      ├── name
      ├── options.baseURL
      └── models
```

Do **not** blindly replace it with the newer experimental/v2:

``` text
providers
```

format when following this guide for OpenCode 1.18.x.

Current OpenCode documentation for the classic provider configuration
shows the same `provider` + `@ai-sdk/openai-compatible` +
`options.baseURL` pattern for Ollama. citeturn0search0turn0search1

------------------------------------------------------------------------

# Part 10 --- Verify OpenCode sees the provider

## 14. List the Ollama models

``` sh
opencode models ollama
```

Expected:

``` text
ollama/gemma4:31b-cloud
```

If you get:

``` text
Error: Provider not found: ollama
```

check that your config uses:

``` json
"provider": {
```

and not:

``` json
"providers": {
```

for OpenCode 1.18.x.

------------------------------------------------------------------------

# Part 11 --- Test the complete chain

## 15. Run OpenCode directly

``` sh
opencode run -m ollama/gemma4:31b-cloud \
  "Say hello in one short sentence."
```

Expected example:

``` text
> build · gemma4:31b-cloud

Hello!
```

A more explicit test:

``` sh
opencode run -m ollama/gemma4:31b-cloud \
  "Reply with exactly: OpenCode to Ollama Cloud is working."
```

Expected:

``` text
OpenCode to Ollama Cloud is working.
```

At this point the complete path is confirmed:

``` text
Acode
  ↓
OpenCode
  ↓
127.0.0.1:11434
  ↓
Ollama
  ↓
Ollama Cloud
  ↓
gemma4:31b-cloud
```

------------------------------------------------------------------------

# Part 12 --- Make Ollama start automatically

Manually running:

``` sh
ollama serve
```

every time is inconvenient.

The solution below starts Ollama in the background when a new Bash shell
loads.

## 16. Check the shell

``` sh
echo "$SHELL"
```

Expected:

``` text
/bin/bash
```

Check for an existing Ollama server:

``` sh
pgrep -af 'ollama serve'
```

------------------------------------------------------------------------

## 17. Create a safe startup script

Create:

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

Test manually:

``` sh
start-ollama
```

Verify:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

Expected:

``` json
{"version":"0.34.0"}
```

The `pgrep` guard is important because it prevents every new Bash shell
from starting another Ollama server.

------------------------------------------------------------------------

# Part 13 --- Add automatic startup to Bash

## 18. Inspect `.bashrc` before changing it

Always inspect first:

``` sh
cat ~/.bashrc
```

If you have old provider/API configurations you no longer use, remove
only those specific lines. Do not blindly overwrite the whole file.

Then add:

``` sh
printf '\n# Start Ollama Cloud bridge\nstart-ollama\n' >> ~/.bashrc
```

Test a fresh Bash shell:

``` sh
bash -c 'sleep 1; curl -s http://127.0.0.1:11434/api/version'
```

Expected:

``` json
{"version":"0.34.0"}
```

Now closing the old Acode terminal and opening a new one should
automatically start the Ollama server.

------------------------------------------------------------------------

# Part 14 --- Normal daily usage

After the setup is complete, the intended workflow is simple.

Open Acode Terminal.

Run:

``` sh
opencode
```

Inside OpenCode, open the model selector and choose:

``` text
Ollama Cloud via Local Ollama
└── Gemma 4 31B Cloud
```

Or run directly:

``` sh
opencode run -m ollama/gemma4:31b-cloud
```

You should no longer need to manually run:

``` sh
ollama serve
```

because `.bashrc` starts the background bridge.

------------------------------------------------------------------------

# Troubleshooting

## Error: `fcntl64: symbol not found`

Example:

``` text
Error relocating /usr/local/bin/ollama:
fcntl64: symbol not found
```

Cause:

Ollama's ARM64 executable is glibc-based while Alpine uses musl.

Solution:

Use the persistent glibc loader/wrapper described in Part 4--6.

Do not use `proot` as a workaround.

------------------------------------------------------------------------

## Error: `Provider not found: ollama`

Check:

``` sh
opencode models ollama
```

Make sure the OpenCode 1.18.x config uses:

``` json
"provider": {
```

not:

``` json
"providers": {
```

Also verify:

``` sh
cat ~/.config/opencode/opencode.jsonc
```

------------------------------------------------------------------------

## Ollama is not reachable

Test:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

If nothing is returned, check:

``` sh
pgrep -af 'ollama'
```

Then check the startup log:

``` sh
cat /tmp/ollama.log
```

You can manually test:

``` sh
ollama serve
```

------------------------------------------------------------------------

## OpenCode works but the model is missing

Run:

``` sh
opencode models ollama
```

Expected:

``` text
ollama/gemma4:31b-cloud
```

Also make sure Ollama is running:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

------------------------------------------------------------------------

## OpenCode environment troubleshooting

Check the shell environment:

``` sh
env | grep -E 'OPENCODE|OLLAMA'
```

If OpenCode is not using the expected provider, check that your shell
environment does not contain unrelated variables that override the
intended setup.

------------------------------------------------------------------------

## `llama-server binary not found`

Ollama may print a message similar to:

``` text
failure during llama-server GPU discovery
...
llama-server binary not found
```

This does not necessarily prevent Ollama from starting.

If the log also shows:

``` text
Listening on 127.0.0.1:11434
```

and:

``` sh
curl -s http://127.0.0.1:11434/api/version
```

returns a version, the Ollama server itself is running.

------------------------------------------------------------------------

# Security

## Keep authentication files private

Do not commit personal authentication files, session data, or private
configuration files to GitHub.

Keep these private:

``` text
Ollama authentication/session files
personal OpenCode configuration
terminal history
private credentials
```

The repository should contain example configuration files only, not
personal account data.

------------------------------------------------------------------------

# Suggested GitHub repository structure

``` text
acode-ollama-opencode/
├── README.md
├── docs/
│   ├── troubleshooting.md
│   ├── architecture.md
│   └── alpine-musl.md
├── config/
│   └── opencode.jsonc.example
├── scripts/
│   └── start-ollama.sh
├── LICENSE
└── .gitignore
```

Do not commit:

``` text
~/.local/share/opencode/auth.json
```

or any file containing credentials.

------------------------------------------------------------------------

# Reference configuration

`config/opencode.jsonc.example`:

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

Reference startup script:

``` sh
#!/bin/sh

if ! pgrep -x ollama >/dev/null 2>&1; then
  nohup ollama serve >/tmp/ollama.log 2>&1 &
fi
```

------------------------------------------------------------------------

# Why this setup is useful

This setup combines:

-   Acode as the Android development environment
-   Alpine Linux as the terminal environment
-   OpenCode as the coding/agent interface
-   Ollama as the local OpenAI-compatible bridge
-   Ollama Cloud for large remote models
-   No proot/chroot
-   No 31B model weights stored locally
-   Automatic Ollama startup
-   A local endpoint for OpenCode

The same architecture can later be adapted to other Ollama Cloud models.

For example, after authenticating with Ollama, a different cloud model
can be used by changing the model ID in the OpenCode configuration.

------------------------------------------------------------------------

# Important limitation

This architecture does **not** make a Cloud model unlimited or bypass
Ollama account/usage policies.

The local component is the bridge:

``` text
OpenCode → local Ollama
```

but the inference for a `*-cloud` model still occurs on Ollama's Cloud
infrastructure.

If you want genuinely quota-free inference, the model must eventually
run locally on the device. That is a separate setup and has
hardware/performance limitations.

------------------------------------------------------------------------

# Official references

-   Ollama Linux installation: https://ollama.com/download/linux
-   Ollama Cloud models: https://ollama.com/blog/cloud-models
-   OpenCode providers: https://opencode.ai/docs/providers
-   OpenCode models: https://opencode.ai/docs/models

------------------------------------------------------------------------

# Final verification checklist

A successful installation should satisfy all of these:

``` text
[✓] Acode terminal works
[✓] Alpine ARM64 detected
[✓] OpenCode installed
[✓] OpenCode musl binary works
[✓] OpenCode moved out of /tmp
[✓] Ollama installed
[✓] glibc compatibility added without proot
[✓] Ollama wrapper works
[✓] Ollama server listens on 127.0.0.1:11434
[✓] Ollama Cloud authentication completed
[✓] gemma4:31b-cloud works through Ollama
[✓] OpenCode custom Ollama provider registered
[✓] ollama/gemma4:31b-cloud appears in OpenCode
[✓] OpenCode successfully sends a request
[✓] Ollama starts automatically with Bash
```

If all checks pass, the Android/Acode → OpenCode → Ollama → Ollama Cloud
setup is complete.
