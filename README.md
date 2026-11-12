# iyl

**HTTP load testing that tells the truth, built for CI.** One native binary,
written in [iyi](https://iyi.dev): it drives every core, measures latency the
way a user feels it, and turns a run into an exit code a pipeline can act on.

```console
$ iyl http://localhost:3000/api -d 30s -c 64 --max-p99 250ms
```

- **Honest latency.** Under `--rate`, every request is also timed from the
  slot the schedule gave it, so a target that stalls is charged for the queue
  it caused instead of looking faster ([below](#honest-latency)).
- **Gates, not graphs.** `--max-p99`, `--min-rps`, `--max-fail-rate` and the
  rest exit 1 when breached; `--url-threshold` scopes a gate to some URLs;
  `--baseline` fails a run that regressed against an earlier one.
- **Exit codes that mean something.** A slow service (1), a broken command
  line (2) and a target that never answered (3) are different failures.
- **Where the time went.** DNS, connect and TLS per new connection; time to
  first byte and total per response; per status, per URL, per error kind.
- **Reports for machines and people.** Text with color, JSON with a versioned
  schema, CSV one row per run, and Markdown for a job summary or a pull
  request. `--save` keeps the JSON beside any of them.
- **Multi-core.** Connections are spread over worker threads, each worker's
  connections taking turns on it, nothing shared but a handful of atomics.
- **HTTP/1.1 done carefully.** Keep-alive, bodies framed by length, chunks or
  close, read through one fixed buffer per connection and never kept;
  redirects, proxies (`CONNECT` for https), Basic and Bearer credentials,
  cookies, custom success codes, request deadlines.

## Install

iyl is built from source with the iyi toolchain
([install iyi](https://github.com/iyilang/iyi#install)):

```sh
git clone https://github.com/sdogruyol/iyl && cd iyl
iyi build --release -o iyl iyl.iyi
./iyl --version
```

The result is a single executable with no runtime to install and no OpenSSL:
TLS is iyi's own.

## Usage

```console
$ iyl [options] <url>...
```

```sh
# 10,000 requests over 100 connections
iyl http://localhost:3000 -n 10000 -c 100

# 30 seconds at a steady 500 requests a second, gated on the honest p99
iyl http://localhost:3000/api -d 30s -q 500 --max-p99 250 --fail-on-rate-miss

# POST JSON, warm up first, never wait more than 2s on one request
iyl http://localhost:3000/orders -d 1m -c 32 -m POST \
  --content-type application/json -b '{"sku":"A-1","qty":2}' \
  --warmup 5s --request-timeout 2s

# Several endpoints in turn, each with its own gate
iyl --urls-file urls.txt -d 30s -c 50 \
  --url-threshold "/search max-p99 400" \
  --url-threshold "/orders max-fail-rate 0.5" \
  --min-rps 2000
```

`iyl --help` lists every option:

| Load | |
|---|---|
| `-n, --requests N` | send N requests in all |
| `-d, --duration TIME` | run for TIME: `30`, `30s`, `2m`, `1h30m`, `500ms` (default 10s). With `-n` too, the run ends at whichever comes first |
| `-c, --connections N` | connections open at once (default 10) |
| `-w, --workers N` | threads driving them (default: the cores, at most one per connection) |
| `-q, --rate R` | requests a second across all connections, fractions allowed |
| `--warmup TIME` | load without measuring first; its connections stay open for the run |
| `--timeout TIME` | bound on each connect, read and write (default 10s, `0` for none) |
| `--request-timeout TIME` | bound on a whole request, redirects and body included |
| `--disable-keepalive` | a new connection for every request, its setup counted |
| `-L, --follow-redirects` | follow 3xx answers, as one request (`--max-redirects N`, default 10) |

| Request | |
|---|---|
| `-m, --method METHOD` | HTTP method (default GET) |
| `-b, --body TEXT`, `--body-file PATH`, `--body-stdin` | request body |
| `-H, --header 'K: V'` | header, repeatable |
| `--content-type TYPE` | Content-Type of the body |
| `-a, --basic-auth USER:PASS`, `--bearer TOKEN` | credentials |
| `--cookie NAME=VALUE` | cookie, repeatable |
| `--user-agent TEXT`, `--host-header HOST` | override those headers |
| `--urls-file PATH` | URLs one a line, optionally after a method: `POST https://...` |
| `--random-path` | a random segment on every path, to get past caches |
| `--proxy URL` | send through an HTTP proxy, `user:pass@` for its credentials |
| `--cacert PATH` | trust the CA certificates in this PEM file instead of the system's |
| `--success-status LIST` | statuses that count as success (default `200-299`), e.g. `200-299,304` |

| Output | |
|---|---|
| `-o, --output-format FMT` | `text`, `json`, `csv`, `markdown` or `quiet` (`--json` for short) |
| `--save PATH` | also write the JSON report to PATH |
| `--progress`, `--no-progress` | live line on stderr (default: when stderr is a terminal) |
| `--no-color` | plain text; `NO_COLOR` does the same |

A URL without a scheme is `http://`; credentials written in a URL
(`http://user:pass@host/`) are sent as Basic.

## Honest latency

A load tester that measures from the moment it sends a request has a blind
spot exactly where it matters. When the target stalls, the connections that
would have sent requests are stuck waiting, nothing is sent, and so nothing
slow is measured: the report shows the fast answers from before and after
the stall. A target that stops answering for a second looks *better*. This is
coordinated omission.

Give iyl a rate and it keeps a schedule: request *k* is due at
`start + k / rate`, whatever happened before it. Slots are handed out on that
fixed timeline, never re-anchored to "now", so a stall leaves a backlog rather
than a gap, and each request is timed twice:

- **service** — from when it was actually sent to the last byte of its answer:
  what the server spent;
- **corrected** — from when it was *due* to the last byte: what a user who
  arrived on schedule waited.

Their difference is the **send delay**, reported on its own. The schedule also
says how many requests fell due and how many were sent; the rest are
**skipped**, so a run can never claim a rate it did not reach.

Here is a target that takes 200 ms per request, asked for 50 a second on one
connection:

```text
Latency           avg      p50      p90      p95      p99    p99.9      max
  service      200.8ms  201.6ms  201.6ms  203.6ms  203.6ms  203.6ms  203.7ms
  corrected      1.83s    1.84s    3.27s    3.46s    3.46s    3.46s    3.46s
  send delay     1.63s    1.63s    3.08s    3.26s    3.26s    3.26s    3.26s

Schedule
  requested 50/s · attained 5.0/s (10.0%)
  scheduled 150 · issued 20 · skipped 130 · worst send delay 3.26s

Gates
  ✘ max-p99                  corrected_p99 3.46s <= 300.0ms
  ✘ fail-on-rate-miss        attainment_percent 9.95 >= 99.00
```

The service p99 of 204 ms is true and would have passed a 300 ms gate; it is
also not what anyone experienced. Latency gates read the corrected numbers
whenever `--rate` is set; `--latency-correction off` makes them read service
time. Without `--rate` there is no schedule, and the two are the same.

## In CI

### Gates

Any breached gate exits 1. Latency limits are milliseconds, or a time with
its unit (`1.5s`).

| Flag | Breached when |
|---|---|
| `--max-p50`, `--max-p75`, `--max-p90`, `--max-p95`, `--max-p99`, `--max-p999 MS` | that percentile is above MS |
| `--max-avg MS` | the average is above MS |
| `--max-latency MS` | the slowest request took longer |
| `--min-rps N` | throughput is below N requests a second |
| `--max-fail-rate PCT` | more than PCT percent of requests failed (HTTP or transport) |
| `--fail-on-error` | any request failed |
| `--fail-on-transport-error` | any request got no HTTP answer |
| `--fail-on-rate-miss` | the run attained less than 99% of `--rate` |
| `--url-threshold 'PATTERN METRIC VALUE'` | METRIC, over the URLs containing PATTERN, breaches VALUE |
| `--baseline PATH` | p50, p99 or throughput regressed more than `--max-regression` percent (default 10) against an earlier JSON report |

A `--url-threshold` whose pattern matches none of the URLs under load is a
usage error, so a renamed endpoint fails the build instead of quietly losing
its gate.

### Exit codes

| Code | Meaning |
|---|---|
| `0` | the run finished and every gate held |
| `1` | a gate was breached |
| `2` | usage or configuration error: a bad flag, URL, file or certificate |
| `3` | the target was unreachable: no HTTP answer at all, and at least one transport error |
| `130` | interrupted (SIGINT) |
| `143` | terminated (SIGTERM) |

A breached gate outranks a signal; an unreachable target outranks a gate,
since there was nothing to measure. The JSON report carries the same as
`verdict.exit_code` and `verdict.reason`.

### GitHub Actions

```yaml
- name: Load test
  run: |
    iyl http://localhost:3000/api -d 30s -q 500 \
      --max-p99 250 --fail-on-rate-miss \
      --baseline perf/baseline.json --save result.json \
      -o markdown >> "$GITHUB_STEP_SUMMARY"
```

The Markdown report lands in the job's summary; `result.json` is the next
run's baseline.

## Reports

`--json` prints one document; every key is always present, `null` where it
does not apply. All latencies are milliseconds.

| Key | |
|---|---|
| `schema_version` | `1`; moves only when a key changes meaning or goes away |
| `tool`, `title`, `mode` | `{name, version}`, the URL (or the first and a count), and `duration`, `requests` or `both` |
| `config` | workers, connections, requests, duration, warmup, rate, method, keep-alive, redirects, latency correction, timeouts, success statuses |
| `summary` | requests, responses, ok, failed (HTTP), transport_errors, abandoned (in flight when the run was cut), redirects, elapsed_seconds, requests_per_second, failure_rate_percent |
| `rate` | requested and attained rate, attainment_percent, scheduled, issued and skipped requests, max_send_delay_ms; all but `attained_per_second` null without `--rate` |
| `transfer` | total body bytes as received, per response, per second |
| `latency_ms`, `corrected_latency_ms`, `send_delay_ms` | count, avg, min, max, stdev, p50, p75, p90, p95, p99, p999 |
| `phases_ms` | `dns`, `connect`, `tls` (per new connection), `ttfb`, `total` (per response): count, avg, min, max, p50, p95, p99 |
| `histogram` | ten bars, each the same ratio wide: start_ms, end_ms, count, percent |
| `statuses` | per status: code, count, percent, success, avg_ms, p50_ms, p95_ms, p99_ms |
| `transport_errors` | per kind: category, count, percent, one sample message |
| `urls` | per URL with more than one URL (up to 1000): requests, responses, failures, rate and latencies; `null` otherwise |
| `baseline` | with `--baseline`: the path, the allowed regression, and before/after/change for each metric compared |
| `gates` | `passed`, and every gate `evaluated` and `breached`: name, scope, metric, comparator, limit, actual, passed |
| `verdict` | exit_code, and reason: `passed`, `gate_breached`, `target_unreachable`, `interrupted`, `terminated` |

Transport errors are one of a fixed vocabulary, so a pipeline can match on
them: `dns_failure`, `connect_refused`, `connect_timeout`, `connect_failed`,
`read_timeout`, `write_timeout`, `request_timeout`, `connection_reset`,
`connection_closed`, `tls_error`, `proxy_error`, `protocol_error`, `other`.
A request whose kept connection the server had closed while idle is sent
again once on a new one, when its method is idempotent, and is not counted
as a failure.

`-o csv` writes a header and one row: the summary and the latency
percentiles, ready to be appended to a history file.

## How it works

Each worker is a task on iyi's pool of threads (`spawn_parallel`), and each
of its connections is a task on that worker, taking turns at every wait on
the network. What all workers read — the plan, built once from the command
line — is immutable, which the compiler checks (`Share`, SPEC.md III.4.4):
the only things they write together are the atomics behind the schedule, the
request count and the progress line. Each worker records into its own
histograms without a lock; they are merged once, at the end.

Latencies go into log-bucketed histograms, 1% wide from a microsecond to an
hour, with Welford's mean and variance, so percentiles cost no samples and
workers merge exactly. A response is parsed where it lies in the
connection's buffer: status, framing and the few headers that matter are
read off the bytes, and the body is counted on its way past.

TLS is iyi's own (TLS 1.3, and 1.2 for a server that speaks no newer). A server's certificate is
always checked: against the system's roots, or the CA certificates `--cacert`
names. There is deliberately no switch to skip the check; a test server's CA
goes in `--cacert` instead.

## Performance

`iyl http://127.0.0.1:3901/ -d 3s -c 128 -w N` against iyi's own HTTP server
on the same machine (Windows 11, 12 virtual CPUs under QEMU, loopback, both
built `--release`, 2-byte answers), the median of three runs:

| `-w` | requests a second | p50 | p99 |
|---:|---:|---:|---:|
| 1 | 18,554 | 5.95 ms | 26.7 ms |
| 4 | 100,912 | 0.88 ms | 5.95 ms |
| 8 | 181,637 | 0.54 ms | 2.94 ms |
| 12 | 184,178 | 0.57 ms | 2.63 ms |

One worker reaches 96% of what a bare loop of socket writes and reads, with
no parsing and no recording, reaches on the same machine and connections
(19,266 a second): measuring costs little. Past 8 workers the client and the
target fight for the same 12 cores and throughput stops rising: `-w` is a
dial, and a number worth reading is one taken where raising it no longer
helps.

These are loopback numbers on one virtual machine, a ceiling for the load
generator rather than a claim about any network.

## Development

```sh
iyi test                                         # every test/*_test.iyi
iyi run test/support/target_server.iyi -- 3000   # a target to point iyl at
iyi build -o iyl iyl.iyi && ./iyl http://127.0.0.1:3000/bytes/1024 -d 5s
```

The end-to-end tests run iyl against servers in their own process: plain,
TLS under a test CA (`test/fixtures`), and a proxy that tunnels.

| Module | |
|---|---|
| `iyl.iyi` | the entry point |
| `iyl/cli` | the command line, read into a plan |
| `iyl/plan` | the immutable plan every worker shares, and the request bytes |
| `iyl/target` | URLs and redirect locations |
| `iyl/engine` | workers, connections, the schedule, warmup, signals, progress |
| `iyl/wire` | HTTP/1.1 over TCP and TLS: connect, send, parse, count |
| `iyl/tally` | what a worker records |
| `iyl/histogram` | log-bucketed latency histograms |
| `iyl/verdict` | gates, the baseline comparison, the exit code |
| `iyl/report` | text, JSON, CSV and Markdown |
| `iyl/duration` | `1h30m` and friends |

## License

MIT
