# Agent Note: Compare the /api Origin fence by hostname, not host:port

Status: implemented

English | [中文](2026-10-30-loopback-origin-hostname-trust.zh.md)

## Problem

The `/api` browser-trust fence rejected every same-origin request from a page served on a loopback address with a non-default port. Chrome serializes the `Origin` header of loopback origins without the port, so a page at `http://127.0.0.1:3080` sends `Origin: http://127.0.0.1` while the `Host` header carries `127.0.0.1:3080`. The fence compared `new URL(origin).host` to the parsed Host `host`, so the port-less Origin never matched the port-bearing Host and every `/api` request returned 403.

## Decision

The Origin fence always compares the origin's hostname to the Host authority's hostname; a different hostname is refused as a cross-site request. The port is then compared only for non-loopback origins. Chrome omits the port on loopback Origins (`http://127.0.0.1` for a page at `http://127.0.0.1:3080`), and every loopback port is the same local machine, so the port adds no defense there. A non-loopback trusted host still distinguishes origins by port — a page on `harness.internal:444` is cross-origin to a listener on `:443` — so the port is compared after default-port normalization.

## Alternatives considered

**Compare host and origin through the same default-port normalization.** Rejected because the divergence is not a default-port artifact — Chrome omits a non-default port for loopback, so normalizing port 80 to empty would not reconcile `http://127.0.0.1` with `127.0.0.1:3080`.

**Special-case only loopback origins.** Rejected because it would split the comparison into two code paths for no security gain, since the port is not load-bearing for either loopback or trusted-host deployments.

**Require the browser to send a port-bearing Origin.** Rejected because the server does not control Chrome's Origin serialization; refusing it leaves the web UI unusable on loopback.

## Verification

`api-request-trust.host.spec.ts` covers the port-less loopback Origin form for `127.0.0.1` and `localhost`, and the existing cross-site, rebinding, and non-loopback port-difference cases still refuse.

## Consequences

Same-origin loopback browser requests are accepted whether or not the Origin includes the port, matching Chrome. Non-loopback origins still match their host and port, so a different port on a trusted host remains refused. Cross-origin and cross-host requests remain refused because their hostname differs.
