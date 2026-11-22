# Contributing to berth

Thanks for wanting to help. berth is early: the design is settled and the first plan is written, but there is no code yet. This is a good time to shape it.

## Where to start

- Read [docs/design.md](docs/design.md) for what berth is and how it fits together.
- Read the current plan in [docs/plans/](docs/plans/). Every task in it has an issue on the [project board](https://github.com/users/raven-clown/projects/7).
- Issues labelled [`good first issue`](https://github.com/raven-clown/berth/labels/good%20first%20issue) and [`help wanted`](https://github.com/raven-clown/berth/labels/help%20wanted) are open for anyone.
- Questions and ideas go to [Discussions](https://github.com/raven-clown/berth/discussions). Bugs and concrete features go to issues.

Comment on an issue before starting on it, so two people do not do the same work. For anything that changes the design, open a discussion or an issue first.

## Development setup

Container features need a Linux kernel.

- **Linux:** install Rust stable, `just` and `cargo-nextest`.
- **Windows 11:** use a dedicated WSL2 distro and run `scripts/wsl-setup.sh` inside it. Keep the build directory on the distro's own disk, not under `/mnt/`. Design section 10.1 explains why.
- **macOS:** portable crates build and test natively. Container tests need a Linux VM.

Once the workspace exists, the commands are:

```shell
just check        # fmt, clippy, cargo-deny, unit tests
just fixtures     # builds the static test fixture binary
just test-linux   # privileged tests, needs root on Linux or WSL2
```

`AGENTS.md`, added together with the workspace, will list the layout and every rule in one place.

## Rules for changes

- Tests come with every change and are written first. No network in tests. No bare sleeps; wait on a condition with a deadline.
- Tests that need root are marked `#[ignore = "privileged"]`.
- `unsafe` is only allowed in `berth-runtime`, `berth-shim` and `berth-net`, and each block needs a one-line reason.
- Library crates never print and never read environment variables.
- Code that is not Linux-specific must pass its tests on Windows and macOS.
- Keep comments short and about why. No em-dashes in code or docs.
- Add a line to `CHANGELOG.md` under Unreleased for anything users will notice.

## Branches, commits and pull requests

- Branch names: `feat/<topic>`, `fix/<topic>`, `docs/<topic>`, `ci/<topic>`, `chore/<topic>`.
- Commit messages follow [Conventional Commits](https://www.conventionalcommits.org/), for example `feat(image): resume interrupted blob downloads`.
- Keep pull requests focused on one thing. Link the issue with `Closes #123`.
- Pull requests are squash-merged once CI passes and review conversations are resolved.
- CI from first-time contributors runs after a maintainer approves it.

## Developer Certificate of Origin

Every commit must carry a `Signed-off-by` line that matches the commit author. The line certifies that you wrote the change, or have the right to submit it under the project's license, as described in the [Developer Certificate of Origin 1.1](https://developercertificate.org/).

Add it with:

```shell
git commit -s
```

To fix commits that are already on your branch:

```shell
git rebase --signoff main
git push --force-with-lease
```

## License

By contributing, you agree that your contributions are licensed under the [Apache License 2.0](LICENSE).
