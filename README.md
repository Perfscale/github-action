# perfscale load test action

Run a load test with [**perfscale**](https://github.com/Perfscale/perfscale) —
k6, locust, JMeter, or perfscale's own native step engine — and collect the
metrics as a downloadable artifact, straight from a GitHub Actions workflow.

The action downloads the pinned `perfscale` release binary for the runner's
OS/arch, verifies its `sha256` against the release's `sha256sums.txt` (fails
closed when the checksum is missing or mismatched), runs your test, and packs
the run summary plus
any generated metrics files into a single zip. By default it also renders the
metric table straight into the **GitHub Actions job summary** and writes a
machine-readable `perfscale-summary.json` (requests, RPS, latency percentiles,
error rate + engine/VUs/duration/timestamp metadata).

## Usage

```yaml
- uses: Perfscale/github-action@v1
  with:
    file: examples/hello.test.yaml
    config: examples/hello.config.yaml
```

Upload the produced artifact:

```yaml
- uses: Perfscale/github-action@v1
  id: loadtest
  with:
    k6: script.js
- uses: actions/upload-artifact@v4
  with:
    name: perfscale-metrics
    path: ${{ steps.loadtest.outputs.output-file }}
```

### Engines

Set **exactly one** of `k6`, `locust`, `jmeter`, or `file`.

```yaml
# k6
- uses: Perfscale/github-action@v1
  with:
    k6: examples/hello.k6.js

# locust
- uses: Perfscale/github-action@v1
  with:
    locust: examples/hello.locust.py
    host: https://example.com

# JMeter — the plan owns the load shape; needs the `jmeter` binary on PATH
- uses: Perfscale/github-action@v1
  with:
    jmeter: examples/hello.jmx

# native step engine (requires config)
- uses: Perfscale/github-action@v1
  with:
    file: examples/hello.test.yaml
    config: examples/hello.config.yaml
```

`k6`, `locust`, and `jmeter` shell out to the respective binary on `PATH` —
the action installs perfscale itself, not the engines. `k6`/`locust` are
one-package installs; JMeter needs Java (preinstalled on GitHub-hosted
runners) plus the distribution tarball:

```yaml
- name: Install JMeter
  run: |
    curl -fsSL https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-5.6.3.tgz \
      | tar -xz -C "$RUNNER_TEMP"
    echo "$RUNNER_TEMP/apache-jmeter-5.6.3/bin" >> "$GITHUB_PATH"
```

JMeter runs emit no latency percentiles — the final console `summary =` line
is translated into the summary block with `avg`/`min`/`max` only, and the
jmeter process exit code is the gate (no thresholds integration).

## Inputs

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | perfscale release to install (tag like `v0.2.0`, or `latest`). |
| `verify-checksum` | `true` | Verify the binary against the release's `sha256sums.txt`. Set to `false` to skip verification when the release ships no checksum entry for your OS/arch (not recommended). |
| `k6` | — | Path to a k6 `.js` script (`perfscale run --k6`). |
| `locust` | — | Path to a locustfile (`perfscale run --locust`). |
| `jmeter` | — | Path to a JMeter `.jmx` test plan (`perfscale run --jmeter`). Requires `jmeter` on `PATH`. |
| `file` | — | Path to a native `test.yaml` (`perfscale run -f`). Requires `config`. |
| `config` | — | Path to `config.yaml` (`-c`). Required with `file`. |
| `host` | — | Target base URL for the locust engine (`--host`). |
| `report` | — | POST the summary to a `perfscale serve` instance (`--report`). |
| `args` | — | Extra raw arguments appended verbatim to `perfscale run`. |
| `summary-export` | `perfscale-summary.json` | Path for the machine-readable JSON summary (`--summary-export`). Empty string disables it. |
| `job-summary` | `true` | Render the metric table into `$GITHUB_STEP_SUMMARY`. |
| `output` | `perfscale-output.zip` | Path of the zip artifact holding the summary + metrics. |
| `working-directory` | `.` | Directory to run perfscale from. |

`summary-export` and `job-summary` need a perfscale release that ships
`--summary-export`; on older releases the action degrades to a warning and
runs the test without them.

If the runner has no zip tool, the artifact is packed as `.tar.gz` instead —
the `output` extension is adjusted automatically and `output-file` points at
the real path.

## Outputs

| Output | Description |
|---|---|
| `summary-file` | Path to the captured run summary (perfscale stdout). |
| `summary-json` | Path to the machine-readable JSON summary (empty when disabled/unsupported). |
| `output-file` | Path to the zip artifact with the summary and metrics files. |
| `exit-code` | Exit code returned by `perfscale run`. |

### Consuming the JSON summary

```yaml
- uses: Perfscale/github-action@v1
  id: loadtest
  with:
    file: examples/hello.test.yaml
    config: examples/hello.config.yaml
- name: Gate on p95
  run: |
    p95=$(jq '.summary.p95_ms' "${{ steps.loadtest.outputs.summary-json }}")
    echo "p95 = ${p95} ms"
    awk "BEGIN { exit !(${p95} < 500) }" || { echo '::error::p95 over budget'; exit 1; }
```

## Exit codes

Mirrors the CLI: `0` = run completed (even if requests/checks failed), `1` =
the run could not execute (missing file, invalid YAML, engine binary not found,
engine crashed before producing results), `2` = invalid arguments.

## Platforms

Linux, macOS, and Windows runners (x64 and arm64) — matching the perfscale
release matrix.

## License

MIT
