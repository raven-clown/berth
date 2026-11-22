# berth

A container platform written in Rust that aims to replace Docker without changing how you work. The same `docker` commands, the same Engine API socket, your existing Dockerfiles, `docker-compose.yml` and Swarm stack files. On top of that, a short app file with smart defaults, so a web app with a database and a cache fits in a few lines instead of forty.

Free and open source under Apache-2.0, with no telemetry.

**Status: design done, implementation not started.** There is no code yet and nothing here works today. The design is in [docs/design.md](docs/design.md), the plans are in [docs/plans/](docs/plans/), and the work is tracked on the [project board](https://github.com/users/raven-clown/projects/7).

## Planned scope

- **Runtime:** OCI containers on Linux with namespaces, cgroups v2, seccomp and overlayfs, built on youki's `libcontainer`. Rootful first, rootless later.
- **Images:** pull and push with any OCI registry, multi-arch, save and load, import from an existing Docker install.
- **Build:** Dockerfiles as they are, with parallel steps and a content-addressed cache.
- **Networking and volumes:** bridge networks, port publishing, built-in DNS, named and bind volumes.
- **Apps:** run `compose.yaml` unchanged, or write a much shorter `berth.yml`:

  ```yaml
  web:
    build: .
    port: 3000
  db: postgres:17
  cache: redis
  ```

- **Drop-in compatibility:** a `docker`-compatible CLI and the Docker Engine API on the same socket, so docker compose, VS Code Dev Containers, Testcontainers and CI tools keep working.
- **Beyond Docker:** local HTTPS domains like `web.myapp.localhost`, file sync for development, zero-downtime deploys over SSH, a terminal dashboard, optional microVM isolation, and image signing and SBOMs.
- **Clusters:** small Swarm-style clusters that deploy stack files unchanged.
- **Everywhere:** Linux natively, Windows through a managed WSL2 distro, macOS through a lightweight VM.

None of this is built yet. The design document lays out the phases and what each one ships.

## Contributing

Issues and pull requests are welcome. Start with [CONTRIBUTING.md](CONTRIBUTING.md), pick an issue labelled `good first issue` or `help wanted`, and ask questions in [Discussions](https://github.com/raven-clown/berth/discussions). Report security problems privately as described in [SECURITY.md](SECURITY.md). Everyone taking part follows the [Code of Conduct](CODE_OF_CONDUCT.md).

## License

Apache-2.0. See [LICENSE](LICENSE).
