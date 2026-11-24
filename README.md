# CapacitorHttp: binary request bodies are lost or corrupted

Minimal reproduction for a Capacitor 8.6.2 bug. With `CapacitorHttp` enabled, the patched `fetch`
does not send binary request bodies as bytes:

- an `ArrayBuffer` body arrives as `{}` (Android, and iOS with `Content-Type: application/json`)
  or makes the request fail (iOS with `Content-Type: application/octet-stream`);
- a `Uint8Array` body is decoded as UTF-8 text, so non-UTF-8 bytes arrive replaced with U+FFFD.

`window.CapacitorWebFetch` (the WebView's original fetch) sends the same bytes correctly.

## Run it

```sh
npm install
npx cap sync
npx cap run android   # or: npx cap run ios
```

Tap **Run**. The app POSTs the same 8 bytes (`1f8b0800ff00807f`, gzip-like and not valid UTF-8)
to `https://httpbin.org/anything` four ways and shows what the server received. All of the app
code is in `www/index.html`; the only configuration is `CapacitorHttp.enabled: true` in
`capacitor.config.json`.

## Results (Capacitor 8.6.2)

| Request | Android 14 emulator (API 34) | iOS 27.2 simulator |
|---|---|---|
| patched `fetch`, `ArrayBuffer`, `application/octet-stream` | received `{}` | error: `CapacitorUrlRequestError error 0` |
| patched `fetch`, `Uint8Array`, `application/octet-stream` | received `1fefbfbd0800efbfbd00efbfbd7f` | received `1fefbfbd0800efbfbd00efbfbd7f` |
| patched `fetch`, `ArrayBuffer`, `application/json` | received `{}` | received `{}` |
| `window.CapacitorWebFetch`, `ArrayBuffer` (control) | received `1f8b0800ff00807f` ✅ | received `1f8b0800ff00807f` ✅ |

## Cause

`convertBody` in `native-bridge.js`:

- has no branch for `ArrayBuffer`, `ArrayBufferView` (other than `Uint8Array`) or `Blob`, so they
  fall through to `return { data: body, type: 'json' }` and are JSON-serialized to `{}`;
- runs `Uint8Array` through `new TextDecoder().decode(...)`, which is lossy for binary data.

Real-world trigger: the Amplitude browser SDK sends uploads of 2 KB or more as a gzip
`ArrayBuffer` with `Content-Type: application/json`, the third row above. On native with
`CapacitorHttp` enabled, those uploads arrive empty and are rejected.
