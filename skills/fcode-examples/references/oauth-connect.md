# OAuth connect — linking a third-party account from install/settings forms

A complete app that connects a customer's third-party account (DATEV, Slack,
Xero, …) through the forms SDK's OAuth connect button: the install and settings
forms carry a Connect button that opens the provider's authorization page in a
popup, a public callback webhook exchanges the code for tokens server-side, and
the tokens are stored per connection for the app's data-plane calls. The flow
semantics and callback contract live in `fcode-forms` §Connect an external
account; the widget's schema options in `fcode-json-schema`. This reference
shows the *processes behind them*.

## Architecture

```
SETUP
  install ──nextProcessId──▶ oauth-connect ◀─preRenderProcess── oauth-connect-prerender
     │ stores the config          │ "Connect" button                │ mints state (+PKCE),
     ▼                            ▼                                 │ builds the authorize URL
                       provider's authorization page (popup)
                                  │ redirects with code + state
                                  ▼
  oauth-callback (public webhook) — burns the state, exchanges the code,
  stores tokens, 302 → /sdk/oauth-callback.html (echoing the state)

MAINTENANCE
  state-janitor  ◀── schedule: sweeps expired one-time states
  uninstall      ◀── appRole UNINSTALL: wipes config + tokens + states
```

Workspace layout:

```
processes/install/                  # config form; chains to oauth-connect
processes/oauth-connect/            # the form with the Connect button
processes/oauth-connect-prerender/  # mints state + authorize URL per render
processes/oauth-callback/           # the provider's redirect_uri
processes/state-janitor/            # scheduled sweep of orphaned states
processes/uninstall/                # teardown (see custom-app-linear)
modules/provider-oauth/             # everything OAuth: state, exchange, tokens
```

## Triggers per process — `metadata.json`

The connect form is reached by `nextProcessId` from install (or settings), and
is also directly openable as a user form. The callback must be a public
webhook: the provider redirects the user's **browser** to it, so no header can
authenticate the call — the single-use `state` is what ties it to a flow this
app started.

`processes/oauth-connect/metadata.json`:

```json
{
  "name": "Connect provider",
  "form": { "enabled": true, "appRole": "USER_FACING_FORM", "authMode": "FACTORIAL" }
}
```

`processes/oauth-callback/metadata.json`:

```json
{
  "name": "OAuth callback",
  "webhook": { "enabled": true }
}
```

## The module — state, exchange, token store

One module owns the whole OAuth surface. The app's OAuth client credentials
come from team variables (`PROVIDER_CLIENT_ID`, and `PROVIDER_CLIENT_SECRET`
as a sensitive variable).

```javascript
// modules/provider-oauth/index.js
const crypto = require("node:crypto");

const CONFIG_KEY = "provider.config";
const TOKENS_PREFIX = "provider.tokens.";   // one entry per connection
const STATE_PREFIX = "provider.oauth.state.";
const STATE_TTL_SECONDS = 15 * 60;          // must survive a slow login

const base64url = (buffer) => buffer.toString("base64url");

// Mint a single-use state and build the authorization URL. Called from the
// connect form's preRenderProcess, so every render carries a fresh state; the
// record keeps everything the callback needs (the PKCE verifier never travels
// through the browser).
async function beginAuthorization(connectionKey) {
  const verifier = base64url(crypto.randomBytes(32));
  const challenge = base64url(crypto.createHash("sha256").update(verifier).digest());
  const state = base64url(crypto.randomBytes(24));
  await fcode.datastore.set(
    STATE_PREFIX + state,
    JSON.stringify({ verifier, connectionKey, createdAt: Math.floor(Date.now() / 1000) })
  );
  const query = new URLSearchParams({
    response_type: "code",
    client_id: process.env.PROVIDER_CLIENT_ID,
    redirect_uri: redirectUri(),
    scope: "openid offline_access <product scopes>",
    state,
    code_challenge: challenge,
    code_challenge_method: "S256",
  });
  return `https://provider.example/oauth/authorize?${query}`;
}

// Validate and burn a state — single use, TTL bound.
async function consumeState(state) {
  if (!state) return null;
  const raw = await fcode.datastore.get(STATE_PREFIX + state);
  if (!raw) return null;
  await fcode.datastore.delete(STATE_PREFIX + state);
  const record = JSON.parse(raw);
  const age = Math.floor(Date.now() / 1000) - record.createdAt;
  return age <= STATE_TTL_SECONDS ? record : null;
}

// Exchange the authorization code and persist the token set.
async function exchangeCode(record, code) {
  const tokens = await postToken({
    grant_type: "authorization_code",
    code,
    redirect_uri: redirectUri(),
    code_verifier: record.verifier,
  });
  await storeTokens(record.connectionKey, tokens);
  return record.connectionKey;
}

