# Chat App

A simple chat app where users can send messages each other in a specific building/organization.

## Functional Requirements

1. As a user, I can **create** a new **general chat** where anyone can access the chat.
2. As a user, I can **create** a new **chat** for a **specific group** of users where only those who are part of the group can access the chat.
3. As a user, I can **send a new message** in the chat.
4. As a user, when I am currently opening a chat, I should be able to see any new chat message in real-time.
5. As a user, I should be able to see a list of all chats that I am allowed to access to.
6. As a user, I should be able to see any "new message" indicator on each chat item/row in the list. This indicator will be removed once I click on it to open the chat.
7. As a user, I should be able to login with my username & password.

## Non-Functional Requirements

1. Exploration / prototype scope — not production-hardened.
2. Expected load is small (a handful of concurrent users).
3. Real-time message delivery should feel near-instant for users who have the chat open.
4. Users, groups, and locations are in-memory and reset when the backend restarts.
5. All timestamp attributes use unix epoch milliseconds.

## Notes

1. Every user is part of a group.
2. A user can be part of more than 1 group.
3. Each chat is created for a specific location/room in the building (e.g. Room 1, Room 2, Room 3, etc).
4. One location can have multiple chats:
   - at most **one active general chat**
   - several active group chats, but at most **one active chat per group** in that location
5. A new general chat in a location is forbidden while another general chat there is still active. The active one must be **closed** first.
6. A new group chat for a group in a location is forbidden while that group already has an active chat there (e.g. "Group XYZ Chat"). The active one must be **closed** first.
7. A closed chat accepts no new messages. Users can still open it and read its message history.
8. Users, groups, and locations are static data that are stored in-memory in the backend app.
9. The frontend (mobile-view) will be built using React JS + TypeScript + TanStack Query + TanStack Router.
10. The backend will be built using Golang with basic `net/http` package.
11. Chat, ChatMessage, and UserChatRead are stored in AWS DynamoDB.
12. Chat / ChatMessage query and ChatMessage real-time subscription are handled with AWS AppSync GraphQL API.
13. **Create chat** goes through the Go backend (backend writes to DynamoDB through AppSync).
14. **List chats** is queried directly from the browser to AppSync GraphQL.
15. The frontend listens to new chat messages directly via AppSync subscription, but **sending a new message** goes to the Go backend first.
16. The "new chat message" handler on the backend adds the message to DynamoDB through AppSync, not directly to DynamoDB, so AppSync can fan out realtime updates to subscribed clients.
17. Authentication uses JWT. Go signs it on login with claims `user_id`, `name`, `group_ids`, and `exp`. AppSync authenticates browser queries and subscriptions with that JWT. AppSync authenticates Go mutations with an API key.
18. "Anyone" for a general chat means any authenticated user in the building/organization.
19. A user can access a group chat if their `group_ids` contains the chat's `group_id`.
20. Opening a chat calls `POST /chats/{id}/read`, which sets `UserChatRead.last_read_at`. The list shows a "new message" indicator when `last_message_at` > `last_read_at`.
21. **Chat message subscription** uses the browser `WebSocket` (`new WebSocket`), the same approach as Glitch's `useAppSyncEventSubscription` for the AppSync Events API. No Amplify, Apollo, or `graphql-ws` client.
    - This is the AppSync **GraphQL** realtime protocol, not the Events protocol (`aws-appsync-event-ws`).
    - Endpoint: `wss://<id>.appsync-realtime-api.<region>.amazonaws.com/graphql`.
    - Subprotocol: `graphql-ws`. Auth is the login JWT, sent as a `header-<base64url>` subprotocol (or `header` / `payload` query params). `host` must be the GraphQL HTTP domain.
    - Handshake: `connection_init` → `connection_ack` → `start` (subscription query in `payload.data`) → listen for `data`. Keepalive frames are `ka`. Unsubscribe with `stop`.
    - Treat silence past `connectionTimeoutMs` from `connection_ack` as a dead socket and reconnect with backoff, same idea as the Glitch hook.
    - GraphQL **queries** (list chats, load messages) use TanStack Query with a plain `fetch` POST to the AppSync GraphQL endpoint and the same JWT. No `graphql-request`, Apollo, or Amplify. Only the subscription uses the socket.

## Data Model

1. User (in-memory)
   - id: int
   - name: string
   - username: string
   - password: string
   - group_ids: int[]

2. Group (in-memory)
   - id: int
   - name: string

3. Location (in-memory)
   - id: int
   - name: string

4. Chat (DynamoDB) — conversation metadata
   - id: string (partition-key)
   - loc_id: int
   - type: string (`general` | `group`)
   - group_id: int (`0` for general; the Group.id for a group chat)
   - created_by: int (User.id)
   - created_at: int (unix epoch ms)
   - last_message_at: int (unix epoch ms, `0` when the chat has no messages)
   - is_active: bool
   - GSI `loc-index`: partition-key `loc_id`, sort-key `id` (so `listChats(loc_id)` is a Query, not a Scan)

