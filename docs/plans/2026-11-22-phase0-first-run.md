# Phase 0 and first run: implementation plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up the workspace, CI and WSL2 development, then make `berth run --rm <image> <cmd>` pull an OCI image from a registry, unpack it onto overlayfs, run it through libcontainer under a berth shim, stream its output and return its exit code.

**Architecture:** A Cargo workspace of focused crates (design section 4.2). This plan builds `berth-core`, `berth-image`, `berth-runtime`, `berth-shim`, a first `berth-daemon::Engine` and the `berth` binary. The CLI calls `Engine` in-process for now. The next plan puts the Engine API between them without changing `Engine`'s interface. Tests run offline against an in-process fake registry and a static fixture binary.

**Tech Stack:** Rust stable, edition 2024; tokio; libcontainer and libcgroups (youki); oci-spec; oci-client; tar, flate2, zstd; rustix; redb; serde and serde_json; clap; axum (test registry); cargo-nextest; just.

**Spec:** `docs/design.md`

**Where this sits in phase 1.** Phase 1 (v0.1) is split into plans, each ending in working software:

1. This plan: foundations and the first `run`.
2. Daemon and Engine API subset (`/_ping`, `/version`, `/info`, images, containers create/start/wait/logs/attach/kill/rm, exec) so the real `docker` CLI drives berthd; shim recovery after a daemon restart.
3. Networking: default bridge, `-p`, host and none modes.
4. Volumes, `login`/`logout` and credential helpers, the rest of the phase 1 commands, `berth runtime` (runc-compatible).
5. Windows: `berth machine` with the managed WSL2 distro, named pipes, path translation; first release v0.1.

**Before Task 1.** Repository setup from design section 10.5 (ruleset, labels, milestones, Project, templates) is done separately and needs the user's go-ahead. Each task below lands as its own branch and pull request with labels, milestone `Phase 0` or `Phase 1` and an assignee set.

## Global Constraints

- Rust stable, edition 2024. `rust-version` in the workspace is current stable minus two, checked with `rustup check` at Task 1 (the local 1.94 toolchain is out of date; run `rustup update stable`).
- Exact pins: `libcontainer = "=0.7.0"`, `libcgroups = "=0.7.0"`, `oci-spec = "=0.10.0"` (the version libcontainer re-exports), `oci-client = "0.18"`, `tar = "0.4.46"`, `tokio = "1.54.1"` (1.54.0 is yanked). Other crates: latest release, locked in `Cargo.lock`.
- Never depend on `tokio-tar` or `astral-tokio-tar`.
- `#![forbid(unsafe_code)]` in every crate except `berth-runtime`, `berth-shim` and `berth-net`; each `unsafe` block there has a one-line reason.
- Library crates never print and never read environment variables; the `berth` binary reads env and passes values in.
- No network in tests. No bare `std::thread::sleep` or `tokio::time::sleep` in tests; wait on a condition with a deadline.
- Tests that need root are `#[ignore = "privileged"]` and run with `just test-linux` (WSL2) or the CI `integration` job.
- Crates and modules that use Linux APIs are behind `#[cfg(target_os = "linux")]`; everything else passes `cargo test` on Windows and macOS.
- Supported container platforms in this plan: `linux/amd64`, `linux/arm64`.
- State lives under `berth_core::Paths`; rootful defaults are `/var/lib/berth` and `/run/berth`.
- Conventional Commits, branches `feat/<topic>`, `ci/<topic>`, `docs/<topic>`, squash merges without trailers, `Signed-off-by` on every commit (`git commit -s`). No em-dashes. Short comments that say why. Never mention AI or code generation anywhere.
- CHANGELOG.md gets an Unreleased entry for anything a user can see.

## Review Focus

1. **Invalid or uppercase image references** (`Nginx`, `nginx:`, empty). Expect the Docker error text `invalid reference format` (plus `: repository name must be lowercase` for uppercase) and exit code 125, not a panic or a registry request. Test in Task 2.
2. **An interrupted pull** (process killed mid-blob) leaves a temp file. Expect the next pull to ignore it, overwrite it and succeed, and a blob only ever appears at its final path after its digest matched. Test in Task 3.
3. **An image with no matching platform** (arm64-only index on amd64). Expect an error that names the requested platform and lists the available ones. Test in Task 4.
4. **A command that does not exist in the image** (`berth run img nope`). Expect `exec: "nope": executable file not found in $PATH` on stderr, exit code 127, and with `--rm` the snapshot and container directory removed. Test in Task 10.
5. **Ctrl-C during `berth run`**. Expect SIGINT forwarded to the container, berth exiting with the container's code (130 when it dies from the signal), and `--rm` cleanup done. Test in Task 10.

---

### Task 1: Workspace, CI and WSL2 setup

**Files:**
- Create: `Cargo.toml` (workspace, `[workspace.package]` with edition, rust-version, license `Apache-2.0`, repository), `rust-toolchain.toml` (channel `stable`, components `rustfmt`, `clippy`), `deny.toml`, `justfile`, `.gitattributes` (`* text=auto eol=lf`), `.config/nextest.toml`
- Create: `crates/berth/Cargo.toml`, `crates/berth/src/main.rs`, `crates/berth/tests/cli.rs`
- Create: `.github/workflows/ci.yml`, `scripts/wsl-setup.sh`
- Create: `AGENTS.md`, `CLAUDE.md` (one line: `@AGENTS.md`), `CHANGELOG.md`
- Modify: `README.md` (add a "Developing" section pointing at AGENTS.md)

