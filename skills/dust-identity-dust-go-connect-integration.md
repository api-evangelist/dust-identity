---
name: dust-go-connect-integration
description: Make a customer web app scan DUST identifiers when it runs inside the DUST Go mobile app, using the @dustid/dust-go-connect npm package — detect the app, capture scans (promise and listener APIs), handle DUST/QR/barcode/NFC payloads, resolve captures server-side against the DUST API, and handle OAuth redirect rewriting inside the WebView. Use this skill whenever you are adding DUST Go support to a web app, debugging the connector bridge, or wiring scan payloads to the /api/v1/tags endpoints.
version: 2026.7.4
---

# DUST Go integration (@dustid/dust-go-connect)

Generated from DUST API OpenAPI spec 2026.7.4 (147 paths), docs version 2026.7.4.

This skill is self-contained. Reference material: the tutorial at https://docs.dustid.io/integrate/dust-go-connect/, the npm package https://www.npmjs.com/package/@dustid/dust-go-connect, and the DUST API docs at https://docs.dustid.io (agent skill: `dice-api-integration`).

## What DUST Go is

DUST Go is DUST Identity's mobile app (App Store id `6636551540`). It is an embedded mobile browser: it loads the customer's ordinary web app in a WebView and exposes the device's DUST scanning hardware (camera + optical accessories) to the page through a small JavaScript bridge — the `@dustid/dust-go-connect` package. The web app never talks to scanning hardware directly and needs no native toolchain.

Constraints that shape integrations:

- **App-link allowlist.** DUST Go only opens URLs registered with DUST Identity (plus custom links added manually on-device for development). The customer's app URL must be registered before production use.
- **A DUST scan is a raw capture, not an identity.** The page receives a base64-encoded JPEG plus metadata. Identification / verification / binding happen server-side via the DUST API `/api/v1/tags/*` endpoints, which require server-held credentials.
- **Version pinning.** Use the latest published scanner-only version available from DUST, but avoid registry version `0.1.9` exactly — it contains a since-removed native-auth surface. If your registry only offers `0.1.9`, pin the latest earlier version or request the replacement package from DUST Identity.

## 1. Install and detect

```bash
npm install @dustid/dust-go-connect
```

The package is dependency-free and ships TypeScript types. The `connector` export is `undefined` outside DUST Go, so one build serves both regular browsers and DUST Go:

```ts
import { connector } from "@dustid/dust-go-connect";

export const insideDustGo = Boolean(connector);
```

- **Import timing:** detection happens at module-import time, in the browser. In SSR apps, gate all connector usage on hydration (the import is safe server-side; `connector` is just `undefined` there).
- **Server-side detection:** DUST Go tags the WebView User-Agent with `DustGo/<version> (<app scheme>)` — usable as a rendering hint before any JS runs, but spoofable; never treat it as a security boundary.

## 2. Capture a scan

### Promise API (single capture)

```ts
import { scanAsync } from "@dustid/dust-go-connect";

const payload = await scanAsync();
// payload: { type: 'DUST' | 'QR' | 'BARCODE' | 'DATA_MATRIX' | 'NFC',
//            data: string, metadata?: ScanMetadata }
```

`scanAsync()` opens the native scanner over the page and resolves with one `ScanPayload`. It **rejects** when the user closes the scanner without capturing, and rejects immediately when no connector is present — guard on `connector` first in shared code paths.

### Listener API (multi-scan; scanner stays open)

```ts
import { connector } from "@dustid/dust-go-connect";

connector?.add("my-listener", (event) => {
  switch (event.type) {
    case "scan": handleScan(event.payload); break;
    case "show": break; // scanner opened
    case "hide": break; // scanner closed
    default: break;     // ignore unknown types — the protocol may grow
  }
});
connector?.showScanner();
// later: connector?.hideScanner(); connector?.remove("my-listener");
```

`calibrationResult` events and `ackCalibrationResults()` are internal to DUST first-party workflows — ignore them.

### Payload contents

| `payload.type` | `payload.data` |
| --- | --- |
| `DUST` | Base64-encoded JPEG of the capture (large) — resolve server-side |
| `QR`, `BARCODE`, `DATA_MATRIX` | The decoded symbol contents |
| `NFC` | The hex id read from the NFC chip |

`payload.metadata` (`ScanMetadata`) carries device identifiers (`deviceId`, `modelName`, `osName`, `osVersion`, `appVersion`), lens/camera selection, and optionally optics (zoom/focus/exposure/ISO), geolocation (`latitude`/`longitude`/`accuracy`), and external-accessory capture-source fields (`captureSource`, `usbVendorId`, `usbProductId`, `dragonBackend`). Forward it opaquely when binding — the API stores it with the Identifier.

## 3. Resolve the capture against the DUST API

DUST captures must be sent as **binary multipart data** (decode the base64 first) to the identifier endpoints. The three operations:

