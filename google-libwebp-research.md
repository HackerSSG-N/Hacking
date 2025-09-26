### Phase 0 — Install required tools

```curl
sudo apt update
sudo apt install -y git clang cmake build-essential python3 python3-pip pkg-config llvm
```


- Purpose: Installs git, clang (has libFuzzer and sanitizers), cmake, compilers/tools to build libwebp.
- Expected output: packages install successfully. You can check tools:

```curl
git --version
clang --version
cmake --version
```

- If not: install failures => check network, run sudo apt --fix-broken install, or install needed packages individually.

### Phase 1 — Clone libwebp source

```curl
git clone https://chromium.googlesource.com/webm/libwebp
cd libwebp
```

- Purpose: get the libwebp source tree.
- Expected: new libwebp/ folder and cd libwebp places you in repo root. git rev-parse --abbrev-ref HEAD should show branch (e.g., master or main).
- If not: ensure git installed and URL is reachable.