**Interfaces:**
- Produces: `just check` (fmt, clippy, deny, tests), `just test-linux` (privileged tests inside WSL2 or on Linux), a `berth` binary whose `--version` prints `berth <version>`.

- [ ] **Step 1: Write the failing test** in `crates/berth/tests/cli.rs` with `assert_cmd`:

```rust
#[test]
fn version_prints_name_and_version() {
    Command::cargo_bin("berth").unwrap()
        .arg("--version")
        .assert()
        .success()
        .stdout(format!("berth {}\n", env!("CARGO_PKG_VERSION")));
}
```

- [ ] **Step 2: Run it.** `cargo test -p berth --test cli`. Expected: FAIL (no binary target).
- [ ] **Step 3: Implement** a clap `Cli` with `#[command(name = "berth", version)]` and no subcommands yet.
- [ ] **Step 4: Run it.** Expected: PASS.
- [ ] **Step 5: Add tooling.**
  - `deny.toml`: allow MIT, Apache-2.0, Apache-2.0 WITH LLVM-exception, BSD-2-Clause, BSD-3-Clause, ISC, Unicode-3.0, Zlib; deny advisories and yanked crates.
  - `justfile` recipes:
    - `check`: `cargo fmt --check && cargo clippy --workspace --all-targets -- -D warnings && cargo deny check && cargo nextest run --workspace`
    - `test-linux`: `cargo nextest run --workspace --run-ignored all` with runner `sudo -E`, `CARGO_TARGET_DIR` defaulting to `$HOME/.cache/berth-target`
    - `fixtures`: placeholder until Task 4
- [ ] **Step 6: CI.** `ci.yml` on `pull_request` and `push` to main, runner labels pinned.
  - `lint`: ubuntu-24.04
  - `test`: matrix of ubuntu-24.04, ubuntu-24.04-arm, windows-latest and macos-latest running `cargo nextest run --workspace`
  - `dco`: checks that every commit in the pull request has `Signed-off-by:` matching its author
  - Cache cargo with `Swatinem/rust-cache` pinned to a commit SHA
- [ ] **Step 7: `scripts/wsl-setup.sh`.** It runs inside a fresh Ubuntu 24.04 WSL distro (only `docker-desktop` exists on the dev machine today; create one with `wsl --install -d Ubuntu-24.04 --name berth-dev`).
  - Installs build-essential, pkg-config, libseccomp-dev, musl-tools, nftables, uidmap and passt, then rustup, cargo-nextest and just.
  - Writes `systemd=true` to `/etc/wsl.conf` if it is missing.
  - Checks that `stat -fc %T /sys/fs/cgroup` prints `cgroup2fs`. If it does not, prints the `.wslconfig` fix from design section 10.1 and exits 1.
  - Checks that `overlay` appears in `/proc/filesystems`.
  - Refuses to run when the repo or `CARGO_TARGET_DIR` resolves under `/mnt/`.
- [ ] **Step 8: AGENTS.md.** Model it on bridge's: project summary, commands (`just check`, `just test-linux`, `just fixtures`), layout (fill in as crates land), and the rules from Global Constraints and design section 10.4.
- [ ] **Step 9: Verify.** `just check` passes on Windows (portable parts) and in WSL. Push the branch; all CI jobs green.
- [ ] **Step 10: Commit.** `git add` each file by name; `git commit -s -m "chore: set up workspace, CI and WSL2 dev"`.

### Task 2: berth-core types

**Files:**
- Create: `crates/berth-core/Cargo.toml`, `src/lib.rs`, `src/reference.rs`, `src/digest.rs`, `src/id.rs`, `src/paths.rs`, `src/platform.rs`

**Interfaces:**
- Produces:
  - `ImageRef { registry: String, repository: String, tag: Option<String>, digest: Option<Digest> }` with `ImageRef::parse(&str) -> Result<ImageRef, RefError>`, `impl Display` (normalized form), `fn tag_or_latest(&self) -> &str`
  - `RefError` with `Display` equal to Docker's message
  - `Digest` (`sha256` only for now) with `Digest::parse(&str) -> Result<Digest, DigestError>`, `Digest::sha256_of(&[u8]) -> Digest`, `fn hex(&self) -> &str`, `impl Display` as `sha256:<hex>`; serde as that string
  - `ContainerId` with `ContainerId::generate() -> ContainerId` (64 lowercase hex), `fn short(&self) -> &str` (first 12), `ContainerId::parse(&str)`
  - `Paths { root: PathBuf, run: PathBuf }` with `Paths::rootful()`, `Paths::under(base: &Path)`, and `content_dir()`, `layers_dir()`, `snapshots_dir()`, `containers_dir()`, `container_dir(&ContainerId)`, `state_db()`, `shim_socket(&ContainerId)`
  - `Platform { os: String, arch: String, variant: Option<String> }` with `Platform::host() -> Platform` (maps `x86_64` to `amd64` and `aarch64` to `arm64` with variant `v8`) and `impl Display` (`linux/amd64`)

- [ ] **Step 1: Write the failing tests.** In `reference.rs`, a table test `parses_and_normalizes`:

