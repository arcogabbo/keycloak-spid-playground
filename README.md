# Keycloak broker playground to external OIDC provider

Keycloak brokers External OIDC UAT and PROD and issues its own JWTs to the public
`portal` client.
There is **no JWKS repair proxy, DNS override, Java extension, or bundled BFF**.
Keycloak accesses the configured endpoints directly and validates
upstream signatures using the unmodified external provider JWKS.
Traefik terminates local HTTPS for Keycloak and a static portal callback page;
the page does not exchange codes or verify JWTs.

The playground uses Keycloak's native realm import and environment substitution.
Broker, client, user profile, claim, and audience settings are in
`config/oneid-realm.json`. Non-secret endpoint defaults are in `docker-compose.yml`;
set real values through environment variables or edit those non-secret defaults.

## Register and configure

| registration | Keycloak alias       | Redirect URI at external provider                                        |
| ------------ | -------------------- | ------------------------------------------------------------------------ |
| UAT client   | `external-oidc-uat`  | `https://localhost:8080/realms/oneid/broker/external-oidc-uat/endpoint`  |
| PROD client  | `external-oidc-prod` | `https://localhost:8080/realms/oneid/broker/external-oidc-prod/endpoint` |

The portal callback, `https://localhost:3000/callback`, is registered **in Keycloak**,
not in external provider. These three redirect URIs are different and not interchangeable.

Set the matching `UAT_OIDC_*` (or `PROD_OIDC_*`) values:

| Environment variable suffix   | Value                                                                       |
| ----------------------------- | --------------------------------------------------------------------------- |
| `ISSUER_URL`                  | Exact discovery `issuer`, including any trailing slash                      |
| `AUTHORIZATION_URL`           | Discovery `authorization_endpoint`, unchanged                               |
| `TOKEN_URL`                   | Discovery `token_endpoint`                                                  |
| `JWKS_URL`                    | Discovery `jwks_uri`, unchanged                                             |
| `CLIENT_ID` / `CLIENT_SECRET` | That environment's broker registration                                      |
| `CLIENT_AUTH_METHOD`          | `client_secret_post` or `client_secret_basic`, matching the registration    |

The provider's `idp` selector is not part of `AUTHORIZATION_URL`. Keycloak forwards
the `idp` query parameter from the client's authorization request to the external
provider (`forwardParameters` on the identity provider), so the client picks the SPID
provider per request. Keycloak forwards any value the client sends: validate it in the
client. If the request has no `idp`, none is sent upstream.

## Start

```sh
docker compose up -d
docker compose logs -f keycloak
```

Open <https://localhost:8080/admin/> and <https://localhost:3000/callback> once
before starting authentication and accept the local certificate warnings.
Traefik uses its automatically generated
self-signed default certificate, which is not browser-trusted and is for this
local experiment only. It may change when the Traefik container restarts.
Keycloak's public URL and broker callbacks use HTTPS on port 8080. Traefik
forwards requests to Keycloak over HTTP only inside the Docker network;
Keycloak's HTTP port is not published. `KC_PROXY_HEADERS=xforwarded` lets
Keycloak interpret Traefik's forwarded scheme, with its public hostname fixed
by `KEYCLOAK_URL` (default `https://localhost:8080`).
Unset an old HTTP `KEYCLOAK_URL` override or replace it with the HTTPS URL,
then recreate the containers. Register the HTTPS broker callbacks above in
External provider panel; the portal callback alone does not control the upstream redirect.

The static page preserves the callback query string in the browser address bar
for the manual code exchange. Callback access logging is disabled, and responses
use `Cache-Control: no-store` and `Referrer-Policy: no-referrer`. No Docker socket
is mounted into Traefik, and the callback backend has no published host port.

Admin Console: <https://localhost:8080/admin/>. The default admin username is
`admin`; the application realm is `oneid`, not `master`.

```sh
curl --insecure --fail --silent --show-error \
  https://localhost:8080/realms/oneid/.well-known/openid-configuration | jq
```