// Many providers ROTATE refresh tokens: always persist the returned set
// before handing out the access token.
async function getValidAccessToken(connectionKey) {
  const record = JSON.parse(await fcode.datastore.get(TOKENS_PREFIX + connectionKey));
  if (Math.floor(Date.now() / 1000) < record.expiresAt - 60) return record.accessToken;
  const tokens = await postToken({
    grant_type: "refresh_token",
    refresh_token: record.refreshToken,
  });
  if (!tokens.refresh_token) tokens.refresh_token = record.refreshToken;
  await storeTokens(connectionKey, tokens);
  return tokens.access_token;
}
```

`postToken` is a plain `fetch` to the provider's token endpoint with HTTP
Basic client authentication; `storeTokens` JSON-encodes
`{ accessToken, refreshToken, expiresAt }` under `TOKENS_PREFIX +
connectionKey`; `redirectUri()` builds the `oauth-callback` webhook URL. The
tokens live only in the datastore — never in a form value, a message, or the
callback redirect.

## The connect form and its pre-render

`processes/oauth-connect/parametersSchema.json` — the authorization URL is
injected with `$ref` (never `{{…}}`; see `fcode-json-schema`), and the
`default` renders the button already connected on revisits:

```json
{
  "preRenderProcess": "oauth-connect-prerender",
  "title": "Connect your account",
  "description": "{{statusText}}",
  "type": "object",
  "properties": {
    "connection": {
      "title": "Provider account",
      "type": "string",
      "minLength": 1,
      "default": "{{connectionValue}}",
      "ui": {
        "ui:widget": "oauth",
        "ui:options": {
          "authorizationUrl": { "$ref": "#/variables/authorizeUrl" },
          "connectLabel": "Connect provider",
          "onComplete": "reload"
        }
      }
    }
  },
  "required": ["connection"]
}
```

```javascript
// processes/oauth-connect-prerender/index.js — runs on every form render
const provider = fcode.import("provider-oauth");

async function main() {
  const config = await provider.getConfig();
  const connection = await provider.getConnection(config.connectionKey);
  return {
    variables: {
      authorizeUrl: connection ? "" : await provider.beginAuthorization(config.connectionKey),
      connectionValue: connection ? config.connectionKey : "",
      statusText: connection ? "Connected." : "Authorize access to continue.",
    },
  };
}
```

With `onComplete: "reload"`, a completed popup refetches this definition: the
pre-render sees the stored tokens and renders the connected state. The submit
handler must still verify server-side (the field value is a signal, not
proof):

```javascript
// processes/oauth-connect/index.js
const connection = await provider.getConnection(config.connectionKey);
if (!connection) {
  return { status: 400, body: { formErrors: {
    fields: { connection: "The connection is not active — click Connect and approve access." } } } };
}
return { message: "Connected." };
```

## The callback webhook

```javascript
// processes/oauth-callback/index.js
const provider = fcode.import("provider-oauth");

const done = (params) => ({
  status: 302,
  headers: {
    Location: "https://code.factorialhr.com/sdk/oauth-callback.html?" +
      new URLSearchParams(params),
  },
});

async function main() {
  const { code, state, error, error_description } = fcode.context.parameters;
  const echo = state ? { state } : {};

  if (error) {
    await provider.consumeState(state); // burn it either way
    return done({ status: "denied", message: error_description || error, ...echo });
  }
  const record = await provider.consumeState(state);
  if (!record) {
    return done({ status: "expired", message: "This link is no longer valid — try connecting again.", ...echo });
  }
  const connectionKey = await provider.exchangeCode(record, code);
  return done({ status: "success", value: connectionKey, ...echo });
}
```

Everything sensitive happens server-side; the browser only ever carries the
`state` (spent by the time it is echoed) and the opaque `value`.

## Maintenance

- **`state-janitor`** — flows begun but never approved leave their single-use
  states behind; a scheduled process enumerates `provider.oauth.state.*` with
  `fcode.datastore.keys()` and deletes records past a max age. Schedule setup
  as in `custom-app-linear`.
- **`uninstall`** (`appRole: "UNINSTALL"`) — the deploy workspace's datastore
  outlives a marketplace uninstall, so without a wipe a reinstall opens
  already-connected: delete the config, every `provider.tokens.*` and every
  pending state. Teardown discipline in `custom-app-linear`.

## Adapting it

- **Register the callback URL with the provider.** The `redirect_uri` must
  match what the provider has on file. Providers that allow only one fixed
  redirect URL cannot carry per-customer routing in the URL — put it inside
  the `state` instead (sign the payload and give it an expiry; it transits the
  browser, so identifiers only, never secrets).
- **Scopes, PKCE and token rotation are provider-specific** — check whether
  the provider requires PKCE (keep it regardless; it costs nothing) and
  whether refresh tokens rotate (persist the returned set every time).
- **One connection or many**: key the token entries by whatever identifies a
  connection for your app (a tenant id, an account id) and store that key in
  the config the install form collects.