| input | Display |
|---|---|
| `nginx` | `docker.io/library/nginx:latest` |
| `nginx:1.27` | `docker.io/library/nginx:1.27` |
| `user/app` | `docker.io/user/app:latest` |
| `docker.io/nginx` | `docker.io/library/nginx:latest` |
| `index.docker.io/library/nginx` | `docker.io/library/nginx:latest` |
| `ghcr.io/o/r:v1` | `ghcr.io/o/r:v1` |
| `localhost:5000/app` | `localhost:5000/app:latest` |
| `127.0.0.1:5000/a/b:t` | `127.0.0.1:5000/a/b:t` |
| `nginx@sha256:<64 x 'a'>` | `docker.io/library/nginx@sha256:<64 x 'a'>` |
| `nginx:1.27@sha256:<64 x 'a'>` | `docker.io/library/nginx:1.27@sha256:<64 x 'a'>` |

  A second table test, `rejects`, for Review Focus 1:

| input | error Display |
|---|---|
| `Nginx` | `invalid reference format: repository name must be lowercase` |
| `nginx:` | `invalid reference format` |
| `` (empty) | `invalid reference format` |
| `nginx@sha256:short` | `invalid reference format` |

  The rule for the registry: the first path component is a registry when it contains `.` or `:` or equals `localhost`; otherwise the registry is `docker.io`, and a single-component repository gets `library/`.

  Further tests:
  - `digest.rs`: `sha256_of(b"")` displays as `sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`. Parsing a digest with uppercase hex or a wrong length fails.
  - `id.rs`: `generate()` is 64 characters of `[0-9a-f]` and two calls differ. `short()` is 12 characters.
  - `paths.rs`: `Paths::under(tmp)` puts `state_db()` at `tmp/lib/state.redb` and `shim_socket(id)` at `tmp/run/shims/<id>.sock`. `Paths::rootful()` puts `state_db()` at `/var/lib/berth/state.redb`.
- [ ] **Step 2: Run.** `cargo nextest run -p berth-core`. Expected: FAIL.
- [ ] **Step 3: Implement** the types with the signatures above. Hand-written parser, no regex crate. Use `sha2` for hashing and `getrandom` for ids.
- [ ] **Step 4: Run.** Expected: PASS on all four CI systems.
- [ ] **Step 5: Commit.** `feat(core): add image references, digests, ids, paths and platforms`

### Task 3: Content store and image database

**Files:**
- Create: `crates/berth-image/Cargo.toml`, `src/lib.rs`, `src/store.rs`, `src/db.rs`

**Interfaces:**
- Consumes: `Digest`, `Paths`, `ImageRef` from Task 2.
- Produces:
  - `ContentStore::open(dir: &Path) -> io::Result<ContentStore>`
  - `fn put(&self, expected: &Digest, reader: impl Read) -> Result<PathBuf, StoreError>`
  - `fn path(&self, d: &Digest) -> Option<PathBuf>`
  - `fn has(&self, d: &Digest) -> bool`
  - `StoreError::{DigestMismatch { expected: Digest, actual: Digest }, Io(io::Error)}`
  - `ImageDb::open(path: &Path) -> Result<ImageDb, DbError>`
  - `fn put(&self, rec: &ImageRecord)`, `fn get(&self, r: &ImageRef) -> Result<Option<ImageRecord>, DbError>`, `fn list(&self) -> Result<Vec<ImageRecord>, DbError>`, `fn remove(&self, r: &ImageRef)`
  - `ImageRecord { reference: String, manifest: Digest, config: Digest, layers: Vec<Layer>, platform: Platform, size: u64, pulled_at: i64 }` and `Layer { digest: Digest, media_type: String, size: u64 }`, serialized as JSON values in a redb table `images` keyed by the normalized reference

- [ ] **Step 1: Write the failing tests** in `store.rs`:
  - `put_then_path_returns_file_with_contents`
  - `put_rejects_digest_mismatch_and_leaves_no_file`: put `b"abc"` under the digest of `b"abd"`. Expect `DigestMismatch`, `has()` false, and nothing left in the store directory.
  - `put_is_idempotent`
  - `stale_temp_file_is_ignored_and_overwritten` (Review Focus 2): create `<dir>/tmp/<hex>.partial` with junk, then put the real bytes. Expect success with the correct contents.
  - `blob_appears_only_after_verification`: give `put` a reader that fails halfway. Expect `path()` to be `None` afterwards.

  In `db.rs`, `round_trips_records` and `list_is_sorted_by_reference`.
- [ ] **Step 2: Run.** `cargo nextest run -p berth-image`. Expected: FAIL.
- [ ] **Step 3: Implement.**
  - `put` streams into `tmp/<hex>.partial` (created with truncate) while hashing, fsyncs, checks the digest, then renames to `blobs/sha256/<hex>` and fsyncs the directory (skip the directory fsync on Windows).
  - `ImageDb` wraps one `redb::Database` with a single table.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit.** `feat(image): add content store and image database`

### Task 4: Test kit, fake registry and puller

**Files:**
- Create: `crates/berth-fixture/` (a binary crate, no dependencies beyond std), `crates/berth-testkit/Cargo.toml`, `src/lib.rs`, `src/image.rs`, `src/registry.rs`
- Create: `crates/berth-image/src/pull.rs`
- Modify: `justfile` (`fixtures` builds `berth-fixture` for `x86_64-unknown-linux-musl` or `aarch64-unknown-linux-musl` to match the host, release mode), `.github/workflows/ci.yml` (build fixtures before tests on Linux)

