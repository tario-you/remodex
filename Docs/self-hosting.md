# Self-Hosting Remodex

This guide is for developers who clone the public GitHub repository and want to run Remodex on infrastructure they control.

It covers two supported setups:

1. Local LAN pairing on your own machine
2. A self-hosted VPS relay that your bridge connects to over the internet

This document intentionally avoids any private hosted-service details. If you are using the public repo, assume you are bringing your own relay endpoint.

The public source tree is local-first and self-host friendly:

- there is no public production relay baked into the GitHub source
- local pairing should work out of the box with `./run-local-remodex.sh`
- internet-facing setups should pass their own relay URL explicitly with `REMODEX_RELAY`
- the first QR scan bootstraps trust, then later reconnects can reuse the same trusted Mac through that relay
- the built-in background daemon for trusted reconnect is currently macOS-only

## What Remodex Self-Hosting Means

Remodex is local-first.

That means:

- the bridge runs on your own Mac
- Codex runs on your own Mac
- git commands run on your own Mac
- your iPhone is a remote control
- the relay is only a transport layer for pairing, trusted-session resolve, and encrypted message forwarding

The relay does not run Codex and does not get your plaintext application payloads after the secure handshake completes.

## Option 1: Local LAN Setup

This is the easiest way to try the public repo, but on iPhone it should be treated as a best-effort local test path. The recommended self-host setup for regular use is Tailscale or another stable private network path to your relay.

### What you need

- a Mac with Codex CLI installed
- an iPhone with a Remodex build installed
- both devices on the same local network

### Start everything locally

From the repo root:

```sh
git clone https://github.com/Emanuele-web04/remodex.git
cd remodex
./run-local-remodex.sh
```

What this does:

- starts a local relay on your machine
- starts the Remodex bridge
- prints a pairing QR code for first-time trust bootstrap or recovery

Then:

1. Open the iPhone app
2. Scan the QR code from inside the app
3. Start a thread and send a message
4. On later launches, let the app try trusted reconnect before scanning again

### If your iPhone cannot reach the default hostname

Pass a hostname or IP address that the phone can actually reach:

```sh
./run-local-remodex.sh --hostname 192.168.1.10
```

If you are running from a source checkout and `remodex up` is not installed globally, that is expected. Use `./run-local-remodex.sh` from the repo root, or install the npm package globally with `npm install -g remodex@latest`.

### Health check

By default the local relay listens on port `9000`.

From the same Mac:

```sh
curl http://127.0.0.1:9000/health
```

You should get:

```json
{"ok":true}
```

## Option 2: Self-Hosted VPS Relay

Use this when you want the bridge on your Mac to connect through a relay you run on a VPS.

This is also the best base for a Tailscale setup: the relay can live on a Mac, a mini server, or a VPS you control, as long as the iPhone can reach it reliably.

### What runs where

On your VPS:

- the Remodex relay

On your Mac:

- the Remodex bridge
- Codex CLI / `codex app-server`

On your iPhone:

- the Remodex app

### Start the relay on the VPS

From the public repo:

```sh
git clone https://github.com/Emanuele-web04/remodex.git
cd remodex/relay
npm install
npm start
```

By default the relay listens on port `9000`.

### Verify the relay

On the VPS:

```sh
curl http://127.0.0.1:9000/health
```

You should get:

```json
{"ok":true}
```

### Put a reverse proxy in front of it

Expose the relay through a public `ws://` or `wss://` endpoint that forwards to the Node relay.

Two common patterns are:

- a dedicated subdomain, for example `wss://relay.example.com/relay`
- a shared-domain subpath, for example `wss://api.example.com/remodex/relay`

If you use a shared-domain subpath, make sure your reverse proxy strips the prefix before forwarding so the Node process still receives `/relay/...`.

### Point the bridge at your VPS relay

On the Mac that runs the bridge:

```sh
REMODEX_RELAY="wss://relay.example.com/relay" remodex up
```

Or, if you are running from source:

```sh
cd phodex-bridge
npm install
REMODEX_RELAY="wss://relay.example.com/relay" npm start
```

The bridge will print a QR code the first time you trust that Mac, or later if you intentionally reset trust.

That QR carries the relay URL and session information, so the iPhone does not need a hardcoded relay endpoint in the public source build.

After the first successful scan:

- the iPhone stores the Mac as a trusted device
- the bridge keeps its local device identity
- the relay can resolve the current live session for that trusted Mac
- the app can reconnect without requiring a new QR every time

Today, that background-service path is built in for macOS. If you self-host against a non-macOS bridge, pairing and relay routing still work, but you must manage persistence/background service behavior yourself.

If you install the bridge from npm and do not use the local launcher, make sure you export `REMODEX_RELAY` before running `remodex up`.

## Push Notifications

Managed push is optional.

For public self-hosting:

- you do not need push to use Remodex
- local in-app and local-device flows can still work without it
- the relay keeps push endpoints disabled by default

