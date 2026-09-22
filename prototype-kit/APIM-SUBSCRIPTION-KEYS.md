# Getting real APIM Subscription Keys working

This is developer setup notes, not part of the v2 site — it never gets exported to `docs/v2/`.
Nothing here needs a code change; `createApimSubscription` in `app/routes.js` already calls the
real Azure Resource Manager API whenever it has a usable credential. What's missing is the
credential itself.

## Current state

`getApimToken()` in `app/routes.js` tries two things, in order:

1. **`APIM_CLIENT_ID` / `APIM_CLIENT_SECRET`** — a client-credentials service principal. This is
   the real, permanent path. Not set today.
2. **The developer's own `az` CLI login** — a temporary, test-only fallback (`execSync('az account
   get-access-token ...')`). Only usable because the signed-in HMCTS account happens to have
   Contributor rights on `rg-sps-platform-sbox` via the `DTS AMp Developers` AD group. This is how
   the two end-to-end tests on 2026-09-18 and 2026-09-21 produced real Subscription Keys.

If neither is available, nothing mocks — the call fails. (An earlier version mock-generated a key
when `APIM_CLIENT_ID` was unset; that fallback was removed once the az-cli path proved the real
call works, so there's currently no silent mock path left in the code.)

## Option A — the real, permanent fix

Get a service principal created **in HMCTS's corporate Azure tenant** (`531ff96d-0ae9-462a-8d2d-bec7c0b42082`,
where `sps-api-mgmt-sbox` actually lives — not `hmctsextsbox`, the separate CIAM tenant this
prototype's own sign-in and app registrations use).

This needs someone with Application Administrator (or Global Administrator) rights in that
tenant — confirmed blocked for this account (`az ad app create` fails with `Insufficient
privileges to complete the operation`).

What to ask them for:

1. **Create an app registration** in tenant `531ff96d-0ae9-462a-8d2d-bec7c0b42082`, e.g.
   `amp-apim-subscription-issuer`.
2. **Create a client secret** for it.
3. **Grant it a role scoped to the APIM instance**, not the whole subscription — narrowest
   reasonable scope is the resource group:
   ```bash
   az role assignment create \
     --assignee <the new app's appId> \
     --role "API Management Service Contributor" \
     --scope "/subscriptions/bd2864ed-4f3e-45ed-9c6a-8d179674bab1/resourceGroups/rg-sps-platform-sbox"
   ```
4. Hand back the **Client ID**, **Client Secret**, and confirm the **tenant ID** (`531ff96d-...`).

Then, in `prototype-kit/.env`:

```
APIM_CLIENT_ID=<the new app's Client ID>
APIM_CLIENT_SECRET=<the new app's Client Secret>
```

`APIM_TENANT_ID`, `APIM_SUBSCRIPTION_ID`, `APIM_RESOURCE_GROUP`, `APIM_SERVICE_NAME` already default
to the right values (see `app/routes.js`) — only override them if pointing at a different instance.

Restart the Kit (`npm run kit`). No code change needed — `getApimToken()` picks up `APIM_CLIENT_ID`
automatically and stops needing the az-cli fallback.

## Option B — keep testing locally without that credential

Only works if the person running the prototype is signed into the Azure CLI with an HMCTS account
that already has Contributor (or better) on `rg-sps-platform-sbox`:

```bash
az login
az account set --subscription bd2864ed-4f3e-45ed-9c6a-8d179674bab1
```

Then run the Kit as normal — `getApimToken()` falls back to `az account get-access-token`
automatically when `APIM_CLIENT_ID` is unset.

This is **not** a substitute for Option A: it depends on one person's own directory group
membership, not a service credential, so it can't be handed to anyone else, can't run
unattended/in CI, and stops working the moment that access is reviewed or revoked. Treat it as
a way to prove the mechanism works end-to-end (which it now has, twice), not as how this should
actually ship.

## Verifying a Subscription Key is real, independently of the app's own claim

The confirmation screen labels a key as real or mock, but don't just trust that label — check it
against Azure directly:

```bash
TOKEN=$(az account get-access-token --resource https://management.azure.com --query accessToken -o tsv)
BASE="https://management.azure.com/subscriptions/bd2864ed-4f3e-45ed-9c6a-8d179674bab1/resourceGroups/rg-sps-platform-sbox/providers/Microsoft.ApiManagement/service/sps-api-mgmt-sbox"

# The subscription id the app computes is `${appName}-${oid-prefix}-${apiId}`, lowercased/sanitised
SUB_ID="<application-name>-<first-8-chars-of-oid>-<api-id>"

curl -s "$BASE/subscriptions/$SUB_ID?api-version=2022-08-01" -H "Authorization: Bearer $TOKEN"
curl -s -X POST "$BASE/subscriptions/$SUB_ID/listSecrets?api-version=2022-08-01" -H "Authorization: Bearer $TOKEN" -H "Content-Length: 0"
```

The `primaryKey` in the response should match what the confirmation page showed.

## Cleaning up test data

Every real end-to-end test creates a real Entra app registration, a real APIM subscription, and
Postgres rows. Delete all three afterwards:

```bash
# 1. Entra app registration (via Graph, using the onboarding credential in .env)
# 2. APIM subscription (DELETE the same $SUB_ID above, via ARM)
# 3. Postgres: delete from applications where client_id = '<the Client ID>';
#    (cascades to api_subscriptions)
```
