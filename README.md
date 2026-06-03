# YourCompanyName-DevOps-BaseImages

This repository is mostly documentation. It explains how to choose Microsoft's official .NET 10 runtime bases (Ubuntu Noble, Ubuntu Chiseled, Alpine) and how this repo's optional yourcompanyname wrapper images fit on top.

Treat the contents as guidelines, not a mandate. The goal is to give teams a clear starting point for picking a base image and to publish a small set of hardened wrapper images that downstream services can build on. Align everything with your own team's standards, registry policy, and risk posture.

---

## Who should read what

**Application developers.** Your main decision is between full Ubuntu (`noble`), Chiseled (`noble-chiseled` / `noble-chiseled-extra`), and Alpine. The short answer for new .NET 10 ASP.NET services is `noble-chiseled` (or `noble-chiseled-extra` if you need ICU). Use full `noble` when you need `apt`, an in-image shell, or heavier native stacks like SkiaSharp or PDF tooling. If you pick Alpine, you own the musl caveats — see [musl vs glibc in practice](#musl-vs-glibc-in-practice).

**Platform and DevOps.** Catalog of images is in [The images at a glance](#the-images-at-a-glance) and [Image size comparison](#image-size-comparison). Build and CI patterns are below; published images live in [Published images (GHCR)](#published-images-ghcr).

**Security and compliance.** Chiseled reduces surface area (no shell, no package manager in the runtime) at the cost of more involved break-glass debugging. Alpine vs glibc affects supply chain and compatibility, not just size. The yourcompanyname **`ubuntu-net-10`** and **`alpine-net-10`** images can include optional diagnostic CLIs at build time — use `--target final` for lean production; see [Ubuntu and Alpine: optional diagnostic tools](#ubuntu-and-alpine-optional-diagnostic-tools). Patching follows whatever upstream Microsoft and distro lifecycles you pin to. This README does not replace your org's image allow-list or tagging policy.

---

## The images at a glance

```bash
# Microsoft official .NET 10 — Ubuntu (Noble = 24.04 LTS)
mcr.microsoft.com/dotnet/aspnet:10.0-noble
mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled        # minimal Ubuntu, distroless-style
mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled-extra  # chiseled + ICU/tzdata

# Microsoft official .NET 10 — Alpine
mcr.microsoft.com/dotnet/aspnet:10.0-alpine
mcr.microsoft.com/dotnet/aspnet:10.0-alpine3.21
```

## Image size comparison

Approximate sizes for the ASP.NET runtime image on .NET 10. Sizes shift with each servicing release; for current numbers see Microsoft's [sample image size report](https://github.com/dotnet/dotnet-docker/blob/main/documentation/sample-image-size-report.md).


| Image                      | Compressed | Uncompressed |
| -------------------------- | ---------- | ------------ |
| `10.0-noble` (full Ubuntu) | ~90 MB     | ~220 MB      |
| `10.0-noble-chiseled`      | ~40 MB     | ~105 MB      |
| `10.0-alpine`              | ~45 MB     | ~115 MB      |
| `10.0-alpine-composite`    | ~38 MB     | ~98 MB       |


Chiseled Ubuntu and Alpine sit within a few megabytes of each other. Size alone is not a strong reason to pick one over the other — the deciding factors are libc, shell access, and package manager availability.

---

## Ubuntu Noble (full)

`mcr.microsoft.com/dotnet/*:10.0-noble` is a conventional Linux container: a shell, `apt`, and the usual Ubuntu base packages. That makes it the most flexible option. You can add native libraries at build time, `docker exec` for break-glass debugging, and most teams find it behaves the way they expect from a "normal" Linux box.

The trade-off is size and attack surface. The full ASP.NET runtime image is noticeably larger than the Chiseled variant for the same .NET version, and it ships more installed packages.

**Strengths**

- glibc — strongest compatibility with NuGet packages that ship native `.so` dependencies.
- `apt` and shell available in the runtime image when policy allows.
- ICU-backed globalization out of the box.
- Behaves like a conventional Ubuntu container, which matches most developer machines.
- Good fit for SkiaSharp, ImageSharp, OpenSSL-heavy workloads, and any "install another deb" scenario.
- Ubuntu 24.04 Noble carries standard Canonical LTS support through April 2029.

**Limitations**

- Larger pulls and larger disk footprint than Chiseled. See [Image size comparison](#image-size-comparison).
- More installed packages means a broader patch surface than the minimal variants.
- Not distroless — there's more in the image than the runtime strictly needs.

When in doubt between Noble and Chiseled: pick Noble when you need `apt`, a shell, or the widest runtime installability. Pick Chiseled when you can accept giving those up in exchange for a smaller, tighter runtime.

---

## Ubuntu Chiseled

Ubuntu Chiseled (`*-noble-chiseled`, `*-noble-chiseled-extra`) is a distroless-style variant built by Canonical in partnership with Microsoft using the Chisel tool. The image contains only the slice of Ubuntu .NET actually needs: no shell, no package manager, non-root by default, and far fewer moving parts than the full Noble image. See the [official overview](https://github.com/dotnet/dotnet-docker/blob/main/documentation/ubuntu-chiseled.md).

There is no Chiseled SDK image. You publish with `mcr.microsoft.com/dotnet/sdk:*-noble` and run on a Chiseled runtime tag.

**Strengths**

- Smaller compressed and uncompressed size than full Noble for framework-dependent deployments.
- Minimal package set — only what Chisel slices in.
- No `apt` and no shell in the runtime image, which removes useful tooling from an attacker as well.
- `noble-chiseled-extra` adds ICU and timezone data without pulling in the full Noble image.
- Non-root by default.
- Same glibc and Ubuntu lineage as Noble, so native libraries behave closer to a developer's full-Ubuntu machine than Alpine does.

**Limitations**

- No shell. `docker exec` into a shell will not work the way it does on full Noble. Plan for sidecars, ephemeral debug pods, or CI-driven diagnostics.
- No package manager. You cannot `apt install` at runtime; changes mean rebuilding the image (or working with Chisel slice definitions directly).
- You have to pick the right variant. Use `chiseled-extra` when you need globalization parity closer to the full image; otherwise the defaults differ from stock `aspnet:noble`.
- Build stage still uses the full Ubuntu SDK image. Only the runtime stage is chiseled.

---

## Alpine Linux

Alpine has been the go-to for very small container images for a long time. It uses `musl` instead of `glibc`, ships a small shell (`sh`) and package manager (`apk`), and produces a runtime image close in size to Chiseled.

The catch is musl. Many NuGet packages with native components either don't support musl, support it partially, or fail in ways that are hard to attribute. Globalization and timezone behavior also need explicit configuration in some scenarios.

**Strengths**

- Small image size — fast pulls, less registry storage, quicker pod starts.
- Minimal default attack surface.
- `apk` still available for runtime additions when needed.
- Shell present, so `docker exec` and basic in-container troubleshooting work.
- Solid fit for simple REST or gRPC services with no native dependencies.
- Cost savings at scale on bandwidth and CI throughput.

**Limitations**

- musl libc is the root of most Alpine .NET pain.
- NuGet packages with native components often don't support musl, or do so only with extra work.
- SkiaSharp, `libgdiplus`, and similar drawing/PDF stacks need extra packages or fail.
- ICU and globalization may need explicit configuration.
- Self-contained publishes need `--runtime linux-musl-x64`, which is easy to forget.
- Some `PInvoke` and interop scenarios behave differently under musl.
- Less alignment with the typical developer machine, so "works locally, fails in Alpine" is a known pattern.
- Microsoft supports Alpine, but recommends Chiseled for most hardened production .NET deployments.

---

## Side-by-side comparison


| Factor                       | Ubuntu Noble | Ubuntu Chiseled  | Alpine              |
| ---------------------------- | ------------ | ---------------- | ------------------- |
| Base C library               | glibc        | glibc            | musl                |
| ASP.NET runtime image size   | ~220 MB      | ~105 MB          | ~115 MB             |
| Native library compatibility | Excellent    | Excellent        | Mixed (musl)        |
| NuGet native dependencies    | Full support | Full support     | Hit or miss         |
| Globalization / ICU          | Out of box   | Via `-extra` tag | Extra config needed |
| Shell access for debugging   | Yes          | No               | Yes                 |
| Package manager              | `apt`        | None             | `apk`               |
| Typical scanner CVE noise    | Higher       | Very low         | Very low            |
| Non-root by default          | No           | Yes              | No                  |
| Microsoft recommendation     | Yes          | Yes (preferred)  | Yes, with caveats   |
| SkiaSharp / System.Drawing   | Works        | Works            | Needs extra libs    |
| Self-contained publish       | `linux-x64`  | `linux-x64`      | `linux-musl-x64`    |
| Kubernetes pod startup       | Medium       | Fast             | Fast                |
| Distro LTS window            | 2029 (Noble) | 2029 (Noble)     | Rolling, no LTS     |
| Best fit                     | General use  | Hardened prod    | Small, simple APIs  |


CVE counts on fresh builds are roughly comparable between Chiseled and Alpine because both ship a minimal package set. Full Noble carries more components, so scanners typically flag more findings — most of them in OS utilities the app does not actually use.

---

## musl vs glibc in practice

The Alpine failures developers hit most often look like this:

```text
# Native lib not found
System.DllNotFoundException: Unable to load shared library 'libgdiplus'

# Globalization mode wrong for the workload
System.Globalization.CultureNotFoundException:
Only the invariant culture is supported in globalization-invariant mode.
```

Common fixes:

```dockerfile
# Fix 1 — install missing libs in the Alpine Dockerfile
RUN apk add --no-cache \
    icu-libs \
    libgdiplus \
    krb5-libs \
    libintl \
    libssl3

# Fix 2 — opt into invariant globalization (you lose culture support)
ENV DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1

# Fix 3 — switch to chiseled-extra, which includes ICU and tzdata
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled-extra
```

Fix 3 is the path of least resistance in most enterprise environments.

---

## Choosing a base image

```text
General-purpose ASP.NET Core API
  -> noble-chiseled — best balance of size, security, and compatibility

Needs SkiaSharp, ImageSharp, System.Drawing, or PDF libraries
  -> noble (full) or noble-chiseled-extra — Alpine will cause pain

Needs globalization or timezone data but wants a small image
  -> noble-chiseled-extra — ICU + tzdata included

Simple gRPC or REST service with no native dependencies
  -> Alpine is fine — small, fast, well-understood (use yourcompanyname alpine-net-10; build without diagnostic tools for prod)

Need in-image dotnet-debug / gcdump / trace on full Noble
  -> ubuntu-net-10 with target final-diagnostics (default CI); use final for lean prod

Need to `docker exec` in for routine debugging
  -> noble (full) or Alpine — chiseled has no shell

Production, security-hardened, non-root by default
  -> noble-chiseled — Microsoft's recommended hardened production image

Self-contained single-binary publish
  -> Alpine with linux-musl-x64, or chiseled with linux-x64
```

Rough rule of thumb across teams:

- Internal tools and prototypes where convenience and quick troubleshooting matter most: **full Noble**.
- Microservices with no native dependencies and a real interest in tiny images: **Alpine**.
- Anything deployed at scale into an environment with strict compliance or zero-CVE expectations: **Chiseled** (with `-extra` if you need ICU).

---

## yourcompanyname wrapper images

This repository builds **three** .NET 10 wrapper images on top of the Microsoft bases. They share the same operational contract: ports 8080/8443 exposed, ICU-backed globalization, server GC with a managed heap cap, a non-root `yourcompanyname` user (UID/GID `7777`) with home `/app`, and `/app/.info` plus `/app/app-data` directories ready for the runtime user.


| Image                                          | Role                | Source Dockerfile                                                                        | Based on                                              |
| ---------------------------------------------- | ------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `ghcr.io/jbaltikauskas/ubuntu-net-10`          | Production          | `[dockerfiles/ubuntu/10/dockerfile](dockerfiles/ubuntu/10/dockerfile)`                   | `mcr.microsoft.com/dotnet/aspnet:10.0-noble`          |
| `ghcr.io/jbaltikauskas/ubuntu-chiseled-net-10` | Production          | `[dockerfiles/ubuntu-chiseled/10/dockerfile](dockerfiles/ubuntu-chiseled/10/dockerfile)` | `mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled` |
| `ghcr.io/jbaltikauskas/alpine-net-10`          | Production (Alpine) | `[dockerfiles/alpine/10/dockerfile](dockerfiles/alpine/10/dockerfile)`                   | `mcr.microsoft.com/dotnet/aspnet:10.0-alpine`         |


### Ubuntu and Alpine: optional diagnostic tools

Two Dockerfiles share the same multi-target layout: `[ubuntu/10](dockerfiles/ubuntu/10/dockerfile)` builds `ubuntu-net-10`, and `[alpine/10](dockerfiles/alpine/10/dockerfile)` builds `alpine-net-10`. Diagnostic CLIs use a separate **build target** so the SDK stage is not pulled for lean images.


| Input                      | Default                                                  | Effect                                                                                                                                                                                                                  |
| -------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Build target               | `final-diagnostics` when `INSTALL_DIAGNOSTIC_TOOLS=true` | `final-diagnostics` installs `dotnet-debug`, `dotnet-gcdump`, and `dotnet-trace` under `/app/app-tools` (on `PATH`) and writes `/app/.info/tools.txt`. `final` skips the SDK stage entirely.                            |
| `INSTALL_DIAGNOSTIC_TOOLS` | `true`                                                   | CI/workflow selects `final-diagnostics` vs `final` (see `[.github/workflows/docker-ubuntu.yml](.github/workflows/docker-ubuntu.yml)` and `[.github/workflows/docker-alpine.yml](.github/workflows/docker-alpine.yml)`). |
| `DOTNET_VERSION`           | `10.0`                                                   | Pins the Microsoft `aspnet` / `sdk` image tag (`${DOTNET_VERSION}-noble` or `${DOTNET_VERSION}-alpine`).                                                                                                                |


**Production fleets** should use target `final` (or set `INSTALL_DIAGNOSTIC_TOOLS=false` in CI) unless you explicitly want in-image dumps and traces. Published CI defaults to `final-diagnostics` for both images.

```bash
# Lean production Ubuntu base (no SDK stage)
docker build -f dockerfiles/ubuntu/10/dockerfile \
  --target final \
  --build-arg INSTALL_DIAGNOSTIC_TOOLS=false \
  -t yourcompanyname/ubuntu-net-10:lean dockerfiles/ubuntu/10

# Lean production Alpine base (no SDK stage)
docker build -f dockerfiles/alpine/10/dockerfile \
  --target final \
  --build-arg INSTALL_DIAGNOSTIC_TOOLS=false \
  -t yourcompanyname/alpine-net-10:lean dockerfiles/alpine/10
```

Published via `[.github/workflows/docker-ubuntu.yml](.github/workflows/docker-ubuntu.yml)` and `[.github/workflows/docker-alpine.yml](.github/workflows/docker-alpine.yml)`.

What each wrapper adds on top of the upstream Microsoft image:

- **Package update and curated install set.** Ubuntu uses `apt update && apt upgrade` plus `curl`, `libicu74`, `tzdata`, `ca-certificates`, `adduser`. Alpine uses `apk update && apk upgrade` plus `ca-certificates`, `curl`, `icu-data-full`, `icu-libs`, `tzdata`. The Chiseled image uses Canonical Chisel to slice in `curl_bins`, `libicu74_libs`, `tzdata_zoneinfo` (and the legacy zoneinfo slice) without dragging in a package manager.
- **Production ASP.NET environment.** `ASPNETCORE_ENVIRONMENT=Production`, `ASPNETCORE_HTTP_PORTS=8080`, `ASPNETCORE_URLS=http://+:8080`, `DOTNET_RUNNING_IN_CONTAINER=true`, `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=false`, `LANG`/`LC_ALL=en_US.UTF-8`, `DOTNET_gcServer=1`, `DOTNET_GCHeapHardLimit=3221225472` (~3 GiB).
- **Exposed ports.** `8080` (HTTP) and `8443` (HTTPS). Kestrel binds 8080 by default; 8443 is exposed for downstream images that wire up HTTPS.
- **Non-root user.** `yourcompanyname` (UID/GID `7777`) with home `/app`, `WORKDIR /app`, and ownership/permissions set so the runtime user can read and execute under `/app`. The Chiseled variant also sets `APP_UID=7777` so the standard ASP.NET Core container conventions resolve to the same user.
- **Runtime metadata snapshots.** `/app/.info/dotnet.txt` (`dotnet --info`) and `/app/.info/linux.txt` (`/etc/os-release`) captured at build time. Ubuntu and Alpine also write `/app/.info/tools.txt` (tool list when built with `final-diagnostics`, or a note when tools are omitted on `final`).
- **Writable app state directory.** `/app/app-data`, owned by `yourcompanyname`, ready for application data.
- **Diagnostic tools (optional, Ubuntu and Alpine).** Target `final-diagnostics` populates `/app/app-tools` and prepends it to `PATH`. Chiseled has no equivalent — use sidecars or ephemeral debug pods.
- **Base HEALTHCHECK.** Validates ASP.NET runtime presence only; application images should override with an HTTP (or gRPC) liveness probe.

Upstream `mcr.microsoft.com/dotnet/aspnet:10.0-noble` and friends are Microsoft's stock images. They do not include any of the above. If you build directly on the stock images, your CI cannot surface regressions tied to the wrapper layers — those layers aren't there.

---

## Multi-stage: stock SDK + yourcompanyname runtime

Publish with `mcr.microsoft.com/dotnet/sdk:10.0-noble`, then run the published output on `ghcr.io/jbaltikauskas/ubuntu-net-10` so local builds and CI pick up the same updated packages, environment, GC and heap settings, listening ports, ICU data, and `/app` layout that production runs.

Stock Microsoft runtime, for comparison:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0-noble AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble AS runtime
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

yourcompanyname-built runtime — reuse the same `build` stage and swap only the final image:

```dockerfile
FROM ghcr.io/jbaltikauskas/ubuntu-net-10 AS runtime
WORKDIR /app
COPY --chown=7777:7777 --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Swap `ubuntu-net-10` for `ubuntu-chiseled-net-10` or `alpine-net-10` to pick a different production base. The contract on `/app`, the listening port, and the runtime user stays the same across those three.

For Ubuntu and Alpine production, prefer `--target final` when building `ubuntu-net-10` or `alpine-net-10` unless you need in-image diagnostic CLIs; see [Ubuntu and Alpine: optional diagnostic tools](#ubuntu-and-alpine-optional-diagnostic-tools).

---

## Getting started

### Prerequisites

- Docker Engine with BuildKit (`docker buildx` available).
- Git.
- A GitHub account or token only if you plan to push to GHCR (`ghcr.io`).

### Clone

```bash
git clone <your-repo-url>
cd yourcompanyname.DevOps.BaseImages
```

### Local build

Local tags below are for development only. Published image names are the `ghcr.io/jbaltikauskas/...` paths.

```bash
# Alpine .NET 10 (diagnostic tools on by default)
docker build -f dockerfiles/alpine/10/dockerfile \
  -t yourcompanyname/alpine-net-10:local dockerfiles/alpine/10

# Alpine .NET 10 lean (no diagnostic CLIs — preferred for production)
docker build -f dockerfiles/alpine/10/dockerfile \
  --target final \
  --build-arg INSTALL_DIAGNOSTIC_TOOLS=false \
  -t yourcompanyname/alpine-net-10:lean dockerfiles/alpine/10

# Ubuntu Noble .NET 10 (diagnostic tools on by default)
docker build -f dockerfiles/ubuntu/10/dockerfile \
  -t yourcompanyname/ubuntu-net-10:local dockerfiles/ubuntu/10

# Ubuntu Noble .NET 10 lean (no diagnostic CLIs — preferred for production)
docker build -f dockerfiles/ubuntu/10/dockerfile \
  --target final \
  --build-arg INSTALL_DIAGNOSTIC_TOOLS=false \
  -t yourcompanyname/ubuntu-net-10:lean dockerfiles/ubuntu/10

# Ubuntu Noble Chiseled .NET 10
docker build -f dockerfiles/ubuntu-chiseled/10/dockerfile \
  -t yourcompanyname/ubuntu-chiseled-net-10:local dockerfiles/ubuntu-chiseled/10
```

### Smoke check

Each image's entrypoint is `dotnet`, so passing `--info` runs `dotnet --info`:

```bash
docker run --rm yourcompanyname/ubuntu-net-10:local --info
docker run --rm yourcompanyname/alpine-net-10:local --info
docker run --rm yourcompanyname/ubuntu-chiseled-net-10:local --info

# Ubuntu or Alpine with tools: inspect baked-in tool versions
docker run --rm yourcompanyname/ubuntu-net-10:local cat /app/.info/tools.txt
docker run --rm yourcompanyname/alpine-net-10:local cat /app/.info/tools.txt
```

You should see the runtime version, host OS info, and available SDKs/runtimes for that image.

---

## Published images (GHCR)

The CI workflows in `.github/workflows/docker-*.yml` push to GitHub Container Registry at `ghcr.io/<lowercase-owner>/<image-name>`. The owner segment is the GitHub owner of this repository (not the repo name). For `jbaltikauskas`:

```bash
docker pull ghcr.io/jbaltikauskas/alpine-net-10
docker pull ghcr.io/jbaltikauskas/ubuntu-net-10
docker pull ghcr.io/jbaltikauskas/ubuntu-chiseled-net-10
```

The same commands work from PowerShell, Command Prompt, and bash. Each image is tagged `latest` and with a UTC publish stamp `yyyyMMdd-HHmm`. For reproducible builds, pin to a specific stamp tag or a digest rather than `latest`. Public packages pull without authentication; private packages need `docker login ghcr.io`.

Published images are signed with [cosign](https://github.com/sigstore/cosign) via the keyless workflow. The signing step runs against the build digest, so signatures are tied to the exact image content, not the tag.

---

## Troubleshooting

**`docker buildx` not available.** Install or enable Docker Buildx in your Docker installation. Older Docker installs may need an explicit `docker buildx install` or a newer Docker Desktop.

**Cannot pull `mcr.microsoft.com/dotnet/aspnet:10.0-*`.** Check outbound network access to `mcr.microsoft.com` and that the Docker daemon is running. Corporate proxies sometimes need to be added to the Docker engine config.

**GHCR push or auth errors.** Authenticate Docker to GHCR (`docker login ghcr.io`) and confirm the token has `write:packages`. The workflow uses `GITHUB_TOKEN` with `packages: write`, which requires the package to allow the repo as a source.

**Chiseled image won't open a shell.** That's expected — there is no shell. For debugging, copy a busybox or debug sidecar image into the same pod, or rebuild against the full Noble image temporarily.

**Alpine app throws `DllNotFoundException`.** The library is almost always missing a musl-compatible native dep. Either add the appropriate `apk` packages, or switch the base to `noble-chiseled-extra` and run on glibc.