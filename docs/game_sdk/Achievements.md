# Achievements

> info
> Need help with the SDK? Talk to us at [dis.gd/devsupport](https://dis.gd/devsupport)

There's no feeling quite like accomplishing a goal that you've set out to achieve. Is killing 1000 zombies in a game as great an achivement as climbing Mt. Everest? Of course it is, and I didn't even have to leave my house. So get off my back, society.

Anyway—Discord has achievements! Show your players just how successful they are.

Achievements are managed in the [Developer Portal](https://discordapp.com/developers/applications). Head over to your application --> `Achievements` to create and manage achievements for your game. You'll give them an icon, a name, and a description; then they'll be assigned an id.

You can also mark achievements as `secret` and `secure`. "Secret" achievements will _not_ be shown to the user until they've unlocked them. "Secure" achievements can only be set via HTTP calls from your server, _not_ by a game client using the SDK. No cheaters here!

## Data Models

###### User Achievement Struct

| name            | type   | description                                                                                |
| --------------- | ------ | ------------------------------------------------------------------------------------------ |
| UserId          | Int64  | the unique id of the user working on the achievement                                       |
| AchievementId   | Int64  | the unique id of the achievement                                                           |
| PercentComplete | Int64  | how far along the user is to completing the achievement (0-100)                            |
| UnlockedAt      | string | the timestamp at which the user completed the achievement (PercentComplete was set to 100) |

## SetUserAchievement

Updates the current user's status for a given achievement.

Returns `Discord.Result` via callback.

###### Parameters

| name            | type  | description                            |
| --------------- | ----- | -------------------------------------- |
| achievementId   | Int64 | the id of the achievement to update    |
| percentComplete | Int64 | the user's updated percentage progress |

###### Example

```cs
achievementManager.SetUserAchievement(580159119969878046, 25, (res) =>
{
  if (res == Discord.Result.Ok)
  {
    Console.WriteLine("Achievement updated for user");
  }
});
```

## FetchUserAchievements

Loads a stable list of the current user's achievements to iterate over. Do your iteration within the callback of this function.

Returns `Discord.Result` via callback.

###### Parameters

None.

###### Example

```cs
achievementManager.FetchUserAchievements((res) =>
{
  if (res == Discord.Result.Ok)
  {
    // Count()
    // for() loop
  }
});
```

## CountUserAchievements

Counts the list of a user's achievements for iteration.

Returns `Int32`.

###### Parameters

None

###### Example

```cs
achievementManager.FetchUserAchievements((res) =>
{
  if (res == Discord.Result.Ok)
  {
    Console.WriteLine("User has {0} achievements for this game", achievementManager.CountUserAchievements());
  }
});
```

## GetUserAchievementAt

Gets the user's achievement at a given index of their list of achievements.

Returns `Discord.UserAchievement`

###### Parameters

| name  | type  | description                               |
| ----- | ----- | ----------------------------------------- |
| index | Int32 | the index at which to get the achievement |

###### Example

```cs
achievementManager.FetchUserAchievements((res) =>
{
  if (res == Discord.Result.Ok)
  {
    for (int i = 0; i < achievementManager.CountUserAchievements(); i++)
    {
      var achievement = achievementManager.GetUserAchievementAt(i);
      Console.WriteLine("Achievement progress for {0} for user {1}: {2}",
                        achievement.AchievementId,
                        achievement.UserId,
                        achievement.PercentComplete);
    }
  }
});
```

## GetUserAchievement

Gets the user achievement for the given achievement id. If you keep a hardcoded mapping of achievement <--> id in your codebase, this will be better than iterating over each achievement. Make sure to call `FetchUserAchievements()` first still!

###### Parameters

| name          | type  | description                      |
| ------------- | ----- | -------------------------------- |
| achievementId | Int64 | the id of the achievement to get |

###### Example

```cs
achievementManager.FetchUserAchievements((res) =>
{
  if (res == Discord.Result.Ok)
  {
    var achievement = achievementManager.GetUserAchievement(580159119969878046);
    Console.WriteLine("Achievement progress for {0} for user {1}: {2}",
                      achievement.AchievementId,
                      achievement.UserId,
                      achievement.PercentComplete);
  }
});
```

## OnUserAchievementUpdate

Fires when an achievement is updated for the currently connected user

###### Parameters

| name        | type                | description                      |
| ----------- | ------------------- | -------------------------------- |
| achievement | ref UserAchievement | the achievement that was updated |

## The API Way

Below are the API endpoints and the parameters they accept. If you choose to interface directly with the Discord API, you will need a token. This is a special authorization token with which your application can access Discord's HTTP API. You can retrieve it using the [Client Credentials OAuth2 Grant](#DOCS_TOPICS_OAUTH2/client-credentials/grant) by requesting the `applications.store.update` scope.

## Get Achievements

`GET https://discordapp.com/api/v6/applications/<application_id>/achievements`

Returns all achievements for the given application. This endpoint has a rate limit of 5 requests per 5 seconds per application.

###### Return Object

```json
[
  {
    "application_id": "461618159171141643",
    "name": {
      "default": "Win the Game"
    },
    "description": {
      "default": "You won!"
    },
    "secret": false,
    "icon_hash": "52c1636444f64ad7cb5368b158847def",
    "id": "580159119969878046",
    "secure": false
  }
]
```

## Get Achievement

`GET https://discordapp.com/api/v6/applications/<application_id>/achievements/<achievement_id>`

Returns the given achievement for the given application. This endpoint has a rate limit of 5 requests per 5 seconds per application.

###### Return Object

```json
{
  "application_id": "461618159171141643",
  "name": {
    "default": "Win the Game"
  },
  "description": {
    "default": "You won!"
  },
  "secret": false,
  "icon_hash": "52c1636444f64ad7cb5368b158847def",
  "id": "580159119969878046",
  "secure": false
}
```

## Create Achievement

`POST https://discordapp.com/api/v6/applications/<application_id>/achievements`

Creates a new achievement for your application. Applications can have a maximum of 1000 achievements. This endpoint has a rate limit of 5 requests per 5 seconds per application.

###### Parameters

| name        | type          | description                             |
| ----------- | ------------- | --------------------------------------- |
| name        | string        | the name of the achievement             |
| description | string        | the user-facing achievement description |
| secret      | bool          | if the achievement is secret            |
| secure      | bool          | if the achievement is secure            |
| icon        | ImageType (?) | the icon for the achievement            |

###### Return Object

```json
{}
```

## Update Achievement

`PATCH https://discordapp.com/api/v6/applications/<application_id>/achievements/<achievement_id>`

Updates the achievement for **\_ALL USERS\_\_**. This is **NOT** to update a single user's achievement progress; this is to edit the UserAchievement itself. This endpoint has a rate limit of 5 requests per 5 seconds per application.

###### Parameters

| name        | type          | description                             |
| ----------- | ------------- | --------------------------------------- |
| name        | string        | the name of the achievement             |
| description | string        | the user-facing achievement description |
| secret      | bool          | if the achievement is secret            |
| secure      | bool          | if the achievement is secure            |
| icon        | ImageType (?) | the icon for the achievement            |

###### Return Object

```json
{}
```

## Delete Achievement

`DELETE https://discordapp.com/api/v6/applications/<application_id>/achievements/<achievement_id>`

Deletes the given achievement from your application. This endpoint has a rate limit of 5 requests per 5 seconds per application.

###### Return Object

```json
// 204 No Content
```

## Update User Achievement

`PUT https://discordapp.com/api/v6/users/<user_id>/applications/<application_id>/achievements/<achievement_id>`

Updates the UserAchievement record for a given user. Use this endpoint to update `secure` achievement progress for users. This endpoint has a rate limit of 5 requests per 5 seconds per application.

###### Parameters

| name             | type | description                                            |
| ---------------- | ---- | ------------------------------------------------------ |
| percent_complete | int  | the user's progress towards completing the achievement |

###### Return Object

```json
{}
```

## Get User Achievements

`GET https://discordapp.com/api/v6/users/@me/applications/<application_id>/achievements`

Returns a list of achievements for the user whose token you're making the request with. This endpoint will **NOT** accept the Bearer token for your application generated via the [Client Crendentials Grant](#DOCS_TOPICS_OAUTH2/client-credentials-grant). You will need the _user's_ bearer token, gotten via either the [Authorization Code OAuth2 Grant](#DOCS_TOPICS_OAUTH2/authorization-code-grant) or via the SDK with [GetOAuth2Token](#DOCS_GAME_SDK_APPLICATIONS/get-oauth2-token). This endpoint has a rate limit of 2 requests per 5 seconds per application per user.

> info
> This endpoint will _not_ return any achievements marked as `secret` that the user has not yet completed.

###### Return Object

```json
[
  {
    "application_id": "461618159171141643",
    "name": {
      "default": "Win the Game"
    },
    "description": {
      "default": "You won!"
    },
    "secret": false,
    "icon_hash": "52c1636444f64ad7cb5368b158847def",
    "id": "580159119969878046",
    "secure": false
  }
]
```
