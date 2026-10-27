# iii issues log — round 2

> Verified against `iii-hq/iii` at `b10cdda46` (2026-10-27), the live docs (`iii.dev/docs`),
> `workers.iii.dev` + its sitemap, and `iii-hq/templates@main`.
>
> Companion file: [`iii-system-understanding.md`](./iii-system-understanding.md).
>
> Severity: **S1** blocks adoption or is legally wrong, **S2** misleads or breaks a real user,
> **S3** polish. Confidence: **verified** (source or live fetch I ran), **likely** (strong
> inference from source), **needs decision** (maintainer call).

Round 1 of this log was discarded. Its surviving items shipped, so this file starts from the
current tree instead of carrying stale rows forward.

---

## Round 1 disposition

| ID | Round 1 subject | Outcome |
| --- | --- | --- |
| I-01 | Go SDK missing from the docs IA | Dropped — **PR #2399** (not mine) already adds it |
| I-02 | CLI licensing says Apache-2.0, lives in the ELv2 tree | Dropped — **PR #2408** covers it |
| I-03 | Roadmap index omits 2 specs | Dropped — my error: drafts are excluded by design (`website/lib/specs.ts:85`) |
| I-04 | `engine/README.md` layout + ports stale | **PR #2410** |
| I-05 | Port ownership contradicts itself | Folded into **#2410** (links the canonical table instead of duplicating it) |
| I-06 | Legal entity: "III, Inc." vs "Motia LLC" | **Issue #2412** (needs a maintainer decision, not a patch) |
| I-07 | `README_zh.md` links to a deleted `console/README.md` | **PR #2411** |
| I-08 | Publish page conflates `iii.worker.yaml` with the runtime surface | Held at your request |
| I-09 | Kits have no docs page | Held at your request |
| I-10 | Example repositories overlap, two are stale | Held at your request |
| I-11 | `iii-stream` deprecated but enabled in the default config | Held at your request |

---

## Summary

| ID | Issue | Area | Severity | Confidence | Proposed form |
| --- | --- | --- | --- | --- | --- |
| N-01 | The shipped Docker Compose stack cannot boot | docker/config | S2 | verified | issue |
| N-02 | Two live docs still tell users to run the removed `--use-default-config` | docs/CLI | S2 | verified | 1 commit into #2410 + 1 PR |
| N-03 | The Go SDK's only example cannot work as written | sdk-go/docs | S2 | verified | PR |
| N-04 | `install.mdx` pins an engine version that was never released | docs | S2 | verified | PR |
| N-05 | `functions.mdx` links to a registry page for an engine-owned worker | docs/registry | S3 | verified | PR |
| N-06 | `docker compose up -d` in `engine/README.md` has no compose file at the repo root | docs | S3 | verified | commit into #2410 |
| N-07 | Quickstart shows an output the scaffold does not produce | docs/tutorial | S3 | verified | PR |

---

## N-01: The shipped Docker Compose stack cannot boot

**Severity:** S2. **Confidence:** verified from source (not executed; the reproduction command is
below and needs one image pull). **Effort:** small, but it is a config change in `engine/`.

**What.** `engine/docker-compose.yml` is presented as "full stack with Redis + RabbitMQ", and its
`iii` service mounts `./config.yaml`, which is `engine/config.yaml`. That file points the
`iii-stream` worker at `redis://localhost:6379`. Inside the `iii` container, `localhost` is the
`iii` container, not the sibling `redis` service, so the connection fails on boot. The engine
treats a failed worker creation as fatal, so the container does not come up.

**Evidence.**

`engine/docker-compose.yml` — the `iii` service mounts the config and declares the dependency:

```yaml
    ports:
      - "49134:49134"  # WebSocket (worker connections)
      - "3111:3111"    # REST API
      - "3112:3112"    # Stream API
      - "9464:9464"    # Prometheus metrics
    volumes:
      - ./config.yaml:/app/config.yaml:ro
    depends_on:
      redis:
        condition: service_healthy
```

`engine/config.yaml:7-14` — the value inside that mount:

```yaml
  - name: iii-stream
    config:
      port: ${STREAM_PORT:3112}
      host: 127.0.0.1
      adapter:
        name: redis
        config:
          redis_url: redis://localhost:6379
```

Nothing rewrites `localhost` for Docker. `III_EXECUTION_CONTEXT`, which the compose file sets, is
read only by telemetry (`engine/src/workers/telemetry/environment.rs:12`), and
`engine/src/workers/config_rewrite.rs` only removes seeded `config:` blocks, it does not rewrite
adapter URLs.

The connection is eager and bounded at 5s, and the error is propagated:

