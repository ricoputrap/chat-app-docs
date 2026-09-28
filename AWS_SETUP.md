# AWS setup

Create the DynamoDB tables, the login-token Lambda, and the AppSync API in the **same region**. Then copy the API URL, realtime URL, host, and API key into `backend/.env` and `frontend/.env`.

The browser talks to AppSync with the login JWT. The Go server talks to AppSync with the API key. The browser never receives that key.

## 1. DynamoDB

Use on-demand capacity. Names below are suggestions; each table is attached to AppSync as its own data source, so the physical name can differ as long as the keys match.

### `Chat`

| Key | Attribute | Type |
| --- | --- | --- |
| Partition key | `id` | String |
| GSI `loc-index` partition key | `loc_id` | Number |
| GSI `loc-index` sort key | `id` | String |

Projection on `loc-index`: **All**. The index name must be `loc-index`. `loc_id` must be a Number. A String key will not match the resolvers.

Attributes written by AppSync: `id`, `loc_id`, `type` (`general` or `group`), `group_id` (Number, `0` for a general chat), `created_by`, `created_at`, `is_active`, `last_message_at`. Timestamps are unix epoch milliseconds.

There is no close-chat API. To close a chat, set `is_active` to `false` on that item. A closed chat can still be read. A new general chat in a room, or a new group chat for the same group in that room, is rejected while the existing one is still active.

### `ChatMessage`

| Key | Attribute | Type |
| --- | --- | --- |
| Partition key | `chat_id` | String |
| Sort key | `sk` | String |

`sk` is `<created_at_ms>:<uuid>`. Other attributes: `msg`, `sender_id` (Number), `created_at` (Number).

### `UserChatRead`

| Key | Attribute | Type |
| --- | --- | --- |
| Partition key | `user_id` | Number |
| Sort key | `chat_id` | String |

Other attribute: `last_read_at` (Number). `user_id` must be a Number.

## 2. Lambda authorizer

Source: `appsync/authorizer/index.mjs`.

| Setting | Value |
| --- | --- |
| Runtime | Node.js 20.x or 22.x |
| Handler | `index.handler` |
| Environment variable | `JWT_SECRET` |

`JWT_SECRET` must be the same string as `JWT_SECRET` in `backend/.env`. The function does not read `appsync/.env`; that file is only a reminder of the value to put in the Lambda console.

The function checks the HS256 login token from the `Authorization` header. It accepts a raw JWT or a `Bearer` prefix. On success it returns `resolverContext`:

- `user_id` — string form of the user id
- `name`
- `group_ids` — JSON array string, for example `[1,2]`

AppSync only allows string values in `resolverContext`. The resolvers parse them back into numbers.

No VPC, database, or extra IAM permissions are required on the function itself. AppSync needs permission to invoke it. Creating the authorizer from the AppSync console can add that resource policy for you. If you attach the function manually, allow `lambda:InvokeFunction` from `appsync.amazonaws.com`.

## 3. AppSync API

Create a GraphQL API, then set authorization **before** saving the schema. The schema uses `@aws_lambda` and `@aws_api_key`, and the save fails if those modes are not on the API.

| Mode | Role |
| --- | --- |
| Default: **AWS Lambda** | The function from section 2. Used by the browser for queries and the message subscription. |
| Additional: **API key** | Used only by the Go server. |

Paste `appsync/schema.graphql` as the schema.

Create an API key (or use the one AppSync creates). Keys expire. Put it in `APPSYNC_API_KEY`. Do not put it in the frontend env file.

### What each auth mode is allowed to call

| Field | Lambda (browser JWT) | API key (Go) |
| --- | --- | --- |
| `listChats` | Yes. Only chats the user may see, with that user's `last_read_at`. | Yes. Every chat in the room, `last_read_at` `0`. |
| `listMessages` | Yes, after an access check. | No |
| `onNewChatMessage` | Yes, after an access check. | No |
| `createChat`, `createChatMessage`, `markChatRead` | No | Yes |

