---
name: deploy
description: >-
    How deployment to Skyr works: pushing to the skyr git remote, environments
    and the deployment lifecycle, rolling a deployment back, exposing pods to
    the internet (ports, InternetAddress, DNS zones), private networking and
    routing between networks, sharing networks, volumes and addresses across
    repositories, first-party plugin capabilities, checking rollout status and
    incidents, and reaching into a running pod (port forwarding, running a
    command or a shell in a container). Use when deploying to Skyr, wiring a
    repository to a Skyr instance, exposing a service publicly, connecting
    private networks, rolling back a bad deployment, debugging a rollout, or
    getting a shell inside a deployed container.
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
- **An edge comes from using another resource's outputs** — either building a
  resource's inputs from them, or gating its declaration on them
  (`if (job.exitCodes["job"] == 0) Container.Pod({...})`). There is no
  `depends_on`; both kinds are worked out for you, and SCL's
  `with (a) B({...})` writes the edge explicitly where neither flow carries it
  (the `scl` skill covers it). Edges order creation and,
  reversed, teardown: nothing is destroyed while something depending on it is
  still there.
- Deleting a ref tears the environment down:
  `git push skyr --delete feature-x` destroys everything that environment
  owns, in dependency order.
- **Rolling back is a first-class operation**, not a git manoeuvre: every
  deployment records which one preceded it, and `skyr deployments rollback <env>`
  redeploys that predecessor's commit as a *new* deployment. A deployment with
  no recorded predecessor is inert, and rolling *it* back tears the environment
  down — see the rollback section below before running it.
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
  registered key, with your username as the SSH user — the scp-style
  remote that `skyr repo create` prints and the repository's web UI page
  shows (an `ssh://<user>@<host>:<port>/<org>/<repo>` URL instead when the
  instance's SSH endpoint is on a non-default port, which scp-style syntax
  cannot carry):

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
  rolls out and adopts shared resources. It keeps serving until the successor
  converges, and *not converged* includes a resource the successor cannot yet
  **declare** — one whose inputs are still pending — not just one it is still
  creating. Being unable to state a resource is not the same as no longer
  wanting it, so the predecessor keeps it alive rather than the successor
  converging around the gap. The deployment log names each such resource
  (`Waiting to declare <resource>: one of its inputs is still pending`, once per
  stuck declaration, and only on a pass with nothing else in flight — while
  resources are being created, a declaration waiting its turn is ordinary
  ordering). Where Skyr can tell which read has not resolved, the line ends by
  naming it (`… (waiting on <resource>)`) — the resource to go look at. Where it
  cannot, the line falls back to listing everything the declaration reads
  (`… (it reads <resource>, <resource>)`), which is a hedge rather than a
  verdict: the culprit is one of them. A resource whose *name* is itself
  derived from another resource's output holds the rollout the same way, and is
  named by type while it waits
  (`Waiting to declare a <type> resource: its own identity is still pending`).
  A **branch** the successor cannot decide holds the rollout the same way, and
  for the sharper version of the same reason: a resource declared inside an
  `if` whose condition is still pending is never even attempted, so there is no
  declaration to name. The log says where the program stopped short instead
  (`Waiting at <construct> in <module> (<span>): a pending value settled
  it, so a region that could declare resources went unexplored`), naming the
  reads it is stuck behind on the same terms as a declaration's line. One
  unreadable value is one line however many regions it kept the pass out of — a
  gate in a frontend, the `if` inside the helper that gate calls — because they
  resolve together. A branch that could not have declared anything is not a
  hold at all: picking between two strings leaves nothing undeclared, so a
  `?? pending` fence over a value you only interpolate is silent. Both rules
  cover every region a pending value skips rather than decides, not only a
  branch: an operator's right-hand side, an index expression, and the parts of
  an interpolated string after a pending one are held when they declare and
  silent when they do not.
  A declaration whose reads have settled is a dead end — nothing the
  deployment is doing can resolve it — and says so at warning severity
  whatever else is going on (`Stuck declaring <resource>: … and nothing in
  flight can resolve it`; `Stuck at <construct> in <module> …` for a branch),
  followed once by what the hold costs: while it
  lasts, resources removed from the program are kept rather than destroyed.
  Changing the program and pushing is the way out; the deployment keeps
  checking and repairing what it already owns in the meantime, and opens no
  incident. A stall is a **hold**, not a failure, and it is carried on the
  deployment as well as in the log — `skyr deployments list` counts holds in a
  `HELD` column and spells each one out beneath the table, and the API answers
  them on `Deployment.status.holds`. That is the surface to check when the log
  has scrolled past the moment the hold began. The same list carries the other
  waits that leave a healthy deployment unfinished: an unset, defaultless
  knob; a read of a resource another deployment has yet to create or update;
  and an unset knob in another deployment's environment, which waits on a
  person you may not be. Each names who is expected to clear it, and only the
  stalled declaration has nobody. The dead-end test asks the narrower set
  wherever Skyr has one: a declaration waiting on a read that is already final
  is stuck even while unrelated work is in flight. Where it cannot say which
  read it waits on, every resource the declaration reads has to have settled
  before the line is earned. **Do not expect the `Stuck declaring` line for
  every dead end**: a declaration deferred on a pending that reaches it with
  no resource behind it at all — a bare `Std/Plugin.pending` a frontend handed
  over without joining it to a read — keeps the ordinary `Waiting to declare`
  line, because Skyr cannot tell that apart from a value still on its way. A
  rollout that is not converging with only waiting lines in the log is that
  case; read the program rather than waiting for a warning.
- **Undesired** — teardown: resources not adopted by the successor are
  destroyed in dependency order.
- **Down** — nothing left; terminal.

**A commit's own tests run before anything is applied.** When the repository
declares any (the `scl` skill covers writing them), Skyr runs them once for the
commit — ahead of evaluating `Main.scl` at all, so a commit whose tests fail
creates, updates and destroys nothing. On a push that means the deployment it
would have replaced keeps serving untouched: the successor never adopts a
resource and never starts the predecessor's teardown, so the old one sits in
Lingering. On a fresh environment it means nothing is created. The blocked
deployment stays **Desired** and retries on the ordinary backoff, still
maintaining the resources it already owns — blocking withholds new work, never
maintenance — until a commit whose tests pass supersedes it. `skyr test` runs
the same cases locally and is what predicts the verdict; a repository with no
test code is unaffected.

Two things legitimately keep a deployment in **Desired forever** — that is
normal operation, not a stuck rollout:

- Any **volatile resource** (a `Container.Pod`, `DNS.Zone`, or `HTTP.Get`
  represents external state that can drift, so Skyr keeps reconciling).
- A **branch or tag pin** in `Package.scle` dependencies: the deployment
  keeps following the foreign ref. Pin commit hashes to settle.

A deployment of only stable resources (keys, random values, artifacts,
images) settles into Up.

## Rolling back

Every deployment Skyr promotes records its **rollback target**: the deployment
that was current immediately before it. A rollback redeploys that target's
commit as a **fresh deployment** — no old deployment is revived — through the
same compile → test → evaluate → converge path as any push.

```sh
skyr deployments rollback main
skyr deployments rollback main --reason "checkout error rate spiked after 3f2a1c"
skyr deployments rollback main --reason-file ./why.txt   # `-` reads stdin
```