- `engine/src/workers/stream/adapters/redis_adapter.rs:44` —
  `timeout(DEFAULT_REDIS_CONNECTION_TIMEOUT, client.get_connection_manager())`
- `engine/src/workers/redis.rs:9` — `DEFAULT_REDIS_CONNECTION_TIMEOUT = 5s`
- `engine/src/workers/config.rs:779-788` — `create_worker(...).await.map_err(...)?` then
  `initialize().await.map_err(...)?`, so one failed worker aborts `build()`

`docker-compose.yml` sets no `redis_url` override, so there is no escape hatch in the compose file
itself.

**Second defect in the same file.** `3112:3112` is published, but `iii-stream` binds
`host: 127.0.0.1` (`engine/config.yaml:10`), and the worker binds that value verbatim
(`engine/src/workers/stream/stream.rs:343-344`: `format!("{}:{}", config.host, config.port)` →
`TcpListener::bind`). Docker forwards to the container's interface, not its loopback, so the
published stream port never answers. `engine/Dockerfile` also `EXPOSE`s `3112`.

**Impact.** The only documented all-in-one Docker path fails, and the failure mode (container
exits after a 5s Redis timeout) reads like a broken image rather than a config mismatch. Everything
under "Docker" in `engine/README.md` dead-ends.

**Reproduction.** From `engine/`:

```bash
docker compose up
# iii-stream: Failed to connect to Redis at redis://localhost:6379: connection refused
```

**Suggested fix.** Ship a Docker-specific config (or `${REDIS_URL:redis://localhost:6379}` plus an
env in the compose file) with `redis_url: redis://redis:6379`, and either bind `0.0.0.0` for the
published `3112` or stop publishing it. The host-run `worker-compose.yaml:11` keeps `localhost`,
which is correct there.

**Overlap to disclose.** #1925 is open against the same file, but it only adds the `/app/data`
volume and does not touch `redis_url`. This is a separate defect; whichever lands second needs a
two-line rebase.

**Why an issue and not a PR.** `engine/config.yaml` is on the repo's "ask first" list, and the
right answer depends on whether binding the stream port on `0.0.0.0` is acceptable.

---

## N-02: Two live docs still tell users to run the removed `--use-default-config`

**Severity:** S2. **Confidence:** verified. **Effort:** trivial.

**What.** `--use-default-config` was removed in 0.21, and the engine now rejects it. Two documents
that ship today still open with it.

**Evidence.**

```
engine/README.md:39                                  iii --use-default-config
sdk/packages/go/iii-example/README.md:14             iii --use-default-config   # engine on ws://localhost:49134, HTTP on :3111
sdk/packages/go/iii-example/main.go:8                //	iii --use-default-config   # in another terminal, engine on :49134 / :3111
```

The removal is asserted in two places:

- `engine/src/main.rs:707-714` — `use_default_config_is_no_longer_a_flag()` expects
  `Cli::try_parse_from(["iii", "--use-default-config"])` to fail.
- `engine/tests/cli_args.rs:51-66` — expects the process to exit non-zero with
  `"unexpected argument"` or the flag name in stderr.

And it is documented as a migration step, which is what makes the two survivors unambiguous:

> `docs/0-21-0/upgrading/from-0-21-x.mdx:54` — "The `--use-default-config` engine flag is
> removed. Running `iii` with no `config.yaml` now creates …"

**Impact.** `engine/README.md` is the first command a contributor copies, and the Go example's
step 1 is the same. Both fail immediately with a clap error.

**Suggested fix.** Bare `iii`. It creates `config.yaml` when missing, which is the behaviour the
migration guide describes.

**Disposition.** The `engine/README.md` line is in the file PR #2410 already owns, under the same
"the documented startup does not work" theme as the CodeRabbit fixes. Fold it there. The two Go
example lines go with N-03.

---

## N-03: The Go SDK's only example cannot work as written

**Severity:** S2. **Confidence:** verified. **Effort:** small.

**What.** `sdk/packages/go/iii-example` is the only runnable Go sample, and its README has three
independent problems: a removed flag, a wrong statement about who serves HTTP, and a dead source
reference.

**Evidence.**

*(a) removed flag* — `README.md:14` and `main.go:8`, see N-02.

*(b) wrong HTTP ownership* — `README.md:14` says:

```text
iii --use-default-config            # engine on ws://localhost:49134, HTTP on :3111
```

The engine does not serve HTTP routes. `3111` appears in `engine/src` only inside configuration
typing tests; inbound routes belong to the `http` Compose worker. The example registers
`hello::greet` behind an `http` trigger (`main.go`), so step 3's `curl localhost:3111/greet`
cannot work on the bare engine the README describes. The prerequisite is missing:
`iii trigger compose::add worker=http`, which `engine/README.md`, `docs/next/quickstart.mdx`, and
`docs/next/using-iii/console.mdx` all use.

