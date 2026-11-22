# Deployment Reference

Details behind the Cloudflare and Vercel deployment tasks. The
[README](../README.md) covers the everyday steps; the
[specification](../spec/server-sdk.md) holds the normative rules.

## Cloudflare Deployment Task

The sample includes `tasks/deploy-cloudflare.hd`, an ordinary task using
`dep.server_sdk.deployment.cloudflare!`. Grant its SDK capability for local
task execution in the application's manifest:

```toml
[dev-capabilities]
"dep.server_sdk.deployment.Deployment" = true
```

The installed SDK carries the JavaScript provider declared by its own
`[host]` table, its runtime snapshot and its expected server ABI data. The
task needs no compiler checkout or compiler-cache path.

From `examples/server-api`, prepare and validate the Worker locally:

```sh
hd build
hd run deploy-cloudflare --engine node
wrangler deploy --dry-run --config cloudflare/wrangler.json
```

Node runs the SDK's package-declared JavaScript provider. The task reads
`build/debug/server_api.wasm` and its `.host.mjs` companion
and writes `cloudflare/wrangler.json` with its complete payload. Artifact
paths in the task resolve from the application root, including when the
task is invoked from a nested directory. The task prepares files locally;
Wrangler performs any later upload separately.

Application permissions come from that root's production `[capabilities]`,
including explicit consent to `dep.server_sdk.server.HttpServer`. Development
consent to the Deployment task does not grant the deployed application's
capabilities. Explicit denial through any alias of the request source
rejects preparation. An imported primitive Clock or Random capability also
rejects preparation when production policy explicitly denies it.

For Env, set `CloudflareOptions.env_bindings` to the names the Worker should
receive and grant those names in production policy. Only selected string
bindings are exposed; configure their values through Cloudflare separately.
The task records required names without reading or embedding values.

Preparation checks the captured Wasm against the installed SDK's nominal
request-source identity and expected boundary ABI. It checks the companion's
contract header without executing its JavaScript body, then emits SDK-owned
bindings. These checks establish ABI compatibility and provenance; they do
not authenticate arbitrary compiler output.

The server schema targets an hd-lang semantic version and the package-host
ABI version. It hashes the SDK-owned `server.hd` declaration for maintenance,
but does not hash std sources; an implementation-only std edit at the same
hd-lang version needs no schema regeneration.

The SDK owns the output marker, generation store and regular `wrangler.json`
selector. Repeated preparation selects a complete immutable generation and
retains previous generations for readers already using them. Unrelated root
files and Wrangler caches are preserved. Preparation refuses an unowned
output directory, changed selector, or modified active or destination payload.
It validates a retained destination before selecting it again. Untouched retained
generations are checked only by the explicit full audit below; other writers
must leave reserved paths unchanged while preparation runs.

The maintained deployment, output, schema and binding tests run with the
other SDK JavaScript tests. Local workerd probes exercised startup Env
decoding, retained string bindings, outgoing Http grants, Clock and Random.
They also exercised short and fixed twenty-second timeouts with cleanup
and recovery.
The synchronous driver rejected a pending `block_on` before native fetch;
this profile does not use a JSPI adapter, even when the engine exposes JSPI.

