# Reference: connecting a third-party account from a form ("GitHub connect")

A three-process app whose form connects the user's **GitHub** account through
OAuth instead of asking for a pasted API token: a form with an
`"ui:widget": "oauth"` field, a pre-render process that starts the flow, and a
completion process that exchanges the code, stores the token and hands the popup
back to the form. The pattern is the same for Slack, Jira, Google, DATEV or any
authorization-code provider — only the two provider URLs change. The contract
itself (options, callback page, `onComplete`) is owned by `fcode-forms`
§Connect an external account; this is the code.

**The platform runs the authorization.** You never build an authorization URL,
mint a `state` or generate a PKCE pair, and the completion process is not a
webhook. That is not a convenience: providers require the redirect URI to be
registered in advance, and every customer install is its own `deploy-`
workspace, so no URL of yours could ever be the registered one. The platform
holds a single registered URI and dispatches each callback to the installation
it belongs to.

## Architecture

```
FORM   star-repository (form, preRenderProcess: star-repository-prerender)
         │ the pre-render calls fcode.oauth.start() and hands the form the
         │ authorizationUrl it returned
         ▼
POPUP  code.factorialhr.com/platform/api/oauth/authorize?state=…
         │ the platform mints PKCE and redirects to the provider
         ▼
       github.com/login/oauth/authorize?…  → user approves
         │ provider redirects to the ONE registered URI,
         │ code.factorialhr.com/platform/api/oauth/callback
         ▼
CALLBACK  the platform verifies and spends the state, then invokes
          github-oauth-callback with code · codeVerifier · redirectUri · data
         │ exchanges code · stores token server-side
         │ 302 → https://code.factorialhr.com/sdk/oauth-callback.html?status=success&value=<login>
         ▼
FORM   onComplete: "reload" → pre-render runs again → "Connected as <login>"
       submit → star-repository verifies the connection, then calls GitHub
```

Workspace layout:

```
processes/star-repository/            # the form + the work; verifies the connection
processes/star-repository-prerender/  # starts the flow, reports the connected state
processes/github-oauth-callback/      # code → token → 302; no endpoint of its own
```

Team variables (`variables.env`; see `fcode-cli`): `GITHUB_CLIENT_ID`,
`GITHUB_CLIENT_SECRET` (sensitive). That is all — there is no state secret and
no redirect URI of your own, because neither is yours any more.

One thing still has to be set on the provider: the GitHub OAuth App's
**Authorization callback URL** must be
`https://code.factorialhr.com/platform/api/oauth/callback`, the platform's
single registered URI. GitHub matches the `redirect_uri` against it, so an
empty or stale value fails with `redirect_uri_mismatch` before the popup ever
reaches the platform.

## Triggers — `metadata.json`

The completion process is invoked by the platform, not called over HTTP, so it
declares **no webhook**. Nothing is exposed and there is nothing to
authenticate.

```json
// processes/github-oauth-callback/metadata.json
{ "name": "GitHub OAuth callback", "tags": ["github", "oauth"],
  "form": { "enabled": false } }
```

```json
// processes/star-repository-prerender/metadata.json — not a form itself
{ "name": "Star repository (pre-render)", "tags": ["github", "oauth"], "form": { "enabled": false } }
```

## The form — `processes/star-repository/parametersSchema.json`

Everything dynamic points at `#/variables`; the values in `variables` are the
fallbacks when the pre-render does not run. The `authorizationUrl` **must** be a
`$ref` — a `{{mustache}}` token would be HTML-escaped and refused.

```json
{
  "title": "Star a repository",
  "description": { "$ref": "#/variables/intro" },
  "type": "object",
  "preRenderProcess": "star-repository-prerender",
  "variables": {
    "authorizeUrl": "",
    "githubAccountDefault": "",
    "connectedLabel": "Connected",
    "intro": "Connect your GitHub account, then pick a repository."
  },
  "properties": {
    "github_account": {
      "title": "GitHub account",
      "description": "You will be asked to authorize in a new window.",
      "type": "string",
      "default": { "$ref": "#/variables/githubAccountDefault" },
      "ui": {
        "ui:widget": "oauth",
        "ui:options": {
          "authorizationUrl": { "$ref": "#/variables/authorizeUrl" },
          "connectLabel": "Connect GitHub",
          "connectedLabel": { "$ref": "#/variables/connectedLabel" },
          "onComplete": "reload"
        }
      }
    },
    "repository": {
      "title": "Repository to star",
      "description": "`owner/name`, e.g. `factorialco/f0`",
      "type": "string",
      "pattern": "^[\\w.-]+/[\\w.-]+$"
    }
  },
  "required": ["github_account", "repository"]
}
```

