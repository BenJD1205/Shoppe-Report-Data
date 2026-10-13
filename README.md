# Shopee Shop Authorization

## Configuration

Set the following variables in `.env` and restart the server:

- `SHOPEE_ENV`: `sandbox` for test shops or `live` for production (default). Both authorization and financial APIs use this setting.
- `SHOPEE_PARTNER_ID` and `SHOPEE_PARTNER_KEY`: credentials for the selected environment. Sandbox requires Test Partner ID/Key; live requires live credentials. Legacy `SHOPEE_LIVE_PARTNER_ID`/`SHOPEE_LIVE_PARTNER_KEY` names remain supported when the corresponding generic variable is unset; their names do not select the environment. Rotate any exposed partner key first.
- `SHOPEE_REDIRECT_URL`: the callback registered in Shopee Open Platform, e.g. `http://localhost:5000/api/v1/shopee/callback`. Use a registered HTTPS URL when Shopee requires it; use the same browser origin throughout the flow.
- `SHOPEE_AUTH_ADMIN_TOKEN`: a random secret of at least 32 characters for the trusted operator, not a Shopee access token.
- `SHOPEE_TOKEN_ENCRYPTION_KEY`: a separate random 32-byte key encoded as 64 hex characters. Keep it stable across restarts and back it up securely. Changing it makes saved tokens unreadable.

Run `node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"` yourself locally, once for each new secret. Do not share the output or commit it.

## Connect a Shop

Authorization follows [Shopee's official guide](https://open.shopee.com/developer-guide/20): browser authorization uses `https://open.sandbox.test-stable.shopee.com/auth` in sandbox or `https://open.shopee.com/auth` in live, with `partner_id`, `auth_type=seller`, `redirect_uri`, `response_type=code` and `state`. Token and financial APIs use the separate configured API host; token requests still require HMAC signatures.

`POST /api/v1/shopee/authorize` requires `Authorization: Bearer <SHOPEE_AUTH_ADMIN_TOKEN>`. It returns an `authorizationUrl` and sets an HttpOnly browser cookie. No Shopee tokens are returned.

Open `/api/v1/shopee/connect` in the same browser that will log in to Shopee. Enter your `SHOPEE_AUTH_ADMIN_TOKEN` in the password field and select **Authorize Shop**. The form sends it only in the Authorization header, clears the input, and starts authorization with the same browser cookie. It does not embed the secret in HTML, URLs or browser storage. Do not use the browser console on a 404/error page: its restrictive CSP can block fetch requests. The connection page has a nonce-scoped CSP allowing only same-origin API requests.

If using an HTTPS tunnel, open `https://<your-tunnel-domain>/api/v1/shopee/connect`, not localhost, so the callback receives the same browser cookie. If ngrok displays its free-tier warning, select **Visit Site** first. Register that domain with Shopee and set the full callback URL, including `/api/v1/shopee/callback`, in `SHOPEE_REDIRECT_URL`. After changing configuration, restart the server and start a new authorization; old URLs retain their original host, callback and session state. Postman cookies are not shared with the browser, so do not start browser authorization from Postman. Use your configured origin/API version when different. In production, restrict this operator page to trusted users/network access; never embed the shared admin secret into public frontend code.

Log in as the seller and approve the app. Shopee redirects to `/api/v1/shopee/callback` with `code`, `shop_id`, and the original `state` authorization parameter. The callback checks the state and browser cookie, exchanges the code, and saves encrypted tokens. It returns only `{ success: true, shopId }`. If the redirect loses the state or cookie, authorization fails closed. Pending sessions last ten minutes and are lost on server restart; start again in that case.

For sandbox, use the Shop Account and Shop Password from your Open Platform test shop, not a production seller account or developer login. If the page shows a main-account login hint such as `XXX:main`, select **Switch to Shop Account** first. This backend supports single-shop authorization via `shop_id`, not main-account multi-shop callbacks via `main_account_id`. If login still fails, verify the account using the test shop's **Login Seller Center** entry or reset/recreate the test shop credentials in the Console; do not send passwords in chat.

Use sandbox while testing a Developing app with Test Partner ID/Key. To work with real orders, go live according to Shopee's requirements, obtain live credentials, register the Live Redirect URL Domain, set `SHOPEE_ENV=live`, and authorize the real seller shop. Changing the environment alone does not make test credentials or accounts valid in production.

Do not log callback query strings in a reverse proxy or hosting platform: they contain the authorization code. Application access logs skip this callback.

## Extract Orders

In Postman, send `POST http://localhost:5000/api/v1/extract-orders`:

- Authorization: Bearer Token = your `SHOPEE_AUTH_ADMIN_TOKEN`.
- Body: form-data, `pdfFile` and `excelFile` as File fields.
- Body: `shopId` as Text with the connected shop ID; `channel` as Text with `Shopee` (optional default).
- Do not send `accessToken`. Let Postman set the multipart Content-Type boundary.

The backend obtains the saved access token and refreshes it within 60 seconds of expiry, using Shopee's `expire_in`. Both newly returned tokens are saved atomically. Concurrent requests for the same shop are serialized within this Node process. A disconnected shop returns 409; rejected token requests return 401 and require reauthorization; transport failures return 502. If Shopee revokes an otherwise unexpired token, reconnect the shop; financial API failures still use the adapter's existing per-order FAIL status.

## Storage and Deployment

Encrypted tokens live in `.data/shopee/<environment>/<partnerId>/`, excluded from Git along with `.env` and credential files. Each app and environment has a separate store to avoid mixing live and test tokens. Tokens from the previous flat directory layout are not reused; reconnect those shops once. Persist this directory across restarts/deployments and restrict its Windows filesystem ACL to the application account (POSIX mode flags do not enforce Windows ACLs). Filesystem permissions and encryption do not replace host security.

This implementation is for one trusted operator and one Node process. Do not run multiple workers/instances against this store: production scaling requires shared storage and a distributed per-shop refresh lock, plus user-to-shop ownership checks instead of a shared admin token. Reauthorize if a process crashes after Shopee rotates a refresh token but before it can be persisted.

Run focused mocked tests with `node --test test/shopee-auth.test.js`. Live authorization still requires your configured Shopee app and seller approval."# Shoppe-Report-Data" 
