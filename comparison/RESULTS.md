# Comparison: the same gojq, four ways

The same software ([gojq](https://github.com/itchyny/gojq) v0.12.19, built with Go 1.27.1)
packaged four ways, then measured for size, contents and vulnerabilities. Everything here uses
public images, public tools and public package repositories, and can be rerun with the commands at
the bottom.

> **Scope:** image C is built *Chainguard-style* in this repo, with melange and apko from public
> [Wolfi](https://wolfi.dev) packages. It is not an official Chainguard image. The point is to show
> how the build approach affects the result, not to benchmark a product.

**Measured:** 2026-10-02, arm64 macOS, Docker 29.4.0 (containerd image store), syft 1.54.0,
grype 0.120.0 (vulnerability DB v6.1.9, built 2026-10-02T06:31:53Z). Vulnerability counts change
as the database updates, so rerun for current numbers.

## The four images

| | How it's built | File |
|---|---|---|
| **A: single-stage** | `FROM golang:1.27`, `RUN go install …@v0.12.19` | [Dockerfile.single](Dockerfile.single) |
| **B: multi-stage** | build in `golang:1.27`, `COPY --from` into `debian:trixie-slim` | [Dockerfile.multistage](Dockerfile.multistage) |
| **C: Chainguard-style** | melange (source → signed apk) + apko (`gojq` + `wolfi-baselayout`) | [../gojq.yaml](../gojq.yaml), [../gojq-image.yaml](../gojq-image.yaml) |
| **D: scratch** | build in `golang:1.27`, `COPY --from` into `FROM scratch` | [Dockerfile.scratch](Dockerfile.scratch) |

A, B and D are deliberately written the way a typical team writes them: unpinned tags, the
default user, a plain `go install`.

## Results at a glance

| | A: single-stage | B: multi-stage | **C: Chainguard-style** | D: scratch |
|---|---|---|---|---|
| Size, compressed | 336 MB | 33.8 MB | **2.0 MB** | 3.6 MB |
| Size, on disk | 1.42 GB | 149 MB | **7.1 MB** | 10 MB |
| Packages found (syft) | 242 | 87 | **12** | 9 |
| **CVEs (grype)** | **1,060** | **182** | **0** | **0** |
| Distro detected | Debian 13.7 | Debian 13.7 | **Wolfi** | *(none)* |
| Shell | yes | yes | **no** | no |
| Runs as | root | root | **65532 (nonroot)** | root |

"On disk" is what `docker images` reports with the containerd store: the compressed download
plus the unpacked files. "Compressed" (`docker image inspect .Size`) is what a registry stores
and a node pulls.

## Vulnerabilities in detail

| Image | Critical | High | Medium | Low | Negligible | **Total** |
|---|---|---|---|---|---|---|
| A | 29 | 154 | 141 | 46 | 690 | **1,060** |
| B | 0 | 60 | 59 | 18 | 45 | **182** |
| **C** | 0 | 0 | 0 | 0 | 0 | **0** |
| D | 0 | 0 | 0 | 0 | 0 | **0** |

**Where they come from:** 100% of A's and B's findings are in Debian (`deb`) packages. **None are
in the gojq binary, its Go modules, or the Go standard library, in any of the four images.** B's
High findings are mostly util-linux libraries (`libmount1`, `libblkid1`, `libuuid1`, `login`, …)
and `libssl3t64` (OpenSSL). gojq uses none of them.

**Can they be fixed?** This is grype's fix state for each finding:

| Image | Fix available | No fix yet | Won't fix (distro decision) |
|---|---|---|---|
| A | 40 | 709 | 311 |
| B | **27** | **50** | **105** |

Of B's 182 findings, updating packages fixes **at most 27**. The other 155 can only be handled by
filing exceptions or removing the packages.

## What's inside C and D

Both contain the same 9 entries that syft reads from the gojq binary's embedded Go build info:
gojq itself, its 7 dependencies, and `stdlib go1.27.1`. The difference is what surrounds the
binary:

| | D: scratch | C: Chainguard-style |
|---|---|---|
| Go modules + stdlib (read from the binary) | ✅ | ✅ |
| Binary recorded as a package (`gojq-0.12.19-r0`) | ❌ just a file at `/gojq` | ✅ |
| License | ❌ | ✅ declared (MIT) and checked against the source |
| Link to build recipe and upstream source commit | ❌ | ✅ SBOM `DESCRIBED_BY` / `GENERATED_FROM` |
| Signature you can verify | ❌ | ✅ |
| Record of the build environment | ❌ | ✅ (compiled recipe embedded in the package) |
| `/etc/os-release` (lets scanners identify the distro) | ❌ | ✅ |
| CA certificates, `/etc/passwd`, `/tmp` | ❌ | ✅ |
| Nonroot by default | ❌ | ✅ |

**Why D is bigger than C, despite containing only the binary:** a plain `go install` keeps DWARF
debug information (`file`: "with debug_info"; 6.4 MB binary). C's build removes debug info
(`-w`, then `strip -g`; 4.7 MB binary) but keeps the symbol table, so vulnerability tools can
still check which functions are compiled in.

## What this shows

1. **The binary is the same; what's around it isn't.** All four contain a gojq built from the
   same source with Go 1.27.1. Every finding comes from software gojq never executes.
2. **Single-stage ships the build machine.** A includes gcc, git and the whole Go toolchain to
   run a single binary. melange avoids this by design: build-time and runtime dependencies are
   separate lists, handled by separate tools.
3. **Multi-stage helps a lot, but the base OS remains.** B removes 83% of A's findings, but a
   general-purpose base still brings a shell, a package manager, and libraries the app never loads.
4. **Most distro findings can't be fixed by patching.** The work moves from "patch it" to
   "triage, justify, and document it for an auditor", over and over.
5. **Scratch is a strong answer for one static binary, on day one.** D matches C on CVEs and comes
   close on size. What it doesn't have is everything that makes an image identifiable,
   verifiable and runnable as non-root, and it only works for software that needs no libc,
   interpreter or runtime. Most C, Python, Java and Node applications can't run on scratch.
6. **Zero is a snapshot, not a permanent state.** C and D are at zero today because Go 1.27.1 and
   gojq's dependencies are current. The next Go security release will add a finding to the
   `stdlib go1.27.1` entry in **all four** images. The difference is who rebuilds and how fast. In
   the Wolfi model, the toolchain package is updated, gojq's package epoch is bumped (`-r0` →
   `-r1`) and rebuilt, and the image is reassembled. For A, B and D, it's the image owner's job.

## Things to be fair about

- Many Low/Negligible findings are marked won't-fix because they're hard to exploit in a
  container. They still have to be triaged and documented, and zero findings means zero triage.
- A, B and D could be improved: pinned digests, a `USER` line, `-ldflags="-s -w"` or `-w`,
  `CGO_ENABLED=0`, distroless bases. Every improvement is something each team has to know about,
  apply, and maintain, for every image.
- This is one small, pure-Go program, which is the best case for scratch. Results for
  applications with native dependencies or language runtimes will look different.

## Side findings

- **CGO:** the gojq built in `golang:1.27` reports `CGO_ENABLED=1`, because that image includes
  gcc. C's build environment has no C compiler, so Go defaulted to `CGO_ENABLED=0`. Both binaries
  are static anyway, because gojq uses no packages that need cgo. That's why D runs on an empty
  filesystem.
- **Build info:** the binaries built with upstream Go in A, B and D record no `-ldflags`. Upstream
  Go leaves them out whenever `-trimpath` is used. C's binary records them, plus the exact
  toolchain package (`chainguard_go_package=go-1.27-1.27.1-r0`), because Wolfi's Go is patched to
  emit both.

## Reproduce

From the repo root, with C built per the [README](../README.md) and loaded as `gojq:local-arm64`:

```bash
docker build -f comparison/Dockerfile.single -t gojq:single comparison/
```

```bash
docker build -f comparison/Dockerfile.multistage -t gojq:multistage comparison/
```

```bash
docker build -f comparison/Dockerfile.scratch -t gojq:scratch comparison/
```

```bash
docker images gojq --format 'table {{.Tag}}\t{{.Size}}'
```

```bash
for t in single multistage local-arm64 scratch; do printf '%-12s ' $t; syft gojq:$t -q -o json | jq '.artifacts | length'; done
```

```bash
for t in single multistage local-arm64 scratch; do printf '%-12s ' $t; grype gojq:$t -q -o json | jq -c '{distro: .distro.name, sev: ([.matches[].vulnerability.severity] | group_by(.) | map({(.[0]): length}) | add // {})}'; done
```

```bash
for t in single multistage; do printf '%-12s ' $t; grype gojq:$t -q -o json | jq -c '[.matches[].vulnerability.fix.state] | group_by(.) | map({(.[0]): length}) | add'; done
```

Make sure `gojq:local-arm64` is the image you think it is. Writing a new `.tar` doesn't update
Docker; run `docker load` again, and compare digests with `docker image inspect`.

Image IDs measured: A `sha256:6f40e360…`, B `sha256:db34d883…`, C `sha256:35c89afa…`,
D `sha256:c17ea85d…`.
