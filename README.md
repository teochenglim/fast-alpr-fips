# fast-alpr on FIPS-Hardened RHEL 9.7

Running [fast-alpr](https://github.com/ankandrew/fast-alpr) in containers on a RHEL 9.7 FIPS-hardened Kubernetes cluster, with a distroless final image.

---

## The Problem

```
FATAL FIPS SELFTEST FAILURE
```

This crash happens on `import cv2` when:

- Your K8s node runs **RHEL 9.7 with FIPS mode enabled** (`/proc/sys/crypto/fips_enabled = 1`)
- You install `opencv-python` or `opencv-python-headless` from PyPI

### Root Cause

The standard PyPI `opencv-python-headless` wheel is built using the [opencv-python manylinux pipeline](https://github.com/opencv/opencv-python). That pipeline historically compiled a **custom OpenSSL 1.1.1w from source** (`/ffmpeg_build/lib/libssl.so`), then `auditwheel repair` bundled it into the wheel alongside `cv2.so`.

At import time, `cv2.so` loads this bundled OpenSSL 1.1.1w. On a FIPS-enforced kernel, OpenSSL performs a self-test of its FIPS module on startup. OpenSSL 1.1.1w is **not FIPS-certified** and fails this check immediately — crashing Python before your code runs.

```
cv2.so
 └── (bundled) libssl.so.1.1  ← OpenSSL 1.1.1w, not FIPS-certified
                               ← kernel sees fips_enabled=1 → SELFTEST FAILURE
```

---

## The Fix

[opencv-python PR #1190](https://github.com/opencv/opencv-python/pull/1190) — **"manylinux: avoid bundling OpenSSL to fix FIPS import crash"**

The PR removes the custom OpenSSL source build from the manylinux Dockerfile and installs system OpenSSL (`openssl`, `openssl-devel`) instead. Because system OpenSSL is in the manylinux `auditwheel` whitelist, it is **not bundled** into the wheel.

At runtime, `cv2.so` uses whatever `libssl.so.3` is on the host:

- On **RHEL 9.7 FIPS**: the system OpenSSL is FIPS-certified → self-test passes ✓  
- In a **Debian/Ubuntu container** on that node: the container's OpenSSL 3.x is used

---

## Deliverables

### 1. Downloadable Wheel (GitHub Actions artifact)

Built on `ubuntu-22.04` with system OpenSSL, for three Python versions. Useful for direct installation without Docker.

| Artifact | Python | Platform |
|---|---|---|
| `opencv-headless-fips-safe-py3.11-linux-x86_64` | 3.11 | linux/amd64 |
| `opencv-headless-fips-safe-py3.12-linux-x86_64` | 3.12 | linux/amd64 |
| `opencv-headless-fips-safe-py3.13-linux-x86_64` | 3.13 | linux/amd64 |

Download from the [Actions tab](../../actions), then:

```bash
pip install opencv_python_headless-*.whl
```

### 2. Container Images (GHCR)

Two variants pushed to `ghcr.io/<owner>/<repo>`:

| Tag | Builder | Python | Final base |
|---|---|---|---|
| `latest` / `main` | ubuntu:22.04 | 3.11 | distroless/base-debian12 |
| `latest-ubuntu2404` / `main-ubuntu2404` | ubuntu:24.04 | 3.12 | distroless/base-debian13 |

```bash
docker pull ghcr.io/<owner>/<repo>:latest
```

---

## Container Design

Each image is a **3-stage build** to keep the final image minimal and secure.

```
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: builder (ubuntu:22.04 / ubuntu:24.04)                  │
│                                                                 │
│  • Installs Python + build tools                                │
│  • git clone opencv-python → git fetch PR #1190                 │
│  • pip wheel . --no-build-isolation   (ENABLE_HEADLESS=1)       │
│  • python -m venv --copies /opt/venv                            │
│    └── pip install <fips-safe wheel> fast-alpr                  │
└──────────────────────────┬──────────────────────────────────────┘
                           │ /opt/venv  /usr/lib/python3.x
                           │ libpython3.x.so
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 2: fips-config (debian:bookworm-slim / debian:trixie-slim)│
│                                                                 │
│  Must match the final stage OS so fips.so is ABI-compatible     │
│  with libcrypto.so.3 at runtime.                                │
│                                                                 │
│  • openssl fipsinstall -module fips.so -out fipsmodule.cnf      │
│    fipsmodule.cnf = HMAC of fips.so (integrity check at load)   │
│  • Writes openssl-fips.cnf activating FIPS + base providers     │
└──────────────────────────┬──────────────────────────────────────┘
                           │ fips.so + fipsmodule.cnf (matched pair)
                           │ openssl-fips.cnf
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│ Stage 3: final (distroless/base-debian12 / distroless/base-     │
│                 debian13)                                        │
│                                                                 │
│  No shell. No package manager. Minimal attack surface.          │
│                                                                 │
│  Copied in:                                                     │
│  • /opt/venv          ← self-contained (--copies, no symlinks)  │
│  • /usr/lib/python3.x ← stdlib                                  │
│  • libpython3.x.so    ← not in distroless/base                  │
│  • fips.so            ← matched to fipsmodule.cnf               │
│  • fipsmodule.cnf     ┘ (must come from same fips-config stage) │
│  • openssl-fips.cnf   ← sets OPENSSL_CONF at runtime           │
│                                                                 │
│  ENV OPENSSL_CONF=/etc/ssl/openssl-fips.cnf                     │
│  ENV OPENSSL_MODULES=/usr/lib/x86_64-linux-gnu/ossl-modules     │
└─────────────────────────────────────────────────────────────────┘
```

### Why `--copies` in venv

`python -m venv /opt/venv` creates symlinks by default (e.g. `python3 → /usr/bin/python3.11`). In distroless there is no `/usr/bin/python3.11` to resolve against. `--copies` writes real binaries so `COPY --from=builder /opt/venv` is fully self-contained.

### Why the FIPS stage must match distroless

`fipsmodule.cnf` stores an HMAC of the exact `fips.so` binary used to generate it. At container startup, OpenSSL recomputes this HMAC and refuses to load the provider if it doesn't match. Copying `fips.so` and `fipsmodule.cnf` from the same stage guarantees the pair stays consistent.

Using `debian:bookworm-slim` (not `ubuntu:22.04`) for the FIPS stage ensures `fips.so` is dynamically linked against the same `libcrypto.so.3` that is present in `distroless/base-debian12`.

---

## Repository Layout

```
fastalpr/
├── Dockerfile.2204          # ubuntu:22.04 → Python 3.11 → distroless/base-debian12
├── Dockerfile.2404          # ubuntu:24.04 → Python 3.12 → distroless/base-debian13
└── .github/workflows/
    └── ci.yml               # 5 parallel jobs: 3 wheel builds + 2 image builds
```

---

## GitHub Actions

All 5 jobs run in parallel:

```
build-wheel (py3.11) ─┐
build-wheel (py3.12) ─┼─ parallel
build-wheel (py3.13) ─┤
build-image (2204)   ─┤
build-image (2404)   ─┘
```

The first image build takes ~40 minutes (opencv compilation). Subsequent runs use BuildKit layer cache stored in GitHub Actions cache, reducing build time significantly. Each image variant has its own cache scope to avoid collisions.

---

## Usage in a Pod

```yaml
spec:
  containers:
    - image: ghcr.io/<owner>/<repo>:latest
      command: ["/opt/venv/bin/python3", "/app/main.py"]
```

Or extend the image with your application code:

```dockerfile
FROM ghcr.io/<owner>/<repo>:latest
COPY main.py /app/main.py
CMD ["/app/main.py"]
```

---

## References

- [fast-alpr](https://github.com/ankandrew/fast-alpr)
- [opencv-python PR #1190](https://github.com/opencv/opencv-python/pull/1190) — the core fix
- [opencv-python PR #1191](https://github.com/opencv/opencv-python/pull/1191) — related
- [GoogleContainerTools/distroless](https://github.com/GoogleContainerTools/distroless)
- [OpenSSL FIPS module documentation](https://www.openssl.org/docs/man3.0/man7/fips_module.html)