*(c) dead source path* — `main.go:32`:

```go
// under "body" — not at the top level. See engine/src/workers/rest_api/README.md.
```

`engine/src/workers/rest_api` does not exist. `engine/src/workers/` has `http_functions`,
`configuration`, `stream`, `observability`, `telemetry`, `worker`, `engine_fn`, `queue`.

**Impact.** A Go developer's first five minutes end in a clap error, then a connection refused on
:3111. The Go SDK is otherwise complete, and its only sample is the one thing that does not run.

**Suggested fix.** Drop the flag, state that the engine serves only WebSocket and that `http`
owns the routes, add the `compose::add worker=http` prerequisite, and repoint the envelope note at
`docs/next/creating-workers/functions.mdx` (or `engine/src/workers/http_functions/`) instead of a
deleted directory.

**Good first issue?** Yes.

---

## N-04: `install.mdx` pins an engine version that was never released

**Severity:** S2. **Confidence:** verified. **Effort:** trivial.

**What.** The Install page's "install one exact version" example, and the reference-row example
under it, both use `0.23.1`. That release does not exist.

**Evidence.**

```console
$ curl -fsSL https://install.iii.dev/iii/main/install.sh | sh -s -- 0.23.1
error: release tag not found: 0.23.1 (tried: iii/v0.23.1, v0.23.1)
```

From the repository:

```console
$ git rev-parse -q --verify refs/tags/iii/v0.23.1 || echo MISSING
MISSING
$ git rev-parse -q --verify refs/tags/v0.23.1 || echo MISSING
MISSING
$ git tag --list 'iii/v0.2[345].*' | sort -V | tail -4
iii/v0.24.6-rc.5
iii/v0.25.0
iii/v0.25.0-rc.1
iii/v0.25.0-rc.2
```

The installer's own error text confirms the lookup order it tried
(`install.sh:447-457`: strip `iii/`, strip `v`, then probe `iii/v<ver>` and `v<ver>`).

Latest stable is `iii/v0.25.0` (`gh release view --json tagName,isLatest` → `iii/v0.25.0`,
`latest=true`).

**Impact.** The command is on the page every new user reads, and it is the documented escape hatch
for pinning. It fails with an error that sounds like an outage, not a typo.

**Suggested fix.** Use `0.25.0` in both places (`install.mdx:106` and the `VERSION` row at
`install.mdx:134`). Consider naming a version that exists in the same PR that cuts the release, or
using `--next`/`--rc` in the examples so they cannot rot.

**Overlap to disclose.** #2028 is open against `docs/next/install.mdx`, but its 10 changed lines
are internal links only; it does not touch the version example.

---

## N-05: `functions.mdx` links to a registry page for an engine-owned worker

**Severity:** S3. **Confidence:** verified. **Effort:** trivial.

**What.** The HTTP-invokable functions section sends readers to
`https://workers.iii.dev/workers/iii-http-functions` for configuration and security controls. That
worker is engine-owned, so it has no registry listing and the page 404s.

**Evidence.**

`docs/next/creating-workers/functions.mdx:286-290`:

```markdown
## HTTP-invokable functions

The `iii-http-functions` worker provides this feature. See its
[Worker Docs](https://workers.iii.dev/workers/iii-http-functions) for its configuration and
security controls.
```

The worker lives in the engine: `engine/src/workers/http_functions/`, and the root README lists it
among engine-owned workers (`README.md:154`). The registry has no entry:

```console
$ curl -s -o /dev/null -w '%{http_code}\n' https://workers.iii.dev/workers/iii-http-functions
404

$ curl -sL https://workers.iii.dev/sitemap.xml | grep -oE 'https://[^<]+' | grep '/workers/' | sed 's|.*/||' | grep -iE 'func|http'
http
```

Every other registry link in the docs resolves: `configuration`, `cron`, `database`, `http`,
`iii-observability`, `iii-sandbox`, `iii-stream`, `iii-worker-manager`, `pubsub`, `queue`,
`rbac-proxy`, and `state` all return 200. Only this one is broken, and only because the target
cannot exist.

**Impact.** A reader who wants to configure an outbound HTTP function lands on a 404 and has no
docs path to the engine-owned worker's settings.

**Suggested fix.** Point at the engine docs (`using-iii/engine` covers engine-owned worker config)
or drop the link and describe the settings inline.

---

## N-06: `docker compose up -d` in `engine/README.md` has no compose file at the repo root

**Severity:** S3. **Confidence:** verified. **Effort:** trivial.

**What.** The Docker Compose section gives a bare command, but the compose files live under
`engine/`, and nothing tells the reader to change directory.

**Evidence.**

