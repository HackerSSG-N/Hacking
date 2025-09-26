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

--- 

### Phase 1 — Clone libwebp source

```curl
git clone https://chromium.googlesource.com/webm/libwebp
cd libwebp
```

- Purpose: get the libwebp source tree.
- Expected: new libwebp/ folder and cd libwebp places you in repo root. git rev-parse --abbrev-ref HEAD should show branch (e.g., master or main).
- If not: ensure git installed and URL is reachable.

--- 

### Phase 2 — Build libwebp with ASan enabled (so bugs crash loudly)


```curl
export CC=clang
export CFLAGS="-g -O1 -fno-omit-frame-pointer -fsanitize=address,undefined -fno-sanitize-recover=all"
export CXX=clang++
export CXXFLAGS="$CFLAGS"
# create build dir, install into build/install so we can link easily
mkdir -p build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Debug -DBUILD_SHARED_LIBS=OFF -DCMAKE_INSTALL_PREFIX="$PWD/install"
make -j"$(nproc)"
make install
```

- Purpose: compile libwebp statically with ASan and put headers/libs in build/install. -O1 keeps fuzzing performant and debuggable.
- Expected: build completes, and build/install/lib contains libwebp.a (static) and build/install/include contains headers. cwebp/dwebp may also be in build/install/bin.
- If not / troubleshooting:

- CMake errors: read the messages — install missing dev packages. For example missing libpng/zlib etc — install libpng-dev zlib1g-dev.

- If only shared libs built: redo cmake with -DBUILD_SHARED_LIBS=OFF.
- If ASan complains during build: ensure clang is used (check which clang and clang --version).