**Interfaces:**
- Consumes: `ContentStore`, `ImageDb`, `ImageRecord` from Task 3; `ImageRef`, `Platform` from Task 2.
- Produces:
  - `berth-fixture` reads its own argv. `echo <words...>` prints the words joined by spaces plus a newline on stdout. `stderr <words...>` prints them on stderr. `exit <n>` exits with code n. `env` prints the environment sorted. `wait-signal` prints `ready`, blocks until SIGINT or SIGTERM, prints `got <NAME>` and exits 0.
  - `testkit::fixture_bin() -> PathBuf`: reads `BERTH_FIXTURE_BIN`, or the default `just fixtures` output path, and panics with "run `just fixtures`" when the binary is missing.
  - `testkit::ImageBuilder::new().layer(entries: Vec<Entry>).entrypoint(&[&str]).build(platform) -> TestImage { index_json, manifest_json, config_json, blobs: HashMap<Digest, Vec<u8>> }`
  - `Entry::{File { path, mode, bytes }, Dir { path }, Symlink { path, target }, Whiteout { path }, Opaque { dir }}`
  - `testkit::fixture_image() -> TestImage`. Layer 1 has `/bin/fixture` (the fixture binary) and `/etc/gone`. Layer 2 whites out `/etc/gone`. The entrypoint is `["/bin/fixture"]`.
  - `FakeRegistry::start(images: Vec<(String, TestImage)>) -> FakeRegistry` with `fn addr(&self) -> SocketAddr`. It serves the OCI distribution GET and HEAD routes for manifests and blobs over plain HTTP, optionally behind a bearer challenge (`FakeRegistry::with_token_auth`), and counts requests per path.
  - `Puller::new(store: ContentStore, db: ImageDb, cfg: PullConfig)`
  - `PullConfig { insecure_registries: Vec<String>, platform: Platform, max_concurrent: usize }`, default `max_concurrent` 3
  - `async fn pull(&self, r: &ImageRef, on_event: impl FnMut(PullEvent)) -> Result<ImageRecord, PullError>`
  - `PullEvent::{Resolving, Layer { digest, done: u64, total: u64 }, Done}`
  - `PullError::{NoMatchingPlatform { wanted: Platform, available: Vec<Platform> }, NotFound(String), Unauthorized, Registry(String), Store(StoreError)}`

- [ ] **Step 1: Write the failing tests** in `pull.rs`. Each test starts a `FakeRegistry` with `fixture_image()` under `test/fixture:1`, with `insecure_registries` set to the registry address.
  - `pulls_index_selects_platform_and_stores_blobs`: the record has 2 layers, every layer and the config are `has()` in the store, and `db.get()` returns the record.
  - `second_pull_downloads_nothing`: the registry's blob request count is unchanged after a second pull.
  - `token_auth_flow`: with `with_token_auth`, the pull succeeds anonymously through the challenge.
  - `no_matching_platform_lists_available` (Review Focus 3): serve an index holding only `linux/arm64` and request `linux/amd64`. Expect the error Display `no matching manifest for linux/amd64 in the manifest list entries (available: linux/arm64)`.
  - `missing_tag_is_not_found`.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement.**
  - `Puller` drives `oci_client::Client` with `ClientProtocol::HttpsExcept(insecure_registries)` and a `platform_resolver` that matches `PullConfig::platform`, keeping the list of available platforms for the error.
  - Blobs stream into `ContentStore::put` with at most `max_concurrent` in flight.
  - Docker Hub references map to the registry host `registry-1.docker.io`.
- [ ] **Step 4: Run.** Expected: PASS on all systems. The registry tests need no root.
- [ ] **Step 5: Commit.** `feat(image): pull images with platform selection and a fake registry for tests`

### Task 5: Layer unpacking

**Files:**
- Create: `crates/berth-image/src/unpack.rs`, `crates/berth-image/src/unpack/fs_linux.rs`, `fuzz/fuzz_targets/unpack.rs`, `fuzz/Cargo.toml`

**Interfaces:**
- Consumes: `testkit::ImageBuilder` and `Entry` from Task 4.
- Produces:
  - `RelPath`: a validated, normalized relative path with no `..` after normalization, no absolute paths and no NUL. `RelPath::new(&str) -> Result<RelPath, UnpackError>`
  - `trait UnpackTarget { fn dir(&mut self, p: &RelPath, m: &Meta) -> Result<()>; fn file(&mut self, p: &RelPath, m: &Meta, r: &mut dyn Read) -> Result<()>; fn symlink(&mut self, p: &RelPath, target: &str, m: &Meta) -> Result<()>; fn hardlink(&mut self, p: &RelPath, to: &RelPath) -> Result<()>; fn whiteout(&mut self, p: &RelPath) -> Result<()>; fn opaque(&mut self, dir: &RelPath) -> Result<()>; }`
  - `Meta { mode: u32, uid: u32, gid: u32, mtime: i64, xattrs: Vec<(String, Vec<u8>)> }`
  - `fn unpack_layer(media_type: &str, reader: impl Read, target: &mut impl UnpackTarget) -> Result<(), UnpackError>`. It handles `application/vnd.oci.image.layer.v1.tar`, `+gzip`, `+zstd`, and the Docker `rootfs.diff.tar.gzip` type.
  - `RecordingTarget` (pub, for tests) collects `Vec<Op>`
  - `#[cfg(target_os = "linux")] FsTarget::new(layer_dir: &Path) -> Result<FsTarget>` writes inside `layer_dir` and never outside it

