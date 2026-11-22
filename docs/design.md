# berth design

Status: draft for review, 2026-11-22.

This document is the overall design for berth. It sets the goals, the architecture, the compatibility contract, the phases and the way the project is built and tested. Each phase after the first gets its own short spec under `docs/specs/` before work starts, and each piece of work gets a plan under `docs/plans/`.

## 1. Goals

berth is an open source container platform written in Rust. People should be able to remove Docker, install berth, and keep working the same day. After that, berth should be better than what it replaced.

1. **Drop-in replacement.** The `docker` command, the Docker Engine API on the usual socket, Dockerfiles, `docker-compose.yml` and Swarm stack files all work unchanged. Tools built on the Engine API (docker compose, VS Code Dev Containers, Testcontainers, CI runners) work without changes.
2. **Complete.** The phases reach parity with Docker Engine, Compose, BuildKit and Swarm, and cover the Podman features people choose Podman for. Section 7 lists every command and capability with the phase that delivers it.
3. **Better.** Short app files with smart defaults are the core difference. Around that: a smoother dev loop, simple deploys to servers, safer defaults, a terminal dashboard, and lower memory and faster starts. Every "better" claim is a target until a benchmark backs it (section 8).
4. **Everywhere.** Linux natively. Windows through a WSL2 distro that berth creates and manages. macOS through a lightweight VM.
5. **Open to contributors.** Anyone can file issues and send pull requests. The repository has the rules, templates, labels, milestones and project board that make that work.

### Non-goals

- Windows containers (Windows Server container images).
- cgroup v1 hosts.
- Docker Desktop extensions, Docker Scout, Docker Hub's hosted services, Docker AI and model runner features.
- Managed Engine plugins (`docker plugin`, plugin API v2). berth offers built-in drivers and documented extension points instead.
- A registry server. A separate Rust registry project covers that; berth pushes to and pulls from it like any OCI registry.
- Rancher-style multi-cluster management. A separate project covers it; berth exposes the API it needs.
- Large-cluster orchestration. A separate Rust Kubernetes alternative covers that and can use berth's runtime (section 11).

## 2. Decisions so far

| Topic | Decision |
|---|---|
| Name | berth, repository `raven-clown/berth` |
| License | Apache-2.0 |
| Audience | Open source for others to adopt |
| Approach | Own the core in Rust, reuse proven crates at the edges (libcontainer, oci-client, rtnetlink) |
| Process model | One multicall binary: CLI, daemon, per-container shim, runc-compatible runtime |
| API | Docker Engine API from the first release, plus a berth API under `/berth/v1` on the same socket |
| Config | Short YAML (`berth.yml`) that is a superset of the Compose Spec; `compose.yaml` runs unchanged |
| Rootless | Rootful first; every layer designed so rootless can follow in phase 6 |
| Platforms | Linux and Windows (WSL2) in the MVP, macOS in phase 2 |
| Cluster | Small Swarm-style clusters (3 to 50 nodes); Swarm stack files deploy unchanged |
| Contributions | DCO sign-off required on outside pull requests |

## 3. Prior art and what berth reuses

Research notes with versions and sources were taken on 2026-11-22. The short version:

| Area | Choice | Why |
|---|---|---|
| Low-level runtime | `libcontainer` and `libcgroups` 0.7 (youki, Apache-2.0) | Usable as a library: create, start, exec (tenant), kill, pause, checkpoint, cgroups v2, seccomp, rootless. Pre-1.0 and thinly documented, so berth wraps it behind its own `Runtime` trait and pins exact versions. |
| OCI types | `oci-spec` 0.10 | Same version libcontainer re-exports. |
| Registry client | `oci-client` 0.18 | Docker Hub token flow, platform resolver for multi-arch, streamed blobs. berth adds credential helpers, `~/.docker/config.json`, and digest checks on resumed downloads. |
| Layer archives | `tar` 0.4 in blocking threads | `tokio-tar` is abandoned and unpatched for CVE-2025-62518. berth handles whiteouts and path escapes itself and fuzzes that code. |
| Image layout | `ocidir` | OCI layout import and export. |
| Overlay storage | Own snapshotter | No maintained Rust snapshotter exists. Kernel overlay through the new mount API is a small, well-understood piece. |
| Networking | Own code on `rtnetlink` and `nftables` | netavark has no supported library API and is version-locked to Podman. berth studies its bridge and nftables code. |
| Dockerfile parsing | `parse-dockerfile` | Maintained parser. The build evaluator (ARG scoping, `--mount`, heredocs, stages) is berth's own. |
| Builder | Own DAG builder | No mature Rust BuildKit alternative exists. |
| CRI | Vendored CRI protobufs with `tonic` | No production Rust CRI server exists. |
| Low-level syscalls | `rustix` for new code, `nix` where libcontainer needs it | |
| Async | `tokio`, kept out of any process that forks into a container | Forking a multithreaded tokio process is unsafe, so the shim is synchronous. |