`github_account` is `required` and empty until the flow succeeds, so the form
cannot be submitted before connecting. After a `reload`, the pre-render fills
its `default` with the login, which is what renders the button as
"Connected as …".

The fallback for `authorizeUrl` is `""` on purpose, and the pre-render below
catches its own errors to reach it: an invalid URL renders the button disabled,
which is the right outcome when the flow cannot be started. It only works
because the pre-render returns — an error thrown out of it is answered with a
502 and the form is never served at all.

## Pre-render — start the flow, report the connected state

```javascript
// processes/star-repository-prerender/index.js
const CONNECTION_KEY = "github.connection";

async function main() {
  const connection = JSON.parse((await fcode.datastore.get(CONNECTION_KEY)) || "null");

  // Never throw: a pre-render that fails makes the form unopenable (the platform
  // answers the form request with a 502), so a transient API error would cost the
  // customer the whole settings page rather than just the button.
  let authorizeUrl = "";
  try {
    // Started on every render: a flow is single-use and expires on its own, so an
    // abandoned one costs nothing. Started even when already connected, so the user can
    // re-authorize from the same form.
    const flow = await fcode.oauth.start({
      authorizeUrl: "https://github.com/login/oauth/authorize",
      clientId: fcode.env.GITHUB_CLIENT_ID,
      scope: ["public_repo"],
      onComplete: "github-oauth-callback",
      // Carried back to the callback untouched. Identifiers only — never a secret.
      data: { startedBy: fcode.context.parameters?.userId ?? null },
    });
    authorizeUrl = flow.authorizationUrl;
  } catch (error) {
    console.error(`Could not start the GitHub OAuth flow: ${error.message}`);
  }

  return {
    variables: {
      authorizeUrl,
      githubAccountDefault: connection?.login ?? "",
      connectedLabel: connection ? `Connected as ${connection.login}` : "Connected",
      intro: connection
        ? `Connected as **${connection.login}**. Pick a repository to star.`
        : "Connect your GitHub account, then pick a repository.",
    },
  };
}

module.exports = { main };
```

`fcode.oauth` is backed by the workspace's meta token, like `fcode.schedule` and
`fcode.storage`, so it is **unavailable under a local `fcode run`** — exercise
the flow on the platform, from an installed app.

## Completion — exchange, store, redirect

```javascript
// processes/github-oauth-callback/index.js
const CONNECTION_KEY = "github.connection";
const SDK_CALLBACK = "https://code.factorialhr.com/sdk/oauth-callback.html";

const sdkRedirect = (params) => ({
  status: 302,
  headers: { Location: `${SDK_CALLBACK}?${new URLSearchParams(params)}` },
});

async function main() {
  const { code, codeVerifier, redirectUri, state, error } = fcode.context.parameters;

  // Echoed so the SDK page can bind the outcome to the popup the form opened.
  const echo = state ? { state } : {};

  if (error) {
    return sdkRedirect({ status: "error", message: "GitHub authorization was refused.", ...echo });
  }

  const token = await fetch("https://github.com/login/oauth/access_token", {
    method: "POST",
    headers: { Accept: "application/json", "Content-Type": "application/json" },
    body: JSON.stringify({
      client_id: fcode.env.GITHUB_CLIENT_ID,
      client_secret: fcode.env.GITHUB_CLIENT_SECRET,
      code,
      // The provider compares this byte for byte against the authorization request,
      // and only the platform knows what it sent.
      redirect_uri: redirectUri,
      code_verifier: codeVerifier,
    }),
  }).then((r) => r.json());

  if (!token.access_token) {
    return sdkRedirect({ status: "error", message: "GitHub did not return a token.", ...echo });
  }

  const user = await fetch("https://api.github.com/user", {
    headers: { Authorization: `Bearer ${token.access_token}`, "User-Agent": "fcode-app" },
  }).then((r) => r.json());

  // The token never reaches the browser: a sensitive variable or the datastore.
  await fcode.variables.set("GITHUB_ACCESS_TOKEN", token.access_token, { sensitive: true });
  await fcode.datastore.set(
    CONNECTION_KEY,
    JSON.stringify({ login: user.login, connectedAt: Date.now() })
  );

  // An opaque handle, never a token: it reaches the browser and travels in the submission.
  return sdkRedirect({ status: "success", value: user.login, ...echo });
}

module.exports = { main };
```