`engine/README.md:90-96`:

```markdown
### Docker Compose (full stack with Redis + RabbitMQ):

```bash
docker compose up -d
```
```

At the repository root:

```console
$ ls docker-compose*.yml
ls: cannot access 'docker-compose*.yml': No such file or directory
$ ls config.yaml
ls: cannot access 'config.yaml': No such file or directory
```

The files that exist are `engine/docker-compose.yml` and `engine/docker-compose.prod.yml`. Compose
resolves the `./config.yaml` and `./Caddyfile` mounts relative to the compose file's directory, so
`docker compose -f engine/docker-compose.yml up -d` from the root behaves identically to running it
inside `engine/`.

**Impact.** `docker compose up -d` at the root fails with "no configuration file provided", which
reads as a missing file rather than a wrong working directory. The reader is one section past a
`docker run` example that does work, so it is easy to conclude the compose path is broken.

**Disposition.** Same file and same section as the CodeRabbit Docker findings. Fold into #2410.

**Caveat worth stating in the PR.** Fixing the path exposes N-01: the stack starts but the engine
will not stay up until the Redis URL is fixed. If #2410 lands first, the note should point readers
at the issue rather than promise a working stack.

---

## N-07: Quickstart shows an output the scaffold does not produce

**Severity:** S3. **Confidence:** verified. **Effort:** trivial.

**What.** Step 5 of the Quickstart shows the literal result of calling the TypeScript worker, but
the template's handler returns an extra field.

**Evidence.**

`docs/next/quickstart.mdx:115-119`:

```bash
iii trigger math::add_two_numbers a=10 b=20
```

```json
{ "c": 30 }
```

`iii-hq/templates@main`, `iii/quickstart/workers/caller-worker/src/worker.ts`:

```typescript
return {
  ...result,
  success:
    "You've connected two workers and they're interoperating seamlessly, now let's add a few more workers to expand this project's functionality.",
};
```

So the real output is `{ "c": 30, "success": "…" }`. The scaffold is otherwise faithful: the tree
in step 1 matches `iii/quickstart` exactly (`workers/math-worker/src/math_worker.py`,
`workers/caller-worker/src/worker.ts`), `math::add` and `math::add_two_numbers` exist, and steps 6
and 7's outputs do match their handlers.

**Impact.** Small, but this is the flagship tutorial and the JSON block is presented as a terminal
transcript. A reader who compares and sees a different shape wonders whether their scaffold is
stale.

**Suggested fix.** Show the full object, or trim the `success` field from the template handler so
the doc stays true. Showing it is better: it is the "two workers interoperating" moment the step is
trying to demonstrate.

---

## Checked and clean

Recorded so these are not re-investigated next round:

- **Internal doc links.** All 76 `docs/next/*.mdx` relative and root-absolute links resolve. The
  two `](/changelog)` links that look broken are rewritten by Mintlify to `/docs/changelog` (200).
- **External links.** 105 unique URLs from the root README, `engine/README.md`, and `docs/next`;
  the only 404s are the ones above plus intentional `example.com` payloads in the Linkly tutorial.
  `crates.io` and `npmjs.com` 403/404s are bot blocking; the registry APIs return 200.
- **Registry links.** 16 `workers.iii.dev/workers/*` targets, all 200 except N-05.
- **`iii.dev/llms.txt`, `iii.dev/AGENTS.md`, `/registry`, `/roadmap`, `/blog/index.md`** — 200.
- **Install-doc env vars.** All 10 `III_*` variables named in `docs/next` are read somewhere in the
  repo, none stale.
- **Quickstart template.** `iii-hq/templates@main` `iii/quickstart` matches the documented tree and
  function names.
- **Skills contract.** Six of seven `iii-*` skills carry `## When to Use` and `## Boundaries`;
  `presentation` uses `Activation` / `Use For` / `Do Not Use For` instead, which contradicts
  `AGENTS.md:87` but is a deliberate tool-skill shape. Not filed; worth a one-line exemption in
  `AGENTS.md` if anyone cares.
- **Engine README ports/layout.** Already in #2410.

## Suggested submission order

1. Fold N-02 (`engine/README.md` half) and N-06 into **#2410**, one commit, since that PR already
   owns the file and the "documented commands must work" theme.
2. **N-03 + the Go half of N-02** → one PR, `docs(sdk-go): make the example runnable`. Two files,
   ~10 lines.
3. **N-04** → one PR, `docs(install): use a released version in the exact-version example`. Two
   lines, one file.
4. **N-05** and **N-07** → one PR each, or a single `docs: fix a 404 and a stale transcript` if you
   prefer fewer branches.
5. **N-01** → issue only. It needs a decision about `host: 0.0.0.0` and the compose Redis URL.
