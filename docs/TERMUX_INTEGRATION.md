# Termux Integration Plan — RedMagic 7S Pro

Status: design baseline, implementation pending.

## Target topology

- Virtual Screen APK owns the Android virtual display, privileged display backend, capture/input and its authenticated MCP endpoint.
- Termux owns the Node.js Gateway, Godot MCP server process, project scripts, health checks and logs.
- Gateway binds only to `127.0.0.1:18765`.
- Virtual Screen MCP binds only to `127.0.0.1:8732` by default.
- Godot Editor MCP uses `127.0.0.1:6550`.
- No LAN binding or public tunnel is enabled by this integration by default.

## Device profile

- Target: RedMagic 7S Pro NX709S, Android 13, ARM64, Root backend.
- Virtual display profile: 2289x1080, rotation 1 (landscape). Runtime creation must still verify the device backend accepts this profile; do not assume `wm size` is available inside Termux.
- Existing Termux Gateway workspace: `~/virtual-screen-gateway`.
- Existing Gateway config: `~/virtual-screen-gateway/config/config.json` (mode 0600; never log or print bearer tokens).
- Existing endpoints: Virtual Screen `http://127.0.0.1:8732/mcp`, Gateway `http://127.0.0.1:18765/mcp`, Godot Editor `6550`.

## Required APK-side adaptations

1. Keep MCP loopback-only by default and preserve Bearer authentication.
2. Add a Termux integration status section: endpoint, listener status, display status/id, backend readiness, last startup error, and a copy-config action that does not reveal the token in logs.
3. Add an explicit one-tap `Start workspace` operation: start foreground service, start MCP listener, create the configured virtual display, then report the display id. Do not silently create duplicate displays; if one exists, reuse it.
4. Add recovery behavior for service/process restart: reconcile desired state with actual backend state; report `NO_VIRTUAL_SCREEN` distinctly from `MCP_LISTENER_STOPPED`.
5. Provide a stable local health/status interface for Termux scripts, while keeping control actions authenticated and lease-protected.
6. Optional Termux:API `RUN_COMMAND` integration may start/stop/check the Gateway only after the user enables Termux's external-app command permission. It must be optional; the APK must remain usable without Termux:API.

## Required Termux-side adaptations

1. Add a `vs-health` script that checks `127.0.0.1:8732` and `127.0.0.1:18765` without printing bearer tokens.
2. Add a `vs-gateway-start` script that starts the existing Gateway idempotently, verifies its PID and health endpoint, and never launches a second listener on port 18765.
3. Keep the Gateway's Virtual Screen token in its private config file with mode 0600; do not hardcode it into scripts, shell history, process arguments, or project documentation.
4. Reconnect to the APK MCP endpoint after service restart; distinguish HTTP 401 (bad/missing token) from connection refused (listener down).
5. Preserve the existing Godot MCP stdio-to-WebSocket bridge and editor port 6550.

## Acceptance tests

- Cold-start APK: one action starts foreground service, MCP listener, and one virtual display.
- Repeated start: no duplicate display; returns existing display id.
- Termux Gateway start: idempotent, loopback-only, health endpoint returns healthy.
- Authenticated MCP initialize and `tools/list` succeed from Termux.
- Termux can acquire/release a control lease, observe display 17 (or newly assigned id), launch Godot, and read the post-launch observation.
- Restart Gateway while APK remains alive: reconnect succeeds without recreating the display.
- Restart APK service while Gateway remains alive: Gateway detects listener loss and reconnects after recovery.
- Reboot/device process death: no claim of automatic recovery unless Android foreground-service and boot policies permit it; expose actionable status instead.

## Build constraints

The upstream project documents JDK 25, Gradle 9.7.1, AGP 9.4.1, Android SDK 37, and NDK 28.2.13676358. The current Termux host has JDK 21 and does not currently expose a complete Android SDK 37 installation. Do not claim a locally built APK until the required toolchain is provisioned and a build succeeds. Prefer CI build on Linux if Android SDK/JDK setup on-device remains brittle.

## Simplified Lab flavor (implementation started)

- `lab` product flavor uses application ID `io.github.lejw0925.virtualscreen.lab`, so it installs alongside the upstream app without replacing its data.
- Lab flavor bypasses the general-purpose first-run guide, defaults to Root, and defaults to the device-specific 2289x1080 / rotation 1 profile without viewer auto-fit.
- `standard` flavor preserves the upstream application ID and general-purpose defaults.
