# High availability

New in 0.3.0. Tested on one machine with two gateway processes; not yet on Kubernetes. Try it before you rely on it.

Several gateways can run behind one address. One leads at a time; when it stops or is lost, another takes over.

## What it gives, and what it doesn't

- The API stays up through the loss of a gateway or its node. A request may go to any gateway.
- After a leader is lost, another leads within about 15 seconds (measured: 13 s after `kill -9`, 4 s after a clean
  stop). Requests in those seconds answer 503 `no_leader` with `Retry-After: 5`.
- Merges pause for those seconds. A merge that was in flight is settled by the engine from the database's own
  record of merges, as after any crash: it either happened whole or not at all.
- **A run that was open when the leader died hard loses its work.** The engine keeps a world's unmerged changes
  through a clean stop, not through a crash. The run's next call answers 409 `world_lost`, the run becomes
  `failed`, and nothing of it has reached the database. Start a new run. A clean handover (a rolling update,
  `SIGTERM`) keeps open runs.
- It does not make the database agents work on highly available: that is yours.

## What it needs

1. **A store in Postgres**, shared by every gateway: `GATEWAY_STORE_URL=postgres://...`. A database for the
   gateway's own tables (keys, runs, approvals, events, the lease). It can live on the server agents work on, in a
   database of its own. Without this setting the store is SQLite in the data folder, and there is one gateway.
2. **One data folder**, mounted by every gateway at `GATEWAY_DATA`: a ReadWriteMany volume (EFS, Filestore, Azure
   Files, CephFS, NFS). It holds the engines' folders, the database URLs and the key that seals the store's secrets.
   Only the leader's engines write it.
3. **`GATEWAY_NODE_URL`** on each gateway: how the others reach it (`http://10.0.0.5:7400`). The leader's goes into
   its lease, and followers send requests there. Allow that traffic between the gateways.
4. The engine run by the gateway itself (not `CHRONOS_ENGINE_URL`).

If the gateways don't share a data folder's `secret.key`, give each the same key file with `GATEWAY_SECRET_FILE`.

## How it works

- **The lease.** A row in the store (`leases`), taken in one statement, renewed every 5 seconds, good for 15. Time
  is the store's clock, not each machine's.
- **The leader** runs the engines and the work that must happen once: the sweeper, webhook deliveries, the model
  reviewer, Chronos Cloud's link.
- **A follower** answers `/health` itself and sends every other request to the leader, unchanged, and returns the
  leader's answer. The engines stay on the leader's loopback.
- **Taking over.** A leader that stops gives the lease up. One that dies stops renewing; after 15 seconds a
  follower takes the lease and starts the engines on the shared folder.
- **Two engines never share a folder.** A leader that can't renew for 8 seconds (the store unreachable, or the
  lease taken) stops its engines, kills any still there 4 seconds later, and exits: all before its 15-second lease
  runs out. Whatever runs it starts it again, as a follower. The lease is renewed and watched on threads of their
  own, so a store that stops answering can't hold them. Measured with the store frozen: engine gone and gateway
  exited after 6.5 s. On Linux an engine is also told to stop when its gateway dies.

`GET /v1/status` says `ha: {shared, node, role, leader: {node, url, expires}}`.

## Kubernetes

```yaml
ha:
  enabled: true
  replicas: 2
  storeExistingSecret: chronos-store      # key storeUrl
  existingClaim: chronos-gateway-shared   # ReadWriteMany
```

The chart sets `GATEWAY_NODE_URL` from each pod's address. With a NetworkPolicy, allow the gateways' pods to reach
each other on 7400.

## Limits

- One connection to the store's Postgres per gateway, one statement at a time.
- A follower holds a request's body in memory to send it on (64 MB at most).
- The fence between an old leader and a new one is time (the lease) and, on Linux, the engine stopping with its
  gateway. A leader paused whole for longer than 15 seconds and then resumed (a frozen VM) runs for a fraction of a
  second more before its watch notices.
- A statement the store's Postgres doesn't answer in 30 seconds fails its request.
- Not built: taking the engine's folder over from object storage (so no shared volume is needed), a pool of store
  connections, moving an existing SQLite store into Postgres.
