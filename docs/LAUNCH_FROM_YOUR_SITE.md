# Launch a token on YeetLaunch from your site

You can let people create a token on your site and launch it on YeetLaunch. Your server asks YeetLaunch for a signing link, the creator opens it and approves the launch in their own wallet on yeetlaunch.io, and your server checks the result.

Your site never handles keys or funds. The creator pays the 0.03 SOL launch fee from their own wallet, and the token is minted with its mint and freeze authorities revoked and its pool opened in the same flow.

If you just want to create a token yourself, use **[yeetlaunch.io/token-creator](https://yeetlaunch.io/token-creator)**.

## 1. Create a signing link (from your server)

```http
POST https://yeetlaunch.io/api/sign-request
Content-Type: application/json

{
  "kind": "create",
  "wallet": "<creator's public address>",
  "source": "yoursite",
  "createParams": {
    "name": "Moon Cat",
    "symbol": "MOONCAT",
    "description": "A cartoon astronaut cat.",
    "logoPrompt": "a cartoon cat in an astronaut helmet"
  }
}
```

Response:

```json
{ "success": true, "data": { "id": "v355dq5g8r", "url": "https://yeetlaunch.io/sign/v355dq5g8r", "expiresInSeconds": 900 } }
```

Send the creator to `data.url` exactly as returned. Don't build the URL yourself.

| Field | Required | Notes |
|---|---|---|
| `kind` | yes | `"create"` |
| `createParams.name` | yes | 1 to 32 characters |
| `createParams.symbol` | no | 2 to 10 letters or digits. Derived from the name if left out |
| `createParams.description` | no | Up to 280 characters |
| `createParams.logoPrompt` | no | Up to 400 characters. YeetLaunch draws the logo from it if no image is given |
| `createParams.logoBase64` | no | Base64 or a data URL. Takes priority over the other logo fields |
| `createParams.logoUrl` | no | Direct https link to an image file. It is copied to yeetlaunch.io |
| `createParams.initialBuySol` | no | 0 to 10 SOL, bought after launch. Adds a second wallet prompt |
| `wallet` | recommended | The creator's public address. Needed for reliable status |
| `source` | no | Your site's name, for support |
| `returnUrl` | no | Adds a "Back to your site" button after launch. See below |

Never ask for, accept or store a private key or seed phrase. Only the public address is needed.

## 2. Check the result (from your server)

```http
GET https://yeetlaunch.io/api/sign-request/<id>/status?wallet=<creator's public address>
```

| `state` | Meaning | What to do |
|---|---|---|
| `live` | The token and its pool exist. The response includes `mintAddress`, `poolAddress`, `tokenUrl` and `creatorWallet` | Check `creatorWallet` is your creator, then save the mint and pool in your own database |
| `launching` | A launch has started and isn't finished yet | Keep polling |
| `unknown` | No result can be shown for this link and wallet | Keep polling, then show "check your wallet". Don't charge again |

**`launching` and `unknown` never prove the fee wasn't paid.** Never start another launch automatically because of either state, a 404, a 5xx or a timeout, because a retry could charge the creator twice. Let the creator decide to try again.

Always pass the creator's wallet. A link can be shared, and the wallet picks the right result.

## 3. Return link (optional)

`returnUrl` must be an `https` URL on a site YeetLaunch has approved. Anything else returns 400. The match is on the exact hostname, so `www.yoursite.com` and `yoursite.com` are approved separately, and custom ports and credentials are refused.

The creator sees a "Back to yoursite.com" button after the launch succeeds. There is no automatic redirect. Every path on an approved host is trusted, so make sure your site has no open redirect.

To get a site approved, contact YeetLaunch at [yeetlaunch.io](https://yeetlaunch.io).

## Limits

- Links last 15 minutes and work once. Create one when the creator clicks Launch, not on page load.
- Each wallet can start 5 launches an hour and 10 a day.
- 20 links a minute per IP address.
- Status records are kept for 7 days and are best effort, so save the mint as soon as you see `live`.
- Call these endpoints from your server. Browser calls from other sites are refused (403), and the signing page can't be shown in an iframe.

## TypeScript SDK

[`@yeetlaunch/yeet-amm-sdk`](https://github.com/YeetLaunch/yeet-amm-sdk) version 0.1.3 or later wraps both calls:

```ts
import { RestClient } from '@yeetlaunch/yeet-amm-sdk';

const rest = new RestClient();
const link = await rest.createLaunchLink({ name: 'Moon Cat', logoPrompt: 'a cartoon cat in an astronaut helmet', wallet: creatorWallet, source: 'yoursite' });
// Send the creator to link.url, then:
const status = await rest.getLaunchStatus(link.id, { wallet: creatorWallet });
```