Keycloak persists its development database, signing keys, users, and sessions in
the Compose volume. **Startup import skips an already-existing realm.** Editing
the JSON or environment settings and restarting does not update the imported
realm. For a clean local re-import, the following deliberately deletes this
playground's volume, including users and signing keys; never use it against data
you need to retain:

```sh
docker compose down --volumes
docker compose up -d
```

Alternatively, update an existing realm through Keycloak's Admin Console or
Admin REST API. Do not put secret-bearing exports into version control.
If you already imported the old HTTP portal callback, update **Clients > portal >
Valid redirect URIs** to `https://localhost:3000/callback` before using this flow,
or perform the clean re-import described above. An exported
`PORTAL_REDIRECT_URI` overrides the new HTTPS default; unset an old HTTP value.

## Broker and claims

The request sequence follows the supplied diagram:

1. The client sends an authorization request to Keycloak's actual
   `/realms/oneid/protocol/openid-connect/auth` endpoint with `kc_idp_hint` (selects the
   Keycloak alias) and `idp` (SPID provider, forwarded upstream).
2. Keycloak redirects to that alias's external provider authorization endpoint, using
   its own upstream state, nonce, client credentials, and broker callback.
3. Provider redirects the browser back to the selected broker callback.
   Keycloak exchanges the **external** code at that environment's token endpoint
   and validates its ID token (issuer, audience, signature, expiration, nonce).
4. Keycloak imports ID-token claims and completes its first-broker-login flow,
   then redirects to the portal callback with a **new Keycloak authorization code**.
5. The portal exchanges that code with its PKCE verifier for Keycloak tokens.
   The BFF must validate the Keycloak access token, not the provider ID token.

Only `openid` is requested upstream. UserInfo calls are disabled so custom claims
are read from the validated ID token.

| Upstream ID-token claim | Keycloak user attribute               | Portal JWT claim |
| ----------------------- | ------------------------------------- | ---------------- |
| `email`                 | Built-in `email` (OIDC broker import) | `email`          |
| `name`                  | `identityName`                        | `name`           |
| `dateOfBirth`           | `dateOfBirth`                         | `dateOfBirth`    |
| `familyName`            | `familyName`                          | `familyName`     |
| `fiscalNumber`          | `fiscalNumber`                        | `fiscalNumber`   |

Custom attributes are declared in the user profile, are not user-editable, and
sync on each broker login (`FORCE`). Tokens are RS256-signed by Keycloak.

Usernames are namespaced by alias and upstream subject. Duplicate emails are
allowed and email login is disabled to keep UAT and PROD identities separate;
email is not trusted as an account-linking proof. This is a shared playground
realm, **not** a production environment isolation boundary. The standard first
broker login flow may display a profile or account-linking screen; no automatic
link-by-email flow is installed.

## Usage

1. Start the services with docker and then use the Bruno API Client to start the flow.
   Set the Bruno environment variable `idp` to choose the SPID provider (empty to omit it).
2. Visit the URL logged into the Bruno console and visit it with a browser of choice. The browser will redirect to the external OIDC provider for authentication.
3. Complete the login flow.
4. You will be redirected to the portal callback page, which will display the query string containing the authorization code.
5. Copy the authorization code from the query string and use it to exchange for Keycloak tokens using the Bruno API Client (env var `authorization_code`).
6. Call Token Endpoint to get Keycloak tokens.

## Official Keycloak references

- [OIDC identity providers and broker settings](https://www.keycloak.org/docs/latest/server_admin/index.html#_identity_broker_oidc)
- [First broker login](https://www.keycloak.org/docs/latest/server_admin/index.html#_identity_broker_first_login)
- [Realm import and environment placeholders](https://www.keycloak.org/server/importExport)
- [Container setup](https://www.keycloak.org/server/containers)
- [OIDC endpoints](https://www.keycloak.org/securing-apps/oidc-layers)
