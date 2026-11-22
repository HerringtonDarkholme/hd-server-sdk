# Server SDK

Write an HTTP server in [hd](https://github.com/HerringtonDarkholme/hd-lang):
one handler function, one shared state value and a pool of workers. Run it
locally with `hd run`, then package the same program for Cloudflare Workers or
Vercel.

```hd
fn handle!(request: Request, hits: &mut Hits) -> Response:
    hits.count += 1
    Response::status(.Ok).set_json(Reply { count: hits.count })

pub fn main!() -> void $ HttpServer:
    hits := box(Hits { count: 0 })
    serve!(4, hits, handle).expect("server")
```

## Requirements

- The `hd` command, from [hd-lang](https://github.com/HerringtonDarkholme/hd-lang).
  Install it with Cargo:

  ```sh
  cargo install hd-lang
  ```

- Node.js 26. The local server and the deployment tasks run on it.

## Quick Start

Create an application and add the SDK:

```sh
hd new --app my-server
cd my-server
hd add HerringtonDarkholme/hd-server-sdk
```

`hd add` records the newest release in `hd.toml`. Run `hd update` later to
move to a newer one.

Allow the application to serve HTTP by adding one line under `[capabilities]`
in `hd.toml`:

```toml
[capabilities]
"dep.server_sdk.server.HttpServer" = true
```

Replace `src/main.hd`:

```hd
use dep.server_sdk.server.{HttpServer, serve}
use std.http.{Request, Response}
use std.serde.{Deserialize, Serialize}

struct Hits:
    count: usize

@derive(Deserialize)
struct Input:
    message: string

@derive(Serialize)
struct Reply:
    count: usize
    message: string

fn handle!(request: Request, hits: &mut Hits) -> Response:
    input := match request.json::[Input]():
        .Ok(input) => input
        .Err(_) => return Response::status(.BadRequest)
    hits.count += 1
    Response::status(.Ok).set_json(Reply { count: hits.count, message: input.message })

pub fn main!() -> void $ HttpServer:
    hits := box(Hits { count: 0 })
    serve!(4, hits, handle).expect("server")
```

Run it and send a request:

```sh
hd run
curl -X POST http://127.0.0.1:8080/ -d '{"message":"hello"}'
# {"count":1,"message":"hello"}
```

Two larger samples:

- [`examples/server-api`](examples/server-api): a JSON API with a slow route
  and both deployment tasks.
- [hdmx-dungeon](https://github.com/HerringtonDarkholme/hdmx-dungeon): a
  dungeon crawler played in the browser through htmx, with all game logic on
  the server.

## Writing A Server

`serve!(workers, state, handler)` runs until the server stops.

| Argument | Meaning |
| --- | --- |
| `workers` | How many requests may be in progress at once. |
| `state` | One value shared by every request, created with `box(...)`. |
| `handler` | A function from a `Request` and `&mut` state to a `Response`. |

`Request` and `Response` are the ordinary `std.http` types, so reading JSON,
setting headers and choosing a status work as they do elsewhere in hd.

A handler can make outgoing HTTP calls, read the clock and print. Name what it
uses after `$` in its signature, and make sure `hd.toml` grants it:

```hd
fn handle!(request: Request, hits: &mut Hits) -> Response $ Clock:
    sleep!(ms(150))
    Response::status(.Ok)
```

```toml
[capabilities]
"dep.server_sdk.server.HttpServer" = true
Clock = true
```

## What The Server Does For You

| Situation | Result |
| --- | --- |
| A handler runs longer than 20 seconds | The client gets `504`. |
| A request times out or the client disconnects | The handler is cancelled and has 100 ms to clean up. |
| A request body is larger than 1 MiB | The client gets `413`. |
| Every worker is busy | The client gets `503` with `Retry-After`. |
| The process receives SIGTERM | New requests are refused and requests in progress finish before the server stops. |

## Local Settings

The local server listens on `127.0.0.1` only.

| Variable | Default | Meaning |
| --- | --- | --- |
| `HD_SERVER_PORT` | `8080` | Port to listen on. `0` picks any free port. |

On start it prints one line of JSON with the port it chose:

```json
{"event":"listening","host":"127.0.0.1","port":8080,"pid":12345}
```

## Deploying

Deployment is an ordinary hd task in your application. The task writes a
ready-to-upload folder; the platform's own tool does the upload. The task
needs no account and no credentials.

Both tasks need this grant in `hd.toml`:

```toml
[dev-capabilities]
"dep.server_sdk.deployment.Deployment" = true
```

They also print the output folder, which uses the `Console` grant that
`hd new` adds by default.

The deployed program gets the permissions in `[capabilities]`. Grants under
`[dev-capabilities]` apply only on your machine.

> **Status.** Vercel deployment is verified on the hosted service:
> [hdmx-dungeon](https://github.com/HerringtonDarkholme/hdmx-dungeon) runs
> at <https://hdmx-dungeon.vercel.app>. The Cloudflare task is tested only
> against a local copy of the Workers runtime.

### Cloudflare Workers

Add `tasks/deploy-cloudflare.hd`:

```hd
use dep.server_sdk.deployment.{
    Artifacts,
    CloudflareOptions,
    Deployment,
    DeploymentError,
    cloudflare,
}

pub fn main!() -> Result[void, DeploymentError] $ Console + Deployment:
    receipt := cloudflare!(
        CloudflareOptions {
            name: "my-server",
            artifacts: Artifacts {
                wasm: "build/debug/my-server.wasm",
                bindings: "build/debug/my-server.host.mjs",
                output: "cloudflare",
            },
            compatibility_date: "2026-10-23",
        },
    )?
    println(receipt.directory)
    .Ok(())
```

Build, prepare the Worker and check it with Wrangler:

```sh
hd build
hd run deploy-cloudflare --engine node
wrangler deploy --dry-run --config cloudflare/wrangler.json
```

Drop `--dry-run` to upload.

A Worker handles up to eight requests at once. One more gets `503` with
`Retry-After: 1`. Request and response bodies are limited to 1 MiB, headers to
64 KiB.

### Vercel

Add `tasks/deploy-vercel.hd`:

```hd
use dep.server_sdk.deployment.{
    Artifacts,
    Deployment,
    DeploymentError,
    SupervisorPolicy,
    VercelOptions,
    vercel,
}

pub fn main!() -> Result[void, DeploymentError] $ Console + Deployment:
    receipt := vercel!(
        VercelOptions {
            name: "my-server",
            artifacts: Artifacts {
                wasm: "build/debug/my-server.wasm",
                bindings: "build/debug/my-server.host.mjs",
                output: ".vercel/output",
            },
            supervisor: SupervisorPolicy {
                workers: 8,
                startup_ms: 3000,
                drain_ms: 1000,
                stop_kill_ms: 300,
                restart_limit: 2,
                restart_window_ms: 60000,
                retry_after: "1",
            },
        },
    )?
    println(receipt.directory)
    .Ok(())
```

Build and prepare the function:

```sh
hd build
hd run deploy-vercel --engine node
```

The result in `.vercel/output` is a prebuilt Vercel deployment for Node 24;
upload it with `vercel deploy --prebuilt`.

On Vercel each worker is a separate copy of your program with its own state,
handling one request at a time. `SupervisorPolicy` controls them:

| Field | Meaning |
| --- | --- |
| `workers` | Number of workers, from 1 to 8. |
| `startup_ms` | How long a worker may take to start. |
| `drain_ms` | How long requests in progress may take to finish during a stop. |
| `stop_kill_ms` | How long a worker has to stop before it is killed. At least 200. |
| `restart_limit` | How many times a failed worker may restart within the window. |
| `restart_window_ms` | Length of that window. |
| `retry_after` | `Retry-After` value sent with `503` when no worker is free. |

Request and response bodies are limited to 1 MiB. Vercel Edge is not supported.

### Environment Variables

List the variable names the deployed program may read in the task's options,
and allow the same names in `hd.toml`:

```hd
CloudflareOptions {
    # name, artifacts and compatibility_date as above
    env_bindings: ["API_TOKEN"],
}
```

```toml
[capabilities]
Env = ["API_TOKEN"]
```

Set the values in Cloudflare or Vercel as usual. The task records the names
only and never reads or stores a value.

### Preparing Again

Run the task again after each `hd build`. It replaces the selected version in
one step and leaves your other files in the output folder alone. It refuses to
write into a folder it did not create, or one whose prepared files were edited
by hand.

## More Documentation

- [Deployment reference](docs/DEPLOYMENT.md): permission rules, output layout,
  validation and the output audit command.
- [JavaScript adapters](docs/JAVASCRIPT_ADAPTERS.md): embedding a built server
  in your own JavaScript host, worker supervision and request logs.
- [Specification](spec/server-sdk.md): the exact rules.
- [Design record](docs/SERVER_APPLICATIONS.md): why the SDK works this way.
- [Developing the SDK](docs/DEVELOPMENT.md): building, testing and repository
  layout.