5. ChatMessage (DynamoDB)
   - chat_id: string (partition-key)
   - sk: string (`<timestamp_ms>:<uuid>`, sort-key)
   - msg: string
   - sender_id: int (User.id)
   - created_at: int (unix epoch ms)

6. UserChatRead (DynamoDB) — unread indicator state
   - user_id: int (partition-key)
   - chat_id: string (sort-key)
   - last_read_at: int (unix epoch ms)

## Authorization

1. Browser queries and the subscription use the login JWT. Resolvers read `user_id` and `group_ids` from the claims.
2. A **general** chat is visible to any authenticated user. A **group** chat is visible when the caller's `group_ids` contains the chat's `group_id`.
3. `listMessages` and `onNewChatMessage` load the chat and apply rule 2. No access means the call is rejected.
4. Mutations are API-key only. The browser never calls them. Go calls them after the REST rules below.

## GraphQL schema

Browser calls queries with `fetch` and the subscription with `WebSocket`. Go calls mutations with the API key.

`listChats(loc_id)` returns chats in one location. The UI loads locations from `GET /locations`, then queries each location. A handful of rooms makes that the full chat list (FR5) without a Scan.

`createChatMessage` also sets `Chat.last_message_at` to the message `created_at`. `listChats` attaches `last_read_at` for the JWT user (`0` when that user has no `UserChatRead` row).

```graphql
type ChatSummary {
  id: ID!
  loc_id: Int!
  type: String!
  group_id: Int!
  created_by: Int!
  created_at: Int!
  is_active: Boolean!
  last_message_at: Int!
  last_read_at: Int!
}

type ChatMessage {
  chat_id: ID!
  sk: String!
  msg: String!
  sender_id: Int!
  created_at: Int!
}

type ChatMessageConnection {
  messages: [ChatMessage!]!
  nextToken: String
}

type UserChatRead {
  user_id: Int!
  chat_id: ID!
  last_read_at: Int!
}

type Query {
  listChats(loc_id: Int!): [ChatSummary!]!
  listMessages(chat_id: ID!, limit: Int, nextToken: String): ChatMessageConnection!
}

type Mutation {
  createChat(
    id: ID!
    loc_id: Int!
    type: String!
    group_id: Int!
    created_by: Int!
    created_at: Int!
    is_active: Boolean!
    last_message_at: Int!
  ): ChatSummary!
  createChatMessage(
    chat_id: ID!
    sk: String!
    msg: String!
    sender_id: Int!
    created_at: Int!
  ): ChatMessage!
  markChatRead(user_id: Int!, chat_id: ID!, last_read_at: Int!): UserChatRead!
}

type Subscription {
  onNewChatMessage(chat_id: ID!): ChatMessage
    @aws_subscribe(mutations: ["createChatMessage"])
}
```

`listMessages` returns newest first. Default `limit` is 50.

## REST API

JSON. Every route except `POST /login` requires `Authorization: Bearer <jwt>`.

| Method | Path | Body | Success | Errors |
| --- | --- | --- | --- | --- |
| POST | `/login` | `{ username, password }` | `200 { token, user: { id, name, username, group_ids } }` | `401` |
| GET | `/me` | | `200 { id, name, username, group_ids }` | `401` |
| GET | `/locations` | | `200 [{ id, name }]` | `401` |
| GET | `/groups` | | `200 [{ id, name }]` | `401` |
| POST | `/chats` | `{ loc_id, type, group_id? }` | `201` chat | `400` `401` `409` |
| POST | `/chats/{id}/messages` | `{ msg }` | `201` message | `400` `401` `403` `404` |
| POST | `/chats/{id}/read` | | `200 { user_id, chat_id, last_read_at }` | `401` `403` `404` |

`POST /chats`

- `loc_id` must exist. `type` is `general` or `group`.
- `general`: `group_id` is stored as `0`. If that location already has an active general chat, respond `409`.
- `group`: `group_id` is required, must exist, and the caller must belong to that group. If that location already has an active chat for the same `group_id`, respond `409`.
- Go sets `id` (uuid), `created_by` from the JWT, `created_at` (epoch ms), `is_active: true`, `last_message_at: 0`, then calls `createChat`.

`POST /chats/{id}/messages`

- `msg` must be non-empty after trim.
- The chat must exist and be active, and the caller must be allowed to access it (`403` otherwise, `404` if missing).
- Go sets `sender_id` from the JWT, `created_at` (epoch ms), `sk` = `<created_at>:<uuid>`, then calls `createChatMessage`.

`POST /chats/{id}/read`

- The chat must exist and the caller must be allowed to access it.
- Go sets `last_read_at` to now (epoch ms) and calls `markChatRead`.
