# Zebra fork — release workflow

This fork carries Zebra-specific patches on top of upstream `lorenzodonini/ocpp-go` releases. The canonical consumer is `zebra-saas-ocpp` (the evse-gateway), which pulls the fork via a `replace` directive in `go.mod`.

## Branch and tag conventions

- **Branch**: `zebra/<short-topic>` (e.g. `zebra/connmutex-deadlock-fix`). Branch off the upstream tag the patches are based on (currently `v0.19.0`).
- **Tag**: `<upstream-version>-zebra<N>` (e.g. `v0.19.0-zebra2`). `N` increments monotonically per fork release. Never reuse a tag — once pushed, it is permanent. If a release is wrong, cut a higher `N` rather than `-f` retagging.

## Per-release sequence

Inside this repo (`/Users/ertankara/Desktop/zebra/v2/lib/ocpp-go`):

1. Make code changes on the active branch.
2. Sanity-build and test (skip Toxiproxy-dependent tests if Toxiproxy isn't running locally):
   ```
   cd /Users/ertankara/Desktop/zebra/v2/lib/ocpp-go && go build ./... && go test ./... -count=1 -timeout 5m -skip 'TestNetworkErrors'
   ```
3. Commit:
   ```
   cd /Users/ertankara/Desktop/zebra/v2/lib/ocpp-go && git add <files> && git commit -m "ws: <short summary>"
   ```
4. Cut the next tag (annotated, with a message — repo's git config requires it):
   ```
   cd /Users/ertankara/Desktop/zebra/v2/lib/ocpp-go && git tag -a v0.19.0-zebraN -m "v0.19.0-zebraN: <one-line reason>"
   ```
5. Push branch and tag:
   ```
   cd /Users/ertankara/Desktop/zebra/v2/lib/ocpp-go && git push origin zebra/<branch> v0.19.0-zebraN
   ```

Inside `admin/zebra-saas-ocpp`:

6. Bump the `replace` directive in `go.mod` to the new tag.
7. Refresh `go.sum` and confirm the gateway still builds:
   ```
   cd /Users/ertankara/Desktop/zebra/v2/admin/zebra-saas-ocpp && go mod tidy && go build ./...
   ```
8. Confirm the redirect resolved:
   ```
   cd /Users/ertankara/Desktop/zebra/v2/admin/zebra-saas-ocpp && go list -m github.com/lorenzodonini/ocpp-go
   ```
   Expected output (with the actual new tag):
   ```
   github.com/lorenzodonini/ocpp-go v0.19.0 => github.com/ertankara/ocpp-go v0.19.0-zebraN
   ```
9. Commit `go.mod` and `go.sum` together — they must move atomically.
10. Deploy via Kamal (standard flow).

## Why `replace` instead of switching imports to the fork directly

Using `replace github.com/lorenzodonini/ocpp-go => github.com/ertankara/ocpp-go vX` keeps all imports in `zebra-saas-ocpp` pointing at the upstream path. That means:

- Zero `import` edits when adopting a fork release.
- Trivial rollback to upstream: delete the `replace` line and run `go mod tidy`.
- The fork's `go.mod` stays as `module github.com/lorenzodonini/ocpp-go`, so module identity is unchanged and Go's checksum machinery still works.

Switching to a direct dep would require renaming the fork module, rewriting every gateway import, and burning the path back to upstream. Don't do it unless this fork becomes permanent.

## Verifying the fork is active in a running gateway

At server boot, `ws.(*Server).Start` prints to stdout:

```
ocpp-go fork ertankara/ocpp-go v0.19.0-zebraN: connMutex deadlock fix active
```

This bypasses the library's pluggable `logging.Logger` (which defaults to `VoidLogger` if the consumer never calls `ws.SetLogger`), so it appears regardless of the consumer's logging setup. Grep the container's logs for `ocpp-go fork` to confirm.

To verify build-time wiring instead of runtime, run inside `zebra-saas-ocpp`:

```
go list -m github.com/lorenzodonini/ocpp-go
```

The `=>` arrow in the output confirms the redirect is in effect.

## Avoiding the most common foot-gun

Never `git tag -f` a previously-pushed tag. Go modules pin by content hash in every consumer's `go.sum`. If you retag, every teammate and every CI run hits `checksum mismatch` errors and has to clear their module cache. Always cut a new `-zebraN` tag instead.

## Eventual exit from the fork

When (and if) the upstream PR for these patches is merged into `lorenzodonini/ocpp-go`:

1. Bump `go.mod` in `zebra-saas-ocpp` to the upstream release that contains the patches (e.g. `github.com/lorenzodonini/ocpp-go v0.20.0`).
2. Delete the `replace` directive.
3. `go mod tidy`, commit, deploy.

The fork can then be archived. Until that day, the workflow above is the steady-state.
