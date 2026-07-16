---
name: deploy
description: >-
    How deployment to Skyr works: pushing to the skyr git remote, environments
    and the deployment lifecycle, exposing pods to the internet (ports,
    InternetAddress, DNS zones), private networking, first-party plugin
    capabilities, and checking rollout status and incidents. Use when deploying
    to Skyr, wiring a repository to a Skyr instance, exposing a service
    publicly, or debugging a rollout.
---

# Deploying to Skyr

Skyr is a Git-native infrastructure orchestrator: a repository of SCL
configuration *is* the deployable unit, and pushing it to a Skyr git remote
*is* the deployment action. There is no separate plan/apply step and no
deploy command — Skyr converges reality to whatever the pushed commit
declares. This skill describes how that works: the lifecycle, what the
first-party plugins can build (with complete examples), and how to observe
and debug a rollout. Authoring the SCL itself — syntax, types, modules,
`Package.scle` — is the `scl` skill's territory.

Examples use `skyr.foo` as the instance host; if the user's repository
deploys to a different Skyr instance, substitute that instance's host.

## Mental model

- The hierarchy is **Organization → Repository → Environment → Deployment**.
  An environment is a git branch or tag (`main`, `tag:v1.0`); a deployment is
  one commit of that environment. Deployment QIDs spell the whole chain:
  `acme/shop::main@a10fb43f…` (`/` org/repo, `::` env, `@` commit).
- `git push skyr main` is the deploy. Skyr evaluates `Main.scl` at the repo
  root, builds the resource dependency graph, and creates/updates/destroys
  resources until reality matches. Pushing a new commit to the same branch
  rolls the environment forward; resources shared between old and new config
  are *adopted* (ownership transfers), not recreated.
- Deleting a ref tears the environment down:
  `git push skyr --delete feature-x` destroys everything that environment
  owns, in dependency order.
- Deleting a whole **repository, organization, or account** goes further than a
  ref: `skyr repo delete`, `skyr org delete <org>`, and
  `skyr auth delete-account` tear down every environment involved, then
  **permanently** erase the entity's data — one-way, no undo, gated behind
  typing the entity's exact name to confirm. The entity stays visible with a
  `deleting` status until the purge finishes; while `deleting` it accepts reads
  but no pushes or other changes. Deleted user and org names are retired for
  good; a deleted repository's name frees up for reuse.
- Every resource has a **region** (a metro label like `"stockholm"`), part of
  its identity, defaulting to the repository's region. Region-placeable
  resources take a `region: Str?` input; changing it is destroy + create.

## Wiring a repository

Deployment needs three things: the CLI (for inspection, not for deploying),
an authenticated identity, and a git remote pointing at the instance.

- **CLI**: `curl -fsSL https://dl.skyr.cloud/install.sh | sh` installs
  `skyr` to `~/.local/bin`. `skyr --version` confirms.
- **Identity** is a Skyr username plus a registered SSH key — the same key
  authenticates both git pushes and CLI sessions. `skyr auth whoami` shows
  the current session; `skyr auth signin --username <u> --key <path>` starts
  one. `skyr auth signup --username <u> --email <e> --region <r>` creates a
  brand-new account and registers the key — that mints a real account, so
  whether (and as whom) to sign up is the user's decision, not a setup step
  to run through.
- **Org and repo** exist server-side before the first push:
  `skyr org create acme --region stockholm`, then `skyr repo create shop`
  (region defaults to the org's; a `--deployment-role` defaults to the org's
  all-access `Super` role).
- **Remote**: plain SSH at the instance host, authenticated by the
  registered key, with your username as the SSH user:

  ```sh
  git remote add skyr alice@skyr.foo:acme/shop
  git push skyr main
  ```