- [ ] **Step 1: Write the failing tests** with `RecordingTarget`. These are portable.
  - `whiteout_file_becomes_whiteout_op`: entry `etc/.wh.gone` becomes `Op::Whiteout("etc/gone")`.
  - `opaque_marker_becomes_opaque_op`: `var/.wh..wh..opq` becomes `Op::Opaque("var")`.
  - `dotdot_entry_is_rejected`: `../evil` gives `UnpackError::PathEscape`.
  - `absolute_entry_is_normalized_relative`: `/bin/x` becomes `bin/x`, as Docker does.
  - `hardlink_outside_layer_is_rejected`.
  - `gzip_and_zstd_unpack_to_same_ops`.

  Privileged tests in `fs_linux.rs`:
  - `whiteout_is_char_device_0_0`
  - `opaque_sets_trusted_overlay_opaque_xattr`
  - `write_through_symlink_cannot_escape`: layer 1 is the symlink `a -> /tmp/outside-<random>` followed by the file `a/pwned`. Expect `PathEscape`, and nothing created in the outside dir.
  - `ownership_and_mode_preserved`
- [ ] **Step 2: Run.** `cargo nextest run -p berth-image unpack` fails, and so does `just test-linux`.
- [ ] **Step 3: Implement.**
  - `unpack_layer` maps tar entries to target calls and decompresses with flate2 and zstd.
  - `FsTarget` opens every path with `rustix::fs::openat2` and `RESOLVE_IN_ROOT | RESOLVE_NO_MAGICLINKS` relative to the layer dir fd.
  - Whiteouts use `mknodat` with char 0:0. Opaque directories set the `trusted.overlay.opaque` xattr to `y`.
- [ ] **Step 4: Run.** Expected: PASS, both portable and privileged.
- [ ] **Step 5: Fuzz target.** `unpack` feeds arbitrary bytes into `unpack_layer` with a `RecordingTarget` and asserts no panic and no op path containing `..`. Run `cargo +nightly fuzz run unpack -- -max_total_time=60` once locally. CI does not run it yet.
- [ ] **Step 6: Commit.** `feat(image): unpack layers safely with whiteout support`

### Task 6: Overlay snapshotter

**Files:**
- Create: `crates/berth-image/src/snapshot.rs` (Linux only)
- Modify: `.github/workflows/ci.yml`: add the `integration` job on ubuntu-24.04 and ubuntu-24.04-arm. It runs `just fixtures`, then `cargo nextest run --workspace --run-ignored only` with `sudo -E`.

**Interfaces:**
- Consumes: `ContentStore`, `ImageRecord`, `Layer`, `unpack_layer`, `FsTarget`, `Paths`, `ContainerId`.
- Produces:
  - `Snapshotter::new(paths: &Paths, store: ContentStore) -> Snapshotter`
  - `fn ensure_layer(&self, layer: &Layer) -> Result<PathBuf, SnapshotError>`: unpacks once into `layers_dir()/<hex>`, writing to `<hex>.tmp` and renaming when complete
  - `fn prepare(&self, id: &ContainerId, layers: &[Layer]) -> Result<Rootfs, SnapshotError>`: mounts overlay at `snapshots_dir()/<id>/merged` with the lowerdirs from the top layer down and with upper and work dirs
  - `fn remove(&self, id: &ContainerId) -> Result<(), SnapshotError>`: unmounts if mounted and deletes the directory, and succeeds when nothing is there
  - `Rootfs { path: PathBuf }`

- [ ] **Step 1: Write the failing privileged tests.**
  - `merged_view_hides_whiteouts`: `fixture_image()` gives a merged dir where `bin/fixture` exists and `etc/gone` does not.
  - `layer_unpacked_once_for_two_containers`: the mtime of the layer dir is unchanged by the second prepare.
  - `remove_unmounts_and_deletes`
  - `remove_is_idempotent`
  - `half_unpacked_layer_is_redone`: leave a `<hex>.tmp` dir behind. Expect `ensure_layer` to delete it and unpack again.
- [ ] **Step 2: Run** `just test-linux`. Expected: FAIL.
- [ ] **Step 3: Implement** the mount with the new mount API (`fsopen("overlay")`, `fsconfig` for `lowerdir+` per layer, then `upperdir` and `workdir`, then `fsmount` and `move_mount`) through rustix, so there is no length limit on lowerdir strings.
- [ ] **Step 4: Run.** Expected: PASS locally and in the CI `integration` job.
- [ ] **Step 5: Commit.** `feat(image): add overlay snapshotter and privileged CI job`

### Task 7: OCI spec builder with Docker defaults

**Files:**
- Create: `crates/berth-runtime/Cargo.toml`, `src/lib.rs`, `src/spec.rs`, `src/defaults.rs`, `src/seccomp.rs`, `profiles/seccomp-default.json`

**Interfaces:**
- Consumes: `ContainerId`, `Platform`.
- Produces:
  - `ImageConfig { entrypoint: Vec<String>, cmd: Vec<String>, env: Vec<String>, working_dir: Option<String>, user: Option<String> }` with `ImageConfig::from_oci(&oci_spec::image::ImageConfiguration)`
  - `RunConfig { entrypoint: Option<Vec<String>>, cmd: Vec<String>, env: Vec<String>, working_dir: Option<String>, user: Option<String>, hostname: String, tty: bool }`
  - `fn build_spec(image: &ImageConfig, run: &RunConfig, rootfs: &Path, platform: &Platform) -> Result<oci_spec::runtime::Spec, SpecError>`
  - `fn default_seccomp(arch: &str) -> oci_spec::runtime::LinuxSeccomp`

- [ ] **Step 1: Write the failing tests.** These are portable.
  - `args_follow_docker_rules`, a table:

