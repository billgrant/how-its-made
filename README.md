# How It's Made: gojq from source to container image

Building a minimal container image from upstream source the way Chainguard does it, step by step,
with the open-source tools ([melange](https://github.com/chainguard-dev/melange),
[apko](https://github.com/chainguard-dev/apko)) and public [Wolfi](https://wolfi.dev) packages.
It's a learning project, not an official Chainguard image.

```
itchyny/gojq @ b7ebffb ──melange──► gojq-0.12.19-r0.apk ──apko──► gojq:local
   (pinned source)        (sandboxed build,     (local signed     (2 MB, no shell,
                           signed + SBOM)        apk repo)          uid 65532)
```

- **melange** builds the source in a throwaway sandbox and produces a signed `.apk` package plus
  an SBOM.
- **apko** assembles packages into an OCI image without running any commands: no shell, no
  `RUN` steps, no package scripts.

How the result compares with Dockerfile builds of the same software is in
[comparison/RESULTS.md](comparison/RESULTS.md). The gotchas found along the way are below and in
[CLAUDE.md](CLAUDE.md) (written for AI assistants, but readable by anyone).

## Files

| File | What it is |
|---|---|
| `gojq.yaml` | melange recipe: source pin, build environment, build pipeline, tests |
| `gojq-image.yaml` | apko config: packages, keys, nonroot user, entrypoint |
| `gojq-image.lock.json` | apko lock file: every image package pinned by version + checksum (regenerated in step 6) |
| `comparison/` | Dockerfile builds of the same gojq, and the measured results |
| `melange.rsa`, `melange.rsa.pub` | your signing key pair, made in step 1 (gitignored) |
| `packages/` | local apk repo produced by melange (gitignored) |

## Prerequisites

Tested with these versions (2026-10-02, arm64 macOS):

- melange 0.61.2, apko 1.4.6, Docker 29.4.0
- `jq` or `gojq` for inspecting JSON. The optional binary inspection uses `go` and `file`.

melange needs a Linux kernel for builds, so on macOS it uses `--runner docker`. On an x86_64
host, replace `aarch64` with `x86_64` everywhere, including `archs:` in `gojq-image.yaml`.

Run every command from the repo root. `gojq-image.yaml` uses relative paths
(`./packages`, `./melange.rsa.pub`).

## Part 1: build the package (melange)

### 1. Generate a signing key

```bash
melange keygen
```

```bash
chmod 600 melange.rsa
```

This writes `melange.rsa` (private) and `melange.rsa.pub` (public). Both are gitignored. The
private key signs your packages and repo index. The public key is what apko trusts when it
installs them.

### 2. Validate the recipe

```bash
melange compile gojq.yaml --arch aarch64 2>/dev/null | jq .
```

This is the `terraform validate` step. melange parses YAML strictly, so a typo in a key is a
hard error. The output is the fully resolved config: variables filled in and pipelines expanded
into shell.

### 3. Build

```bash
melange build gojq.yaml --arch aarch64 --runner docker --signing-key melange.rsa
```

What to look for in the log:

- **The build environment:** 1 declared package (`go-1.27`) resolves to about 47, all pinned.
  The sandbox is itself an apko-built image.
- **`[git checkout] tag v0.12.19 is b7ebffb…`:** the `expected-commit` integrity check passed.
- **Linters, license check (MIT, confidence 1.0), dependency scan:** the only finding is
  `provides: cmd:gojq`, with no runtime dependencies.
- **Output:** `packages/aarch64/gojq-0.12.19-r0.apk` and a signed `APKINDEX.tar.gz`.

Commit before building. melange records the recipe's git commit in the package metadata.

### 4. Test

```bash
melange test gojq.yaml --arch aarch64 --runner docker --repository-append ./packages --keyring-append melange.rsa.pub
```

The test runs in a **fresh** container that holds only the built package plus `test.environment`
packages (busybox, for the shell). It proves the package works without the build environment.
Expect `gojq 0.12.19 (rev: b7ebffb/go1.27.1)`.

### 5. Inspect the package (optional, recommended)

```bash
mkdir -p /tmp/gojq-apk && tar -xzf packages/aarch64/gojq-0.12.19-r0.apk -C /tmp/gojq-apk
```

```bash
cat /tmp/gojq-apk/.PKGINFO
```

```bash
jq '.packages[] | {name, versionInfo}' /tmp/gojq-apk/var/lib/db/sbom/gojq-0.12.19-r0.spdx.json
```

```bash
go version -m /tmp/gojq-apk/usr/bin/gojq
```

```bash
file /tmp/gojq-apk/usr/bin/gojq
```

| File | What to notice |
|---|---|
| `.SIGN.RSA256.melange.rsa.pub` | The signature. The filename names the **public** key that verifies it. |
| `.PKGINFO` | Metadata. `datahash` is the SHA-256 of the data section, so the signature covers every file. No `depend` lines. |
| `.melange.yaml` | The compiled, version-locked recipe: the full build environment and pipeline scripts. |
| SBOM | Package, recipe (`DESCRIBED_BY`) and upstream source (`GENERATED_FROM`). The Go modules are in the binary, not the SBOM. |
| Binary | Statically linked, `CGO_ENABLED=0`, **not stripped** (symbol table kept for govulncheck), with `chainguard_go_package` stamped in by Wolfi's patched Go. |

## Part 2: build the image (apko)

### 6. Preview and lock package versions

```bash
apko show-packages gojq-image.yaml --arch aarch64
```

```bash
apko lock gojq-image.yaml
```

`show-packages` is the `terraform plan` step. `apko lock` (which isn't listed in `apko --help`)
writes `gojq-image.lock.json`. Expect exactly 3 packages: `gojq`, `wolfi-baselayout`,
`ca-certificates-bundle`. If you see glibc or OpenSSL, a package name is wrong. For example,
`ca-certificates` is the tooling package; `ca-certificates-bundle` is just the cert file.

Regenerate the lock whenever you rebuild `gojq` with different inputs, or when you want newer
Wolfi packages.

### 7. Build, load, run

```bash
apko build --lockfile gojq-image.lock.json gojq-image.yaml gojq:local gojq.tar
```

```bash
docker load < gojq.tar
```

```bash
echo '{"name":"bill","role":"SE"}' | docker run --rm -i gojq:local-arm64 .role
```

That should print `"SE"`. `docker load` prints the exact tag; apko adds an arch suffix.

### 8. Verify the image

```bash
docker image inspect gojq:local-arm64 --format '{{.Config.User}}'
```

```bash
docker run --rm gojq:local-arm64 -Rr . /etc/passwd
```

```bash
docker run --rm --entrypoint /bin/sh gojq:local-arm64
```

The first prints `65532` (nonroot). The second prints `nonroot:x:65532:65532:Account created by
apko:…`. The third **fails**: there's no shell, and no `id`, `ls` or `cat` either. The only
executable is gojq.

### 9. Check reproducibility

```bash
apko build --lockfile gojq-image.lock.json gojq-image.yaml gojq:local /tmp/build1.tar && apko build --lockfile gojq-image.lock.json gojq-image.yaml gojq:local /tmp/build2.tar
```

```bash
shasum -a 256 /tmp/build1.tar /tmp/build2.tar
```

The two hashes must be identical. That takes three things: the same config (no commands that
could vary), the same package versions (the lock file), and fixed timestamps (all dates are
`1970-01-01`, which is deliberate; set `--build-date` or `SOURCE_DATE_EPOCH` for a real but
fixed date).

## Gotchas we hit

- **The test environment doesn't inherit the build environment's repositories/keyring.** Declare
  them in `test.environment.contents` too.
- **`@local` is for repositories only.** Write `"@local ./packages"`, quoted, because `@` can't
  start an unquoted YAML value. Don't add the arch subdirectory. Packages then use `gojq@local`.
  Keyring entries are plain paths.
- **Use `entrypoint.command`, not `cmd`.** With no entrypoint, `cmd` runs through `/bin/sh -c`,
  which doesn't exist in this image.
- **No `accounts:` block means the image runs as root.**
- **Go `-X` on a variable that doesn't exist, or on a `const`, is silently ignored.** Check the
  project's source before copying ldflags from another recipe.
- **Never use `apko build --ignore-signatures`.**

## Comparison with Dockerfile builds

[comparison/RESULTS.md](comparison/RESULTS.md) has the same gojq built with single-stage,
multi-stage and `FROM scratch` Dockerfiles, measured for size, packages, CVEs and what each image
can prove about itself.

## Not covered yet

cosign signing and attestations, and the CVE lifecycle (version bumps, `go/bump`, advisories).

## License

The contents of this repo are [MIT](LICENSE). gojq itself is MIT-licensed by its author and is
fetched from upstream at build time, not stored here.
