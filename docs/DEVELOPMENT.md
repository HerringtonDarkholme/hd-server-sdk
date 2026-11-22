# Developing The SDK

Notes for people working on the SDK itself. To use the SDK in an application,
start with the [README](../README.md).

## Implementation Overview

An application owns its ordinary `main!`, one state object and a joined
group of workers. `server.HttpServer` is an SDK-owned package capability;
requests and responses use `std.http` values.

The implementation supplies the typed source, joined workers, bounded
request cleanup and a loopback Node listener. A request races its source
cancellation signal against a handler wrapped by `apply!(timeout(20s, cleanup=100ms))`.
The source supplies its cleanup clock, so `serve!` adds no ambient Clock row.
Every accepted lease is released after response delivery or cleanup.
The public JavaScript adapters supply retained synchronous turns,
lease-owned outgoing I/O, admission deadlines, request logs and Node worker
supervision. The SDK also prepares Cloudflare Worker output through an
ordinary deployment task. A second task prepares supervised Vercel Node
functions. Hosted deployment acceptance remains unproven.

## Building And Testing

Build `hd` from the [hd-lang](https://github.com/HerringtonDarkholme/hd-lang)
repository (`cargo build -p hd_command --features wasmtime`), then run
`hd run` from `examples/server-api`, granting the SDK trait in the
application's manifest. The sample accepts a JSON object with a `message`
string; malformed JSON returns 400. `POST /slow` suspends for 150 ms and
demonstrates overlapping workers. `HD_SERVER_PORT` selects the loopback
port (8080 by default, 0 for an ephemeral test port). SIGTERM closes
admission and joins workers after their admitted requests finish.

Focused checks from the repository root, with scratch `HD_CACHE`:

```sh
hd check --all
hd test --engine node
hd test --engine wasmtime
```

The library tests bind an ordinary HD provider. The live Node check uses
the actual sample and package-provider boundary:

```sh
node --test test/*.test.ts
```

`HD_SDK_TEST_HD` can name a different compiled CLI. No test needs a public
listener, external network service or deployment credentials.

SDK adapters, providers, packaging tools and tests are plain `.ts` sources.
Deployment packaging strips their erasable types and publishes runnable `.js` modules;
published bundles never contain TypeScript source.

## Repository

This package moved out of the hd-lang repository on 2026-11-21, with its git
history. hd's standard library keeps HTTP in both directions: `std.http`'s
client, the `std.http_wire` byte messages, and the `request_from_wire` and
`response_to_wire` conversions, specified in hd-lang's `spec/std/http.md`.
This repository holds what is specific to the SDK:

- the `server_sdk` package (the root `hd.toml`, `src`, `providers`,
  `adapters`, `runtime`, `packaging`);
- `examples/server-api`, an application with Cloudflare and Vercel deploy tasks;
- `spec/server-sdk.md`, the package's specification (its wire-message rules are
  now std's, in hd-lang's `spec/std/http.md`);
- `docs/SERVER_APPLICATIONS.md`, the design record;
- tests: `test/` (Node, run with `HD_SDK_TEST_HD` set to an `hd` binary),
  `tests/` (hd tests), and `tests/conformance/` (runtime cases that use the
  package).