Five fresh local workerd processes reached their first response in a median
40.183 ms from spawn, measured externally. A private instrumented copy used
10.9375 MiB across both guest linear memories after eight overlapping
one-MiB requests. The [design record](SERVER_APPLICATIONS.md#cost-and-measurements)
records the probe versions, measurements and limits. These probes establish
local behavior, not hosted acceptance, CPU startup or total isolate memory.

## Vercel Node Deployment Task

The sample's `tasks/deploy-vercel.hd` uses
`dep.server_sdk.deployment.vercel!` through the same Deployment capability
as the Cloudflare task. Keep its development capability consent and the
application's separate production grants in the root manifest.

From `examples/server-api`, prepare the Node function locally:

```sh
hd build
hd run deploy-vercel --engine node
```

The task reads the built Wasm and binding companion and writes
`.vercel/output/config.json` with an immutable function generation. Vercel
uploads are a separate step using its ordinary prebuilt-output workflow.
The task performs no upload and requires no credentials. Its generated
function declares `nodejs24.x`, a [supported Vercel Node version](https://vercel.com/docs/functions/runtimes/node-js/node-js-versions).

`VercelOptions` selects the function name, `Artifacts`, explicit
`SupervisorPolicy`, and named `env_bindings`. The policy supplies worker
count, startup, drain, outer-stop, restart count/window and Retry-After.
Retry-After must be nonempty printable ASCII.
Choose one to eight workers; each retains its own application state and
accepts one active request. The request budget stays twenty seconds, with
the existing hundred-millisecond SDK cleanup and two-hundred-millisecond
host backstop.

Unavailable worker capacity returns 503 with the selected Retry-After
value, including during startup. Native Node streams feed the SDK's
bounded Fetch mapper, preserving request cancellation and separate
Set-Cookie values. Whole-body limits remain one MiB in either direction.

Production source consent, SDK identity, ABI and companion validation use
the same preparation checks as Cloudflare. Only explicitly selected
string environment bindings reach the Env provider, subject to production
grants. Their values are captured at Node entry initialization rather than
embedded in generated output.

Repeated preparation atomically changes `config.json` to select a complete
immutable function under `functions/`. Earlier generations remain available
to existing readers; unrelated root files and caches are preserved. An
unowned output directory, changed selector, or changed active or destination
function is refused. Untouched retained functions and unknown inactive directories
are checked only by the explicit full audit. Other writers must leave the SDK's
reserved paths unchanged during publication.

To validate the entire retained store, run this command from the installed SDK
directory while no publisher or other writer is changing its reserved paths:

```sh
node --experimental-strip-types packaging/audit-output.ts cloudflare /path/to/cloudflare
node --experimental-strip-types packaging/audit-output.ts vercel-node /path/to/.vercel/output
```

The audit checks the ownership marker, selector and every retained generation,
including unknown inactive directories. It reports errors without changing files.
Ordinary publication checks only the active and destination generations, so its
work does not grow with the retained generation count. Retention is unchanged.

Local controls use the official `@vercel/node` launcher and build-utils ZIP
builder, exercising retained requests, overlapping work, body limits and
production policy. The generated route uses a documented `request.path`
transform to pass the matched pathname to the Node runtime. Local launcher
controls bypass deployment-wide routing and do not establish hosted
build/deployment acceptance. Malformed native authority,
targets and headers receive 400 before guest dispatch.

Vercel Edge is outside this SDK's scope because Vercel deprecated standalone
[Edge Functions](https://vercel.com/docs/functions/runtimes/edge/edge-functions.rsc);
a later request requires its own issue.

## Deploy-Time Production Grants

`packaging/grants.ts` is the shared manifest reader for the SDK deployment tasks. `readProductionGrants(rootDirectory)` reads exactly that root's `hd.toml` on each call. `parseProductionGrants(text)` provides the same validation for an already-read manifest. Both preserve explicit true, false and scope lists, including package-capability consent; neither imports development grants or supplies missing grants.

The reader accepts TOML quoted keys, multiline arrays and ordinary escaping through a pinned, vendored parser. It rejects malformed TOML, invalid grant shapes and non-UTF-8 input. Error text omits source excerpts and grant values. Scope paths keep their manifest spelling for the consuming platform adapter to resolve; this helper does not invent a platform filesystem.

The policy conformance inputs are in `test/fixtures/deployment-grants.json`, executed by `node --test test/grants.test.ts` from the repository root and by the CLI's maintained SDK test wrapper. `hd build` remains policy-free: there is no `NAME.host.json`, and deployment grants are not read from compiler caches or build artifacts. The Cloudflare task uses this reader before preparing its output.

## Portable Runtime Snapshot

The SDK carries the platform-neutral host graph and its ABI model in `runtime/runtime-sources.json`. Deployment tools use `packaging/portable-runtime.ts` to validate this snapshot and emit it into a fresh task-owned staging directory. `readPortableRuntime()` returns the file texts and identity metadata; `emitPortableRuntime(directory)` also returns public module URLs for the emitted core and package-contract reader. Existing output is refused.

Snapshot source text uses LF independently of checkout line endings. Emitted files match those archived bytes.

The fingerprint covers emitted names and bytes, the package-host ABI version and the source base revision. The content hashes identify the actual source even when maintenance runs on an uncommitted host edit. These are integrity and provenance records, not signatures. The package-host ABI version is distinct from a claim that every compiler/runtime ABI is versioned. The Cloudflare task validates the built module and companion data before publishing its local output.

The installed SDK needs no compiler checkout, build companion or compiler cache to emit its runtime. The relocation control builds the sample, removes its HD source and SDK development provider, then uses the installed snapshot and production grants to serve counters 1, 2 and 3 and stop normally in a fresh Node process. Platform build validation remains part of each deploy task's acceptance.

Repository maintainers refresh the generated snapshot with `node packaging/sync-runtime.ts` when the portable host graph changes. The maintained SDK tests reject a snapshot that differs from the current host sources. Application deploy tasks consume the snapshot and never run this repository-maintenance command.
