# Agent Note: Compare the /api Origin fence by hostname, not host:port

Status: implemented

English | [中文](2026-10-30-loopback-origin-hostname-trust.zh.md)

## Problem

The `/api` browser-trust fence rejected every same-origin request from a page served on a loopback address with a non-default port. Chrome serializes the `Origin` header of loopback origins without the port, so a page at `http://127.0.0.1:3080` sends `Origin: http://127.0.0.1` while the `Host` header carries `127.0.0.1:3080`. The fence compared `new URL(origin).host` to the parsed Host `host`, so the port-less Origin never matched the port-bearing Host and every `/api` request returned 403.

## Decision

The Origin fence compares the origin's hostname to the Host authority's hostname (`new URL(origin).hostname === hostUrl.hostname`) instead of their host:port forms. The Host fence, which runs first, already binds the request to a loopback or declared trusted hostname; the Origin fence's job is to reject a request whose browser-marked initiator names a different host, and a different host is a different hostname. The port adds no defense against either DNS rebinding or cross-site requests, so dropping it from the comparison changes no protected outcome.

## Alternatives considered

**Compare host and origin through the same default-port normalization.** Rejected because the divergence is not a default-port artifact — Chrome omits a non-default port for loopback, so normalizing port 80 to empty would not reconcile `http://127.0.0.1` with `127.0.0.1:3080`.

**Special-case only loopback origins.** Rejected because it would split the comparison into two code paths for no security gain, since the port is not load-bearing for either loopback or trusted-host deployments.

**Require the browser to send a port-bearing Origin.** Rejected because the server does not control Chrome's Origin serialization; refusing it leaves the web UI unusable on loopback.

## Verification

`api-request-trust.host.spec.ts` covers the port-less loopback Origin form for `127.0.0.1` and `localhost`, and the existing cross-site and rebinding cases still refuse. The change is a one-line hostname comparison with no new package surface.

## Consequences

Same-origin loopback and trusted-host browser requests are accepted whether or not the Origin includes the port, matching Chrome and every other browser that omits the default port. Cross-origin and cross-host requests remain refused because their hostname differs. The Origin fence is now hostname-scoped like the Host fence, and exact port matching for trusted authorities remains the Host fence's responsibility, unchanged.