Do not turn push on unless you are also ready to configure:

- a bridge-side `REMODEX_PUSH_SERVICE_URL`
- APNs credentials on the relay side
- your own operational setup for notification delivery

If you do nothing here, push stays off.

## Reverse Proxy Notes

If your relay sits behind Traefik, Nginx, or Caddy:

- forward WebSocket upgrades correctly
- forward the `/relay/...` path to the relay process
- only enable `REMODEX_TRUST_PROXY=true` when the proxy is trusted and sanitizes forwarded IP headers

## What Not to Commit

If you are self-hosting from the public repo, keep these things out of Git:

- your real relay hostname
- your private VPS IP addresses
- any APNs credentials
- any private package or App Store build defaults

The public repo should stay generic. Your actual deployment values belong in your own environment, build pipeline, or private config.

## Troubleshooting

### The bridge starts but the iPhone cannot connect

Check:

- the relay is reachable from the phone
- your reverse proxy forwards WebSockets
- the bridge is using the correct `REMODEX_RELAY`
- the public endpoint uses `wss://` if you are going over the internet

### Local LAN pairing fails

Try a concrete LAN IP:

```sh
./run-local-remodex.sh --hostname 192.168.1.10
```

If local LAN pairing still fails on iPhone even though the relay health check works, prefer a Tailscale-reachable relay instead of continuing to rely on plain `ws://` over the same Wi-Fi.

### The relay health check works, but pairing still fails

That usually means one of these:

- the public path is wrong
- the reverse proxy is not forwarding upgrades
- the bridge is pointing at the wrong relay base URL

### The phone shows fewer models than Codex desktop

Check the backend that Remodex is actually using before changing the iPhone app:

- the iPhone loads models from `model/list`
- the bridge usually spawns a local `codex app-server`
- an older or different local Codex CLI can expose a smaller model catalog than the desktop app UI

Start with:

```sh
codex --version
```

Then restart the bridge after any CLI update so a newly spawned `codex app-server` picks up the newer binary.

## Audited Learnings From The 2026-03 LAN Pairing Debug

This section captures the specific local-source and iPhone-pairing failure mode that was debugged in the public repo, along with the source files that were audited to verify each conclusion.

### Audited sources

- [`../run-local-remodex.sh`](../run-local-remodex.sh)
- [`../CodexMobile/CodexMobile/Services/CodexService+SecureTransport.swift`](../CodexMobile/CodexMobile/Services/CodexService+SecureTransport.swift)
- [`../CodexMobile/CodexMobile/Views/Home/ContentViewModel.swift`](../CodexMobile/CodexMobile/Views/Home/ContentViewModel.swift)
- [`../CodexMobile/CodexMobile/Services/CodexService+Connection.swift`](../CodexMobile/CodexMobile/Services/CodexService+Connection.swift)
- [`../phodex-bridge/src/secure-transport.js`](../phodex-bridge/src/secure-transport.js)

### What changed

- `run-local-remodex.sh` was made compatible with the stock macOS `/bin/bash` by replacing the Bash 4-only `${var,,}` lowercase expansion with a portable `tr` normalization step.
- The README and this guide now document the source-checkout path explicitly so `remodex up` is not assumed to exist before `npm install -g remodex@latest`.
- The local-LAN recovery flow now calls out that switching to a concrete LAN IP requires a fresh QR scan, not just tapping `Reconnect`.

### What worked

- Fixing the Bash portability issue let the launcher reach the QR-printing stage again on macOS without requiring Homebrew Bash.
- Rerunning the launcher with `--hostname <lan-ip>` produced a QR whose advertised relay URL uses a concrete host the iPhone can reach directly.
- Using `Scan New QR Code` after changing the hostname forced the iPhone to save the new relay URL and session instead of retrying the old one.

### What did not work

- Re-running the launcher with the default `.local` hostname did not address the iPhone reachability problem.
- Tapping `Reconnect` after the relay host changed did not help, because the app reuses the saved relay URL and session from the previous QR scan.
- Fixing only the launcher crash did not fix pairing by itself; it only removed the first blocker and exposed the next one.

### Why the previous fix did not fully solve the problem

The first fix addressed a shell compatibility bug in the launcher. It made `./run-local-remodex.sh` start the relay and print the QR again, but it did not change the advertised relay hostname inside that QR.

The audited app-side code shows why that matters:

- the launcher writes `REMODEX_PUBLIC_RELAY` into the QR payload
- the iPhone saves that scanned relay URL and session when pairing
- later `Reconnect` attempts prefer the saved/trusted session path and fall back to the saved QR session

So the real transport issue remained until the QR was regenerated with a concrete LAN IP and the iPhone scanned that new QR.

### Recommended recovery flow for this failure mode

1. Start the local launcher with a concrete host the phone can reach, for example `./run-local-remodex.sh --hostname 192.168.1.10`.
2. Keep the terminal open after the QR is printed.
3. On the iPhone, tap `Forget Pair`.
4. Confirm `Settings > Remodex > Local Network` is enabled.
5. Use `Scan New QR Code`.
6. Only use `Reconnect` after a successful scan has already saved the correct relay URL.

