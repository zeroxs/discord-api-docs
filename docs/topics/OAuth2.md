# OAuth2

OAuth2 enables application developers to build applications that utilize authentication and data from the Discord API. Within Discord, there are multiple types of OAuth2 authentication. We support the authorization code grant, the implicit grant, client credentials, and some modified special-for-Discord flows for Bots and Webhooks

## Shared Resources

The first step in implementing OAuth2 is [registering a developer application](#MY_APPLICATIONS/top) and retrieving your client ID and client secret. Most people who will be implementing OAuth2 will want to find and utilize a library in the language of their choice. For those implementing OAuth2 from scratch, please see [RFC 6749](https://tools.ietf.org/html/rfc6749) for details. After you create your application with Discord, make sure that you have your `client_id` and `client_secret` handy. The next step is to figure out which OAuth2 flow is right for your purposes.

###### OAuth2 URLs

| URL                                            | Description            |
| ---------------------------------------------- | ---------------------- |
| https://discordapp.com/api/oauth2/authorize    | Base authorization URL |
| https://discordapp.com/api/oauth2/token        | Token URL              |
| https://discordapp.com/api/oauth2/token/revoke | Revocation URL         |

> warn
> In accordance with [RFC 6749](https://tools.ietf.org/html/rfc6749), the [token URL](#DOCS_TOPICS_OAUTH2/shared-resources-oauth2-urls) **only** accepts a content type of `x-www-form-urlencoded`. JSON content is not permitted and will return an error.

###### OAuth2 Scopes

| Name                   | Description                                                                                                                                           |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| bot                    | for oauth2 bots, this puts the bot in the user's selected guild by default                                                                            |
| connections            | allows [/users/@me/connections](#DOCS_RESOURCES_USER/get-user-connections) to return linked third-party accounts                                      |
| email                  | enables [/users/@me](#DOCS_RESOURCES_USER/get-current-user) to return an `email`                                                                      |
| identify               | allows [/users/@me](#DOCS_RESOURCES_USER/get-current-user) without `email`                                                                            |
| guilds                 | allows [/users/@me/guilds](#DOCS_RESOURCES_USER/get-current-user-guilds) to return basic information about all of a user's guilds                     |
| guilds.join            | allows [/guilds/{guild.id}/members/{user.id}](#DOCS_RESOURCES_GUILD/add-guild-member) to be used for joining users to a guild                         |
| gdm.join               | allows your app to [join users to a group dm](#DOCS_RESOURCES_CHANNEL/group-dm-add-recipient)                                                         |
| messages.read          | for local rpc server api access, this allows you to read messages from all client channels (otherwise restricted to channels/guilds your app creates) |
| rpc                    | for local rpc server access, this allows you to control a user's local Discord client                                                                 |
| rpc.api                | for local rpc server api access, this allows you to access the API as the local user                                                                  |
| rpc.notifications.read | for local rpc server api access, this allows you to receive notifications pushed out to the user                                                      |
| webhook.incoming       | this generates a webhook that is returned in the oauth token response for authorization code grants                                                   |

> info
> `guilds.join` and `bot` require you to have a bot account linked to your application. Also, in order to add a user to a guild, your bot has to already belong to that guild.

## State and Security

Before we dive into the semantics of the different OAuth2 grants, we should stop and discuss security, specifically the use of the `state` parameter. [Cross Site Request Forgery](https://en.wikipedia.org/wiki/Cross-site_request_forgery), or CSRF, and [Clickjacking](https://en.wikipedia.org/wiki/Clickjacking) are security vulnerabilities that must be addressed by individuals implementing OAuth. This is typically accomplished using the `state` parameter. `state` is sent in the authorization request and returned back in the response and should be a value that binds the user's request to their authenticated state. For example, `state` could be a hash of the user's session cookie, or some other nonce that can be linked to the user's session.

When a user begins an authorization flow on the client, a `state` is generated that is unique to that user's request. This value is stored somewhere only accessible to the client and the user, i.e. protected by the [same-origin policy](https://en.wikipedia.org/wiki/Same-origin_policy). When the user is redirected, the `state` parameter is returned. The client validates the request by checking that the `state` returned matches the stored value. If they match, it is a valid authorization request. If they do not match, it's possible that someone intercepted the request or otherwise falsely authorized themselves to another user's resources, and the request should be denied.

While Discord does not require the use of the `state` parameter, we support it and highly recommend that you implement it for the security of your own applications and data.

## Authorization Code Grant

The authorization code grant is what most developers will recognize as "standard OAuth2" and involves retrieving an access code and exchanging it for a user's access token. It allows the authorization server to act as an intermediary between the client and the resource owner, so the resource owner's credentials are never shared directly with the client.

###### Authorization URL Example

```
https://discordapp.com/api/oauth2/authorize?response_type=code&client_id=157730590492196864&scope=identify%20guilds.join&state=15773059ghq9183habn&redirect_uri=https%3A%2F%2Fnicememe.website&prompt=consent
```

`client_id` is your application's `client_id`. `scope` is a list of [OAuth2 scopes](#DOCS_TOPICS_OAUTH2/shared-resources-oauth2-scopes) separated by url encoded spaces (`%20`). `redirect_uri` is whatever URL you registered when creating your application, url-encoded. `state` is the unique string mentioned in [State and Security](#DOCS_TOPICS_OAUTH2/state-and-security).

When someone navigates to this URL, they will be prompted to authorize your application for the requested scopes. On acceptance, they will be redirected to your `redirect_uri`, which will contain an additional querystring parameter, `code`. `state` will also be returned if previously sent, and should be validated at this point.

`prompt` controls how the authorization flow handles existing authorizations. If a user has previously authorized your application with the requested scopes and prompt is set to `consent`, it will request them to reapprove their authorization. If set to `none`, it will skip the authorization screen and redirect them back to your redirect URI without requesting their authorization. For passthrough scopes, like `bot` and `webhook.incoming`, authorization is always required.

###### Redirect URL Example

```
https://nicememe.website/?code=NhhvTDYsFcdgNLnnLijcl7Ku7bEEeee&state=15773059ghq9183habn
```

`code` is now exchanged for the user's access token by making a `POST` request to the [token URL](#DOCS_TOPICS_OAUTH2/shared-resources-oauth2-urls) with the following parameters:

- `client_id` - your application's client id
- `client_secret` - your application's client secret
- `grant_type` - must be set to `authorization_code`
- `code` - the code from the querystring
- `redirect_uri` - your `redirect_uri`
- `scope` - the scopes requested in your authorization url, space-delimited

###### Access Token Exchange Example

```python
API_ENDPOINT = 'https://discordapp.com/api/v6'
CLIENT_ID = '332269999912132097'
CLIENT_SECRET = '937it3ow87i4ery69876wqire'
REDIRECT_URI = 'https://nicememe.website'

def exchange_code(code):
  data = {
    'client_id': CLIENT_ID,
    'client_secret': CLIENT_SECRET,
    'grant_type': 'authorization_code',
    'code': code,
    'redirect_uri': REDIRECT_URI,
    'scope': 'identify email connections'
  }
  headers = {
    'Content-Type': 'application/x-www-form-urlencoded'
  }
  r = requests.post('%s/oauth2/token' % API_ENDPOINT, data=data, headers=headers)
  r.raise_for_status()
  return r.json()
```

You can also pass your `client_id` and `client_secret` as basic authentication with `client_id` as the username and `client_secret` as the password. In response, you will receive:

###### Access Token Response

```json
{
  "access_token": "6qrZcUqja7812RVdnEKjpzOL4CvHBFG",
  "token_type": "Bearer",
  "expires_in": 604800,
  "refresh_token": "D43f5y0ahjqew82jZ4NViEr2YafMKhue",
  "scope": "identify"
}
```

Having the user's access token allows your application to make certain requests to the API on their behalf, restricted to whatever scopes were requested. `expires_in` is how long, in seconds, until the returned access token expires, allowing you to anticipate the expiration and refresh the token. To refresh, make another `POST` request to the [token URL](#DOCS_TOPICS_OAUTH2/shared-resources-oauth2-urls) with the following parameters:

- `client_id` - your application's client id
- `client_secret` - your application's client secret
- `grant_type` - must be set to `refresh_token`
- `refresh_token` - the user's refresh token
- `redirect_uri` - your `redirect_uri`
- `scope` - the scopes requested in your authorization url, space-delimited

###### Refresh Token Exchange Example

```python
API_ENDPOINT = 'https://discordapp.com/api/v6'
CLIENT_ID = '332269999912132097'
CLIENT_SECRET = '937it3ow87i4ery69876wqire'
REDIRECT_URI = 'https://nicememe.website'

def refresh_token(refresh_token):
  data = {
    'client_id': CLIENT_ID,
    'client_secret': CLIENT_SECRET,
    'grant_type': 'refresh_token',
    'refresh_token': refresh_token,
    'redirect_uri': REDIRECT_URI,
    'scope': 'identify email connections'
  }
  headers = {
    'Content-Type': 'application/x-www-form-urlencoded'
  }
  r = requests.post('%s/oauth2/token' % API_ENDPOINT, data=data, headers=headers)
  r.raise_for_status()
  return r.json()
```

Boom; fresh [access token response](#DOCS_TOPICS_OAUTH2/authorization-code-grant-access-token-response)!

## Implicit Grant

The implicit OAuth2 grant is a simplified flow optimized for in-browser clients. Instead of issuing the client an authorization code to be exchanged for an access token, the client is directly issued an access token. The URL is formatted as follows:

###### Authorization URL Example

```
https://discordapp.com/api/oauth2/authorize?response_type=token&client_id=290926444748734499&state=15773059ghq9183habn&scope=identify
```

On redirect, your redirect URI will contain additional **URI fragments**: `access_token`, `token_type`, `expires_in`, `scope`, and [`state`](#DOCS_TOPICS_OAUTH2/state-and-security)(if specified). **These are not querystring parameters.** Be mindful of the "#" character:

###### Redirect URL Example

```
https://findingfakeurlsisprettyhard.tv/#access_token=RTfP0OK99U3kbRtHOoKLmJbOn45PjL&token_type=Bearer&expires_in=604800&scope=identify&state=15773059ghq9183habn
```

There are tradeoffs in using the implicit grant flow. It is both quicker and easier to implement, but rather than exchanging a code and getting a token returned in a secure HTTP body, the access token is returned in the URI fragment, which makes it possibly exposed to unauthorized parties. **You also are not returned a refresh token, so the user must explicitly re-authorize once their token expires.**

## Client Credentials Grant

The client credential flow is a quick and easy way for bot developers to get their own bearer tokens for testing purposes. By making a `POST` request to the [token URL](#DOCS_TOPICS_OAUTH2/shared-resources-oauth2-urls) with a grant type of `client_credentials`, you will be returned an access token for the bot owner. Therefore, always be super-extra-very-we-are-not-kidding-like-really-be-secure-make-sure-your-info-is-not-in-your-source-code careful with your `client_id` and `client_secret`. We don't take kindly to imposters around these parts.

You can specify scopes with the `scope` parameter, which is a list of [OAuth2 scopes](#DOCS_TOPICS_OAUTH2/shared-resources-oauth2-scopes) separated by spaces:

###### Client Credentials Token Request Example

```python
import base64

API_ENDPOINT = 'https://discordapp.com/api/v6'
CLIENT_ID = '332269999912132097'
CLIENT_SECRET = '937it3ow87i4ery69876wqire'

def get_token():
  data = {
    'grant_type': 'client_credentials',
    'scope': 'identify connections'
  }
  headers = {
    'Content-Type': 'application/x-www-form-urlencoded'
  }
  r = requests.post('%s/oauth2/token' % API_ENDPOINT, data=data, headers=headers, auth=(CLIENT_ID, CLIENT_SECRET))
  r.raise_for_status()
  return r.json()
```

In return, you will receive an access token (without a refresh token):

###### Client Credentials Access Token Response

```json
{
  "access_token": "6qrZcUqja7812RVdnEKjpzOL4CvHBFG",
  "token_type": "Bearer",
  "expires_in": 604800,
  "scope": "identify connections"
}
```

## Bots

So, what are bot accounts?

### Bot vs User Accounts

Discord's API provides a separate type of user account dedicated to automation, called a bot account. Bot accounts can be created through the [applications page](#MY_APPLICATIONS/top), and are authenticated using a token (rather than a username and password). Unlike the normal OAuth2 flow, bot accounts have full access to all API routes without using bearer tokens, and can connect to the [Real Time Gateway](#DOCS_TOPICS_GATEWAY/gateways). Automating normal user accounts (generally called "self-bots") outside of the OAuth2/bot API is forbidden, and can result in an account termination if found.

Bot accounts have a few differences in comparison to normal user accounts, namely:

1. Bots are added to guilds through the OAuth2 API, and cannot accept normal invites.
2. Bots cannot have friends, nor be added to or join Group DMs.
3. Bots do not have a maximum number of Guilds (unlike user accounts, which are limited to 100).
4. Bots have an entirely separate set of [Rate Limits](#DOCS_TOPICS_RATE_LIMITS/rate-limits).

### Bot Authorization Flow

Bot authorization is a special server-less and callback-less OAuth2 flow that makes it easy for users to add bots to guilds. The URL you create looks similar to what we use for full stack implementation:

###### URL Example

```
https://discordapp.com/api/oauth2/authorize?client_id=157730590492196864&scope=bot&permissions=1
```

In the case of bots, the `scope` parameter should be set to `bot`. There's also a new parameter, `permissions`, which is an integer corresponding to the [permission calculations](#DOCS_TOPICS_PERMISSIONS/permissions-bitwise-permission-flags) for the bot. You'll also notice the absence of `response_type` and `redirect_uri`. Bot authorization does not require these parameters because there is no need to retrieve the user's access token.

When the user navigates to this page, they'll be prompted to add the bot to a guild in which they have proper permissions. On acceptance, the bot will be added. Super easy!

If your bot is super specific to your private clubhouse, or you just don't like sharing, you can leave the `Public Bot` option unchecked in your application's settings. If unchecked, only you can add the bot to guilds. If marked as public, anyone with your bot's URL can add it to guilds in which they have proper permissions.

### Advanced Bot Authorization

Devs can extend the bot authorization functionality. You can request additional scopes outside of `bot`, which will prompt a continuation into a complete [authorization code grant flow](#DOCS_TOPICS_OAUTH2/authorization-code-grant) and add the ability to request the user's access token. If you request any scopes outside of `bot`, `response_type` is again mandatory; we will also automatically redirect the user to the first uri in your application's registered list unless `redirect_uri` is specified.

When receiving the access code on redirect, there will be additional querystring parameters of `guild_id` and `permissions`. The `guild_id` parameter should only be used as a hint as to the relationship between your bot and a guild. To be sure of the relationship between your bot and the guild, consider requiring the Oauth2 code grant in your bot's settings. Enabling it requires anyone adding your bot to a server to go through a full OAuth2 [authorization code grant flow](#DOCS_TOPICS_OAUTH2/authorization-code-grant). When you retrieve the user's access token, you'll also receive information about the guild to which your bot was added:

###### Extended Bot Authorization Access Token Example

```json
{
  "token_type": "Bearer",
  "guild": {
    "mfa_level": 0,
    "emojis": [],
    "application_id": null,
    "name": "SomeTest",
    "roles": [
      {
        "hoist": false,
        "name": "@everyone",
        "mentionable": false,
        "color": 0,
        "position": 0,
        "id": "290926798626357250",
        "managed": false,
        "permissions": 49794241
      }
    ],
    "afk_timeout": 300,
    "system_channel_id": null,
    "widget_channel_id": null,
    "region": "us-east",
    "default_message_notifications": 1,
    "embed_channel_id": null,
    "explicit_content_filter": 0,
    "splash": null,
    "features": [],
    "afk_channel_id": null,
    "widget_enabled": false,
    "verification_level": 0,
    "owner_id": "53908232999183680",
    "embed_enabled": false,
    "id": "2909267986347357250",
    "icon": null
  },
  "access_token": "zMndOe7jFLXGawdlxMOdNvXjjOce5X",
  "scope": "bot",
  "expires_in": 604800,
  "refresh_token": "mgp8qnvBwJcmadwgCYKyYD5CAzGAX4"
}
```

### Two-Factor Authentication Requirement

For bots with [elevated permissions](#DOCS_TOPICS_PERMISSIONS/bitwise-permission-flags) (permissions with a `*` next to them), we enforce two-factor authentication on the owner's account when added to guilds that have server-wide 2FA enabled.

## Webhooks

Discord's webhook flow is a specialized version of an [authorization code](#DOCS_TOPICS_OAUTH2/authorization-code-grant) implementation. In this case, the `scope` querystring parameter needs to be set to `webhook.incoming`:

###### URL Example

```
https://discordapp.com/api/oauth2/authorize?response_type=code&client_id=157730590492196864&scope=webhook.incoming&state=15773059ghq9183habn&redirect_uri=https%3A%2F%2Fnicememe.website
```

When the user navigates to this URL, they will be prompted to select a channel in which to allow the webhook. When the webhook is [executed](#DOCS_RESOURCES_WEBHOOK/execute-webhook), it will post its message into this channel. On acceptance, the user will be redirected to your `redirect_uri`. The URL will contain the `code` querystring parameter which should be [exchanged for an access token](#DOCS_TOPICS_OAUTH2/authorization-code-grant-access-token-exchange-example). In return, you will receive a slightly modified token response:

###### Webhook Token Response Example

```json
{
  "token_type": "Bearer",
  "access_token": "GNaVzEtATqdh173tNHEXY9ZYAuhiYxvy",
  "scope": "webhook.incoming",
  "expires_in": 604800,
  "refresh_token": "PvPL7ELyMDc1836457XCDh1Y8jPbRm",
  "webhook": {
    "name": "testwebhook",
    "url": "https://discordapp.com/api/webhooks/347114750880120863/kKDdjXa1g9tKNs0-_yOwLyALC9gydEWP6gr9sHabuK1vuofjhQDDnlOclJeRIvYK-pj_",
    "channel_id": "345626669224982402",
    "token": "kKDdjXa1g9tKNs0-_yOwLyALC9gydEWP6gr9sHabuK1vuofjhQDDnlOclJeRIvYK-pj_",
    "avatar": null,
    "guild_id": "290926792226357250",
    "id": "347114750880120863"
  }
}
```

From this object, you should store the `webhook.token` and `webhook.id`. To execute a particular webhook, send a `POST` request to:

###### Webhook Execution URL

```
https://discordapp.com/api/webhooks/WEBHOOK_ID/WEBHOOK_TOKEN
```

Any user that wishes to add your webhook to their channel will need to go through the full OAuth2 flow. A new webhook is created each time, so you will need to save the token and id. If you wish to send a message to all your webhooks, you'll need to iterate over each stored id:token combination and make `POST` requests to each one. Be mindful of our [Rate Limits](#DOCS_TOPICS_RATE_LIMITS/rate-limits)!

## Get Current Application Information % GET /oauth2/applications/@me

Returns the bot's OAuth2 application info.

###### Response Structure

| Field                  | Type                                                       | Description                                                                                                               |
| ---------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| id                     | snowflake                                                  | the id of the app                                                                                                         |
| name                   | string                                                     | the name of the app                                                                                                       |
| icon                   | ?string                                                    | the icon hash of the app                                                                                                  |
| description            | string                                                     | the description of the app                                                                                                |
| rpc_origins?           | array of strings                                           | an array of rpc origin urls, if rpc is enabled                                                                            |
| bot_public             | boolean                                                    | when false only app owner can join the app's bot to guilds                                                                |
| bot_require_code_grant | boolean                                                    | when true the app's bot will only join upon completion of the full oauth2 code grant flow                                 |
| owner                  | partial [user](#DOCS_RESOURCES_USER/user-object) object    | partial user object containing info on the owner of the application                                                       |
| summary                | string                                                     | if this application is a game sold on Discord, this field will be the summary field for the store page of its primary sku |
| verify_key             | string                                                     | the base64 encoded key for the GameSDK's [GetTicket](#DOCS_GAME_SDK_APPLICATIONS/get-ticket)                              |
| team                   | ?[team](#DOCS_TOPICS_TEAMS/data-models-team-object) object | if the application belongs to a team, this will be a list of the members of that team                                     |
| guild_id?              | snowflake                                                  | if this application is a game sold on Discord, this field will be the guild to which it has been linked                   |
| primary_sku_id?        | snowflake                                                  | if this application is a game sold on Discord, this field will be the id of the "Game SKU" that is created, if exists     |
| slug?                  | string                                                     | if this application is a game sold on Discord, this field will be the URL slug that links to the store page               |
| cover_image?           | string                                                     | if this application is a game sold on Discord, this field will be the hash of the image on store embeds                   |

###### Example Application Information

```json
{
  "bot_public": true,
  "bot_require_code_grant": false,
  "cover_image": "31deabb7e45b6c8ecfef77d2f99c81a5",
  "description": "Test",
  "guild_id": "290926798626357260",
  "icon": null,
  "id": "172150183260323840",
  "name": "Baba O-Riley",
  "owner": {
    "avatar": null,
    "discriminator": "1738",
    "flags": 1024,
    "id": "172150183260323840",
    "username": "i own a bot"
  },
  "primary_sku_id": "172150183260323840",
  "slug": "test",
  "summary": "This is a game",
  "team": {
    "icon": "dd9b7dcfdf5351b9c3de0fe167bacbe1",
    "id": "531992624043786253",
    "members": [
      {
        "membership_state": 2,
        "permissions": ["*"],
        "team_id": "531992624043786253",
        "user": {
          "avatar": "d9e261cd35999608eb7e3de1fae3688b",
          "discriminator": "0001",
          "id": "511972282709709995",
          "username": "Mr Owner"
        }
      }
    ]
  },
  "verify_key": "1e0a356058d627ca38a5c8c9648818061d49e49bd9da9e3ab17d98ad4d6bg2u8"
}
```