- **It is ordinary safe supersession.** The deployment being rolled back goes
  Lingering and **keeps serving** until the rollback deployment bootstraps. If
  the restored commit no longer compiles or its tests now fail, nothing is torn
  down and the serving deployment carries on. Pushing again recovers.
- **Rollbacks compose.** A rollback result takes its *commit* from the target but
  inherits the target's *own* target, so repeated rollbacks walk back one
  deployment at a time. Deployed `A -> B -> C`: rolling back `C` gives `B'`
  (whose target is `A`), rolling back `B'` gives `A'` (no target).
- **A deployment with no target tears the environment down — the one dangerous
  outcome here.** It does not fail and does not no-op: it destroys every resource
  in the environment, exactly as `git push skyr --delete` would. Three kinds of
  deployment have no target: an
  environment's first one, a rollback result that restored such a first one, and
  **any deployment created before the instance recorded rollback lineage** (it is
  never backfilled, so one ordinary push is what gives such an environment a
  target again). Check before acting: `skyr deployments list` prints
  `none (teardown)` in `RESTORES` for an inert deployment. The CLI prints the
  consequence and then makes you type the environment's QID for that outcome
  (`--yes` states the intent up front, and is **required** with no terminal to
  prompt on — a piped stdin, or one already spent on `--reason-file -`). The
  restoring outcome is deliberately ungated. The git ref survives a teardown, so
  pushing re-creates the environment.
- **The request binds to one deployment.** The command takes an environment but
  resolves its current deployment and submits that exact one, so a push landing
  in between makes the rollback **refuse** and name what is current now. Retries
  are safe: the plan is recorded durably before anything happens, so a repeat, a
  crash, or a racing request all resolve to that same rollback.
- **Reasons** are kept verbatim as permanent provenance, newlines included: at
  least one non-whitespace character, at most 512 characters. `--reason` and
  `--reason-file` conflict. Omitted, the recorded default names the asker —
  `Manual rollback` for a person, `Rollback condition reached` for
  `Rollout.Rollback`.
- **Permission** is `environment:Rollback` on the environment, covering both
  outcomes, teardown included. It implies neither push nor delete authority and
  is never anonymous. A rollback from config runs on the *deployment role*, which
  must be granted the verb explicitly.
- **History** answers all of it and `skyr deployments list` prints it: `RESTORES`
  (target, or `none (teardown)`), `ROLLED BACK` (`-> <id>` or `torn down`),
  `ROLLBACK OF` (this row is itself a rollback result), plus each reason verbatim
  beneath the table. Two reading rules: `RESTORES` is **lineage, not an offer** —
  it is filled in for Lingering and Down rows too, and only the *current*
  deployment can actually be rolled back; and a successor id is reserved before
  its row exists, so a reference to it can briefly resolve to nothing and is
  expected to appear on the next poll.

**From the config itself.** `Rollout.Rollback` is the same operation as a
resource, so a deployment can roll *itself* back:

```scl
import Skyr/Rollout

// Whatever your code decides from — a knob, a version, or a pod's own
// `health` output (`.degraded`/`.failing` once that pod has worked at least
// once; pending before then, so a condition over it simply waits).
if (degraded)
    Rollout.Rollback({ reason: "Health check remained degraded" })
else
    nil
```

Declaring it **is** the request; there is nothing to read off it, and whether to
roll back is ordinary control flow rather than an input. The record is required
even when empty — `Rollout.Rollback({})`; `Rollout.Rollback()` does not compile,
since SCL has no omittable arguments. Identity is *this deployment plus the
reason* and nothing else, so: the same reason twice in one deployment is one
request; two different reasons are two requests that the platform serializes to
one recorded rollback; a reconcile retry asks for the same one; and the same
reason in a *later* deployment is a new request — a condition still true after
the rollback rolls back again, one step further. Every attempted reason lands in
the deployment log even when it loses the race or is refused. The deployment role
needs `environment:Rollback`, granted in the repository's own `IAM.Policy`.

**This is the surface with no interlock**, unlike the CLI and the web dialog: it
asks nobody and runs on a reconciliation pass, so if the declaring deployment has
no rollback target the declaration tears the environment down unattended. Check
the environment's `RESTORES` before wiring one up, and note that each rollback
leaves a deployment whose own target is one step further back — a condition still
true afterwards walks the lineage down to the inert one.

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
(`#{ "name": { ... } }`); the key names the container in logs and, for
containers that can finish, in the pod's `exitCodes` output. `cpu` and
`memory` are **required** on every container — both are hard limits. Containers in the same pod share a network namespace, so
siblings reach each other on `localhost`.

**Running something other than the image's default.** A container starts its
image's own `ENTRYPOINT`/`CMD` unless it says otherwise. The optional
`execute` field replaces exactly one of those two halves — `.arguments([…])`
keeps the entrypoint and replaces the `CMD` (the flag-passing case, which needs
an image that *has* an `ENTRYPOINT`: on a `CMD`-only image the first argument is
exec'd as the program and the container dies), `.command([…])` replaces the
entrypoint and drops the `CMD` (the whole command line — one image serving both
a service and a migration, and the right choice for a `CMD`-only image). Nothing parses a
shell, so a pipeline is `.command(["/bin/sh", "-c", "a | b"])`, and an empty
argv is rejected. An argv takes no `.secret(…)` and is stored, logged and
hashed verbatim — pass credentials through `env`. `workingDirectory: Str?`
replaces the image's `WORKDIR` and must be an **absolute** path; a tenant
container's rootfs is **read-only** (unless it sets `privileged: true` — see
Privilege under Other capabilities), so the directory must exist in the image
or be one of its mount paths. Both are part of the pod's identity, so changing
either recreates the pod.

```scl
containers: #{ "migrate": {
    image: image.url,        // the same image the service runs
    cpu: 500,
    memory: 268435456,
    execute: .command(["/app/bin/migrate", "--yes"]),
    workingDirectory: "/app",
    restart: .onFailure,
} }
```

**Narrowing the build context.** Everything under `context` goes to the
builder, so a `COPY . .` bakes in whatever is lying around — `node_modules`, a
stray `.env`, a local build directory. The optional `ignorefile` input excludes
paths, and like `containerfile` it takes the file's *content*, not a path to
it: `ignorefile: Path.read(./app/.containerignore)`, or the patterns written
inline. The syntax is Docker's and podman's, not gitignore's — one pattern per
line, `#` comments, anchored at the context root (`node_modules` excludes the
top-level entry only, `**/node_modules` excludes it at any depth), `*` and `?`
never crossing a `/`, and a leading `!` negating with the *last* matching
pattern deciding (`*` followed by `!dist` ships `dist/` and nothing else).
There is no filename convention: a `.containerignore` sitting in the context is
an ordinary file until something passes it as this input, so the patterns may
live anywhere or come from any file. Excluded files never reach the builder at
all, and a line that isn't a valid pattern fails the build naming it.

### Open ports — how ingress works

Every pod gets a **public, internet-routable IPv6** (`pod.address`). By
default it is inert for ingress: the pod's firewall only accepts connections
from internal networks the pod is attached to, and the IPv6 serves egress.
Until the backend has allocated it, `pod.address` reads as pending, so
resources built from it defer. Opening a port changes that:

```scl
let http = pod.Port({ port: 8080, public: true })
// http.address is "[<pod-ipv6>]:8080". Until the pod's address has been
// allocated it reads as pending, so resources built from it defer.
```

`public: true` opens the port to the internet — both on the pod's IPv6 and
on any bound IPv4 address (next section). Without it the port only accepts
traffic from attached internal networks.