| image entrypoint | image cmd | run entrypoint | run cmd | args |
|---|---|---|---|---|
| `[/bin/fixture]` | `[]` | none | `[echo, hi]` | `[/bin/fixture, echo, hi]` |
| `[]` | `[sh]` | none | `[]` | `[sh]` |
| `[]` | `[sh]` | none | `[ls]` | `[ls]` |
| `[/e]` | `[c]` | `[/x]` | `[]` | `[/x]` |
| `[/e]` | `[c]` | `[]` (empty string given) | `[ls]` | `[ls]` |

  - `env_merges_image_then_run_and_adds_path`: no `PATH` in the image gives `PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`, and run env overrides image env for the same key. `HOSTNAME=<hostname>` is set.
  - `default_capabilities_match_docker`: the bounding, effective and permitted sets equal `AUDIT_WRITE CHOWN DAC_OVERRIDE FOWNER FSETID KILL MKNOD NET_BIND_SERVICE NET_RAW SETFCAP SETGID SETPCAP SETUID SYS_CHROOT`.
  - `masked_and_readonly_paths_match_docker`: equal to the lists in moby v29.10.0 `oci/defaults.go`. Copy them from that file at implementation time and cite the tag in a comment.
  - `mounts_match_docker`:
    - `/proc` (proc)
    - `/dev` (tmpfs, `nosuid,strictatime,mode=755,size=65536k`)
    - `/dev/pts`
    - `/dev/shm` (tmpfs, `size=64m`)
    - `/dev/mqueue`
    - `/sys` (sysfs, ro)
    - `/sys/fs/cgroup` (cgroup2, ro)
  - Namespaces: pid, ipc, uts, mount, cgroup and network. For network, pass the namespace path in; this plan uses a new empty one. No user namespace.
  - `seccomp_default_is_moby_profile`: `profiles/seccomp-default.json` is moby v29.10.0 `profiles/seccomp/default.json`, vendored with its Apache-2.0 notice. `default_seccomp("amd64")` has default action `SCMP_ACT_ERRNO` with errno `EPERM` and allows `read`. It denies `kexec_load` (not in the allow list). Rules that are conditional on capabilities apply for the default set.
- [ ] **Step 2: Run** `cargo nextest run -p berth-runtime`. Expected: FAIL.
- [ ] **Step 3: Implement** with oci-spec builders. `seccomp.rs` converts moby's profile format (includes and excludes by arch and caps) into `LinuxSeccomp` for one arch and cap set.
- [ ] **Step 4: Run.** Expected: PASS on all systems.
- [ ] **Step 5: Commit.** `feat(runtime): build OCI specs with Docker's defaults`

### Task 8: Runtime trait and libcontainer adapter

**Files:**
- Create: `crates/berth-runtime/src/runtime.rs`, `src/libcontainer.rs` (Linux only), `src/fake.rs`

**Interfaces:**
- Consumes: `build_spec`, `ContainerId`.
- Produces:
  - `#[cfg(unix)] pub struct Stdio { pub stdin: Option<OwnedFd>, pub stdout: OwnedFd, pub stderr: OwnedFd }`
  - `pub trait Runtime: Send + Sync { fn create(&self, id: &ContainerId, bundle: &Path, stdio: Stdio) -> Result<u32, RuntimeError>; fn start(&self, id: &ContainerId) -> Result<(), RuntimeError>; fn kill(&self, id: &ContainerId, signal: i32) -> Result<(), RuntimeError>; fn delete(&self, id: &ContainerId, force: bool) -> Result<(), RuntimeError>; fn state(&self, id: &ContainerId) -> Result<ContainerState, RuntimeError>; }`. `create` returns the init pid.
  - `ContainerState::{Created, Running, Stopped, Paused}`
  - `RuntimeError::{NotFound, CommandNotFound(String), NotExecutable(String), Other(String)}`
  - `LibcontainerRuntime::new(state_root: PathBuf)`: the youki state lives in `state_root`
  - `FakeRuntime`: records calls, and lets a test script exit codes for later tasks

- [ ] **Step 1: Write the failing privileged tests** with a bundle built from `Snapshotter::prepare` on `fixture_image()` and `build_spec`.
  - `runs_fixture_echo`: stdout pipe reads `hi\n`, and the process exits 0, observed with `waitpid` on the returned pid.
  - `kill_sigterm_stops_wait_signal`
  - `delete_removes_state`
  - `command_not_found_maps_to_error`: cmd `nope` gives `RuntimeError::CommandNotFound("nope")`.
  - `cgroup_v2_created_under_berth_slice`: `/sys/fs/cgroup/berth/<id>` exists while the container runs. Without systemd this is the cgroupfs path `berth/<id>`.
- [ ] **Step 2: Run** `just test-linux`. Expected: FAIL.
- [ ] **Step 3: Implement** with `ContainerBuilder::new(id, SyscallType::default()).with_root_path(state_root).with_stdout(..).with_stderr(..).as_init(bundle).with_systemd(false).build()`. `start` and `kill` use `Container::load`. Map libcontainer's "executable not found" error to `CommandNotFound`. Keep every libcontainer type inside this file.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit.** `feat(runtime): add Runtime trait with libcontainer adapter`

### Task 9: Shim

**Files:**
- Create: `crates/berth-shim/Cargo.toml`, `src/lib.rs`, `src/protocol.rs`, `src/logs.rs`, `src/server.rs` (Linux only), `src/client.rs` (Linux only)
- Modify: `crates/berth/src/main.rs` (hidden `shim` subcommand that calls `berth_shim::server::run`)

