# Reference: connecting a third-party account from a form ("GitHub connect")

A three-process app whose form connects the user's **GitHub** account through
OAuth instead of asking for a pasted API token: a form with an
`"ui:widget": "oauth"` field, a pre-render process that mints the authorization
URL, and a public callback webhook that exchanges the code, stores the token
and hands the popup back to the form. The pattern is the same for Slack, Jira,
Google or any authorization-code provider — only the two provider URLs change.
The contract itself (options, callback page, `onComplete`) is owned by
`fcode-forms` §Connect an external account; this is the code.

## Architecture

```
FORM   star-repository (form, preRenderProcess: star-repository-prerender)
         │ renders a Connect button whose URL the pre-render minted,
         │ with a per-render signed `state`
         ▼
POPUP  github.com/login/oauth/authorize?…&state=<nonce>.<hmac>
         │ user approves
         ▼
CALLBACK  github-oauth-callback (public GET webhook)
         │ verifies state · exchanges code · stores token server-side
         │ 302 → https://code.factorialhr.com/sdk/oauth-callback.html?status=success&value=<login>
         ▼
FORM   onComplete: "reload" → pre-render runs again → "Connected as <login>"
       submit → star-repository verifies the connection, then calls GitHub
```

Workspace layout:

```
processes/star-repository/            # the form + the work; verifies the connection
processes/star-repository-prerender/  # authorize URL, signed state, connected state
processes/github-oauth-callback/      # public GET webhook: code → token → 302
```

Team variables (`variables.env`; see `fcode-cli`): `GITHUB_CLIENT_ID`,
`GITHUB_CLIENT_SECRET` (sensitive), `OAUTH_STATE_SECRET` (sensitive, 32+ random
chars), `OAUTH_REDIRECT_URI` (the callback webhook URL, registered verbatim with
the provider: `https://code.factorialhr.com/platform/api/<team-slug>/webhooks/github-oauth-callback`).

## Triggers — `metadata.json`

The callback must be public: the provider redirects a browser to it, and a
redirect carries no header, so the process authenticates the request with the
signed `state` instead. Writing `"authMode": "NONE"` explicitly is the
documented way (`fcode-cli`).

```json
// processes/github-oauth-callback/metadata.json
{ "name": "GitHub OAuth callback", "tags": ["github", "oauth"],
  "webhook": { "enabled": true, "authMode": "NONE" }, "form": { "enabled": false } }
```

```json
// processes/star-repository-prerender/metadata.json — not a form itself
{ "name": "Star repository (pre-render)", "tags": ["github", "oauth"], "form": { "enabled": false } }
```

```json
// processes/star-repository/metadata.json
{ "name": "Star repository", "tags": ["github"], "form": { "enabled": true, "authMode": "FACTORIAL" } }
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
    "githubAuthorizeUrl": "https://github.com/login/oauth/authorize",
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
          "authorizationUrl": { "$ref": "#/variables/githubAuthorizeUrl" },
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

## Pre-render — mint the `state`, build the URL, report the state

```javascript
// processes/star-repository-prerender/index.js
const crypto = require("crypto");

const CONNECTION_KEY = "github.connection";
const STATE_TTL_SECONDS = 600;

const sign = (nonce) =>
  crypto.createHmac("sha256", fcode.env.OAUTH_STATE_SECRET).update(nonce).digest("hex");

async function main() {
  // Single-use, short-lived: the callback deletes it, the TTL expires it.
  const nonce = crypto.randomUUID();
  await fcode.datastore.set(`oauth.state:${nonce}`, "pending");
  await fcode.datastore.expire(`oauth.state:${nonce}`, STATE_TTL_SECONDS);

  const authorizeUrl = new URL("https://github.com/login/oauth/authorize");
  authorizeUrl.searchParams.set("client_id", fcode.env.GITHUB_CLIENT_ID);
  authorizeUrl.searchParams.set("redirect_uri", fcode.env.OAUTH_REDIRECT_URI);
  authorizeUrl.searchParams.set("scope", "read:user public_repo");
  authorizeUrl.searchParams.set("state", `${nonce}.${sign(nonce)}`);

  // Default missing state, never throw: a pre-render failure makes the form unopenable.
  const connection = JSON.parse((await fcode.datastore.get(CONNECTION_KEY)) || "null");

  return {
    variables: {
      githubAuthorizeUrl: authorizeUrl.toString(),
      githubAccountDefault: connection ? connection.login : "",
      connectedLabel: connection ? `Connected as ${connection.login}` : "Connected",
      intro: connection
        ? `GitHub is connected as **${connection.login}** (since ${connection.connectedAt}).`
        : "Connect your GitHub account, then pick a repository.",
    },
  };
}