`protocols` defaults to `[.tcp]`; `.udp` is the other member. One call is one
opening, so a service that shares a port number across transports names them
together rather than declaring the port twice:

```scl
pod.Port({ port: 443, protocols: [.tcp, .udp], public: true })  // HTTP/2 + HTTP/3
pod.Port({ port: 53, protocols: [.tcp, .udp] })                 // DNS
```

Order and repeats don't matter — the opening is the set — and an empty list is
refused rather than treated as the default.

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
Like `pod.address` it reads as pending until the allocation lands.
An address routes to **exactly one pod at a time, anywhere in Skyr** — two
pods in one program naming it is an eval-time error, and a pod elsewhere
trying to take an address another pod already holds fails the deploy, naming
the holder. (Replacing a bound pod is fine: the successor in the same
environment takes the claim over.) The claim is released when the bound pod
stops naming the address or is deleted. Binding needs
`resource:BindInternetAddress` on the address, and the pod must be in the
address's region. The pod's firewall still applies: only `public: true` ports
accept internet traffic on the address.

**Serve public traffic on IPv6.** A pod's public entry point is its IPv6
address, and a bound `InternetAddress` (public IPv4) is delivered to the pod
translated to that IPv6. So a workload that must be reachable from the internet
has to listen on IPv6 — bind `[::]` (or dual-stack), not only `0.0.0.0`. A
process listening on IPv4 `0.0.0.0` alone will not receive public traffic,
including traffic to a bound `InternetAddress`.

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
  volatile `status` output reports what public resolvers see as an enum
  atom: `.delegated`, `.partial`, `.wrong`, or `.pending`.
- Record names are relative to the apex: `"@"` for the apex, `"*"` for a
  wildcard. Available types: `ARecord`, `AAAARecord`, `CNAMERecord` (not at
  the apex), `ALIASRecord` (apex-safe server-side alias), `TXTRecord`,
  `MXRecord`, `SRVRecord`, `NSRecord` (sub-delegation), `CAARecord`.
- Bad inputs raise `DNS.InvalidDnsInput` at evaluation time: a name outside the
  grammar, a CNAME or NS at the apex, a CAA `flags` outside 0–255, and a
  `ttl`/`defaultTtl` that is not a whole number of seconds from 1 to 2147483647
  — which rules out anything sub-second or fractional, and anything past
  roughly 68 years. Use a multiple of `Time.second`, `Time.minute`, `Time.hour`
  or `Time.day`. The raise fails at the offending call rather than at the
  plugin; `skyr run` surfaces it locally, and `skyr check` does not evaluate, so
  it does not. A calendar-month span (`Time.month`, `Time.year`) is refused
  earlier still: a TTL is a `Time.Duration`, a fixed length, and a calendar span
  is a `Time.CalendarDuration` — a type error, which `skyr check` does report.
  Records default to the zone's `defaultTtl`, itself 5 minutes unless set.
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
    files: #{ "Caddyfile": .literal("example.com\n\nreverse_proxy localhost:8080\n") },
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
    privateKey: key.pem,   // the sealed key's Secret Version QID, never a PEM
    domains: ["example.com", "*.example.com"],
    zone: zone,   // the DNS.Zone above — it satisfies the ChallengeZone façade
})
// cert.certificate is pending until issued (status `.issued`), so anything
// mounting cert.certificate.pem is ordered after issuance. Skyr publishes the
// challenge records, drives validation, and renews automatically with no gap.
```

`directoryUrl` must be an `https` CA on the public internet: a deployed
account refuses a loopback, private, link-local, or otherwise non-public
directory (an internal CA is reachable only under `skyr run`, which hosts the
plugin on your own machine).
The same restriction applies to the hosts the challenge self-check reaches, so
an HTTP-01 domain and a DNS-01 zone's nameservers have to resolve publicly —
but there it shows up as a certificate that waits in its challenge state
rather than one that fails, since an unreachable host is indistinguishable
from a challenge that is not in place yet.

Certificates/chains are public PEM outputs; the private key stays sealed in
the secrets vault — deliver it by seeding an ephemeral volume with
`.secret(key.pem)` and mounting that volume into whatever terminates TLS:

```scl
let tls = Container.ephemeralVolume({
    files: #{
        "tls.crt": .literal("{cert.certificate.pem}{cert.certificate.chainPem}"),
        "tls.key": .secret(key.pem),
    },
})

// …inside the terminating container: reads /run/tls/tls.{crt,key}
mounts: #{ "/run/tls": { volume: tls, readOnly: true, userId: 1000 } }
```

A volume carrying any `.secret` is mounted owner-only (`0700` root dir, `0600`
files — an explicit `permissions` on the mount overrides the mode) owned by
the mount's `userId`/`groupId`, defaulting to root — so name the uid the
image's `USER` runs as, or the process cannot read its own key.
Grants are explicit: the deployment role needs
`secret:View`/`Write`/`Delete` on the key's and the ACME account's
resource-scoped secrets (an `IAM.Policy` with wildcard objects like
`"<org>/<repo>::*"` covers them — same stanza as the Secrets bullet below).

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
inner IPv4 per attachment in `networkAddresses`, keyed the same way; every
interface you attached is present, reading as pending until its address is
allocated, so the record above defers instead of publishing an empty one.
Attached pods can also open non-`public` ports to accept internal-only traffic.
Internal DNS records require the network to have a `name` and resolve as
`<record>.<netname>.internal` (`"@"` for the network apex). A pod may not
attach two networks that share a `name` — the lookup would be ambiguous, so
it's rejected at eval. Traffic on a Network never leaves the private plane and
is not metered.

An internal DNS name belongs to whoever publishes it **first**: a name another
environment already holds on the same network is not overwritten — the deploy
fails, naming the holder — and only the holder can change or remove its own
record. *Removing* it always works, even after the holder's grant is
withdrawn; *changing* it goes through the grant. Attaching needs
`resource:AttachPodToNetwork` on the network; publishing needs the separate
`resource:AttachDnsRecordToNetwork`.

### Routing between networks

A `Network` on its own is a closed island. `Container.Router` connects several
of them and routes between them, with an optional stateful ACL. There is no
peering resource — a router is *a machine on each LAN*:

```scl
let corp = Container.Network({ name: "corp", cidr: "10.42.0.0/24" })
let dmz = Container.Network({ name: "dmz", cidr: "10.43.0.0/24" })

