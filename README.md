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

| Workflow | Project | Measured phases ([suite action](.github/actions)) |
|---|---|---|
| [`typescript-mastra.yml`](.github/workflows/typescript-mastra.yml) | [mastra-ai/mastra](https://github.com/mastra-ai/mastra) @ `b6eeb9d` | [checkout, cold install (`pnpm install --frozen-lockfile`), lint (`pnpm lint:format`), build (`pnpm build:core`)](.github/actions/typescript-mastra/action.yml) |
| [`rust-spotify-player.yml`](.github/workflows/rust-spotify-player.yml) | [aome510/spotify-player](https://github.com/aome510/spotify-player) @ `7dc9d17` | [checkout, system packages, toolchain, `cargo fmt`, `cargo test`, `cargo clippy`, `cargo clippy` (no features)](.github/actions/rust-spotify-player/action.yml) |

The mastra phases follow the realworld mastra profile of
[hpc-sandbox-benchmarks](https://github.com/starslingdev/hpc-sandbox-benchmarks); the Rust phases
follow the rust suite of
[runs-on-demo/benchmark-public](https://github.com/runs-on-demo/benchmark-public).

## Runners

Every benchmark runs the same matrix, defined once in
[`bench.yml`](.github/workflows/bench.yml): vCPU tier × provider × replica, with `replicas` jobs
(default 5) per cell:

| vCPU | StarSling | GitHub |
|---|---|---|
| 2 | `starsling-ubuntu-26.04-2` | `ubuntu-26.04-2` (larger runner) |
| 4 | `starsling-ubuntu-26.04` | `ubuntu-26.04` (standard runner) |
| 8 | `starsling-ubuntu-26.04-8` | `ubuntu-26.04-8` (larger runner) |

The vCPU column is what each label advertises. Every job records what it actually got before the
build starts and warns if the vCPU count differs.

## Results

Each run ends with a summary job that posts a table to the run's summary page: one row per runner,
with the p50 / p90 of every phase and of the total over that runner's successful replicas, and how
many of the requested replicas succeeded (a job that failed before reporting still counts against
it). Rows are sorted by tier, then by p50 total. Every job's raw timings are
attached to the run as a `results.json` artifact. See the [Actions tab](../../actions) for runs.

## Methodology

- **Same work everywhere.** Every benchmark runs through the same pipeline on the same runners. Both
  projects are pinned to a commit, the Rust toolchain to 1.98.1 and
  Node to 22, with pnpm pinned by mastra's `packageManager` field.
- **Cold caches.** Nothing is restored from a cache. The mastra job points pnpm's store and corepack
  at empty directories and refuses to start if a pnpm store already exists, so the install is always
  cold. `TURBO_FORCE` rules out turbo cache hits. Cargo downloads every crate.
- **Only the phases are timed.** Each measured phase is a `phase: <name>` step opened by
  [`bin/mark`](bin/mark), which records a microsecond timestamp, and the last one is closed when
  the job finishes. Setup outside the phases (checking
  out this repository, recording host info, installing Node and pnpm for mastra) is not counted.
- **Independent jobs.** `fail-fast` is off, so one runner failing never cancels the others.
- **Replicas.** Every cell runs `replicas` independent jobs, each on a fresh runner, all dispatched
  together so every runner sees the same time of day. Percentiles use linear interpolation over the
  successful replicas only.
- **Limits.** Network-bound phases (checkout, installs, crate downloads) vary from run to run, and a
  few replicas per cell can't separate small differences. Raise `replicas` before drawing
  conclusions from them.

## Running

Both workflows run on demand, with `replicas` (1–40, default 5) jobs per runner:

```sh
gh workflow run typescript-mastra.yml
gh workflow run rust-spotify-player.yml -f replicas=10
```

## Layout

| Path | What it is |
|---|---|
| [`.github/workflows/<suite>.yml`](.github/workflows) | Entry point per benchmark: the `replicas` input, then a call to `bench.yml`. |
| [`.github/workflows/bench.yml`](.github/workflows/bench.yml) | The shared pipeline: a `plan` job validates the inputs, a `bench` job runs the suite once per runner and replica, and a `summary` job renders the table. |
| [`.github/actions/<suite>`](.github/actions) | The suite's phases, as a composite action. |
| [`.github/actions/host-info`](.github/actions/host-info/action.yml) | Records the host (vCPU, RAM, disk, OS image) before the suite starts. |
| [`.github/actions/bench-finish`](.github/actions/bench-finish/action.yml) | Closes the last phase with the job status and uploads host info and timings. |
| [`.github/actions/summarize`](.github/actions/summarize/action.yml) | Builds the p50 / p90 table with `core.summary` and uploads `results.json`. |
| [`bin/mark`](bin/mark) | Records a phase boundary. |

## Development

Tools are pinned in [`mise.toml`](mise.toml):

```sh
mise trust
mise exec -- actionlint
mise exec -- zizmor .
mise exec -- shellcheck bin/mark
```

To add a runner, add an `include` entry (and its tier or provider if new) to the matrix in
[`bench.yml`](.github/workflows/bench.yml), and the label to
[`.github/actionlint.yaml`](.github/actionlint.yaml) if actionlint doesn't know it.

To add a benchmark:

1. Write its phases as a composite action in `.github/actions/<suite>/action.yml`: a `mark <phase>`
   step before each `phase: <name>` step, and `mark -` before untimed setup.
2. Add `<suite>` to the `plan` job's `case` and a matching `if: inputs.suite == '<suite>'` step to
   the `bench` job in `bench.yml`. `uses:` can't take an expression, so each suite is its own step.
3. Add an entry workflow `.github/workflows/<suite>.yml` that calls `bench.yml` with
   `suite: <suite>`, like the existing ones.

## License

[MIT](LICENSE)