**Interfaces:**
- Consumes: `Runtime`, `LibcontainerRuntime`, `Paths`, `ContainerId`.
- Produces:
  - Framing: `fn write_frame<W: Write, T: Serialize>(w: &mut W, msg: &T) -> io::Result<()>` and `fn read_frame<R: Read, T: DeserializeOwned>(r: &mut R) -> io::Result<T>`. Each frame is a u32 big-endian length followed by JSON. Frames over 16 MiB are an `InvalidData` error.
  - Messages: `Request::{Start, Signal { signal: i32 }, Wait}` and `Response::{Started { pid: u32 }, Exited { code: i32 }, Ok, Error { kind: ErrorKind, message: String }}`
  - `LogWriter::create(path: &Path) -> io::Result<LogWriter>` with `fn write(&mut self, stream: Stream, chunk: &[u8], at: SystemTime)`. It writes one line per newline-terminated chunk in Docker's json-file format: `{"log":"hi\n","stream":"stdout","time":"2026-11-22T10:00:00.000000000Z"}`.
  - `LogReader::open(path) -> io::Result<LogReader>` with `fn follow(self) -> impl Iterator<Item = io::Result<LogLine>>`, which ends when the shim's `exit.json` appears and the file is drained
  - `LogLine { stream: Stream, bytes: Vec<u8>, time: String }`
  - `ShimClient::spawn(berth_exe: &Path, paths: &Paths, id: &ContainerId) -> Result<ShimClient, ShimError>` with `fn start(&mut self) -> Result<u32, ShimError>`, `fn signal(&mut self, sig: i32) -> Result<(), ShimError>` and `fn wait(&mut self) -> Result<i32, ShimError>`
  - The exit code is the process code, or 128 plus the signal number when a signal killed it. The shim writes it to `container_dir/exit.json` as `{"code":N}`.

- [ ] **Step 1: Write the failing tests.**
  - Portable: `frame_round_trip`, `oversized_frame_rejected`, `log_line_format_matches_docker` (fixed time `2026-11-22T10:00:00Z` gives the exact line above), `partial_line_buffered_until_newline`, `log_reader_follows_until_exit`.
  - Privileged:
    - `shim_runs_fixture_and_reports_exit_3`: cmd `exit 3` gives `wait() == 3`.
    - `shim_survives_client_drop`: drop the client, then reconnect to the socket. Expect `Wait` to still return the code.
    - `signal_death_is_128_plus`: `wait-signal` killed by SIGKILL gives 137.
    - `logs_written_for_stdout_and_stderr`
- [ ] **Step 2: Run** both test sets. Expected: FAIL.
- [ ] **Step 3: Implement.**
  - The shim is a synchronous process: it double-forks to detach, calls `setsid`, and listens on `Paths::shim_socket`.
  - It creates the container with pipes and pumps them into `LogWriter` on two threads.
  - It reaps with `waitpid`, writes `exit.json`, and serves `Wait` to any client until a `Delete` (added in the next plan) or until the daemon removes the socket.
  - No tokio in this crate.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Commit.** `feat(shim): add per-container shim with json-file logs`

### Task 10: Engine and `berth run`, `pull`, `images`

**Files:**
- Create: `crates/berth-daemon/Cargo.toml`, `src/lib.rs`, `src/engine.rs`, `src/config.rs`
- Create: `crates/berth/src/cmd/mod.rs`, `src/cmd/run.rs`, `src/cmd/pull.rs`, `src/cmd/images.rs`, `crates/berth/tests/run.rs`
- Modify: `crates/berth/src/main.rs`, `CHANGELOG.md`, `AGENTS.md` (layout)