Container.Router({
    name: "edge",
    networks: #{ "corp": corp, "dmz": dmz },
    defaultAction: .deny,
    rules: [
        { source: corp.cidr, destination: dmz.cidr, action: .allow },
    ],
})
```

- **Real addresses, no NAT.** Pods reach peer-network pods at their own inner
  IPv4s and keep exactly the interfaces they declared. A deployed router draws
  an ordinary address on each member network — reported in `addresses`, keyed by
  the member labels, and answering `ping` and nothing else. A router's
  `networks` keys are **labels**, not interface names: no interface, no DNS
  meaning.
- **Member CIDRs must be disjoint.** Overlapping members are an eval error; a
  member whose address space is already routed in the closure it joins fails the
  deploy, naming **the CIDR and your own member label** (never the foreign
  network). Two deploys racing into an overlap resolve deterministically — the
  earlier-established membership wins and the loser's space is routed nowhere.
- **Any member count is valid**, zero and one included (a useful intermediate
  state while a handle or grant is pending). Members **may live in different
  regions**; a router takes no `region` input.
- **ACL:** `rules` match top to bottom, **first match wins**, falling through to
  `defaultAction` (`.allow` by default; omit both and everything routes).
  **Stateful** — replies on established connections always pass, so rules govern
  who may *initiate*. `source`/`destination` are CIDR strings matched on real
  addresses (`"0.0.0.0/0"` = anything), and transit passing through is filtered
  by the same rules. The router is the only filter — there's no second, per-pod
  rule set to keep in step. It governs **access, not exposure**: denied traffic
  still crosses the private network before it's dropped at the destination's
  boundary, so an ACL is not a shield against volume.
- **Chaining is transitive.** Two routers sharing a network route through each
  other, so joining a router gets you everything it routes, including onward
  routers — reachability exists only where a chain creates it. Source addresses
  can't be forged across a router; there's nothing to configure.
- **Updates are in place.** Changing `networks`, `rules` or `defaultAction`
  reconverges the router on every node within seconds — surviving members keep
  their addresses and no pod is recreated; only a new `name` is a new router.
  The router is rebuilt as it reconverges, so expect a brief window where it
  forwards nothing. That makes the ACL the router owner's prompt kill switch.
- **DNS follows routability, not the ACL.** A pod also resolves
  `<record>.<netname>.internal` on named networks it can *reach* — even through
  a deny-all router (a name isn't traffic). Routed names are **qualified only**
  (search domains stay the attached netnames), direct attachment shadows
  routing, and two routed networks sharing a netname resolve to **NXDOMAIN**.
  Give the networks of a shared fabric distinct names.
- Membership needs `resource:AttachRouterToNetwork` on each member network,
  checked on every transition of the router.

**Cross-org: the DMZ pattern.** Never make your network a member of the other
org's router. Org A declares a small transit network T and grants org B's
deployment role `AttachRouterToNetwork` on T — the *only* cross-org grant. Each
side then connects with a router it **owns** (A: its network + T; B: its network
+ T), and chaining carries traffic between them. Each side's ACL sits on its own
router, so each keeps a kill switch that converges in seconds. Revoking the
grant is lazy and *freezes* rather than evicts (the router's next transition
fails; established routes stay), and a network owner who doesn't own the
adjacent router has no lever short of deleting the network — which is exactly
why each side brings its own router.

**Under `skyr run`** routers are emulated: same reachability, same ACL, same DNS
closure, and member networks are isolated from each other absent a router. Four
local-only divergences, none of which a deployment produces: every local pod
also shares the `skyr-run-default` network (so isolation is faithful for
member-network addresses, not for the shared ones behind them); DNS records are
injected at pod creation while routes converge live, so a running local pod can
hold a route to a network whose names it can't yet resolve; a local router's
address on a member network is that network's gateway (the reserved first host)
rather than a pool address, which is what `addresses` reports locally; and those
addresses answer more than `ping`, since they are the bridge gateways local name
resolution and internet egress already go through.

## Sharing networks, volumes, and addresses across repos

Networks, persistent volumes, and internet addresses are shareable across
repositories: the owner exports the handle its constructor returned, the
consumer imports it via a `Package.scle` dependency (the `scl` skill covers
imports). The identity strings on those handles are opaque and carry their
owner — guessing one is not a route in, and a hand-assembled handle is refused
where a pod attaches, mounts, or binds it.

Using a shared resource needs the read chain (`repository:View` +
`environment:View` + `resource:View`) **plus** the use verb, whose object is
the used resource's full QID. One policy in the *owner's* config covers it:

```scl
IAM.Policy({
    name: "app-uses-platform",
    subjects: ["acme/app::*:Skyr/IAM.Role:app-deployer"],
    verbs: ["repository:View", "environment:View", "resource:View",
            "resource:AttachPodToNetwork", "resource:MountVolume"],
    objects: ["acme/platform", "acme/platform::main",
              "acme/platform::*:Skyr/Container.Network:*",
              "acme/platform::*:Skyr/Container.PersistentVolume:uploads"],
})
```

The five use verbs are `resource:AttachPodToNetwork`,
`resource:AttachRouterToNetwork`, `resource:AttachDnsRecordToNetwork`,
`resource:MountVolume`, and `resource:BindInternetAddress`. (Two more verbs sit
outside the standard family without being use verbs at all:
`resource:ForwardPodPort` and `resource:ExecInPod` are checked when an
*operator* reaches into a running pod, not at any transition — see "Watching a
rollout".) Enforcement is
**uniform** — your own environment's resources are checked too; it's invisible
only because a repo's deployment role defaults to the org's `Super` role, which
short-circuits within that org. A restricted role needs the grants for its own
resources as well. A refusal names the verb and the object QID, plus the acting
role when the deployment presented one.

Revocation is **lazy**: it bites on the holder's next transition, never
detaching a running pod. There is no owner-initiated eviction, so a resource
you share can be held by its current user until they release it — for a shared
`InternetAddress` the owner's only forced remedy is delete + recreate, which
changes the public IP; for a network someone else's router routes, the prompt
lever belongs to whoever owns that router (its ACL), which is why cross-org
connections are shaped so each side owns the router in front of its own
network. Dropping an attachment/membership/mount/binding always succeeds, even
after the grant is gone.

## Restart, jobs, and scheduled runs

There is **no Job or CronJob resource** — run-to-completion and scheduling are
per-container policy on an ordinary `Container.Pod`. Each container takes five
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
- **`timeout`** — `Time.Duration?`, a per-attempt kill budget. A
  `Time.Duration` is a fixed length (a multiple of `Time.second`/`Time.day`/…);
  a calendar-month span is a `Time.CalendarDuration` and does not type-check
  here.
- **`probe`** — `Probe?`, absent by default: how Skyr asks a container whether
  it is still *serving*, for a workload that can stop serving without exiting.
  `{ kind: .http({ port, path? }) }` (2xx/3xx passes) or
  `{ kind: .tcp({ port }) }` (a completed connection passes), plus optional
  `initialDelay`/`interval`/`timeout`/`startupWindow` (each a `Time.Duration`,
  so a fixed length, and positive) and `failureThreshold` (a positive count),
  each defaulted by Skyr when omitted. The check runs *from inside the pod*, so the
  port need not be one `pod.Port` opened — and usually should not be. **One
  probe covers startup, readiness and liveness, because its role changes with
  the container's phase**: before its first success it is readiness (a failure
  kills nothing, so a slow start is safe; a container still failing after
  `startupWindow` is restarted as hung), after it is liveness
  (`failureThreshold` failures in a row stop the container). Its first success
  is also what the pod's readiness waits for, which is what a `ReplicaSet`
  replica built from the pod stands on. A probe
  never restarts anything by a route of its own — it stops the container and
  `restart` decides, so a workload that hangs sick gets the same backoff,
  retry cap and crash-loop judgment as one that exits sick. Adding, changing
  or removing a probe **recreates the pod**. `skyr run` accepts one and does
  not execute it.

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

**Completion** surfaces as the pod's `exitCodes: #{Str: Int}` output — the
final exit code of each container that **can reach one**, keyed by name. Only
`.onFailure` and `.never` containers are keyed; a `.always` one restarts
forever and has **no key**, so reading it yields `nil`. A key that exists is a
**pending** value until that container's final termination, so you sequence and
gate with plain control flow, keying the specific container:

