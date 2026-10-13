# github-actions-benchmarks

Real-world CI builds, timed phase by phase, on next-generation
[StarSling Runners](https://docs.starsling.dev) against GitHub-hosted runners at 2, 4 and 8 vCPU.

Next-generation StarSling Runners keep the `runs-on` label you already use and add two things behind
it:

- **[Sandbox Router](https://docs.starsling.dev/runners/sandbox-router)** picks the sandbox
  provider behind each label, fails over to the next healthy provider when one rejects a job, and
  optimizes wall-clock time (queue plus execution) from the telemetry of completed jobs.
- **[Afterburner](https://docs.starsling.dev/runners/afterburner)** resizes each job's sandbox every
  second while it runs, adding up to 25% more CPU and 50% more RAM to steps under pressure at no
  extra cost.

These workflows measure what that means for two real builds, on the same commits, with the same
commands and cold caches on every runner.

## Benchmarks

| Workflow | Project | Measured phases |
|---|---|---|
| [`typescript-mastra.yml`](.github/workflows/typescript-mastra.yml) | [mastra-ai/mastra](https://github.com/mastra-ai/mastra) @ `b6eeb9d` | checkout, cold install (`pnpm install --frozen-lockfile`), lint (`pnpm lint:format`), build (`pnpm build:core`) |
| [`rust-spotify-player.yml`](.github/workflows/rust-spotify-player.yml) | [aome510/spotify-player](https://github.com/aome510/spotify-player) @ `7dc9d17` | checkout, system packages, toolchain, `cargo fmt`, `cargo test`, `cargo clippy`, `cargo clippy` (no features) |

The mastra phases follow the realworld mastra profile of
[hpc-sandbox-benchmarks](https://github.com/starslingdev/hpc-sandbox-benchmarks); the Rust phases
follow the rust suite of
[runs-on-demo/benchmark-public](https://github.com/runs-on-demo/benchmark-public).

## Runners

Each workflow runs one job per cell:

| vCPU | StarSling | GitHub |
|---|---|---|
| 2 | `starsling-ubuntu-26.04-2` | `ubuntu-26.04-2` (larger runner) |
| 4 | `starsling-ubuntu-26.04` | `ubuntu-26.04` (standard runner) |
| 8 | `starsling-ubuntu-26.04-8` | `ubuntu-26.04-8` (larger runner) |

The vCPU column is what each label advertises. Every job records what it actually got before the
build starts and warns if the vCPU count differs.

## Results

Each run ends with a summary job that posts a table to the run's summary page: one row per runner,
one column per phase, sorted by tier and then by total time, with failed jobs last. The same data is
attached to the run as a `results.json` artifact. See the [Actions tab](../../actions) for runs.

## Methodology

- **Same work everywhere.** Both projects are pinned to a commit, the Rust toolchain to 1.98.1 and
  Node to 22, with pnpm pinned by mastra's `packageManager` field.
- **Cold caches.** Nothing is restored from a cache. The mastra job points pnpm's store and corepack
  at empty directories and refuses to start if a pnpm store already exists, so the install is always
  cold. `TURBO_FORCE` rules out turbo cache hits. Cargo downloads every crate.
- **Only the phases are timed.** Each measured phase is a `phase: <name>` step opened by
  [`bin/mark`](bin/mark), which records a microsecond timestamp. Setup outside the phases (checking
  out this repository, recording host info, installing Node and pnpm for mastra) is not counted.
- **Independent jobs.** `fail-fast` is off, so one runner failing never cancels the others.
- **Limits.** Each dispatch is one sample per runner, and network-bound phases (checkout, installs,
  crate downloads) vary from run to run. Compare several runs before drawing conclusions from small
  differences.

## Running

Both workflows run on demand:

```sh
gh workflow run typescript-mastra.yml
gh workflow run rust-spotify-player.yml
```

## Development

Tools are pinned in [`mise.toml`](mise.toml):

```sh
mise trust
mise exec -- actionlint
mise exec -- zizmor .
mise exec -- shellcheck bin/mark
```

To add a runner, add an `include` entry (and its tier or provider if new) to both workflows'
matrices, and to [`.github/actionlint.yaml`](.github/actionlint.yaml) if actionlint doesn't know
the label.

## License

[MIT](LICENSE)
