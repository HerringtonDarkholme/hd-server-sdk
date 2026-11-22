# JavaScript Adapters

Reference for embedding a built server application in your own JavaScript host.
Most applications never need this: `hd run` and the deployment tasks wire these
pieces together. The [specification](../spec/server-sdk.md) holds the normative rules.

## Adapter Boundary

`adapters/application.ts` exports `createApplication({ module, bindings,
createCoreHost, grants, sink, log })`. The caller supplies the built Wasm module,
its signature-generated `.host.mjs` exports, the public core host factory,
an explicit capability map and an output sink or private log observer. No compiler-cache
path is an SDK interface.

`dispatch(request, signal)` takes the plain `std.http.Request` JavaScript
value and resolves to `{ response, cancelled, abandoned }` after guest
cleanup. An aborted signal preserves the owned `Deadline` or `Stopping`
reason; other abort reasons become `Disconnected`. The platform caller
constructs its native Response afterward.
`stop()` closes admission and joins the retained main after admitted jobs;
`fail(error)` closes the poisoned host and rejects all active dispatches.

`adapters/dispatcher.ts` provides the lower-level lease scheduler. A native
operation's start receives resolve/reject callbacks and returns optional
disposal. Both run in its lease's `run()` coroutine, even when another
request caused the guest poll. Late callbacks are ignored after disposal.
Shared state stores readiness and owner wake callbacks, without retaining a
global request Promise. This boundary is usable by subsequent fetch and
platform adapters; it does not yet supply their complete policies.

## Outgoing Fetch Transport

`adapters/http.ts` exports `createHttpClient(lease, options)`. It takes a
resolved core grant plus the public `isAsciiUrl` and `writtenHost` helpers;
its returned `send(request, context)` method uses `lease.operation` to
schedule native I/O. The optional fetch implementation supports platform
adapters and deterministic controls. The provider timeout defaults to
thirty seconds, covering redirects and body reading.

The transport validates grants before each manual redirect hop and strips
credentials on cross-origin redirects. Responses preserve raw body bytes,
non-success statuses and separate Set-Cookie values. Non-cookie headers
may be combined by the platform according to RFC 9110 §5.3. The serving loop
binds each request's source client and source clock around its handler and
cleanup. Returned handles expire when the lease finishes;
they cannot start a later request. The Node provider and Application adapter
use the core runtime's narrow `context.http` policy view, so explicit denial
and host scopes are checked before native I/O and on every redirect.
Admission limits and deployment packaging remain separate adapter layers.

## Fetch Mapping And Cleanup Backstop

`adapters/fetch.ts` exports `createFetchAdapter(options)` for one retained
Application, or `createFetchHandler(application, options)` for an existing
Application or Node supervisor. Both consume standard Fetch requests and
return standard Fetch responses. Runtime and generated bindings remain
explicit inputs.

The mapper reserves at most eight requests, including body reading, with a
one-MiB body limit and 64-KiB header limit on either side. A ninth request
gets 503 and Retry-After: 1. Its twenty-second deadline includes body reading.
Native stream cancellation runs in the request's own continuation.

The SDK's cooperative cleanup budget remains **100 ms from cleanup start**.
The adapter's separate watchdog fires at **200 ms from cancellation**, only
if no cleanup acknowledgement has arrived. A normal abandoned-cleanup
acknowledgement around 105 ms keeps the instance healthy. Platform CPU
limits remain independent; a blocked isolate needs an external supervisor.

## Node Worker Supervision

`adapters/supervisor.ts` exports `createSupervisor({ program, policy, event,
log, exhausted })`. `program` supplies a compiled WebAssembly.Module, explicit
runtime/bindings/Application module URLs, and resolved grants. Each worker
retains its own application state and accepts one active request at a time.
`createFetchHandler(supervisor)` keeps physical response ownership outside
the interruptible worker.

The cleanup watchdog is fixed at 200 ms and runs on the supervisor's
thread. Startup, request, drain, outer-stop, restart-count/window, and
Retry-After values are required caller inputs. An outer-stop budget below
200 ms is rejected, so it cannot truncate the cleanup contract. First stop
drains, second stop cancels, and stop/close completion joins termination.
Forced termination reports cleanup as unconfirmed.

`Application.start()` runs startup eagerly. Optional `onReady` and
`onStopped` observers report a completed turn parked on request acceptance
and stopped main completion, respectively. A main that never yields cannot
announce readiness.

These are explicit embedding components. The development `hd run` provider
does not automatically select the worker supervisor. Aggregate memory
acceptance remains part of each platform's release checks.

## Application Logs

The serving loop binds a request-owned `Console` beside its Http and Clock
handles. Handler and cleanup writes keep that lease's identity even when
another request causes the guest turn. Console denial returns `NotGranted`
before capture; a retained console rejects writes after its lease ends.

`log(record)` receives `{ version: 1, lease, stream, text }`. The stream is
`out` or `err`; lease generation and sequence are decimal strings. Startup
and unassociated runtime diagnostics in Application use `lease: null`. Without this
observer, Application forwards text through its ordinary sink.

Supervision adds `worker: { slot, generation }` and `request`, the physical
request ID, or null for unassociated output. Its log callback is separate
from lifecycle `event` callbacks. Both are synchronous observers; return
values are not awaited, and throwing does not own response or cleanup.
The SDK adds no headers, bodies or environment values to metadata, and
never copies private log text into generic HTTP errors.

The development Node provider writes request-console JSON records to their
corresponding standard streams; ambient CLI output keeps its ordinary Console
format. This does not add an aggregate memory
guarantee or select deployment budgets.