module.exports = { main };
```

Only the login is reported back — the token stays server-side (pre-fill
contract in `fcode-forms`, `references/advanced.md`).

## Callback webhook — verify, exchange, store, redirect

```javascript
// processes/github-oauth-callback/index.js
const crypto = require("crypto");

const CONNECTION_KEY = "github.connection";

const donePage = (params) =>
  `https://code.factorialhr.com/sdk/oauth-callback.html?${new URLSearchParams(params)}`;
const redirect = (params) => ({ status: 302, headers: { Location: donePage(params) } });

// Signature check (constant-time) + single use: the nonce must still be in the datastore.
const verifyState = async (state) => {
  const [nonce, signature] = String(state || "").split(".");
  if (!nonce || !signature) return false;
  const expected = crypto
    .createHmac("sha256", fcode.env.OAUTH_STATE_SECRET).update(nonce).digest("hex");
  const a = Buffer.from(signature, "utf8");
  const b = Buffer.from(expected, "utf8");
  if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) return false;
  if (!(await fcode.datastore.get(`oauth.state:${nonce}`))) return false;
  await fcode.datastore.del(`oauth.state:${nonce}`);
  return true;
};

async function main() {
  // GET query parameters arrive as ordinary parameters.
  const { code, state, error, error_description } = fcode.context.parameters;

  if (error) return redirect({ status: "error", message: error_description || error });
  if (!(await verifyState(state))) {
    return redirect({ status: "error", message: "The authorization request expired. Try again." });
  }

  const tokenResponse = await fetch("https://github.com/login/oauth/access_token", {
    method: "POST",
    headers: { Accept: "application/json", "Content-Type": "application/json" },
    body: JSON.stringify({
      client_id: fcode.env.GITHUB_CLIENT_ID,
      client_secret: fcode.env.GITHUB_CLIENT_SECRET,
      code,
      redirect_uri: fcode.env.OAUTH_REDIRECT_URI,
    }),
  });
  const token = await tokenResponse.json();
  if (!token.access_token) {
    return redirect({ status: "error", message: token.error_description || "Token exchange failed." });
  }

  const user = await (await fetch("https://api.github.com/user", {
    headers: { Authorization: `Bearer ${token.access_token}`, "User-Agent": "fcode-app" },
  })).json();

  // The token never reaches the browser: a (sensitive by default) team variable
  // for the secret, the datastore for the public part the form displays.
  await fcode.variables.set("GITHUB_ACCESS_TOKEN", token.access_token);
  await fcode.datastore.set(
    CONNECTION_KEY,
    JSON.stringify({ login: user.login, id: user.id, connectedAt: new Date().toISOString() })
  );

  // `value` is an opaque handle the form can show and submit — never the token.
  return redirect({ status: "success", value: user.login });
}

module.exports = { main };
```

Every exit is a redirect to the callback page: with `status=error` the page
posts the `message`, which the form shows under the button, and the form reloads
its definition (in every `onComplete` mode) so this pre-render mints a fresh
nonce — necessary here, since `verifyState` deletes the nonce on first use and
the old URL is spent. The user retries with the new URL, typed values intact.
A thrown error would leave the popup on a platform error page instead. Note the
early `if (error)` return runs before `verifyState`: when the provider itself
denies, the nonce stays in the datastore until its TTL, which is harmless.

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

1. Swap the two provider URLs (authorize, token) and the user-info call; keep
   `redirect_uri` exactly as registered with the provider (`OAUTH_REDIRECT_URI`).
2. Keep the `state` discipline: random nonce, HMAC with a dedicated secret,
   datastore TTL, deleted on first use, constant-time comparison. Without it the
   public webhook is an open door.
3. Store the token in a sensitive variable or the datastore, never in the form
   value; put a handle (login, account id, connection id) in `value`.
4. Pick `onComplete`: `reload` when the connected state should change the form
   (as here); `submit` when connecting is the last thing the form does;
   `none` when the user still has fields to fill.
5. Verify the connection server-side in every process that trusts it — the
   field value is what the browser said, not what the callback stored.
6. For a marketplace app, give the form `authMode: FACTORIAL` and store the
   connection per installation workspace (the `deploy-` workspace's own
   variables and datastore); add its teardown to the uninstall process
   (see `references/custom-app-linear.md`).
