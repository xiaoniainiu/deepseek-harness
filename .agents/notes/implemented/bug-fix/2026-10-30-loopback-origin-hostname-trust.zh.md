# Agent Note: /api Origin 围栏按主机名而非 host:port 比较

Status: implemented

[English](2026-10-30-loopback-origin-hostname-trust.md) | 中文

## Problem

`/api` 浏览器信任围栏拒绝了所有来自非默认端口回环地址页面的同源请求。Chrome 序列化回环来源的 `Origin` 头时会省略端口，因此位于 `http://127.0.0.1:3080` 的页面发送的 `Origin: http://127.0.0.1` 不带端口，而 `Host` 头却携带 `127.0.0.1:3080`。围栏原来比较 `new URL(origin).host` 与解析后的 Host `host`，无端口的 Origin 永远无法匹配带端口的 Host，于是每个 `/api` 请求都被返回 403。

## Decision

Origin 围栏始终比较来源的主机名与 Host 授权的主机名，主机名不同即作为跨站请求拒绝。端口只在非回环来源上再比较。Chrome 会省略回环来源的端口（`http://127.0.0.1:3080` 页面发送 `http://127.0.0.1`），且回环的每个端口都是同一台本机，因此端口在此不提供额外防御。非回环的受信主机仍用端口区分来源——`harness.internal:444` 上的页面对 `:443` 的监听器是跨源的——所以端口在默认端口归一化后再比较。

## Alternatives considered

**用相同的默认端口归一化比较 host 与 origin。** 不采用，因为这里的差异不是默认端口造成的——Chrome 对回环地址连非默认端口也会省略，把端口 80 归一化为空无法调和 `http://127.0.0.1` 与 `127.0.0.1:3080`。

**只对回环来源做特判。** 不采用，因为会把比较拆成两条代码路径，却没有任何安全收益——端口对回环和受信主机部署都不承担防御作用。

**要求浏览器发送带端口的 Origin。** 不采用，因为服务端无法控制 Chrome 的 Origin 序列化，拒绝它会让回环上的 Web UI 完全不可用。

## Verification

`api-request-trust.host.spec.ts` 覆盖了 `127.0.0.1` 和 `localhost` 的无端口回环 Origin 形式，现有的跨站、rebinding 与非回环端口差异用例仍然拒绝。

## Consequences

同源的回环浏览器请求无论 Origin 是否带端口都会被接受，这与 Chrome 一致。非回环来源仍匹配主机与端口，因此受信主机上的不同端口仍被拒绝。跨源与跨主机请求仍因主机名不同而被拒绝。