Where berth has to beat others:

- **Docker 29:** the default everywhere. Complaints: root daemon, iptables and firewall breakage, Desktop licensing and RAM, verbose compose files.
- **Podman 6:** rootless and daemonless. Complaints: podman-compose is not drop-in, rootless networking friction, Quadlet is even more verbose than compose.
- **containerd and nerdctl:** stable but built for platforms, many moving parts.
- **BuildKit:** fast and powerful. Complaints: opaque caching, builder-instance confusion, hard debugging.
- **Swarm:** simple, but stagnant. No autoscaling, weak volume story, overlay bugs.
- **Uncloud:** the closest "simpler Swarm" (WireGuard mesh, no control plane) and the main multi-host design to beat.
- **Kamal:** tiny deploy story, but one app plus accessories rather than a stack.

Honest size: a solo developer needs about 35 to 50 full-time weeks for a single-host platform with reuse, and cluster mode roughly doubles that. Full parity plus the extras in this document is a multi-year project. The phases are cut so each release is useful on its own, and the repository is set up for other contributors from the start.

## 4. Architecture

### 4.1 Processes

berth ships as one static binary. What it does depends on the subcommand or the name it was called by.

| Role | How it runs | Platforms | Job |
|---|---|---|---|
| CLI | `berth ...`, or `docker ...` through an alias | Linux, Windows, macOS | Docker-compatible commands plus berth's own. Talks to the daemon over the API. |
| Daemon | `berth daemon` (installed as `berthd` service) | Linux (inside the WSL2 distro or VM on other systems) | Serves the Engine API and the berth API, owns state, images, networks, volumes and app reconciliation. |
| Shim | `berth shim`, one per container | Linux | Synchronous, no tokio. Creates the container through libcontainer, holds its stdio, writes logs, records the exit code. Outlives daemon restarts and upgrades. |
| Runtime | `berth runtime` or a `berth-runc` symlink | Linux | runc-compatible CLI (`create`, `start`, `state`, `kill`, `delete`, `exec`) for containerd, CRI-O and the Kubernetes project. |

### 4.2 Crates

```
crates/
  berth-core      ids, names, paths (rootful and rootless), errors, events
  berth-runtime   Runtime trait, libcontainer adapter, OCI spec builder, default seccomp and capabilities
  berth-shim      shim process and the daemon-to-shim protocol
  berth-image     content store, pull, push, unpack, overlay snapshotter, GC, save and load, import from Docker
  berth-net       bridge, veth, IPAM, nftables NAT and port publishing, embedded DNS, pasta backend (rootless)
  berth-volume    named, bind, tmpfs and idmapped mounts, volume drivers
  berth-build     Dockerfile to DAG, parallel executor, cache, multi-stage, mounts, secrets
  berth-app       berth.yml, Compose Spec and stack file parsing into one model, recipes, reconcile
  berth-api       Docker Engine API and berth API (axum), version negotiation, Docker error format
  berth-daemon    wiring: config, state store, event bus, recovery on start
  berth-machine   managed WSL2 distro on Windows, VM on macOS, path translation, socket relay
  berth-proxy     local HTTPS proxy and `.localhost` routing (phase 3), deploy proxy (phase 7)
  berth-cluster   Raft, WireGuard mesh, scheduler, services and tasks (phase 8)
  berth-cri       CRI server (phase 9)
  berth-tui       terminal dashboard (ratatui)
  berth           the multicall binary and the CLI
```

Each crate has one job and a small public API. Library crates never print and never read the process environment directly. `berth-runtime`, `berth-image` and `berth-net` are published to crates.io so other projects can reuse them.

### 4.3 State and storage