- `POST /api/v1/tags/identify` — which Thread does this capture belong to? (no `threadId`; optional `searchGroupIds` JSON array of Team ids narrows the search)
- `POST /api/v1/tags/bind` — attach the capture to a Thread (`threadId` required; DUST binds may pass `options.enrollmentSessionId`, a client-generated UUID grouping multi-scan enrollment)
- `POST /api/v1/tags/verify` — does this capture match a specific Thread's identifiers? (`threadId` required)

```ts
async function identifyDustScan(base64Jpeg: string) {
  const bytes = Uint8Array.from(atob(base64Jpeg), (c) => c.charCodeAt(0));
  const form = new FormData();
  form.set("tagType", "DUST");
  form.set("data", new Blob([bytes], { type: "image/jpeg" }));
  // API field names keep legacy "group" naming — these are Team ids.
  form.set("searchGroupIds", JSON.stringify([TEAM_ID]));

  const response = await fetch(`${APID_URL}/api/v1/tags/identify`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Dust-Ctx-Org-Id": ORGANIZATION_ID,
    },
    body: form,
  });
  if (!response.ok) throw new Error(`identify failed: ${response.status}`);
  return await response.json();
  // → { type: "identified", identified: { tag, thread, ... } } on a confident match
  //   or { type: "matches", matches: [...] } for candidates
}
```

Text payloads (`QR`/`BARCODE`/`DATA_MATRIX`/`NFC`) send `payload.data` directly as the `data` form field — no decoding.

**Credentials are server-side only.** Never embed a long-lived DUST credential or token in the page. Have the customer's backend either proxy the `/api/v1/tags/*` calls or mint short-lived scoped tokens per browser session. Auth details (Service Account API key → `GET /api/auth/token` with `x-api-key` header → `Authorization: Bearer`, plus the `Dust-Ctx-Org-Id`/`Dust-Ctx-Team-Id` context headers) are covered by the `dice-api-integration` skill and https://docs.dustid.io/api/authentication/.

## 4. Geolocation

Standard `navigator.geolocation` works inside DUST Go — proxied through the native OS permission prompt. No connector code needed.

## 5. OAuth / OIDC inside DUST Go

DUST Go hands external identity-provider navigations to the system browser; the callback returns to the page via a custom URL scheme. Always wrap OAuth `redirect_uri` construction:

```ts
import { connector } from "@dustid/dust-go-connect";

const origin = window.location.origin;
const redirectUri = (
  connector?.rewriteRedirect(new URL(`${origin}/auth/callback`)) ??
  new URL(`${origin}/auth/callback`)
).href;
```

Outside DUST Go this is a no-op. Inside, it swaps the protocol for the app's custom scheme (e.g. `com.dustidentity.dustgo://your-host/auth/callback`; the exact scheme comes from the DUST Go build, `dustgo` is the fallback), which the identity provider must have registered as an allowed redirect URI.

### Logout and account switching

DUST Go's embedded WebView and the device's system browser do not necessarily share a cookie jar. Clearing the web app's session during an explicit logout therefore may leave the identity-provider session in the system browser intact, causing the next authorization request to sign the same account back in silently. To preserve normal SSO while still allowing account switching, set a short-lived, one-use first-party marker when the app logs out; consume it on the next login and add the standard OIDC `prompt=login` parameter to that authorization request. Do not add the prompt to every login, and do not rely on clearing an identity-provider cookie from the WebView.

**Known limitation:** current DUST Go builds rewrite the `redirect_uri` query parameter on *any* cross-origin navigation carrying one. Third-party providers that reject unregistered custom-scheme redirect URIs (most public IdPs) will fail to complete sign-in inside DUST Go. If the app needs third-party OAuth inside DUST Go, register the DUST Go schemes with the provider if possible, and contact DUST Identity.

## 6. Protocol versioning

The connector speaks protocol version 2: at import time it sends an automatic `hello` handshake advertising the events the page supports. Keep the package current; switch on `event.type` and ignore unknown event types.

## 7. Testing checklist

1. Install DUST Go from the App Store (`id6636551540`); DUST capture requires a supported optical accessory (see https://docs.dustid.io/integrate/supported-devices/).
2. Serve the app over HTTPS on a URL the device can reach (LAN is fine for development).
3. Add the URL as a custom app link in DUST Go and open it.
4. Confirm `connector` is defined, run `scanAsync()`, and verify the payload reaches the backend and resolves via `/api/v1/tags/identify`.
5. A diagnostic page exercising the whole bridge (detection, `scanAsync`, event log) is available in the package repository.

## Verification checklist for generated integrations

- All connector usage is guarded (`connector?.` or an `insideDustGo` check) so the code also runs in regular browsers.
- DUST payloads are base64-decoded to binary before multipart upload; text payloads are sent as-is.
- No DUST API key or long-lived token appears in client-side code.
- Unknown connector event types are ignored, not errors.
- The package version is the latest scanner-only release available from DUST, never exactly `0.1.9`.