```scl
// `serve` is created only once the migration exits cleanly; while the code is
// pending the `if` condition is pending, neither branch runs, and the pass says
// so — it is a branch that could have declared and did not, which holds the
// rollout exactly as a resource with a pending input does.
let serve = if (migrate.exitCodes["migrate"] == 0)
    Container.Pod({ name: "serve", containers: #{ "web": { image: webImage.url, cpu: 1000, memory: 536870912 } } })
```

The gate lives in the `if` condition, not the pod's inputs, so `serve`'s
identity stays stable — and gating on `migrate` is an edge all the same, so
teardown destroys `serve` first and `migrate` outlives it. An `else` branch
handles remediation on a final non-zero exit.

Two consequences of unkeyed `.always` containers, both quiet: a comparison
against a missing key is `nil == 0` — **false immediately**, taking the `else`
branch rather than waiting, so a gate naming the wrong container looks like a
gate whose condition is merely false (compare against `nil` when you mean "has
it terminated"); and a fold over the whole map no longer wedges on pending,
but over a pod of nothing but `.always` containers it folds an **empty** map,
where `List.all` is vacuously `true`. Key the container you mean.

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
  `files: #{ "<path relative to the mount root>": .literal("…") }` — values are
  the same two arms as `env` (`.literal(…)`/`.secret(qid)`), and a bare string
  is a type error (see the Caddy example). **Mounting a seeded volume is how
  files reach a container** — there is no pod-level file input. Mount via a
  container's `mounts` dict keyed by absolute path:
  `#{ "/data": { volume: v } }`, with optional
  `readOnly`/`subPath`/`permissions`/`userId`/`groupId`; two containers mounting
  the same ephemeral volume must spell the same permissions and owner.
  Read-only mounts of seeded ephemeral content update in place when the content
  changes, unless the new content outgrows the disk the pod claimed (each
  non-empty file rounds up to a whole 4 KiB page, so adding one to a volume
  that already holds one does) — that replaces the pod, as does every other
  mount change: the volume's name or size, which mounts share it, and a
  writable mount's seed. A persistent volume's identity includes
  its owning environment, so another repository's volume of the same name is a
  different volume with its own data; mounting one needs `resource:MountVolume`
  on it (see the sharing section above).
- **Privilege.** A container runs confined by default — a **read-only** root
  filesystem, the default seccomp profile, and no added Linux capabilities.
  `privileged: true` on a container drops that confinement for that one
  container: the full capability set, no seccomp filter, a **writable** root
  filesystem, and access to the guest's `/dev` — `/dev/kvm` included, so a
  nested hypervisor (an L3 guest) runs. It is safe to offer because a container
  is alone in its pod's own VM, so the worst it can reach is that VM, which is
  already the tenant's; there are no finer knobs (capability lists, sysctls) —
  the boolean is honest about what it grants. Changing it **recreates** the pod;
  leaving it out or spelling `false` is exactly the confined default and does
  not rename or recreate a pod that never asked for privilege. A writable root
  is scratch space, not storage: root writes land in a guest tmpfs backed by the
  pod's RAM and **charged to the container's `memory`**, so an overwrite-heavy
  workload is OOM-killed like any memory-hungry one, and the writable layer has
  a hard size ceiling the node sets from the machine's memory (an eighth of it,
  clamped 16–256 MiB) — a write past it fails with `ENOSPC`. That is a different
  failure mode from the confined default, which answers a root write with
  `EROFS`. Declare a `Mount` for anything that must persist or grow.