### How to avoid this debugging loop next time

- Separate launcher failures from transport failures. If the script does not reach the QR, fix that first before reasoning about iPhone networking.
- When testing LAN pairing on iPhone, prefer a concrete LAN IP first and treat `.local` as a convenience path, not the most reliable baseline.
- After changing `--hostname`, always rescan a fresh QR. Do not assume `Reconnect` will pick up the new host automatically.
- Capture the Mac terminal logs immediately after scanning. The most useful lines are the relay connection line and any secure-handshake lines from the bridge. Redact live session IDs before sharing logs.
- If local Wi-Fi remains flaky even with a concrete LAN IP and a healthy relay, move to a Tailscale-reachable relay instead of continuing to iterate on plain local `ws://`.

## Audited Learnings From The 2026-03 Runtime Model Catalog Debug

This section captures the separate failure mode where the iPhone showed fewer models than Codex on the Mac. The root cause was not in the iPhone picker UI. It was the backend runtime that the bridge had spawned.

### Audited sources

- [`../CodexMobile/CodexMobile/Services/CodexService+RuntimeConfig.swift`](../CodexMobile/CodexMobile/Services/CodexService+RuntimeConfig.swift)
- [`../phodex-bridge/src/codex-transport.js`](../phodex-bridge/src/codex-transport.js)
- local verification with `codex --version`
- local verification by querying `model/list` from a spawned `codex app-server`

### What changed

- The local Codex CLI on the Mac was updated from `codex-cli 0.72.0` to `codex-cli 0.117.0` through the Bun-managed `@openai/codex` install.
- The bridge service was restarted so its spawned `codex app-server` used the updated CLI instead of the older binary that had been running before.
- This guide now documents the backend-first check so future debugging starts with the runtime catalog instead of the mobile UI.

### What worked

- Checking [`../CodexMobile/CodexMobile/Services/CodexService+RuntimeConfig.swift`](../CodexMobile/CodexMobile/Services/CodexService+RuntimeConfig.swift) confirmed that the phone loads its picker from `model/list` with `includeHidden: false`. That ruled out the theory that the iPhone app was hardcoding a smaller model list.
- Checking [`../phodex-bridge/src/codex-transport.js`](../phodex-bridge/src/codex-transport.js) confirmed that the bridge normally spawns `codex app-server` locally unless an explicit endpoint override is configured. That identified the real backend to inspect.
- Querying the spawned `codex app-server` directly before and after the CLI update showed the real difference: the older runtime did not advertise `gpt-5.4`, while the updated runtime did.
- Restarting the bridge after updating the CLI made the new catalog visible to Remodex.

### What did not work

- Debugging the model picker as if it were a phone-only UI problem did not help.
- Comparing only against the desktop app UI did not identify the real issue, because the desktop app and the bridge do not have to be using the exact same runtime binary or process.
- Fixing pairing and transport issues alone did not make `gpt-5.4` appear, because the spawned Codex runtime was still older.

### Why the previous fixes did not fully solve the problem

The earlier fixes were still correct, but they solved different layers:

- the launcher fix solved a macOS shell compatibility bug
- the LAN-IP + fresh-QR fix solved a transport reachability problem between iPhone and relay

Neither of those changes altered the model catalog exposed by `codex app-server`. Remodex was still talking to an older local Codex CLI, so the phone could reconnect successfully and still not see `gpt-5.4`.

### Recommended recovery flow for this failure mode

1. Check the local runtime version with `codex --version`.
2. If the version is old, update the local CLI, for example `bun add -g @openai/codex@latest` if `codex` is Bun-installed.
3. Restart the bridge or daemon so it spawns a fresh `codex app-server` from the updated CLI.
4. Reconnect the phone to that restarted bridge.
5. If the app is still pinned to an older stopped session, use `Scan New QR Code` and pair against the fresh bridge session.

### How to avoid this debugging loop next time

- Debug from the backend forward, not from the picker backward.
- Treat the output of `model/list` from the actual runtime behind the bridge as the source of truth.
- Verify `codex --version` on the Mac before assuming the phone app is missing models.
- After updating the local Codex CLI, always restart the bridge. A long-lived bridge can keep serving the old runtime until it is relaunched.
- Only investigate the iPhone UI after the backend runtime has been verified to advertise the expected model IDs.

## Minimal Summary

If you cloned the public repo, the supported self-hosting story is:

- run the relay yourself
- prefer a relay path reachable from iPhone over Tailscale or another stable private network
- point the bridge at your relay with `REMODEX_RELAY`
- scan the QR from the iPhone app once to trust the Mac
- let reconnect reuse that trusted Mac over the same relay
- remember that the built-in daemon path is currently macOS-only
- keep private hostnames and credentials out of the public repo