The CLI infers org/repo from the `skyr` remote (falling back to `origin`)
and the environment from the current branch. Everything can be overridden
with `--org`/`--repo`/`--env` flags or `SKYR_ORG`/`SKYR_REPO`/`SKYR_ENV`/
`SKYR_API_URL` env vars, and any command takes `--format json` for
machine-readable output.

## The deployment lifecycle

A deployment moves through five states:

- **Desired** — actively reconciled: evaluate config, create/update/adopt
  resources. New deployments start here.
- **Up** — converged *and* every resource is non-volatile; Skyr stops
  re-evaluating until the next push.
- **Lingering** — superseded by a newer push; waits while the new deployment
  rolls out and adopts shared resources.
- **Undesired** — teardown: resources not adopted by the successor are
  destroyed in dependency order.
- **Down** — nothing left; terminal.

Two things legitimately keep a deployment in **Desired forever** — that is
normal operation, not a stuck rollout:

- Any **volatile resource** (a `Container.Pod`, `DNS.Zone`, or `HTTP.Get`
  represents external state that can drift, so Skyr keeps reconciling).
- A **branch or tag pin** in `Package.scle` dependencies: the deployment
  keeps following the foreign ref. Pin commit hashes to settle.

A deployment of only stable resources (keys, random values, artifacts,
images) settles into Up.

## From a pod to a public domain

The complete path from "a container runs" to "https://example.com serves
it". Each stage is deployable on its own; later stages extend the same file.

### Run a container

```scl
import Skyr/Container
import Std/Path

let image = Container.Image({
    name: "web",
    context: ./app,
    containerfile: Path.read(./app/Containerfile),
})

let pod = Container.Pod({
    name: "web",
    containers: #{ "web": {
        image: image.url,
        cpu: 500,          // millicores; hard limit and reservation
        memory: 268435456, // bytes (256 MiB); hard limit and reservation
    } },
})
```

`Image` builds from a directory in the repo and pushes to the instance
registry; `image.url` is digest-pinned. Public images (`"caddy:2"`) work
directly as `image` values. `containers` is a **name-keyed map**
(`#{ "name": { ... } }`); the key names the container in logs and in the pod's
`exitCodes` output. `cpu` and `memory` are **required** on every container —
both are hard limits. Containers in the same pod share a network namespace, so
siblings reach each other on `localhost`.

### Open ports — how ingress works

Every pod gets a **public, internet-routable IPv6** (`pod.address`). By
default it is inert for ingress: the pod's firewall only accepts connections
from internal networks the pod is attached to, and the IPv6 serves egress.
Opening a port changes that:

```scl
let http = pod.Port({ port: 8080, public: true })
// http.address is "[<pod-ipv6>]:8080"
```

`public: true` opens the port to the internet — both on the pod's IPv6 and
on any bound IPv4 address (next section). Without it the port only accepts
traffic from attached internal networks. `protocol` defaults to `"tcp"`.

### A stable public IPv4

The pod's IPv6 changes when the pod is replaced (its identity hashes its
configuration). For a stable entry point, allocate an `InternetAddress` and
route it to the pod:

```scl
let ip = Container.InternetAddress({ name: "front-door" })

let pod = Container.Pod({
    name: "web",
    containers: #{ "web": { image: image.url, cpu: 500, memory: 268435456 } },
    internetAddress: ip,
})
```

`ip.address` is a public IPv4 that survives pod replacement and redeploys.
An address binds to **exactly one pod** — binding it twice is an eval-time
error. The pod's firewall still applies: only `public: true` ports accept
internet traffic on the address.

### A custom domain

`Skyr/DNS` serves authoritative DNS for user-owned domains:

```scl
import Skyr/DNS

let zone = DNS.Zone({ domain: "example.com" })

zone.ARecord({ name: "@", addresses: [ip.address] })
zone.AAAARecord({ name: "@", addresses: [pod.address] })
zone.CNAMERecord({ name: "www", target: "example.com" })
```