Whatever this returns becomes the response the popup follows, so every exit must
be one of these redirects — an HTML page or a traceback would strand the popup
instead.

## The form process — verify, then do the work

```javascript
// processes/star-repository/index.js
const CONNECTION_KEY = "github.connection";

async function main() {
  const { github_account, repository } = fcode.context.parameters;

  // The connected state in the browser is a signal, not a proof.
  const connection = JSON.parse((await fcode.datastore.get(CONNECTION_KEY)) || "null");
  if (!connection || connection.login !== github_account || !fcode.env.GITHUB_ACCESS_TOKEN) {
    return {
      status: 400,
      body: { formErrors: { fields: { github_account: "Connect your GitHub account first." } } },
    };
  }

  const response = await fetch(`https://api.github.com/user/starred/${repository}`, {
    method: "PUT",
    headers: {
      Authorization: `Bearer ${fcode.env.GITHUB_ACCESS_TOKEN}`,
      "User-Agent": "fcode-app",
      "Content-Length": "0",
    },
  });
  if (!response.ok) {
    return { status: 400, body: { errorMessage: `GitHub answered ${response.status} for **${repository}**.` } };
  }

  return { message: `⭐ **${repository}** starred as **${connection.login}**.` };
}

module.exports = { main };
```

## Adapting to another provider — checklist

1. Swap the two provider URLs (authorize, token) and the user-info call. There
   is no `redirect_uri` to configure: register
   `https://code.factorialhr.com/platform/api/oauth/callback` with the provider
   and the platform replays that exact value into the token exchange.
2. Pass provider-specific authorization parameters as `extraParams` (a `nonce`,
   an `audience`, a tenant). They cannot override the parameters the protocol
   depends on.
3. **PKCE is on by default.** Pass `pkce: false` only for a provider that
   cannot cope with it. The verifier never travels through the browser and
   never reaches your code except in the callback's parameters.
4. Put routing and identifiers in `data` — a company id, a legal entity, which
   account is being connected — and **never a secret**: it is stored for the
   life of the flow. It comes back to the completion process untouched.
5. Store the token in a sensitive variable or the datastore with the encrypted
   flag (`fcode.datastore.set(key, token, true)`), never in the form value; put a
   handle (login, account id, connection id) in `value`.
6. Pick `onComplete`: `reload` when the connected state should change the form
   (as here); `submit` when connecting is the last thing the form does;
   `none` when the user still has fields to fill.
7. Verify the connection server-side in every process that trusts it — the
   field value is what the browser said, not what the callback stored.
8. For a marketplace app, store the connection per installation workspace (the
   `deploy-` workspace's own variables and datastore) and remember the form
   itself is public — the process authorizes the caller; add the connection's
   teardown to the uninstall process (see `references/custom-app-linear.md`).
9. Providers whose access tokens expire return a `refresh_token` — and many
   rotate it: persist the returned token set on every refresh before using the
   new access token.
10. For more than one connection per workspace, key the stored record (and
    token) per connection — an account id, a tenant id — instead of a single
    `CONNECTION_KEY`.
11. The connect field doesn't need a form of its own: it can sit directly in
    an install or settings form, or live in a dedicated connect process
    reached with `nextProcessId` (or opened directly as a user form) when
    other steps must run first.
