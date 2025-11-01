# Browser SDK Documentation

A lightweight JavaScript SDK to interact with the Mega‑Tera browser APIs from the browser or Node.js environments.

## Requirements

- Modern browser or Node.js 18+ (for built‑in `fetch`).
- A valid API key

## Quickstart

### In a browser (ES Modules)

```html
<!doctype html>
<html>
  <head><meta charset="utf-8" /><title>Browser SDK Quickstart</title></head>
  <body>
    <script type="module">
      import { BrowserSDK } from '../sdk.js'; // adjust path as needed

      const apiKey = '<YOUR_PUBLIC_OR_TEMPORARY_API_KEY>'; // Do not expose secrets publicly

      const sdk = new BrowserSDK(apiKey);

      async function main() {
        try {
          // 1) Create a session (automatically waits until it becomes active)
          const { sessionId } = await sdk.createSession({ region: 'us-central1' });
          console.log('Session created:', sessionId);

          // 2) Inspect the remote browser
          const info = await sdk.getBrowserInfo();
          console.log('Browser info:', info);

          // 3) Create a new page (tab)
          const page = await sdk.createPage('https://example.com');
          console.log('New page:', page);

          // 4) Optionally obtain a CDP endpoint (for Node tooling)
          const ws = await sdk.getBrowserCDPUrl();
          console.log('CDP URL:', ws);

          // 5) Cleanup
          await sdk.endSession();
          console.log('Session ended');
        } catch (err) {
          console.error('SDK error:', err);
        }
      }

      main();
    </script>
  </body>
</html>
```

### In Node.js (ESM)

```js
// Run with Node 18+ (fetch available). If using older Node, polyfill fetch.
import { BrowserSDK } from './sdk.js';

const sdk = new BrowserSDK(process.env.MEGA_TERA_API_KEY);

try {
  const { sessionId } = await sdk.createSession({ region: 'us-central1' });
  console.log('Session:', sessionId);

  const cdpUrl = await sdk.getBrowserCDPUrl();
  console.log('CDP URL:', cdpUrl);

  // Example: connect with Puppeteer
  // npm i puppeteer-core
  // import puppeteer from 'puppeteer-core';
  // const browser = await puppeteer.connect({ browserWSEndpoint: cdpUrl });
  // const page = await browser.newPage();
  // await page.goto('https://example.com');
  // await browser.disconnect();

  await sdk.endSession();
} catch (e) {
  console.error(e);
}
```

## API Reference

All methods throw on failure; wrap calls in try/catch and handle errors appropriately.

### new BrowserSDK(apiKey: string, baseURL?: string)
- apiKey: Bearer token string used for authorization.
- baseURL: Optional API origin. Defaults to `https://lisa-taurine.tera.space`.

Initializes the client. CDP routing uses the built‑in Milan host `gcp-usc1-1.milan-taurine.tera.space`.

### async createSession(targeting: object = {}): Promise<{ sessionId: string, ... }>
Creates a new browser session.
- Retries: Up to 3 attempts with incremental backoff.
- Post‑create: Waits until the session status becomes `active` (up to 20s) before resolving.
- Returns: The API response, including `sessionId`.

Example targeting fields (subject to backend support): `region`, `os`, `device`, etc.

### async endSession(): Promise<boolean>
Ends the current session. Throws if no active session or if the API call fails.

### async getSessionStatus(): Promise<Session>
Fetches the current session object (e.g., `{ status: 'active', ... }`). Throws if there is no active session.

### async getBrowserCDPUrl(): Promise<string>
Returns a secure WebSocket (WSS) DevTools endpoint to the remote browser.
- Internally adapts the local `ws://127.0.0.1` URL to a public `wss://<milan-host>/v1/consumer/<sessionId>/...` URL.
- Use with tools like Puppeteer/Playwright/Chrome DevTools.

### async getBrowserInfo(): Promise<object>
Returns the `/json/version` object from the remote browser (e.g., `Browser`, `Protocol-Version`, etc.).

### async createPage(url?: string): Promise<object>
Creates a new target (tab). Defaults to `about:blank` when `url` is not provided.
- Returns the target description object from the API.

### async closePage(targetId: string): Promise<boolean>
Closes a previously created target identified by `targetId`.

## Behavior, Errors, and Timeouts

- Session activation timeout: `createSession` waits up to 20 seconds for status `active`, then throws.
- Retries: `createSession` retries up to 3 times before failing.
- Common errors:
  - 401 Unauthorized: missing/invalid API key.
  - Network/CORS errors: browser could block cross‑origin requests if CORS isn’t enabled by the API.
  - 404/410: session expired or not found (ensure `sessionId` is current).
- State guards: Methods that require an active session throw if `sessionId` is not set.

## Security Notes

- Do not embed long‑lived or sensitive API keys in public web pages. Prefer ephemeral or scoped tokens, or route requests through your server.
- Treat CDP URLs as sensitive: anyone with the URL could control the browser session while it’s active.

## Usage Patterns

- Always wrap session lifecycle in a try/finally and call `endSession()` to avoid leaks.
- It’s safe to create multiple pages per session; store each returned `targetId` if you plan to close them individually.
- For automation, obtain the CDP URL and connect from a trusted Node environment.

## Troubleshooting

- "Session did not become active within 20 seconds":
  - Check quota, region capacity, or network egress rules.
  - Try again later or specify different targeting.
- CORS errors in the browser:
  - Ensure the API enables CORS for your origin; otherwise call the SDK from your backend.
- 401 Unauthorized:
  - Verify the `Authorization: Bearer <token>` header is correct and not expired.

## Minimal Example (All-in-one)

```js
import { BrowserSDK } from './sdk.js';

const sdk = new BrowserSDK('<API_KEY>');

try {
  const { sessionId } = await sdk.createSession();
  console.log('Active session:', sessionId);

  const page = await sdk.createPage('https://example.com');
  console.log('Created page:', page);

  console.log('CDP:', await sdk.getBrowserCDPUrl());
  console.log('Info:', await sdk.getBrowserInfo());
} finally {
  // Ensure cleanup even if errors occur
  try { await sdk.endSession(); } catch {}
}
```