- The zone's `nameservers` output lists four hostnames under the instance
  apex. Delegating the domain to them at the registrar is what makes Skyr
  authoritative — the delegation itself is the proof of ownership. The
  volatile `status` output reports what public resolvers see: `"delegated"`,
  `"partial"`, `"wrong"`, or `"pending"`.
- Record names are relative to the apex: `"@"` for the apex, `"*"` for a
  wildcard. Available types: `ARecord`, `AAAARecord`, `CNAMERecord` (not at
  the apex), `ALIASRecord` (apex-safe server-side alias), `TXTRecord`,
  `MXRecord`, `SRVRecord`, `NSRecord` (sub-delegation), `CAARecord`.
- The AAAA record above tracks the pod's IPv6 because Skyr re-evaluates the
  config and updates the record when the pod is replaced. Anything *outside*
  Skyr should reference the stable IPv4 or the domain, never a pod IPv6.

### TLS

Skyr routes TCP to the pod; it does not terminate TLS — certificates live in
your containers. For public HTTPS the practical pattern is a
[Caddy](https://caddyserver.com) sidecar: it obtains and renews ACME
certificates automatically and reverse-proxies the app over `localhost`:

```scl
let caddyfile = Container.ephemeralVolume({
    files: #{ "Caddyfile": "example.com\n\nreverse_proxy localhost:8080\n" },
})

let pod = Container.Pod({
    name: "web",
    internetAddress: ip,
    containers: #{
        "app": { image: image.url, cpu: 500, memory: 268435456 },
        "caddy": {
            image: "caddy:2",
            cpu: 250,
            memory: 134217728,
            mounts: #{ "/etc/caddy": { volume: caddyfile, readOnly: true } },
        },
    },
})

pod.Port({ port: 80, public: true })   // ACME HTTP-01 challenge + redirect
pod.Port({ port: 443, public: true })
```

(The domain must already resolve to the pod for ACME to succeed.) The zone's
`CAARecord` can restrict which CAs may issue for the domain.

Alternatively, obtain the certificate itself declaratively with
`Skyr/PKI/ACME` and mount the PEM into whatever terminates TLS — no in-container
ACME client. DNS-01 composes with `Skyr/DNS` and is the only method that issues
wildcards:

```scl
import Skyr/PKI
import Skyr/PKI/ACME

let key = PKI.ECDSAPrivateKey({ name: "web-tls" })

let account = ACME.Account({
    name: "prod",
    directoryUrl: "https://acme-v02.api.letsencrypt.org/directory",
    contacts: ["mailto:ops@example.com"],
    agreeToTermsOfService: true,
})

let cert = account.DNS01Certificate({
    privateKeyPem: key.pem,
    domains: ["example.com", "*.example.com"],
    zone: zone,   // the DNS.Zone above — it satisfies the ChallengeZone façade
})
// cert.certificate is pending until issued (status `.issued`), so anything
// mounting cert.certificate.pem is ordered after issuance. Skyr publishes the
// challenge records, drives validation, and renews automatically with no gap.
```

For private or internal chains, `Skyr/PKI` generates keys and signs CSRs
in-config.

### Private networking

Pods reach each other privately over a `Network` — a virtual layer-2 network
with an RFC1918 CIDR (`/16`–`/30`):

```scl
let net = Container.Network({ name: "app", cidr: "10.42.0.0/24" })

let api = Container.Pod({
    name: "api",
    containers: #{ "api": { image: image.url, cpu: 500, memory: 268435456 } },
    networks: #{ "app0": net },
})

// Internal DNS: resolvable as api.app.internal by attached pods.
net.dns.ARecord({
    name: "api",
    addresses: [api.networkAddresses["app0"] ?? ""],
})
```

`networks` is keyed by interface name (not `eth0`/`lo`). Each pod gets an
inner IPv4 per attachment in `networkAddresses`, keyed the same way. Attached
pods can also open non-`public` ports to accept internal-only traffic.
Internal DNS records require the network to have a `name` and resolve as
`<record>.<netname>.internal` (`"@"` for the network apex). Traffic on a
Network never leaves the private plane and is not metered.

## Restart, jobs, and scheduled runs

There is **no Job or CronJob resource** — run-to-completion and scheduling are
per-container policy on an ordinary `Container.Pod`. Each container takes four
optional policy fields:

- **`restart`** — `.always` (default: restart on any exit — crash recovery for
  services and sidecars), `.onFailure` (restart only on a non-zero exit; a zero
  exit is final — retry-until-success), or `.never` (any exit is final).
  Restarts run in place with automatic exponential backoff. The default means
  every pod gets crash recovery for free.
- **`keepAlive`** — `Bool`, default `true`. The pod is **reaped** (node
  resources freed) once every keep-alive container has terminated. At least one
  container must keep the pod alive (else rejected at eval). A `keepAlive: false`
  sidecar is stopped when the pod reaps.
- **`maxRetries`** — `Int?`, valid only with `.onFailure`; absent means retry
  forever, `N` permits `N + 1` executions before the last failing exit is final.
- **`timeout`** — `Time.Duration?`, a per-attempt kill budget. Must be
  millisecond-valued (a multiple of `Time.second`/`Time.day`/…); a
  calendar-month duration is rejected.

A **job** is just a pod whose container uses `.onFailure`/`.never`; job-ness is
implied by configuration, never declared.

```scl
import Skyr/Container
import Std/Time

// A one-shot migration: retries until it exits 0, then the pod reaps.
let migrate = Container.Pod({
    name: "migrate",
    containers: #{ "migrate": {
        image: migrateImage.url,
        cpu: 500,
        memory: 268435456,
        restart: .onFailure,
        maxRetries: 5,
        timeout: Time.multiply(Time.minute, 10),
    } },
})
```

**Completion** surfaces as the pod's `exitCodes: #{Str: Int}` output — each
container's final exit code, keyed by name. A key is a **pending** value until
that container's final termination, so you sequence and gate with plain control
flow, keying the specific container (never folding the whole map — a `.always`
container never resolves, collapsing a fold to pending forever):

```scl
// `serve` is created only once the migration exits cleanly; while the code is
// pending the `if` condition is pending and `serve` is deferred.
let serve = if (migrate.exitCodes["migrate"] == 0)
    Container.Pod({ name: "serve", containers: #{ "web": { image: webImage.url, cpu: 1000, memory: 536870912 } } })
```

The gate lives in the `if` condition, not the pod's inputs, so `serve`'s
identity stays stable. An `else` branch handles remediation on a final non-zero
exit.

**Cron** is a job pod whose *name* embeds a tick value from `Time.now` (or
`Time.tick(interval, offset)` for an offset schedule like "04:00 UTC daily").
Each new tick is a new resource identity, so the new-named pod is created and
the previous destroyed:

```scl
let tick = Time.now(Time.hour)
Container.Pod({
    name: "hourly-{tick.epochMillis}",
    containers: #{ "report": { image: reportImage.url, cpu: 500, memory: 268435456, restart: .onFailure } },
})
```

Accepted limits: concurrency is **Replace only** (a tick boundary kills a still-
running previous pod — keep `timeout` under the interval), **missed ticks are
skipped** (no catch-up), a pod's recorded history **evaporates one window
later**, and boundaries are **UTC only**. Completion is **at-least-once**, so
**jobs must be idempotent** — a node death before completion is recorded re-runs
the job. Full reference: `curl -s https://skyr.foo/~docs/jobs.md`.

## Other capabilities

- **Volumes.** `Container.PersistentVolume({ name, size })` is region-scoped
  storage that outlives pods (min 8 MiB; a pod can only mount volumes from
  its own region). `Container.ephemeralVolume({ files, size, name })` is
  scratch space living and dying with the pod, optionally seeded with
  in-config files (see the Caddy example). Mount via a container's `mounts`
  dict keyed by absolute path: `#{ "/data": { volume: v } }`, with optional
  `readOnly`/`subPath`/`permissions`/`userId`/`groupId`. Read-only mounts of
  seeded ephemeral content update in place when the content changes; every
  other mount change replaces the pod.
- **Secrets.** Store sensitive values (passwords, tokens, TLS keys) with the
  `skyr secrets` CLI, then consume them in SCL without any plaintext in git.
  `skyr secrets set <name>` reads the value from piped stdin, a hidden prompt,
  or `--from-file` — never an argument, so it stays out of shell history;
  `skyr secrets list` shows metadata only (never values); `skyr secrets delete`
  clears a value. Values are repository-scoped by default, with
  `--environment [name]` for a per-environment override. In config, read them
  via `Std/Secret`: `Secret.get(name).qid` is an opaque reference. A pod's `env`
  and its `files` are single maps whose values are `.literal("…")` (a plain
  value) or `.secret(qid)` — pass a secret as `.secret(Secret.get(name).qid)`,
  in a container/pod `env` var or a pod `files` entry (an absolute path). The
  plaintext is resolved inside the platform at pod materialization and never
  enters git, the stored resource inputs, or any log. Because the pinned version
  is part of the pod's identity, rotating a secret (`skyr secrets set` again)
  recreates the pod to pick up the new value — rotation is a deploy, not a hot
  reload. `skyr run` can't resolve secret plaintext locally, so it rejects a pod
  with any `.secret(…)` value in `env`/`files` (a `.literal`-only pod runs);
  deploy the environment instead.
- **`Artifact.File({ name, contents, mediaType })`** stores a file and
  returns a time-limited download `url` — useful for exposing generated
  config, reports, or deployment metadata.
- **`HTTP.Get({ url, headers })`** performs a GET as a resource (volatile;
  re-performed on input changes) and outputs `status`, `body`, `headers` —
  a probe or a way to pull external data into the config.
- **`PKI.*`** — `ED25519PrivateKey`/`ECDSAPrivateKey`/`RSAPrivateKey`,
  `CertificationRequest`, `CertificateSignature`: build self-managed
  certificate chains (CAs, client certs, internal TLS) entirely in-config.
- **`PKI/ACME.*`** — publicly-trusted TLS certificates over ACME (RFC 8555, the
  Let's Encrypt protocol). An `Account` yields `DNS01Certificate` (composes with
  `Skyr/DNS`, does wildcards) and `HTTP01Certificate` (you serve the token)
  sub-constructors; Skyr drives order → challenge → issue → renew with no
  imperative steps and renews with no gap.
- **`Random.Int({ name, min, max })`** — a random value minted once and then
  stable across deploys.
- **`IAM.Role`/`IAM.Policy`** — org-scoped authorization: a `Role` is an
  empty subject; a `Policy` grants `subjects` (role QIDs) `verbs` over
  `objects`, all `*`-pattern matchers, default-deny. A repo's deployments
  act as its deployment role (default `Super`).
- **`Rollout.Group({ name, members })`** — groups the resources created
  inside its `members` closure under one parent resource, giving them a
  shared handle in the resource tree.
- **`Resource.Definition<T>("Name")`** — define your *own* resource type
  whose backend is SCL. One repo declares a typed definition and exports its
  `.Resource` constructor; other repos in the org import the module and call
  the constructor to register instances; the defining repo reads every live
  instance back as `def.instances: [T]` and folds them into its own resources
  (e.g. one repo collects ingress rules that many app repos declare). Instances
  are content-addressed by payload and write-only (no return channel to the
  caller in v1), and the collected set refreshes automatically — so a
  definition is volatile and its deployment never settles into `Up`. A consumer
  scopes to a definition through its `Package.scle` dependency pin; pin it to a
  branch or tag, not a commit hash.
- **`Skyr/AWS/*`** — resources in *your own* AWS account (billed by AWS, not
  Skyr), surfacing the Terraform AWS provider as one module per AWS service
  (`Skyr/AWS/S3`, `Skyr/AWS/DynamoDB`, …); import only the services you use.
  Each service module exports `provider(config)` — bind it once with `region`
  plus `accessKey`/`secretKey` as `.literal(…)`/`.secret(qid)` values (same
  secret machinery as pod `env`), then call constructors off the bound
  provider: `s3.Bucket("skyr-name", { …inputs })`. The first positional
  argument is the Skyr identity (renaming = destroy + create), and inputs the
  provider deems non-updatable also replace on change, mirroring Terraform's
  requires-replace. Field names come from the provider's own schema — trust
  editor completions/hover over guessing. (`HashiCorp/Random` is the same
  machinery wired only into the local dev harness.)

Exact inputs/outputs for all of these live in the stdlib reference — look
them up rather than guessing (see below).

## Watching a rollout

```sh
skyr deployments list             # states of this repo's deployments
skyr deployments logs --follow    # stream deployment progress in real time
skyr resources list               # resources in the current env
skyr resources list --env staging
skyr resources logs <resource-qid>
```

A rollout is done when the deployment reports converged — state `Up`, or
`Desired` with all resources settled if anything is volatile (see the
lifecycle section). `skyr resources list` shows per-resource state; JSON
output (`--format json`) is the reliable way to script against either.

To reach a not-publicly-exposed port from the local machine, forward it over
SSH (find the port resource's QID via `skyr resources list`):

```sh
skyr port-forward acme/shop::main::stockholm:Skyr/Container.Pod.Port:web-1a2b3c:8080/tcp 8080
```

## When a rollout goes wrong

Failures surface as **incidents** — durable records that open when something
is wrong long enough to matter, close automatically on recovery, and email
every member of the owning org on both events. Removing the offending
resource from the config also counts as recovery. The category names the
user-visible consequence:

| Category | Meaning |
|---|---|
| `BadConfiguration` | Skyr refuses to roll out config it determined invalid. The system works; the config doesn't. |
| `CannotProgress` | The entity is stable, but something derived/dependent could not be applied. |
| `InconsistentState` | Reality drifted from config and reconciliation can't close the gap. |
| `Crash` | The thing is misbehaving with user-visible downtime. |

An **uncaught SCL exception** during evaluation — reading a secret that isn't
set, unwrapping a `nil`, any top-level `raise` your config doesn't `catch` —
opens a `CannotProgress` incident carrying the exception's message, so a missing
input surfaces on the deployment instead of silently stalling.

Incidents are listed on the website at `/<org>/~i` (the CLI doesn't surface
them). For diagnosis: `skyr deployments logs` shows evaluation and rollout
errors (a `BadConfiguration` usually reproduces locally with `skyr check`);
`skyr resources logs <qid>` shows a specific resource, including container
output from pods.

## Looking up documentation

The instance serves its docs as raw markdown, ideal for fetching and
grepping:

```sh
curl -s https://skyr.foo/llms.txt                    # index of all doc pages
curl -s https://skyr.foo/~docs/deployments.md        # lifecycle, supersession, ownership in depth
curl -s https://skyr.foo/~docs/jobs.md               # restart policy, jobs, cron-style scheduling
curl -s https://skyr.foo/~docs/status.md             # health, incidents, notifications
curl -s https://skyr.foo/~docs/secrets.md            # secrets: scopes, CLI, consumption, IAM
curl -s https://skyr.foo/~docs/deletion.md           # deleting repos, orgs, and accounts
curl -s https://skyr.foo/~docs/scl/stdlib.md         # every Std/* and Skyr/* signature
curl -s https://skyr.foo/~docs/scl/stdlib.md | grep -n -A 40 '^### Container.Pod'
```