Rootful layout (rootless uses the XDG equivalents under the user's home):

```
/etc/berth/berthd.yml          daemon config, every key in docs/config.md, BERTH_* env overrides
/var/lib/berth/state.redb      containers, images, networks, volumes, apps (redb, pure Rust)
/var/lib/berth/content/sha256  blobs, written to a temp file, digest-checked, fsynced, renamed
/var/lib/berth/snapshots/      overlay lower and upper directories
/var/lib/berth/containers/<id> bundle (config.json), logs, hostname, resolv.conf, hosts
/var/lib/berth/volumes/<name>  named volume data
/run/berth/berthd.sock         API socket
/run/berth/shims/<id>.sock     one control socket per shim
```

The daemon can also listen on `/var/run/docker.sock`. It only takes that path when no Docker daemon owns it, so Docker and berth can run side by side during a migration. The default bridge subnet is `10.90.0.0/16` so it collides with neither Docker (`172.17.0.0/16`) nor Podman (`10.88.0.0/16`).

### 4.4 Flow of `berth run -p 8080:80 nginx`

1. The CLI resolves the context (`DOCKER_HOST`, berth contexts) and calls `POST /containers/create`.
2. `berth-image` resolves `nginx` to `docker.io/library/nginx:latest`, picks the host platform from the index, and pulls missing blobs into the content store.
3. The snapshotter prepares an overlay rootfs from the layer chain.
4. `berth-net` creates a veth pair, attaches it to the bridge, allocates an address, and adds the nftables DNAT rule for port 8080.
5. `berth-runtime` writes the bundle `config.json` with Docker's default capabilities, seccomp profile, masked paths and mounts.
6. The daemon spawns `berth shim`, which calls libcontainer to create and start the container, then reports the pid.
7. The daemon emits events; the CLI streams them and attaches to output as `docker run` does.

### 4.5 Daemon to shim protocol

Length-prefixed messages over the shim's Unix socket: `Start`, `Exec`, `Signal`, `Resize`, `Attach`, `Stats`, `Wait`, `Delete`. The shim writes logs in Docker's `json-file` format with rotation, so `docker logs` semantics hold and existing log shippers keep working. On start, the daemon lists `/run/berth/shims`, reconnects to every live shim, and reconciles with `state.redb`: containers whose shim died are marked exited with the recorded code, and their restart policy applies.

### 4.6 Security defaults

The defaults match Docker so images behave the same: the moby default seccomp profile (Apache-2.0, vendored), the same default capability set, masked and read-only paths, and an AppArmor profile equivalent to `docker-default` where AppArmor is active. berth adds opt-in hardening (`no-new-privileges`, read-only rootfs per app, user namespaces) and, in phase 6, rootless mode and microVM isolation.

### 4.7 Errors

API errors use Docker's JSON shape (`{"message": "..."}`) and status codes, so clients that parse them keep working. CLI exit codes match docker: 125 for daemon or berth errors, 126 when the command cannot run, 127 when it is not found, otherwise the container's exit code. berth-only commands give errors with a cause and a suggested fix, and config errors point at the file, line and column.

### 4.8 Windows and macOS

- **Windows (MVP).** `berth machine init` imports a small distro named `berth` into WSL2. Its rootfs is built in CI and ships berthd with systemd enabled. `berth.exe` serves `\\.\pipe\docker_engine` (optional) and `\\.\pipe\berth` and relays them to the daemon. Phase 1 relays over a localhost TCP port inside the distro protected by a per-machine token file readable only by the user; Hyper-V sockets are evaluated as a faster transport. Bind mounts from Windows paths (`C:\code\app`) are translated to `/mnt/c/code/app`, and berth warns when a project lives on a Windows drive because file access there is slow.
- **macOS (phase 2).** A small Linux VM on Virtualization.framework, API over vsock, project files shared with virtiofs. GitHub's Apple silicon runners cannot run nested VMs, so macOS machine tests run on a real Mac before each release.

## 5. App files and smart defaults

### 5.1 Finding the file

`berth up` uses `berth.yml` if it exists, then `compose.yaml`, `compose.yml`, `docker-compose.yaml`, `docker-compose.yml`. `-f` picks a file explicitly, as with compose.

### 5.2 Format

A berth.yml is a map of services. A service value is either a short string or a mapping.

- A string starting with `.` or `/` is a build context: `web: .`
- Any other string is an image reference: `db: postgres:17`
- A mapping uses berth's short keys, and also accepts every Compose Spec service key, so anything compose can express can be written in berth.yml.

```yaml
web:
  build: .
  port: 3000
db: postgres:17
cache: redis
```

This file does what a compose file of about 40 lines does: a build, three services, a network, a data volume for Postgres, health checks, credentials, connection settings, start order and a URL for the web app.

Short keys:

| Key | Meaning |
|---|---|
| `build` | Build context, or a mapping with `dockerfile`, `target`, `args` |
| `image` | Image reference |
| `port` | Container port (number or list). HTTP ports get a `.localhost` URL; nothing binds on the host unless `publish` says so |
| `publish` | Host ports, as in `docker run -p` |
| `env` | Map or list. `.env` next to the file is loaded automatically |
| `cmd` | Command, string or list |
| `volumes` | `name:/path`, `./dir:/path`, or just `/path` for an automatic named volume |
| `replicas` | Number of instances |
| `inject` | `false` to turn off automatic connection variables for this service |

### 5.3 Recipes

A recipe describes how a popular image is normally run. berth ships recipes for postgres, mysql, mariadb, redis, valkey, mongo, rabbitmq, kafka, minio, opensearch, elasticsearch and nginx, and anyone can add more:

- Recipes are YAML files embedded in the binary, overridable from `~/.config/berth/recipes/` or a project's `recipes/` folder.
- Image authors can ship a recipe with the image as an OCI artifact attached to it or as an image label.
- A recipe sets the data volume path, the health check, development credentials (generated once per app and stored as berth secrets), and the connection URL it exports.

Connection variables follow one rule: every service that is not itself a recipe service gets `<SERVICE>_URL` for every recipe service in the app (`DB_URL`, `CACHE_URL`). The conventional name (`DATABASE_URL`, `REDIS_URL`) is added only when the app has exactly one service of that kind, so it is never ambiguous. `inject: false` turns this off.

### 5.4 Defaults for every service in a berth.yml

- One network per app, with service names resolvable by DNS.
- Restart policy `unless-stopped`.
- Log rotation (10 MB, 3 files).
- Start order derived from references: a service that uses another service's variables waits until that service is healthy. No `depends_on` needed; explicit `depends_on` still works.
- HTTP services reachable at `https://<service>.<app>.localhost` through berth's local proxy, with certificates from a local CA berth installs on request (phase 3).

### 5.5 Environments

```yaml
envs:
  prod:
    web: { replicas: 3 }
    db: { image: postgres:17, volumes: ["pgdata:/var/lib/postgresql/data"] }
```

`berth up --env prod` and `berth deploy --env prod` merge the named overlay. No override files.

### 5.6 Transparency

- `berth config` prints the fully expanded app, as `docker compose config` does.
- `berth explain <service>` prints every setting with its source: user file, recipe, default or environment overlay.
- A JSON Schema for berth.yml is published for editor completion and validation.

### 5.7 Compose files

`compose.yaml` is parsed against the full Compose Spec into the same model. Smart defaults are not applied to compose files, so they behave exactly as with docker compose. A compose file opts in with a top-level `x-berth: smart`. `berth convert compose.yaml` writes the equivalent berth.yml, drops every value that equals a default, and reports how many lines it saved. Swarm stack files (compose files with `deploy:` sections) load into the same model and deploy to a cluster in phase 8.

Language detection (building a project without a Dockerfile) is out of scope until after v1.0.

## 6. Compatibility

### 6.1 Contract

| Layer | Promise | From phase |
|---|---|---|
| Engine API | The API version Docker Engine 29.x reports, with version negotiation and the same minimum client version Docker 29 accepts | 1 (subset), 5 (complete) |
| Socket | `/var/run/docker.sock` on Linux when free, `\\.\pipe\docker_engine` on Windows, `~/.berth/run/docker.sock` on macOS, `DOCKER_HOST` honored | 1 |
| CLI | When called as `docker`, the same commands, flags, output, `--format` Go templates and exit codes. `berth install --docker-alias` sets it up | 1 (subset), 5 (complete) |
| docker compose | The real `docker compose` plugin works against berthd, and `berth compose` / `berth up` run the same files natively | 3 |
| Dockerfile | Dockerfile frontend syntax as of BuildKit 0.35 / frontend 1.29 | 4 |
| docker build | Legacy `/build` endpoint first, then the BuildKit control API subset that buildx uses with the docker driver | 4 |
| Stack files | `docker stack deploy` and `berth stack deploy` with unchanged files | 8 |
| Migration | `berth migrate` copies images, volumes, networks and container definitions from a running Docker through its own API, then can switch the socket | 5 |

Go templates for `--format` need a Rust implementation of Go's `text/template` with the functions docker adds (`json`, `table`, `join`, `upper`, and the rest). berth uses or forks an existing crate if one passes docker's format tests, and writes its own otherwise.

`docker build` with today's docker CLI talks BuildKit's gRPC API through the daemon. Until phase 4 finishes that API, `docker build` works with `DOCKER_BUILDKIT=0` and `berth build` works fully. `docs/compat.md` says so plainly.

### 6.2 Compatibility tests

The `compat` CI job runs real upstream tools against a berthd started in the job:

- moby's integration tests, pointed at berthd with `DOCKER_HOST`
- docker/cli e2e tests
- docker/compose e2e tests
- Testcontainers test suites (Java and Go first)
- `@devcontainers/cli` tests
- From phase 8, Swarm integration tests and `docker stack deploy` scenarios
- The sibling registry project as a push and pull target, next to Docker Hub and a local `registry:2`

Each suite records pass, fail and skip per test. `docs/compat.md` is generated from those results with each phase. A test that passed before and fails now fails the job. v1.0 requires the pass rates set in the phase 5 spec.

### 6.3 Migration guide

`docs/migrate.md` covers: installing berth next to Docker, `berth migrate`, switching the socket and the `docker` alias, moving CI, known differences, and rolling back.

## 7. Feature matrix

"Better" entries are targets until section 8 benchmarks or tests back them. Columns: D = Docker 29 (Engine, Compose v5, BuildKit 0.35, Swarm), P = Podman 6.2, Ph = berth phase.

### 7.1 Containers

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| `run`, `create`, `start` | yes | yes | 1 | Faster cold and warm start, no containerd hop | bench: `run --rm alpine true` |
| `ps`, `inspect`, `logs`, `wait` | yes | yes | 1 | `logs` across a whole app with colors per service | golden output tests |
| `exec`, `attach` | yes | yes | 1 | | compat suites |
| `stop`, `kill`, `restart`, `rm` | yes | yes | 1 | | compat suites |
| `pause`, `unpause` | yes | yes | 2 | | compat suites |
| `cp`, `diff`, `export` | yes | yes | 2 | | compat suites |
| `top`, `stats` | yes | yes | 2 | Live view in the TUI | compat suites |
| `port`, `rename`, `update` | yes | yes | 2 | | compat suites |
| `commit` | yes | yes | 2 | | compat suites |
| `events` | yes | yes | 2 | | compat suites |
| Restart policies, health checks | yes | yes | 2 | Start order waits for health by default in berth.yml | integration tests |
| `container prune` | yes | yes | 2 | | compat suites |
| `checkpoint create/ls/rm` (CRIU) | experimental | yes | later | | integration tests |
| Resource limits (cgroups v2) | yes | yes | 1 | | integration tests |
| Devices, GPUs (CDI) | yes | yes | later | | integration tests |
| Rootless | yes (separate setup) | yes (default) | 6 | One install path, pasta networking without manual setup | integration tests on rootless runners |
| microVM isolation per service | no | via krun | 6 | One key in berth.yml (`isolation: vm`) | integration tests |
| Pods (shared namespaces) | no | yes | 3 | Services in a berth.yml can share namespaces with `with: <service>` | integration tests |

### 7.2 Images

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| `pull`, `images`, `image ls/inspect/rm`, `rmi` | yes | yes | 1 | Parallel layer pulls, resumable downloads | bench: pull time |
| `tag`, `push` | yes | yes | 2 | | compat suites, registry project |
| `save`, `load`, `import`, `history` | yes | yes | 2 | | compat suites |
| `image prune`, `system df`, `system prune` | yes | yes | 2 | `system df` explains what is safe to delete | golden tests |
| `login`, `logout`, credential helpers | yes | yes | 1 | | integration tests with fake registry |
| `search` | yes | yes | 2 | | compat suites |
| `manifest create/inspect/push/annotate/rm` | yes | yes | 4 | | compat suites |
| Multi-arch pull | yes | yes | 1 | | integration tests |
| Lazy pulling (eStargz or similar) | via snapshotters | partial | later | Start before the image is fully pulled | bench |
| Content trust (`trust`, Notary v1) | deprecated | no | no | Replaced by sigstore verification | |
| Signature verification (sigstore) | no | yes (policy.json) | 6 | Policy in berth.yml per app | integration tests |
| SBOM, vulnerability scan | Scout (cloud) | no | 6 | Local, offline-capable (OSV database) | integration tests |
| Import from Docker | n/a | partial | 5 | `berth migrate` in one command | migration tests |

### 7.3 Build

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| `build` (Dockerfile syntax 1.29) | yes | yes (Buildah) | 4 | Cache decisions explained per step (`berth build --explain`) | BuildKit Dockerfile frontend tests |
| Multi-stage, `--target` | yes | yes | 4 | | frontend tests |
| `RUN --mount=type=cache/secret/ssh/bind/tmpfs` | yes | yes | 4 | | frontend tests |
| Heredocs, `# syntax=` directive | yes | yes | 4 | `# syntax=` frontends other than Dockerfile are not run | frontend tests |
| Parallel stages | yes | partial | 4 | | bench: build time |
| Multi-platform (QEMU binfmt) | yes | yes | 4 | | integration tests |
| `buildx build/ls/create/use/rm/inspect/prune/du/imagetools` | yes | n/a | 4 | No builder instances to manage | compat suites |
| `buildx bake` | yes | no | 4 | berth.yml services build together without a bake file | compat suites |
| Cache export and import (registry, local) | yes | partial | 4 | | integration tests |
| Attestations (provenance, SBOM) | yes | partial | 6 | | integration tests |
| Debugging a failed step | `buildx debug` (limited) | no | 4 | `berth build` drops into a shell at the failed step on request | integration tests |
| Builds without a Dockerfile | `docker init` templates | no | after 1.0 | | |

### 7.4 Networking

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| Default bridge, `-p` publishing | yes | yes | 1 | Own nftables table only, never rewrites other rules; works next to ufw and firewalld | integration tests with ufw and firewalld |
| `network create/ls/inspect/rm/prune` | yes | yes | 2 | | compat suites |
| `network connect/disconnect`, aliases | yes | yes | 2 | | compat suites |
| Embedded DNS | yes | yes (aardvark) | 2 | | integration tests |
| `host`, `none`, `container:` modes | yes | yes | 1 (host, none), 2 (container) | | integration tests |
| IPv6 | yes | yes | 2 | | integration tests |
| macvlan, ipvlan | yes | yes | later | | integration tests |
| Rootless networking (pasta) | yes (slirp/rootlesskit) | yes (pasta) | 6 | | bench: throughput |
| Local HTTPS and `.localhost` names | no | no | 3 | No port juggling across projects | e2e tests |
| Overlay networks (cluster) | yes | no | 8 | WireGuard mesh, encrypted by default | bench: throughput against Swarm overlay |
| Ingress routing mesh | yes | no | 8 | Hostname routing built in | e2e tests |

### 7.5 Volumes and storage

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| Named, bind, tmpfs mounts | yes | yes | 1 (named, bind), 2 (tmpfs) | | integration tests |
| `volume create/ls/inspect/rm/prune` | yes | yes | 1 | | compat suites |
| Volume drivers with options (nfs, local) | yes | yes | 2 | | integration tests |
| Idmapped mounts | partial | yes | 6 | | integration tests |
| Volume backup and restore | manual | export/import | 3 | `berth volume backup` and `restore` | integration tests |
| Cluster volumes (CSI) | yes (new) | no | later | | |
| Windows path translation | Desktop | machine | 1 | | machine tests |

### 7.6 Compose and apps

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| Compose Spec files | yes | partial (podman-compose) | 3 | Same parser, drop-in | compose e2e tests |
| `compose up/down/ps/logs/exec/run/build/pull/push` | yes | partial | 3 | | compose e2e tests |
| `compose start/stop/restart/pause/unpause/kill/rm/create` | yes | partial | 3 | | compose e2e tests |
| `compose config/images/port/top/events/cp/ls/version/wait/stats/attach/scale/export/commit/volumes` | yes | partial | 3 | | compose e2e tests |
| `compose watch` | yes | no | 3 | `berth up --watch` with sync, rebuild and restart rules inferred from the build context | e2e tests |
| `compose publish` (OCI artifact) | yes | no | 4 | | compose e2e tests |
| Profiles, `include`, `extends`, `x-` fields | yes | partial | 3 | `envs:` replaces override files | parser tests |
| Top-level `jobs:` | yes (v5.8) | no | 3 | | compose e2e tests |
| Short app format with recipes | no | no | 3 | Few lines for common stacks | line counts on reference apps |
| `berth explain`, `berth convert` | no | no | 3 | | golden tests |
| `kube play`, `kube generate` | no | yes | later | | integration tests |
| systemd units (Quadlet equivalent) | no | yes | later | Generated from berth.yml, not written by hand | integration tests |
| Auto-update | no | yes | later | | integration tests |
| Terminal dashboard | no (Desktop GUI) | no (Desktop GUI) | 3 | Apps, logs, stats, exec and restart in one screen | TUI golden tests |

### 7.7 Deploy and cluster

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| `context create/ls/use/rm/inspect/show/update/export/import` | yes | `system connection` | 1 (env and use), 2 (complete) | | compat suites |
| Deploy to servers over SSH, zero downtime | no (Kamal is separate) | no | 7 | Same berth.yml from dev to servers | e2e tests with VMs |
| `swarm init/join/join-token/leave/update/unlock/unlock-key/ca` | yes | no | 8 | | Swarm integration tests |
| `node ls/inspect/ps/promote/demote/update/rm` | yes | no | 8 | | Swarm integration tests |
| `service create/ls/ps/inspect/logs/update/scale/rollback/rm` | yes | no | 8 | Automatic rollback on failed health | Swarm integration tests |
| `stack deploy/ls/ps/services/rm/config` | yes | no | 8 | Unchanged stack files | Swarm integration tests |
| `secret`, `config` (`create/ls/inspect/rm`) | yes | `secret` only (local) | 3 (local), 8 (cluster) | | Swarm integration tests |
| Rolling updates, placement constraints | yes | no | 8 | | Swarm integration tests |
| Autoscaling | no | no | after 2.0 | | |
| CRI for Kubernetes | via containerd | via CRI-O | 1 (`berth runtime`), 9 (CRI) | | `critest` |

### 7.8 System and tooling

| Command or capability | D | P | Ph | Where berth aims to be better | How it is checked |
|---|---|---|---|---|---|
| `version`, `info` | yes | yes | 1 | | golden tests |
| `system events/df/prune/info` | yes | yes | 2 | | compat suites |
| `--format` Go templates | yes | yes | 1 (common fields), 5 (complete) | | docker/cli format tests |
| Daemon config | `daemon.json` | `containers.conf` | 1 | `berth migrate` translates `daemon.json` | migration tests |
| Logging drivers (`json-file`, `local`, `journald`, `syslog`, `none`) | yes | yes | 1 (json-file, none), 2 (rest) | | integration tests |
| `plugin *` | yes | no | no | Built-in drivers instead | |
| `init`, `scout`, `debug`, `model`, Desktop extensions | Desktop/cloud | no | no | | |
| `machine init/start/stop/rm/ls/ssh` | Desktop | yes | 1 (Windows), 2 (macOS) | One command, no separate desktop app | machine tests |
| Shell completion | yes | yes | 1 | | |

## 8. Proving "better"

`bench/` holds `berth-bench`, which runs the same scenarios against Docker, Podman and berth in the same CI job on the same machine, repeats each scenario, and reports median and p95. Hosted runners are noisy, so results are only compared within one job. Results are published per release.

| Scenario | Metric |
|---|---|
| `run --rm alpine true` cold and warm | wall time |
| Daemon idle | resident memory (berthd vs dockerd plus containerd) |
| 50 idle containers | memory per container (shim vs containerd-shim) |
| Pull of a reference image set | wall time, bytes on disk |
| Build of reference projects (Node, Go, Rust, Python) with and without cache | wall time |
| `up` of a 10-service reference app | time until every service is healthy |
| Reference apps in berth.yml vs compose | lines and characters |
| Cluster rolling update of 20 replicas | time, failed requests during the update |
| Overlay network | throughput and latency between nodes |

The phase 0 work records a baseline for Docker and Podman. Each later phase spec sets numeric targets from that baseline. Until a target is met, docs and the README do not claim it.

Claims that will be hard to back up, stated now:

- **Build speed.** BuildKit is mature and well optimized. Beating it on raw speed is uncertain. The likelier wins are cache transparency and debugging failed steps.
- **Runtime security.** runc has far more users than libcontainer. berth can claim safer defaults (rootless, microVM, signature policy), not a more secure low-level runtime.
- **Cluster networking.** WireGuard against Swarm overlay needs real throughput numbers before any claim.

## 9. Phases

Each phase ends in a release and has a milestone on GitHub. Estimates assume one person full-time and will shrink with contributors.

| Phase | Version | Delivers | Estimate |
|---|---|---|---|
| 0 Foundations | none | Workspace, CI with privileged Linux jobs, WSL2 dev setup, AGENTS.md, benchmark baseline harness | 1 to 2 weeks |
| 1 MVP: run | v0.1 | Pull from Docker Hub (auth, multi-arch), overlay snapshots, libcontainer plus shim, `run/create/start/ps/logs/exec/attach/wait/stop/kill/restart/rm/pull/images/rmi/login/logout/inspect/version/info`, default bridge with `-p`, host and none networks, named and bind volumes with `volume` commands, json-file logs, Engine API subset for all of these so the real `docker` CLI drives it, Windows CLI through the managed WSL2 distro, `berth runtime` as a runc replacement | 12 to 16 weeks |
| 2 Single host complete | v0.2 | User networks with DNS, connect and disconnect, IPv6, restart policies, health checks, `cp/diff/export/stats/top/events/pause/commit/port/rename/update`, tmpfs, more log drivers, tag, push, save, load, contexts, macOS VM | 8 to 12 weeks |
| 3 Apps | v0.3 | berth.yml, full Compose Spec, recipes, `explain` and `convert`, docker compose against berthd, local HTTPS proxy and `.localhost` names, `up --watch`, local secrets, volume backup, pods, first TUI | 10 to 14 weeks |
| 4 Build | v0.4 | DAG builder, cache, multi-stage, mounts, secrets, SSH, multi-platform, legacy `/build`, BuildKit control API subset for buildx, manifest commands, compose publish | 14 to 20 weeks |
| 5 Drop-in | v1.0 | Complete Engine API and CLI, complete Go templates, `berth migrate`, `--docker-alias`, compat suites at the agreed pass rates | 10 to 14 weeks |
| 6 Rootless and security | v1.1 | Rootless with pasta, idmapped mounts, microVM isolation (libkrun), sigstore policy, SBOM and attestations, OSV scanning | 10 to 14 weeks |
| 7 Deploy | v1.2 | `berth deploy` over SSH with zero-downtime switching, the same berth.yml and `envs` | 6 to 8 weeks |
| 8 Cluster | v2.0 | Swarm API, Raft (openraft), WireGuard mesh, services, replicas, rolling update and rollback, ingress, cluster secrets and configs, unchanged stack files | 20 to 30 weeks |
| 9 Kubernetes | v2.1 | CRI server passing `critest` | 6 to 8 weeks |
| Later | | Lazy pulling, composefs-backed store, checkpoint and restore, GPUs through CDI, auto-update, systemd unit generation, `kube play`, macvlan and ipvlan, builds without a Dockerfile, autoscaling | |

The order follows adoption: people switch when a single host replaces Docker, so v1.0 is the single-host drop-in. The things that make berth better than Docker (short config, the dev loop, the TUI) land from phase 3, before v1.0.

## 10. Development, testing and CI

### 10.1 Developing on Windows 11

Container features need a Linux kernel. Development on Windows uses WSL2:

- A dedicated distro (Ubuntu 24.04 or 26.04), not one shared with Docker Desktop, because both would manage nftables in the same VM.
- `systemd=true` in `/etc/wsl.conf`. Check for a pure cgroup v2 hierarchy; if WSL mounts a hybrid layout, set `kernelCommandLine = cgroup_no_v1=all` under `[wsl2]` in `%UserProfile%\.wslconfig` and run `wsl --shutdown`.
- The source can stay at `D:\workspace\berth` for editing. `CARGO_TARGET_DIR` and berth's store (`/var/lib/berth`) live on the distro's ext4 disk, because overlay upper directories on `/mnt/d` fail and drvfs is slow.
- `scripts/wsl-setup.sh` installs the toolchain and packages (`uidmap`, `nftables`, `pasta`) and checks the kernel features. `just test-linux` runs the privileged tests inside the distro.
- Crates that do not touch Linux APIs build and test on Windows with plain `cargo test`.

### 10.2 Test layers

- **Unit tests** in every crate, written first, table-driven where there is a table. No network, no root, no bare sleeps.
- **Fakes**: `Runtime` and `Network` fakes for daemon tests, and an in-process axum OCI registry that serves fixtures, so pull and push are tested offline.
- **Privileged integration tests**, marked `#[ignore]`, run with `cargo nextest run --run-ignored only` under `sudo -E` in the privileged job: namespaces, cgroups v2, overlay, veth, nftables, shim recovery after a daemon kill.
- **Golden tests** for CLI output, compared with docker's output byte for byte.
- **Fuzzing** with `cargo-fuzz` for everything that reads outside input: tar unpacking (path escapes, whiteouts, links), the Dockerfile parser, the Compose and berth.yml parsers, and API request bodies.
- **Compatibility suites** (section 6.2) and **benchmarks** (section 8).
- **Network tests against Docker Hub** in a nightly job only, never on pull requests.

### 10.3 CI

GitHub Actions, with runner labels pinned (`ubuntu-24.04`, `ubuntu-24.04-arm`; `ubuntu-latest` moved to 26.04 in 2026):

| Job | Runs |
|---|---|
| `lint` | `cargo fmt --check`, `cargo clippy -- -D warnings`, `cargo deny check` (licenses, advisories) |
| `test` | Unit tests on ubuntu-24.04, ubuntu-24.04-arm, windows-latest, macos-latest |
| `integration` | Privileged tests on ubuntu-24.04 and ubuntu-24.04-arm |
| `compat` | Upstream suites against berthd, growing with each phase |
| `dco` | `Signed-off-by` on every commit of a pull request |
| `bench` | Nightly |
| `release` | On a `vX.Y.Z` tag: cargo-dist builds static musl binaries for Linux, installers for Windows and macOS, and the WSL2 distro image; notes come from the CHANGELOG |

Toolchain: Rust stable, edition 2024, MSRV two releases behind stable. Rootless jobs (phase 6) set `kernel.apparmor_restrict_unprivileged_userns=0` on the runner.

### 10.4 Code rules

These go into `AGENTS.md` and `CONTRIBUTING.md`:

- Tests with every change, written first.
- `#![forbid(unsafe_code)]` in every crate except `berth-runtime`, `berth-net` and `berth-shim`, where each `unsafe` block carries a short reason.
- Library crates never print and never read the environment directly.
- Text from containers, images and registries reaches the terminal only through an escaping helper.
- Paths go through `std::path`; portable crates pass their tests on Windows.
- Branches `feat/`, `fix/`, `docs/`, `ci/`, `chore/`; Conventional Commits; squash merges without trailers.
- CHANGELOG.md under Unreleased for anything users notice.
- No em-dashes; short comments that say why.

### 10.5 Contributors

After this design is approved, the repository gets: CONTRIBUTING.md (with DCO), CODE_OF_CONDUCT.md, SECURITY.md with private vulnerability reporting, issue templates for bugs and features with `config.yml`, a pull request template, CODEOWNERS, labels (`good first issue`, `help wanted`, `area/*`, `priority/*`, `kind/*`), one milestone per phase, a GitHub Project with one issue per plan task, a ruleset on `main` (pull requests required, required CI checks, no force push or deletion, conversations resolved, linear history), Dependabot, approval required for workflows from fork pull requests, and Discussions.

## 11. Reuse by the Kubernetes and registry projects

- **Through containerd or CRI-O, from phase 1.** `berth runtime` is runc-compatible, so the Kubernetes project can run pods on berth's runtime with containerd or CRI-O as the CRI layer before berth has its own CRI server.
- **As libraries.** `berth-runtime` (container lifecycle behind a trait), `berth-image` (pull, store, snapshot) and `berth-net` (bridge and nftables) are published crates. A Rust kubelet alternative can embed them directly instead of talking to a daemon.
- **Through CRI, from phase 9.** `berth-cri` implements RuntimeService and ImageService on the same daemon, validated with `critest`. A pod maps to a berth pod (shared namespaces, section 7.1).
- **Registry.** berth's push and pull are tested against the sibling registry project in the `compat` job, and berth's recipes and compose artifacts are stored as OCI artifacts any registry can hold.
- **Cluster management.** The berth API exposes nodes, services, events and stats so a Rancher-style manager can drive berth clusters without screen-scraping the CLI.

## 12. Open questions

- The Go template implementation for `--format`: adopt an existing crate or write one. Decided in the phase 1 plan after testing candidates against docker/cli's format tests.
- The Windows relay transport after phase 1: Hyper-V sockets or the TCP relay. Decided by measuring both in phase 2.
- The Raft library for phase 8 (openraft is the first candidate). Decided in the phase 8 spec.
- Pass-rate thresholds for v1.0. Set in the phase 5 spec from the measured rates at the end of phase 4.