**Interfaces:**
- Consumes: everything above.
- Produces (the next plan's API server calls these unchanged):
  - `Engine::open(cfg: EngineConfig) -> Result<Engine, EngineError>`
  - `EngineConfig { paths: Paths, insecure_registries: Vec<String>, berth_exe: PathBuf, platform: Platform }`
  - `async fn pull(&self, r: &ImageRef, on_event: impl FnMut(PullEvent)) -> Result<ImageRecord, EngineError>`
  - `fn images(&self) -> Result<Vec<ImageRecord>, EngineError>`
  - `async fn create(&self, req: CreateRequest) -> Result<ContainerId, EngineError>`, where `CreateRequest { image: ImageRef, run: RunConfig, auto_remove: bool }` and the hostname defaults to the short id
  - `async fn start(&self, id: &ContainerId) -> Result<(), EngineError>`
  - `async fn wait(&self, id: &ContainerId) -> Result<i32, EngineError>`
  - `async fn signal(&self, id: &ContainerId, sig: i32) -> Result<(), EngineError>`
  - `fn logs(&self, id: &ContainerId) -> Result<LogReader, EngineError>`
  - `async fn remove(&self, id: &ContainerId) -> Result<(), EngineError>`
  - `EngineError` carries an `exit_code()`: 125 for engine errors, 126 for `NotExecutable`, 127 for `CommandNotFound`
- CLI:
  - `berth run [--rm] [-e KEY=VALUE]... [--entrypoint CMD] [-w DIR] [-u USER] IMAGE [ARG...]`
  - `berth pull IMAGE`
  - `berth images`, with columns `REPOSITORY  TAG  IMAGE ID  CREATED  SIZE`. IMAGE ID is the first 12 hex of the config digest, CREATED is relative ("5 minutes ago"), SIZE is in decimal units like docker ("7.8MB").
  - `BERTH_ROOT` overrides `Paths` (tests use a temp dir). `BERTH_INSECURE_REGISTRIES` is a comma list.
  - Errors print as `berth: <message>` on stderr.
  - Running as non-root exits 125 with `berth: this release needs root; run it with sudo (rootless support is planned)`.

- [ ] **Step 1: Write the failing tests.**
  - Portable, in `engine.rs` with `FakeRuntime` and `FakeRegistry`: `create_pulls_missing_image` and `exit_codes_map_from_runtime_errors` (a table of `CommandNotFound` to 127, `NotExecutable` to 126 and `Other` to 125).
  - Privileged, in `crates/berth/tests/run.rs`. Each test starts a `FakeRegistry` and sets `BERTH_ROOT` and `BERTH_INSECURE_REGISTRIES`.
    - `run_echo_prints_and_exits_0`: `berth run --rm <reg>/test/fixture:1 echo hi` gives stdout `hi\n` and exit 0.
    - `run_exit_code_propagates`: `exit 3` gives exit 3.
    - `run_stderr_goes_to_stderr`
    - `run_env_flag`: `-e A=1 env` output contains `A=1`.
    - `command_not_found` (Review Focus 4): `--entrypoint nope` gives exit 127, stderr containing `exec: "nope": executable file not found in $PATH`, and with `--rm` no directory left under `containers/` or `snapshots/`.
    - `ctrl_c_forwards_and_cleans` (Review Focus 5): start `wait-signal` and read stdout until `ready` arrives, with a 10 second deadline. The test does not open `state.redb`, because redb locks the file to one process. Send SIGINT to the berth process. Expect stdout `ready\ngot SIGINT\n`, exit 0, and no leftover directories.
    - `rm_after_exit_cleans_snapshot`
    - `pull_then_images_lists_it`
    - `non_root_is_refused`: run as `nobody` with `sudo -u nobody`.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement.**
  - `Engine::create` resolves the image (pulling when it is missing), ensures the layers, prepares the rootfs, builds the spec and writes `config.json` into `container_dir`.
  - `start` spawns the shim.
  - `run.rs` composes create, start, logs follow and wait, forwards SIGINT and SIGTERM through `Engine::signal`, and calls `remove` on `--rm` along every exit path.
  - Container metadata goes into `state.redb`, table `containers`, keyed by id.
- [ ] **Step 4: Run.** Both sets pass, locally in WSL and in CI.
- [ ] **Step 5: Nightly job.** Add `.github/workflows/nightly.yml`. On ubuntu-24.04 it runs `sudo -E berth run --rm alpine echo hi` against Docker Hub and checks the output is `hi`.
- [ ] **Step 6: CHANGELOG.** Under Unreleased, add: "`berth run`, `berth pull` and `berth images` run OCI images from any registry on Linux (root only)."
- [ ] **Step 7: Commit.** `feat: run images with berth run, pull and images`

### Task 11: Benchmark baseline harness

**Files:**
- Create: `bench/berth-bench/Cargo.toml`, `src/main.rs`, `src/scenario.rs`, `src/report.rs`, `.github/workflows/bench.yml`

**Interfaces:**
- Consumes: the `berth` binary from Task 10.
- Produces:
  - `berth-bench --engines docker,podman,berth --runs 20 --out results.json`
  - Scenarios `run-true-warm` (image already pulled) and `run-true-cold` (image removed first) run `<engine> run --rm alpine true`. `daemon-idle-rss` sums the RSS of `dockerd` plus `containerd`, or of `berthd` when it exists (skipped for berth in this plan, since it has no daemon yet).
  - Output: `{"engine": "...", "scenario": "...", "runs": N, "median_ms": f64, "p95_ms": f64}` per pair, plus a markdown table on stdout.

- [ ] **Step 1: Write the failing tests** in `scenario.rs` and `report.rs`: `median_and_p95_of_known_samples` (samples 1 through 20 give a median of 10.5 and a p95 of 19.05 by linear interpolation), and `markdown_table_has_one_row_per_pair`.
- [ ] **Step 2: Run.** Expected: FAIL.
- [ ] **Step 3: Implement** with `std::process::Command` and wall-clock timing. An engine missing from PATH is skipped with a note in the report.
- [ ] **Step 4: Run.** Expected: PASS.
- [ ] **Step 5: Workflow.** `bench.yml` runs nightly and on manual dispatch on ubuntu-24.04: it installs podman from apt (docker is preinstalled), builds berth, runs the bench with sudo and uploads `results.json` as an artifact. This first run is the baseline that design section 8 asks for.
- [ ] **Step 6: Commit.** `feat(bench): add benchmark harness and nightly baseline`

---

## Self-review notes

- Spec coverage for this plan's slice:
  - Phase 0 (workspace, CI, WSL2, AGENTS.md, bench baseline) is Tasks 1 and 11.
  - The phase 1 pieces here are pull, overlay, libcontainer plus shim, and `run`.
  - Docker-default security is Task 7. Fuzzing unpack is Task 5.
  - Networking, volumes, the Engine API, `berth runtime` and Windows are in plans 2 to 5, listed at the top.
- The names `ContentStore`, `ImageDb`, `ImageRecord`, `Layer`, `Snapshotter`, `Rootfs`, `Runtime`, `RunConfig`, `ImageConfig`, `ShimClient`, `LogReader` and `Engine` match across tasks.