Go uses `listChats` with the API key so it can tell a missing chat from a forbidden one, and so it can reject a second active chat. A JWT call of `listChats` hides chats the user cannot access, which would make those checks wrong.

## 4. Data sources

Create three DynamoDB data sources in this API, one table each. Let AppSync create a service role, or use a role that can `GetItem`, `PutItem`, `UpdateItem`, and `Query` on all three tables and on `Chat`'s `loc-index`.

| Data source | Table |
| --- | --- |
| Chat | `Chat` |
| ChatMessage | `ChatMessage` |
| UserChatRead | `UserChatRead` |

## 5. Resolvers

Use the AppSync JavaScript runtime (`APPSYNC_JS`).

Pipeline resolvers use `appsync/resolvers/pipeline.js` as the pipeline request and response. That file returns the last function's result. Each function in the pipeline has its own data source and its own file.

Unit resolvers are a single file attached directly to the field.

| Field | Kind | Code, in order | Data source |
| --- | --- | --- | --- |
| `Query.listChats` | Pipeline | `functions/listChatsQuery.js`, then `functions/attachLastRead.js` | Chat, then UserChatRead |
| `Query.listMessages` | Pipeline | `functions/getChatForMessages.js`, then `functions/queryMessages.js` | Chat, then ChatMessage |
| `Mutation.createChat` | Unit | `functions/putChat.js` | Chat |
| `Mutation.createChatMessage` | Pipeline | `functions/putMessage.js`, then `functions/updateLastMessageAt.js` | ChatMessage, then Chat |
| `Mutation.markChatRead` | Unit | `functions/putRead.js` | UserChatRead |
| `Subscription.onNewChatMessage` | Unit | `functions/authorizeSubscription.js` | Chat |

`createChatMessage` writes the message and then sets `Chat.last_message_at` to that message's `created_at`. The subscription is tied to that mutation with `@aws_subscribe`, so connected browsers receive the new message. The subscription resolver only checks that this user may listen to that chat. Returning `null` from it allows the subscription.

`listMessages` returns newest first. The default limit is 50.

## 6. Env files

From the AppSync API settings page, copy:

| AppSync value | `backend/.env` | `frontend/.env` |
| --- | --- | --- |
| GraphQL endpoint, including `/graphql` | `APPSYNC_GRAPHQL_URL` | `VITE_APPSYNC_GRAPHQL_URL` |
| API key | `APPSYNC_API_KEY` | — |
| Realtime endpoint | — | `VITE_APPSYNC_REALTIME_URL` |
| GraphQL host, no scheme and no path | — | `VITE_APPSYNC_HOST` |

The realtime host swaps `appsync-api` for `appsync-realtime-api` and uses `wss`. The path stays `/graphql`.

```text
https://xxxxxxxx.appsync-api.ap-southeast-1.amazonaws.com/graphql
wss://xxxxxxxx.appsync-realtime-api.ap-southeast-1.amazonaws.com/graphql
xxxxxxxx.appsync-api.ap-southeast-1.amazonaws.com
```

`JWT_SECRET` in `backend/.env` and on the Lambda must match. Restart the Go server after changing `.env`. Restart `pnpm run dev` after changing the frontend `.env`.

## 7. Quick check

1. Sign in as `alice` / `alice` in the web app. The chat list loads rooms from Go and chats from `listChats`. An empty room is success. `Unauthorized` means the Lambda rejected the JWT, usually a mismatched `JWT_SECRET`.
2. Create a general chat in Room 1. Go calls `createChat` with the API key. A second general chat in Room 1 returns `409` until you set the first item's `is_active` to `false`.
3. Send a message with the chat open in two browsers. Both should see it without a refresh. That path is `POST /chats/{id}/messages` → `createChatMessage` → the `onNewChatMessage` subscription.