- **Secrets.** Store sensitive values (passwords, tokens, TLS keys) with the
  `skyr secrets` CLI, then consume them in SCL without any plaintext in git.
  `skyr secrets set <name>` reads the value from piped stdin, a hidden prompt,
  or `--from-file` — never an argument, so it stays out of shell history;
  `skyr secrets list` shows metadata only (never values); `skyr secrets delete`
  clears a value. Values are repository-scoped by default, with
  `--environment [name]` for a per-environment override.
  `skyr secrets list --environment <env>` lists that environment's whole
  **inventory** — repository scope, the environment's overrides, *and* the
  secrets its resources own (a service account's key, a PKI key), which no
  other listing reaches; it needs `environment:View` on the environment as
  well as the per-row `secret:View`. `skyr secrets get <version-qid>` prints
  one **value** to stdout (raw bytes, no trailing newline) for a caller
  holding `secret:View` on it — `--encoding openssh` converts a generated
  ed25519 PKCS#8 key, and a value with control characters is refused on a
  terminal, so redirect it. The web equivalent is the environment's Secrets
  tab, with a per-row reveal. In config, read them
  via `Std/Secret`: `Secret.get(name).qid` is an opaque reference. Everywhere a
  value may be sensitive it is written `.literal("…")` (a plain value) or
  `.secret(qid)` — pass a secret as `.secret(Secret.get(name).qid)`. Two places
  take such values: a pod's or a container's `env`, and an ephemeral volume's
  `files` seed, mounted into the containers that need it (the only route for a
  value that can't be an env var — TLS keys, keytabs, anything binary). The
  plaintext is resolved inside the platform at pod materialization and never
  enters git, the stored resource inputs, or any log; a volume seeded with any
  secret is mounted owner-only (`0700`/`0600`, overridable with an explicit
  `permissions`) under the mount's `userId`/`groupId`, root by default.
  Consumption is IAM-gated with no implicit grant: the repo's **deployment
  role** must hold
  `secret:View` on each consumed secret, via an ordinary policy stanza —
  `IAM.Policy({ name: "read-secrets", subjects: [deployer.qid], verbs:
  ["secret:View"], objects: ["<org>/<repo>!*", "<org>/<repo>::*"] })` — or
  `Secret.get` raises `Secret.NotFound` at eval. That same verb is what
  releases plaintext to a *person* (`secrets get`, the web reveal): there is
  no narrower "metadata only" grant, so a wildcard `secret:View` over a repo
  is an export grant for every value in it — including keys resources
  generated — and anyone who can `assumeRole` into that deployment role has
  it too. **Rotation** (`skyr secrets
  set` again) takes effect on the next deployment, and how depends on where the
  secret is consumed: in `env` the pinned version is part of the pod's identity,
  so the pod is **recreated**; in a read-only volume seed the content is
  identity-excluded, so the new plaintext is written into the running pod **with
  no restart** — except when it outgrows the pod's disk claim, and except for a
  writable mount's seed, both of which recreate. Live delivery is not atomic and
  a process that reads its files once at startup never sees it: bump the
  volume's `name` to force a fresh pod. `skyr run` can't resolve secret
  plaintext locally, so it rejects a pod with any `.secret(…)` value in an `env`
  or a mounted volume's seed (a `.literal`-only pod runs); deploy the
  environment instead.
- **Knobs.** `Skyr/Knob.{Text,Toggle,Integer,Choice}` are manually provided,
  per-environment values a human turns out of band — a feature flag, a banner
  string, a worker count, a deployment mode. Declare one like any resource
  (`Knob.Integer({ name: "workers", default: 4 })`,
  `Knob.Choice({ name: "mode", choices: [.blue, .green], default: .blue })`) and
  read its `.value`, the effective value: the supplied value, else the declared
  `default`, else **pending**. A knob *with* a `default` resolves immediately; a
  knob *without* one holds `.value` pending, gating every dependent that reads it
  (they show as not-yet-created, no incident) until someone supplies a value —
  "awaiting input" is a first-class state, not a stuck rollout, and the
  deployment log names each open gate (`Waiting for user input: knob <name> —
  <description>`, once per gate, re-announced if the knob is later cleared),
  alongside a `Waiting to declare <resource>` line for each dependent the gate
  is holding back. The open gate is also a **hold** on the deployment itself
  (`skyr deployments list`'s `HELD` column, `Deployment.status.holds`), which is
  where to look when the log has scrolled past.
  Turn a knob with
  `skyr knobs set <name> <value>` (plain argv — knobs are non-secret; env from
  ambient context or `--environment`) or the web env **Knobs** tab (`~k`);
  `skyr knobs list` shows effective values and provenance, and `skyr knobs unset`
  clears non-destructively (converged dependents keep their last value). A set
  flows into the deployment on the next reconcile pass, rolling dependents —
  converge-plane, no hot reload. `Choice` offers a fixed set of concrete values
  and a consumer switching over its `.value` narrows exhaustively over them; a
  `Choice` selection is set by its rendered form (`skyr knobs set mode green`).
  Give knobs a `default` if you deploy to ephemeral/PR environments, which start
  unset. Turning a knob needs `resource:Update` on it; reading needs
  `resource:View` (the ordinary resource verbs).
- **`Artifact.File({ name, contents, mediaType })`** stores a file and
  returns a time-limited download `url` — useful for exposing generated
  config, reports, or deployment metadata.
- **`HTTP.Get({ url, headers })`** performs a GET as a resource (volatile;
  re-performed on input changes) and outputs `status`, `body`, `headers` —
  a probe or a way to pull external data into the config. Hosted deployments
  fetch public destinations only: `http`/`https` URLs, with requests to
  loopback/private/link-local addresses refused (including hostnames or
  redirect hops resolving to them); requests time out after 30s and the
  response is capped at 4 MiB. `skyr run` applies no address restrictions,
  so probing `localhost` locally keeps working.
- **`PKI.*`** — `ED25519PrivateKey`/`ECDSAPrivateKey`/`RSAPrivateKey`,
  `CertificationRequest`, `CertificateSignature`: build self-managed
  certificate chains (CAs, client certs, internal TLS) entirely in-config.
  An `RSAPrivateKey` `size` outside 2048–16384 bits raises
  `PKI.InvalidPKIInput` at evaluation time, failing at the offending call
  rather than at the plugin; `skyr run` surfaces it locally.
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
  act as its deployment role (default `Super`). A subject may **not**
  wildcard its org identity: a `*` before the first `/` or `::` — a bare `*`
  included — is rejected at authoring time, so name each org (a `*` past the
  org, like `"acme::*"` or `"acme/*::…"`, is fine). The one bare-string
  subject is `Anonymous`, the
  global sentinel meaning *anyone at all*: naming it publishes the
  granted reads to the public — and `repository:View` on it makes the source
  cloneable over `ssh://nobody@host/org/repo` — reaching every caller alike,
  signed out, signed in elsewhere, or a member of the org acting as a narrow
  role, while only a small read allowlist ever takes effect for it. Full model — matcher semantics, object
  shapes, what each verb gates, the anonymous surface:
  `curl -s https://skyr.foo/~docs/iam.md`.
- **`IAM.ServiceAccount({ name, role })`** — a **service account**: the third
  kind of principal, a non-human identity an external system (CI, a bot, an
  agent) signs in as with a role of its own instead of borrowing a person's.
  Outputs `qid` (its full resource QID — that *is* its account name at every
  sign-in surface, so renaming the resource is a different account),
  `privateKey` (the Secret Version QID of a generated Ed25519 key, sealed into
  the vault on creation and never in resource state — read it out with
  `skyr secrets get <that qid> --encoding openssh`), and `publicKeyPem`.
  `role` is a role QID and may name **another org's** role; the account is a
  member of the *role's* org, not the declaring one, and its standing is that
  role's, only there. Declaring one needs the declaring repo's deployment role
  to hold `role:AssignAsRootRole` on the role — decided in the **role's** org,
  which is that org's opt-in, and it is the *only* check (no
  `organization:InviteMember` beside it) — plus `secret:Write`/`secret:Delete` on
  the account's own key, matcher
  `"<org>/<repo>::*:Skyr/IAM.ServiceAccount:*!pem"`. Sign in with
  `skyr auth signin --username '<qid>' --key <pkcs8-or-openssh-key>`; over git
  the QID is the SSH username **percent-encoded** — once in the scp-style
  remote (`<encoded-qid>@host:org/repo`), twice in an `ssh://` one, since git
  URL-decodes that form before parsing it. Build the remote from that rule, or
  read `Repository.sshUrl` **while signed in as the account**
  (`skyr api query '{ organization(name: "<org>") { repository(name: "<repo>")
  { sshUrl } } }'`); never copy the clone URL a browser shows, which is
  composed for whoever is signed in there — an account has no browser sign-in,
  so that URL carries a human's username. Sessions get no refresh token
  (re-signing in is the refresh), and credential management, the assistant,
  notifications, org creation and the about-yourself surfaces (profile, and the
  membership listing behind `skyr org list`) refuse an account outright —
  everything an org owns is decided by policy as usual. Rotation = destroy +
  re-declare;
  removal = delete the declaration and deploy, which unregisters it
  (`skyr resources delete` is refused — only the declaring deployment may —
  and there is no membership mutation for it).
- **`Rollout.Group({ name, members })`** — groups the resources created
  inside its `members` closure under one parent resource, giving them a
  shared handle in the resource tree.
- **`Rollout.Version({ name, major, minor?, patch? })`** — a
  `major.minor.patch` number that advances itself. Wire anything hashable
  into a tier (a string, a record, a path — `major: ./src` means "bump the
  major number whenever anything under `src` changes") and that tier's
  counter goes up by one on the first deployment where its content differs
  from the deployment before. Creates at `0.0.0`; the *most significant*
  changed tier wins, advancing by one and zeroing every lesser tier;
  starting to track a tier or dropping one counts as a change of it; a
  deployment that changes nothing never bumps. Outputs are exactly
  `{ major: Int, minor: Int, patch: Int }`; render them with
  `Rollout.formatVersion(version)`. `name` is the whole identity, so
  renaming replaces the resource and the replacement restarts at `0.0.0` —
  there is no way to set or roll back a number. Use it for image tags,
  release labels, or anything that should move when content moves. Under
  `skyr run` (and `skyr check`/the REPL) paths carry no content identifier,
  so a path-backed tier never advances locally — that only happens on a
  real deployment.
- **`Rollout.ReplicaSet({ name, count, replica, maxSurge?, maxUnavailable? })`**
  — `count` copies of one function, rolled a few at a time. `replica` is
  called once per replica with `{ rev: Str }` and whatever it returns is
  collected into `.replicas`; name what it builds after `rev` so two
  revisions can be alive at once. Replicas are **cattle**: each is built at
  one *generation* and destroyed at it, never updated in place, so there is
  no stable per-replica identity (no volume that survives a roll).
  A generation is the replica function's identity — its code *plus
  everything it reads* — so a changed image URL or secret version rolls the
  set with the source untouched, and a set whose function reads a resource
  that has not materialized simply holds until it has.
  `maxSurge` (default `1`) is how far above `count` the set may go;
  `maxUnavailable` (default `0`) is how far below it may fall; both zero is
  refused. Two things worth stating outright: **retirement is paced by
  `maxUnavailable` alone** — at most `max(1, maxUnavailable)` teardowns in
  flight, waiting for each to finish, however much surge headroom there is,
  so `maxSurge: count` builds a whole generation at once and still retires
  one at a time; and **`maxSurge` bounds what the set asks for, not what is
  standing**, so teardown lag can briefly leave more than `count + maxSurge`
  replicas up — leave room for it in anything counted. Mid-roll `.replicas`
  deliberately holds both generations, so a DNS record or member list over
  the whole set stays complete. A replica counts as up only once everything
  it is made of has materialized — and where a resource can say whether it
  *works*, materialized includes having been seen working: a pod counts once
  its addresses are allocated, its finishable containers have finished, and
  every container that keeps it alive is either running and answering its
  `probe` where it declares one, or finished with the work it was given (a
  clean exit for an `.onFailure` job, any exit for a `.never` one — a job that
  gave up does not count). So returning the whole pod from `replica` is what
  makes a replacement that comes up broken **hold** the roll (old generation
  still serving, a `Crash` incident saying why) instead of retiring a replica
  that works; the gate is one-way, and a replica that has stood is not
  retired by later unhealth. The set is volatile (deployment stays `Desired`)
  until it settles. Everything a replica builds is a sub-resource of the set.
  `name` is the whole identity — renaming rebuilds from scratch,
  while `count: 0` keeps the set and holds nothing. Under `skyr run` the loop
  reconciles on change rather than on a timer, so a roll advances one step per
  save locally, and the replicas a roll retires come down when the run restarts
  or on ctrl+C — except for whatever a run's one held orphan walk still finds
  when it discharges, on the first pass that names everything with nothing in
  flight; a real deployment paces itself.
- **`Rollout.Rollback({ reason? })`** — declaring it asks the platform to roll
  this deployment back; see the rollback section above for the identity rules,
  the required `environment:Rollback` grant, and the inert-deployment hazard.
  Note the braces: `Rollback({})` is the no-reason form and `Rollback()` does not
  compile.
- **`Resource.Definition<T>("Name")`** — define your *own* resource type
  whose backend is SCL. One repo declares a typed definition and exports its
  `.Resource` constructor; other repos in the **same org** import the module
  and call the constructor to register instances (cross-org imports compile in
  general, but a cross-org instance registration is rejected at transition
  time); the defining repo reads every live
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
  editor completions/hover over guessing. A value the provider *returns* and
  flags sensitive is sealed into the vault as the resource's own secret and its
  output carries the Secret Version QID, not the value — and so does the
  provider state Skyr stores, which resolves the reference back on each
  reconcile (so the repo's deployment role needs `secret:View` as well as
  `secret:Write` and `secret:Delete`, the same three a `Skyr/PKI` key needs —
  a `Read*` data source seals the same way but keeps no state to reconcile
  from, so it wants only the other two);
  a sensitive value of any other shape than a string has no output field at
  all. An output that
  echoes back a secret you *supplied* — a connection string built around a
  password — reads `nil` for the same reason: no output serves secret material.
  (`HashiCorp/Random` is the same machinery wired only into the local dev
  harness.)

Exact inputs/outputs for the first-party modules live in the generated module
reference — look them up rather than guessing (see below).

## Watching a rollout

```sh
skyr deployments list             # states + rollback provenance of this repo's deployments
skyr deployments logs --follow    # stream deployment progress in real time
skyr resources list               # resources in the current env
skyr resources list --env staging
skyr resources logs Skyr/Container.Pod:web
skyr resources logs -f stockholm:Skyr/Container.Pod:web
```

A rollout is done when the deployment reports converged — state `Up`, or
`Desired` with all resources settled if anything is volatile (see the
lifecycle section). `skyr resources list` shows per-resource state; JSON
output (`--format json`) is the reliable way to script against either.

**Naming a resource.** Every command that takes one accepts any tail of the
resource QID `org/repo::env::region:Type:name`, completing the rest from
`--org`/`--repo`/`--env` (or the git checkout):

| Argument | Completed |
|---|---|
| `acme/shop::main::stockholm:Skyr/Container.Pod:web` | nothing |
| `main::stockholm:Skyr/Container.Pod:web` | org/repo |
| `stockholm:Skyr/Container.Pod:web` | org/repo, env |
| `Skyr/Container.Pod:web` | org/repo, env, region |

The `org/repo::env` head drops from the left only; the region is independent and
may be dropped from *any* of these (it then defaults to the repository's region,
which is not necessarily the org's home region). Types are always
plugin-qualified — `Skyr/Container.Pod`, never `Container.Pod`. Because every
wire parameter comes from the identifier rather than from flags, a fully-spelled
QID reaches another repo/env/region with no flags at all.

`skyr resources list` prints the reverse: its ID column is the shortest tail
that names each row **from the invocation that printed it, flags included** — so
rows listed under `--env staging` must be pasted back with `--env staging` (or
with the environment spelled into the identifier). The region is always shown.
`--format json` carries the full `qid` instead, which is what to script against.

To reach a not-publicly-exposed port from the local machine, forward it over
SSH. A port's ID as `skyr resources list` prints it pastes in verbatim; since a
forward only ever targets a `Skyr/Container.Pod.Port`, the type may also be left
out and the port named alone:

```sh
skyr port-forward stockholm:Skyr/Container.Pod.Port:web-3c6e542590f0cc60:8080/tcp 8080
skyr port-forward web-3c6e542590f0cc60:8080/tcp 8080
skyr port-forward web:8080 8080
```

The `-3c6e542590f0cc60` is a hash of the pod's spec: it changes whenever the pod
does, so it can't be typed from memory. Because `port-forward` and `exec` each
target exactly one type, they also match a name with the hash left out against
what is actually running — the third line above. Anything that resolves exactly
is used as it stands; a name several alive resources answer to is reported with
the candidates listed as pasteable identifiers.

Anything else — a `Skyr/Container.Pod`, say — is refused before the tunnel is
dialled: a declared port is what opens the pod's firewall, so it is the only
thing a forward can land on. The edge to tunnel through is the git server of the
checkout you run from; override with `--scs-address host:port`, and outside a
checkout an explicitly set `--api-url`/`SKYR_API_URL` names the instance (port
22), falling back to `skyr.cloud:22`. `exec` picks its edge the same way, and
every SSH connection the CLI makes is held to the host keys the instance's API
publishes — a mismatch is a hard error naming both keys, never a prompt.

To run a command — or get an interactive shell — inside a running container,
without exposing a port:

```sh
skyr exec web                           # a shell in the pod's only container
skyr exec web -c app ls -l /app         # a named container, and one command
echo 'select 1' | skyr exec db psql     # a pipe: input streams in, no terminal
```

Everything after the pod belongs to the command, so put `--` after the pod when
the command's own first word looks like one of `skyr exec`'s flags. A pod with
one container needs no `-c`; one with several is not guessed at, and the error
lists the names. Interactivity is detected rather than requested: input is
forwarded because there is input, and a terminal is allocated only when the
calling process is attached to one at both stdin and stdout (`--no-tty` and
`--no-stdin` opt out), so driving this from a script needs no flags. The
command's exit code is `skyr exec`'s, verbatim; every way `skyr exec` *itself*
fails exits **255** instead, which is how a script tells "the command exited 1"
from "the command never ran". A pod that was never created, has been destroyed,
or is not yet placed on a node is refused before anything is dialled.

Exec is authorized on `resource:ExecInPod` against the pod — a separate and far
broader grant than `resource:ForwardPodPort`, since a command inside a container
reaches that container's environment and mounted secrets.

## When a rollout goes wrong

Failures surface as **incidents** — durable records that open when something
is wrong long enough to matter, close automatically on recovery, and email
every member of the owning org on both events. Removing the offending
resource from the config also counts as recovery. The category names the
user-visible consequence, least to most severe:

| Category | Meaning |
|---|---|
| `TestFailure` | The commit failed its own tests, so Skyr rolled out nothing. Whatever was already deployed keeps serving. |
| `BadConfiguration` | Skyr refuses to roll out config it determined invalid. The system works; the config doesn't. |
| `CannotProgress` | The entity is stable, but something derived/dependent could not be applied. |
| `InconsistentState` | Reality drifted from config and reconciliation can't close the gap. |
| `Crash` | The thing is misbehaving with user-visible downtime. |

The badge on a deployment or resource is the **worse** of two readings: what
its open incidents say about Skyr's ability to operate it, and what the last
check said about whether the live thing is working. For a pod the second is a
real judgment: `Crash` means a container is crash-looping, has stopped
answering its probe, or is a job that spent every retry without succeeding, and
the incident message names that container and what is wrong with it. Three
readings that surprise people: a pod lost with its node or sandbox is **not** a
crash (it is rebuilt silently — `Crash` is for infrastructure that is fine
under a workload that is not); a **degraded** judgment opens no incident and
*closes* open ones, because a pod that is merely restarting too much is still
serving; and a verdict is held until something disproves it, so a probe-bearing
service that crashed once while coming up reads down until its probe actually
passes, silence never counting as recovery. A pod that has never served is
reported broken *and* does not count as standing for a `ReplicaSet` — that is
what explains a roll that is not advancing.

An **uncaught SCL exception** during evaluation — reading a secret that isn't
set, unwrapping a `nil`, any top-level `raise` your config doesn't `catch` —
opens a `CannotProgress` incident carrying the exception's message, so a missing
input surfaces on the deployment instead of silently stalling.

Any evaluation that can't finish fails the pass the same way — same
`CannotProgress` incident, same backoff — and applies **no** partial effects:
nothing is created, updated, or destroyed from an evaluation that didn't
complete. Beyond an uncaught exception, the other way to trip this is
**exhausting the per-pass resource budget**: each evaluation runs under a
generous ceiling on work, memory, and wall-clock time (ten seconds by default),
so a runaway loop or an enormous value (`List.range` over billions, a string
repeated to gigabytes) fails with an incident naming the limit hit. Two further
limits are fixed rather than granted: how deeply the *built-in* higher-order
functions (`List.map` and its kin) may nest inside one another, and how deeply a
single *value* may nest — a list inside a list, a record field holding a record,
a function closing over a function. A program's own functions do not count
against the first, recursion included, and real configuration reaches neither:
the value bound is some forty levels, which a program reaches only by wrapping a
value per call, never by writing one out. A budget
failure is *not* an SCL exception — `try`/`catch` cannot intercept it; the fix
is to bound the work each pass does.

A **`TestFailure`** incident means the commit's own tests did not pass, so the
deployment applied nothing (see the lifecycle section). Its message is the run's
summary — `3 of 12 cases failed`, or the reason a run produced no summary at
all, such as an exception raised outside every case. The per-case detail is in
the **deployment log**, written once at the pass that decided the verdict, one
entry per failing case:

```
fail  podName › when it disagrees
        toEqual: expected "main-web", got "main-api"
        at [fn] (acme/shop/Main 15:3,15:58)
        …
```

(The trace runs on for as many frames as the call had, and the log adds its
usual timestamp and severity prefix.) `fail` is an expectation the case did not
meet, `raise` an exception it let escape, and `await` a case that reached no
verdict at all. Reading a resource is **usually a `fail`**, not an `await`: a
test run holds no resource state, so every resource output is pending, and an
expectation over one is a `fail` whose `actual` is `<pending>` — only a case
that made no expectation and merely computed from a resource is an `await`.
Both name the resource awaited. The same lines come out of `skyr test` locally,
which is where to reproduce and fix it.

Incidents are listed on the website at `/<org>/~i` (the CLI doesn't surface
them). For diagnosis: `skyr deployments logs` shows evaluation and rollout
errors (a `BadConfiguration` usually reproduces locally with `skyr check`, a
`TestFailure` with `skyr test`); `skyr resources logs Skyr/Container.Pod:web`
shows a specific resource, including container output from pods.

## Looking up documentation

The instance serves its docs as raw markdown, ideal for fetching and
grepping:

```sh
curl -s https://skyr.foo/llms.txt                    # index of all doc pages
curl -s https://skyr.foo/~docs/deployments.md        # lifecycle, supersession, ownership in depth
curl -s https://skyr.foo/~docs/resources.md          # what every resource does: transitions, drift, reads
curl -s https://skyr.foo/~docs/jobs.md               # restart policy, jobs, cron-style scheduling
curl -s https://skyr.foo/~docs/networking.md         # private networks, routers, ACLs, internal DNS, the DMZ pattern
curl -s https://skyr.foo/~docs/status.md             # health, incidents, notifications
curl -s https://skyr.foo/~docs/secrets.md            # secrets: scopes, CLI, consumption, IAM
curl -s https://skyr.foo/~docs/knobs.md              # knobs: the four types, awaiting-input gating, skyr knobs CLI
curl -s https://skyr.foo/~docs/deletion.md           # deleting repos, orgs, and accounts
curl -s https://skyr.foo/~docs/cross-repo-imports.md # depending on another repo, and sharing resources with one
curl -s https://skyr.foo/~docs/iam.md                # permissions: roles, policies, matchers, verbs
curl -s https://skyr.foo/~docs/terraform.md          # Skyr/AWS and the other provider-backed modules
curl -s https://skyr.foo/~docs/scl/reference.md      # every documented module, one line each
```

Module signatures are two curls away: the reference index above lists every
`Std/*` and first-party `Skyr/*` module with a summary, and each module's own
markdown carries one `### <ExportName>` section per export — unqualified names,
types first, then functions and values — each with its signature, docs, and a
bullet per field. Modules generated from a Terraform provider's live schema
(`Skyr/AWS/*`, `HashiCorp/Random`) are not in the index; `~docs/terraform.md`
covers those, and their field names come from the provider's schema.

```sh
# the whole module
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md

# every export it has, in page order
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md | grep -n '^### '

# one export's whole section, ending at the next export's heading
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md | sed -n '/^### Pod$/,/^### /p'
```

A name that is both a type and a constructor — `Pod` the output record and
`Pod` the resource call — has a section apiece, so that last command prints
both. Sections run long, so prefer the range form over `grep -A <n>`, which
cuts off mid-section.

Permissions are documented next to the operation that needs them: wherever a
page describes something IAM gates, it carries a `> *IAM*:` line naming the
verb and what it allows. Grepping the raw markdown for that marker answers
"what must I be allowed to do here":

```sh
curl -s https://skyr.foo/~docs/secrets.md | grep '> \*IAM\*:'
curl -s https://skyr.foo/~docs/scl/reference/Skyr/Container.md | grep '> \*IAM\*:'
```

Every verb the documentation describes is also collected, with a link back to
the page describing it, in the searchable index at
`https://skyr.foo/~docs/iam/verbs/` (a rendered page, not raw markdown).
