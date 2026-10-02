# How It's Made

Guidance for AI assistants working in this repo. Humans: start with [README.md](README.md).

A hands-on walkthrough of building a container image from source the Chainguard way:
melange (source → signed .apk) → local apk repo → apko (packages → OCI image), plus a
comparison against Dockerfile builds in [comparison/](comparison/).

## Pinned facts (verified 2026-10-02)
- Reference host: arm64 macOS → melange arch `aarch64`, `--runner docker`
- melange 0.61.2, apko 1.4.6, syft 1.54.0, grype 0.120.0
- Target: gojq v0.12.19, tag commit b7ebffbfc038677520df0bae4c8c2d877f88ffea
- Wolfi repo: https://packages.wolfi.dev/os
- Wolfi key:  https://packages.wolfi.dev/os/wolfi-signing.rsa.pub
- Reference recipe style: wolfi-dev/os `yq.yaml` (uses `go/build/v2`, `go-${{vars.go-version}}`,
  go-version "1.27"). Wolfi doesn't ship a gojq recipe, so this one is written from scratch.

## Guardrails
- Never commit `melange.rsa` (private key). Each user generates their own with `melange keygen`.
- Public sources only. Nothing vendor-internal or customer-specific goes in this repo.
- Never use `apko build --ignore-signatures`.
- Check claims about tool behavior against the docs for the pinned versions; these tools change fast.

## Lessons learned
- wolfi-dev/os recipes don't list `repositories:`/`keyring:` in `environment:`. The repo's
  Makefile passes those on the command line. A standalone recipe has to declare them itself.
- melange decodes YAML strictly: a misspelled or wrong-case key is a hard error, not silently
  ignored. `melange compile <recipe> --arch aarch64` is the `terraform validate` equivalent:
  it parses the recipe, fills in `${{vars.*}}`, and prints the resolved config. Run it before
  every build.
- `test/tw/*` pipelines (ver-check, ldd-check, help-check) are defined in wolfi-dev/os
  `pipelines/test/tw/`, not built into melange. Standalone recipes can't `uses:` them unless you
  vendor that directory and pass `--pipeline-dir`. Use plain `runs:` tests instead.
- `melange build` doesn't run the `test:` block. `melange test` is a separate command, and it runs
  in a fresh container holding only the built package plus `test.environment` packages.
- The test environment does NOT inherit the build `environment:` repositories/keyring. A
  standalone recipe needs them in `test.environment.contents` too.
- Go `-X` on a variable that doesn't exist (or on a `const`) is silently ignored by the linker.
  Check the target variable in the project's source before copying ldflags from another recipe.
- gojq builds as `CGO_ENABLED=0`, statically linked, "not stripped" (symbol table kept), with no
  `depend` lines. The recipe never sets CGO; Go turns it off because the sandbox has no C compiler.
  Verified: the same build in `golang:1.27` (which has gcc) gives `CGO_ENABLED=1`, though it's
  still static because gojq uses no cgo packages.
- Wolfi's Go stamps `chainguard_go_package=go-1.27-<ver>-rN` into each binary's build info
  (`go version -m`). This identifies which binaries need rebuilding when the toolchain has a CVE.
- Upstream Go leaves `-ldflags` out of the build info whenever `-trimpath` is set
  (cmd/go/internal/load/pkg.go). Wolfi's Go has a patch,
  `0001-go-cmd-go-always-emit-ldflags-version-information.patch`, that always records it. Patch
  0003 adds `chainguard_go_package`.
- Upstream gojq v0.12.19 release binary: go1.26.1, `stripped`, no -ldflags in build info. This
  repo's build: go1.27.1, `not stripped`, ldflags recorded. Same source commit (b7ebffb).
- melange's per-package SBOM lists only 4 entries: the OS placeholder, the apk, the recipe
  (DESCRIBED_BY, at the recipe commit), and the upstream source (GENERATED_FROM, at
  expected-commit). It does NOT list the Go modules. Those live in the binary's build info,
  where scanners (syft/grype) read them.
- The "unknown" fields in the SBOM come from flags not passed: `--namespace` (purl namespace,
  e.g. wolfi) and `--git-repo-url`. Timestamps are 1970-01-01 because SOURCE_DATE_EPOCH=0, which
  keeps rebuilds byte-identical.
- `apko lock <config>` exists in apko 1.4.6 but isn't listed in `apko --help`. It writes
  `<config>.lock.json`, pinning every package (dependencies included) to a version, URL and
  checksum. Use it with `apko build --lockfile`.
- Wolfi `ca-certificates` is the tooling package (needs glibc + OpenSSL). `ca-certificates-bundle`
  is just the cert file, which `wolfi-baselayout` already depends on. Adding the wrong one takes the
  image from 3 packages to 13.
- apko doesn't write `/etc/os-release` itself (it warns when it's missing). `wolfi-baselayout`
  provides it, plus passwd/group/hosts/nsswitch and /tmp permissions. Scanners use os-release to
  identify the distro.
- apko records its own input in the image (`/etc/apko.json`, `/etc/apk/world`) and the git remote
  of the checkout in the `org.opencontainers.image.source` annotation. The same packages built
  through a different invocation (with vs without `--lockfile`) give a different digest.
  Reproducible means the same inputs, including the command line.
- Docker tags can be moved: writing a new `.tar` doesn't update Docker. Run `docker load` again
  and compare layer digests before scanning.
