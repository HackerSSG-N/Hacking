## Phase 0 — Install required tools

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

## Phase 1 — Clone libwebp source

```curl
git clone https://chromium.googlesource.com/webm/libwebp
cd libwebp
```

- Purpose: get the libwebp source tree.
- Expected: new libwebp/ folder and cd libwebp places you in repo root. git rev-parse --abbrev-ref HEAD should show branch (e.g., master or main).
- If not: ensure git installed and URL is reachable.

--- 

## Phase 2 — Build libwebp with ASan enabled (so bugs crash loudly)


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

---

## Phase 3 — Write a tiny libFuzzer harness (one file)

- Create a file simple_webp_fuzz.c in the repo root (i.e., libwebp/simple_webp_fuzz.c).

```c
cat > simple_webp_fuzz.c <<'CFA'
#include <stddef.h>
#include <stdint.h>
#include <webp/decode.h>
#include <webp/types.h>
#include <stdlib.h>

// libFuzzer entry point
int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    if (size == 0) return 0;
    // Try decode to RGBA; decoder returns NULL on invalid input.
    int width = 0, height = 0;
    // First, quick check: do not pass absurdly large sizes to avoid OOM in fuzz environment
    if (size > 50 * 1024 * 1024) return 0; // skip extremely large inputs
    uint8_t *out = WebPDecodeRGBA(data, size, &width, &height);
    if (out) {
        // touch first and last bytes to ensure memory actually mapped/accessible
        volatile uint8_t x = out[0];
        volatile uint8_t y = out[(width * height * 4) > 0 ? (width * height * 4 - 1) : 0];
        (void)x; (void)y;
        WebPFree(out);
    }
    return 0;
}
CFA
```

- Purpose: This harness feeds libFuzzer's random bytes into WebPDecodeRGBA() — the decoder you want to fuzz. It avoids massive allocations and frees output properly.
- Expected: file simple_webp_fuzz.c created.

---

## Phase 4 — Compile the harness linking libwebp & libFuzzer

- From libwebp/build (we installed into build/install), run:

```curl
# ensure we are in libwebp/build
pwd
# compile harness (adjust paths if your install prefix differs)
clang -g -O1 -fno-omit-frame-pointer \
  -fsanitize=address,undefined,fuzzer \
  -I"$PWD/install/include" \
  ../simple_webp_fuzz.c \
  -L"$PWD/install/lib" -lwebp -o simple_webp_fuzz
```
### faced error here

```curl
fatal error: 'webp/decode.h' file not found
```

- Fix: Build and install libwebp locally

- Run these commands inside ~/google-research/libwebp:

```curl
# 1. Create a build directory
mkdir build-dir && cd build-dir

# 2. Configure build with install prefix
cmake -DCMAKE_BUILD_TYPE=Debug -DCMAKE_INSTALL_PREFIX="$PWD/../install" ..

# 3. Build everything
make -j$(nproc)

# 4. Install into ../install (headers and libs go there)
make install
```

### What happens:

- After step 2 → CMake checks your system and prepares build files.
Expected output: no fatal errors. You’ll see -- Configuring done and -- Generating done.

- After step 3 → it compiles libwebp.
Expected output: lots of Building C object..., then no errors.

- After step 4 → it copies headers to ../install/include/webp/ and libs to ../install/lib/.
Expected output: you should now have:

```curl
ls ../install/include/webp
# should show decode.h, encode.h, etc.

ls ../install/lib
# should show libwebp.a or libwebp.so
```
 
- check harness built or not

```curl
ls -lh simple_webp_fuzz
```

---

## Phase 5 — Create seed corpus


```curl
# from build/ or repo root (adjust path to simple_webp_fuzz)
mkdir -p seeds
# seed1: empty file
: > seeds/seed_empty.bin
# seed2: a few arbitrary bytes
printf '\x00\x00\x00\x00' > seeds/seed_rand4.bin
# seed3: if cwebp exists, build a small valid webp; else fallback to small random bytes
if [ -x "$PWD/install/bin/cwebp" ]; then
  # make a 16x16 blank PPM and convert to webp
  printf "P6\n16 16\n255\n" > seeds/tmp.ppm
  # fill with black pixels
  dd if=/dev/zero bs=1 count=$((16*16*3)) >> seeds/tmp.ppm 2>/dev/null
  "$PWD/install/bin/cwebp" seeds/tmp.ppm -o seeds/seed_valid.webp >/dev/null 2>&1 || true
  rm -f seeds/tmp.ppm
else
  printf '\xff\xff\xff\xff\xff' > seeds/seed_valid.webp
fi
ls -l seeds
```

- Purpose: Provide the fuzzer some starting examples so mutation coverage is better.
- Expected: ls -l seeds should show at least the three files.
- If not: check that cwebp built (look in build/install/bin), or just rely on arbitrary bytes — it still works.

---

## Phase 6 — Run the fuzzer

- From build/ where simple_webp_fuzz lives:

```curl
# run for 5 minutes (300s) as an example; remove -max_total_time to run longer
mkdir -p artifacts
./simple_webp_fuzz seeds -artifact_prefix=./artifacts/ -max_total_time=300 -rss_limit_mb=2048
```


- Purpose: start libFuzzer. It will mutate seed files and run LLVMFuzzerTestOneInput on many inputs. ASan will cause immediate, clear crash messages if issues are found.
- Expected output sample (libFuzzer status lines):

```c
INFO: seed corpus: 3 files
INFO: A new test case was generated with x bytes
#... libFuzzer periodic progress lines like: cov: 10 ft: 20 corp: 3/5 exec/s: 100
```

- If a bug is found, you’ll see ASan output like:

```curl
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x...
READ of size 1 at 0x...
    #0 0x... in some_libwebp_function .../somefile.c:123
    #1 0x... in WebPDecodeRGBA .../decode.c:456
    #2 0x... in LLVMFuzzerTestOneInput .../simple_webp_fuzz.c:12
```

- and a crash file will be written to ./artifacts/ (files named like id:000000,sig:06,...).

- If nothing happens (no crash):
- Expected: sometimes you find nothing quickly. Let it run longer (hours/days) for more coverage.
- Improve seeds: add many real-world .webp images to seeds/ (download publicly-available webp images).
- Try different fuzzers (OSS-Fuzz harnesses) for more coverage.

---

## Phase 7 — If a crash occurs — reproduce & collect evidence
