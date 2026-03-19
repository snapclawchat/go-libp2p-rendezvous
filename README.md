# go-libp2p-rendezvous

Rendezvous (meeting point) protocol implementation for libp2p, used to **register and discover peers within a namespace**.

This repository provides:

- **Server**: `RendezvousService` (handles `/rendezvous/1.0.0` streams)
- **Client**: `RendezvousClient` / `RendezvousPoint`
- **Discovery adapter**: `RendezvousDiscovery` implements `github.com/libp2p/go-libp2p/core/discovery.Discovery`
- **Storage backends**: SQLite / SQLCipher (see `db/sqlite`, `db/sqlcipher`)

## Requirements

- **Go**: `go1.25.7` (`go.mod` includes `toolchain go1.25.7`)

## Install

```bash
go get github.com/kaleidra/go-libp2p-rendezvous@latest
```

## Quickstart

The following snippets show the basic flow: run a rendezvous server, register from a client, then discover peers.

### 1) Run a server (`RendezvousService`)

The server needs a libp2p `host.Host` and a database implementing `db.DB` (e.g. sqlite).

```go
package main

import (
  "context"
  "log"

  libp2p "github.com/libp2p/go-libp2p"
  rendezvous "github.com/kaleidra/go-libp2p-rendezvous"
  sqlite "github.com/kaleidra/go-libp2p-rendezvous/db/sqlite"
)

func main() {
  ctx := context.Background()

  h, err := libp2p.New()
  if err != nil {
    log.Fatal(err)
  }
  defer h.Close()

  db, err := sqlite.OpenDB(ctx, ":memory:")
  if err != nil {
    log.Fatal(err)
  }
  defer db.Close()

  _ = rendezvous.NewRendezvousService(h, db)

  // Keep your process alive; omitted for brevity.
  select {}
}
```

### 2) Register (`RendezvousClient.Register`)

The TTL parameter is in **seconds** (`int`). The client will refresh registrations in the background (see `registerRefresh`).

```go
rc := rendezvous.NewRendezvousClient(myHost, rendezvousServerPeerID)

// Register this host under the "chat" namespace
ttl := rendezvous.DefaultTTL
recordTTL, err := rc.Register(ctx, "chat", ttl)
if err != nil {
  // handle err
}
_ = recordTTL
```

### 3) Discover (`RendezvousClient.Discover`)

Discover supports pagination via a cookie:

```go
peers, cookie, err := rc.Discover(ctx, "chat", 100, nil)
if err != nil {
  // handle err
}

// peers is []peer.AddrInfo; you can connect via myHost.Connect(ctx, peers[i]).
_ = peers
_ = cookie
```

### 4) Using libp2p Discovery (`RendezvousDiscovery`)

If your code already uses `core/discovery.Discovery`, you can plug this in directly:

```go
rd := rendezvous.NewRendezvousDiscovery(myHost, rendezvousServerPeerID)

// Advertise uses discovery.TTL(time.Duration) (e.g. 2 hours)
_, err := rd.Advertise(ctx, "chat")
if err != nil {
  // handle err
}

ch, err := rd.FindPeers(ctx, "chat")
if err != nil {
  // handle err
}
for pi := range ch {
  _ = pi // peer.AddrInfo
}
```

## Run tests

```bash
go test ./...
go test -race ./...
```

## FAQ

### What is the TTL unit?

- Registration TTL is in **seconds**.
- `RendezvousDiscovery.Advertise` converts `discovery.Options.Ttl` to seconds before registering.

### What is the discover cookie?

- Cookies are used for incremental discovery (pagination).
- Cookie validity is checked by the server DB (nonce + namespace + counter). Clients just echo it back.

### Which backend should I use: sqlite or sqlcipher?

- `db/sqlite`: plain SQLite for local/non-encrypted use cases.
- `db/sqlcipher`: SQLCipher (requires environment support) for encrypted-at-rest use cases.

### Can I use a pure-Go sqlite driver?

Yes. `db/sqlite` now supports explicit driver selection:

```go
db, err := sqlite.OpenDBWithDriver(ctx, ":memory:", sqlite.SQLiteDriverModernc) // pure-go
```

The default `sqlite.OpenDB` behavior is unchanged and uses `sqlite.SQLiteDriverMattn`.

